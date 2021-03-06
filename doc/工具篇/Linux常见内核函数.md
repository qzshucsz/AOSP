Linux系统有提供许多方便的API，就像Andoird中TextView的setText方法一样，我们只需要简单调用就可以实现一些功能，为了方便大家阅读Linux源码，我将一些常用的API列举出来

我先大致分个类吧

- 进程与进程调度
- 同步与锁
- 内存与内存策略

## 一、进程与进程调度
### 1.1 kernel_thread
出现在《Android系统启动流程之Linux内核》
```C
kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);
```
这个函数作用是启动进程
- 第一个参数表示新进程工作函数，相当于Java的构造函数
- 第二个参数是工作函数的参数，相当于Java带参构造函数的参数
- 第三个参数表示启动方式

|参数名|作用|
| :-- | :-- |
| CLONE_PARENT | 创建的子进程的父进程是调用者的父进程，新进程与创建它的进程成了“兄弟”而不是“父子”|
| CLONE_FS    |      子进程与父进程共享相同的文件系统，包括root、当前目录、umask |
| CLONE_FILES   |  子进程与父进程共享相同的文件描述符（file descriptor）表 |
| CLONE_NEWNS | 在新的namespace启动子进程，namespace描述了进程的文件hierarchy |
| CLONE_SIGHAND | 子进程与父进程共享相同的信号处理（signal handler）表 |
| CLONE_PTRACE | 若父进程被trace，子进程也被trace |
| CLONE_UNTRACED | 若父进程被trace，子进程不被trace |
| CLONE_VFORK  |  父进程被挂起，直至子进程释放虚拟内存资源 |
| CLONE_VM     |     子进程与父进程运行于相同的内存空间 |
| CLONE_PID    |    子进程在创建时PID与父进程一致 |
| CLONE_THREAD  | Linux 2.4中增加以支持POSIX线程标准，子进程与父进程共享相同的线程群 |
### 1.2 sched_setscheduler_nocheck
出现在《Android系统启动流程之Linux内核》
```C
sched_setscheduler_nocheck(kthreadd_task, SCHED_FIFO, &param);
```
这个函数作用是设置进程调度策略
- 第一个参数是进程的task_struct
- 第二个是进程调度策略
- 第三个是进程优先级

进程调度策略如下:
- SCHED_FIFO和SCHED_RR和SCHED_DEADLINE则采用不同的调度策略调度实时进程，优先级最高

- SCHED_NORMAL和SCHED_BATCH调度普通的非实时进程，优先级普通

- SCHED_IDLE则在系统空闲时调用idle进程，优先级最低

参考[Linux进程调度器的设计](http://blog.csdn.net/gatieme/article/details/51702662)
## 二、同步与锁
### 2.1 rcu_read_lock、rcu_read_unlock
出现在《Android系统启动流程之Linux内核》
```C
	rcu_read_lock(); 
	kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
	rcu_read_unlock();
```
RCU（Read-Copy Update）是数据同步的一种方式，在当前的Linux内核中发挥着重要的作用。RCU主要针对的数据对象是链表，目的是提高遍历读取数据的效率，为了达到目的使用RCU机制读取数据的时候不对链表进行耗时的加锁操作。这样在同一时间可以有多个线程同时读取该链表，并且允许一个线程对链表进行修改（修改的时候，需要加锁）

参考[Linux 2.6内核中新的锁机制--RCU](https://www.ibm.com/developerworks/cn/linux/l-rcu/)

## 三、内存与内存策略
### 3.1 numa_default_policy
设定NUMA系统的默认内存访问策略

## 四、通信
### 4.1 int socketpair(int d, int type, int protocol, int sv[2])
创建一对socket，用于本机内的进程通信<br>
参数分别是：<br>
- d 套接口的域 ,一般为AF_UNIX，表示Linux本机<br>
- type 套接口类型,参数比较多<br>
SOCK_STREAM或SOCK_DGRAM，即TCP或UDP<br>
SOCK_NONBLOCK   read不到数据不阻塞，直接返回0<br>
SOCK_CLOEXEC    设置文件描述符为O_CLOEXEC <br>
- protocol 使用的协议，值只能是0<br>
- sv 指向存储文件描述符的指针<br>

### 4.2  int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);

- signum：要操作的信号。
- act：要设置的对信号的新处理方式。
- oldact：原来对信号的处理方式。
- 返回值：0 表示成功，-1 表示有错误发生。

 struct sigaction 类型用来描述对信号的处理，定义如下：
 struct sigaction
 {
  void     (*sa_handler)(int);
  void     (*sa_sigaction)(int, siginfo_t *, void *);
  sigset_t  sa_mask;
  int       sa_flags;
  void     (*sa_restorer)(void);
 };


sa_handler 是一个函数指针，其含义与 signal 函数中的信号处理函数类似
sa_sigaction 则是另一个信号处理函数，它有三个参数，可以获得关于信号的更详细的信息。当 sa_flags 成员的值包含了 SA_SIGINFO 标志时，系统将使用 sa_sigaction 函数作为信号处理函数，否则使用 sa_handler 作为信号处理函数。在某些系统中，成员 sa_handler 与 sa_sigaction 被放在联合体中，因此使用时不要同时设置。
sa_mask 成员用来指定在信号处理函数执行期间需要被屏蔽的信号，特别是当某个信号被处理时，它自身会被自动放入进程的信号掩码，因此在信号处理函数执行期间这个信号不会再度发生。
sa_flags 成员用于指定信号处理的行为，它可以是一下值的“按位或”组合。

|参数名|作用|
| :-- | :-- |
| SA_RESTART| 使被信号打断的系统调用自动重新发起|
| SA_NOCLDSTOP| 使父进程在它的子进程暂停或继续运行时不会收到 SIGCHLD 信号|
| SA_NOCLDWAIT| 使父进程在它的子进程退出时不会收到 SIGCHLD 信号，这时子进程如果退出也不会成为僵尸进程|
| SA_NODEFER| 使对信号的屏蔽无效，即在信号处理函数执行期间仍能发出这个信号|
| SA_RESETHAND| 信号处理之后重新设置为默认的处理方式|
| SA_SIGINFO| 使用 sa_sigaction 成员而不是 sa_handler 作为信号处理函数|