---
layout: post
title: Linux进程间通信--共享内存
categories: Linux系统编程
description: Linux进程间通信--共享内存
keywords: Linux, IPC, 共享内存
---

## 背景
项目采用基于GRPC框架后，客户端和服务端单次交互过程中存在large size消息(超过20M)。因为客户端和服务端同在同一服务器中部署，所以期望通过GRPC + IPC的方式提高硬件利用率，降低时延。
关于此过程中共享内存的使用方式，记录如下。

## 先说结论
使用过程中发现应用侧开发过程中常用的共享内存主要有两套接口：

1. System V 
2. POSIX

实际使用感受POSIX接口更加灵活方便，对于需要创建多个共享内存或者做共享内存池的场景更加友好。

## System V接口使用

函数使用方法可以通过man func_name 或者查看相关参考资料学习

```c++
// shared_memory.h
class SharedMemory {
 public:
  int Create(const std::string& path, uint64_t memory_size);
  int Attach();
  void Detach();
  void Destroy();
  // 创建文件名 保证唯一性
  std::string GetRandFileName(const std::string prefix);
  uint8_t *GetSharedMemoryAddr() { return shmat_addr_; }

 private:
  // std::mutex mutex_;
  std::string file_path_;
  int shm_id_ = -1;
  key_t key_;
  uint8_t *shmat_addr_ = nullptr;
  uint32_t size_ = 0;
};
```

```c++
// shared_memory.cpp
int SharedMemory::Create(const std::string& path, uint64_t memory_size)
{
    file_path_ = path;
    /* ftok生成key_t， 是System V IPC key类型。这个函数时根据已存在文件的路径名和proj_id 的8bit有效位生成key。
	    key_t ftok(const char *pathname, int proj_id);
	    其底层是根据inode索引节点编号生成，所以这里如果重新删建，有可能会导致key重复。
    */
    if ((key_ = ftok(path.c_str(), 10)) < 0) { 
        std::cout << path + ": ftok error!" << std::endl;
        return -1;
    }
    size_ = memory_size;
    auto access_mode = S_IRUSR | S_IWUSR | S_IROTH | S_IWOTH | S_IRGRP | S_IWGRP;
    // shm_id_ = shmget(key_, memory_size, IPC_CREAT | IPC_EXCL | access_mode);
    shm_id_ = shmget(key_, memory_size, IPC_CREAT | access_mode);
    if (shm_id_ == -1) {
        std::cout << "Shared memory creation failed. Errno " + std::to_string(errno) << std::endl;
        return -1;
    }
    return 0;
}
int SharedMemory::Attach(){
    void *shmat_addr = shmat(shm_id_, nullptr, 0);
    if (shmat_addr == reinterpret_cast<void *>(-1)) {
        std::cout << "Shared memory attach failed. Errno " + std::to_string(errno) << std::endl;
        return -1;
    }
    shmat_addr_ = reinterpret_cast<uint8_t *>(shmat_addr);
    return 0;
}
void SharedMemory::Detach()
{
    if (shmat_addr_ != nullptr) {
        auto err = shmdt(shmat_addr_);
        if (err == -1) {
            std::cout << "Shared memory detach failed. Errno " + std::to_string(errno) << std::endl;
            return;
        }
    }
    shmat_addr_ = nullptr;
}
void SharedMemory::Destroy()
{
    auto err = shmctl(shm_id_, IPC_RMID, nullptr);
    if (err == -1) {
        std::string errMsg = "Unable to remove shared memory with id " + std::to_string(shm_id_);
        errMsg += ". Errno :" + std::to_string(errno);
        errMsg += "\nPlesae remove it manually using ipcrm -m command";
         std::cout << errMsg << std::endl;
    }
    err = remove(file_path_.c_str());
    if (err == -1) {
        std::cout << "Remove file: "<< file_path_ << " failed. Errno " + std::to_string(errno) << std::endl;
    }
    return;
}

```

共享内存创建后可以通过ipcs 命令查看相应的共享内存信息

![image-20221205101152323](/images/posts/shared_mem/image-20221205101152323.png)
**还有两个需要注意的点：**
1. 客户端和服务端detach后不会自动删除共享内存,需要调用shmctl进行删除。如果忘记代码中删除，需要手动输入命令行ipcrm进行删除。
2. 创建用来关联共享内存的文件不会自动删除，需要在相应的时机进行删除。对于System V接口可以在使用者使用完成后进行删除释放，对应代码为上文中的Destroy()接口。即创建者创建->创建者detach->使用者attach使用->使用者detach->使用者destroy().


## POSIX 接口使用

```c++
// shared_memory.h
struct SharedMemory {
  std::string memory_key;
  uint64_t bytes_size = 0;
  uint8_t *address = nullptr;
  // std::set<uint64_t> free_queue;
};

struct SharedMemoryGroup {
  std::unordered_map<std::string, std::shared_ptr<SharedMemory>> shm_map_;
  std::queue<std::string> free_queue_;
};

class SharedMemoryAllocator {
 public:
  static SharedMemoryAllocator &Instance();
  SharedMemoryAllocator();
  ~SharedMemoryAllocator();
  int NewMemoryBuffer(uint64_t item_size, uint64_t init_item_count);
  std::string AllocMemory(uint64_t mem_size);
  int ReleaseMemory(const std::string &memory_key);
  void MemPoolInfo();
 private:
  std::string GetRandFileName(const std::string prefix);
  int AddShmMemoryBuffer(std::shared_ptr<SharedMemoryGroup>& shm_group, uint64_t item_size);
 private:
  
  std::mutex lock_;
  std::map<int64_t, std::shared_ptr<SharedMemoryGroup>> size_group_map_;
};

struct SharedMemoryAttach {
  std::string memory_key;
  uint64_t bytes_size = 0;
  uint8_t *address = nullptr;
};
class SharedMemoryManager {
 public:
  static SharedMemoryManager &Instance();
  SharedMemoryManager();
  ~SharedMemoryManager();
  std::shared_ptr<SharedMemoryAttach> Attach(const std::string &memory_key, uint64_t bytes_size);
  int Detach(const std::string &memory_key);
 private:
  std::vector<std::shared_ptr<SharedMemoryAttach>> attached_shm_list_;
  std::mutex lock_;
};

```

```c++
// shared_memory.cpp
std::atomic<uint64_t> shared_mem_counter{0};
SharedMemoryAllocator &SharedMemoryAllocator::Instance() 
{
  static SharedMemoryAllocator instance;
  return instance;
}
SharedMemoryAllocator::SharedMemoryAllocator() = default;

SharedMemoryAllocator::~SharedMemoryAllocator() 
{
    std::unique_lock<std::mutex> lock(lock_);
    for(auto& group : size_group_map_) {
        auto& shm_map_ = group.second->shm_map_;
        for(auto& item: shm_map_) { 
            auto ret = munmap(item.second->address, item.second->bytes_size);
            if (ret == -1) {
                std::cout << "Failed to munmap, memory key: " << item.second->memory_key << std::endl;
            }
            ret = shm_unlink(item.second->memory_key.c_str());
            if (ret == -1) {
                std::cout << "Failed to shm_unlink " << item.second->memory_key 
                        << ", errno: " << errno << std::endl;
            }
        }
        shm_map_.clear();
    }
    size_group_map_.clear();
      std::cout << "SharedMemoryAllocator destroy" <<std::endl; 
}
std::string SharedMemoryAllocator::GetRandFileName(const std::string prefix)
{
    std::string ret = prefix + std::to_string(getpid()) + "_"  + std::to_string(shared_mem_counter);
    if(shared_mem_counter != std::numeric_limits<uint64_t>::max()) {
        shared_mem_counter++;
    }
    else {
        shared_mem_counter = 0;
    }
    return ret;
}
int SharedMemoryAllocator::AddShmMemoryBuffer(std::shared_ptr<SharedMemoryGroup>& shm_group, uint64_t item_size) {
    const auto memory_key = GetRandFileName("xxx_");
    if(shm_group->shm_map_.find(memory_key) != shm_group->shm_map_.end()) {
        std::cout << "Shared memory key : "<< memory_key << " has already been inited" << std::endl;
    }
    // auto shm_fd = shm_open(memory_key.c_str(), O_CREAT | O_RDWR, S_IRUSR | S_IWUSR);
    mode_t old_umask = umask(0);
    // shm_open 需要在创建是指定权限, 另外需要清楚umask, 避免屏蔽掉部分权限
    auto shm_fd = shm_open(memory_key.c_str(), O_CREAT | O_RDWR, S_IRWXU | S_IRWXG | S_IRWXO);
    umask(old_umask);
    if (shm_fd == -1) {
        std::cout << "Failed to shm_open " << memory_key << " , errno: " << errno << std::endl;
        return -1;
    }
    auto ret = ftruncate(shm_fd, item_size);
    if (ret == -1) {
        std::cout << "Failed to ftruncate " << memory_key << ", errno: " << errno 
                    << ", memory size: " << item_size << std::endl;
        return -1;
    }
    auto address = mmap(nullptr, item_size, PROT_READ | PROT_WRITE, MAP_SHARED, shm_fd, 0);
    if (address == MAP_FAILED) {
        std::cout << "Failed to mmap " << memory_key << ", errno: " << errno 
                    << ", memory size: " << item_size << std::endl;
        return -1;
    }
    ret = close(shm_fd);
    std::shared_ptr<SharedMemory> shared_mem = std::make_shared<SharedMemory>();
    shared_mem->memory_key = memory_key;
    shared_mem->bytes_size = item_size;
    shared_mem->address = reinterpret_cast<uint8_t *>(address);
    shm_group->shm_map_[memory_key] = std::move(shared_mem);
    shm_group->free_queue_.push(memory_key);
    return 0;
}
int SharedMemoryAllocator::NewMemoryBuffer(uint64_t item_size, uint64_t item_count) 
{
    std::unique_lock<std::mutex> lock(lock_);
    if(size_group_map_.find(item_size) == size_group_map_.end()){
        size_group_map_[item_size] = std::make_shared<SharedMemoryGroup>();
    }
    auto& group = size_group_map_[item_size];
    for (size_t i = 0; i < item_count; i++) {
        auto ret = AddShmMemoryBuffer(group, item_size);
        if(ret == -1){
            std::cout << "Insert shared mem buffer failed. " << std::endl;
            return -1;
        }
    }
    return 0;
}
std::string SharedMemoryAllocator::AllocMemory(uint64_t mem_size)
{
    std::unique_lock<std::mutex> lock(lock_);
    for(auto& group : size_group_map_) {
        if(group.first < mem_size){
            continue;
        }
        else {
            if(group.second->free_queue_.empty() == false) {
                auto ret = group.second->free_queue_.front();
                group.second->free_queue_.pop();
                return ret;
            }
            else {
                auto status = AddShmMemoryBuffer(group.second, group.first);
                if (status == -1) {
                    std::cout << "Insert shared mem buffer failed. " << std::endl;
                    return "";
                }
                auto ret = group.second->free_queue_.front();
                group.second->free_queue_.pop();
                return ret;
            }
        }
    }
    size_group_map_[mem_size] = std::make_shared<SharedMemoryGroup>();
    auto& group = size_group_map_[mem_size];
    auto status = AddShmMemoryBuffer(group, mem_size);
    if(status == -1){
        std::cout << "Insert shared mem buffer failed. " << std::endl;
        return "";
    }
    auto ret = group->free_queue_.front();
    group->free_queue_.pop();
    return ret;
}
int SharedMemoryAllocator::ReleaseMemory(const std::string &memory_key)
{
    std::unique_lock<std::mutex> lock(lock_);
    for (const auto p : size_group_map_) {
        if(p.second->shm_map_.find(memory_key) != p.second->shm_map_.end()) {
            p.second->free_queue_.push(memory_key);
            return 0;
        }
    }
    std::cout << "memory_key: " << memory_key << " is not exist. " << std::endl;
    return -1;
}
void SharedMemoryAllocator::MemPoolInfo()
{
    std::ostringstream oss;
    for (const auto it : size_group_map_) {
        oss << "mem size: " << std::to_string(it.first) << ", ";
        oss << "group size: " << it.second->shm_map_.size() << ", ";
        oss << "free size: " << it.second->free_queue_.size() << "\n";
    }
    std::cout << oss.str();
}
/**
 * @brief Shared Memory Manager object
 * 
 */
SharedMemoryManager::SharedMemoryManager() = default;
SharedMemoryManager::~SharedMemoryManager() {
  std::unique_lock<std::mutex> lock(lock_);
  for (auto &item : attached_shm_list_) {
    auto ret = munmap(item->address, item->bytes_size);
    if (ret == -1) {
      std::cout << "Failed to munmap, memory key: " << item->memory_key << std::endl;
    }
  }
  std::cout << "SharedMemoryManager destroy" <<std::endl; 
  attached_shm_list_.clear();
}

SharedMemoryManager &SharedMemoryManager::Instance() {
  static SharedMemoryManager instance;
  return instance;
}

std::shared_ptr<SharedMemoryAttach> SharedMemoryManager::Attach(const std::string &memory_key, uint64_t bytes_size) {
  std::unique_lock<std::mutex> lock(lock_);
  的mode参数总是需要设置, 如果oflag没有指定了O_CREAT，可以指定为0
  auto shm_fd = shm_open(memory_key.c_str(), O_RDWR, 0); // open existing object;
  if (shm_fd == -1) {
    std::cout << "Failed to shm_open " << memory_key << " , errno: " << errno << std::endl;
    return nullptr;
  }
  auto address = mmap(nullptr, bytes_size, PROT_READ | PROT_WRITE, MAP_SHARED, shm_fd, 0);
  if (address == MAP_FAILED) {
    std::cout << "Failed to mmap " << memory_key 
              << ", errno: " << errno << ", memory size: " << bytes_size << std::endl;
    return nullptr;
  }
  auto ret = close(shm_fd);
  if (ret == -1) {
    std::cout << "Failed to close " << memory_key << ", errno: " << errno << std::endl;
    return nullptr;
  }
  std::shared_ptr<SharedMemoryAttach> attach_mem = std::make_shared<SharedMemoryAttach>();
  attach_mem->memory_key = memory_key;
  attach_mem->bytes_size = bytes_size;
  attach_mem->address = static_cast<uint8_t *>(address);
  attached_shm_list_.push_back(attach_mem);
  return attach_mem;
}
int SharedMemoryManager::Detach(const std::string &memory_key) {
  std::unique_lock<std::mutex> lock(lock_);
  auto it = std::find_if(attached_shm_list_.begin(), attached_shm_list_.end(),
                         [&memory_key](const std::shared_ptr<SharedMemoryAttach> &item) { return memory_key == item->memory_key; });
  if (it == attached_shm_list_.end()) {
     std::cout << "Cannot find shared memory " << memory_key << std::endl;
     return -1;
  }
  auto ret = munmap((*it)->address, (*it)->bytes_size);
  if (ret == -1) {
    std::cout << "Failed to munmap, memory key: " << memory_key << std::endl;
    return -1;
  }
  attached_shm_list_.erase(it);
  return 0;
}

```



## 参考

- <Linux/Unix 系统编程手册>
