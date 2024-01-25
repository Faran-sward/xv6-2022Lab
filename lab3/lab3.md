# Lab: page tables

# 1. Speed up system calls ([easy](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在某些操作系统（例如Linux）中，通过在用户空间和内核之间共享只读区域中的数据，加快了某些系统调用的速度。这样可以在执行这些系统调用时避免内核交叉。为了帮助你学习如何在页表中插入映射，你的第一个任务是为`xv6`实现在`getpid()`系统调用中进行这种优化。

简而言之，你需要修改`xv6`的实现，使得当进程调用`getpid()`系统调用时，它能够直接访问内核中保存进程`ID`的只读数据，而无需切换到内核模式。这样可以避免额外的开销和切换，并提高`getpid()`系统调用的执行效率。

### 2 ) 实验步骤

查看位于`kernel/memlayout.h`的`usyscall`结构体：

```
struct usyscall {
  int pid;  // Process ID
};
```

当进程创建，创建页表时分配一个新的页存储进程的pid，所以为`kernel/proc.h`中的`struct proc`添加一个指针：

```
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  ...
  char name[16];               // Process name (debugging)
  struct usyscall *uscall;     // add
};
```

修改`kernel/proc.c`中创建进程的函数`allocproc`，分配一个物理页，用于存放进程的`usyscall`结构体：

```
  ...
  // Allocate a trapframe page.
  if ((p->trapframe = (struct trapframe *)kalloc()) == 0)
  {
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // add: allocate a page for usyacall
  if ((p->uscall = (struct usyscall *)kalloc()) == 0)
  {
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  
  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if (p->pagetable == 0)
  {
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  // add
  p->uscall->pid = p->pid;

  return p;
}
```

注意：这里写的是先分配`trapframe`，再分配`usyscall`，再为进程创建页表，需要注意这里分配的顺序。进程创建好页表后就可以写入值了，将`uscall`结构体写入当前创建进程的pid。

创建页表也要修改，因为每添加一个页，就要进行虚拟地址和物理地址的映射，修改`kernel/proc.c`中的`proc_pagetable`函数：

```
// Create a user page table for a given process, with no user memory,
// but with trampoline and trapframe pages.
pagetable_t
proc_pagetable(struct proc *p) // 页表创建函数
{
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if (pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if (mappages(pagetable, TRAMPOLINE, PGSIZE,
               (uint64)trampoline, PTE_R | PTE_X) < 0)
  {
    uvmfree(pagetable, 0);
    return 0;
  }

  // map the trapframe page just below the trampoline page, for
  // trampoline.S.
  if (mappages(pagetable, TRAPFRAME, PGSIZE,
               (uint64)(p->trapframe), PTE_R | PTE_W) < 0)
  {
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  if (mappages(pagetable, USYSCALL, PGSIZE,
               (uint64)(p->uscall), PTE_R | PTE_U) < 0)
  {
    uvmunmap(pagetable, TRAMPOLINE, 1, 0); // 这一步如果创建失败
    uvmunmap(pagetable, TRAPFRAME, 1, 0);  // 需要删除之前的映射
    uvmfree(pagetable, 0);
    return 0;
  }

  return pagetable;
}
```

注意：这里页表映射的逻辑，首先为页表本身分配内存，出错返回0；为`TRAMPOLINE`创建映射，如果失败，则需要释放页表本身分配内存，我们新添加的`USYSCALL`也是如此，如果失败，就要把之前已经申请的内存全部释放，并且要把之前已经完成的映射全部删除。

最后，当进程结束时，需要把多申请的页释放：修改`kernel/proc.c`中的`freeproc`函数：

```
// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  if (p->trapframe)
    kfree((void *)p->trapframe);
  p->trapframe = 0;
  // add
  if (p->uscall)
    kfree((void *)p->uscall);  // add
  p->uscall = 0;
  if (p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```

按照之前的逻辑，先释放页面，再释放页表。

进程结束后释放页表前还要解除映射：修改`kernel/proc.c`中的`proc_freepagetable`函数：

```
// Free a process's page table, and free the
// physical memory it refers to.
void proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmunmap(pagetable, USYSCALL, 1, 0);  // add
  uvmfree(pagetable, sz);
}
```

### 3 ) 实验中遇到的问题和解决办法

问题：为什么总是报错？

解决方法：管理内存就要滴水不漏，映射也占内存，也当内存管理。

### 4 ) 实验心得

这次修改的地方都在进程，比较方便，不像syscall，但是跑起来不那么顺利，总是发现加的几行函数搞错位置了，不过认真看两眼就看出问题了。

## 2. Print a page table ([easy](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

为了帮助你可视化`RISC-V`的页表，并可能在以后的调试中提供帮助，你的第二个任务是编写一个函数来打印页表的内容。

定义一个名为`vmprint()`的函数。它应该接受一个`pagetable_t`类型的参数，并以下面描述的格式打印该页表的内容。在`exec.c`中，在`return argc`之前插入`if(p->pid==1) vmprint(p->pagetable)`，以打印第一个进程的页表。如果通过`make grade`的打印测试，你的实现将得到全部的分数。

### 2 ) 实验步骤

主要是实现`vmprint`：首先`kernel/defs.h`添加声明：

```
...
// vm.c
...
int             copyinstr(pagetable_t, char *, uint64, uint64);
void            vmprint(pagetable_t pagetable);  // add
```

`kernel/vm.c`编写`vmprint`函数：

```
...
// add
void vmprint(pagetable_t pagetable)
{
  static int num = 0;
  if(num == 0)
    printf("page table %p\n", pagetable);
  for(int i = 0; i < PGSIZE / sizeof(pagetable_t); i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V){
      for(int j = 0; j <= num; j++)
        printf("..");
      pagetable_t child = (pagetable_t)PTE2PA(pte);
      printf("%d: pte %p pa %p\n", i, pte, child);
      if (num != 2){
        ++num;
        vmprint(child);
        --num;
      }
    }
  }
}
```

### 3 ) 实验中遇到的问题和解决办法

问题：页表，页到底是怎么实现的？

解决方法：`F12`查看`kernel/riscv.h`中的定义：

```
typedef uint64 pte_t;
typedef uint64 *pagetable_t; // 512 PTEs
```

说白了页表就uint64数组，页表项就uint64单元，不过存储的是下一级页表的位置。

还有类型间的转换：

```
#define PA2PTE(pa) ((((uint64)pa) >> 12) << 10)

#define PTE2PA(pte) (((pte) >> 10) << 12)

#define PTE_FLAGS(pte) ((pte) & 0x3FF)
```

这个与xv6的三级页表演示图是一致的。

### 4 ) 实验心得

函数就遍历一个高度为三的页表树，`dfs`的方式搜索，弄清楚具体类型就轻松了。

## 3. Detect which pages have been accessed ([hard](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这部分的实验中，你将向`xv6`添加一个新功能，通过检查`RISC-V`页表中的访问位，检测并向用户空间报告哪些页面已被访问（读取或写入）。当`RISC-V`硬件页表访问器解析`TLB`缺失时，它会在PTE中标记这些位。

你的任务是实现一个名为`pgaccess()`的系统调用，用于报告哪些页面已被访问。该系统调用需要三个参数。首先，它需要第一个用户页面的起始虚拟地址来检查。其次，它需要要检查的页面数量。最后，它需要一个用户地址指向一个缓冲区，用于将结果存储为位掩码（一个数据结构，每个页面使用一个位，其中第一个页面对应于最低有效位）。如果在运行`pgtbltest`时，`pgaccess()`测试用例通过，你将得到全部的分数。

### 2 ) 实验步骤

给页表Flags中添加一位用于标记页面是否被访问：`PTE_A`,在`kernel/riscv.h`中添加相应的宏定义：

```
...
#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // user can access
#define PTE_A (1L << 6) // add: have been accessed
```

弄清楚`kernel/vm.c`中的`walk`函数:

```
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");  // 虚拟地址不允许超过最大虚拟地址

  for(int level = 2; level > 0; level--) {
    // 计算当前级别的页表项（PTE）在页表中的索引，得到对应的 PTE 
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {  // PTE_V标志位是否被设置
    // 更新为下一级的页表的物理地址
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
```

`walk`根据给定的虚拟地址，从顶层的页表开始，逐级遍历页表树，找到对应的页表项（`PTE`）。如果页表项存在，函数返回该`PTE`的指针，从而找到对应的物理地址。如果页表项不存在（即`PTE_V`标志位未设置），根据 `alloc` 参数的值，可能会分配新的页表，并将相应的`PTE`项设置为新的页表的地址。

实现系统调用`sys_pgaccess`：

```
#ifdef LAB_PGTBL
int
sys_pgaccess(void)
{
  // lab pgtbl: your code here.
  uint64 address, mask_address;
  argaddr(0, &address);
  argaddr(2, &mask_address);
  int len;
  argint(1, &len);
  if(len > 32)
    return -1;
  struct proc* proc = myproc();
  uint32 mask = 0;
  for(int i = 0; i < len; i++){
    pte_t* pte = walk(proc->pagetable, address + i * PGSIZE, 0);
    if(*pte & PTE_A){
      mask |= 1 << i;  // 将当前虚拟地址对应的位设置为1
      *pte &= ~PTE_A;  // 重置当前页表项的页面访问位
    }
  }
  if(copyout(proc->pagetable, mask_address, (char*)&mask, 4) < 0)
    return -1;
  return 0;
}
```

使用

```
make grade
```

查看以上实验结果：

```
== Test pgtbltest == 
$ make qemu-gdb
(2.3s) 
== Test   pgtbltest: ugetpid == 
  pgtbltest: ugetpid: OK 
== Test   pgtbltest: pgaccess == 
  pgtbltest: pgaccess: OK 
== Test pte printout == 
$ make qemu-gdb
pte printout: OK (1.0s) 
== Test answers-pgtbl.txt == answers-pgtbl.txt: OK 
== Test usertests == 
$ make qemu-gdb
(52.0s) 
== Test   usertests: all tests == 
  usertests: all tests: OK 
```

### 3 ) 实验中遇到的问题和解决办法

问题：为什么把`PTE_A`设置为第6位而不是第7位就会报错？

解决方法：因为页访问机制会设置第七位为访问位，我们的实现仅仅是利用了这一点，不能随意改动。

### 4 ) 实验心得

出了一个很简单的错误，实现系统调用时把命令行参数位置搞错了，导致len一直报错。整个实现就是遍历进程的页表，通过walk函数获取物理页并查看对应位是否为1。
