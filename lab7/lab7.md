# Lab: networking

# 1. Your Job ([hard](https://pdos.csail.mit.edu/6.828/2022/labs/guidance.html))

### 1 ) 实验目的

在这个实验中，你将为 `xv6 `操作系统编写一个网络接口卡（NIC）的设备驱动程序。在这个实验中，你将使用一个名为 `E1000 `的网络设备来处理网络通信。对于 `xv6` 操作系统（以及你编写的驱动程序），`E1000 `看起来就像是连接到真实以太网局域网（LAN）的真实硬件设备。实际上，你的驱动程序将与 `qemu` 提供的一个模拟 `E1000 `设备进行通信，而该设备又连接到 `qemu `模拟的局域网。在这个模拟的局域网上，`xv6 `操作系统（"guest"）有一个 IP 地址为 `10.0.2.15`。`qemu `还会为运行 `qemu `的计算机在局域网上分配一个 IP 地址，通常为 `10.0.2.2`。当 `xv6` 使用` E1000 `发送一个数据包到 `10.0.2.2` 时，`qemu` 将把该数据包传递给运行 `qemu` 的计算机上相应的应用程序（"host"）。

你将使用 QEMU 的 "user-mode network stack" 来进行网络通信。QEMU 的文档中有关于` user-mode `网络栈的更多信息。我们已经更新了 `Makefile`，以启用` QEMU `的 user-mode 网络栈和 `E1000 `网卡。

Makefile 配置了` QEMU`，将所有传入和传出的数据包记录到你的实验目录下的 `packets.pcap `文件中。你可以查看这些记录来确认 xv6 是否发送和接收了你预期的数据包。

==你的任务是完成 `kernel/e1000.c` 中的 `e1000_transmit() `和 `e1000_recv() `函数==，以使驱动程序能够发送和接收数据包。当 make grade 命令显示你的解决方案通过了所有测试时，你就完成了任务。

在本实验中，你需要编写一个` xv6 `的网络接口卡（NIC）设备驱动程序，用于处理网络通信。你将使用一个名为 `E1000 `的网络设备进行通信。对于` xv6`（以及你编写的驱动程序），`E1000 `看起来像连接到真实以太网局域网（LAN）的真实硬件。

请阅读 `E1000 `软件开发手册，该手册涵盖了几个相关的以太网控制器。QEMU 模拟的是 `82540EM`。先浏览第 2 章，了解这个设备的基本情况。编写驱动程序时，你需要熟悉第 3 章和第 14 章，以及第 4.1 节（不需要关注 4.1 的子节）。同时，你需要参考第 13 章。其他章节主要涵盖` E1000 `的一些高级功能，但你的驱动程序不需要与这些功能交互。初期不必担心细节，只需要了解文档的结构，以便稍后找到需要的信息。`E1000 `有很多高级特性，但只有一小部分基本功能是完成本实验所需的。

在 `e1000.c` 中，我们为你提供了 `e1000_init() `函数，它配置了` E1000`以从 `RAM` 中读取待传输的数据包，并将接收到的数据包写入 RAM。这种技术称为 DMA（直接内存访问），因为 E1000 硬件直接在 RAM 中读写数据包。

由于数据包可能以比驱动程序处理的速度更快的速度到达，`e1000_init() `为 `E1000 `提供了多个缓冲区，用于写入数据包。`E1000` 要求这些缓冲区在 RAM 中由一个数组的 "描述符" 描述，每个描述符包含一个 RAM 地址，`E1000 `可以在其中写入接收到的数据包。`struct rx_desc `描述了描述符的格式。该数组称为接收环（receive ring）或接收队列（receive queue）。在这个环中，当网卡或驱动程序到达数组的末尾时，会回到开头。`e1000_init() `使用 `mbufalloc()` 为` E1000` 分配了用于 DMA 的 mbuf 数据包缓冲区。还有一个传输环（transmit ring），驱动程序需要将要发送的数据包放入其中。`e1000_init() `配置了这两个环的大小为 `RX_RING_SIZE `和 `TX_RING_SIZE`。

当 `net.c `中的网络堆栈需要发送数据包时，它会调用 `e1000_transmit()`，并传入一个包含要发送的数据包的` mbuf`。你的传输代码必须将数据包的指针放入 TX（传输）环中的一个描述符中。`struct tx_desc `描述了描述符的格式。你需要确保每个 `mbuf `最终会被释放，但要在 `E1000 `完成传输数据包后释放（`E1000` 在描述符中设置 `E1000_TXD_STAT_DD `位来表示这一点）。

当 `E1000` 从以太网接收到每个数据包时，它将数据包通过 `DMA `复制到下一个 RX（接收）环描述符中的内存。如果 `E1000` 尚未有待处理的中断，它会请求 PLIC 在启用中断后尽快传递中断。你的 `e1000_recv() `代码必须扫描 RX 环，并通过调用 `net_rx()` 将每个新数据包的 `mbuf `传递给网络堆栈（在 `net.c` 中）。然后，你需要分配一个新的 `mbuf`，并将其放入描述符中，以便当 `E1000 `再次到达 RX 环中的那个位置时，它可以找到一个新的缓冲区，用于进行新的数据包的 `DMA`。

除了在 RAM 中读写描述符环之外，你的驱动程序还需要通过` E1000` 的内存映射控制寄存器与 `E1000 `进行交互，以检测接收到的数据包是否可用，并通知 `E1000 `驱动程序已经填充了一些用于发送数据包的 TX 描述符。全局变量` regs` 持有指向 `E1000 `的第一个控制寄存器的指针；你的驱动程序可以通过将 regs 视为数组来访问其他寄存器。你需要特别使用 `E1000_RDT `和` E1000_TDT `索引。

要测试你的驱动程序，在一个窗口中运行 make server，在另一个窗口中运行 make qemu，然后在 xv6 中运行 nettests。nettests 中的第一个测试尝试将 UDP 数据包发送到主机操作系统，地址是 make server 运行的程序。如果你还没有完成实验，E1000 驱动程序实际上不会发送数据包，所以不会发生太多事情。

完成实验后，`E1000 `驱动程序将发送数据包，QEMU 将其传递到你的主机计算机，make server 将看到数据包，并发送一个响应数据包，然后` E1000 `驱动程序和 nettests 将看到响应数据包。然而，在主机发送回复之前，它会向 xv6 发送一个` "ARP" `请求数据包，以查找其 48 位以太网地址，并期望 xv6 以 ARP 回复进行响应。完成 E1000 驱动程序的工作后，`kernel/net.c `会处理这一过程。如果一切顺利，`nettests `将打印 testing ping: OK，而 make server 将打印一条来自 xv6 的消息！

你的输出可能会略有不同，但应该包含字符串 `"ARP, Request"`、`"ARP, Reply"`、`"UDP"`、`"a.message.from.xv6" `和 `"this.is.the.host"`。

首先，在` e1000_transmit()` 和` e1000_recv()` 中添加打印语句，并运行 make server 和（在 xv6 中）nettests。通过你的打印语句，你应该能够看到 `nettests` 调用了` e1000_transmit`。

实现` e1000_transmit` 的一些建议：

1. 首先，通过读取` E1000_TDT` 控制寄存器，向` E1000 `查询它所期望的下一个包的 TX 环位置。
2. 然后检查环是否溢出。如果在索引为 `E1000_TDT` 的描述符中 `E1000_TXD_STAT_DD `没有设置，表示 E1000 还没有完成对应的上一个传输请求，因此返回一个错误。
3. 否则，使用 `mbuffree() `来释放上一个从该描述符传输的` mbuf（`如果有的话）。
4. 接下来，填充描述符。`m->head `指向内存中包的内容，`m->len` 是包的长度。设置必要的 `cmd `标志（参考 E1000 手册中的 3.3 节），并存储 `mbuf `的指针以供稍后释放。
5. 最后，通过在` E1000_TDT `上加 1 取模 `TX_RING_SIZE` 来更新环位置。
6. 如果` e1000_transmit() `成功将 `mbuf `添加到环中，则返回 0。如果失败（例如没有可用的描述符来传输 `mbuf`），则返回 -1，以便调用者知道释放`mbuf`。

实现 `e1000_recv` 的一些建议：

1. 首先，通过获取 `E1000_RDT` 控制寄存器并加一取模` RX_RING_SIZE`，查询下一个正在等待接收的包（如果有）位于的环位置。
2. 然后，通过检查描述符的状态部分中的` E1000_RXD_STAT_DD` 位，判断是否有新的包可用。如果没有，停止处理。
3. 否则，将` mbuf` 的` m->len` 更新为描述符中报告的长度。使用` net_rx()` 将 `mbuf `传递给网络堆栈。
4. 接下来，使用 `mbufalloc() `分配一个新的 `mbuf `来替换刚刚传递给 `net_rx() `的 `mbuf`。将其数据指针（`m->head`）编程为描述符。
5. 最后，将 `E1000_RDT `寄存器更新为刚刚处理过的最后一个环描述符的索引。
6. `e1000_init()` 使用` mbuf `来初始化 RX 环，你需要查看它是如何做的，并可能借用一些代码。
7. 在某一时刻，到目前为止到达的包的总数将超过环的大小（16），确保你的代码能够处理这种情况。

你需要使用锁来应对` xv6 `可能从多个进程使用 `E1000 `的情况，或者在中断到来时 xv6 可能在内核线程中使用` E1000 `的情况。

### 2 ) 实验步骤

主要实现`transmit`与`recv`和两个方法，主要是一些常用的寄存器和状态：

```
E1000_TDT : 用于transmit函数，获取数据包的索引值
E1000_RDT : 用于recv函数，获取数据包的索引值
```

索引值范围在`0`到`RX_RING_SIZE`之间

```
E1000_TXD_CMD_EOP : 用于判断数据是否在包的末尾
E1000_TXD_STAT_DD : 用于判断ring(数据来源)是否准备好
```

在`transmit`函数与`recv`函数中，我们需要更新`buf`中的`addr`，`length`属性：

```
int
e1000_transmit(struct mbuf *m)
{
  //
  // Your code here.
  //
  // the mbuf contains an ethernet frame; program it into
  // the TX descriptor ring so that the e1000 sends it. Stash
  // a pointer so that it can be freed after sending.
  //
  acquire(&(e1000_lock));
  int index = regs[E1000_TDT];  // 获取下一个传到packet的数据的索引
  if((tx_ring[index].status & E1000_TXD_STAT_DD) == 0){  // E1000_TXD_STAT_DD: 描述符完成的标志
    release(&(e1000_lock));  // 如果上一个传输还未完成，那么无法继续发送
    return -1;
  }
  if (tx_mbufs[index])
    mbuffree(tx_mbufs[index]);  // 上一次发送的数据已经完成并且可以释放

  tx_mbufs[index] = m;
  tx_ring[index].length = m->len;  // 设置当前发送描述符的长度
  tx_ring[index].addr = (uint64) m->head;  // 设置当前发送描述符的数据地址
  tx_ring[index].cmd = E1000_TXD_CMD_RS | E1000_TXD_CMD_EOP;  // 将数据加到包的末尾
  regs[E1000_TDT] = (index + 1) % TX_RING_SIZE;  // 更新寄存器 E1000_TDT
  release(&(e1000_lock));
  
  return 0;
}
```

```
static void
e1000_recv(void)
{
  //
  // Your code here.
  //
  // Check for packets that have arrived from the e1000
  // Create and deliver an mbuf for each packet (using net_rx()).
  //
  while (1) {
    int index = (regs[E1000_RDT] + 1) % RX_RING_SIZE;  // 找到接收的索引
    if ((rx_ring[index].status & E1000_RXD_STAT_DD) == 0) {  // 还未完成
      return;
    }
    rx_mbufs[index]->len = rx_ring[index].length;  // 从ring的到的数据长度写到buf
    net_rx(rx_mbufs[index]);  // 交付给上层协议处理
    // allocate a new mbuf and clear its status
    rx_mbufs[index] = mbufalloc(0);  // 下一个接收描述符分配一个新的 mbuf
    rx_ring[index].status = 0;  // 清除状态字段，下一次接收就绪
    rx_ring[index].addr = (uint64)rx_mbufs[index]->head;  // 将新的接受数据地址应用与ring
    regs[E1000_RDT] = index;  // 更新RDT
  }
}
```

使用

```
make grade
```

验证结果：

```
== Test running nettests == 
$ make qemu-gdb
(3.5s) 
== Test   nettest: ping == 
  nettest: ping: OK 
== Test   nettest: single process == 
  nettest: single process: OK 
== Test   nettest: multi-process == 
  nettest: multi-process: OK 
== Test   nettest: DNS == 
  nettest: DNS: OK 
```

### 3 ) 实验中遇到的问题和解决办法

问题：我 到 底 要 干 什 么？

解决办法：很简单啊！就是两个函数transmit和recv捏！那我到底怎么实现呢？E1000 有很多高级特性，但只有==一小部分==基本功能是完成本实验所需的。如何找到这==一小部分==呢？我曾不止一次搜索栏直接查找“Lab7 需要使用哪些寄存器”等直接字眼，我再也不想碰文档了。

### 4 ) 实验心得

材料太多了…为什么这些东西你要花这么多篇幅去讲，我本来英语就不好，机翻又丢了许多细节…在大佬手把手的指导下我终于简要的看完了该看的部分。你确定这题不是先射箭再画靶？我现在终于明白了会写linux驱动的都是人才，尤其是会写网卡驱动的，
