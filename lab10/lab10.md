# Lab: mmap ([hard](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

`mmap` 和 `munmap` 系统调用允许 UNIX 程序对其地址空间进行详细控制。它们可用于在进程之间共享内存，将文件映射到进程地址空间中，以及作为用户级页错误方案的一部分，例如课程中讨论的垃圾回收算法。在这个实验中，你将向 xv6 添加 `mmap` 和 `munmap` 系统调用，重点是内存映射文件。

`mmap` 可以以多种方式调用，但这个实验仅要求实现与文件内存映射相关的一部分功能。你可以假设 `addr` 始终为零，这意味着内核应该决定在哪个虚拟地址映射文件。`mmap` 返回该地址，如果失败则返回 0xffffffffffffffff。`length` 是要映射的字节数；它可能与文件的长度不同。`prot` 表示内存是否应该映射为可读、可写和/或可执行；你可以假设 `prot` 是 `PROT_READ` 或 `PROT_WRITE` 或两者都有。`flags` 将是 `MAP_SHARED`，表示对映射内存的修改应该写回文件，或者是 `MAP_PRIVATE`，表示它们不应该写回文件。你不需要实现 `flags` 中的其他位。`fd` 是要映射的文件的已打开文件描述符。你可以假设 `offset` 是零（它是要映射的文件中的起始点）。

如果映射了相同的 `MAP_SHARED` 文件的进程不共享物理页面，那是可以的。

`munmap(addr, length)` 应该移除指定地址范围内的 `mmap` 映射。如果进程修改了内存并将其映射为 `MAP_SHARED`，则修改应该首先写回文件。`munmap` 调用可能仅覆盖 `mmap` 区域的一部分，但你可以假设它将映射区域的开始、结尾或整个区域（但不是在区域中间创建一个空洞）。

你应该实现足够的 `mmap` 和 `munmap` 功能，以使 `mmaptest` 测试程序正常工作。

hints：

1. 首先，将 `_mmaptest` 添加到 `UPROGS` 中，并添加 `mmap` 和 `munmap` 系统调用，以使得 `user/mmaptest.c` 可以编译。暂时只返回 `mmap` 和 `munmap` 的错误。我们已经在 `kernel/fcntl.h` 中为你定义了 `PROT_READ` 等常量。运行 `mmaptest`，它会在第一个 `mmap` 调用失败。
2. 惰性地填充页表，以响应页面错误。也就是说，在 `mmap` 中不要分配物理内存或读取文件。而是在 `usertrap` 中的页面错误处理代码中（或者由其调用），类似于懒惰页面分配实验。惰性的原因是确保大文件的 `mmap` 是快速的，并且可以映射大于物理内存的文件。
3. 跟踪每个进程的 `mmap` 映射情况。定义一个与在第 15 讲中描述的 VMA（虚拟内存区域）相对应的结构，记录由 `mmap` 创建的虚拟内存范围的地址、长度、权限、文件等。由于 xv6 内核在内核中没有内存分配器，所以可以声明一个固定大小的 VMA 数组，并根据需要从该数组中分配。大小为 16 应该足够。
4. 实现 `mmap`：在进程的地址空间中找到未使用的区域，将文件映射到该区域，并将一个 VMA 添加到进程的映射区域表中。VMA 应该包含一个指向被映射文件的 `struct file` 的指针；`mmap` 应该增加文件的引用计数，以防止在文件关闭时结构体消失（提示：参考 `filedup`）。运行 `mmaptest`：第一个 `mmap` 应该成功，但对 `mmap` 内存的第一个访问将导致页面错误并终止 `mmaptest`。
5. 添加代码以在 `mmap` 区域内导致页面错误，以分配一页物理内存，将相关文件的 4096 字节读入该页，并将其映射到用户地址空间。使用 `readi` 读取文件，该函数带有一个要在文件中读取的偏移量参数（但你将不得不在传递给 `readi` 的 `inode` 上锁定/解锁）。不要忘记在页面上正确设置权限。运行 `mmaptest`；它应该可以到达第一个 `munmap`。
6. 实现 `munmap`：找到地址范围的 VMA，并取消映射指定的页面（提示：使用 `uvmunmap`）。如果 `munmap` 移除了先前 `mmap` 的所有页面，它应该减少相应的 `struct file` 的引用计数。如果已解除映射的页面已被修改，并且文件被映射为 `MAP_SHARED`，则将页面写回文件。参考 `filewrite` 进行灵感。
7. 理想情况下，你的实现只会写回程序实际修改过的 `MAP_SHARED` 页面。RISC-V PTE 中的 "dirty" 位（D 位）指示页面是否被写入。然而，`mmaptest` 不会检查未修改的页面是否被写回；因此，你可以在不查看 D 位的情况下将页面写回。
8. 修改 `exit`，使其解除进程的映射区域，就像调用了 `munmap` 一样。运行 `mmaptest`；`mmap_test` 应该通过，但 `fork_test` 可能不会通过。
9. 修改 `fork`，确保子进程具有与父进程相同的映射区域。不要忘记增加 VMA 的 `struct file` 的引用计数。在子进程的页面错误处理程序中，分配新的物理页而不是与父进程共享一个页面是可以的。后者可能更高级，但需要更多的实现工作。运行 `mmaptest`；它应该通过 `mmap_test` 和 `fork_test`。

### 2 ) 实验步骤

按照hints，在Makefile中添加_mmaptest:

```
UPROGS=\
	...
	$U/_mmaptest\
```

老样子，系统调用四件套：user.h，usys.pl，syscall.c，syscall.h：

在kernel/syscall.h中添加mmap和munmap的系统调用号：

```
#define SYS_mmap   22
#define SYS_munmap 23
```

在kernel/syscall.c中添加mmap和munmap的声明和映射：

```
...
extern uint64 sys_mmap(void);  // add
extern uint64 sys_munmap(void);  // add
...
[SYS_mmap]    sys_mmap,  // add
[SYS_mummap]  sys_munmap,  // add
...
```

在user/usys.pl中添加mmap和munmap的跳板函数：

```
...
entry("mmap");
entry("munmap");
```

在user/user.h中添加mmap和munmap的用户函数声明：

```
...
void *mmap(void *addr, int length, int prot, int flags,
           int fd, uint32 offset);  // add
int munmap(void* addr, int length);  // add
...
```

实验让我们自行定义vma，在kernel/proc.h中加入关于vma的定义，包含虚拟内存映射的字节数，地址，权限，文件偏移，是否已经映射，文件指针，vma还需要标识位：还要在进程中添加vma结构体数组vmas：

```
struct vma{  // add
  uint64 addr;
  int mapped;
  int len;
  int prot;
  int flags;
  int offset;
  struct file* f;
};  // add:end
#define NVMA 16  // add

// Per-process state
struct proc {
  ...
  struct vma vmas[NVMA];         // add
};
```

在`kernel/sysfile.c`编写`sys_mmap`函数，解决惰性填充页表，只需扩大`size`，在`pagefault`时再处理：

```
uint64 // add
sys_mmap(void)
{
  uint64 addr;
  int len, prot, flags, offset;
  struct file* f;
  struct proc* p;

  argaddr(0, &addr);
  argint(1, &len);
  argint(2, &prot);
  argint(3, &flags);
  argint(5, &offset);

  if(argfd(4, 0, &f) < 0)
    return -1;

  if (flags != MAP_SHARED && flags != MAP_PRIVATE)
    return -1;

  if(f->writable == 0 && (prot & PROT_WRITE) && (flags == MAP_SHARED))
    return -1;  // 只读文件不进行写映射

  if (len < 0 || offset < 0 || offset % PGSIZE)
    return -1;

  len = PGROUNDUP(len);  // 页对齐

  for(int i = 0; i < NVMA; i++){
    p = myproc();
    if(!p->vmas[i].mapped){
      p->vmas[i].addr = addr? addr: p->sz;  // 如果addr为0则默认使用sz
      p->vmas[i].offset = offset;
      p->vmas[i].mapped = 1;
      p->vmas[i].len = len;
      p->vmas[i].prot = prot;
      p->vmas[i].flags = flags;
      p->vmas[i].f = f;
      filedup(f);  // 增加文件引用计数
      p->sz += len;
      return p->vmas[i].addr;
    }
  }
  return -1;  // vma消耗尽

}  // add:end
```

这里再补充一个函数，通过虚拟地址找到进程中对应的`vma`：在`kernel/proc.c`中添加，并在`defs.h`中声明：

```
struct vma*  // add
findvma(uint64 addr)
{
  struct proc *p = myproc();
  int i;
  for(i=0;i<16;i++)
  {
    if(p->vmas[i].mapped){
      uint64 a=p->vmas[i].addr;
      uint64 b=p->vmas[i].addr + p->vmas[i].len;
      if(addr>=a && addr<b){
        return &p->vmas[i];
      } 
    }
  }
  return 0;
}  // add:end
```

在`kernel/trap.c`的`usertrap`中添加处理`pagefault`（包括读和写错误）的判断：

```
else if(r_scause() == 13 || r_scause() == 15) { // add
    uint64 va = r_stval();
    struct vma* vma = 0;

    if(va >= p->sz || va <= p->trapframe->sp) /** va 超出范围*/
      setkilled(p);
    
    vma = findvma(va);

    if(!vma)
      setkilled(p);

    /** 在 vm 中找到了缺页的文件对象 */
    va = PGROUNDDOWN(va);
    
    /** 尝试为文件对象的 vm 分配内存，用来容乃新的内容 */
    char* mem = kalloc();
    if(mem == 0)
      setkilled(p);
    
    memset(mem, 0, PGSIZE);
    /** 将存储在 disk 中的文件对象的新内容拷贝到 vm */
    vcp(vma, mem, va);

    /** 根据 prot 设置 PTE 权限 */
    int flags = PTE_U;
    if(vma->prot & PROT_READ) 
      flags |= PTE_R;
    if(vma->prot & PROT_WRITE)
      flags |= PTE_W;
    if(vma->prot & PROT_EXEC)
      flags |= PTE_X;
    
    if(mappages(p->pagetable, va, PGSIZE, (uint64)mem, flags) != 0)
      kfree(mem);
  } // add:end
```

这里使用了一个新的函数`vcp`用于拷贝，在`kernel/fs.c`中实现同时还要在`kernel/defs.h`中声明：

```
void  // add
vcp(struct vma* vma, char* mem, uint64 va)
{
  ilock(vma->f->ip);
  readi(vma->f->ip, 0, (uint64)mem, va-vma->addr + vma->offset, PGSIZE);
  iunlock(vma->f->ip);
}  // add:end
```

补上刚刚`findvma`函数的声明：

```
...
// fs.c
...
void            vcp(struct vma* vma, char* mem, uint64 va); // add
// proc.c
...
struct vma*     findvma(uint64 addr);  // add
...
```

在 `kernel/sysfile.c `中实现系统调用 `sys_munmap()`：

```
uint64  // add
sys_munmap(void)
{
  uint64 addr;
  argaddr(0, &addr);
  int len;
  argint(1, &len);
  
  addr = PGROUNDDOWN(addr);
  len = PGROUNDUP(len);
  struct vma* v = findvma(addr);

  if(!v || addr != v->addr)
    return -1;

  v->addr += len;
  v->len -= len;
  if(v->flags & MAP_SHARED)
    filewrite(v->f, addr, len);
  uvmunmap(myproc()->pagetable, addr, len/PGSIZE, 1);

  return 0;
}  // add:end
```

修改`kernel/vm.c`中的`uvmunmap`函数，取消映射的页面可能并未实际分配, 此时跳过即可

```
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      // panic("uvmunmap: not mapped");
      continue;  // change
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

同样修改`kernel/vm.c`中的`uvmcopy`函数：

```
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      // panic("uvmcopy: page not present");
      continue;  // change
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

对于`fork`，还需要拷贝虚拟内存的相关信息到子进程：修改`kernel/proc.c`的`fork`函数：

```
int
fork(void)
{
  ...

  acquire(&np->lock);
  np->state = RUNNABLE;
  for(int i=0; i<NVMA; i++) {  // add
    memmove(&np->vmas[i], &p->vmas[i], sizeof(p->vmas[i]));
    if(p->vmas[i].f)
      filedup(p->vmas[i].f); 
  }// add:end
  release(&np->lock);

  return pid;
}
```

同理修改exit函数：

```
  for(int i=0; i<NVMA; i++) {  // add
    uvmunmap(p->pagetable, p->vmas[i].addr, p->vmas[i].len/PGSIZE, 1);
  }  // add:end

  begin_op();
  ...
```

### 3 ) 实验中遇到的问题和解决办法

问题：写完了编译不出来？

解决办法：事实证明随随便便添加头文件，然后直接调用其他模块的特有的结构体是不行的…这个虚拟内存的中断处理既要管`file`和`fs`，又要管`vm`，不能随便拿来定义直接把代码加入`trap`的逻辑里面，还得在各自的实现中封装起来再调用…前前后后搞了不少时间。

问题2：我终于编译出来了为啥又fail了…

解决办法：哈哈忘记给`len`页对齐了

问题3：我明明写了判断va是否合法为什么每次都给我报

```
scause 0x000000000000000d
sepc=0x0000000080003472 stval=0x0000000000000020
panic: kerneltrap
```

解决办法：不要迷信`setkilled(p)`，我以为设置完就没事了，结果后面的程序带着非法地址进去找对应`vma`了，难蚌，直接exit(-1)！

贴上我的

```
make grade
```

结果：

```
== Test running mmaptest == 
$ make qemu-gdb
(3.2s) 
== Test   mmaptest: mmap f == 
  mmaptest: mmap f: OK 
== Test   mmaptest: mmap private == 
  mmaptest: mmap private: OK 
== Test   mmaptest: mmap read-only == 
  mmaptest: mmap read-only: OK 
== Test   mmaptest: mmap read/write == 
  mmaptest: mmap read/write: OK 
== Test   mmaptest: mmap dirty == 
  mmaptest: mmap dirty: OK 
== Test   mmaptest: not-mapped unmap == 
  mmaptest: not-mapped unmap: OK 
== Test   mmaptest: two files == 
  mmaptest: two files: OK 
== Test   mmaptest: fork_test == 
  mmaptest: fork_test: OK 
== Test usertests == 
$ make qemu-gdb
usertests: OK (51.0s) 
```



### 4 ) 实验心得

终于做完了，我靠，一开始写的很乱导致编译都编译不出来属实打击信心了，本来想着这个做不出来就不做了，最后想想还是咬牙写吧，现在是2023.8.9的19.34分我靠巨开心。
