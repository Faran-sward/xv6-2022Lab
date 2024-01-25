# Lab: locks

# 1. Memory allocator ([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这个实验中，你必须使用一个专用的未加载的多核机器。你的任务是实现每个CPU的空闲列表，并在一个CPU的空闲列表为空时进行"偷取"。你必须给所有的锁命名以"kmem"开头。也就是说，你应该为你的每个锁调用initlock，并传递一个以"kmem"开头的名字。运行kalloctest来查看你的实现是否减少了锁争用。为了检查是否仍然可以分配所有的内存，运行usertests sbrkmuch。你的输出将类似于下面显示的内容，在kmem锁的总体争用中大大减少，尽管具体的数字会有所不同。确保usertests  -q中的所有测试都通过。make grade应该显示kalloctests通过。

1. 你可以使用`kernel/param.h`中的常量`NCPU`。 
2. 让`freerange`函数将所有的空闲内存分配给运行`freerange`的CPU。 
3. `cpuid`函数返回当前核心编号，但只有在关闭中断时才能安全地调用它并使用其结果。你应该使用`push_off()`和`pop_off()`来关闭和开启中断。
4.  可以参考`kernel/sprintf.c`中的`snprintf`函数来进行字符串格式化的想法。尽管可以将所有锁命名为"kmem"，但也可以选择不这样做。

### 2 ) 实验步骤

首先为了避免一把锁导致的太多冲突，为每个CPU都开一把锁维护各自的空闲内存列表，所以将原来的struct kmem改为struct kmem[NCPU]，修改位于`kernel/kalloc.c`的代码：

```
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];  // change
// } kmem;
```

修改位于`kernel/kalloc.c`中的kinit函数：只用初始化一次

```
void
kinit()
{
  // initlock(&kmem.lock, "kmem");
  for(int i = 0; i < NCPU; i++)
    initlock(&(kmem[i].lock), "kmem");  // change
  freerange(end, (void*)PHYSTOP);
}
```

修改位于`kernel/kalloc.c`中的kalloc函数：这里可能会遇到kmem中的freelist为空的情形，需要遍历其他的CPU获取空闲内存：

```
void *
kalloc(void)
{
  struct run *r;

  int id;  // add:
  id = cpuid();  // add:end

  // acquire(&kmem.lock);
  acquire(&kmem[id].lock);  // change
  // r = kmem.freelist;
  r = kmem[id].freelist;  // change
  if(r)
    // kmem.freelist = r->next;
    kmem[id].freelist = r->next;  // change
  // release(&kmem.lock);
  release(&kmem[id].lock);  // change

  // add: 如果该CPU的内存序列用完，则需要从其他CPU“借”
  for(int i = 0; i < NCPU && !r; i++){
      if(id != i){
        acquire(&(kmem[i].lock));
        r = kmem[i].freelist;
        if(r) kmem[i].freelist = r->next;
        release(&(kmem[i].lock));
      }
    }// add: end
  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

修改位于kernel/kalloc.c中的kfree函数，只要将CPU释放的内存添加到各自的freelist就好：

```
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  int id = cpuid();  // add
  // acquire(&kmem.lock);
  acquire(&kmem[id].lock);  // change
  // r->next = kmem.freelist;
  r->next = kmem[id].freelist;  // change
  // kmem.freelist = r;
  kmem[id].freelist = r;  // change
  // release(&kmem.lock);
  release(&kmem[id].lock);  // change
}
```

kalloctest运行结果：

```
$ kalloctest
start test1
test1 results:
--- lock kmem/bcache stats
lock: kmem: #test-and-set 0 #acquire() 41361
lock: kmem: #test-and-set 0 #acquire() 197921
lock: kmem: #test-and-set 0 #acquire() 193787
lock: bcache: #test-and-set 0 #acquire() 1270
--- top 5 contended locks:
lock: virtio_disk: #test-and-set 66881 #acquire() 141
lock: proc: #test-and-set 48740 #acquire() 1111480
lock: proc: #test-and-set 38740 #acquire() 1111484
lock: proc: #test-and-set 37229 #acquire() 711401
lock: proc: #test-and-set 16592 #acquire() 711361
tot= 0
test1 OK
start test2
total free number of pages: 32497 (out of 32768)
.....
test2 OK
start test3
child done 1
child done 100000
test3 OK
```

usertests sbrkmuch运行结果：

```
usertests sbrkmuch
usertests starting
test sbrkmuch: OK
ALL TESTS PASSED
```

usertests -q运行结果：

```
$ usertests sbrkmuch
usertests starting
test sbrkmuch: OK
ALL TESTS PASSED
$ usertests -q
usertests starting
test copyin: OK
test copyout: OK
test copyinstr1: OK
test copyinstr2: OK
test copyinstr3: OK
test rwsbrk: OK
test truncate1: OK
test truncate2: OK
test truncate3: OK
test openiput: OK
test exitiput: OK
test iput: OK
test opentest: OK
test writetest: OK
test writebig: OK
test createtest: OK
test dirtest: OK
test exectest: OK
test pipe1: OK
test killstatus: OK
test preempt: kill... wait... OK
test exitwait: OK
test reparent: OK
test twochildren: OK
test forkfork: OK
test forkforkfork: OK
test reparent2: OK
test mem: OK
test sharedfd: OK
test fourfiles: OK
test createdelete: OK
test unlinkread: OK
test linktest: OK
test concreate: OK
test linkunlink: OK
test subdir: OK
test bigwrite: OK
test bigfile: OK
test fourteen: OK
test rmdot: OK
test dirfile: OK
test iref: OK
test forktest: OK
test sbrkbasic: OK
test sbrkmuch: OK
test kernmem: usertrap(): unexpected scause 0x000000000000000d pid=6484
            sepc=0x00000000000021ee stval=0x0000000080000000
usertrap(): unexpected scause 0x000000000000000d pid=6485
            sepc=0x00000000000021ee stval=0x000000008000c350
usertrap(): unexpected scause 0x000000000000000d pid=6486
            sepc=0x00000000000021ee stval=0x00000000800186a0
usertrap(): unexpected scause 0x000000000000000d pid=6487
            sepc=0x00000000000021ee stval=0x00000000800249f0
usertrap(): unexpected scause 0x000000000000000d pid=6488
            sepc=0x00000000000021ee stval=0x0000000080030d40
usertrap(): unexpected scause 0x000000000000000d pid=6489
            sepc=0x00000000000021ee stval=0x000000008003d090
usertrap(): unexpected scause 0x000000000000000d pid=6490
            sepc=0x00000000000021ee stval=0x00000000800493e0
usertrap(): unexpected scause 0x000000000000000d pid=6491
            sepc=0x00000000000021ee stval=0x0000000080055730
usertrap(): unexpected scause 0x000000000000000d pid=6492
            sepc=0x00000000000021ee stval=0x0000000080061a80
usertrap(): unexpected scause 0x000000000000000d pid=6493
            sepc=0x00000000000021ee stval=0x000000008006ddd0
usertrap(): unexpected scause 0x000000000000000d pid=6494
            sepc=0x00000000000021ee stval=0x000000008007a120
usertrap(): unexpected scause 0x000000000000000d pid=6495
            sepc=0x00000000000021ee stval=0x0000000080086470
usertrap(): unexpected scause 0x000000000000000d pid=6496
            sepc=0x00000000000021ee stval=0x00000000800927c0
usertrap(): unexpected scause 0x000000000000000d pid=6497
            sepc=0x00000000000021ee stval=0x000000008009eb10
usertrap(): unexpected scause 0x000000000000000d pid=6498
            sepc=0x00000000000021ee stval=0x00000000800aae60
usertrap(): unexpected scause 0x000000000000000d pid=6499
            sepc=0x00000000000021ee stval=0x00000000800b71b0
usertrap(): unexpected scause 0x000000000000000d pid=6500
            sepc=0x00000000000021ee stval=0x00000000800c3500
usertrap(): unexpected scause 0x000000000000000d pid=6501
            sepc=0x00000000000021ee stval=0x00000000800cf850
usertrap(): unexpected scause 0x000000000000000d pid=6502
            sepc=0x00000000000021ee stval=0x00000000800dbba0
usertrap(): unexpected scause 0x000000000000000d pid=6503
            sepc=0x00000000000021ee stval=0x00000000800e7ef0
usertrap(): unexpected scause 0x000000000000000d pid=6504
            sepc=0x00000000000021ee stval=0x00000000800f4240
usertrap(): unexpected scause 0x000000000000000d pid=6505
            sepc=0x00000000000021ee stval=0x0000000080100590
usertrap(): unexpected scause 0x000000000000000d pid=6506
            sepc=0x00000000000021ee stval=0x000000008010c8e0
usertrap(): unexpected scause 0x000000000000000d pid=6507
            sepc=0x00000000000021ee stval=0x0000000080118c30
usertrap(): unexpected scause 0x000000000000000d pid=6508
            sepc=0x00000000000021ee stval=0x0000000080124f80
usertrap(): unexpected scause 0x000000000000000d pid=6509
            sepc=0x00000000000021ee stval=0x00000000801312d0
usertrap(): unexpected scause 0x000000000000000d pid=6510
            sepc=0x00000000000021ee stval=0x000000008013d620
usertrap(): unexpected scause 0x000000000000000d pid=6511
            sepc=0x00000000000021ee stval=0x0000000080149970
usertrap(): unexpected scause 0x000000000000000d pid=6512
            sepc=0x00000000000021ee stval=0x0000000080155cc0
usertrap(): unexpected scause 0x000000000000000d pid=6513
            sepc=0x00000000000021ee stval=0x0000000080162010
usertrap(): unexpected scause 0x000000000000000d pid=6514
            sepc=0x00000000000021ee stval=0x000000008016e360
usertrap(): unexpected scause 0x000000000000000d pid=6515
            sepc=0x00000000000021ee stval=0x000000008017a6b0
usertrap(): unexpected scause 0x000000000000000d pid=6516
            sepc=0x00000000000021ee stval=0x0000000080186a00
usertrap(): unexpected scause 0x000000000000000d pid=6517
            sepc=0x00000000000021ee stval=0x0000000080192d50
usertrap(): unexpected scause 0x000000000000000d pid=6518
            sepc=0x00000000000021ee stval=0x000000008019f0a0
usertrap(): unexpected scause 0x000000000000000d pid=6519
            sepc=0x00000000000021ee stval=0x00000000801ab3f0
usertrap(): unexpected scause 0x000000000000000d pid=6520
            sepc=0x00000000000021ee stval=0x00000000801b7740
usertrap(): unexpected scause 0x000000000000000d pid=6521
            sepc=0x00000000000021ee stval=0x00000000801c3a90
usertrap(): unexpected scause 0x000000000000000d pid=6522
            sepc=0x00000000000021ee stval=0x00000000801cfde0
usertrap(): unexpected scause 0x000000000000000d pid=6523
            sepc=0x00000000000021ee stval=0x00000000801dc130
OK
test MAXVAplus: usertrap(): unexpected scause 0x000000000000000f pid=6525
            sepc=0x000000000000229a stval=0x0000004000000000
usertrap(): unexpected scause 0x000000000000000f pid=6526
            sepc=0x000000000000229a stval=0x0000008000000000
usertrap(): unexpected scause 0x000000000000000f pid=6527
            sepc=0x000000000000229a stval=0x0000010000000000
usertrap(): unexpected scause 0x000000000000000f pid=6528
            sepc=0x000000000000229a stval=0x0000020000000000
usertrap(): unexpected scause 0x000000000000000f pid=6529
            sepc=0x000000000000229a stval=0x0000040000000000
usertrap(): unexpected scause 0x000000000000000f pid=6530
            sepc=0x000000000000229a stval=0x0000080000000000
usertrap(): unexpected scause 0x000000000000000f pid=6531
            sepc=0x000000000000229a stval=0x0000100000000000
usertrap(): unexpected scause 0x000000000000000f pid=6532
            sepc=0x000000000000229a stval=0x0000200000000000
usertrap(): unexpected scause 0x000000000000000f pid=6533
            sepc=0x000000000000229a stval=0x0000400000000000
usertrap(): unexpected scause 0x000000000000000f pid=6534
            sepc=0x000000000000229a stval=0x0000800000000000
usertrap(): unexpected scause 0x000000000000000f pid=6535
            sepc=0x000000000000229a stval=0x0001000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6536
            sepc=0x000000000000229a stval=0x0002000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6537
            sepc=0x000000000000229a stval=0x0004000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6538
            sepc=0x000000000000229a stval=0x0008000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6539
            sepc=0x000000000000229a stval=0x0010000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6540
            sepc=0x000000000000229a stval=0x0020000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6541
            sepc=0x000000000000229a stval=0x0040000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6542
            sepc=0x000000000000229a stval=0x0080000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6543
            sepc=0x000000000000229a stval=0x0100000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6544
            sepc=0x000000000000229a stval=0x0200000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6545
            sepc=0x000000000000229a stval=0x0400000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6546
            sepc=0x000000000000229a stval=0x0800000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6547
            sepc=0x000000000000229a stval=0x1000000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6548
            sepc=0x000000000000229a stval=0x2000000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6549
            sepc=0x000000000000229a stval=0x4000000000000000
usertrap(): unexpected scause 0x000000000000000f pid=6550
            sepc=0x000000000000229a stval=0x8000000000000000
OK
test sbrkfail: usertrap(): unexpected scause 0x000000000000000d pid=6562
            sepc=0x000000000000498a stval=0x0000000000013000
OK
test sbrkarg: OK
test validatetest: OK
test bsstest: OK
test bigargtest: OK
test argptest: OK
test stacktest: usertrap(): unexpected scause 0x000000000000000d pid=6570
            sepc=0x000000000000240c stval=0x0000000000010eb0
OK
test textwrite: usertrap(): unexpected scause 0x000000000000000f pid=6572
            sepc=0x000000000000248c stval=0x0000000000000000
OK
test pgbug: OK
test sbrkbugs: usertrap(): unexpected scause 0x000000000000000c pid=6575
            sepc=0x0000000000005c4e stval=0x0000000000005c4e
usertrap(): unexpected scause 0x000000000000000c pid=6576
            sepc=0x0000000000005c4e stval=0x0000000000005c4e
OK
test sbrklast: OK
test sbrk8000: OK
test badarg: OK
ALL TESTS PASSED
```

### 3 ) 实验中遇到的问题和解决办法

问题1：使用

```
make clean
make KCSAN=1 qemu
kalloctest
```

测试出错

解决方法：在某CPU内存耗尽时，需要向其他CPU挪用内存，如果不关开中断，就会导致大量锁竞争，影响性能，只需要修改为：

```
  // add: 如果该CPU的内存序列用完，则需要从其他CPU“借”
  for(int i = 0; i < NCPU && !r; i++){
      if(id != i){
        acquire(&(kmem[i].lock));
        push_off();  // 关中断 
        r = kmem[i].freelist;
        if(r) kmem[i].freelist = r->next;
        pop_off();  // 开中断
        release(&(kmem[i].lock));
      }
    }// add: end
```

问题2：有必要在获取cpuid前后关开中断吗？我没写中断前面的测试照样过了

解决方法：为某个CPU分配内存时如果内存不够，就需要干预其他CPU的内存，应该需要预防两个CPU同时申请一块内存的情形，但是在访问可用内存前已经使用互斥锁了，这一部分应该没问题，反而是不关中断会导致锁竞争，这样性能会降低。

### 4 ) 实验心得

因为申请释放统一，锁与解锁的方式也是统一的，不太可能出现死锁情形。对于问题2我在遍历所有CPU的空闲内存时加了关开中断后，optional tests也过了，挺有意思。

# 2. Buffer cache  ([hard](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

如果多个进程频繁使用文件系统，它们很可能会争用`bcache.lock`，该锁用于保护内核中的磁盘块缓存（kernel/bio.c）。`bcachetest`创建多个进程，重复读取不同的文件，以产生对`bcache.lock`的争用。修改块缓存，以便在运行bcachetest时，bcache中所有锁的acquire循环迭代次数接近零。理想情况下，所有涉及块缓存的锁的计数总和应为零，但如果总和小于500，也是可以的。修改bget和brelse，以便在bcache中对不同块的并发查找和释放不太可能在锁上发生冲突（例如，不必全部等待bcache.lock）。您必须保持每个块最多缓存一个副本的不变条件。

请为所有锁命名以“bcache”开头。也就是说，您应该为每个锁调用initlock，并传递以“bcache”开头的名称。

减少块缓存中的争用比kalloc更棘手，因为bcache缓冲区确实在进程（因此在CPU之间）之间共享。对于kalloc，可以通过为每个CPU提供自己的分配器来消除大部分争用，但对于块缓存而言，这种方法行不通。我们建议您在具有每个哈希桶的锁的哈希表中查找缓存中的块号。

有些情况下，如果您的解决方案存在锁冲突也是可以接受的：

- 当两个进程同时使用相同的块号时。bcachetest test0永远不会这样做。
- 当两个进程同时在缓存中未命中，并需要找到一个未使用的块来替换时。bcachetest test0永远不会这样做。
- 当两个进程同时使用在您用于分区块和锁的任何方案中相冲突的块时，例如，如果两个进程使用的块的块号在哈希表中的相同槽位上进行哈希。bcachetest test0可能会这样做，取决于您的设计，但您应该尝试调整您方案的细节以避免冲突（例如，更改哈希表的大小）。

bcachetest的test1使用比缓冲区更多的不同块，并且涵盖了许多文件系统代码路径。

- 阅读xv6书籍中有关块缓存的描述（第8.1-8.3节）。
- 可以使用固定数量的桶，不必动态调整哈希表的大小。使用质数个桶（例如13个）可以减少哈希冲突的可能性。
- 在哈希表中搜索缓冲区并在未找到缓冲区时分配一个条目必须是原子的。
- 删除所有缓冲区的列表（例如bcache.head等）并且不实现LRU。通过这种更改，brelse不需要获取bcache锁。在bget中，您可以选择任何refcnt == 0的块，而不是最近最少使用的块。
- 您可能无法原子地检查缓存的缓冲区并（如果未缓存）找到未使用的缓冲区；如果缓冲区不在缓存中，则可能需要放弃所有锁并从头开始。在bget中找到未使用的缓冲区时（即在缓存中查找失败的部分），进行序列化是可以的。
- 在某些情况下，您的解决方案可能需要同时持有两个锁；例如，在淘汰时，您可能需要持有bcache锁和每个桶的锁。确保避免死锁。
- 替换块时，您可能需要将一个struct buf从一个桶移动到另一个桶，因为新块的哈希与不同的桶匹配。您可能会遇到一个棘手的情况：新块的哈希可能与旧块相同。确保在这种情况下避免死锁。
- 一些调试技巧：实现桶锁，但在bget开头/结尾保留全局bcache.lock acquire/release以对代码进行序列化。一旦确定没有竞争条件，可以删除全局锁并处理并发问题。您还可以运行make CPUS=1 qemu以使用一个核心进行测试。
- 使用xv6的竞态检测器查找潜在的竞争条件（请参阅上文如何使用竞态检测器）。

### 2 ) 实验步骤

我们在kernel/bio.c中设置哈希桶的个数：

```
#define NBUCKETS 31  // add
```

相当于我们把原来的一条很长的链表用哈希桶分成了一个个小的链表组成的数组，在此基础上修改kernel/bio.c中的bcache：

```
struct {
  // struct spinlock lock;
  struct spinlock lock[NBUCKETS];  // change
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  
  // struct buf head;
  struct buf head[NBUCKETS];  // change
} bcache;
```

与上一个实验类似，修改kernel/bio.c中的binit函数：所有链表结构均初始化

```
void
binit(void)
{
  struct buf *b;

  for(int i = 0; i < NBUCKETS; i++){  // add:添加循环，初始化每个链表
      // initlock(&bcache.lock, "bcache");
      initlock(&bcache.lock[i], "bcache");  // change

    // Create linked list of buffers
    // bcache.head.prev = &bcache.head;
    bcache.head[i].prev = &bcache.head[i];  // change
    // bcache.head.next = &bcache.head;
    bcache.head[i].next = &bcache.head[i];  // change
  }  // add:end循环结束

  for(b = bcache.buf; b < bcache.buf+NBUF; b++){  // 初始化：将所有块放在第0个链表中
    // b->next = bcache.head.next;
    b->next = bcache.head[0].next;  // change
    // b->prev = &bcache.head;
    b->prev = &bcache.head[0];  // change
    initsleeplock(&b->lock, "buffer");
    // bcache.head.next->prev = b;
    bcache.head[0].next->prev = b;  // change
    // bcache.head.next = b;
    bcache.head[0].next = b;  // change
  }
}
```

修改kernel/bio.c中的bget函数，我们要找的磁盘块可能在哈希值对应的链表中，也可能不在，所以要分两种情况：

```
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;
  int hashvalue = blockno % NBUCKETS;  // add
  // acquire(&bcache.lock);
  acquire(&bcache.lock[hashvalue]);  // change: 先锁住最有可能的桶

  // Is the block already cached?
  // for(b = bcache.head.next; b != &bcache.head; b = b->next){  // 在哈希值对应的链表中寻找
  for(b = bcache.head[hashvalue].next; b != &bcache.head[hashvalue]; b = b->next){  // change
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      // release(&bcache.lock);
      release(&bcache.lock[hashvalue]);  // change: 找到了就解锁
      acquiresleep(&b->lock);
      return b;
    }
  }

  for(int i = (hashvalue + 1) % NBUCKETS; i % NBUCKETS != hashvalue; i = (i + 1) % NBUCKETS){  // add: 如果对应链表找不到，只好在其他桶找
    acquire(&(bcache.lock[i]));  // add
    // Not cached.
    // Recycle the least recently used (LRU) unused buffer.
    // for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    for(b = bcache.head[i].prev; b != &bcache.head[i]; b = b->prev){  // change: 注意他的遍历方向，是反过来的
      if(b->refcnt == 0) {
        b->dev = dev;
        b->blockno = blockno;
        b->valid = 0;
        b->refcnt = 1;

        //add:将找到的块插回对应链表
        b->next->prev = b->prev;
        b->prev->next = b->next;  // 从原来的链表断开
        b->next = bcache.head[hashvalue].next;
        b->prev = &bcache.head[hashvalue];  // 插在对应链表的头节点的后面
        bcache.head[hashvalue].next->prev = b;
        bcache.head[hashvalue].next = b;
        release(&bcache.lock[i]);  // 找到磁盘块所在的链也要解锁
        //add:end
        // release(&bcache.lock);
        release(&bcache.lock[hashvalue]);  // change: 准备将找到的块接回原来的哈希值对应的链表
        acquiresleep(&b->lock);
        return b;
      }
    }
    release(&(bcache.lock[i]));  // add
  }  // add:end
  panic("bget: no buffers");
}
```

修改kernel/bio.c中的brelse函数，比较简单，添加哈希值对应索引即可：

```
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);
  int hashvalue = b->blockno % NBUCKETS;  // add
  // acquire(&bcache.lock);
  acquire(&bcache.lock[hashvalue]);  // change
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->next->prev = b->prev;
    b->prev->next = b->next;
    // b->next = bcache.head.next;
    b->next = bcache.head[hashvalue].next;  // change
    // b->prev = &bcache.head;
    b->prev = &bcache.head[hashvalue];  // change
    // bcache.head.next->prev = b;
    bcache.head[hashvalue].next->prev = b;  // change
    // bcache.head.next = b;
    bcache.head[hashvalue].next = b;  // change
  }
  
  // release(&bcache.lock);
  release(&bcache.lock[hashvalue]);  // change
}
```

最后别忘了同样的方法修改kernel/bio.c中的bpin和bunpin函数：

```
void
bpin(struct buf *b) {
  int hashvalue = b->blockno % NBUCKETS;  // add
  // acquire(&bcache.lock);
  acquire(&bcache.lock[hashvalue]);  // change
  b->refcnt++;
  // release(&bcache.lock);
  release(&bcache.lock[hashvalue]);  // change
}

void
bunpin(struct buf *b) {
  int hashvalue = b->blockno % NBUCKETS;  // add
  // acquire(&bcache.lock);
  acquire(&bcache.lock[hashvalue]);  // change
  b->refcnt--;
  // release(&bcache.lock);
  release(&bcache.lock[hashvalue]);  // change
}
```

使用

```
make grade
```

查看结果：

```
== Test running kalloctest == 
$ make qemu-gdb
(88.4s) 
== Test   kalloctest: test1 == 
  kalloctest: test1: OK 
== Test   kalloctest: test2 == 
  kalloctest: test2: OK 
== Test   kalloctest: test3 == 
  kalloctest: test3: OK 
== Test kalloctest: sbrkmuch == 
$ make qemu-gdb
kalloctest: sbrkmuch: OK (8.3s) 
== Test running bcachetest == 
$ make qemu-gdb
(10.5s) 
== Test   bcachetest: test0 == 
  bcachetest: test0: OK 
== Test   bcachetest: test1 == 
  bcachetest: test1: OK 
== Test usertests == 
$ make qemu-gdb
usertests: OK (54.3s) 
```

### 3 ) 实验中遇到的问题和解决办法

问题：为什么很多hint我看不懂？然后感觉也没用到？

解决办法：要么就是看漏要求了…到头来还得在检查一遍，或者就是他的要求已经被实现了，比如

- ==删除所有缓冲区的列表（例如bcache.head等）并且不实现LRU。通过这种更改，brelse不需要获取bcache锁。在bget中，您可以选择任何refcnt == 0的块，而不是最近最少使用的块。==
- ==您可能无法原子地检查缓存的缓冲区并（如果未缓存）找到未使用的缓冲区；如果缓冲区不在缓存中，则可能需要放弃所有锁并从头开始。在bget中找到未使用的缓冲区时（即在缓存中查找失败的部分），进行序列化是可以的。==

FIFO最简单就直接写了，然后发现hint也默认是这样的

### 4 ) 实验心得

好多索引忘记加了哈哈哈…编译器报一堆错acquire(&bcache.lock);这样子，只要bget写对了，return前看一眼锁有没有release，所有锁都release就好
