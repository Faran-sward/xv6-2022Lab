# Lab: Copy-on-Write Fork for xv6

# 1. Implement copy-on-write fork([hard](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这个任务中，你需要实现` xv6 `内核中的写时复制 (COW) fork。`COW fork()` 仅为子进程创建一个页表，其中用户内存的 PTE 指向父进程的物理页面。`COW fork()` 将父进程和子进程中的所有用户  PTE 都标记为只读。当任一进程尝试写入这些 `COW` 页面时，CPU  会强制产生页面故障。内核页面故障处理程序会检测到这种情况，为发生故障的进程分配一个物理内存页面，并将原始页面复制到新页面中，然后修改发生故障的进程中的相关 `PTE`，此时将` PTE` 标记为可写。当页面故障处理程序返回时，用户进程将能够写入其页面的副本。

你需要：

1. 修改 `uvmcopy() `函数，将父进程的物理页面映射到子进程中，而不是分配新页面。对于设置了 `PTE_W` 的页面，将父进程和子进程中的所有用户 `PTE` 的 `PTE_W `清零。
2. 修改` usertrap() `函数，识别页面故障。当发生写页面故障，并且这是一个 COW 页面，并且最初是可写的，则使用 `kalloc() `为发生故障的进程分配一个新的页面，并将原始页面复制到新页面中，并在发生故障的进程的 `PTE `中安装新页面，此时将 `PTE_W` 设置为可写。原始是只读的页面（未设置 `PTE_W`，例如文本段中的页面）应保持只读，并在父进程和子进程之间共享；尝试写入此类页面的进程应被杀死。
3. 确保每个物理页面在最后一个引用消失时被释放，但不要在之前释放。一个不错的方法是为每个物理页面保持一个“引用计数”，表示引用该页面的用户页表数目。当` kalloc() `分配一个页面时，将页面的引用计数设置为 1。当 fork 导致子进程共享页面时，增加页面的引用计数，并在任何进程从其页表中删除页面时减少页面的引用计数。只有当页面的引用计数为零时，`kfree() `才会将页面放回空闲列表中。你可以使用固定大小的整数数组来保存这些计数，例如根据页面的物理地址除以 4096 来索引数组，然后给数组分配与 `kalloc.c `中的 `kinit() `分配的页面的最高物理地址相等数量的元素。
4. 修改 `copyout() `函数，以与页面故障相同的方式处理 COW 页面。

### 2 ) 实验步骤

在发生页面错误时，我们需要为这个物理页设置一个额外的控制位COW，标识子进程需要的被复制的页面，我们在`kernel/riscv.h`中添加：

```
...
#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // user can access
#define PTE_COW (1L << 9)  // add
...
```

写时复制导致一个物理页会被潜在的其他进程使用，所以不能在一个进程被销毁时就释放所有它申请的物理页，所以必须记录每个物理页的被索引次数，只有当索引为0时才能释放该页，类似全局文件打开表，我们在`kernel/kalloc.c`中声明一个全局数组：

```
...
uint8 page_index_count[PHYSTOP / PGSIZE] = {};  // add
...
```

这里默认从0地址开始，包含内核访问的物理页，因为一开始为内核分配的物理页不是用`kalloc`分配的而是`freerange`函数分配一片内存到内核，所以需要修改位于`kernel/kalloc.c`的`freerange`函数：

```
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE){
    page_index_count[(uint64)p / PGSIZE] = 1;  // add
    kfree(p);
  }
```

因为`page_index_count`会被多个进程访问，所以要加上互斥锁，保证引用计数的正常更新。 

修改`kernel/kalloc.c`中的`kalloc`函数，当分配页面时，将`page_index_count`中将分配的页面对应引用设置为1：

```
...
r = kmem.freelist;
  if(r){
    kmem.freelist = r->next;
    page_index_count[(uint64)r / PGSIZE] = 1;  // add
  }
...
```

修改`kernel/kalloc.c`中的`kfree`函数，当收回分配页面时，将`page_index_count`中将分配的页面对应引用减1，当引用计数大于0时直接返回。

```
void
kfree(void *pa)
{
  // add
  acquire(&(kmem.lock));
  if(--page_index_count[(uint64)pa / PGSIZE]){
    release(&(kmem.lock));
    return;
  }
  release(&(kmem.lock));  // add:end

  struct run *r;
  ...
```

`fork`函数调用时还需要对引用计数+1,因此我们在`kernel/kalloc.c`中添加一个函数`index_plusplus`：

```
void  // add
index_plusplus(uint64 pa)  // 物理地址
{
  acquire(&(kmem.lock));
  page_index_count[pa / PGSIZE]++;
  release(&(kmem.lock));
}
```

顺便实现`index_subsub`函数用于引用计数-1：

```
void  // add
index_subsub(uint64 pa)
{
  acquire(&(kmem.lock));
  page_index_count[pa / PGSIZE]--;
  release(&(kmem.lock));
}
```

并在`kernel/defs.h`中添加声明：

```
...
// kalloc.c
void*           kalloc(void);
void            kfree(void *);
void            kinit(void);
void            index_plusplus(uint64);  // add
...
```

完成上述准备后，我们按照hint修改位于`kernel/vm.c`中的`uvmcopy`函数：

注意：uvmcopy已经不再实现复制页面了！！！需要等到页面错误时再调用memmove，所以这里要删除memmove, kfree, mappage等调用

```
int  // 用于复制虚拟内存中的页面数据
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;  // 保存旧页表中页面的页表项
  uint64 pa, i;  // 保存页面对应的物理地址
  uint flags;  // 保存页面的标志位
  // del: char *mem;  // 保存新分配的物理内存

  for(i = 0; i < sz; i += PGSIZE){
    // 通过 walk 函数找到旧页表中虚拟地址 i 对应的页表项 pte
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    // 检查该页面是否已经被加载到物理内存中
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);  // 获取页面的物理地址 
    *pte &= ~PTE_W;  // add
    *pte |= PTE_COW;  // add:end
    flags = PTE_FLAGS(*pte);  // 获取页面的标志位 
    /* del:
    if((mem = kalloc()) == 0)  // 使用 kalloc 函数分配一块新的物理内存
      goto err;
    memmove(mem, (char*)pa, PGSIZE);  // 复制数据到新页表的i页表项
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);  // 将新分配的物理内存映射到新页表中的虚拟地址 i 处
      goto err;
    }
    del:end*/
    if(mappages(new, i, PGSIZE, pa, flags) != 0)  // add:将原来的物理页映射到新的页表
      goto err;  // add
    index_plusplus(pa);  // add
  }
  return 0;
```

现在将原来的物理页的写权限去除，导致所有进程对其的写动作均发生`page fault`，修改`kernel/trap.c`中的`usertrap`函数，让其转入`cowfault`函数：

```
void
usertrap(void)
{
	...
  
  if(r_scause() == 8){
    // system call
	...
  } 
  else if(r_scause() == 15){  // add
    struct proc* p = myproc();
    if(cowfault(p->pagetable, r_stval()) < 0){
      setkilled(p);
      exit(-1);  // add:end
  } 
  } ...
}
```

接下来详细说明在`kernel/kalloc.c`中添加的`cowfault`函数，在此之前，先在`kernel/defs.h`中添加定义：

```
...
// kalloc.c
void*           kalloc(void);
void            kfree(void *);
void            kinit(void);
void            index_plusplus(uint64);  // add
int             cowfault(pagetable_t, uint64); // add
...
```

`cowfault`函数用于处理访问`PTE_COW`页的情形，具体分为两种情况，如果访问页只有一个进程使用，那么可以恢复为原来的可写页；如果有两个以上的进程使用，那么就需要重新复制一个物理页分给待使用的进程。

```
int  // add
cowfault(pagetable_t pagetable, uint64 va)
{
  if(va >= MAXVA || va <= 0)
    return -1; // 地址不合法
  pte_t* pte = walk(pagetable, va, 0);  // 获取地址对应页表项
  if(!pte || !(*pte & PTE_COW))
    return -1; // 非COW页
  if(page_index_count[PTE2PA(*pte) / PGSIZE] >= 2){
    char* mem;
    if((mem = kalloc()) == 0)
      return -1;  // 申请物理页失败
    memmove(mem, (char*)PTE2PA(*pte), PGSIZE);
    uint flag = (PTE_FLAGS(*pte) | PTE_W) & (~PTE_COW);
    *pte = PA2PTE((uint64)mem) | flag;  // 将新的物理页设置可写分配给访问进程
  }  // 如果有两个以上的进程使用该页就需要复制
  else{
    *pte |= PTE_W;
    *pte &= (~PTE_COW);
  }  // 否则说明fork的进程未使用该页就已经结束了，只需恢复原来的写权限
  return 0;
}
```

最后修改`kernel/vm.c`中的`copyout`函数：

```
int  // 将内核空间的数据复制到指定虚拟地址
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);  // dstva 所在页面的起始虚拟地址 va0
    // add：之前默认访问非COW页
    if(va0 >= MAXVA)  // 虚拟地址越界
      return -1;
    pte_t* pte = walk(pagetable, va0, 0);  // 找到页表项
    if (pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0)
      return -1;
    if((*pte & PTE_COW) && cowfault(pagetable, va0) < 0)  // 重新分配失败
      return -1;  // add:end
    pa0 = walkaddr(pagetable, va0);  // 对应的物理地址
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);  // 计算当前页面中可复制的字节数
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;  // dstva 更新为下一个页面的起始虚拟地址
  }
  return 0;
}
```

最后贴一下成果：

```
== Test running cowtest == 
$ make qemu-gdb
(6.6s) 
== Test   simple == 
  simple: OK 
== Test   three == 
  three: OK 
== Test   file == 
  file: OK 
== Test usertests == 
$ make qemu-gdb
(42.5s) 
    (Old xv6.out.usertests failure log removed)
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyout == 
  usertests: copyout: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK 
```

### 3 ) 实验中遇到的问题和解决办法

问题1：为什么我`xv6`打不开了？第一个进程卡住了？

解决方法：第一片内存不是`kalloc`开辟的，是`freerange`开辟的，所以得补个索引++

问题2：怎么`init`创建了2000多个进程报错？

解决方法：`uvmcopy`忘记把`mappage`那块删掉重写了，导致读的页都是非法的…

问题3：`usertests`的`copyout`怎么又出事了？

解决方法：虚拟地址越界了我是真没想到，还得加个判断

问题4：在胜利的最后一步的`textwrite`倒下了

解决方法：应该是`cowfault`出错时忘记结束进程了，少了一句exit(-1)，结果改完后还是有这个问题，看了一下要给`va=0`时直接return掉就好？

### 4 ) 实验心得

错误频出，有些地方非常玄学，

```
uint8 page_index_count[PHYSTOP / PGSIZE] = {};  // add
```

不加大括号也能错？活生生多改了一小时bug
