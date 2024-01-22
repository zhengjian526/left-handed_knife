---
layout: post
title: 记录一个多进程高并发框架(不断维护优化)
categories: Linux内核架构与系统编程
description: 记录一个多进程高并发框架(不断维护优化)
keywords: Linux, multi-process
---

# 多进程高并发代码框架
```c++
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdint.h>

//函数指针 返回值 xx 参数
typedef void (*spawn_proc_pt) (void *data);//函数指针，这里接收void* 类型的参数

static void worker_process_cycle(void *data);
static void start_worker_processes(int n);
pid_t spawn_process(spawn_proc_pt proc, void *data, char *name); 

int main(int argc,char **argv){
    //调用启动工作进程-4个
    start_worker_processes(4);
    //管理子进程
    wait(NULL);
}

//启动子进程
void start_worker_processes(int n){
    int i=0;
    for(i = n - 1; i >= 0; i--){
        //第一个参数为工作进程的处理周期
       spawn_process(worker_process_cycle,(void *)(intptr_t) i, "worker process");
    }
}

//创建子进程
pid_t spawn_process(spawn_proc_pt proc, void *data, char *name){

    pid_t pid;
    pid = fork();//创建子进程

    switch(pid){
    case -1:
        fprintf(stderr,"fork() failed while spawning \"%s\"\n",name);
        return -1;
    case 0:
          proc(data);
          return 0;
    default:
          break;
    }   
    printf("start %s %ld\n",name,(long int)pid);
    return pid;
}

//设置cpu亲和性，进程绑核
static void worker_process_init(int worker){
    cpu_set_t cpu_affinity;

    //多核高并发处理 
    CPU_ZERO(&cpu_affinity);
    //参数 -  cpu编号 -掩码地址
    CPU_SET(worker % CPU_SETSIZE,&cpu_affinity);
    //sched_setaffinity
    if(sched_setaffinity(0,sizeof(cpu_set_t),&cpu_affinity) == -1){
       fprintf(stderr,"sched_setaffinity() failed\n");
    }
}

void worker_process_cycle(void *data){
     int worker = (intptr_t) data;
    //工作进程初始化
     worker_process_init(worker);

    //干活
    for(;;){
      sleep(10);
      printf("pid %ld ,doing ...\n",(long int)getpid());
    }
}

```


# 参考

- https://blog.51cto.com/u_15333750/5587734
