ucontext机制是GNU C库提供的一组创建，保存，切换用户态执行上下文的API，从上面的描述可以看出ucontext_t结构体使得用户在程序中保存当前的上下文成为可能。我们也可以利用此实现用户级线程，即协程。
# ucontext_t以及ucontext族函数
## ucontext_t结构体
ucontext_t结构体定义,一个ucontext_t至少包括以下四个成员,可能依据不同系统包括其他不同的成员。
```cpp
#include <ucontext.h>
typedef struct ucontext_t {
  struct ucontext_t* uc_link;
  sigset_t uc_sigmask;
  stack_t uc_stack;
  mcontext_t uc_mcontext;
  ...
};
```
类成员解释:

- uc_link:为当前context执行结束之后要执行的下一个context，若uc_link为空，执行完当前context之后退出程序。
- uc_sigmask : 执行当前上下文过程中需要屏蔽的信号列表，即信号掩码
- uc_stack : 为当前context运行的栈信息。
- uc_mcontext : 保存具体的程序执行上下文，如PC值，堆栈指针以及寄存器值等信息。它的实现依赖于底层，是平台硬件相关的。此实现不透明。
## ucontext族函数
ucontext族函数主要包括以下四个:
```cpp
#include <ucontext.h>
void makecontext(ucontext_t* ucp, void (*func)(), int argc, ...);
int swapcontext(ucontext_t* olducp, ucontext_t* newucp);
int getcontext(ucontext_t* ucp);
int setcontext(const ucontext_t* ucp);
```
makecontext:初始化一个ucontext_t,func参数指明了该context的入口函数，argc为入口参数的个数，每个参数的类型必须是int类型。另外在makecontext之前，一般需要显示的初始化栈信息以及信号掩码集同时也需要初始化uc_link，以便程序退出上下文后继续执行。
swapcontext:原子操作，该函数的工作是保存当前上下文并将上下文切换到新的上下文运行。
getcontext:将当前的执行上下文保存在ucp中，以便后续恢复上下文
setcontext : 将当前程序切换到新的context,在执行正确的情况下该函数直接切换到新的执行状态，不会返回。
注意:setcontext执行成功不返回，getcontext执行成功返回0，若执行失败都返回-1。若uc_link为NULL,执行完新的上下文之后程序结束。
# 使用案例
```cpp
#include <ucontext.h>
#include <iostream>
#include <unistd.h>

void newContextFun() {
  std::cout << "this is the new context" << std::endl;
}

int main() {
  char stack[10*1204];

  //get current context
  ucontext_t curContext;
  getcontext(&curContext);

  //modify the current context
  ucontext_t newContext = curContext;
  newContext.uc_stack.ss_sp = stack;
  newContext.uc_stack.ss_size = sizeof(stack);
  newContext.uc_stack.ss_flags = 0;

  newContext.uc_link = &curContext;

  //register the new context
  makecontext(&newContext, (void(*)(void))newContextFun, 0);
  swapcontext(&curContext, &newContext);
  printf("main\n");

  return 0;
}

```
# 转载
[https://blog.csdn.net/u014630623/article/details/89020088](https://blog.csdn.net/u014630623/article/details/89020088)
