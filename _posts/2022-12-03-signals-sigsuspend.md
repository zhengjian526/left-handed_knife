---
layout: post
title: Linux等待信号--sigsuspend
categories: Linux系统编程
description: Linux等待信号--sigsuspend
keywords: Linux, signals, sigsuspend
---

## Linux 等待信号(sigsuspend)
```c++
/* sigsuspend()函数说明 */

#include <stdio.h>
#include <signal.h>

/*
知识补充:
    sigsuspend()函数

    函数原型:
    #include <signal.h>
    int sigsuspend(const sigset_t *mask);

    参数说明:
    @mask 希望屏蔽的信号

    返回值:
    sigsuspend返回后将恢复调用之前的的信号掩码。信号处理函数完成后，进程将继续执行。该系统调用始终返回-1，并将errno设置为EINTR.

    备注:
    进程执行到sigsuspend时，sigsuspend并不会立刻返回，进程处于TASK_INTERRUPTIBLE状态并立刻放弃CPU，
    等待UNBLOCK（mask之外的）信号的唤醒。进程在接收到UNBLOCK（mask之外）信号后，调用处理函数，然后还原信号集，
    sigsuspend返回，进程恢复执行。

    使用场景:
    sigsuspend() 函数可以更改进程的信号屏蔽字可以阻塞所选择的信号，或解除对它们的阻塞。
    使用这种技术可以保护不希望由信号中断的代码临界区。
    如果希望对一个信号解除阻塞，然后pause等待以前被阻塞的信号发生，那么必须使用 sigsuspend() 函数
    pause() 函数无法达成上述目的

    //a.阻塞信号
    if(sigprocmask(SIG_BLOCK, &newmask, &oldmask) < 0)
        err_sys("SIG_BLOCK error");

    //b.解除阻塞
    if(sigprocmask(SIG_SETMASK, &oldmask, NULL) < 0)
        err_sys("SIG_SETMASK error");

    //c.等待信号
    pause();    

    说明: 如果在等待信号阶段，发送信号给程序，那么 pause()函数退出，处理信号
        如果在解除阻塞之前发送信号，信号一被解除，立马执行信号处理函数，执行完毕后，回来执行 pause()函数，程序被卡住

        为了纠正此问题，需要在一个原子操作中先恢复信号屏蔽字，然后使进程休眠。这种功能是由 sigsuspend() 函数提供的。

*/

void test()
{
    sigset_t set;

    //1.设置需要处理的信号
    sigemptyset(&set);
    sigaddset(&set, SIGCHLD);
    sigaddset(&set, SIGALRM);
    sigaddset(&set, SIGIO);
    sigaddset(&set, SIGINT);

    //2.先屏蔽这些信号
    if (sigprocmask(SIG_BLOCK, &set, NULL) == -1) 
    {
        printf("sigprocmask() failed .\n");
        return;
    }

    //3.做一些初始化操作

    //4.开始接收这些信号
    sigemptyset(&set);    //清空信号集

    sigsuspend(&set);    //等待信号过来处理

}

int main()
{
    test();
    printf("-----ok-------\n");
    return 0;
}
```


## 参考

- https://www.cnblogs.com/zhanggaofeng/p/12050961.html
