# Lab: file system

# 1. Large files ([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这个任务中，您将增加xv6文件的最大大小。目前，xv6文件的大小限制为268个块，或者268 * BSIZE字节（在xv6中，BSIZE为1024）。这个限制源于xv6 inode包含12个“直接”块号和一个“一级间接”块号，后者引用一个块，其中包含最多256个更多的块号，总共为12 + 256 = 268个块。

bigfile命令会创建它能够创建的最长文件，并报告该文件的大小。您将更改xv6文件系统代码，以支持每个inode中的“双间接”块，其中包含256个单间接块的地址，每个单间接块可以包含最多256个数据块的地址。结果是，一个文件最多可以由65803个块组成，或者256 * 256 + 256 + 11个块（11代替12，因为我们将牺牲一个直接块编号用于双间接块）。

磁盘上inode的格式由fs.h中的struct dinode定义。您特别关注NDIRECT、NINDIRECT、MAXFILE以及struct dinode的addrs[]元素。在xv6文本的图8.3中有标准xv6 inode的示意图。

在fs.c中的bmap()函数负责找到文件在磁盘上的数据。查看一下它并确保理解它在做什么。bmap()在读取和写入文件时都会被调用。在写入时，bmap()会根据需要分配新的块来保存文件内容，如果需要保存块地址，则还会分配一个间接块。

bmap()处理两种类型的块号。bn参数是“逻辑块号”——文件内的块号，相对于文件的开头。ip->addrs[]中的块号以及传递给bread()的参数是磁盘块号。您可以将bmap()视为将文件的逻辑块号映射到磁盘块号。

hints：

- 确保您理解bmap()函数。绘制一个图表，展示ip->addrs[]、间接块、双重间接块以及它指向的单重间接块和数据块之间的关系。确保您理解为什么添加一个双重间接块会将最大文件大小增加256*256个块（实际上是-1，因为您必须减少一个直接块的数量）。
-  考虑如何使用逻辑块号索引双重间接块以及它所指向的单重间接块。 如果更改了NDIRECT的定义，则可能需要更改file.h中struct inode中的addrs[]声明。确保struct  inode和struct dinode在它们的addrs[]数组中具有相同数量的元素。
-  如果更改了NDIRECT的定义，请务必创建一个新的fs.img，因为mkfs使用NDIRECT来构建文件系统。
-  如果您的文件系统处于不良状态，可能是由于崩溃引起的，请删除fs.img（在Unix中执行此操作，而不是在xv6中）。make将为您构建一个新的干净的文件系统映像。 
- 不要忘记对每个使用brelse()函数释放的块进行brelse()。 
- 您应该像原始的bmap()函数一样，根据需要分配间接块和双重间接块。 
- 确保itrunc()函数释放文件的所有块，包括双重间接块。 
- 由于本实验中FSSIZE较大且大文件较多，usertests运行时间较长。

### 2 ) 实验步骤

首先扩大可使用文件的大小，修改kernel/fs.h中的关于MAXFILE等定义，添加二级索引：

```
// #define NDIRECT 12
#define NDIRECT 11  // change
#define NINDIRECT (BSIZE / sizeof(uint))
#define NBINDIRECT (BSIZE / sizeof(uint)) * (BSIZE / sizeof(uint))  // add:增加二级索引表示的文件大小
// #define MAXFILE (NDIRECT + NINDIRECT)
#define MAXFILE (NDIRECT + NINDIRECT + NBINDIRECT)  // change: 前面系数都为1,表示11个直接索引，1个间接索引，1个二级索引表示的文件的最大范围
```

原先的dinode定义中addrs只使用了两级索引，所以需要在kernel/fs.h修改：

```
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  // uint addrs[NDIRECT+1];   // Data block addresses
  uint addrs[NDIRECT+2];   // change: Data block addresses
};
```

保证addrs使用了13个地址，按照同样的方法修改位于kernel/file.h中的定义：

```
// in-memory copy of an inode
struct inode {
  ...
  uint size;
  // uint addrs[NDIRECT+1];
  uint addrs[NDIRECT+2];  // change
};
```

还要告诉kernel/fs.h关于一级索引和二级索引在addrs中的位置，因为原来的实现是默认最后一个为一级索引，其余为直接索引的：

```
#define DINDIRECT_PTR 12  // add: 第13个地址
```

接下来修改kernel/fs.c中的bmap函数，bmap()函数负责将文件的逻辑块号映射为对应的磁盘块号，以便在读取和写入文件时找到正确的数据块，为其添加二级索引的映射方式，参考一级索引即可：

```
  ...
  //add
  bn -= NINDIRECT;
  if(bn < NBINDIRECT){  // 逻辑块号属于二级索引
    if((addr = ip->addrs[DINDIRECT_PTR])  == 0){  // 该块还没有分配
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
        ip->addrs[DINDIRECT_PTR] = addr;  // 分配新块
    }
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn / NINDIRECT]) == 0){  // 访问索引块
      addr = balloc(ip->dev);
      if(addr){
        a[bn / NINDIRECT] = addr;
        log_write(bp);
      }
    }
    brelse(bp);  // 别忘记释放一级指针
    bp = bread(ip->dev, addr);  // 访问第二级
    a = (uint*)bp->data;
    if((addr = a[bn % NINDIRECT]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn % NINDIRECT] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }  // add:end

  panic("bmap: out of range");
}
```

再修改kernel/fs.c中释放文件所有块的itrunc函数：添加释放二级索引块的方式：

```
void
itrunc(struct inode *ip)
{  // 在调用这个函数之前，调用者必须已经持有了这个inode的锁
  int i, j;
  struct buf *bp, *subbp;  // add
  uint *a, *b;  // add
  
  ...
  
    if(ip->addrs[DINDIRECT_PTR]){  // add:释放二级索引块
    bp = bread(ip->dev, ip->addrs[DINDIRECT_PTR]);
    a = (uint*)bp->data;
    for(i = 0; i < NINDIRECT; i++){
      if(a[i]){
        subbp = bread(ip->dev, a[i]);
        b = (uint*)subbp->data;
        for(j = 0; j < NINDIRECT; j++)
          if(b[j])
            bfree(ip->dev, b[j]);
        brelse(subbp);
        bfree(ip->dev, a[i]);
        a[i] = 0;
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[DINDIRECT_PTR]);
    ip->addrs[DINDIRECT_PTR] = 0;
  }// add:end
  ip->size = 0;
  iupdate(ip);
}
```

使用

```
bigfile
usertests -q
```

在qemu环境测试：结果如下：

```
$ bigfile
..................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................
wrote 65803 blocks
bigfile done; ok
```

### 3 ) 实验中遇到的问题和解决办法

问题1：brelse和bfree搞混了

解决方案：后面看了代码发现brelse是上一个lab主要用的，来释放磁盘缓冲区的，先持有缓冲区struct buf的锁，在持有bcache的全局锁，然后修改引用计数和链表，这是上一个lab主要实现的；bfree纯粹释放磁盘块的，按照逻辑块数访问不同等级索引的块，只不过释放的过程需要用bread将块读到缓冲区，也就自然用到了brelse。

### 4 ) 实验心得

问题讲过了，把两个函数搞混了，写一半才发现，不过对这已经有的代码写挺简单。

# 2. Symbolic links ([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这个练习中，你将向xv6添加符号链接（Symbolic Links）。符号链接是通过路径名引用链接文件的一种方式；当打开一个符号链接时，内核会根据链接的路径找到实际的文件。符号链接类似于硬链接（Hard Links），但硬链接只能指向同一磁盘上的文件，而符号链接可以跨越不同的磁盘设备。

你将实现`symlink(char *target, char *path)`系统调用，该调用在路径`path`创建一个新的符号链接，该符号链接指向`target`所指定的文件。

1. 首先，为`symlink`创建一个新的系统调用号，将其添加到`user/usys.pl`、`user/user.h`中，并在`kernel/sysfile.c`中实现一个空的`sys_symlink`函数。
2. 在`kernel/stat.h`中添加一个新的文件类型（`T_SYMLINK`）来表示符号链接。
3. 在`kernel/fcntl.h`中添加一个新的标志（`O_NOFOLLOW`），可用于与`open`系统调用一起使用。注意，传递给`open`的标志使用位OR操作符进行组合，因此你的新标志不应与任何现有标志重叠。一旦将`user/symlinktest.c`添加到Makefile中，这将使你能够编译该文件。
4. 实现`symlink(target, path)`系统调用，以在路径`path`创建一个新的符号链接，该符号链接指向`target`。请注意，为了使系统调用成功，`target`无需存在。你需要选择一个位置来存储符号链接的目标路径，例如，在inode的数据块中。`symlink`应该返回一个表示成功（0）或失败（-1）的整数，类似于`link`和`unlink`。
5. 修改`open`系统调用，以处理路径引用符号链接的情况。如果文件不存在，`open`必须失败。当进程在`open`的标志中指定了`O_NOFOLLOW`时，`open`应该打开符号链接（而不是跟随符号链接）。
6. 如果链接的文件也是符号链接，你必须递归地跟随它，直到达到非链接文件。如果链接形成循环，你必须返回一个错误代码。你可以通过在链接深度达到某个阈值（例如，10）时返回错误代码来近似实现此操作。
7. 其他系统调用（例如，`link`和`unlink`）不得跟随符号链接；这些系统调用仅操作符号链接本身。
8. 对于此实验，你不必处理对目录的符号链接。

### 2 ) 实验步骤

按照hint，先添加系统调用sys_symlink：

修改kernel/syscall.h，添加系统调用号：

```
...
#define SYS_symlink 22  // add
```

修改kernel/syscall.c，添加声明，建立映射：

```
extern uint64 sys_symlink(void);  // add
...
[SYS_symlink] sys_symlink,  // add
```

修改user/usys.pl，添加跳板函数：

```
entry("symlink");
```

修改user/user.h，添加用户函数声明：

```
int symlink(char*, char*);  // add
```

修改kernel/stat.h，添加新的文件类型T_SYMLINK

```
#define T_SYMLINK 4   // add
```

修改kernel/fcntl.h，添加新的文件访问类型：O_NOFOLLOW

```
#define O_NOFOLLOW 0x800  // add
```

在Makefile添加symlinktest：

```
...
$U/_symlinktest\
```

在kernel/sysfile.c中添加sys_symlink函数：

```
uint64  // add
sys_symlink(void)
{
  char name[DIRSIZ], target[MAXPATH], path[MAXPATH];
  struct inode *dp, *ip, *sym;
  //ip指向target dp指向path的父目录
  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
      return -1;

  begin_op();
  if((ip = namei(target)) != 0) // 提示要求target指定的文件是可以不存在的
  {
    ilock(ip);

    if(ip->type == T_DIR) // 根据提示，不需要处理target是一个目录文件的情况
    {
      iunlockput(ip);
      end_op();
     return -1;
   }
    iunlockput(ip);
  }

  if((dp = nameiparent(path, name)) == 0) //获取path的父目录
   {
      end_op();
      return -1;
   }
  ilock(dp);
 if((sym = dirlookup(dp, name, 0)) != 0)  //如果有同名文件，则生成失败
  {
   iunlockput(dp);
   end_op();
  return -1;
  }

  if((sym = ialloc(dp->dev, T_SYMLINK)) == 0)  //仿照create生成符号链接类型的inode
        panic("create: ialloc");

    ilock(sym);
    sym->nlink = 1;
    iupdate(sym);

    if(dirlink(dp, name,sym->inum) < 0) //把符号链接文件加入到path的父目录中
        panic("create: dirlink");

    iupdate(dp);
    //把target字符串写入符号链接文件里面
    if(writei(sym, 0, (uint64)&target, 0, strlen(target)) != strlen(target))
        panic("symlink: writei");
    iupdate(sym);//注意调用iupdate把inode信息写入磁盘
    iunlockput(dp);
    iunlockput(sym);
    end_op();
    return 0;

}  // add:end
```

修改kernel/sysfile.c中的sys_open函数，在处理完T_DEVICE后添加：

```
  //add
  int i = 0;
  while(ip->type == T_SYMLINK && i++ < 10 && !(omode & O_NOFOLLOW)){
    if(readi(ip, 0, (uint64)sympath, 0, ip->size) != ip->size){
      iunlockput(ip);
      end_op();
      return -1;
    }
    iunlockput(ip);
    if(!(sym = namei(sympath))){
      end_op();
      return -1;
    }
    ip = sym;
    ilock(ip);
  } // 是符号文件且没到最大深度
  if(i == 11){
    iunlockput(ip);
    end_op();
    return -1;
  }  //  add:end
```

使用

```
symlinktest
```

测试结果：

```
$ symlinktest
Start: test symlinks
test symlinks: ok
Start: test concurrent symlinks
test concurrent symlinks: ok
```

使用

```
make grade
```

查看结果：

```
== Test running bigfile == 
$ make qemu-gdb
running bigfile: OK (83.1s) 
== Test running symlinktest == 
$ make qemu-gdb
(0.6s) 
== Test   symlinktest: symlinks == 
  symlinktest: symlinks: OK 
== Test   symlinktest: concurrent symlinks == 
  symlinktest: concurrent symlinks: OK 
== Test usertests == 
$ make qemu-gdb
usertests: OK (138.9s) 
```

### 3 ) 实验中遇到的问题和解决办法

问题：检查一遍又一遍就是不行，关机再开机就全通过了…

```
    if(readi(ip, 0, (uint64)&sympath, 0, ip->size) != ip->size){
      iunlockput(ip);
      end_op();
      return -1;
    }
```

每次都是这一句报错，什么都读不出来，

解决方法：我实在找不出来，玄学

### 4 ) 实验心得

我ilock都及时关了为什么还报错呢？实在不行重写一遍sys_symlink，结果又好了？
