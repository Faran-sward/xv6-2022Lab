

# Lab: Multithreading

# 1. Uthread: switching between threads ([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这个练习中，您将设计一个用户级线程系统的上下文切换机制，并进行实现。为了帮助您开始，xv6中提供了两个文件`user/uthread.c`和`user/uthread_switch.S`，以及在`Makefile`中添加了一个规则来构建uthread程序。`uthread.c`包含了大部分用户级线程包的代码，以及三个简单测试线程的代码。线程包缺少一些创建线程和在线程之间进行切换的代码。您需要补充这些代码来完成用户级线程系统的实现。您的任务是制定一个计划来创建线程，并保存/恢复寄存器以在线程之间进行切换，并实现该计划。

您需要在`user/uthread.c`中的`thread_create()`和`thread_schedule()`函数中添加代码，并在`user/uthread_switch.S`中添加`thread_switch`函数。一个目标是确保当`thread_schedule()`第一次运行给定的线程时，该线程在其自己的堆栈上执行传递给`thread_create()`的函数。另一个目标是确保`thread_switch`保存要切换出的线程的寄存器状态，恢复要切换入的线程的寄存器状态，并返回到后者线程上次执行的指令点。您需要决定在哪里保存/恢复寄存器；修改`struct  thread`以保存寄存器是一个不错的计划。在`thread_schedule`中需要添加一个对`thread_switch`的调用；您可以传递`thread_switch`所需的任何参数，但意图是从线程t切换到`next_thread`。

### 2 ) 实验步骤

`kernel/proc.h`中已经声明了线程切换需要保存的上下文`context`，但是我们需要在用户态切换线程，所以在`user/uthread.c`中重新声明一遍`ucontext`：

```
// add
struct ucontext {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};  // add:end
```

然后我们在`user/uthread.c`中的`struct thread`中加上一个成员，用于保存线程切换时的上下文：

```
struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct ucontext context;      // add
};
```

接下来修改`user/uthread.c`中的`thread_create`函数：创建新线程的实质就是在线程序列中找到待使用的单元并设置其sp和ra寄存器。

```
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }  // 找到可用的线程
  t->state = RUNNABLE;
  // YOUR CODE HERE: add
  t->context.sp = (uint64)(t->stack) + STACK_SIZE;
  t->context.ra = (uint64)func;
  // 创建线程后就要在自己的栈中执行指令，将sp设为栈顶
  // 用ra保存func的返回地址
  // add:end
}
```

接下来修改`user/uthread_switch.S`中的`thread_switch`函数：可以直接借鉴内核线程的上下文切换：位于`kernel/swtch.S`：

```
	.text

	/*
         * save the old thread's registers,
		 * void swtch(struct context *old, struct context *new);
         * restore the new thread's registers.
         */

	.globl thread_switch
thread_switch:
	/* YOUR CODE HERE */
	        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
	ret    /* return to ra */

```

最后修改`user/uthread.c`中的`thraed_shcedule`函数：按照提示只需要加一句`thread_switch`调用即可：

```
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->context, (uint64)&current_thread->context);  // add
```

使用

```
make GRADEFLAGS=uthread grade
```

查看结果：

```
== Test uthread == uthread: OK (2.1s) 
```

### 3 ) 实验中遇到的问题和解决办法

问题：gdb调试好麻烦

解决方法：按照提示

```
file user/_uthread
b uthread.c:80  // 我修改了原文件导致对应行数变化
c
```

在第一个gdb中执行uthread：第二个gdb结果

```
Thread 1 hit Breakpoint 1, thread_schedule () at user/uthread.c:80
80	    current_thread = next_thread;
```

使用

```
p/x *next_thread
```

以十六进制print变量next_thread，结果：

```
(gdb) p/x *next_thread
Cannot access memory at address 0x4f1f
```

命令 "x/x" 用于查看内存中指定地址的内容，并以十六进制形式显示，使用

```
x/x next_thread->stack
```

访问stack成员内容，结果：

```
(gdb) x/x next_thread->stack
0x4f1f <all_thread+16759>:	0x0000000a
```

使用

```
b thread_switch
c
```

跳转到thread_switch，接下来不停摁c就能查看线程一步一步schedule的情形了。

### 4 ) 实验心得

gdb还不熟练，我调个gdb花半天还看不明白结果，还不如直接printf快。

# 2. Using threads  ([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这个作业中，你将使用线程和锁来探索并行编程，并实现一个哈希表。你应该在一台拥有多个核心的真实 Linux 或 MacOS 计算机上完成这个作业（不是在 xv6 或 qemu 上）。大多数最新的笔记本电脑都有多核处理器。

该作业使用 UNIX 的 pthread 线程库。文件 `"notxv6/ph.c" `包含一个简单的哈希表，在单线程中使用时是正确的，但在多线程中使用时是不正确的。请注意，为了构建 ph，Makefile 使用的是你操作系统的 gcc，而不是 6.S081 工具。ph 的参数指定了在哈希表上执行 put 和 get 操作的线程数。请查看 notxv6/ph.c 文件，特别是 put() 和 insert() 函数的实现。

为了避免上述事件序列，需要在 notxv6/ph.c 中的 put 和 get 函数中插入 lock 和 unlock 语句，以确保在两个线程中键的丢失数量始终为0。使用 pthread 库来管理线程锁，相关的函数调用如下：

```
pthread_mutex_t lock;            // 声明一个锁
pthread_mutex_init(&lock, NULL); // 初始化锁
pthread_mutex_lock(&lock);       // 获取锁
pthread_mutex_unlock(&lock);     // 释放锁
```

在某些情况下，同时进行的 put() 操作在哈希表中读写的内存没有重叠，因此它们不需要锁来保护彼此。你可以改变 ph.c 的实现以利用这种情况，从而对一些 put() 操作获得并行加速。

### 2 ) 实验步骤

临界资源就是`NBUCKET`个哈希桶，所以在`notxv6/ph.c`中配上对应的互斥锁：

```
pthread_mutex_t locks[NBUCKET];  // add
```

并且在`notxv6/ph.c`中的main函数中，用`pthread_mutex_init`初始化：

```
int
main(int argc, char *argv[])
{
  pthread_t *tha;
  void *value;
  double t1, t0;

  for(int i = 0; i < NBUCKET; i++)  // add: 互斥锁初始化
    pthread_mutex_init(&locks[i], NULL);  // add:end
    ...
```

给写临界区资源的函数put上锁：

```
static 
void put(int key, int value)
{
  int i = key % NBUCKET;
  pthread_mutex_lock(&locks[i]);  // add:lock
  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&locks[i]);  // add:unlock
}
```

使用

```
make ph
./ ph 1
./ ph 2
```

观察结果：

```
(base) asahi@PC:~/project/xv6/lab6/xv6-labs-2022$ ./ph 1
100000 puts, 8.446 seconds, 11840 puts/second
0: 0 keys missing
100000 gets, 8.479 seconds, 11793 gets/second
(base) asahi@PC:~/project/xv6/lab6/xv6-labs-2022$ ./ph 2
100000 puts, 3.571 seconds, 28002 puts/second
0: 0 keys missing
1: 0 keys missing
200000 gets, 8.712 seconds, 22957 gets/second
```

### 3 ) 实验中遇到的问题和解决办法

问题：Why are there missing keys with 2 threads, but not with 1 thread?  Identify a sequence of events with 2 threads that can lead to a key being missing. 

解决方法：考虑以下顺序：

1. 线程 1 和线程 2 同时执行 put() 操作，它们读取哈希表中的散列桶索引。
2. 线程 1 先执行并找到要插入的散列桶，然后暂停执行。
3. 线程 2 也找到了同一个散列桶，并执行了插入操作。
4. 线程 1 继续执行，并在相同的散列桶上执行插入操作，覆盖了线程 2 插入的内容。

因此，线程 2 插入的键被线程 1 覆盖，导致这些键在哈希表中丢失。

### 4 ) 实验心得

觉得目前以来最简单的实验，太舒服了。

# 3. Barrier([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这个任务中，你将实现一个屏障（barrier）：在应用程序中的一个点，所有参与的线程必须等待，直到所有其他参与的线程也到达该点。你将使用 pthread 条件变量，这是一种类似于 xv6 的 sleep 和 wakeup 的序列协调技术。你应该在一台真实的计算机上完成这个任务（不是在 xv6 或 qemu 上）。文件 `notxv6/barrier.c` 包含一个错误的barrior。

每个线程执行一个循环，在每次循环迭代中，线程调用  barrier()，然后休眠一段随机的微秒时间。期望的行为是，每个线程都在 barrier() 中阻塞，直到所有 nthread 个线程都调用了 barrier()。你的目标是实现所需的屏障行为。除了你在 ph assignment 中见过的锁原语之外，你还需要以下新的 pthread 原语：

```
pthread_cond_wait(&cond, &mutex); // 在 cond 上进入睡眠状态，释放 mutex 锁，在唤醒时重新获取锁
pthread_cond_broadcast(&cond); // 唤醒在 cond 上睡眠的所有线程
```

确保你的解决方案通过 `make grade` 的 barrier 测试。你必须处理一系列的 barrier 调用，我们将每个称为一个 `round`。`bstate.round` 记录了当前的 `round`。每当所有线程都到达屏障时，你应该增加 `bstate.round `的值。
你必须处理这样一种情况：一个线程在其他线程退出屏障之前先绕过了循环。特别是，你要重用 `bstate.nthread `变量从一个` round `到下一个 `round`。确保当一个线程离开屏障并绕过循环时，它不会在之前的 `round `还在使用 `bstate.nthread `时增加它的值。

### 2 ) 实验步骤

每个进程访问`barrior`函数都会改变`nthread`（当前已就绪的线程数）或`round`（当前轮数）中的一个，所以访问该函数要上锁，然后根据nthread判断即可

```
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&(bstate.barrier_mutex));  // add
  if(++bstate.nthread < nthread)  // 又一个进程进入barrier 且 并非所有进程都进入barrior
    pthread_cond_wait(&(bstate.barrier_cond), &(bstate.barrier_mutex));  // 睡眠该进程
  else{
    bstate.round++;
    bstate.nthread = 0;
    pthread_cond_broadcast(&(bstate.barrier_cond));  // 唤醒所有线程
  }
  pthread_mutex_unlock(&(bstate.barrier_mutex));  // add:end
  
}
```

使用

```
make barrier
./barrier 2
```

观察结果：

```
(base) asahi@PC:~/project/xv6/lab6/xv6-labs-2022$ make barrier
gcc -o barrier -g -O2 -DSOL_THREAD -DLAB_THREAD notxv6/barrier.c -pthread
(base) asahi@PC:~/project/xv6/lab6/xv6-labs-2022$ ./barrier 2
OK; passed
```

最后

```
make grade
```

检查所有实验结果：

```
== Test uthread == 
$ make qemu-gdb
uthread: OK (2.2s) 
== Test answers-thread.txt == answers-thread.txt: OK 
== Test ph_safe == make[1]: 进入目录“/home/asahi/project/xv6/lab6/xv6-labs-2022”
gcc -o ph -g -O2 -DSOL_THREAD -DLAB_THREAD notxv6/ph.c -pthread
make[1]: 离开目录“/home/asahi/project/xv6/lab6/xv6-labs-2022”
ph_safe: OK (9.6s) 
== Test ph_fast == make[1]: 进入目录“/home/asahi/project/xv6/lab6/xv6-labs-2022”
make[1]: “ph”已是最新。
make[1]: 离开目录“/home/asahi/project/xv6/lab6/xv6-labs-2022”
ph_fast: OK (21.4s) 
== Test barrier == make[1]: 进入目录“/home/asahi/project/xv6/lab6/xv6-labs-2022”
gcc -o barrier -g -O2 -DSOL_THREAD -DLAB_THREAD notxv6/barrier.c -pthread
make[1]: 离开目录“/home/asahi/project/xv6/lab6/xv6-labs-2022”
barrier: OK (2.8s) 
```

### 3 ) 实验中遇到的问题和解决办法

问题：没用过make编译程序（除了make qemu）

解决办法：hint给了如何编译成可执行文件

### 4 ) 实验心得

一旦远离xv6难度就降了不少，比电梯算法好多了
