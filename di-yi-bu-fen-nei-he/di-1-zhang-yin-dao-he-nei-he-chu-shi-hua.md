# 第 1 章 引导和内核初始化

## 1.1. 概述

本章概述了从 BIOS（固件）POST 开始，到第一个用户进程创建的启动和系统初始化过程。由于系统启动的初始步骤非常依赖架构，本章以 IA-32 架构为例进行说明。然而，AMD64 和 ARM64 架构是更为重要和引人注目的例子，应该在不久的将来根据本文档的主题进行解释。

FreeBSD 启动过程可能出乎意料地复杂。在控制权从 BIOS 转交之后，必须进行大量的低级配置，才能加载并执行内核。此设置必须以简单且灵活的方式完成，以便用户能够进行大量的自定义操作。

## 1.2. 概述

启动过程是一个极其依赖机器的活动。编写代码时不仅需要考虑每种计算机架构，还可能存在同一架构下的多种启动方式。例如，**stand** 目录中包含了大量与架构相关的代码。每种受支持的架构都有一个对应的目录。FreeBSD 支持 CSM 启动标准（兼容性支持模块）。因此，支持 CSM（支持 GPT 和 MBR 分区）以及 UEFI 启动（完全支持 GPT，MBR 主要支持）。它还支持从 ext2fs、MSDOS、UFS 和 ZFS 加载文件。FreeBSD 还支持 ZFS 的启动环境功能，该功能允许主机操作系统传递有关启动的信息，这超越了过去简单的分区方式。但如今 UEFI 比 CSM 更为相关。下面的示例展示了如何从 MBR 分区的硬盘启动 x86 计算机，使用 FreeBSD **boot0** 多重引导加载程序，存储在第一个扇区。该引导代码启动了 FreeBSD 三阶段的引导过程。

理解此过程的关键在于它是由一系列复杂性逐渐增加的阶段组成。这些阶段包括 **boot1**、**boot2** 和 **loader**（有关更多详细信息，请参见 [boot(8)](https://man.freebsd.org/cgi/man.cgi?query=boot&sektion=8&format=html)）。启动系统按顺序执行每个阶段。最后一个阶段 **loader** 负责加载 FreeBSD 内核。以下部分将逐一讲解每个阶段。

下面是不同引导阶段生成的输出示例。实际输出可能因机器而异：



| **FreeBSD 组件** | **输出（可能有所不同）** |
| ------------------- | --------------------------- |
| `boot0`                  | <pre>F1    FreeBSD<br />F2    BSD<br />F5    Disk 2</pre>              |
| `boot2` \^[[1](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnotedef_1 "View footnote.")]\^     | <pre>>FreeBSD/x86 BOOT<br />Default: 0:ad(0p4)/boot/loader<br />boot:</pre>                  |
| 装载程序          | <pre>BTX loader 1.00 BTX version is 1.02<br />Consoles: internal video/keyboard<br />BIOS drive C: is disk0<br />BIOS 639kB/2096064kB available memory<br />FreeBSD/x86 bootstrap loader, Revision 1.1<br />Console internal video/keyboard<br />(root@releng1.nyi.freebsd.org, Fri Apr  9 04:04:45 UTC 2021)<br />Loading /boot/defaults/loader.conf<br />/boot/kernel/kernel text=0xed9008 data=0x117d28+0x176650 syms=[0x8+0x137988+0x8+0x1515f8]</pre>       |
| 内核              | <pre>Copyright (c) 1992-2021 The FreeBSD Project.<br />Copyright (c) 1979, 1980, 1983, 1986, 1988, 1989, 1991, 1992, 1993, 1994<br />        The Regents of the University of California. All rights reserved.<br />FreeBSD is a registered trademark of The FreeBSD Foundation.<br />FreeBSD 13.0-RELEASE 0 releng/13.0-n244733-ea31abc261f: Fri Apr  9 04:04:45 UTC 2021<br />    root@releng1.nyi.freebsd.org:/usr/obj/usr/src/i386.i386/sys/GENERIC i386<br />FreeBSD clang version 11.0.1 (git@github.com:llvm/llvm-project.git llvmorg-11.0.1-0-g43ff75f2c3fe)</pre>      |

## 1.3. BIOS

当计算机开机时，处理器的寄存器被设置为一些预定义的值。其中一个寄存器是 *指令指针* 寄存器，它在开机后的值是固定的：它是一个 32 位的值 `0xfffffff0`。指令指针寄存器（也称为程序计数器）指向处理器将要执行的代码。另一个重要的寄存器是 `cr0` 32 位控制寄存器，它在重启后刚开始时的值为 `0`。`cr0` 寄存器的一个位，PE（保护启用）位，表示处理器是否在 32 位保护模式下运行，还是在 16 位实模式下运行。由于此位在启动时被清除，处理器以 16 位实模式启动。实模式意味着，在此模式下，线性地址和物理地址是相同的。处理器不立即进入 32 位保护模式的原因是向后兼容性，特别是启动过程依赖于 BIOS 提供的服务，而 BIOS 本身是在传统的 16 位代码下工作的。

`0xfffffff0` 的值略小于 4 GB，因此除非机器有 4 GB 的物理内存，否则无法指向有效的内存地址。计算机的硬件会将这个地址转换，使其指向 BIOS 的内存块。

BIOS（基本输入输出系统）是主板上的一块芯片，包含相对较小的只读存储器（ROM）。这些内存包含与主板配套硬件相关的各种底层例程。处理器首先会跳转到地址 `0xfffffff0`，该地址实际上位于 BIOS 的内存中。通常，该地址包含跳转指令，跳转到 BIOS 的 POST 例程。

POST（开机自检）是一组例程，包括内存检查、系统总线检查以及其他底层初始化，以便 CPU 能够正确地设置计算机。该阶段的重要步骤是确定启动设备。现代 BIOS 实现允许选择启动设备，支持从软盘、CD-ROM、硬盘或其他设备启动。

POST 的最后一步是 `INT 0x19` 指令。`INT 0x19` 处理程序从启动设备的第一个扇区读取 512 字节数据到内存地址 `0x7c00`。*第一个扇区* 这一术语源自硬盘架构，其中磁盘被分为多个圆柱形的磁道。磁道是有编号的，每个磁道又分为多个（通常为 64）扇区。磁道编号从 0 开始，而扇区编号从 1 开始。磁道 0 是磁盘上最外层的部分，第 1 扇区具有特殊用途，也叫做主引导记录（MBR，Master Boot Record）。第一个磁道上的其余扇区从未被使用。

这个扇区是我们启动序列的起点。正如我们将看到的，这个扇区包含了我们的 **boot0** 程序。BIOS 会跳转到地址 `0x7c00`，使得该程序开始执行。

## 1.4. 主引导记录（`boot0`）

当控制权从 BIOS 转交到内存地址 `0x7c00` 时，**boot0** 开始执行。它是第一个在 FreeBSD 控制下执行的代码。**boot0** 的任务非常简单：扫描分区表并让用户选择要从哪个分区启动。分区表是一个特殊的标准数据结构，嵌入在主引导记录（MBR）中（因此也嵌入在 **boot0** 中），描述了四个标准的 PC “分区”。**boot0** 存在于文件系统中，路径为 **/boot/boot0**。它是一个 512 字节的小文件，正是 FreeBSD 安装程序在安装时选择了“引导管理器”选项时写入硬盘 MBR 的内容。实际上，**boot0** 就是 MBR。

如前所述，我们调用 BIOS 的 `INT 0x19` 指令，将 MBR（**boot0**）加载到内存地址 `0x7c00`。**boot0** 的源文件可以在 **stand/i386/boot0/boot0.S** 中找到——这是 Robert Nordier 编写的一段非常棒的代码。

从 MBR 的偏移量 `0x1be` 开始的一个特殊结构被称为 *分区表*。它包含四个记录，每个记录 16 字节，称为 *分区记录*，这些记录表示硬盘的分区情况，或者用 FreeBSD 的术语来说，就是分片。每个 16 字节中的一个字节表示该分区（分片）是否可引导。必须恰好有一个记录设置了这个标志，否则 **boot0** 的代码将拒绝继续执行。

分区记录包含以下字段：

* 1 字节的文件系统类型
* 1 字节的可引导标志
* 6 字节的 CHS 格式描述符
* 8 字节的 LBA 格式描述符

分区记录的描述符包含关于分区在磁盘上确切位置的信息。LBA 和 CHS 两种描述符提供相同的信息，但方式不同：LBA（逻辑块寻址）提供分区的起始扇区和分区的长度，而 CHS（柱面、磁头、扇区）提供分区第一个和最后一个扇区的坐标。分区表以特殊的签名 `0xaa55` 结束。

MBR 必须适合 512 字节，即一个磁盘扇区。这个程序使用了低级“技巧”，例如利用某些指令的副作用，并重用先前操作中的寄存器值，以最大限度地减少指令的使用。当处理分区表时，必须特别小心，因为它是嵌入在 MBR 本身中的。因此，在修改 **boot0.S** 时要非常小心。

请注意，**boot0.S** 源文件是“原样”汇编的：指令逐一转换为二进制，没有附加信息（例如没有 ELF 文件格式）。这种低级控制是在链接时通过传递特殊的控制标志给链接器来实现的。例如，程序的文本段被设置为位于地址 `0x600`。实际上，这意味着 **boot0** 必须被加载到内存地址 `0x600` 才能正常工作。

查看 **boot0** 的 **Makefile**（**stand/i386/boot0/Makefile**）是很有意义的，因为它定义了 **boot0** 的一些运行时行为。例如，如果通过串行端口（COM1）连接了终端进行输入输出，则必须定义宏 `SIO`（`-DSIO`）。`-DPXE` 启用通过 PXE 启动，并可以通过按 F6 启动。此外，程序还定义了一组 *标志*，允许进一步修改其行为。所有这些在 **Makefile** 中都有说明。例如，查看链接器指令，命令链接器将文本段从地址 `0x600` 开始，并“原样”构建输出文件（去掉任何文件格式）：

```asm
BOOT_BOOT0_ORG?=0x600
      ORG=${BOOT_BOOT0_ORG}
```

**stand/i386/boot0/Makefile**

现在让我们开始学习 MBR 或 **boot0**，从执行开始的地方入手。

>**注意**
>
> 为了更好地展示，某些指令已进行了一些修改。例如，扩展了一些宏，并省略了某些宏测试，当测试的结果已知时。这适用于所有显示的代码示例。 

```asm
start:
      cld			# 字符串操作递增
      xorw %ax,%ax		# 清零
      movw %ax,%es		# 设置地址
      movw %ax,%ds		# 设置数据
      movw %ax,%ss		# 设置堆栈
      movw $LOAD,%sp		# 设置堆栈指针
```

**stand/i386/boot0/boot0.S**

这段代码是程序的入口点，也是 BIOS 转交控制的地方。首先，它确保字符串操作能够自动递增其指针操作数（使用 `cld` 指令）^\[[2](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnotedef_2 "查看脚注")^。然后，由于它不假设段寄存器的状态，因此它会初始化这些寄存器。最后，它将堆栈指针寄存器（`%sp`）设置为（\$LOAD = 地址 `0x7c00`），以确保堆栈可用。

接下来的代码块负责将程序搬迁到新的地址，并跳转到搬迁后的代码。

```asm
movw %sp,%si     # 源地址
      movw $start,%di		# 目标地址
      movw $0x100,%cx		# 字长计数
      rep			# 重复
      movsw			# 复制代码
      movw %di,%bp		# 设置变量地址
      movb $0x8,%cl		# 要清零的字数
      rep			# 清零
      stosw			# 进行清零
      incb -0xe(%di)		# 将 S 字段设置为 1
      jmp main-LOAD+ORIGIN	# 跳转到搬迁后的代码
```

**stand/i386/boot0/boot0.S**

由于 **boot0** 被 BIOS 加载到地址 `0x7C00`，它将自己复制到地址 `0x600`，然后将控制权转移到该位置（记得它被链接为在地址 `0x600` 执行）。源地址 `0x7c00` 被复制到寄存器 `%si` 中，目标地址 `0x600` 被复制到寄存器 `%di` 中。复制的字数（程序的大小 = 512 字节）被复制到寄存器 `%cx` 中。接下来，`rep` 指令重复执行后续的 `movsw` 指令，重复次数由 `%cx` 寄存器的值决定。`movsw` 指令将由 `%si` 指向的字复制到由 `%di` 指向的地址。这一过程重复 255 次。每次重复后，源寄存器和目标寄存器（`%si` 和 `%di`）都会分别增加 1。因此，经过 256 字（512 字节）的复制后，`%di` 的值为 `0x600+512=0x800`，而 `%si` 的值为 `0x7c00+512=0x7e00`，完成了代码的 *搬迁*。自本文档最后一次更新以来，代码中的复制指令已发生变化，现已使用 `movsw` 和 `stosw`，这会一次复制 2 字节（1 字）。

接下来，目标寄存器 `%di` 被复制到 `%bp`，`%bp` 的值为 `0x800`。值 `8` 被复制到 `%cl`，为新的字符串操作做准备（就像前面的 `movsw`）。现在，执行 `stosw` 8 次。此指令将 `0` 值复制到目标寄存器 `%di`（即 `0x800`）所指向的地址，并自增 `%di`。这一过程重复 7 次，最终 `%di` 的值为 `0x810`。有效地，这会清除地址范围 `0x800`-`0x80f`。这个范围被用作（假的）分区表，以便将 MBR 写回到磁盘。最后，CHS 寻址的分区的扇区字段被赋值为 1，并跳转到搬迁后的代码的主函数。请注意，在跳转到搬迁后的代码之前，任何对绝对地址的引用都被避免了。

以下代码块测试是否应该使用 BIOS 提供的驱动器号，还是使用存储在 **boot0** 中的驱动器号。

```asm
main:
      testb $SETDRV,_FLAGS(%bp)	# 设置驱动器号？
#ifndef CHECK_DRIVE	/* 禁用驱动器检查 */
      jz save_curdrive		# 否，使用默认驱动器号
#else
      jnz disable_update	# 是的
      testb %dl,%dl		# 驱动器号有效？
      js save_curdrive		# 可能有效（0x80 被设置）
#endif
```

**stand/i386/boot0/boot0.S**

这段代码测试 `SETDRV` 位（`0x20`）在 *标志* 变量中的状态。记得 `%bp` 寄存器指向地址 `0x800`，因此测试是针对地址 `0x800-69=0x7bb` 处的 *标志* 变量进行的。这是对 **boot0** 可以进行的修改类型的一个示例。`SETDRV` 标志默认没有设置，但可以在 **Makefile** 中设置。当设置时，使用存储在 MBR 中的驱动器号，而不是 BIOS 提供的驱动器号。我们假设使用默认设置，并且 BIOS 提供了有效的驱动器号，因此跳转到 `save_curdrive`。

接下来的代码块将保存 BIOS 提供的驱动器号，并调用 `putn` 来打印一个换行符。

```asm
save_curdrive:
      movb %dl, (%bp)		# 保存驱动器号
      pushw %dx			# 也保存到堆栈
#ifdef	TEST	/* 测试代码，打印内部 BIOS 驱动器 */
      rolb $1, %dl
      movw $drive, %si
      call putkey
#endif
      callw putn		# 打印换行符
```

**stand/i386/boot0/boot0.S**

请注意，我们假设 `TEST` 没有被定义，因此其中的条件代码不会被汇编，也不会出现在我们编译后的 **boot0** 可执行文件中。

接下来的代码块实现了实际的分区表扫描。它将每个分区表项的类型打印到屏幕上，并将其与已知操作系统文件系统的列表进行比较。常见的分区类型包括 NTFS（Windows®，ID 0x7）、`ext2fs`（Linux®，ID 0x83），当然还有 `ffs`/`ufs2`（FreeBSD，ID 0xa5）。该实现相对简单。

```asm
movw $(partbl+0x4),%bx	# 分区表 (+4)
      xorw %dx,%dx		# 项目编号

read_entry:
      movb %ch,-0x4(%bx)	# 清除活动标志（ch == 0）
      btw %dx,_FLAGS(%bp)	# 项目启用？
      jnc next_entry		# 否
      movb (%bx),%al		# 加载类型
      test %al, %al		# 跳过空分区
      jz next_entry
      movw $bootable_ids,%di	# 查找表
      movb $(TLEN+1),%cl	# 条目数量
      repne			# 定位
      scasb			#  类型
      addw $(TLEN-1), %di	# 调整
      movb (%di),%cl		# 分区
      addw %cx,%di		#  描述
      callw putx		# 显示

next_entry:
      incw %dx			# 下一项
      addb $0x10,%bl		# 下一个条目
      jnc read_entry		# 直到完成
```

**stand/i386/boot0/boot0.S**

需要注意的是，每个条目的活动标志都被清除，因此扫描完成后，在 **boot0** 的内存副本中 *没有* 分区条目是活动的。稍后，活动标志将被设置为所选分区。这确保了只有一个活动分区存在，如果用户选择将更改写回磁盘。

接下来的代码块测试其他驱动器。在启动时，BIOS 将计算机中存在的驱动器数量写入地址 `0x475`。如果存在其他驱动器，**boot0** 会将当前驱动器打印到屏幕上。用户稍后可以命令 **boot0** 扫描另一个驱动器上的分区。

```asm
popw %ax			# 驱动器号
      subb $0x80-0x1,%al		# 下一个驱动器？
      cmpb NHRDRV,%al		#  驱动器存在？（来自 BIOS？）
      jb print_drive		# 是
      decw %ax			# 已经是驱动器 0？
      jz print_prompt		# 是
```

**stand/i386/boot0/boot0.S**

我们假设存在单一驱动器，因此不会跳转到 `print_drive`。我们还假设没有发生任何异常，因此跳转到 `print_prompt`。

接下来的代码块只是输出一个提示符，后面跟着默认选项：

```asm
print_prompt:
      movw $prompt,%si		# 显示
      callw putstr		#  提示符
      movb _OPT(%bp),%dl	# 显示
      decw %si			#  默认
      callw putkey		#  按键
      jmp start_input		# 跳过蜂鸣
```
**stand/i386/boot0/boot0.S**

最后，跳转到 `start_input`，在这里使用 BIOS 服务启动定时器并读取用户的键盘输入；如果定时器到期，则会选择默认选项：

```asm
start_input:
      xorb %ah,%ah		# BIOS: 获取
      int $0x1a			#  系统时间
      movw %dx,%di		# 计时器
      addw _TICKS(%bp),%di	#  超时
read_key:
      movb $0x1,%ah		# BIOS: 检查
      int $0x16			#  是否按键
      jnz got_key		# 已输入
      xorb %ah,%ah		# BIOS: int 0x1a, 00
      int $0x1a			#  获取系统时间
      cmpw %di,%dx		# 超时？
      jb read_key		# 否
```

**stand/i386/boot0/boot0.S**

请求中断 `0x1a`，并将参数 `0` 放入寄存器 `%ah`。BIOS 有一套预定义的服务，应用程序通过软件生成的中断 `int` 指令请求这些服务，并将参数传递到寄存器（在这里是 `%ah`）。在此，我们请求自午夜以来的时钟滴答数；该值由 BIOS 通过 RTC（实时时钟）计算。这个时钟可以被设置为 2 Hz 到 8192 Hz 的频率，BIOS 在启动时设置为 18.2 Hz。当请求完成时，BIOS 会将 32 位结果返回到寄存器 `%cx` 和 `%dx`（其中 `%dx` 的低字节存储了结果）。此结果（即 `%dx` 部分）被复制到寄存器 `%di`，然后将 `TICKS` 变量的值加到 `%di` 中。该变量存储在 **boot0** 中，位于寄存器 `%bp`（指向 `0x800`）偏移 `_TICKS` 处（这是一个负值）。该变量的默认值是 `0xb6`（十进制 182）。此时，**boot0** 会不断请求 BIOS 获取时间，并且当寄存器 `%dx` 返回的值大于存储在 `%di` 中的值时，时间到期，将选择默认选项。由于 RTC 每秒滴答 18.2 次，因此此条件将在 10 秒后满足（这个默认行为可以在 **Makefile** 中更改）。在这段时间内，**boot0** 会不断通过 `int 0x16` 和参数 `1` 在 `%ah` 中向 BIOS 查询用户输入。

无论是按下了键，还是时间到期，接下来的代码都会验证选择。根据选择，寄存器 `%si` 会指向分区表中的相应条目。这个新的选择会覆盖先前的默认选择，实际上，它会成为新的默认选择。最后，选定分区的 ACTIVE 标志被设置。如果在编译时启用了该功能，带有这些修改值的 **boot0** 内存副本将被写回到磁盘上的 MBR。我们将此实现的细节留给读者。

最后，我们通过以下代码块结束对 **boot0** 程序的研究：

```asm
movw $LOAD,%bx		# 读取地址
      movb $0x2,%ah		# 读取扇区
      callw intx13		#  从磁盘读取
      jc beep			# 如果错误
      cmpw $MAGIC,0x1fe(%bx)	# 可引导？
      jne beep			# 否
      pushw %si			# 保存选定分区的指针
      callw putn		# 留出空间
      popw %si			# 恢复，下一阶段使用
      jmp *%bx			# 调用引导加载程序
```

**stand/i386/boot0/boot0.S**

回顾一下，寄存器 `%si` 指向所选的分区条目。该条目告诉我们该分区在磁盘上的起始位置。我们假设，当然，所选的分区实际上是一个 FreeBSD 切片（slice）。

>**注意**
>
> 从现在开始，我们将优先使用技术上更准确的术语“切片”（slice）而不是“分区”（partition）。

传输缓冲区被设置为 `0x7c00`（寄存器 `%bx`），并请求通过调用 `intx13` 读取 FreeBSD 切片的第一个扇区。我们假设一切正常，因此不会跳转到 `beep`。特别地，新读取的扇区必须以魔法序列 `0xaa55` 结束。最后，寄存器 `%si` 中的值（指向选定分区表的指针）被保存，以供下一阶段使用，然后跳转到地址 `0x7c00`，在该位置启动我们刚刚读取的代码块的执行。

## 1.5. `boot1` 阶段

到目前为止，我们已经经历了以下的引导过程：

* BIOS 完成了早期的硬件初始化，包括自检（POST）。MBR（**boot0**）被从磁盘的绝对扇区一加载到地址 `0x7c00`。控制权被传递到该位置。
* **boot0** 将自己重新定位到它被链接执行的位置（`0x600`），然后跳转到适当的位置继续执行。最后，**boot0** 从 FreeBSD 切片加载了第一个磁盘扇区到地址 `0x7c00`。控制权被传递到该位置。

**boot1** 是引导加载序列中的下一步，它是三个引导阶段中的第一个。需要注意的是，我们一直在处理磁盘扇区。实际上，BIOS 加载了绝对的第一个扇区，而 **boot0** 加载了 FreeBSD 切片的第一个扇区。这两个加载都发生在地址 `0x7c00`。我们可以从概念上将这些磁盘扇区视为包含 **boot0** 和 **boot1** 的文件，但实际上对于 **boot1** 来说，这并不完全准确。严格来说，和 **boot0** 不同，**boot1** 并不是引导块的一部分 ^\[[3](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnotedef_3 "查看脚注")]^。相反，一个完整的文件 **boot**（即 **/boot/boot**）最终被写入磁盘。这个文件结合了 **boot1**、**boot2** 和 `Boot Extender`（或 BTX）。这个单一文件的大小大于单个扇区（大于 512 字节）。幸运的是，**boot1** 占据了这个文件的前 512 字节，因此当 **boot0** 加载 FreeBSD 切片的第一个扇区（512 字节）时，它实际上是加载了 **boot1** 并将控制权传递给它。

**boot1** 的主要任务是加载下一个引导阶段。下一个阶段更加复杂，它由一个名为 "Boot Extender"（BTX）的服务器和一个名为 **boot2** 的客户端组成。正如我们将看到的，最后的引导阶段 **loader** 也是 BTX 服务器的一个客户端。

现在，让我们详细看看 **boot1** 做了什么，首先从它的入口点开始，就像我们分析 **boot0** 一样：

```asm
start:
    jmp main
```

**stand/i386/boot2/boot1.S**

在 `start` 处的入口点仅仅是跳过一个特殊的数据区域，跳到标签 `main`，后者的内容如下：

```asm
main:
      cld			# 字符串操作递增
      xor %cx,%cx		# 清零
      mov %cx,%es		# 地址
      mov %cx,%ds		# 数据
      mov %cx,%ss		# 设置堆栈
      mov $start,%sp		# 堆栈指针
      mov %sp,%si		# 源
      mov $MEM_REL,%di		# 目标
      incb %ch			# 字节计数
      rep			# 复制
      movsw			# 代码
```

**stand/i386/boot2/boot1.S**

和 **boot0** 一样，这段代码将 **boot1** 重新定位到内存地址 `0x700`。然而，不像 **boot0**，它并没有跳转到那个位置。**boot1** 被链接到地址 `0x7c00` 执行，实际上是它最初被加载的位置。关于这种重新定位的原因，我们稍后将讨论。

接下来是一个循环，用于查找 FreeBSD 切片。尽管 **boot0** 已经从 FreeBSD 切片加载了 **boot1**，但它并没有传递关于该切片的信息 ^\[[4](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnotedef_4 "查看脚注")]^，因此 **boot1** 必须重新扫描分区表以找出 FreeBSD 切片的起始位置。因此，它重新读取 MBR：

```asm
mov $part4,%si		# 分区
      cmpb $0x80,%dl		# 硬盘？
      jb main.4			# 否
      movb $0x1,%dh		# 块计数
      callw nread		# 读取 MBR
```

**stand/i386/boot2/boot1.S**

在上面的代码中，寄存器 `%dl` 保持着关于启动设备的信息。这些信息由 BIOS 传递并由 MBR 保留。`0x80` 及以上的数字表示我们正在处理硬盘，因此会调用 `nread` 来读取 MBR。`nread` 的参数通过 `%si` 和 `%dh` 传递。标签 `part4` 处的内存地址被复制到 `%si`。该内存地址保存了一个“伪分区”，由 `nread` 使用。以下是伪分区中的数据：

```asm
part4:
	.byte 0x80, 0x00, 0x01, 0x00
	.byte 0xa5, 0xfe, 0xff, 0xff
	.byte 0x00, 0x00, 0x00, 0x00
	.byte 0x50, 0xc3, 0x00, 0x00
```

**stand/i386/boot2/boot1.S**


特别地，这个伪分区的 LBA 被硬编码为零。这被用作传递给 BIOS 的参数，用来读取硬盘的绝对扇区一。或者，也可以使用 CHS 定址，在这种情况下，伪分区包含了柱面 0、磁头 0 和扇区 1，这相当于绝对扇区一。

接下来，我们来看一下 `nread` 的实现：

```
nread:
      mov $MEM_BUF,%bx		# 转移缓冲区
      mov 0x8(%si),%ax		# 获取
      mov 0xa(%si),%cx		# LBA
      push %cs			# 从
      callw xread.1		# 读取磁盘
      jnc return		# 如果成功，返回
```

**stand/i386/boot2/boot1.S**

回想一下，`%si` 指向伪分区。偏移量 `0x8` 处的字节被复制到寄存器 `%ax`，偏移量 `0xa` 处的字节被复制到 `%cx`。它们被 BIOS 解释为表示要读取的 LBA 的低 4 字节值（高四个字节假定为零）。寄存器 `%bx` 保存着 MBR 将被加载到的内存地址。将 `%cs` 压入堆栈这一指令非常有趣。在这种情况下，它没有实际效果。然而，正如我们稍后将看到的，**boot2** 与 BTX 服务器一起也使用了 `xread.1`。这一机制将在下一节讨论。

`xread.1` 的代码进一步调用了 `read` 函数，实际上是向 BIOS 请求读取磁盘扇区：

```asm
xread.1:
	pushl $0x0		# 绝对地址
	push %cx		# 块
	push %ax		# 数量
	push %es		# 地址
	push %bx		# 转移缓冲区
	xor %ax,%ax		# 块数量
	movb %dh,%al		# 传输块数量
	push %ax		# 传输块
	push $0x10		# 数据包大小
	mov %sp,%bp		# 数据包指针
	callw read		# 从磁盘读取
	lea 0x10(%bp),%sp	# 清空堆栈
	lret			# 返回给远端调用者
```

**stand/i386/boot2/boot1.S**

请注意这段代码结束时的 `lret` 指令。这个指令弹出了 `nread` 压入的 `%cs` 寄存器，然后返回。最终，`nread` 也返回。

在 MBR 加载到内存后，实际查找 FreeBSD 切片的循环开始了：

```asm
mov $0x1,%cx		 # 两次遍历
main.1:
	mov $MEM_BUF+PRT_OFF,%si # 分区表
	movb $0x1,%dh		 # 分区
main.2:
	cmpb $PRT_BSD,0x4(%si)	 # 我们的分区类型？
	jne main.3		 # 否
	jcxz main.5		 # 如果是第二次遍历
	testb $0x80,(%si)	 # 是否激活？
	jnz main.5		 # 是
main.3:
	add $0x10,%si		 # 下一个条目
	incb %dh		 # 分区号递增
	cmpb $0x1+PRT_NUM,%dh		 # 在表中？
	jb main.2		 # 是
	dec %cx			 # 遍历两次
	jcxz main.1		 # 继续第二次遍历
```

**stand/i386/boot2/boot1.S**

如果找到了 FreeBSD 切片，执行将继续到 `main.5`。注意，当找到 FreeBSD 切片时，`%si` 指向分区表中的相应条目，`%dh` 保存了分区号。我们假设找到了 FreeBSD 切片，因此继续执行 `main.5`：

```asm
main.5:
	mov %dx,MEM_ARG			   # 保存参数
	movb $NSECT,%dh			   # 扇区计数
	callw nread			   # 读取磁盘
	mov $MEM_BTX,%bx			   # BTX
	mov 0xa(%bx),%si		   # 获取 BTX 长度并设置
	add %bx,%si			   # 使 %si 指向 boot2.bin 开始处
	mov $MEM_USR+SIZ_PAG*2,%di			   # 客户端页面 2
	mov $MEM_BTX+(NSECT-1)*SIZ_SEC,%cx			   # 字节
	sub %si,%cx			   # 计数
	rep				   # 重新定位
	movsb				   # 客户端
```

**stand/i386/boot2/boot1.S**

请记住，此时寄存器 `%si` 指向 MBR 分区表中的 FreeBSD 切片条目，因此对 `nread` 的调用将有效地读取此分区开始处的扇区。传递给寄存器 `%dh` 的参数告诉 `nread` 读取 16 个磁盘扇区。回想一下，FreeBSD 切片的前 512 字节（即第一个扇区）与 **boot1** 程序相对应。还要记住，写入 FreeBSD 切片开头的文件不是 **/boot/boot1**，而是 **/boot/boot**。我们来看一下这些文件在文件系统中的大小：

```asm
-r--r--r--  1 root  wheel   512B Jan  8 00:15 /boot/boot0
-r--r--r--  1 root  wheel   512B Jan  8 00:15 /boot/boot1
-r--r--r--  1 root  wheel   7.5K Jan  8 00:15 /boot/boot2
-r--r--r--  1 root  wheel   8.0K Jan  8 00:15 /boot/boot
```

**boot0** 和 **boot1** 都是 512 字节，所以它们正好适合一个磁盘扇区。**boot2** 要大得多，包含了 BTX 服务器和 **boot2** 客户端。最后，名为 **boot** 的文件比 **boot2** 大 512 字节。这个文件是 **boot1** 和 **boot2** 的串联。正如前面所述，**boot0** 是写入绝对第一个磁盘扇区（MBR）中的文件，而 **boot** 是写入 FreeBSD 切片第一个扇区中的文件；**boot1** 和 **boot2** 并没有直接写入磁盘。用于将 **boot1** 和 **boot2** 串联成一个 **boot** 的命令是 `cat boot1 boot2 > boot`。

因此，**boot1** 正好占据了 **boot** 的前 512 字节，并且因为 **boot** 被写入 FreeBSD 切片的第一个扇区，所以 **boot1** 完全适合这个第一个扇区。当 `nread` 读取 FreeBSD 切片的前 16 个扇区时，它实际上读取了整个 **boot** 文件 ^\[[6](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnotedef_6 "View footnote.")]。我们将在下一节看到更多关于 **boot** 如何由 **boot1** 和 **boot2** 组成的细节。

回想一下，`nread` 使用内存地址 `0x8c00` 作为转移缓冲区来存储读取的扇区。这个地址选择得非常方便。实际上，因为 **boot1** 属于前 512 字节，它正好位于地址范围 `0x8c00`-`0x8dff` 中。接下来的 512 字节（地址范围 `0x8e00`-`0x8fff`）用来存储 *bsdlabel* ^\[[7](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnotedef_7 "View footnote.")]。

从地址 `0x9000` 开始是 BTX 服务器的起始位置，紧接着是 **boot2** 客户端。BTX 服务器充当内核，并以受保护模式在最高特权级别下执行。相反，BTX 客户端（例如 **boot2**）以用户模式执行。我们将在下一节中详细讨论如何实现这一点。调用 `nread` 后的代码会定位 **boot2** 在内存缓冲区中的起始位置，并将其复制到内存地址 `0xc000`。这是因为 BTX 服务器将 **boot2** 安排在从 `0xa000` 开始的段中。我们将在接下来的章节中详细探讨这一点。

**boot1** 的最后一段代码启用对 1MB 以上内存的访问 ^\[[8](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnotedef_8 "View footnote.")]，并以跳转到 BTX 服务器的起始点作为结束：

```asm
seta20:
	cli			# 禁用中断
seta20.1:
	dec %cx			# 超时？
	jz seta20.3		# 是

	inb $0x64,%al		# 获取状态
	testb $0x2,%al		# 繁忙？
	jnz seta20.1		# 是
	movb $0xd1,%al		# 命令：写入
	outb %al,$0x64		# 输出端口
seta20.2:
	inb $0x64,%al		# 获取状态
	testb $0x2,%al		# 繁忙？
	jnz seta20.2		# 是
	movb $0xdf,%al		# 启用
	outb %al,$0x60		# A20
seta20.3:
	sti			# 启用中断
	jmp 0x9010		# 启动 BTX
```

**stand/i386/boot2/boot1.S**

请注意，在跳转之前，中断已被启用。

## 1.6. BTX 服务器

接下来，在我们的启动序列中是 BTX 服务器。让我们快速回顾一下我们是如何到达这里的：

* BIOS 将绝对扇区一（MBR，或 **boot0**）加载到地址 `0x7c00` 并跳转到该地址。
* **boot0** 将自身重新定位到 `0x600`（它被链接执行的地址），然后跳转过去。接着，它读取 FreeBSD 分区的第一个扇区（包含 **boot1**）到地址 `0x7c00`，并跳转过去。
* **boot1** 将 FreeBSD 分区的前 16 个扇区加载到地址 `0x8c00`。这 16 个扇区或 8192 字节构成了整个 **boot** 文件。该文件是 **boot1** 和 **boot2** 的连接体。**boot2** 进一步包含了 BTX 服务器和 **boot2** 客户端。最后，跳转到地址 `0x9010`，即 BTX 服务器的入口点。

在详细研究 BTX 服务器之前，我们先回顾一下如何创建单一的、集成的 **boot** 文件。**boot** 文件的构建方式在其 **Makefile**（**stand/i386/boot2/Makefile**）中定义。我们来看一下创建 **boot** 文件的规则：

```c
boot: boot1 boot2
	cat boot1 boot2 > boot
```

**stand/i386/boot2/Makefile**

这告诉我们，**boot1** 和 **boot2** 是必需的，规则简单地将它们连接起来生成一个名为 **boot** 的单一文件。创建 **boot1** 的规则也很简单：

```c
boot1: boot1.out
	${OBJCOPY} -S -O binary boot1.out ${.TARGET}

      boot1.out: boot1.o
	${LD} ${LD_FLAGS} -e start --defsym ORG=${ORG1} -T ${LDSCRIPT} -o ${.TARGET} boot1.o
```

**stand/i386/boot2/Makefile**

要应用创建 **boot1** 的规则，必须先解决 **boot1.out**。这反过来依赖于 **boot1.o** 的存在。这个文件就是将我们熟悉的 **boot1.S** 汇编后的结果，且没有进行链接。现在，创建 **boot1.out** 的规则被应用。这告诉我们，**boot1.o** 应该使用 `start` 作为入口点，并从地址 `0x7c00` 开始链接。最终，**boot1** 是从 **boot1.out** 创建的，通过应用适当的规则。这个规则就是对 **boot1.out** 使用 **objcopy** 命令。请注意传给 **objcopy** 的标志：`-S` 告诉它去除所有的重定位和符号信息；`-O binary` 指示输出格式是简单的、未格式化的二进制文件。

有了 **boot1**，我们再看看 **boot2** 是如何构建的：

```c
boot2: boot2.ld
	@set -- `ls -l ${.ALLSRC}`; x=$$((${BOOT2SIZE}-$$5)); \
	    echo "$$x bytes available"; test $$x -ge 0
	${DD} if=${.ALLSRC} of=${.TARGET} bs=${BOOT2SIZE} conv=sync

      boot2.ld: boot2.ldr boot2.bin ${BTXKERN}
	btxld -v -E ${ORG2} -f bin -b ${BTXKERN} -l boot2.ldr \
	    -o ${.TARGET} -P 1 boot2.bin

      boot2.ldr:
	${DD} if=/dev/zero of=${.TARGET} bs=512 count=1

      boot2.bin: boot2.out
	${OBJCOPY} -S -O binary boot2.out ${.TARGET}

      boot2.out: ${BTXCRT} boot2.o sio.o ashldi3.o
	${LD} ${LD_FLAGS} --defsym ORG=${ORG2} -T ${LDSCRIPT} -o ${.TARGET} ${.ALLSRC}

      boot2.h: boot1.out
	${NM} -t d ${.ALLSRC} | awk '/([0-9])+ T xread/ \
	    { x = $$1 - ORG1; \
	    printf("#define XREADORG %#x\n", REL1 + x) }' \
	    ORG1=`printf "%d" ${ORG1}` \
	    REL1=`printf "%d" ${REL1}` > ${.TARGET}
```

**stand/i386/boot2/Makefile**

构建 **boot2** 的机制要复杂得多。我们来指出其中最相关的内容。依赖关系列表如下：

```c
boot2: boot2.ld
      boot2.ld: boot2.ldr boot2.bin ${BTXDIR}
      boot2.bin: boot2.out
      boot2.out: ${BTXDIR} boot2.o sio.o ashldi3.o
      boot2.h: boot1.out
```

**stand/i386/boot2/Makefile**

注意，最初并没有头文件 **boot2.h**，但是它的创建依赖于我们已经拥有的 **boot1.out**。创建它的规则有点简略，但重要的是，输出的 **boot2.h** 文件内容大致如下：

```c
#define XREADORG 0x725
```

**stand/i386/boot2/boot2.h**

回想一下，**boot1** 是被重新定位的（即，从 `0x7c00` 复制到 `0x700`）。这个重新定位现在变得有意义，因为正如我们将看到的，BTX 服务器回收了一些内存，包括原来 **boot1** 被加载的空间。然而，BTX 服务器需要访问 **boot1** 中的 `xread` 函数；根据 **boot2.h** 的输出，该函数位于地址 `0x725`。实际上，BTX 服务器正是从 **boot1** 被重新定位的代码中使用 `xread` 函数。这个函数现在可以从 **boot2** 客户端中访问。

接下来的规则指导链接器链接多个文件（**ashldi3.o**、**boot2.o** 和 **sio.o**）。请注意，输出文件 **boot2.out** 被链接为在地址 `0x2000` 执行（\${ORG2}）。回想一下，**boot2** 将在用户模式下执行，在 BTX 服务器设置的特殊用户段中。这个段从 `0xa000` 开始。此外，记住 **boot2** 部分已经被复制到地址 `0xc000`，即从用户段的起始位置偏移了 `0x2000`，因此，当我们将控制权转交给它时，**boot2** 将能够正常工作。接下来，**boot2.bin** 是从 **boot2.out** 生成的，通过去除其符号和格式信息；**boot2.bin** 是一个 *原始* 二进制文件。现在，请注意，文件 **boot2.ldr** 是一个 512 字节的全零文件。这个空间预留给 bsdlabel。

现在，我们已经有了 **boot1**、**boot2.bin** 和 **boot2.ldr** 文件，剩下的就是 BTX 服务器，才能完成创建全功能 **boot** 文件的过程。BTX 服务器位于 **stand/i386/btx/btx**，它有自己的 **Makefile** 和一套构建规则。需要注意的是，BTX 服务器也是作为一个 *raw* 二进制文件编译的，并且它被链接到地址 `0x9000` 执行。详细信息可以在 **stand/i386/btx/btx/Makefile** 中找到。

拥有了构成 **boot** 程序的文件后，最后一步是将它们 *合并*。这通过一个名为 **btxld** 的特殊程序来完成（源代码位于 **/usr/src/usr.sbin/btxld**）。该程序的一些参数包括输出文件名（**boot**）、入口点 (`0x2000`) 和文件格式（原始二进制）。这些文件最终由该工具合并成 **boot** 文件，这个文件包含了 **boot1**、**boot2**、`bsdlabel` 和 BTX 服务器。这个文件大小正好为 16 个扇区，或 8192 字节，实际在安装过程中写入 FreeBSD 分区的开头。接下来，我们将研究 BTX 服务器程序。

BTX 服务器准备了一个简单的环境，并在将控制权交给客户端之前，从 16 位实模式切换到 32 位保护模式。这个过程包括初始化和更新以下数据结构：

* 修改 `中断向量表（IVT）`。IVT 提供了实模式代码的异常和中断处理程序。
* 创建 `中断描述符表（IDT）`。为处理器异常、硬件中断、两个系统调用和 V86 接口提供了条目。IDT 为保护模式代码提供异常和中断处理程序。
* 创建一个 `任务状态段（TSS）`。这是必要的，因为处理器在执行客户端（**boot2**）时处于 *最不特权级*，而在执行 BTX 服务器时处于 *最高特权级*。
* 设置 GDT（全局描述符表）。为监督代码和数据、用户代码和数据，以及实模式代码和数据提供了条目。

接下来，我们将开始研究实际的实现。回想一下，**boot1** 跳转到地址 `0x9010`，即 BTX 服务器的入口点。在研究程序执行之前，请注意，BTX 服务器在其入口点之前（即地址范围 `0x9000-0x900f`）有一个特殊的头部。这个头部定义如下：

```asm
start:						# 代码起始位置
/*
 * BTX 头部。
 */
btx_hdr:	.byte 0xeb			# 机器 ID
		.byte 0xe			# 头部大小
		.ascii "BTX"			# 魔数
		.byte 0x1			# 主版本
		.byte 0x2			# 次版本
		.byte BTX_FLAGS			# 标志
		.word PAG_CNT-MEM_ORG>>0xc	# 分页控制
		.word break-start		# 代码大小
		.long 0x0			# 入口地址
```

**stand/i386/btx/btx/btx.S**

注意前两个字节是 `0xeb` 和 `0xe`。在 IA-32 架构中，这两个字节被解释为一个相对跳转，跳过头部直接进入入口点。因此，理论上 **boot1** 可以直接跳到这里（地址 `0x9000`），而不是跳到地址 `0x9010`。请注意，BTX 头部的最后一个字段是指向客户端（**boot2**）入口点的指针。这个字段在链接时会被修补。

紧接着头部的是 BTX 服务器的入口点：

```asm
/*
 * 初始化例程。
 */
init:		cli				# 禁用中断
		xor %ax,%ax			# 清零/段寄存器
		mov %ax,%ss			# 设置堆栈段
		mov $MEM_ESP0,%sp		# 设置堆栈指针
		mov %ax,%es			# 设置额外段
		mov %ax,%ds			# 设置数据段
		pushl $0x2			# 清除标志
		popfl				# 恢复标志
```

**stand/i386/btx/btx/btx.S**

这段代码禁用中断，设置一个工作堆栈（从地址 `0x1800` 开始），并清除 EFLAGS 寄存器中的标志。请注意，`popfl` 指令会从堆栈中弹出一个双字（4 字节），并将其放入 EFLAGS 寄存器。由于实际弹出的值为 `2`，所以 EFLAGS 寄存器会被清除（IA-32 要求 EFLAGS 寄存器的第 2 位始终为 1）。

接下来的代码块清除（设置为 `0`）内存范围 `0x5e00-0x8fff`。这个范围是用来创建各种数据结构的：

```asm
/*
 * 初始化内存。
 */
		mov $MEM_IDT,%di		# 要初始化的内存
		mov $(MEM_ORG-MEM_IDT)/2,%cx	# 要清零的字数
		rep				# 清零操作
		stosw				# 存储字
```

**stand/i386/btx/btx/btx.S**


回想一下，**boot1** 最初加载到地址 `0x7c00`，因此，在这次内存初始化之后，那个副本实际上已经消失了。然而，**boot1** 也被重定位到 `0x700`，所以 *那个* 副本仍然在内存中，并且 BTX 服务器将会使用它。

接下来，实模式的中断向量表（IVT）被更新。IVT 是一个包含异常和中断处理程序的段/偏移对数组。BIOS 通常将硬件中断映射到中断向量 `0x8` 到 `0xf` 以及 `0x70` 到 `0x77`，但正如我们将看到的那样，8259A 可编程中断控制器（控制硬件中断映射的芯片）被编程为将这些中断向量从 `0x8-0xf` 映射到 `0x20-0x27`，并将 `0x70-0x77` 映射到 `0x28-0x2f`。因此，中断处理程序被提供给中断向量 `0x20-0x2f`。之所以不直接使用 BIOS 提供的处理程序，是因为它们只能在 16 位实模式下工作，而不能在 32 位保护模式下工作。处理器模式将很快切换到 32 位保护模式。然而，BTX 服务器设置了一种机制，实际上利用 BIOS 提供的处理程序：

```asm
/*
 * 更新实模式 IDT 以反映硬件中断。
 */
		mov $intr20,%bx			# 第一个处理程序的地址
		mov $0x10,%cx			# 处理程序数量
		mov $0x20*4,%di			# 第一个实模式 IDT 条目
init.0:		mov %bx,(%di)			# 存储 IP
		inc %di				# 地址下一个条目
		inc %di				# 条目
		stosw				# 存储 CS
		add $4,%bx			# 下一个处理程序
		loop init.0			# 下一个 IRQ
```

**stand/i386/btx/btx/btx.S**

接下来的代码块创建了 IDT（中断描述符表）。IDT 在保护模式下类似于实模式下的 IVT。也就是说，IDT 描述了处理器在执行保护模式下的各种异常和中断处理程序。它本质上也是一个段/偏移对的数组，尽管结构稍微复杂一些，因为在保护模式下，段与实模式下的不同，并且应用了各种保护机制：

```asm
/*
 * 创建 IDT。
 */
		mov $MEM_IDT,%di		# IDT 的地址
		mov $idtctl,%si			# 控制字符串
init.1:		lodsb				# 获取条目
		cbw				# 转换为字节
		xchg %ax,%cx			# 交换字节
		jcxz init.4			# 如果完成，跳转
		lodsb				# 获取段
		xchg %ax,%dx			# P:DPL:type
		lodsw				# 获取控制
		xchg %ax,%bx			# 设置
		lodsw				# 获取处理程序偏移
		mov $SEL_SCODE,%dh		# 设置段选择符
init.2:		shr %bx				# 处理此中断？
		jnc init.3			# 如果不需要，跳过
		mov %ax,(%di)			# 设置处理程序偏移
		mov %dh,0x2(%di)		# 以及选择符
		mov %dl,0x5(%di)		# 设置 P:DPL:type
		add $0x4,%ax			# 下一个处理程序
init.3:		lea 0x8(%di),%di		# 下一个条目
		loop init.2			# 直到完成
		jmp init.1			# 继续
```

**stand/i386/btx/btx/btx.S**

IDT 中的每个条目长 8 字节。除了段/偏移信息外，它们还描述了段的类型、特权级别，以及段是否存在于内存中。构建方式是，中断向量 `0` 到 `0xf`（异常）由函数 `intx00` 处理；中断向量 `0x10`（也是异常）由 `intx10` 处理；硬件中断，从中断向量 `0x20` 到 `0x2f`，由函数 `intx20` 处理。最后，中断向量 `0x30`，用于系统调用，由 `intx30` 处理，而向量 `0x31` 和 `0x32` 由 `intx31` 处理。需要注意的是，只有中断向量 `0x30`、`0x31` 和 `0x32` 的描述符被设置为特权级别 3，这与 **boot2** 客户端的特权级别相同，这意味着客户端可以通过 `int` 指令执行软件生成的中断来访问这些向量而不失败（这就是 **boot2** 使用 BTX 服务器提供的服务的方式）。同时，只有软件生成的中断才会在较低特权级别的代码执行时得到保护。硬件生成的中断和处理器生成的异常始终得到适当的处理，无论实际的特权级别如何。

接下来的步骤是初始化 TSS（任务状态段）。TSS 是一种硬件特性，帮助操作系统或执行软件通过进程抽象实现多任务功能。IA-32 架构要求在使用多任务功能或定义不同特权级别时，必须至少创建一个 TSS。由于 **boot2** 客户端在特权级别 3 中执行，而 BTX 服务器在特权级别 0 中运行，因此必须定义一个 TSS：

```asm
/*
 * 初始化 TSS。
 */
init.4:		movb $_ESP0H,TSS_ESP0+1(%di)	# 设置 ESP0
		movb $SEL_SDATA,TSS_SS0(%di)	# 设置 SS0
		movb $_TSSIO,TSS_MAP(%di)	# 设置 I/O 位图基址
```

**stand/i386/btx/btx/btx.S**


请注意，TSS 中为特权级 0 的堆栈指针和堆栈段设置了一个值。因为如果在特权级 3 执行 **boot2** 时接收到中断或异常，处理器会自动切换到特权级 0，所以需要新的工作堆栈。最后，TSS 的 I/O 映射基地址字段被赋予一个值，该值是从 TSS 开始到 I/O 权限位图和中断重定向位图的 16 位偏移量。

在创建 IDT 和 TSS 后，处理器准备好切换到保护模式。接下来的代码块执行这一操作：

```asm
/*
 * 启动系统。
 */
		mov $0x2820,%bx			# 设置保护模式
		callw setpic			#  IRQ 偏移
		lidt idtdesc			# 设置 IDT
		lgdt gdtdesc			# 设置 GDT
		mov %cr0,%eax			# 切换到保护
		inc %ax				#  模式
		mov %eax,%cr0			#
		ljmp $SEL_SCODE,$init.8		# 跳转到 32 位代码
		.code32
init.8:		xorl %ecx,%ecx			# 清零
		movb $SEL_SDATA,%cl		# 设置为 32 位
		movw %cx,%ss			#  堆栈
```

**stand/i386/btx/btx/btx.S**

首先，调用 `setpic` 来编程 8259A PIC（可编程中断控制器）。该芯片连接到多个硬件中断源。接收到来自设备的中断时，它通过适当的中断向量通知处理器。可以自定义此映射，使特定中断与特定的中断向量关联。接下来，使用 `lidt` 和 `lgdt` 指令加载 IDTR（中断描述符表寄存器）和 GDTR（全局描述符表寄存器）。这两个寄存器分别加载 IDT 和 GDT 的基地址和限制地址。接下来的三条指令设置 `%cr0` 寄存器的保护启用（PE）位。这样就有效地将处理器切换到 32 位保护模式。然后，使用段选择符 SEL\_SCODE 进行长跳转到 `init.8`，该段选择符选择了监督代码段。跳转后，处理器实际上在 CPL 0（最高特权级）下执行。最后，通过将段选择符 SEL\_SDATA 分配给 `%ss` 寄存器，选择监督数据段作为堆栈。这一数据段的特权级也是 `0`。

我们的最后一个代码块负责加载 TR（任务寄存器）与我们之前创建的 TSS 的段选择符，并在将执行控制传递给 **boot2** 客户端之前设置用户模式环境。

```asm
/*
 * 启动用户任务。
 */
		movb $SEL_TSS,%cl		# 设置任务
		ltr %cx				#  寄存器
		movl $MEM_USR,%edx		# 用户基地址
		movzwl %ss:BDA_MEM,%eax		# 获取空闲内存
		shll $0xa,%eax			# 转换为字节
		subl $ARGSPACE,%eax		# 减去参数空间
		subl %edx,%eax			# 减去基地址
		movb $SEL_UDATA,%cl		# 用户数据选择符
		pushl %ecx			# 设置 SS
		pushl %eax			# 设置 ESP
		push $0x202			# 设置标志（IF 设置）
		push $SEL_UCODE			# 设置 CS
		pushl btx_hdr+0xc		# 设置 EIP
		pushl %ecx			# 设置 GS
		pushl %ecx			# 设置 FS
		pushl %ecx			# 设置 DS
		pushl %ecx			# 设置 ES
		pushl %edx			# 设置 EAX
		movb $0x7,%cl			# 设置剩余的
init.9:		push $0x0			#  常规
		loop init.9			#  寄存器
#ifdef BTX_SERIAL
		call sio_init			# 设置串行控制台
#endif
		popa				#  初始化
		popl %es			# 初始化
		popl %ds			#  用户
		popl %fs			#  段
		popl %gs			#  寄存器
		iret				# 跳转到用户模式
```

**stand/i386/btx/btx/btx.S**

请注意，客户端的环境包括堆栈段选择符和堆栈指针（寄存器 `%ss` 和 `%esp`）。事实上，一旦 TR 被加载到适当的堆栈段选择符（指令 `ltr`），堆栈指针将被计算并与堆栈的段选择符一起推送到堆栈。接下来，值 `0x202` 被推送到堆栈，它是当控制权传递给客户端时，EFLAGS 寄存器将获得的值。此外，用户模式代码段选择符和客户端的入口点也被推送。请记住，这个入口点是在链接时通过 BTX 头文件进行修补的。最后，存储在寄存器 `%ecx` 中的段选择符（用于 `%gs, %fs, %ds` 和 `%es` 的段寄存器）也被推送到堆栈中，连同 `%edx` 中的值（`0xa000`）。请记住，已经推送到堆栈中的各种值（它们稍后将被弹出）。接下来，剩余的通用寄存器的值也被推送到堆栈中（注意 `loop` 指令，它推送了 7 次 `0`）。现在，值将开始从堆栈中弹出。首先，`popa` 指令从堆栈中弹出最近推送的 7 个值。它们按顺序存储在通用寄存器中，顺序为 `%edi, %esi, %ebp, %ebx, %edx, %ecx, %eax`。然后，推送的各种段选择符被弹出并复制到各个段寄存器中。堆栈中仍然剩下 5 个值。它们在执行 `iret` 指令时被弹出。该指令首先弹出 BTX 头文件推送的值。这个值是 **boot2** 的入口点指针，它被放置在寄存器 `%eip`（指令指针寄存器）中。接下来，用户代码段的段选择符被弹出并复制到寄存器 `%cs` 中。请记住，这个段的特权级是 3，即最低特权级。这意味着我们必须为这个特权级的堆栈提供值。这就是为什么处理器除了进一步弹出 EFLAGS 寄存器的值之外，还会从堆栈中弹出两个值。这些值被放入堆栈指针（`%esp`）和堆栈段（`%ss`）中。现在，执行控制权将继续在 `boot0` 的入口点。

需要注意的是，用户代码段的定义。该段的 *基地址* 设置为 `0xa000`。这意味着代码内存地址是相对于 `0xa000` 的；如果正在执行的代码从地址 `0x2000` 获取，*实际* 地址将是 `0xa000+0x2000=0xc000`。

## 1.7. boot2 阶段

`boot2` 定义了一个重要的结构体，`struct bootinfo`。这个结构体由 `boot2` 初始化并传递给加载器，随后加载器再传递给内核。这个结构体的部分节点由 `boot2` 设置，其他部分则由加载器设置。该结构体包含了许多信息，例如内核文件名、BIOS 硬盘几何信息、引导设备的 BIOS 驱动器编号、可用的物理内存、`envp` 指针等。其定义如下：

```c
/usr/include/machine/bootinfo.h:
struct bootinfo {
	u_int32_t	bi_version;
	u_int32_t	bi_kernelname;		/* 表示一个 char * */
	u_int32_t	bi_nfs_diskless;	/* struct nfs_diskless * */
				/* 总是存在的字段结束。 */
#define	bi_endcommon	bi_n_bios_used
	u_int32_t	bi_n_bios_used;
	u_int32_t	bi_bios_geom[N_BIOS_GEOM];
	u_int32_t	bi_size;
	u_int8_t	bi_memsizes_valid;
	u_int8_t	bi_bios_dev;		/* 引导设备 BIOS 单元号 */
	u_int8_t	bi_pad[2];
	u_int32_t	bi_basemem;
	u_int32_t	bi_extmem;
	u_int32_t	bi_symtab;		/* struct symtab * */
	u_int32_t	bi_esymtab;		/* struct symtab * */
				/* 仅由高级引导加载器设置的项目 */
	u_int32_t	bi_kernend;		/* 内核空间结束位置 */
	u_int32_t	bi_envp;		/* 环境变量 */
	u_int32_t	bi_modulep;		/* 预加载的模块 */
};
```

`boot2` 进入一个无限循环，等待用户输入，然后调用 `load()`。如果用户没有按任何键，循环会在超时后中断，接着 `load()` 会加载默认文件（**/boot/loader**）。`ino_t lookup(char *filename)` 和 `int xfsread(ino_t inode, void *buf, size_t nbyte)` 函数用于将文件的内容读取到内存中。**/boot/loader** 是一个 ELF 二进制文件，但在 ELF 头部之前有一个 **a.out** 的 `struct exec` 结构。`load()` 扫描加载器的 ELF 头部，将 **/boot/loader** 的内容加载到内存，并将执行权传递给加载器的入口点：

```c
stand/i386/boot2/boot2.c:
    __exec((caddr_t)addr, RB_BOOTINFO | (opts & RBX_MASK),
	   MAKEBOOTDEV(dev_maj[dsk.type], dsk.slice, dsk.unit, dsk.part),
	   0, 0, 0, VTOP(&bootinfo));
```

## 1.8. loader 阶段

loader 也是一个 BTX 客户端。这里我不再详细描述它，Mike Smith 写的详细手册 [loader(8)](https://man.freebsd.org/cgi/man.cgi?query=loader&sektion=8&format=html) 可以提供更多信息。上面已经讨论了底层机制和 BTX。

loader 的主要任务是引导内核。当内核被加载到内存中后，loader 会调用内核：

```c
stand/common/boot.c:
    /* 调用与内核匹配的加载器中的 exec 处理函数 */
    file_formats[fp->f_loader]->l_exec(fp);
```

## 1.9. 内核初始化

让我们看看链接内核的命令。这将帮助我们确定加载器将执行权传递给内核的确切位置。这个位置就是内核的实际入口点。这个命令现在已经从 **sys/conf/Makefile.i386** 中移除。我们感兴趣的内容可以在 **/usr/obj/usr/src/i386.i386/sys/GENERIC/** 中找到。

```c
/usr/obj/usr/src/i386.i386/sys/GENERIC/kernel.meta:
ld -m elf_i386_fbsd -Bdynamic -T /usr/src/sys/conf/ldscript.i386 --build-id=sha1 --no-warn-mismatch \
--warn-common --export-dynamic  --dynamic-linker /red/herring -X -o kernel locore.o
<许多内核 .o 文件>
```

这里可以看到几个有趣的事情。首先，内核是一个 ELF 动态链接的二进制文件，但内核的动态链接器是 **/red/herring**，这显然是一个虚假的文件。其次，查看文件 **sys/conf/ldscript.i386** 可以了解到在编译内核时使用的 ld 选项。读完前几行后，可以看到字符串

```c
sys/conf/ldscript.i386:
ENTRY(btext)
```

这表明内核的入口点是符号 `btext`。这个符号在 **locore.s** 中定义：

```c
sys/i386/i386/locore.s:
	.text
/**********************************************************************
 *
 * 这就是引导块开始的地方，让一切开始运行...
 *
 */
NON_GPROF_ENTRY(btext)
```

首先，EFLAGS 寄存器被设置为预定义的值 0x00000002。然后初始化所有段寄存器：

```asm
sys/i386/i386/locore.s:
/* 不相信 BIOS 给出的 eflags。 */
	pushl	$PSL_KERNEL
	popfl

/*
 * 不相信 BIOS 给出的 %fs 和 %gs。相信引导程序来设置 %cs、%ds、%es 和 %ss。
 */
	mov	%ds, %ax
	mov	%ax, %fs
	mov	%ax, %gs
```

btext 调用了 **locore.s** 中定义的 `recover_bootinfo()` 和 `identify_cpu()` 函数。以下是它们的作用说明：

| `recover_bootinfo` | 该例程解析引导程序传递给内核的参数。内核可能通过三种方式启动：通过上面描述的加载器，通过旧的磁盘引导块，或者通过旧的无盘引导程序。此函数确定引导方式，并将 `struct bootinfo` 结构存储到内核内存中。 |
| ------------------ | ----------------------------------------------------------------------------------------------------------- |
| `identify_cpu`     | 该函数试图找出运行的 CPU 类型，并将找到的值存储在变量 `_cpu` 中。                                                                     |

接下来的步骤是启用 VME，如果 CPU 支持的话：

```asm
sys/i386/i386/mpboot.s:
	testl	$CPUID_VME,%edx
	jz	3f
	orl	$CR4_VME,%eax
3:	movl	%eax,%cr4
```

然后，启用分页：

```asm
sys/i386/i386/mpboot.s:
/* 现在启用分页 */
	movl	IdlePTD_nopae, %eax
	movl	%eax,%cr3			/* 将 ptd 地址加载到 mmu 中 */
	movl	%cr0,%eax			/* 获取控制字 */
	orl	$CR0_PE|CR0_PG,%eax		/* 启用分页 */
	movl	%eax,%cr0			/* 现在开始分页！ */
```

接下来的三行代码是因为已设置分页，因此需要跳转到虚拟地址空间中继续执行：

```
sys/i386/i386/mpboot.s:
	pushl	$mp_begin				/* 跳转到高内存 */
	ret

/* 现在运行在 KERNBASE 处，即系统被链接以运行的地方 */
mp_begin:	/* 现在运行在 KERNBASE */
```

内核在调用 `mi_startup()` 后完成引导，而在此之前调用了 `init386()`。`init386()` 是一个与架构相关的初始化函数，而 `mi_startup()` 是一个与架构无关的函数（'mi\_' 前缀表示与机器无关）。内核在调用 `mi_startup()` 后不会再返回，且通过调用它，内核完成了引导：

```asm
sys/i386/i386/locore.s:
	pushl	physfree			/* 传递给 init386(first) 的第一个物理页面地址 */
	call	init386				/* 配置 386 芯片以供 Unix 操作 */
	addl	$4,%esp
	movl	%eax,%esp			/* 切换到真实的栈顶 */
	call	mi_startup			/* 自动配置，挂载根文件系统等 */
	/* NOTREACHED */
```

### 1.9.1. `init386()`

`init386()` 定义在 **sys/i386/i386/machdep.c** 中，执行与 i386 芯片特定的低级初始化。保护模式的切换由加载器完成。加载器已经创建了第一个任务，内核在其中继续运行。在查看代码之前，可以考虑处理器必须完成的任务，以初始化保护模式执行：

* 初始化从引导程序传递过来的内核可调参数。
* 准备 GDT（全局描述符表）。
* 准备 IDT（中断描述符表）。
* 初始化系统控制台。
* 初始化 DDB（调试器），如果它被编译到内核中。
* 初始化 TSS（任务状态段）。
* 准备 LDT（局部描述符表）。
* 设置线程 0 的 PCB（进程控制块）。

`init386()` 通过设置环境指针（envp）并调用 `init_param1()` 来初始化从引导程序传递过来的可调参数。envp 指针是通过 `bootinfo` 结构从加载器传递过来的：

```c
sys/i386/i386/machdep.c:
	/* 初始化基本的可调参数，hz 等 */
	init_param1();
```

`init_param1()` 定义在 **sys/kern/subr\_param.c** 中。该文件有多个 sysctl 以及两个函数 `init_param1()` 和 `init_param2()`，它们都从 `init386()` 被调用：

```c
sys/kern/subr_param.c:
	hz = -1;
	TUNABLE_INT_FETCH("kern.hz", &hz);
	if (hz == -1)
		hz = vm_guest > VM_GUEST_NO ? HZ_VM : HZ;
```

`TUNABLE_INT_FETCH` 用于从环境中获取值：

```c
/usr/src/sys/sys/kernel.h:
#define	TUNABLE_INT_FETCH(path, var)	getenv_int((path), (var))
```

Sysctl `kern.hz` 是系统时钟滴答的频率。此外，`init_param1()` 还设置了以下 sysctl：`kern.maxswzone`、`kern.maxbcache`、`kern.maxtsiz`、`kern.dfldsiz`、`kern.maxdsiz`、`kern.dflssiz`、`kern.maxssiz`、`kern.sgrowsiz`。

然后，`init386()` 准备了全局描述符表（GDT）。在 x86 上，每个任务都在自己的虚拟地址空间中运行，这个空间由段：偏移对来寻址。例如，假设当前由处理器执行的指令位于 CS\:EIP，那么该指令的线性虚拟地址将是“代码段 CS 的虚拟地址”+ EIP。为了方便，段从虚拟地址 0 开始，到达 4GB 边界。因此，在此示例中，指令的线性虚拟地址仅为 EIP 的值。像 CS、DS 等段寄存器就是选择符，即 GDT 的索引（准确地说，选择符的索引字段才是索引，而不是选择符本身）。FreeBSD 的 GDT 为每个 CPU 持有 15 个选择符的描述符：

### 1.9.1. `init386()` 中的 GDT 和 IDT 初始化

在 `init386()` 中，首先会初始化 GDT（全局描述符表）。以下代码定义了初始的 GDT，并设置为 `gdt0`：

```
sys/i386/i386/machdep.c:
union descriptor gdt0[NGDT];	/* 初始的全局描述符表 */
union descriptor *gdt = gdt0;	/* 全局描述符表 */
```

然后，在 **sys/x86/include/segments.h** 中定义了 GDT 中的各个选择符（selector）：

```
sys/x86/include/segments.h:
/*
 * GDT 表中的各个条目
 */
#define	GNULL_SEL	0	/* 空描述符 */
#define	GPRIV_SEL	1	/* SMP 每个处理器私有数据 */
#define	GUFS_SEL	2	/* 用户 %fs 描述符（顺序至关重要：1） */
#define	GUGS_SEL	3	/* 用户 %gs 描述符（顺序至关重要：2） */
#define	GCODE_SEL	4	/* 内核代码描述符（顺序至关重要：1） */
#define	GDATA_SEL	5	/* 内核数据描述符（顺序至关重要：2） */
#define	GUCODE_SEL	6	/* 用户代码描述符（顺序至关重要：3） */
#define	GUDATA_SEL	7	/* 用户数据描述符（顺序至关重要：4） */
#define	GBIOSLOWMEM_SEL	8	/* BIOS 低内存访问（必须是条目 8） */
#define	GPROC0_SEL	9	/* 任务状态进程槽零及其之后 */
#define	GLDT_SEL	10	/* 默认用户 LDT */
#define	GUSERLDT_SEL	11	/* 用户 LDT */
#define	GPANIC_SEL	12	/* 考虑 panic 的任务状态 */
#define	GBIOSCODE32_SEL	13	/* BIOS 接口（32位代码） */
#define	GBIOSCODE16_SEL	14	/* BIOS 接口（16位代码） */
#define	GBIOSDATA_SEL	15	/* BIOS 接口（数据） */
#define	GBIOSUTIL_SEL	16	/* BIOS 接口（工具） */
#define	GBIOSARGS_SEL	17	/* BIOS 接口（参数） */
#define	GNDIS_SEL	18	/* 用于 NDIS 层 */
#define	NGDT		19
```

这些定义并不是选择符本身，而仅仅是选择符的索引字段。它们对应 GDT 中的各个条目的索引。例如，内核代码选择符（`GCODE_SEL`）的实际选择符值是 0x20。

接下来，`init386()` 会初始化中断描述符表（IDT）。该表由处理器在发生软件或硬件中断时引用。例如，用户应用程序通过发出 `INT 0x80` 指令来进行系统调用。这是一个软件中断，处理器的硬件会查找 IDT 中索引为 0x80 的记录，这个记录指向处理该中断的例程。在这种情况下，指向的就是内核的系统调用门（syscall gate）。IDT 最多可以有 256 条记录。内核为 IDT 分配了 NIDT 条记录，其中 NIDT 是最大值（256）：

```
sys/i386/i386/machdep.c:
static struct gate_descriptor idt0[NIDT];
struct gate_descriptor *idt = &idt0[0];	/* 中断描述符表 */
```

然后，为每个中断设置相应的处理程序。`INT 0x80` 的系统调用门也被设置：

```
sys/i386/i386/machdep.c:
	setidt(IDT_SYSCALL, &IDTVEC(int0x80_syscall),
			SDT_SYS386IGT, SEL_UPL, GSEL(GCODE_SEL, SEL_KPL));
```

当用户空间的应用程序发出 `INT 0x80` 指令时，控制将转移到内核代码段中的函数 `_Xint0x80_syscall`，并以 supervisor 特权级别执行。

接下来，控制台和 DDB（内核调试器）被初始化：

```
sys/i386/i386/machdep.c:
	cninit();
/* 省略 */
  kdb_init();
#ifdef KDB
	if (boothowto & RB_KDB)
		kdb_enter(KDB_WHY_BOOTFLAGS, "Boot flags requested debugger");
#endif
```

### 1.9.1. `init386()` 中的 TSS 和 LDT 初始化

任务状态段（TSS）是另一个 x86 保护模式结构。TSS 由硬件在任务切换时存储任务信息。接下来是局部描述符表（LDT）的初始化。LDT 用于引用用户空间的代码和数据。以下是定义 LDT 中选择符的几个宏，它们是系统调用门和用户代码、数据选择符：

```
sys/x86/include/segments.h:
#define	LSYS5CALLS_SEL	0	/* 强制由 intel BCS */
#define	LSYS5SIGR_SEL	1
#define	LUCODE_SEL	3
#define	LUDATA_SEL	5
#define	NLDT		(LUDATA_SEL + 1)
```

### 1.9.1. `init386()` 中的 PCB 初始化

接下来，初始化 `proc0` 的进程控制块（PCB）结构。`proc0` 是一个描述内核进程的 `struct proc` 结构，它在内核运行时始终存在，因此它与 `thread0` 关联：

```
sys/i386/i386/machdep.c:
register_t
init386(int first)
{
    /* ... 省略 ... */

    proc_linkup0(&proc0, &thread0);
    /* ... 省略 ... */
}
```

`struct pcb` 是 `proc` 结构的一部分，定义了与 i386 架构相关的进程信息，如寄存器值。

### 1.9.2. `mi_startup()` 系统初始化

`mi_startup()` 函数执行所有系统初始化对象的冒泡排序，然后依次调用每个对象的入口函数：

```c
sys/kern/init_main.c:
for (sipp = sysinit; sipp < sysinit_end; sipp++) {

		/* ... 省略 ... */

		/* 调用函数 */
		(*((*sipp)->func))((*sipp)->udata);
		/* ... 省略 ... */
	}
```

每个系统初始化对象（sysinit 对象）都是通过调用 `SYSINIT()` 宏来创建的。以下是一个 `announce` sysinit 对象的示例，它打印版权信息：

```c
sys/kern/init_main.c:
static void
print_caddr_t(void *data __unused)
{
	printf("%s", (char *)data);
}
/* ... 省略 ... */
SYSINIT(announce, SI_SUB_COPYRIGHT, SI_ORDER_FIRST, print_caddr_t, copyright);
```

该对象的子系统 ID 为 `SI_SUB_COPYRIGHT`（0x0800001），因此版权信息会在控制台初始化之后首先打印。

宏 `SYSINIT()` 扩展为 `C_SYSINIT()` 宏。`C_SYSINIT()` 宏再扩展为一个静态的 `struct sysinit` 结构声明，并调用 `DATA_SET` 宏：

```c
/usr/include/sys/kernel.h:
#define C_SYSINIT(uniquifier, subsystem, order, func, ident) \
      static struct sysinit uniquifier ## _sys_init = { \ subsystem, \
      order, \ func, \ (ident) \ }; \ DATA_WSET(sysinit_set,uniquifier ##
      _sys_init);

#define	SYSINIT(uniquifier, subsystem, order, func, ident)	\
	C_SYSINIT(uniquifier, subsystem, order,			\
	(sysinit_cfunc_t)(sysinit_nfunc_t)func, (void *)(ident))
```

宏 `DATA_SET()` 扩展为 `_MAKE_SET()`，这是隐藏所有 sysinit 魔法的地方：

```c
/usr/include/linker_set.h:
#define TEXT_SET(set, sym) _MAKE_SET(set, sym)
#define DATA_SET(set, sym) _MAKE_SET(set, sym)
```

执行这些宏后，内核中会创建多个段，包括 `set.sysinit_set`。通过运行 `objdump` 命令查看内核二进制文件时，你可能会发现这些小段的存在：

```sh
% llvm-objdump -h /kernel
Sections:
Idx Name                               Size     VMA      Type
 10 set_sysctl_set                     000021d4 01827078 DATA
 16 set_kbddriver_set                  00000010 0182a4d0 DATA
 20 set_scterm_set                     0000000c 0182c75c DATA
 21 set_cons_set                       00000014 0182c768 DATA
 33 set_scrndr_set                     00000024 0182c828 DATA
 41 set_sysinit_set                    000014d8 018fabb0 DATA
```

这段输出显示 `set.sysinit_set` 的大小是 0x14d8 字节，因此 `0x14d8/sizeof(void *)` 个 sysinit 对象被编译进内核。其他段如 `set.sysctl_set` 表示其他链接器集。

通过定义一个类型为 `struct sysinit` 的变量，`set.sysinit_set` 部分的内容将被“收集”到该变量中：

```c
sys/kern/init_main.c:
  SET_DECLARE(sysinit_set, struct sysinit);
```

`struct sysinit` 定义如下：

```asm
sys/sys/kernel.h:
  struct sysinit {
	enum sysinit_sub_id	subsystem;	/* 子系统标识符 */
	enum sysinit_elem_order	order;		/* 子系统内的初始化顺序 */
	sysinit_cfunc_t func;			/* 函数 */
	const void	*udata;			/* 多路复用器/参数 */
};
```

回到 `mi_startup()` 的讨论，现在必须清楚，sysinit 对象是如何组织的。`mi_startup()` 函数对它们进行排序并逐个调用。最后一个对象是系统调度器：

```asm
/usr/include/sys/kernel.h:
enum sysinit_sub_id {
	SI_SUB_DUMMY		= 0x0000000,	/* 未执行；用于链接器 */
	SI_SUB_DONE		= 0x0000001,	/* 已处理 */
	SI_SUB_TUNABLES		= 0x0700000,	/* 建立可调参数 */
	SI_SUB_COPYRIGHT	= 0x0800001,	/* 控制台首次使用 */
...
	SI_SUB_LAST		= 0xfffffff	/* 最终初始化 */
};
```

系统调度器的 sysinit 对象定义在文件 **sys/vm/vm\_glue.c** 中，该对象的入口点是 `scheduler()`。该函数实际上是一个无限循环，它代表一个进程，PID 为 0，即交换进程。之前提到的 `thread0` 结构用于描述它。

第一个用户进程，即 *init*，由 sysinit 对象 `init` 创建：

```
sys/kern/init_main.c:
static void
create_init(const void *udata __unused)
{
	struct fork_req fr;
	struct ucred *newcred, *oldcred;
	struct thread *td;
	int error;

	bzero(&fr, sizeof(fr));
	fr.fr_flags = RFFDG | RFPROC | RFSTOPPED;
	fr.fr_procp = &initproc;
	error = fork1(&thread0, &fr);
	if (error)
		panic("cannot fork init: %d\n", error);
	KASSERT(initproc->p_pid == 1, ("create_init: initproc->p_pid != 1"));
	/* 将 init 的凭证与内核的凭证分离 */
	newcred = crget();
	sx_xlock(&proctree_lock);
	PROC_LOCK(initproc);
	initproc->p_flag |= P_SYSTEM | P_INMEM;
	initproc->p_treeflag |= P_TREE_REAPER;
	oldcred = initproc->p_ucred;
	crcopy(newcred, oldcred);
#ifdef MAC
	mac_cred_create_init(newcred);
#endif
#ifdef AUDIT
	audit_cred_proc1(newcred);
#endif
	proc_set_cred(initproc, newcred);
	td = FIRST_THREAD_IN_PROC(initproc);
	crcowfree(td);
	td->td_realucred = crcowget(initproc->p_ucred);
	td->td_ucred = td->td_realucred;
	PROC_UNLOCK(initproc);
	sx_xunlock(&proctree_lock);
	crfree(oldcred);
	cpu_fork_kthread_handler(FIRST_THREAD_IN_PROC(initproc), start_init, NULL);
}
SYSINIT(init, SI_SUB_CREATE_INIT, SI_ORDER_FIRST, create_init, NULL);
```

`create_init()` 函数通过调用 `fork1()` 分配一个新进程，但没有标记它为可运行。当这个新进程被调度器调度执行时，将会调用 `start_init()`。该函数定义在 **init\_main.c** 中。它尝试加载并执行 **init** 二进制文件，首先探测 **/sbin/init**，然后是 **/sbin/oinit**、**/sbin/init.bak**，最后是 **/rescue/init**：

```c
sys/kern/init_main.c:
static char init_path[MAXPATHLEN] =
#ifdef	INIT_PATH
    __XSTRING(INIT_PATH);
#else
    "/sbin/init:/sbin/oinit:/sbin/init.bak:/rescue/init";
#endif
```
[1](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnoteref_1). 如果用户在 boot0 阶段选择操作系统后按下键盘，就会显示此提示。

[2](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnoteref_2). 如果有疑问，我们建议读者参考官方的 Intel 手册，手册中详细描述了每条指令的语义。

[3](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnoteref_3). 有一个文件 /boot/boot1，但它并不会写入到 FreeBSD 切片的开头。相反，它与 boot2 拼接在一起，形成 boot，这个 boot 会被写入 FreeBSD 切片的开头，并在启动时读取。

[4](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnoteref_4). 实际上，我们将指向切片条目的指针传递到了寄存器 %si 中。然而，boot1 并不假设它是由 boot0 加载的（也许是其他 MBR 加载了它，并且没有传递这个信息），因此它什么也不假设。

[5](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnoteref_5). 在 16 位实模式的上下文中，一个字是 2 字节。

[6](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnoteref_6). 512\*16=8192 字节，正好是 boot 的大小。

[7](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnoteref_7). 历史上称为 disklabel。如果你曾经想知道 FreeBSD 将这些信息存储在哪里，它就在这个区域——请参见 [bsdlabel(8)](https://man.freebsd.org/cgi/man.cgi?query=bsdlabel&sektion=8&format=html)。

[8](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnoteref_8). 这是由于遗留原因所需的。有兴趣的读者可以查看。

[9](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnoteref_9). 在从保护模式切换回实模式时，实模式代码和数据是必要的，如 Intel 手册所建议。
