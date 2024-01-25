# Lab: traps

# 1. RISC-V assembly ([easy](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

重要的是要理解一些RISC-V汇编语言，你在6.1910（6.004）中接触过它。在你的xv6仓库中有一个文件`user/call.c`。运行`make fs.img`将编译它，并在`user/call.asm`中生成可读的汇编版本。

阅读`call.asm`中的代码，查看函数`g`、`f`和`main`。RISC-V的指令手册在参考页上。以下是一些你应该回答的问题（将答案存储在`answers-traps.txt`文件中）：

### 2 ) 实验步骤

问题1：Which registers contain arguments to functions?  For example, which register holds 13 in main's call to `printf`?  

答案：a0～a7，查看`user/call.asm`第45行：

```
void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
```

得知寄存器`a2`存放13

问题2：Where is the call to function `f` in the assembly code for main? Where is the call to `g`?  

答案：函数`f`与函数`g`直接被编译器优化了

问题3：At what address is the function `printf` located?  

答案：直接搜索“printf”，在`user/call.asm`的1095行：

```
0000000000000642 <printf>:
```

地址为0x0000000000000642

问题4：What value is in the register `ra` just after the `jalr` to `printf` in `main`?  

答案：在RISC-V架构中，`jalr`指令是无条件跳转指令，它会将当前指令的地址保存到寄存器`ra`（即x1），然后跳转到指定地址执行。在`main`函数调用`printf`之后，寄存器`ra`将包含`printf`函数返回后应该继续执行的地址。这个地址是`printf`函数调用之后的下一条指令地址：0x38

问题5：Run the following code.      

```
unsigned int i = 0x00646c72;
printf("H%x Wo%s", 57616, &i);
```

What is the output? 

The output depends on that fact that the RISC-V is little-endian.  If the RISC-V were instead big-endian what would you set `i` to in order to yield the same output? Would you need to change      `57616` to a different value?

答案：运行qemu并执行call，得到结果

```
$ call
HE110 World$ 
```

因为57616的十六进制表示是0xe110，所以printf的结果就是E110；而0x00646c72中，0x64对应d，0x6c对应l，0x72对应r，在数字中的顺序为dlr，但是打印时顺序为rld，所以riscv在存储数据时将低位数字存放在低位，即小端。（小端为“低对低，高对高”）

如果riscv变为大端，需要将先打印的字母（较低的内存地址）放在较高的数值位置，即0x726c6400，打印数字与大端小端无关。

问题6：In the following code, what is going to be printed after      `'y='`?  (note: the answer is not a specific value.)  Why      does this happen?       

```
printf("x=%d y=%d", 3);
```

答案：

执行以下函数，并查看对应汇编代码：

```
void main(void) {
  printf("%d %d\n", f(8)+1, 13);
  printf("x=%d y=%d", 3);
  exit(0);
}
```

结果：

```
$ call
12 13
x=3 y=1$ 
```

`user/call.asm`:

```
void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7d850513          	addi	a0,a0,2008 # 800 <malloc+0xf4>
  30:	00000097          	auipc	ra,0x0
  34:	624080e7          	jalr	1572(ra) # 654 <printf>
  printf("x=%d y=%d", 3);
  38:	458d                	li	a1,3
  3a:	00000517          	auipc	a0,0x0
  3e:	7ce50513          	addi	a0,a0,1998 # 808 <malloc+0xfc>
  42:	00000097          	auipc	ra,0x0
  46:	612080e7          	jalr	1554(ra) # 654 <printf>
  exit(0);
  4a:	4501                	li	a0,0
  4c:	00000097          	auipc	ra,0x0
  50:	28e080e7          	jalr	654(ra) # 2da <exit>
```

可以看到a1寄存器被赋值为3,所以y的值应该为a0保留的值

### 3 ) 实验中遇到的问题和解决办法

问题：汇编没看过代码，指令不熟悉

解决办法：对着实验里给出的reference page找…

### 4 ) 实验心得

感觉在做大一高程作业，小端大端分析，问得很多问题都是riscv指令里的。

## 2. Backtrace ([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

为了进行调试，拥有一个回溯（backtrace）通常非常有用：它是在发生错误的地方之上的堆栈中函数调用的列表。为了帮助进行回溯，编译器会生成与当前调用链中的每个函数相对应的堆栈帧的机器代码。每个堆栈帧包含返回地址和指向调用者堆栈帧的"帧指针"。寄存器s0包含一个指针，指向当前的堆栈帧（实际上它指向堆栈上保存的返回地址的地址加上8）。你的回溯应该使用帧指针来沿着堆栈向上走，并打印每个堆栈帧中保存的返回地址。

### 2 ) 实验步骤

首先在`kernel/defs.h`中增加`backtrace`函数的声明:

```
...
// printf.c
void            printf(char*, ...);
void            panic(char*) __attribute__((noreturn));
void            printfinit(void);
void            backtrace(void);  // add
```

`kernel/riscv.h`增加寻找当前栈指针的函数`r_fp`:

```
#ifndef __ASSEMBLER__

// add
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```

调用栈是用于记录函数调用关系的一种数据结构，每次函数调用时，相关的信息（例如返回地址和调用者的栈指针等）会被压入栈中。通过回溯调用栈，可以查看当前程序执行过程中经过的函数调用链。

在`kernel/printf.c`中添加`backtrace`函数：

```
// add
void
backtrace()
{
  uint64 p = r_fp();  // 获取当前栈指针
  while(p != PGROUNDUP(p))  // 回溯指针应该沿着栈向上走
  {
    printf("%p\n", *(uint64*)(p-8));
    p = *(uint64*)(p-16);  // 跳过栈帧返回地址
  }
}
```

栈指针通常指向当前函数调用的栈帧（Stack Frame）的底部。然后，进入一个循环，直到找到栈指针指向的栈帧为止。使用 `printf()` 打印当前栈帧中保存的返回地址。继续向上移动栈指针，以便找到上一个栈帧的位置。在xv6中，栈帧之间的间隔通常为16字节（8字节用于返回地址，另外8字节用于保存上一个栈帧的指针）。

最后在`kernel/sysproc.c`的`sys_sleep`中插入`backtrace`：

```
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;

  backtrace();  // add
  ...
}
```

在qemu中执行bttest:

```
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ bttest
0x000000008000212e
0x0000000080002020
0x0000000080001d16
```

退出qemu，执行addr2line -e kernel/kernel，输入刚刚的地址：

```
(base) asahi@PC:~/project/xv6/lab4/xv6-labs-2022$ addr2line -e kernel/kernel
0x000000008000212e
0x0000000080002020
0x0000000080/home/asahi/project/xv6/lab4/xv6-labs-2022/kernel/sysproc.c:58
/home/asahi/project/xv6/lab4/xv6-labs-2022/kernel/syscall.c:141
001d16
/home/asahi/project/xv6/lab4/xv6-labs-2022/kernel/trap.c:76
```

按ctrl+D退出

### 3 ) 实验中遇到的问题和解决办法

问题：为什么我不会写？

解决办法：因为前面觉得课比较啰嗦…不论是jyy的课还是lecture…lab看的一头雾水还是区听别人讲，总比自己乱写好。

### 4 ) 实验心得

只能说实在不知道怎么办就去重新看一遍课程…那个栈机制讲的够详细了。

## 3. Alarm ([hard](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这个练习中，你将向xv6添加一个新的功能，它会定期向一个进程发出警报，当进程使用CPU时间时。这对于需要限制CPU时间的计算密集型进程，或者需要在计算的同时进行一些周期性操作的进程可能很有用。更一般地说，你将实现一种用户级中断/故障处理器的原始形式；你可以使用类似的方法来处理应用程序中的页面故障，例如。

你应该添加一个名为`sigalarm(interval, handler)`的新系统调用。如果一个应用程序调用`sigalarm(n, fn)`，那么在程序消耗的每n个"ticks"的CPU时间之后，内核应该调用应用程序中的函数fn。当fn返回后，应用程序应该继续执行在它停下来的地方。在xv6中，一个"tick"是一个相当任意的时间单位，由硬件定时器产生中断的频率决定。如果一个应用程序调用`sigalarm(0, 0)`，内核应该停止生成定期的警报调用。

你会在xv6仓库中找到一个名为`user/alarmtest.c`的文件。将其添加到`Makefile`中。在添加sigalarm和sigreturn系统调用之前，它不会正确编译（请参阅下面的说明）。

`alarmtest`在`test0`中调用`sigalarm(2, periodic)`，要求内核每2个ticks强制调用`periodic()`函数，然后等待一段时间。你可以在`user/alarmtest.asm`中看到`alarmtest`的汇编代码，这对于调试可能很有用。当`alarmtest`产生类似下面的输出，并且`usertests -q`也能正确运行时，你的解决方案是正确的：

### 2 ) 实验步骤

由于要使用sigalarm和sigreturn系统调用，所以在kernel/syscall.c添加声明：

```
...
extern uint64 sys_close(void);
extern uint64 sys_sigalarm(void);  // add
extern uint64 sys_sigreturn(void);  // add

...
[SYS_close]   sys_close,
[SYS_sigalarm] sys_sigalarm,  // add
[SYS_sigreturn] sys_sigreturn,  // add
};
```

在kernel/syscall.h添加系统调用号：

```
// System call numbers
...
#define SYS_close  21
#define SYS_sigalarm 22  // add
#define SYS_sigreturn 23  // add
```

在user/usys.pl中添加跳板函数：

```
...
entry("uptime");
entry("sigalarm");
entry("sigreturn");
```

在user/user.h中添加声明：

```
...
int uptime(void);
int sigalarm(int interval, void (*handler)());  // add
int sigreturn(void);  // add
```

需要保存和恢复寄存器才能正确地恢复中断的代码，所以在kernel/proc.h的struct proc中添加以下参数用来保存现场：

```
// Per-process state
struct proc {
  ...
  char name[16];               // Process name (debugging)
  // add
  int interval;                // 表示每隔多少个 "ticks" 的 CPU 时间，内核会调用一次 handler 函数
  uint64 handler;              // 函数指针，表示用户程序中的处理函数
  struct trapframe save;       // 保存现场
  int passed_ticks;            // 记录中断前的ticks
  int is_return;               // 判断该进程是否返回，防止重复调用
};
```

因为我们在`proc`中增加了属性，所以要在`kernel/proc.c`中`freeproc`和`allocproc`修改，另外，这里不用struct trapframe*是为了简化函数，避免像lab3一样重新分配内存，建立映射，再释放内存删除映射…等等一系列步骤，且没必要，因为不需要将这个参数用copyout传递到用户态。

```
static struct proc*
allocproc(void)
{
  ...
  p->interval = 0;  // add
  p->is_return = 1;  // 进程刚创建时，alarm可以自由使用
  p->handler = 0;

  return p;
}
```

```
static void
freeproc(struct proc *p)
{
  ...

  p->handler = 0;  // add
  p->interval = 0;
  p->is_return = 0;  // 进程已经释放，is_return无意义
}
```

在`kernel/sysproc.c`实现`sys_sigalarm`和`sys_sigreturn`的逻辑：

```
// add
uint64 sys_sigalarm(void){
  int interval;
  uint64 handler;
  argint(0, &interval);
  argaddr(1, &handler);
  struct proc *p = myproc();
  p->interval = interval;
  p->handler = (uint64)handler;
  return 0;
}

uint64 sys_sigreturn(void){
  struct proc *p = myproc();
  p->is_return = 1; // 
  *p->trapframe = p->save; // 恢复中断保存的寄存器
  return p->trapframe->a0;
}
```

在`kernel/trap.c`中`usertrap`函数添加中断逻辑，只有当有定时器中断时，才需要操纵进程的 alarm ticks：

```
void
usertrap(void)
{
 ...

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
  {
    if (p->interval && p->is_return)
    {
      if (++p->passed_ticks == p->interval)  // 当is_return 可用且 passed_ticks 未满足间隔时跳过
      {
        p->save = *p->trapframe;  // 每interval保存现场
        p->trapframe->epc = p->handler;  // 设置处理函数
        p->passed_ticks = 0;  // 清零
        p->is_return = 0;  // 此时并未结束，须等到sigreturn 设置为1
      }
    }
    yield();
  }

  usertrapret();
}
```

最后别忘了在`Makefile`中添加`_alarmtest`：

```
UPROGS=\
	...
	$U/_alarmtest\
```

在qemu中运行`alarmtest` 和`usertest -q`

```
$ alarmtest
test0 start
..............................................alarm!
test0 passed
test1 start
....alarm!
...alarm!
...alarm!
...alarm!
...alarm!
...alarm!
..alarm!
...alarm!
...alarm!
...alarm!
test1 passed
test2 start
.....................................................alarm!
test2 passed
test3 start
alarm!
alarm!
alarm!
alarm!
test3 passed
```

```
$ usertests -q
usertests starting
...
ALL TESTS PASSED
```

### 3 ) 实验中遇到的问题和解决办法

问题1：为什么test3总是不能通过？

解决方案：原来a0寄存器的值还要在sys_sigreturn返回

==Make sure to restore a0.  `sigreturn` is a system call, and its return value is stored in a0.==    

问题2：要不要单独为save分配一个页？

解决方案：在这个实验里不是必要的。

### 4 ) 实验心得

虽然不只保存一个寄存器听起来很吓人，但是实现还蛮简单，直接保存trapframe就好。还有就是==Prevent re-entrant calls to the handler----if a handler hasn't      returned yet, the kernel shouldn't call it again. `test2`      tests this.==

我们希望每次只进入一个alarm，但是这不是多线程，所以只用一个变量标识是否有alarm就可以了。
