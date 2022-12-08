fork 的作用是克隆进程，由已经存在的进程调用 fork，然后新进程拷贝原有的进程，它们之间没有太大的区别。下面是 fork 函数的原型，它有三个返回值。
```c
pid_t fork(void);
```

1. 该进程为父进程时，返回子进程的 pid
2. 该进程为子进程时，返回 0
3. fork执行失败，返回 -1
## 使用 fork() 创建子进程
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

void error_handler(char* msg)
{
    perror(msg);
    exit(1);
}

int main()
{
    pid_t pid;

    printf("before execute fork\n");
    
    pid = fork();
    if (pid == -1) {
        error_handler("fork() failed\n");
    } else if (pid > 0) {
        printf("I am parent, my pid = %u, my ppid = %u\n", getpid(), getppid());
        sleep(1);
    } else if (pid == 0) {
        printf("I am child, my pid = %u, my ppid = %u\n", getpid(), getppid());
    }

    return 0;
}
```
```shell
before execute fork
I am parent, my pid = 921, my ppid = 32479
I am child, my pid = 922, my ppid = 921
```
可以从结果看到，子进程的 ppid 就是父进程的 pid，子进程由父进程创建而来。我们也可以查看父进程的 ppid是谁。
```shell
ps aux | grep 32479
```
```shell
root       974  0.0  0.0  16196  1132 pts/4    S+   15:47   0:00 grep --color=auto 32479
root     32479  0.0  0.1  23376  4004 pts/4    Ss   15:22   0:00 /bin/bash
```
父进程由 /bin/bash 创建，也是通过 fork 的方法。
## 循环创建多个子进程
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

const int PROCESS_NUMBER = 5;

void error_handler(char* msg)
{
    perror(msg);
    exit(1);
}

int main()
{
    pid_t pid;
    int i;

    printf("before execute fork\n");
    
    for (i = 0; i < PROCESS_NUMBER; i++) {
        pid = fork();
        if (pid == -1) {
            error_handler("fork() failed\n");
        } else if (pid == 0) {
            break;
        }
    }

    if (i < PROCESS_NUMBER) {
        sleep(i);
        printf("I am %d child, my pid = %u, my ppid = %u\n", i + 1, getpid(), getppid());
    } else if (i == PROCESS_NUMBER) {
        sleep(i);
        printf("I am parent, my pid = %u, my ppid = %u\n", getpid(), getppid());
    }
   
    return 0;
}
```

```c
before execute fork
I am 1 child, my pid = 1388, my ppid = 1387
I am 2 child, my pid = 1389, my ppid = 1387
I am 3 child, my pid = 1390, my ppid = 1387
I am 4 child, my pid = 1391, my ppid = 1387
I am 5 child, my pid = 1392, my ppid = 1387
I am parent, my pid = 1387, my ppid = 32479
```
创建多个进程时，多个进程会抢占 CPU。所以为了保证进程有序执行，我们使用 sleep 函数让指定进程沉睡，从而让它不参与竞争。在这段期间，让别的进程抢占 CPU 去执行程序。

我们希望让先创建的子进程输出信息，并且间隔一秒输出下一子进程信息，并且父进程最后退出程序。通过 sleep 函数使得子进程有序打印，后创建的子进程沉睡时间更久，使得间隔一秒打印子进程信息。

而父进程打印信息时，i 值最大。因此，沉睡最久，不会提前退出。

## 多进程环境分析

我们再延申一下，shell 也是一个进程。每当我们运行命令时，我们的 shell 进程就会隐藏（阻塞）在后面，等待我们的程序运行完毕，然后才继续跟我们交互。

```shell
root@iZwz9eojvzsrz78f673t51Z:/system_code/fork# gcc forks.c -o forks & ./forks
```

shell 阻塞住，运行我们的 forks 程序。

```shell
before execute fork
I am 1 child, my pid = 1432, my ppid = 1430
I am 2 child, my pid = 1433, my ppid = 1430
I am 3 child, my pid = 1434, my ppid = 1430
I am 4 child, my pid = 1435, my ppid = 1430
I am 5 child, my pid = 1436, my ppid = 1430
I am parent, my pid = 1430, my ppid = 32479
[1]+  Done                    gcc forks.c -o forks
```

程序 return 0 运行完毕，shell 重新掌握 CPU。

```shell
root@iZwz9eojvzsrz78f673t51Z:/home/shang/code/C++/system_code/fork#
```

再看下面这个例子。这是第一个案例的代码的运行情况，如果我们去掉 sleep 函数，就会出现这样的现象。

```shell
I am parent, my pid = 2079, my ppid = 1896
root@iZwz9eojvzsrz78f673t51Z:/home/shang/code/C++/system_code/fork# I am child, my pid = 2080, my ppid = 1
```

因为我们的父进程先退出了，但是创建的子进程还没执行。这时，shell 进程就会和创建出的子进程争抢终端，所以我们才会看到子进程的输出信息在 shell 的提示信息后。

shell 是看到了父进程程序的 return 0 才知道这个进程结束了，我可以抢占了。所以我们让子进程在父进程之前结束，我们让父进程 sleep(1) 去等待子进程执行完毕。

## fork 分析 
传统的 fork 直接拷贝所有资源给子进程，着过于简单且效率低下，慢且浪费资源。

Linux 的 fork 使用 `写时拷贝` 实现，写时拷贝是一种可以推迟甚至免除拷贝数据的技术。内核并不复制整个地址空间，而是父子进程共享同一个拷贝，只有在需要写入的时候，才会复制（在此之前是以只读方式共享）。

通常来说，我们 fork 得到的子进程都会直接执行另一个程序，所以实在没有必要再将父进程的所有数据复制到子进程，因为这些数据很大可能就没有被使用。比如 fork 之后立即调用 exec，那么就无需复制操作了。这样子fork 的实际开销就是复制父进程的页表以及给子进程创建 `task_struct`。