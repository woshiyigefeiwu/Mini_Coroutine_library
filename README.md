# Mini_Coroutine_library
**基于 Linux ucontext 函数族实现的 简易的，非对称的 协程库**

----------------------------------------------------------------------------------------------------------------------------------------

**首先声明：这个项目有两个分支（main,v2）**

**main分支：实现的是最基础的协程...（也就是仅仅实现了函数的挂起和唤醒，然而协程远远不止于此！）** 

**v2分支：在main分支的基础上，对一些系统调用进行hook，实现了对异步的封装，真正体现了协程的作用。** 

----------------------------------------------------------------------------------------------------------------------------------------

这个 readme 讲的是main分支，封装异步的协程库 在v2分支（也有讲解）。

之前写的那个协程库有很大的问题...

就是之前的协程库切换时是通过协程的上下文和调度器中的那个上下文进行交换；

但是这样有很大的问题，这样也就是说我们只能在主线程中去唤醒协程，不能在协程中去唤醒其他的协程（这样的话会把调度器中的上下文给覆盖掉，所以会有问题）；

所以我改进了一下：

这个调度关系我们直接在协程体中维护，也就是说，在协程体结构体中还需要保存调取这个协程（父协程）的协程体指针；

这样子的话，在切换的时候就能够做到真正的非对称调度，而且更加灵活！

下面我做了两个测试：

test1：多个协程的嵌套唤醒和切出

test2：跨协程唤醒

具体可以看一下代码！

-------------------------------------------------------------------------------------------------------------------------------------------

**上面是协程库重写后的说明，下面是具体的协程库（main分支）的介绍，看完看 v2 分支。**

-------------------------------------------------------------------------------------------------------------------------------------------


### 项目介绍：

此项目是基于 Linux ucontext 函数族实现的 简易的、非对称的 协程库；

ucontext 函数族主要有 ucontext_t 结构体 以及 几个函数；

ucontext_t 结构体是用来存放当前上下文环境的（后继上下文，栈的信息）。

由于协程的切换需要保存上下文信息嘛，其实不只是协程，线程，进程的切换也需要保存上下文信息；

只不过是操作系统帮我们解决了，而协程是用户及线程，需要我们手动的进行上下文信息的设置。

协程的上下文切换需要借助下面几个函数，ucontext函数族，下面是几个函数：

```C++
int getcontext(ucontext_t* ucp)；
将当前程序的上下文信息保存到 ucp 中；
```

```C++
int setcontext(const ucontext_t *ucp)；
将ucontext_t结构体变量ucp中的上下文信息重新恢复到cpu中并执行；
```

```C++
void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...)
修改上下文信息，参数ucp就是我们要修改的上下文信息结构体；func是上下文的入口函数；
argc是入口函数的参数个数，后面的…是具体的入口函数参数，该参数必须为整形值。
```

```C++
int swapcontext(ucontext_t *oucp, ucontext_t *ucp)
功能：将当前cpu中的上下文信息保存带oucp结构体变量中，然后将ucp中的结构体的上下文信息恢复到cpu中。
这里可以理解为调用了两个函数，第一次是调用了getcontext(oucp)然后再调用setcontext(ucp)。
```

### 项目设计
在具体项目的架构设计方面，所以我设计了两个类，一个是协程体类，一个是调度器类：

协程类 里面：
```C++
    ucontext_t ctx;     //当前协程的上下文环境
    Fun func;           //当前协程要执行的函数
    void *arg;          //func的参数
    enum CoroutineState state;      //表示当前协程的状态
    char stack[DEFAULT_STACK_SZIE]; //每个协程的独立的栈

    int id;             //当前协程的编号
    coroutine *fa_co;   //调用当前协程的父协程体
```

调度器类 里面：
```C++
  ucontext_t main;            //主协程的上下文，方便切回主协程
  int running_thread;         //当前正在运行的协程编号
  coroutine *coroutine_pool;  //协程队列(协程池)
  int max_index;              // 曾经使用到的最大的index + 1
```

然后整体的架构是一个线程创建一个调度器，然后所有的有关协程的控制（创建，挂起，唤醒）都要由这个调度器来操作；

下面是一些主要函数：

协程的创建：co_create
```C++
	在调度器里面的协程池里面找到一个空位置；
	然后设置一下这个协程的部分参数（函数，以及函数的参数等）
	这里并没有直接就执行协程。
```

协程的挂起：co_yield
```C++
	将当前协程挂起，回到调度这个协程的父协程中；
	通过 swapcontext(&(t->ctx),&(t->fa_co->ctx));
	保存上下文到 t->ctx，切换到父协程中
```

协程的唤醒：co_resume
```C++
  这个就稍微复杂了一点：
	判断协程是否是第一次唤醒（协程是 就绪，还是 挂起状态；只有这两个状态才能被resume）
	如果是挂起状态（不是第一次唤醒）
		则直接切换到协程的上下文就行；
	如果是就绪状态（第一次唤醒）
		由于我们在创建的时候并没有设置上下文信息；
		所以在这里我们需要设置一下上下文信息（栈空间，栈空间大小，设置后继上下文）；
    并设置要被调用协程的父协程信息；
		同时设置协程的入口函数 makecontext(&(t->ctx),(void (*)(void))(co_body),1,&schedule);
		接着切换到协程的上下文就行。
```

还有一些辅助函数：

```C++
// 判断schedule中所有的协程是否都执行完毕，是返回1，否则返回0（其实没什么用hhh）
int schedule_finished(const schedule_t &schedule);
```

```C++
// 协程执行的入口函数
static void co_body(schedule_t *ps);
```


### 使用
git clone 下来；

然后可以看一下两个test


### 关于改进
由于我们在创建协程的时候是在协程数组里面找一个空闲的位置分给协程，这样的话时间复杂度就比较高；

我们可以参考 LRU 的改进算法：

维护两条协程链表，一条是创建好的协程（活跃链表），一条是空闲链表；
（当然这里还需要对 协程的编号 以及 协程体这个节点 做一个双向映射）
	
当我们创建协程的时候，直接在空闲链表里面取一个空闲节点；
初始化 并 加入到第一条链表就行；
	
当协程运行完成，我们就将它从第一条链表中删除，然后将这个节点加入空闲链表就行；

这样的时间复杂度会降低很多，由于需要对 协程号 和 协程体节点 做映射；

这里我考虑是用map，所以时间复杂度是O（logn）的；

当然用unordered_map的话，平均时间复杂应该是O（1）的。

（这个我就暂时还没改hhh）

### 然后看v2版本，这个才是重点！！！
