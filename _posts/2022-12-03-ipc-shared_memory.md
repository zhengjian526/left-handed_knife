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



## 参考

- <Linux/Unix 系统编程手册>
