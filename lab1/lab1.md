# Lab: Xv6 and Unix utilities

## 1. Boot xv6 ([easy](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

搭建`xv6`环境并检验是否能成功运行，尝试已经实现的xv6指令如ls

### 2 ) 实验步骤

`git`下载实验环境，在` terminal` 执行以下指令：

```
git clone git://g.csail.mit.edu/xv6-labs-2022
cd xv6-labs-2022
git status
git log
git checkout util
```

结果如下：

```
位于分支 util
您的分支与上游分支 'origin/util' 一致。
```

在` terminal `执行 `make qemu `指令：

结果如下：

```
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ 
```

### 3 ) 实验中遇到的问题和解决办法

问题：不熟悉 linux 命令行

解决办法：熟悉常见指令如 grep，cp 等指令

### 4 ) 实验心得

一开始觉得挺神奇的，可以用C语言写 xv6 

## 2. sleep ([easy](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在 `xv6` 中实现`UNIX`程序`sleep`，使得`sleep`可以暂停用户指定数量的`tick` 数。`tick` 是由 `xv6 `内核定义的时间单位，即定时器芯片产生两次中断之间的时间间隔。你的解决方案应该在文件`user/sleep.c`中。

### 2 ) 实验步骤

新建 user/sleep.c 文件：

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char** argv){
    if(argc < 2){
        fprintf(2, "please input time to sleep.");
        exit(1);
    }
    sleep(atoi(argv[1]));
    exit(0);
    
}
```

在`Makefile`中添加`_sleep`：

```
UPROGS=\
	$U/_cat\
	$U/_echo\
	...
	$U/_wc\
	$U/_zombie\
	$U/_sleep\
```

使用

```
make GRADEFLAGS=sleep grade
```

判断是否成功，实验结果：

```
== Test sleep, no arguments == sleep, no arguments: OK (2.3s) 
== Test sleep, returns == sleep, returns: OK (1.0s) 
== Test sleep, makes syscall == sleep, makes syscall: OK (1.0s) 
```

### 3 ) 实验中遇到的问题和解决办法

问题1：一开始不知道如何实现

解决办法：实验给出了提示：

Look at some of the other programs in `user/`    (e.g., `user/echo.c`, `user/grep.c`,    and `user/rm.c`)    to see    how you can obtain the command-line arguments passed to a program.     

看懂其他指令的实现方式，照葫芦画瓢。

问题2：还是不知道用什么实现

解决方法：实验给出了提示：

See `kernel/sysproc.c` for    the xv6 kernel code that implements the `sleep` system call (look for `sys_sleep`), `user/user.h`    for the C definition of `sleep` callable from a    user program, and `user/usys.S` for the assembler    code that jumps from user code into the kernel for `sleep`.     

`xv6`实现了`sleep`的系统调用，直接使用`user.h`中声明的函数就好。

问题3：还是不知道有了参数怎么实现

解决方法：`xv6`里有一些小的C语言库和已经实现的函数，比如`atoi`将字符串转为数字，直接使用就好。

### 4 ) 实验心得

很不习惯，很难把指令的实现和这么短的代码联系在一起，中间的很多部分xv6已经写好了，现在看起来确实不难，一开始绕来绕去那么多hints就为了这样简短的调用，为什么不把话说的更明白点，可能教授们喜欢循循善诱吧。

## 3. pingpong ([easy](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

你需要编写一个程序，使用`UNIX`系统调用在两个进程之间通过一对管道（每个方向一个管道）进行`“ping-pong”`操作。父进程将发送一个字节给子进程；子进程将打印`"<pid>: received ping"`，其中<pid>是它的进程`ID`，将该字节写入管道给父进程，并退出；父进程将从子进程读取该字节，打印`"<pid>: received pong"`，然后退出。你的解决方案应该在文件`user/pingpong.c`中。

### 2 ) 实验步骤

新建 `user/pingpong.c` 文件：

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(){

    int p1[2], p2[2];
    pipe(p1);  // from parent to child
    pipe(p2);  // 
    char buf[64] = {};
    if(fork() == 0){
        {
            if(read(p1[0], buf, 4) != 4){
                fprintf(2, "child read error\n");
                exit(1);
            }
            close(p1[0]);
            fprintf(0, "%d: received ping\n", getpid());
        }
        {
            if(write(p2[1], "pong", 4) != 4){
                fprintf(2, "child write error\n");
                exit(1);
            }
            close(p2[1]);
        }
    }
    else{
        {
            if(write(p1[1], "ping", 4) != 4){
                fprintf(2, "parent write error\n");
                exit(1);
            }
            close(p1[1]);
        }
        {
            if(read(p2[0], buf, 4) != 4){
                fprintf(2, "parent read error\n");
                exit(1);
            }
            fprintf(0, "%d: received pong\n", getpid());
            close(p2[0]);
        }
    }
    exit(0);
}
```

在`Makefile`中添加`_pingpong`：

```
UPROGS=\
	$U/_cat\
	...
	$U/_sleep\
	$U/_pingpong\
```

使用

```
make GRADEFLAGS=pingpong grade
```

判断是否成功，实验结果：

```
== Test pingpong == pingpong: OK (1.9s) 
```

### 3 ) 实验中遇到的问题和解决办法

问题1：这题到底要不要用IO重定向？

解决方法：根本没必要把管道的文件描述符重定向到0号标准输出，直接让fprintf在0号标准输出打印需要的结果即可

问题2：怎么排列read和write的顺序？

解决方法：如果写管道未关闭且还没写好，那么读就是堵塞的，所以可以利用这一点让一个进程先写，另一个进程先读，确保一条语句先被打印出来，就不会存在两条语句同时打印的情况。

### 4 ) 实验心得

在是否IO重定向我纠结了很久，浪费了很多时间，因为并发执行导致如果重定向两条语句可能会同时打印。

追求严谨可以在每个进程中提前关闭不需要的文件描述符，如果要读，可以提前关闭写管道，如果要写，则写完后及时关闭两个管道。

## 4. primes ([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))/([hard](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

你需要编写一个使用管道实现并发版本的素数筛法`（prime sieve）`。这个想法是由`Unix`管道的发明者Doug McIlroy提出的。在这个页面的中部，有一张图和周围的文字解释了如何实现。你的解决方案应该在文件`user/primes.c`中。

你的目标是使用管道和`fork`来设置管道线。第一个进程将数字`2`到`35`送入管道线。对于每个素数，你将创建一个进程，该进程从其左邻居通过管道读取数据，并通过另一个管道将数据写入其右邻居。由于xv6有限的文件描述符和进程数量，第一个进程可以在35处停止。

### 2 ) 实验步骤

新建 `user/primes.c `文件：

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
const int N = 35;

void input(int p[2], int list[], int length){
    for(int i = 0; i < length; i++)
        if(write(p[1], (list + i), sizeof(list[i])) < sizeof(list[i])){
            fprintf(2, "write error in pipe input.");
            exit(1);
        }
    close(p[1]);
    close(p[1]); 
}

int output(int p[2], int list[]){
    close(p[1]);
    int index = 0, front, len = read(p[0], &front, sizeof(len));
    if(len == 0)
        return 0;
    if(len < sizeof(len)){
        fprintf(2, "read error in pipe output.");
        exit(1);
    }
    fprintf(0, "prime %d\n", front);
    while((len = read(p[0], index + list, sizeof(len))) == sizeof(len)){
        if(list[index] % front != 0)
            index++;
    }
    close(p[0]);
    return index;
}

void process(int p[2]){

    int num[N], length = output(p, num), next[2];
    if(length == 0)
        exit(0);
    pipe(next);
    input(next, num, length);
    if(fork() == 0)
        process(next);
}

int main(){
    int p[2], num[N];
    pipe(p);

    if(fork() == 0){
        process(p);
        wait(0);
        exit(0);
    }
    else{
        for(int i = 2, j = 0; i <= N; i++, j++){
            num[j] = i;
        }
        input(p, num, N - 1);
    }
    wait(0);
    exit(0);
}
```

在`Makefile`中添加`_primes`：

```
UPROGS=\
	$U/_cat\
	...
	$U/_sleep\
	$U/_pingpong\
	$U/_primes\
```

使用

```
make GRADEFLAGS=primes grade
```

判断是否成功，实验结果：

```
== Test primes == primes: OK (2.4s) 
```

### 3 ) 实验中遇到的问题和解决办法

问题1：怎么循环建立进程？

解决方法：原来在函数里面循环调用就好，只不过需要在递归语句前面加上    if(fork() == 0)

问题2：要是管道开的太多怎么办要不要只开两个管道？

解决方法：只有两个管道实际上是把这4个文件描述符反复打开了很多次，根本不能做到hint中的当写管道关闭且读管道读完时返回0，最后一个被创建的进程也就不会被释放！

### 4 ) 实验心得

浪费的时间很长，原因在于没有弄清fork进程时文件描述符不仅会复制到子进程，并且在整个的文件打开表中增加计数，而且一开始程序写的比较乱，没有弄清楚每个进程从上一个管道读什么，从下一个管道些什么，后面才弄清相当于从2到$\sqrt{n}$为每个数建立一个进程筛选掉倍数。

## 5. find ([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

你需要编写一个简单版本的UNIX `find`程序：在目录树中查找所有具有特定名称的文件。你的解决方案应该在文件`user/find.c`中。

### 2 ) 实验步骤

新建 `user/find.c` 文件：

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

int match(const char *s, const char *pattern)
{
    for (int i = 0; i <= strlen(s) - strlen(pattern); i++)
        for (int j = 0; j < strlen(pattern); j++)
        {
            if (s[i + j] != pattern[j])
                break;
            if (j == strlen(pattern) - 1)
                return 1;
        }
    return 0;
}

void find(char *current_pos, char *target)
{
    int fd;
    char buf[512] = {}, *p;
    struct stat st;
    struct dirent de;

    if ((fd = open(current_pos, 0)) < 0)
    {
        fprintf(2, "this position is wrong.");
        exit(1);
    }
    if (fstat(fd, &st) < 0)
    {
        fprintf(2, "this position can not stat.");
        close(fd);
        exit(1);
    }

    switch (st.type)
    {
    case T_FILE:
        if (match(current_pos, target))
            printf("%s\n", current_pos);
        break;
    case T_DIR:
        strcpy(buf, current_pos);

        p = buf + strlen(buf);
        *p++ = '/';
        while (read(fd, &de, sizeof(de)) == sizeof(de))
        {
            if (de.inum == 0)
                continue;
            strcpy(p, de.name); // 拼接文件名
            if (strcmp(p, "..") == 0 || strcmp(p, ".") == 0)
                continue;
            find(buf, target);
        }
        break;
    }
    close(fd);
}

int main(int argc, char **argv)
{
    if (argc < 2)
    {
        printf("please give the current position and the target file name.");
        exit(1);
    }
    find(argv[1], argv[2]);
    return 0;
}
```

在`Makefile`中添加`_find`：

```
UPROGS=\
	$U/_cat\
	...
	$U/_sleep\
	$U/_pingpong\
	$U/_primes\
	$U/_find\
```

使用

```
make GRADEFLAGS=find grade
```

判断是否成功，实验结果：

```
== Test find, in current directory == find, in current directory: OK (2.0s) 
== Test find, recursive == find, recursive: OK (1.0s) 
```

### 3 ) 实验中遇到的问题和解决办法

问题1：.和..为什么会循环？

解决方法：使用printf进行debug的过程中发现了，如果出现.且让其参与递归会出现这样的情形：

```
./
./.
./../
./../.
./../../
./../../.
```

最后递归溢出报错。问题出在这一句：

```
strcpy(p, de.name); // 拼接文件名
```

需要在这之前把类似./..的情形略过。

问题2：fstat之类的结构体怎么用…

解决方法：ls的实现，vscode转到结构体定义，看源码，照葫芦画瓢，但这并不简单

### 4 ) 实验心得

难度说是moderate，但是还是做了很久，因为没有看到hint要忽略..这样循环递归的情形，一开始觉得写完了但是被xargs打回来了。

## 6. xargs ([moderate](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

你需要编写一个简单版本的UNIX `xargs`程序：它的参数描述了要运行的命令，它从标准输入读取行，并且对于每一行，它运行该命令，并将该行添加到命令的参数中。你的解决方案应该在文件`user/xargs.c`中。

### 2 ) 实验步骤

新建 user/xargs.c 文件：

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"

const int ARG_SIZE = 64;

char *getWord()
{
    char buf;
    char *ret = (char *)malloc(ARG_SIZE), *p = ret;
    while (read(0, &buf, sizeof(char)) == sizeof(char))
    {
        if (buf == ' ' || buf == '\n')
        {
            break;
        }
        else
            *p++ = buf;
    }
    *p++ = '\0';
    return ret;
}

char **getInstruction(int argc, char **argv, int mode)
{
    char **ret = (char **)malloc(MAXARG * sizeof(char *)), **p = ret + argc;
    for (int i = 0; i < argc; i++)
    {
        ret[i] = (char *)malloc(ARG_SIZE);
        strcpy(ret[i], argv[i]);
    }
    int i = 0;
    while (1)
    {
        p[i++] = getWord();
        if (strlen(p[i - 1]) == 0 || mode == i || argc + i == MAXARG)
            break;
    }
        p[i] = (char *)malloc(sizeof(char));
        p[i] = "";
    return ret; // 长度为argc + mode 或 argc 或未知
}

int getInstructionLength(char **list)
{
    char **p = list;
    while (strlen(*p++) > 0)
        ;
    return p - list - 1;
}

void freeInstruction(int length, char **list)
{
    for (int i = 0; i < length; i++)
        free(list[i]);
    free(list);
}

void printInstruction(char **list)
{
    char **p = list;
    while (strlen(*p) > 0)
    {
        fprintf(2, *p++);
        fprintf(2, " ");
    }
    fprintf(2, "\n");
}

int main(int argc, char *argv[])
{
    if (argc < 2)
    {
        fprintf(2, "please input essential args.");
        exit(1);
    }

    int flag = 1, mode, dx;
    if (argc < 3 || strcmp(argv[1], "-n") != 0)
    {
        dx = 1;
        mode = MAXARG;
    }
    else
    {
        dx = 3;
        mode = atoi(argv[2]);
    }
    // fprintf(2, "%d", mode);
    for (int i = dx; i < argc; i++)
        argv[i - dx] = argv[i];
    argc -= dx; // 处理xargs及之后参数

    while (flag)
    {
        char **list = getInstruction(argc, argv, mode);
        //printInstruction(list);
        int length = getInstructionLength(list);
        if (length != argc)
        {
            if (fork() == 0)
            {
                exec(list[0], list);
                exit(0);
            }
            else
            {
                wait(0);
            }
        }
        else
            flag = 0;
        freeInstruction(length, list);
    }
    return 0;
}
```

在`Makefile`中添加`_xargs`：

```
UPROGS=\
	$U/_cat\
	...
	$U/_sleep\
	$U/_pingpong\
	$U/_primes\
	$U/_find\
	$U/_xargs\
```

使用

```
make GRADEFLAGS=xargs grade
```

判断是否成功，实验结果：

```
== Test xargs == xargs: OK (2.0s) 
```

### 3 ) 实验中遇到的问题和解决办法

问题1：xargs到底是干什么用的？

解决方法：原来这是linux的一条指令，在网上看到可以根据标准输出管道内容作为其他指令的参数

问题2：处理命令行为什么这么麻烦？

解决方法：从标准输出中读取单词个数，还要根据个数拼成一条条新的指令再执行，十分考验字符串功底，不过后来好像发现xv6好像提供了相应的解析命令行的函数……纯纯给自己找麻烦

### 4 ) 实验心得

这条指令确实很有用，但是把字符流变成命令行参数再处理太麻烦了，没用到什么新的知识，倒是将find的输出作为echo的输入内容这我是没想到的，害我找不到到底是xargs的问题还是find的问题
