## 前言

exec函数族是什么，这我们就需要从进程的创建方式开始讲了。不同的操作系统对于进程的创建有不同的实现方法，我们主要学习的是 Linux 下的进程创建方式。

- 其他操作系统：提供产生进程的机制，在新的地址空间创建进程，读入可执行文件，然后执行
- Unix：将上述步骤分成 fork 和 exec 来执行

   - fork 通过拷贝当前进程创建一个子进程，子进程和父进程的 PID、PPID、某些资源的统计量(比如挂起的信号)  不一样
   - exec 函数负责读取可执行文件并将其载入地址空间开始运行

而 fork 我们之前已经讲过了，接下来就是讲解 exec 是什么了。

## exec函数族

exec 指的是一组函数，一共有 6 个：

```
#include <unistd.h>
extern char **environ;

int execl(const char *path, const char *arg);
int execlp(const char *file, const char *arg);
int execle(const char *path, const char *arg, char * const envp[] */);
int execv(const char *path, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execvpe(const char *file, char *const argv[], char *const envp[]);
```

其中只有 execve() 是真正意义上的系统调用，其它都是在此基础上经过包装的库函数。

exec 函数族提供了六种在进程中启动另一个程序的方法。exec 函数族的作用是根据指定的文件名或目录名找到可执行文件，并用它来取代调用进程的内容，换句话说，就是在调用进程内部执行一个可执行文件。

exec 函数族只会在出错时返回，且返回值为 -1，并且写入了错误号到环境变量中。失败后从原程序的调用点接着往下执行。

**函数说明**

- path：可执行文件的路径名字
- arg：可执行程序所带的参数，arg 必须以 NULL 结尾
- argv：传入指针数组
- file：如果其中包含 `/`，则将其视为路径名，否则视为环境变量，在指定的目录中搜索

## execl

下面，我们将通过使用 `int execl(const char *path, const char *arg);` 函数来更好的理解。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    printf("before execl\n");
 	
    // 执行外部程序
	execl("/bin/ls", "ls", "-a", "-l", "-h", NULL);
    
    // 如果 execl() 执行成功，下面执行不到，因为当前进程已经被执行的 ls 替换了
	perror("execl() failed");
    printf("after exec\n");
 
    return 0;
}
```

参数分析，我们想要让这个程序执行 `ls -a -l -h` 命令，但是我们想通过 execl 函数来实现。

- `/bin/ls`：外部程序，这里是 /bin 目录的 ls 可执行程序，必须带上路径（相对或绝对）
- `ls`：没有意义，如果需要给这个外部程序传参，这里必须要写上字符串，至于字符串内容任意（写成 lssss 也不影响执行）
- `-a，-l，-h`：给外部程序 ls 传的参数
- `NULL`：这个必须写上，代表给外部程序 ls 传参结束

```shell
before execl
total 24K
drwxr-xr-x 2 root root 4.0K Jun 12 17:04 .
drwxr-xr-x 6 root root 4.0K Jun 12 16:58 ..
-rwxr-xr-x 1 root root 8.3K Jun 12 17:04 execl
-rw-r--r-- 1 root root  914 Jun 12 17:04 execl.c
```

## execv

execv 用法与 execl 基本一致，只不过是以指针数组的方式传入命令参数。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
    char *arg[]={"ls", "-a", "-l", "-h", NULL};
    printf("before execl\n");

    // /bin/ls：外部程序，这里是/bin目录的 ls 可执行程序，必须带上路径（相对或绝对）
    // arg： 上面定义的指针数组地址
	execv("/bin/ls", arg);
	perror("execv");
 
    printf("after exec\n");

    return 0;
}
```

## execlp 和 execvp

execlp() 和 execl() 的区别在于，execlp() 指定的可执行程序可以不带路径名，如果不带路径名的话，会在环境变量 PATH 指定的目录里寻找这个可执行程序，而 execl() 指定的可执行程序，必须带上路径名。

```c
#include <stdio.h>
#include <unistd.h>

int main(int argc, char* argv[])
{
    execlp("ls", "ls", "-a", "-l", "-h", NULL);
    perror("execlp");

    return 0;
}
```

execlvp() 也在环境变量 PATH 指定的目录中寻找可执行程序。但是它是以指针数组的方式传参。

```c
#include <stdio.h>
#include <unistd.h>

int main(int argc, char* argv[])
{
    char *arg[]={"ls", "-a", "-l", "-h", NULL};
    
    execvp("ls", arg);
    perror("execvp");

    return 0;
}
```

## execle 和 execve

execle() 和 execve() 改变的是 exec 启动的程序的环境变量（**只会改变进程的环境变量，不会影响系统的环境变量**），其他四个函数启动的程序则使用默认系统环境变量。

```c
#include <stdio.h>
#inclu-de <unistd.h>
#include <stdlib.h> // getenv()

int main(int argc, char *argv[])
{
    // getenv() 获取指定环境变量的值
    printf("before exec: USER=%s, HOME=%s\n", getenv("USER"), getenv("HOME"));

    // 指针数据
    char *env[]={"USER=MIKE", "HOME=/tmp", NULL};
    
    /* 
        ./mike：外部程序，当前路径的 mike 程序，通过 gcc mike.c -o mike 编译
        mike：这里没有意义
        NULL：给 mike 程序传参结束
        env：改变 mike 程序的环境变量，正确来说，让 mike 程序只保留 env 的环境变量
    */
    execle("./mike", "mike", NULL, env);

    /*
        char *arg[]={"mike", NULL};		
        execve("./mike", arg, env);	
    */

    perror("execle");

    return 0;
}
```

mike.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
 
int main(int argc, char *argv[])
{
    printf("\nin the mike fun, after exec: \n");
    printf("USER=%s\n", getenv("USER"));
    printf("HOME=%s\n", getenv("HOME"));
    
    return 0;
}
```

```shell
gcc mike.c -o mike
./execle
before exec: USER=root, HOME=/root

in the mike fun, after exec: 
USER=MIKE
HOME=/tmp
```

## 参考

[Linux系统编程——进程替换：exec 函数族 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/203015620)
