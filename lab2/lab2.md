# Lab: system calls

# 1. Using gdb ([easy](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在许多情况下，使用打印语句可能已经足够来调试你的内核，但有时候能够逐步执行一些汇编代码或检查栈上的变量是很有帮助的，学习如何使用gdb

### 2 ) 实验步骤

在第一个 terminal 执行以下指令：

```
make qemu-gdb
```

结果如下：

```
*** Now run 'gdb' in another window.
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::26000
```

在第二个 terminal 执行以下指令：

```
gdb-multiarch
target remote localhost:26000
```

即可进入gdb。

调试方法：

使用file命令进入可执行文件：

```
file kernel/kernel
```

在syscall函数打上断点：

```
b syscall
```

按 c 键continue

问题1：Looking at the backtrace output, which function called `syscall`?  

答案：usertrap

使用layout src命令将terminal分成两个部分，同时查看当前执行的代码以及使用backtrace查看内核栈的情形

```
layout src
backtrace
```

结果如下：

```
(gdb) backtrace
#0 syscall () at kernel/syscall.c:163
#1 0x0000000080001dc6 in usertrap () at kernel/trap.c:67
#2 0x0505050505050505 in ?? ()
```

问题2：What is the value of `p->trapframe->a7` and what does that value represent?  

答案：SYS_exec的索引，也就是7，因为第一个执行的initcode.S将系统调用SYS_exec装入内核

问题3：What was the previous mode that the CPU was in?   

答案：user mode

使用命令`n`来单步执行

使用p /x *p命令以十六进制来查看当前进程的结构

```
p /x *p
```

结果如下：

![image-20230803184339978](/home/asahi/project/xv6/lab2/picture/image-20230803184339978.png)

使用p /x $sstatus命令查看特权寄存器

```
p /x $sstatus
```

结果：

```
$3 = 0x22
```

​			通过寄存器mstatus的值判断SPP位为0,所以从用户态进入内核

问题4：Write down the assembly instruction the kernel is panicing at.    

​			  Which register corresponds to the varialable `num`?  

答案：s2

​			更改语句后的结果如下：

```
xv6 kernel is booting

hart 2 starting
hart 1 starting
scause 0x000000000000000d
sepc=0x00000000800020aa stval=0x0000000000000000
panic: kerneltrap
```

​			在对应的kernel/kernel.asm中找到对应的pc地址：

```
num = * (int *) 0;
    800020aa:	00002903          	lw	s2,0(zero) # 0 <_entry-0x80000000>
  // num = p->trapframe->a7;
```

问题5：Why does the kernel crash? Hint: look at figure 3-3 in the text;    is address 0 mapped in the kernel address space?  Is that    confirmed by the value in `scause` above? 

答案：0地址不能被映射

==When a trap is delegated to S-mode, the scause register is written with the trap cause; the sepc
register is written with the virtual address of the instruction that took the trap; the stval register
is written with an exception-specific datum; the SPP field of mstatus is written with the active
privilege mode at the time of the trap; the SPIE field of mstatus is written with the value of the
SIE field at the time of the trap; and the SIE field of mstatus is cleared. The mcause, mepc, and
mtval registers and the MPP and MPIE fields of mstatus are not written==《riscv-privileged-20211203-1》

| Bit  | Attribute  | Corresponding Exception        |
| ---- | ---------- | ------------------------------ |
| 0    | (See text) | Instruction address misaligned |

输入

```
b *0x00000000800020aa
```

结果如下：

```
Breakpoint 1 at 0x800020aa: file kernel/syscall.c, line 166.
```

在panic前设置断点，并执行

```
layout asm
c
```

结果如下：

![截图 2023-08-03 19-19-17](/home/asahi/project/xv6/lab2/picture/截图 2023-08-03 19-19-17.png)

问题6：What is the name of the binary that was running when the kernel paniced? What is its process id (pid)? 

答案：initcode，1

输入指令

```
p p->name
p p->pid
```

结果如下：

```
$1 = "initcode\000\000\000\000\000\000\000"
$2 = 1
```

### 3 ) 实验中遇到的问题和解决办法

问题：寄存器不知道什么意思

解决办法：老老实实看riscv文档以及gdb debugger文档

### 4 ) 实验心得

没用过gdb调试程序，因为从来没有在机器级的层面debug过，很多时候还得根据scause里的错误码查资料，很头疼

## 2. System call tracing ([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这个任务中，你将添加一个系统调用跟踪功能，它可能在后续的实验中帮助你进行调试。你将创建一个新的trace系统调用来控制跟踪。它应该接受一个参数，一个整数“mask”，其位指定要跟踪的系统调用。例如，要跟踪fork系统调用，一个程序调用trace(1 <<  SYS_fork)，其中SYS_fork是来自kernel/syscall.h的系统调用号。你需要修改xv6内核，在每个系统调用即将返回时打印一行信息，如果该系统调用的编号在mask中设置了，则打印该行信息。该行应包含进程ID、系统调用的名称和返回值；你不需要打印系统调用的参数。trace系统调用应该为调用它的进程以及之后fork的任何子进程启用跟踪，但不应影响其他进程。

### 2 ) 实验步骤

在user/user.h添加新的系统调用：

```
int uptime(void);
int trace(int);  // add
```

在user/usys.pl添加存根：

```
entry("uptime");
entry("trace");  // add
```

在kernel/syscall.h中为新的系统调用分配新的系统调用号：

```
#define SYS_close  21
#define SYS_trace  22  // add
```

在kernel/syscall.c添加外部声明，因为要显示trace系统调用的名字，所以额外添加一个字符串数组：

```
extern uint64 sys_close(void);
extern uint64 sys_trace(void);  // add
...
[SYS_close]   sys_close,
[SYS_trace]   sys_trace,  // add
...
const char *syscall_names[] = {  // add
[SYS_fork]    "fork",
[SYS_exit]    "exit",
[SYS_wait]    "wait",
[SYS_pipe]    "pipe",
[SYS_read]    "read",
[SYS_kill]    "kill",
[SYS_exec]    "exec",
[SYS_fstat]   "fstat",
[SYS_chdir]   "chdir",
[SYS_dup]     "dup",
[SYS_getpid]  "getpid",
[SYS_sbrk]    "sbrk",
[SYS_sleep]   "sleep",
[SYS_uptime]  "uptime",
[SYS_open]    "open",
[SYS_write]   "write",
[SYS_mknod]   "mknod",
[SYS_unlink]  "unlink",
[SYS_link]    "link",
[SYS_mkdir]   "mkdir",
[SYS_close]   "close",
[SYS_trace]   "trace",
};
```

前面的准备完成后，为进程添加trace_for_syscall属性用于跟踪：

```
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  ...
  char name[16];               // Process name (debugging)
  uint64 trace_for_syscall;    // add
};
```

编写sys_trace函数：在kernel/sysproc.c中添加：

```
uint64
sys_trace(void)
{
  int mask;
  argint(0, &mask);  // 通过读取进程的 trapframe，获得 mask 参数
  myproc()->trace_for_syscall = mask;
  return 0;
}
```

创建进程时让trace_for_syscall赋值为0，所以修改kernel/proc.c中的allocproc函数：

```
static struct proc*
allocproc(void)
{
  struct proc *p;
  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  p->trace_for_syscall = 0;  // add
  return 0;

found:
  p->pid = allocpid();
  p->state = USED;

  ...
}
```

父子进程间要传递trace_for_syscall，所以修改kernel/proc.c中的fork函数：

```
int
fork(void)
{
  ...

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);

  // copy trace
  np->trace_for_syscall = p->trace_for_syscall;  // add

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;

  ...
}
```

那么syscall如何使用已经实现的系统调用trace呢？需要修改kernel/syscall.c中的syscall函数：

```
void
syscall(void)
{
  int num;
  struct proc *p = myproc();
  // num = * (int *) 0;
  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    // Use num to lookup the system call function for num, call it,
    // and store its return value in p->trapframe->a0
    p->trapframe->a0 = syscalls[num]();
    if((p->trace_for_syscall >> num) & 1) {  // add
      printf("%d: syscall %s -> %d\n", p->pid, syscall_names[num], p->trapframe->a0);  // add
    }  // pid 系统调用名称 返回值
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

在Makefile中添加_trace：

```
UPROGS=\
	...
	$U/_zombie\
	$U/_trace\
```

使用

```
make GRADEFLAGS=trace grade
```

判断是否成功，实验结果：

```
== Test trace 32 grep == trace 32 grep: OK (1.6s) 
== Test trace all grep == trace all grep: OK (1.0s) 
== Test trace nothing == trace nothing: OK (1.0s) 
== Test trace children == trace children: OK (10.8s) 
```

### 3 ) 实验中遇到的问题和解决办法

问题1：trace是什么？

解决办法：实验给出了提示：

==In the first example above, trace invokes grep tracing just the read system call. The 32 is `1<<SYS_read`. In the second example, trace runs grep while tracing all system calls; the 2147483647 has all 31 low bits set. In the third example, the program isn't traced, so no trace output is printed. In the fourth example, the fork system calls of all the descendants of the `forkforkfork` test in `usertests` are being traced. Your solution is correct if your program behaves as shown above (though the process IDs may be different).==    

一个进程会使用不止一个系统调用，在实现中，我们的系统调用号是从1开始连续声明的，通过1 << SYS_read这样的位运算以及这些数据的并运算，我们可以实现一个整数对应一个或多个系统调用，让进程间传递这个整数，并打印出相关系统调用在哪些进程中发生，这就是trace的作用。

问题2：打印的工作是SYS_trace还是syscall承担？

解决方法：SYS_trace的工作是给出trace，它也不能干扰其他系统调用的执行，但是所有系统调用都会经过syscall函数来实现，所以最好的位置就是在syscall中添加if语句判断。

问题3：SYS_trace到底是怎么调用的？

解决方法：用户态调用的函数提供接口，产生中断，并跳到内核，原先以为根据这个思路我们vscode一路F12就行…后来发现中间的程序使用汇编写的，而且他们的执行过程是统一的(usys.pl)，根据接口的名字，将对应的系统函数调用号传到a7寄存器再ecall，通过syscall函数定位到某个具体的系统调用实现。

### 4 ) 实验心得

更不习惯了，为什么一个系统调用这么复杂，这就是隔离用户态与内核态的代价。

## 3. Sysinfo ([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这个任务中，你将添加一个系统调用`sysinfo`，用于收集有关运行系统的信息。该系统调用接受一个参数：一个指向`struct sysinfo`结构的指针（参见`kernel/sysinfo.h`）。内核应该填充该结构的字段：`freemem`字段应该设置为空闲内存的字节数，`nproc`字段应该设置为状态不是UNUSED的进程数量。我们提供了一个测试程序`sysinfotest`；如果它打印出"sysinfotest: OK"，则表示通过了这个任务。

### 2 ) 实验步骤

在user/user.h添加新的系统调用：

```
struct stat;
struct sysinfo;  // add
...
int uptime(void);
int trace(int);
int sysinfo(struct sysinfo *);  // add
```

在user/usys.pl添加存根：

```
...
entry("uptime");
entry("trace");
entry("sysinfo");  // add
```

在kernel/syscall.h中为新的系统调用分配新的系统调用号：

```
...
#define SYS_close  21
#define SYS_trace  22
#define SYS_sysinfo 23  // add
```

在kernel/syscall.c添加外部声明，并且建立系统调用号与系统调用间的映射：

```
...
extern uint64 sys_close(void);
extern uint64 sys_trace(void);
extern uint64 sys_sysinfo(void);  // add
...
[SYS_close]   "close",
[SYS_trace]   "trace",
[SYS_sysinfo] "sysinfo",  // add
```

完成以上工作后，需要添加函数声明，用于获取当前活动进程数和空闲内存，在kernel/def.h中添加声明：

```
// kalloc.c
void*           kalloc(void);
void            kfree(void *);
void            kinit(void);
uint64          get_free_memory_num(void);  // add
...
int             either_copyout(int user_dst, uint64 dst, void *src, uint64 len);
int             either_copyin(void *dst, int user_src, uint64 src, uint64 len);
void            procdump(void);
int             get_process_num(void);  // add
```

在kernel/kalloc.c实现get_free_memory_num(void)函数：

```
uint64
get_free_memory_num(void)
{
  struct run *r;
  acquire(&(kmem.lock));
  r = kmem.freelist; // pointer to head
  int count = 0;  // calculate pages
  while(r)
  {
    count++;
    r = r->next;
  }
  release(&(kmem.lock));
  return count * PGSIZE;
}
```

在kernel/proc.c实现get_process_num(void)函数：

```
int
get_process_num(void)
{
  struct proc *p = proc;
  int count = 0;

  while(p < proc + NPROC)
  {
    acquire(&(p->lock));
    if(p->state != UNUSED)
    {
      count++;
    }
    release(&(p->lock));
      p++;
  }
  return count;
}
```

在kernel/sysproc.c中实现sys_sysinfo函数：

```
uint64
sys_sysinfo(void)
{
  uint64 addr;
  argaddr(0, &addr);
  struct sysinfo info;
  info.freemem = get_free_memory_num();
  info.nproc = get_process_num();

  if(copyout(myproc() -> pagetable, addr, (char *)&info, sizeof(info)) < 0)
    return -1;
  return 0;
}
```

在Makefile中添加_sysinfotest：

```
UPROGS=\
	...
	$U/_trace\
	$U/_sysinfotest\
```

使用

```
make GRADEFLAGS=sysinfotest grade
```

判断是否成功，实验结果：

```
== Test sysinfotest == sysinfotest: OK (3.7s) 
```

### 3 ) 实验中遇到的问题和解决办法

问题1：内存怎么组织起来的，进程又是怎么组织起来的，怎么看空闲内存，怎么找UNUSED进程？

解决方法：看kalloc源码，看proc源码，发现内存是拿链表写的，进程是拿数组存的，灵活运用ctrl+F和F12快速定位…以前没有接触这么多代码的project，不过这一部分没有汇编会舒服很多。

问题2：copyout是什么？

解决方法：用来传一块内存上的值的函数，因为内核态与用户态指针不互通，需要一个函数传递值。你要把内核态的结构体数据传到用户态。

### 4 ) 实验心得

hint叫我看来看去，就是想让我多读源码。
