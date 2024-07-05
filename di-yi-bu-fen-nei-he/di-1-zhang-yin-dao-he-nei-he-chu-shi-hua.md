# 第 1 章 引导和内核初始化

## 1.1. 概要

本章节是关于引导和系统初始化过程的概述，从 BIOS（固件）POST 开始，到第一个用户进程的创建。由于系统启动的初始步骤非常依赖架构，以 IA-32 架构作为示例。但是，AMD64 和 ARM64 架构是更重要和引人注目的示例，根据本文档的主题，应该在不久的将来进行解释。

FreeBSD 的引导过程可能会出人意料地复杂。在控制权从 BIOS 传递后，必须进行大量的低级配置，然后才能加载和执行内核。这个设置必须以简单灵活的方式完成，允许用户有很多定制的可能性。

## 1.2. 概述

引导过程是一个极其依赖机器的活动。不仅每种计算机架构都需要编写代码，而且在同一架构上也可能有多种引导类型。例如，列出 stand 目录显示了大量与架构相关的代码。每种支持的架构都有一个目录。FreeBSD 支持 CSM 引导标准（兼容性支持模块）。因此，CSM 是支持的（同时支持 GPT 和 MBR 分区）并且支持 UEFI 引导（完全支持 GPT，主要支持 MBR）。它还支持从 ext2fs、MSDOS、UFS 和 ZFS 加载文件。FreeBSD 还支持 ZFS 的引导环境功能，使得主机操作系统可以传达有关引导的详细信息，这些信息比过去简单的分区更加复杂。但如今 UEFI 比 CSM 更为重要。下面的例子展示了如何从一个 MBR 分区的硬盘上引导一个 x86 计算机，使用存储在第一个扇区的 FreeBSD boot0 多重引导加载程序。该引导代码启动了 FreeBSD 的三阶段引导过程。

理解这一过程的关键在于它是一个复杂性逐渐增加的阶段系列。这些阶段是 boot1、boot2 和 loader（参见 boot(8) 了解更多详情）。引导系统按顺序执行每个阶段。最后一个阶段，loader，负责加载 FreeBSD 内核。以下部分将详细介绍每个阶段。

这是由不同引导阶段生成的输出示例。实际输出可能因机器而异：

| ** FreeBSD 组件** | ** 输出（可能有所不同）** |
| ------------------- | --------------------------- |
| `boot0`                  | `F1    FreeBSD`<br />`F2    BSD`<br />`F5    Disk 2`                      |
| `boot2` \^[[1](https://docs.freebsd.org/en/books/arch-handbook/boot/#_footnotedef_1 "View footnote.")]\^     | `>>FreeBSD/x86 BOOT`<br />`Default: 0:ad(0p4)/boot/loader`<br />`boot:`                      |
| 装载程序          | `BTX loader 1.00 BTX version is 1.02`<br />`Consoles: internal video/keyboard`<br />`BIOS drive C: is disk0`<br />`BIOS 639kB/2096064kB available memory`<br />`FreeBSD/x86 bootstrap loader, Revision 1.1`<br />`Console internal video/keyboard`<br />`(root@releng1.nyi.freebsd.org, Fri Apr  9 04:04:45 UTC 2021)`<br />`Loading /boot/defaults/loader.conf`<br />`/boot/kernel/kernel text=0xed9008 data=0x117d28+0x176650 syms=[0x8+0x137988+0x8+0x1515f8]`          |
| 内核              | `Copyright (c) 1992-2021 The FreeBSD Project.`<br />`Copyright (c) 1979, 1980, 1983, 1986, 1988, 1989, 1991, 1992, 1993, 1994`<br />`        The Regents of the University of California. All rights reserved.`<br />`FreeBSD is a registered trademark of The FreeBSD Foundation.`<br />`FreeBSD 13.0-RELEASE 0 releng/13.0-n244733-ea31abc261f: Fri Apr  9 04:04:45 UTC 2021`<br />`    root@releng1.nyi.freebsd.org:/usr/obj/usr/src/i386.i386/sys/GENERIC i386`<br />`FreeBSD clang version 11.0.1 (git@github.com:llvm/llvm-project.git llvmorg-11.0.1-0-g43ff75f2c3fe)`              |

## 1.3. BIOS

当计算机启动时，处理器的寄存器被设置为一些预定义的值。其中一个寄存器是指令指针寄存器，在开机后的值是明确定义的：它是一个 32 位的值 0xfffffff0 。指令指针寄存器（也称为程序计数器）指向处理器要执行的代码。另一个重要的寄存器是 cr0 32 位控制寄存器，在重新启动后的值是 0 。其中 cr0 的位之一，即 PE（Protection Enabled）位，指示处理器是否在 32 位保护模式或 16 位实模式下运行。由于此位在引导时被清除，处理器以 16 位实模式引导。实模式意味着，线性地址和物理地址是相同的。处理器不立即以 32 位保护模式启动的原因是向后兼容性。特别是，引导过程依赖于 BIOS 提供的服务，而 BIOS 本身在传统的 16 位代码中运行。

0xfffffff0 的值略小于 4 GB，因此，除非计算机具有 4 GB 的物理内存，否则它无法指向有效的内存地址。计算机的硬件会将此地址转换，使其指向 BIOS 内存块。

基本输入输出系统（BIOS）是主板上的一块芯片，具有相对较小的只读存储器（ROM）。该存储器包含各种特定于主板提供的硬件的低级例程。处理器首先会跳转到地址 0xfffffff0，实际上位于 BIOS 存储器中。通常，该地址包含一个跳转指令，指向 BIOS 的 POST 例程。

上电自检（POST）是一组例程，包括内存检查、系统总线检查和其他低级初始化，以便 CPU 可以正确设置计算机。此阶段的重要步骤是确定引导设备。现代 BIOS 实现允许选择引导设备，允许从软盘、CD-ROM、硬盘或其他设备引导。

POST 的最后一步是 INT 0x19 指令。 INT 0x19 处理程序将引导设备的第一个扇区中的 512 字节读入地址 0x7c00 处的内存中。术语“第一个扇区”源自硬盘架构，其中磁盘分为多个圆柱磁道。磁道编号，每个磁道分为多个扇区（通常为 64 个）。磁道编号从 0 开始，但扇区编号从 1 开始。磁道 0 是磁盘的最外层，第一个扇区即第 1 扇区具有特殊用途。它也称为 MBR 或主引导记录。第一个磁道上的其余扇区从不使用。

这个扇区是我们的引导序列起始点。正如我们将看到的，这个扇区包含我们的 boot0 程序的副本。BIOS 跳转到地址 0x7c00 开始执行。

## 1.4. 主引导记录（ boot0 ）

在从 BIOS 接收到控制权并位于内存地址 0x7c00 处后，boot0 开始执行。这是 FreeBSD 控制下的第一段代码。boot0 的任务非常简单：扫描分区表，并让用户选择要从哪个分区引导。分区表是嵌入在 MBR 中的特殊标准数据结构（因此嵌入在 boot0 中），描述了四个标准 PC “分区”。boot0 位于文件系统中的 /boot/boot0。它是一个小的 512 字节文件，它正是 FreeBSD 安装过程在硬盘的 MBR 中写入的内容，如果您在安装时选择了“bootmanager”选项。实际上，boot0 就是 MBR。

如前所述，我们称 BIOS INT 0x19 为在地址 0x7c00 处将 MBR（boot0）加载到内存中。boot0 的源文件可以在 stand/i386/boot0/boot0.S 中找到 - 这是由 Robert Nordier 编写的一段很棒的代码。

从 MBR 中的偏移 0x1be 开始的特殊结构称为分区表。它有四个记录，每个记录 16 字节，称为分区记录，代表硬盘如何被分区，或者在 FreeBSD 的术语中称为切片。这 16 个字节中的一个字节表示分区（切片）是否可引导。必须有一个记录设置了该标志，否则 boot0 的代码将拒绝继续执行。

分区记录有以下字段：

* 1 字节的文件系统类型
* 1 字节的可引导标志
* CHS 格式中的 6 字节描述符
* 以 LBA 格式的 8 字节描述符

分区记录描述符包含有关分区在驱动器上的确切位置的信息。 LBA 和 CHS 两种描述符以不同的方式描述相同的信息：LBA（逻辑块寻址）具有分区的起始扇区和分区长度，而 CHS（柱面磁头扇区）具有分区的第一个和最后一个扇区的坐标。 分区表以特殊签名 0xaa55 结束。

MBR 必须适合 512 字节，即单个磁盘扇区。 该程序使用低级“技巧”，如利用某些指令的副作用并重用先前操作的寄存器值，以充分利用尽可能少的指令。 处理嵌入在 MBR 中的分区表时也必须小心。 出于这些原因，在修改 boot0.S 时务必小心。

注意，boot0.S 源文件是按原样汇编的：指令逐条转换为二进制，没有额外的信息（例如没有 ELF 文件格式）。这种低级控制是通过传递给链接器的特殊控制标志在链接时实现的。例如，程序的文本部分被设置为位于地址 0x600 。实际上，这意味着 boot0 必须加载到内存地址 0x600 才能正常运行。

值得查看 boot0 的 Makefile（stand/i386/boot0/Makefile），因为它定义了 boot0 的一些运行时行为。例如，如果连接到串口port（COM1）的终端用于 I/O，则必须定义宏 SIO （ -DSIO ）。按下 F6 启用通过 PXE 引导。此外，该程序定义了一组标志，允许进一步修改其行为。所有这些都在 Makefile 中有所体现。例如，查看链接器指令，该指令命令链接器从地址 0x600 开始启动文本部分，并构建输出文件“按原样”（剥离任何文件格式）：

```
      BOOT_BOOT0_ORG?=0x600
      ORG=${BOOT_BOOT0_ORG}
```

stand/i386/boot0/Makefile

现在让我们开始学习 MBR，或者 boot0，从执行开始的地方开始。

|  | 对一些指令进行了一些修改，以便更好地阐明。例如，某些宏被展开，当测试的结果已知时，某些宏测试被省略。 这适用于所示的所有代码示例。 |
| -- | ----------------------------------------------------------------------------------------------------------------------------------- |

```
start:
      cld			# String ops inc
      xorw %ax,%ax		# Zero
      movw %ax,%es		# Address
      movw %ax,%ds		#  data
      movw %ax,%ss		# Set up
      movw $LOAD,%sp		#  stack
```

stand/i386/boot0/boot0.S

这段代码的第一个块是程序的入口点。这是 BIOS 转移控制的地方。首先，它确保字符串操作自动递增其指针操作数（ cld 指令）^[ 2]^。然后，由于它对段寄存器的状态没有假设，它对它们进行初始化。最后，它将堆栈指针寄存器（ %sp ）设置为（$LOAD = 地址 0x7c00 ），这样我们就有一个可用的堆栈。

下一个块负责重定位并跳转到重定位后的代码。

```
      movw %sp,%si     # Source
      movw $start,%di		# Destination
      movw $0x100,%cx		# Word count
      rep			# Relocate
      movsw			#  code
      movw %di,%bp		# Address variables
      movb $0x8,%cl		# Words to clear
      rep			# Zero
      stosw			#  them
      incb -0xe(%di)		# Set the S field to 1
      jmp main-LOAD+ORIGIN	# Jump to relocated code
```

stand/i386/boot0/boot0.S

当 boot0 被 BIOS 加载到地址 0x7C00 时，它将自身复制到地址 0x600 ，然后将控制权转移到那里（回想一下，它被链接以在地址 0x600 处执行）。源地址 0x7c00 被复制到寄存器 %si 。目标地址 0x600 被复制到寄存器 %di 。要复制的字数 256 （程序大小 = 512 字节）被复制到寄存器 %cx 。接下来， rep 指令重复执行后面的指令，即 movsw ，由 %cx 寄存器指示的次数。 movsw 指令将由 %si 指向的字复制到由 %di 指向的地址。这将重复另外 255 次。在每次重复中，源和目标寄存器 %si 和 %di 都会递增一次。因此，在完成 256 个字（512 字节）的复制后， %di 的值为 0x600512 = 0x800 ， %si 的值为 0x7c00512 = 0x7e00 ；因此我们完成了代码重定位。自上次更新此文档以来，代码中的复制指令已更改，因此不再使用 movsb 和 stosb，而是引入了 movsw 和 stosw，它们在一次迭代中复制 2 字节（1 个字）。

接下来，目标寄存器 %di 被复制到 %bp 。 %bp 得到值 0x800 。值 8 被复制到 %cl 以准备进行新的字符串操作（类似于我们之前的 movsw ）。现在， stosw 被执行 8 次。此指令将一个 0 值复制到由目标寄存器指向的地址（ %di ，即 0x800 ），并递增它。这将重复另外 7 次，因此 %di 最终值为 0x810 。实际上，这清除了地址范围 0x800 - 0x80f 。该范围用作（虚假）分区表，用于将 MBR 写回磁盘。最后，给这个虚假分区的 CHS 寻址的扇区字段赋值为 1，并跳转到从重定位代码中的主函数。请注意，在跳转到重定位代码之前，避免了对绝对地址的任何引用。

以下代码块测试 BIOS 提供的驱动器编号是否应该使用，还是存储在 boot0 中的驱动器编号。

```
main:
      testb $SETDRV,_FLAGS(%bp)	# Set drive number?
#ifndef CHECK_DRIVE	/* disable drive checks */
      jz save_curdrive		# no, use the default
#else
      jnz disable_update	# Yes
      testb %dl,%dl		# Drive number valid?
      js save_curdrive		# Possibly (0x80 set)
#endif
```

stand/i386/boot0/boot0.S

此代码测试标志变量中的 SETDRV 位（ 0x20 ）。请记住，寄存器 %bp 指向地址位置 0x800 ，因此测试是针对地址 0x800 - 69 = 0x7bb 处的标志变量进行的。这是可以对 boot0 进行的修改类型的示例。 SETDRV 标志默认未设置，但可以在 Makefile 中设置。设置后，将使用 MBR 中存储的驱动器编号，而不是 BIOS 提供的驱动器编号。我们假设使用默认值，并且 BIOS 提供了有效的驱动器编号，因此我们跳转到 save_curdrive 。

下一个块保存 BIOS 提供的驱动器编号，并调用 putn 在屏幕上打印新行。

```
save_curdrive:
      movb %dl, (%bp)		# Save drive number
      pushw %dx			# Also in the stack
#ifdef	TEST	/* test code, print internal bios drive */
      rolb $1, %dl
      movw $drive, %si
      call putkey
#endif
      callw putn		# Print a newline
```

stand/i386/boot0/boot0.S

请注意，我们假设 TEST 未定义，因此其中的条件代码未被汇编，也不会出现在我们的可执行 boot0 中。

我们的下一个模块实现了对分区表的实际扫描。它会将每个分区表中的四个条目的分区类型打印到屏幕上。它会将每种类型与一组知名操作系统文件系统进行比较。被识别的分区类型的示例有 NTFS（Windows®，ID 0x7）、 ext2fs （Linux®，ID 0x83），当然还有 ffs / ufs2 （FreeBSD，ID 0xa5）。实现相当简单。

```
      movw $(partbl+0x4),%bx	# Partition table (+4)
      xorw %dx,%dx		# Item number

read_entry:
      movb %ch,-0x4(%bx)	# Zero active flag (ch == 0)
      btw %dx,_FLAGS(%bp)	# Entry enabled?
      jnc next_entry		# No
      movb (%bx),%al		# Load type
      test %al, %al		# skip empty partition
      jz next_entry
      movw $bootable_ids,%di	# Lookup tables
      movb $(TLEN+1),%cl	# Number of entries
      repne			# Locate
      scasb			#  type
      addw $(TLEN-1), %di	# Adjust
      movb (%di),%cl		# Partition
      addw %cx,%di		#  description
      callw putx		# Display it

next_entry:
      incw %dx			# Next item
      addb $0x10,%bl		# Next entry
      jnc read_entry		# Till done
```

stand/i386/boot0/boot0.S

重要的是要注意，每个条目的活动标志都被清除了，因此在扫描之后，我们在 boot0 的内存副本中没有活动分区条目。稍后，将为选定的分区设置活动标志。这确保了如果用户选择将更改写回磁盘，只存在一个活动分区。

下一个块检测其他驱动器。在启动时，BIOS 会将计算机中存在的驱动器数量写入地址 0x475 。如果存在其他驱动器，boot0 会将当前驱动器打印到屏幕上。用户可以随后命令 boot0 扫描另一个驱动器上的分区。

```
      popw %ax			# Drive number
      subb $0x80-0x1,%al		# Does next
      cmpb NHRDRV,%al		#  drive exist? (from BIOS?)
      jb print_drive		# Yes
      decw %ax			# Already drive 0?
      jz print_prompt		# Yes
```

stand/i386/boot0/boot0.S

我们假设只有一个驱动器存在，因此不执行跳转到 print_drive 。我们也假设没有发生奇怪的事情，因此跳转到 print_prompt 。

这个下一块只是打印一个提示，然后是默认选项：

```
print_prompt:
      movw $prompt,%si		# Display
      callw putstr		#  prompt
      movb _OPT(%bp),%dl	# Display
      decw %si			#  default
      callw putkey		#  key
      jmp start_input		# Skip beep
```

stand/i386/boot0/boot0.S

最后，跳转到 start_input ，这里使用 BIOS 服务启动计时器和从键盘读取用户输入；如果计时器超时，将选择默认选项：

```
start_input:
      xorb %ah,%ah		# BIOS: Get
      int $0x1a			#  system time
      movw %dx,%di		# Ticks when
      addw _TICKS(%bp),%di	#  timeout
read_key:
      movb $0x1,%ah		# BIOS: Check
      int $0x16			#  for keypress
      jnz got_key		# Have input
      xorb %ah,%ah		# BIOS: int 0x1a, 00
      int $0x1a			#  get system time
      cmpw %di,%dx		# Timeout?
      jb read_key		# No
```

站/i386/boot0/boot0.S

当需要编号为 0x1a 和参数为 0 的中断时，通过 int 指令由应用程序请求由 BIOS 提供的一系列预定义服务，参数存储在寄存器中（在本例中为 %ah ）。在这里，特别是我们请求自午夜以来的时钟滴答数；BIOS 通过 RTC（实时时钟）计算此值。此时钟可以在范围从 2 赫兹到 8192 赫兹的频率下运行。BIOS 在启动时将其设置为 18.2 赫兹。当请求得到满足时，BIOS 会在寄存器 %cx 和 %dx 中返回一个 32 位结果（低字节在 %dx 中）。该结果（第 %dx 部分）会被复制到寄存器 %di 中，并且将 TICKS 变量的值加到 %di 中。该变量驻留在 boot0 中，距离寄存器 %bp 的偏移量为 _TICKS （一个负值）（回想一下，该寄存器指向 0x800 ）。该变量的默认值为 0xb6 （十进制中为 182）。现在，boot0 的想法是不断向 BIOS 请求时间，当在寄存器 %dx 中返回的值大于存储在 %di 中的值时，时间就到了，并且将进行默认选择。由于 RTC 每秒滴答 18.2 次，此条件将在 10 秒后满足（此默认行为可以在 Makefile 中更改）。在此时间过去之前，boot0 将不断通过 int 0x16 ，参数 1 ，在 %ah 中向 BIOS 请求任何用户输入。

不论是按下了键还是时间到了，后续代码会验证选择。根据选择，寄存器 %si 会被设置为指向分区表中的相应分区条目。这个新选择会覆盖之前的默认选择。事实上，它成为新的默认选择。最后，所选分区的 ACTIVE 标志会被设置。如果在编译时启用了此功能，则这些修改后的值的内存版本会写回磁盘上的 MBR。我们将此实现的细节留给读者。

我们现在以 boot0 程序中的最后一个代码块结束我们的研究：

```
      movw $LOAD,%bx		# Address for read
      movb $0x2,%ah		# Read sector
      callw intx13		#  from disk
      jc beep			# If error
      cmpw $MAGIC,0x1fe(%bx)	# Bootable?
      jne beep			# No
      pushw %si			# Save ptr to selected part.
      callw putn		# Leave some space
      popw %si			# Restore, next stage uses it
      jmp *%bx			# Invoke bootstrap
```

stand/i386/boot0/boot0.S

回想一下， %si 指向所选分区条目。该条目告诉我们分区在磁盘上的起始位置。当然，我们假设所选的分区实际上是一个 FreeBSD 切片。

|  | 从现在开始，我们将更倾向于使用技术上更准确的术语“切片”而不是“分区”。 |
| -- | -------------------------------------------------------------------------- |

传输缓冲区设置为 0x7c00 （寄存器 %bx ），并通过调用 intx13 请求读取 FreeBSD 切片的第一个扇区。我们假设一切顺利，因此不执行跳转到 beep 。特别是，新扇区读取必须以魔术序列 0xaa55 结束。最后， %si 处的值（指向所选分区表的指针）被保留供下一阶段使用，并跳转到地址 0x7c00 ，执行我们的下一阶段（刚读取的块）的执行开始。

## 1.5. boot1 阶段

到目前为止，我们已经按照以下顺序进行了：

* BIOS 进行了一些早期硬件初始化，包括 POST。MBR（boot0）从绝对磁盘扇区一加载到地址 0x7c00 。执行控制权被传递到该位置。
* boot0 将自身重定位到链接到执行的位置（ 0x600 ），然后跳转到适当位置继续执行。最后，boot0 从 FreeBSD 切片加载第一个磁盘扇区到地址 0x7c00 。执行控制被传递到该位置。

boot1 是引导加载序列中的下一步。它是三个引导阶段中的第一个。请注意，我们一直在处理磁盘扇区。实际上，BIOS 加载绝对的第一个扇区，而 boot0 加载 FreeBSD 切片的第一个扇区。这两个加载都是到地址 0x7c00 。我们可以在概念上将这些磁盘扇区视为分别包含文件 boot0 和 boot1，但实际上对于 boot1 来说这并不完全正确。严格来说，与 boot0 不同，boot1 不是引导块的一部分^[3]。相反，一个单独的完整文件 boot（/boot/boot）最终被写入磁盘。这个文件是 boot1、boot2 和 Boot Extender （或 BTX）的组合。这个单一文件的大小大于一个扇区（大于 512 字节）。幸运的是，boot1 恰好占据这个单一文件的前 512 字节，因此当 boot0 加载 FreeBSD 切片的第一个扇区（512 字节）时，实际上是加载 boot1 并将控制传递给它。

boot1 的主要任务是加载下一个引导阶段。这个下一个阶段稍微复杂一些。它由一个名为“Boot Extender”或 BTX 的服务器和一个名为 boot2 的客户端组成。正如我们将看到的，最后一个引导阶段 loader 也是 BTX 服务器的客户端。

现在让我们详细看看 boot1 到底做了什么，就像我们为 boot0 做的那样，在其入口点开始：

```
start:
	jmp main
```

stand/i386/boot2/boot1.S

start 处的入口点简单地跳过一个特殊数据区域到标签 main ，然后看起来像这样：

```
main:
      cld			# String ops inc
      xor %cx,%cx		# Zero
      mov %cx,%es		# Address
      mov %cx,%ds		#  data
      mov %cx,%ss		# Set up
      mov $start,%sp		#  stack
      mov %sp,%si		# Source
      mov $MEM_REL,%di		# Destination
      incb %ch			# Word count
      rep			# Copy
      movsw			#  code
```

stand/i386/boot2/boot1.S

就像 boot0 一样, 这段代码将 boot1 重新定位到内存地址 0x700 。 但与 boot0 不同的是, 它不会跳转到那里。 boot1 链接以在地址 0x7c00 处执行, 实际上就是它最初加载的位置。 关于这种重定位的原因很快会讨论。

接下来是一个循环，用于查找 FreeBSD 分区。 虽然 boot0 从 FreeBSD 分区加载了 boot1，但它未传递关于这一点的任何信息 ^[ 4]^，因此 boot1 必须重新扫描分区表以找到 FreeBSD 分区的起始位置。 因此, 它重新读取 MBR:

```
      mov $part4,%si		# Partition
      cmpb $0x80,%dl		# Hard drive?
      jb main.4			# No
      movb $0x1,%dh		# Block count
      callw nread		# Read MBR
```

stand/i386/boot2/boot1.S

在上面的代码中，寄存器 %dl 保存有关引导设备的信息。这是由 BIOS 传递的，并由 MBR 保留。数字 0x80 及更大的数字告诉我们正在处理硬盘，因此会调用 nread ，读取 MBR。参数传递给 nread 通过 %si 和 %dh 。标签 part4 处的内存地址被复制到 %si 。这个内存地址保存着一个供 nread 使用的“虚拟分区”。以下是虚拟分区中的数据：

```
      part4:
	.byte 0x80, 0x00, 0x01, 0x00
	.byte 0xa5, 0xfe, 0xff, 0xff
	.byte 0x00, 0x00, 0x00, 0x00
	.byte 0x50, 0xc3, 0x00, 0x00
```

stand/i386/boot2/boot1.S

特别是，此虚假分区的 LBA 硬编码为零。这用作 BIOS 读取硬盘绝对扇区一的参数。或者，可以使用 CHS 寻址。在这种情况下，虚假分区保留柱面 0、磁头 0 和扇区 1，这相当于绝对扇区一。

现在让我们继续看一下 nread ：

```
nread:
      mov $MEM_BUF,%bx		# Transfer buffer
      mov 0x8(%si),%ax		# Get
      mov 0xa(%si),%cx		#  LBA
      push %cs			# Read from
      callw xread.1		#  disk
      jnc return		# If success, return
```

stand/i386/boot2/boot1.S

回想一下， %si 指向虚假分区。偏移 0x8 处的字 ^[ 5]^ 被复制到寄存器 %ax ，偏移 0xa 处的字被复制到 %cx 。它们被 BIOS 解释为表示要读取的 LBA 的低 4 字节值（假定高 4 字节为零）。寄存器 %bx 存储着 MBR 将被加载的内存地址。将 %cs 推送到堆栈的指令非常有趣。在这种情况下，它什么也没做。然而，正如我们很快将看到的，boot2 与 BTX 服务器一起，也使用了 xread.1 。这个机制将在下一节中讨论。

xread.1 处的代码进一步调用 read 函数，实际上调用 BIOS 请求磁盘扇区：

```
xread.1:
	pushl $0x0		#  absolute
	push %cx		#  block
	push %ax		#  number
	push %es		# Address of
	push %bx		#  transfer buffer
	xor %ax,%ax		# Number of
	movb %dh,%al		#  blocks to
	push %ax		#  transfer
	push $0x10		# Size of packet
	mov %sp,%bp		# Packet pointer
	callw read		# Read from disk
	lea 0x10(%bp),%sp	# Clear stack
	lret			# To far caller
```

stand/i386/boot2/boot1.S

注意此块末尾的长返回指令。此指令弹出 %cs 寄存器，该寄存器是由 nread 推送的，并返回。最后， nread 也返回。

将 MBR 加载到内存后，开始搜索 FreeBSD 分区的实际循环：

```
	mov $0x1,%cx		 # Two passes
main.1:
	mov $MEM_BUF+PRT_OFF,%si # Partition table
	movb $0x1,%dh		 # Partition
main.2:
	cmpb $PRT_BSD,0x4(%si)	 # Our partition type?
	jne main.3		 # No
	jcxz main.5		 # If second pass
	testb $0x80,(%si)	 # Active?
	jnz main.5		 # Yes
main.3:
	add $0x10,%si		 # Next entry
	incb %dh		 # Partition
	cmpb $0x1+PRT_NUM,%dh		 # In table?
	jb main.2		 # Yes
	dec %cx			 # Do two
	jcxz main.1		 #  passes
```

stand/i386/boot2/boot1.S

如果检测到 FreeBSD 分区，执行会继续在 main.5 处。请注意，当找到 FreeBSD 分区时， %si 指向分区表中的适当条目， %dh 存储分区号。我们假设已经找到 FreeBSD 分区，所以在 main.5 处继续执行：

```
main.5:
	mov %dx,MEM_ARG			   # Save args
	movb $NSECT,%dh			   # Sector count
	callw nread			   # Read disk
	mov $MEM_BTX,%bx			   # BTX
	mov 0xa(%bx),%si		   # Get BTX length and set
	add %bx,%si			   #  %si to start of boot2.bin
	mov $MEM_USR+SIZ_PAG*2,%di			   # Client page 2
	mov $MEM_BTX+(NSECT-1)*SIZ_SEC,%cx			   # Byte
	sub %si,%cx			   #  count
	rep				   # Relocate
	movsb				   #  client
```

stand/i386/boot2/boot1.S

在这一点上，请回忆一下，寄存器 %si 指向 MBR 分区表中的 FreeBSD 分区条目，因此对 nread 的调用将有效地读取该分区开头的扇区。传递给寄存器 %dh 的参数告诉 nread 读取 16 个磁盘扇区。请回忆，前 512 字节，或者 FreeBSD 分区的第一个扇区，与 boot1 程序重合。还请回忆，写入 FreeBSD 分区开头的文件不是/boot/boot1，而是/boot/boot。让我们看一下这些文件在文件系统中的大小：

```
-r--r--r--  1 root  wheel   512B Jan  8 00:15 /boot/boot0
-r--r--r--  1 root  wheel   512B Jan  8 00:15 /boot/boot1
-r--r--r--  1 root  wheel   7.5K Jan  8 00:15 /boot/boot2
-r--r--r--  1 root  wheel   8.0K Jan  8 00:15 /boot/boot
```

boot0 和 boot1 各占用 512 个字节，因此它们恰好适合一个磁盘扇区。boot2 要大得多，包含 BTX 服务器和 boot2 客户端。最后，一个名为 boot 的文件比 boot2 大 512 个字节。该文件是 boot1 和 boot2 的串联。正如前面所指出的，boot0 是写入绝对第一个磁盘扇区（MBR）的文件，boot 是写入 FreeBSD 分区的第一个扇区的文件；boot1 和 boot2 并未写入磁盘。将 boot1 和 boot2 串联为单个 boot 的命令仅仅是 cat boot1 boot2 > boot 。

因此，boot1 恰好占用 boot 的前 512 个字节，并且，由于 boot 写入了 FreeBSD 分区的第一个扇区，boot1 正好适合于这个第一个扇区。当 nread 读取 FreeBSD 分区的前 16 个扇区时，实际上已经读取了整个 boot 文件。我们将在下一节详细了解 boot 是如何由 boot1 和 boot2 形成的。

回想一下， nread 使用存储器地址 0x8c00 作为传输缓冲区，用于保存读取的扇区。这个地址选择得很方便。的确，因为 boot1 属于前 512 个字节，它最终落入地址范围 0x8c00 - 0x8dff 内。随后的 512 个字节（范围 0x8e00 - 0x8fff ）用于存储 bsdlabel。

从地址 0x9000 开始是 BTX 服务器的起点，紧随其后的是 boot2 客户端。BTX 服务器充当内核，在最高特权级别的保护模式下执行。相比之下，BTX 客户端（例如 boot2）在用户模式下执行。我们将在下一节看到如何实现这一点。在调用 nread 后的代码定位内存缓冲区中 boot2 的起始位置，并将其复制到内存地址 0xc000 。这是因为 BTX 服务器安排 boot2 在从 0xa000 开始的段中执行。我们将在接下来的部分详细探讨这一点。

boot1 的最后一个代码块允许访问 1MB 以上的内存 ^[ 8]^，并以跳转到 BTX 服务器的起始点结束：

```
seta20:
	cli			# Disable interrupts
seta20.1:
	dec %cx			# Timeout?
	jz seta20.3		# Yes

	inb $0x64,%al		# Get status
	testb $0x2,%al		# Busy?
	jnz seta20.1		# Yes
	movb $0xd1,%al		# Command: Write
	outb %al,$0x64		#  output port
seta20.2:
	inb $0x64,%al		# Get status
	testb $0x2,%al		# Busy?
	jnz seta20.2		# Yes
	movb $0xdf,%al		# Enable
	outb %al,$0x60		#  A20
seta20.3:
	sti			# Enable interrupts
	jmp 0x9010		# Start BTX
```

stand/i386/boot2/boot1.S

注意，在跳转之前，中断被启用。

## 1.6. BTX 服务器

我们引导顺序中的下一个步骤是 BTX 服务器。让我们快速回顾一下我们是如何到达这里的：

* BIOS 加载绝对扇区一（MBR 或 boot0），跳转到地址 0x7c00 。
* boot0 会重新定位到 0x600 ，即链接执行的地址，并跳转到那里。然后，它将读取 FreeBSD 切片的第一个扇区（包含 boot1），加载到地址 0x7c00 ，然后跳转到那里。
* boot1 将 FreeBSD 切片的前 16 个扇区加载到地址 0x8c00 。这 16 个扇区，或 8192 字节，是整个文件 boot。该文件是 boot1 和 boot2 的串联。boot2 则包含 BTX 服务器和 boot2 客户端。最后，跳转到地址 0x9010 ，即 BTX 服务器的入口点。

在详细学习 BTX 服务器之前，让我们进一步审查单一的、一体化的引导文件是如何创建的。引导的构建方式是在其 Makefile（stand/i386/boot2/Makefile）中定义的。让我们看一下创建引导文件的规则：

```
      boot: boot1 boot2
	cat boot1 boot2 > boot
```

stand/i386/boot2/Makefile

这告诉我们需要 boot1 和 boot2，并且该规则简单地将它们连接起来以生成一个名为 boot 的单一文件。创建 boot1 的规则也很简单：

```
      boot1: boot1.out
	${OBJCOPY} -S -O binary boot1.out ${.TARGET}

      boot1.out: boot1.o
	${LD} ${LD_FLAGS} -e start --defsym ORG=${ORG1} -T ${LDSCRIPT} -o ${.TARGET} boot1.o
```

stand/i386/boot2/Makefile

要应用创建 boot1 的规则，必须解析 boot1.out。这又取决于 boot1.o 文件的存在。最后一个文件只是我们熟悉的 boot1.S 汇编的结果，没有链接。现在，应用创建 boot1.out 的规则。这告诉我们 boot1.o 应该与 start 作为其入口点链接，并从地址 0x7c00 开始。最后，通过将适当的规则应用于 boot1.out 创建 boot1。这个规则是应用于 boot1.out 的 objcopy 命令。注意传递给 objcopy 的标志： -S 告诉它剥离所有重定位和符号信息； -O binary 表示输出格式，即一个简单的、无格式的二进制文件。

有了 boot1，让我们看看 boot2 是如何构建的：

```
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

stand/i386/boot2/Makefile

构建 boot2 的机制要复杂得多。让我们指出最相关的事实。依赖列表如下：

```
      boot2: boot2.ld
      boot2.ld: boot2.ldr boot2.bin ${BTXDIR}
      boot2.bin: boot2.out
      boot2.out: ${BTXDIR} boot2.o sio.o ashldi3.o
      boot2.h: boot1.out
```

stand/i386/boot2/Makefile

请注意，最初没有头文件 boot2.h，但它的创建依赖于我们已经有的 boot1.out。其创建规则有点简洁，但重要的是输出文件 boot2.h 是这样的：

```
#define XREADORG 0x725
```

stand/i386/boot2/boot2.h

回忆一下，boot1 被重新定位（即，从 0x7c00 复制到 0x700 ）。这种重新定位现在有意义了，因为正如我们将看到的，BTX 服务器回收了一些内存，包括最初加载 boot1 的空间。然而，BTX 服务器需要访问 boot1 的 xread 函数；根据 boot2.h 的输出，这个函数位于 0x725 。确实，BTX 服务器使用 boot1 重新定位代码中的 xread 函数。这个函数现在可以从 boot2 客户端中访问。

下一个规则指示链接器链接各种文件（ashldi3.o、boot2.o 和 sio.o）。请注意，输出文件 boot2.out 被链接以在地址 0x2000 （$ {ORG2}）执行。回想一下，boot2 将在用户模式下执行，在 BTX 服务器设置的特殊用户段内。该段从 0xa000 开始。还记得 boot 中的 boot2 部分被复制到地址 0xc000 ，也就是从用户段开始的偏移 0x2000 ，所以当我们转移控制到它时，boot2 将正常工作。接下来，boot2.bin 是通过删除其符号和格式信息从 boot2.out 创建的；boot2.bin 是一个原始二进制文件。现在，请注意一个名为 boot2.ldr 的文件创建为一个全是零的 512 字节文件。这个空间是留给 bsdlabel 的。

现在我们拥有 boot1、boot2.bin 和 boot2.ldr 文件，在创建这个全合一启动文件之前，唯一缺失的是 BTX 服务器。BTX 服务器位于 stand/i386/btx/btx；它有自己的 Makefile，其中包含了用于构建的一套规则。需要注意的重要一点是，它也被编译为原始二进制文件，并被链接以在地址 0x9000 执行。详细信息可以在 stand/i386/btx/btx/Makefile 中找到。

拥有构成启动程序的文件后，最后一步是合并它们。这由一个名为 btxld 的特殊程序完成（源文件位于/usr/src/usr.sbin/btxld）。该程序的一些参数包括输出文件的名称（boot）、其入口点（ 0x2000 ）和文件格式（原始二进制）。这个实用程序最终将各种文件合并为 boot 文件，其中包括 boot1、boot2、 bsdlabel 和 BTX 服务器。这个文件正好占据 16 个扇区或 8192 字节，实际上是在安装期间写入 FreeBSD 分区的开始部分。现在让我们继续研究 BTX 服务器程序。

BTX 服务器在将控制权交给客户端之前，准备一个简单的环境，并从 16 位实模式切换到 32 位保护模式。这包括初始化和更新以下数据结构：

* 修改 Interrupt Vector Table (IVT) 。IVT 为实模式代码提供异常和中断处理程序。
* 创建 Interrupt Descriptor Table (IDT) 。为处理器异常、硬件中断、两个系统调用和 V86 接口提供条目。IDT 为保护模式代码提供异常和中断处理程序。
* 创建了一个 Task-State Segment (TSS) 。这是必要的，因为处理器在执行客户端（boot2）时工作在最低特权级别，但在执行 BTX 服务器时工作在最高特权级别。
* 设置了 GDT（全局描述符表）。为监管者代码和数据、用户代码和数据以及实模式代码和数据提供了条目（描述符）。 ^[ 9]^

现在让我们开始研究实际实现。回想一下，boot1 跳转到地址 0x9010 ，即 BTX 服务器的入口点。在那里研究程序执行之前，请注意 BTX 服务器在地址范围 0x9000-0x900f 之前有一个特殊的头。这个头的定义如下：

```
start:						# Start of code
/*
 * BTX header.
 */
btx_hdr:	.byte 0xeb			# Machine ID
		.byte 0xe			# Header size
		.ascii "BTX"			# Magic
		.byte 0x1			# Major version
		.byte 0x2			# Minor version
		.byte BTX_FLAGS			# Flags
		.word PAG_CNT-MEM_ORG>>0xc	# Paging control
		.word break-start		# Text size
		.long 0x0			# Entry address
```

站/i386/btx/btx/btx.S

注意，前两个字节分别为 0xeb 和 0xe 。在 IA-32 架构中，这两个字节被解释为相对于头部跳转到入口点，因此理论上，boot1 可以跳转到这里（地址 0x9000 ）而不是地址 0x9010 。请注意，BTX 头部中的最后一个字段是指向客户端（boot2）入口点 b2 的指针。此字段在链接时被修补。

紧随头部之后是 BTX 服务器的入口点：

```
/*
 * Initialization routine.
 */
init:		cli				# Disable interrupts
		xor %ax,%ax			# Zero/segment
		mov %ax,%ss			# Set up
		mov $MEM_ESP0,%sp		#  stack
		mov %ax,%es			# Address
		mov %ax,%ds			#  data
		pushl $0x2			# Clear
		popfl				#  flags
```

stand/i386/btx/btx/btx.S

此代码禁用中断，设置工作堆栈（从地址 0x1800 开始），并清除 EFLAGS 寄存器中的标志位。请注意， popfl 指令从堆栈中弹出一个双字（4 字节）并将其放入 EFLAGS 寄存器中。由于实际弹出的值为 2 ，因此 EFLAGS 寄存器实际上被清除（IA-32 要求 EFLAGS 寄存器的第 2 位始终为 1）。

我们的下一个代码块清除（设置为 0 ）内存范围 0x5e00-0x8fff 。这个范围是各种数据结构将被创建的地方：

```
/*
 * Initialize memory.
 */
		mov $MEM_IDT,%di		# Memory to initialize
		mov $(MEM_ORG-MEM_IDT)/2,%cx	# Words to zero
		rep				# Zero-fill
		stosw				#  memory
```

FreeBSD/i386/btx/btx/btx.S

回想一下，boot1 最初加载到地址 0x7c00 ，因此，随着这个内存初始化，该副本实际上消失了。但是，也要记住，boot1 已经重新定位到 0x700 ，因此该副本仍然在内存中，并且 BTX 服务器将利用它。

接下来，实模式 IVT（中断向量表）被更新。IVT 是一组用于异常和中断处理程序的段/偏移对数组。BIOS 通常将硬件中断映射到中断向量 0x8 到 0xf 和 0x70 到 0x77 ，但是，正如将要看到的，8259A 可编程中断控制器，控制实际将硬件中断映射到中断向量的芯片，被编程为将这些中断向量重新映射从 0x8-0xf 到 0x20-0x27 和从 0x70-0x77 到 0x28-0x2f 。因此，为中断向量 0x20-0x2f 提供了中断处理程序。之所以不直接使用 BIOS 提供的处理程序是因为它们在 16 位实模式下工作，但不在 32 位保护模式下工作。处理器模式将很快切换到 32 位保护模式。但是，BTX 服务器建立了一种机制，有效地使用 BIOS 提供的处理程序：

```
/*
 * Update real mode IDT for reflecting hardware interrupts.
 */
		mov $intr20,%bx			# Address first handler
		mov $0x10,%cx			# Number of handlers
		mov $0x20*4,%di			# First real mode IDT entry
init.0:		mov %bx,(%di)			# Store IP
		inc %di				# Address next
		inc %di				#  entry
		stosw				# Store CS
		add $4,%bx			# Next handler
		loop init.0			# Next IRQ
```

站/i386/btx/btx/btx.S

下一个块创建 IDT（中断描述符表）。在保护模式中，IDT 类似于实模式中的 IVT。也就是说，在处理器在保护模式下执行时，IDT 描述了使用的各种异常和中断处理程序。实质上，它也由一系列段/偏移对组成，尽管结构略微更加复杂，因为在保护模式下，段与实模式下的段不同，并且各种保护机制适用：

```
/*
 * Create IDT.
 */
		mov $MEM_IDT,%di		# IDT's address
		mov $idtctl,%si			# Control string
init.1:		lodsb				# Get entry
		cbw				#  count
		xchg %ax,%cx			#  as word
		jcxz init.4			# If done
		lodsb				# Get segment
		xchg %ax,%dx			#  P:DPL:type
		lodsw				# Get control
		xchg %ax,%bx			#  set
		lodsw				# Get handler offset
		mov $SEL_SCODE,%dh		# Segment selector
init.2:		shr %bx				# Handle this int?
		jnc init.3			# No
		mov %ax,(%di)			# Set handler offset
		mov %dh,0x2(%di)		#  and selector
		mov %dl,0x5(%di)		# Set P:DPL:type
		add $0x4,%ax			# Next handler
init.3:		lea 0x8(%di),%di		# Next entry
		loop init.2			# Till set done
		jmp init.1			# Continue
```

站/i386/btx/btx/btx.S

IDT 中的每个条目都有 8 字节长。除了段/偏移信息外，它们还描述段类型、特权级别以及段是否存在于内存中。构造如下：从 0 到 0xf 的中断向量（异常）由函数 intx00 处理；向量 0x10 （同样是异常）由 intx10 处理；稍后配置为从中断向量 0x20 开始直到中断向量 0x2f 的硬件中断由函数 intx20 处理。最后，用于系统调用的中断向量 0x30 由 intx30 处理，向量 0x31 和 0x32 由 intx31 处理。必须注意的是，只有中断向量 0x30 、 0x31 和 0x32 的描述符被赋予特权级别 3，与 boot2 客户端相同特权级别，这意味着客户端可以通过 int 指令执行对这些向量的软件生成中断而不会失败（这是 boot2 使用 BTX 服务器提供的服务的方式）。此外，请注意，只有软件生成的中断受保护，免受在较低特权级别执行代码的影响。硬件生成的中断和处理器生成的异常始终得到充分处理，无论涉及的实际特权级别为何。

下一步是初始化 TSS（任务状态段）。TSS 是一个硬件特性，通过处理抽象进程，帮助操作系统或执行软件实现多任务处理功能。如果使用多任务处理设施或定义不同的特权级别，IA-32 架构要求创建并使用至少一个 TSS。由于 boot2 客户端在特权级别 3 中执行，但 BTX 服务器在特权级别 0 中运行，因此必须定义一个 TSS：

```
/*
 * Initialize TSS.
 */
init.4:		movb $_ESP0H,TSS_ESP0+1(%di)	# Set ESP0
		movb $SEL_SDATA,TSS_SS0(%di)	# Set SS0
		movb $_TSSIO,TSS_MAP(%di)	# Set I/O bit map base
```

stand/i386/btx/btx/btx.S

注意，在 TSS 中为特权级 0 的堆栈指针和堆栈段提供了一个值。这是必要的，因为如果在特权级 3 中执行 boot2 时收到中断或异常，处理器会自动执行到特权级 0，因此需要一个新的工作堆栈。最后，TSS 的 I/O 映射基地址字段被赋予一个值，这是从 TSS 开头到 I/O 权限位图和中断重定向位图的 16 位偏移量。

在创建 IDT 和 TSS 之后，处理器已准备好切换到保护模式。这在下一个块中完成：

```
/*
 * Bring up the system.
 */
		mov $0x2820,%bx			# Set protected mode
		callw setpic			#  IRQ offsets
		lidt idtdesc			# Set IDT
		lgdt gdtdesc			# Set GDT
		mov %cr0,%eax			# Switch to protected
		inc %ax				#  mode
		mov %eax,%cr0			#
		ljmp $SEL_SCODE,$init.8		# To 32-bit code
		.code32
init.8:		xorl %ecx,%ecx			# Zero
		movb $SEL_SDATA,%cl		# To 32-bit
		movw %cx,%ss			#  stack
```

stand/i386/btx/btx/btx.S

首先，调用 setpic 来编程 8259A PIC（可编程中断控制器）。该芯片连接到多个硬件中断源。当从设备接收到中断时，它会向处理器发出适当的中断向量信号。这可以定制，以便特定中断与特定中断向量相关联，如前所述。接下来，将 IDTR（中断描述符表寄存器）和 GDTR（全局描述符表寄存器）加载为分别为 lidt 和 lgdt 的指令。这些寄存器加载了 IDT 和 GDT 的基地址和限制地址。接下来的三条指令设置 %cr0 寄存器的保护使能（PE）位。这有效地将处理器切换到 32 位保护模式。接下来，通过使用段选择器 SEL_SCODE 进行长跳转到 init.8 。这选择了监控代码段。在此跳转后，处理器有效地在 CPL 0 中执行，即最高特权级别。最后，通过将段选择器 SEL_SDATA 分配给 %ss 寄存器，为堆栈选择了监控数据段。该数据段的特权级别也为 0 。

我们的最后一个代码块负责使用我们之前创建的 TSS 的段选择器将 TR（任务寄存器）加载，并在将执行控制传递给 boot2 客户端之前设置用户模式环境。

```
/*
 * Launch user task.
 */
		movb $SEL_TSS,%cl		# Set task
		ltr %cx				#  register
		movl $MEM_USR,%edx		# User base address
		movzwl %ss:BDA_MEM,%eax		# Get free memory
		shll $0xa,%eax			# To bytes
		subl $ARGSPACE,%eax		# Less arg space
		subl %edx,%eax			# Less base
		movb $SEL_UDATA,%cl		# User data selector
		pushl %ecx			# Set SS
		pushl %eax			# Set ESP
		push $0x202			# Set flags (IF set)
		push $SEL_UCODE			# Set CS
		pushl btx_hdr+0xc		# Set EIP
		pushl %ecx			# Set GS
		pushl %ecx			# Set FS
		pushl %ecx			# Set DS
		pushl %ecx			# Set ES
		pushl %edx			# Set EAX
		movb $0x7,%cl			# Set remaining
init.9:		push $0x0			#  general
		loop init.9			#  registers
#ifdef BTX_SERIAL
		call sio_init			# setup the serial console
#endif
		popa				#  and initialize
		popl %es			# Initialize
		popl %ds			#  user
		popl %fs			#  segment
		popl %gs			#  registers
		iret				# To user mode
```

stand/i386/btx/btx/btx.S

请注意，客户端的环境包括堆栈段选择器和堆栈指针（寄存器 %ss 和 %esp ）。实际上，一旦 TR 加载了适当的堆栈段选择器（指令 ltr ），堆栈指针就会被计算并推送到堆栈上，同时推送堆栈的段选择器。接下来，值 0x202 被推送到堆栈上；这是当控制权传递给客户端时 EFLAGS 将获得的值。此外，用户模式代码段选择器和客户端的入口点也被推送。请记住，此入口点在链接时在 BTX 标头中进行了修补。最后，段选择器（存储在寄存器 %ecx 中）用于段寄存器 %gs, %fs, %ds and %es 也被推送到堆栈上，以及 %edx （ 0xa000 ）处的值。请记住已经推送到堆栈上的各种值（它们很快将被弹出）。接下来，剩余通用寄存器的值也被推送到堆栈上（请注意 loop 会将值 0 推送七次）。现在，值将开始从堆栈中弹出。首先， popa 指令从堆栈中弹出最近推送的七个值。它们按顺序存储在通用寄存器中 %edi, %esi, %ebp, %ebx, %edx, %ecx, %eax 。然后，推送的各种段选择器被弹出到各种段寄存器中。堆栈上仍然有五个值。当执行 iret 指令时，它们将被弹出。此指令首先弹出从 BTX 标头推送的值。此值是指向 boot2 入口点的指针。它被放置在寄存器 %eip 中，即指令指针寄存器。接下来，用户代码段的段选择器被弹出并复制到寄存器 %cs 中。请记住，此段的特权级别为 3，是最低特权级别。这意味着我们必须为此特权级别的堆栈提供值。这就是处理器除了进一步从堆栈中弹出 EFLAGS 寄存器的值外，还要从堆栈中弹出两个值的原因。 这些值输入堆栈指针（ %esp ）和堆栈段（ %ss ）。现在，执行继续在 boot0 的入口点。

注意用户代码段的定义方式很重要。该段的基址设置为 0xa000 。这意味着代码内存地址是相对于地址 0xa000 的；如果执行的代码是从地址 0x2000 获取的，实际内存地址为 0xa000+0x2000=0xc000 。

## 1.7. 启动阶段

boot2 定义了一个重要的结构， struct bootinfo 。这个结构由 boot2 初始化并传递给加载程序，然后进一步传递给内核。这个结构的一些节点由 boot2 设置，其余由加载程序设置。除其他信息外，这个结构包含内核文件名、BIOS 硬盘几何、用于引导设备的 BIOS 驱动器号、可用物理内存、 envp 指针等。它的定义如下：

```
/usr/include/machine/bootinfo.h:
struct bootinfo {
	u_int32_t	bi_version;
	u_int32_t	bi_kernelname;		/* represents a char * */
	u_int32_t	bi_nfs_diskless;	/* struct nfs_diskless * */
				/* End of fields that are always present. */
#define	bi_endcommon	bi_n_bios_used
	u_int32_t	bi_n_bios_used;
	u_int32_t	bi_bios_geom[N_BIOS_GEOM];
	u_int32_t	bi_size;
	u_int8_t	bi_memsizes_valid;
	u_int8_t	bi_bios_dev;		/* bootdev BIOS unit number */
	u_int8_t	bi_pad[2];
	u_int32_t	bi_basemem;
	u_int32_t	bi_extmem;
	u_int32_t	bi_symtab;		/* struct symtab * */
	u_int32_t	bi_esymtab;		/* struct symtab * */
				/* Items below only from advanced bootloader */
	u_int32_t	bi_kernend;		/* end of kernel space */
	u_int32_t	bi_envp;		/* environment */
	u_int32_t	bi_modulep;		/* preloaded modules */
};
```

boot2 进入一个无限循环，等待用户输入，然后调用 load() 。如果用户没有按任何键，循环会因超时而中断，因此 load() 将加载默认文件（/boot/loader）。函数 ino_t lookup(char *filename) 和 int xfsread(ino_t inode, void *buf, size_t nbyte) 用于将文件内容读入内存。/boot/loader 是一个 ELF 二进制文件，但 ELF 头部前面附加了 a.out 的 struct exec 结构。 load() 扫描加载程序的 ELF 头部，将/boot/loader 的内容加载到内存中，并将执行传递给加载程序的入口：

```
stand/i386/boot2/boot2.c:
    __exec((caddr_t)addr, RB_BOOTINFO | (opts & RBX_MASK),
	   MAKEBOOTDEV(dev_maj[dsk.type], dsk.slice, dsk.unit, dsk.part),
	   0, 0, 0, VTOP(&bootinfo));
```

## 1.8. 加载程序阶段

加载程序也是一个 BTX 客户端。我在这里不会详细描述它，有一个由 Mike Smith 编写的全面的手册，加载程序(8)。底层机制和 BTX 在前面已经讨论过。

加载程序的主要任务是引导内核。当内核被加载到内存中时，加载程序会调用它：

```
stand/common/boot.c:
    /* Call the exec handler from the loader matching the kernel */
    file_formats[fp->f_loader]->l_exec(fp);
```

## 1.9. 内核初始化

让我们看一下链接内核的命令。这将帮助确定加载程序将执行传递给内核的确切位置。这个位置是内核的实际入口点。此命令现已从 sys/conf/Makefile.i386 中排除。我们感兴趣的内容可以在/usr/obj/usr/src/i386.i386/sys/GENERIC/找到。

```
/usr/obj/usr/src/i386.i386/sys/GENERIC/kernel.meta:
ld -m elf_i386_fbsd -Bdynamic -T /usr/src/sys/conf/ldscript.i386 --build-id=sha1 --no-warn-mismatch \
--warn-common --export-dynamic  --dynamic-linker /red/herring -X -o kernel locore.o
<lots of kernel .o files>
```

这里可以看到一些有趣的东西。首先，内核是一个 ELF 动态链接的二进制文件，但内核的动态链接器是/red/herring，这绝对是一个虚假文件。其次，查看文件 sys/conf/ldscript.i386 可以了解在编译内核时使用了哪些 ld 选项。通过阅读前几行，字符串

```
sys/conf/ldscript.i386:
ENTRY(btext)
```

表明内核的入口点是符号 btext 。这个符号在 locore.s 中定义：

```
sys/i386/i386/locore.s:
	.text
/**********************************************************************
 *
 * This is where the bootblocks start us, set the ball rolling...
 *
 */
NON_GPROF_ENTRY(btext)
```

首先，寄存器 EFLAGS 被设置为预定义值 0x00000002。然后初始化所有段寄存器：

```
sys/i386/i386/locore.s:
/* Don't trust what the BIOS gives for eflags. */
	pushl	$PSL_KERNEL
	popfl

/*
 * Don't trust what the BIOS gives for %fs and %gs.  Trust the bootstrap
 * to set %cs, %ds, %es and %ss.
 */
	mov	%ds, %ax
	mov	%ax, %fs
	mov	%ax, %gs
```

btext 调用了在 locore.s 中也定义的例程 recover_bootinfo() ， identify_cpu() 。以下是它们的功能描述：

| `recover_bootinfo` | 此例程解析从引导加载程序传递给内核的参数。内核可能以 3 种方式引导：通过上述的加载程序、通过旧的磁盘引导块，或通过旧的无盘引导过程。此函数确定引导方法，并将 struct bootinfo 结构存储到内核内存中。 |
| -- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `identify_cpu` | 此函数尝试查找其正在运行的 CPU，并将找到的值存储在变量 _cpu 中。                                                                                                                                   |

接下来的步骤是启用 VME，如果 CPU 支持的话：

```
sys/i386/i386/mpboot.s:
	testl	$CPUID_VME,%edx
	jz	3f
	orl	$CR4_VME,%eax
3:	movl	%eax,%cr4
```

 然后，启用分页：

```
sys/i386/i386/mpboot.s:
/* Now enable paging */
	movl	IdlePTD_nopae, %eax
	movl	%eax,%cr3			/* load ptd addr into mmu */
	movl	%cr0,%eax			/* get control word */
	orl	$CR0_PE|CR0_PG,%eax		/* enable paging */
	movl	%eax,%cr0			/* and let's page NOW! */
```

下面的三行代码是因为设置了分页，所以需要跳转才能在虚拟地址空间中继续执行：

```
sys/i386/i386/mpboot.s:
	pushl	$mp_begin				/* jump to high mem */
	ret

/* now running relocated at KERNBASE where the system is linked to run */
mp_begin:	/* now running relocated at KERNBASE */
```

函数 init386() 被调用，传递了一个指向第一个空闲物理页的指针，之后 mi_startup() 。 init386 是一个与体系结构相关的初始化函数， mi_startup() 是一个与体系结构无关的函数（'mi_'前缀代表 Machine Independent）。内核从不返回 mi_startup() ，通过调用它，内核完成引导：

```
sys/i386/i386/locore.s:
	pushl	physfree			/* value of first for init386(first) */
	call	init386				/* wire 386 chip for unix operation */
	addl	$4,%esp
	movl	%eax,%esp			/* Switch to true top of stack. */
	call	mi_startup			/* autoconfiguration, mountroot etc */
	/* NOTREACHED */
```

### 1.9.1. `init386()`

init386() 在 sys/i386/i386/machdep.c 中定义，执行特定于 i386 芯片的低级初始化。切换到保护模式是由加载器完成的。加载器创建了第一个任务，在其中内核继续操作。在查看代码之前，请考虑处理器必须完成的任务以初始化保护模式执行：

* 初始化内核可调参数，从引导程序传递而来。
* 准备 GDT。
* 准备 IDT。
* 初始化系统控制台。
* 如果编译到内核中，请初始化 DDB。
* 初始化 TSS。
* 准备 LDT。
* 设置 thread0 的 PCB。

init386() 通过设置环境指针（envp）并调用 init_param1() 来初始化从引导加载程序传递的可调参数。envp 指针已经从加载程序传递给了 bootinfo 结构：

```
sys/i386/i386/machdep.c:
	/* Init basic tunables, hz etc */
	init_param1();
```

init_param1() 定义在 sys/kern/subr_param.c 中。该文件有许多 sysctl，还有两个函数， init_param1() 和 init_param2() ，它们从 init386() 调用：

```
sys/kern/subr_param.c:
	hz = -1;
	TUNABLE_INT_FETCH("kern.hz", &hz);
	if (hz == -1)
		hz = vm_guest > VM_GUEST_NO ? HZ_VM : HZ;
```

TUNABLE__FETCH 用于从环境中获取值：

```
/usr/src/sys/sys/kernel.h:
#define	TUNABLE_INT_FETCH(path, var)	getenv_int((path), (var))
```

Sysctl kern.hz 是系统时钟滴答。此外，这些 sysctl 由 init_param1() 设置： kern.maxswzone, kern.maxbcache, kern.maxtsiz, kern.dfldsiz, kern.maxdsiz, kern.dflssiz, kern.maxssiz, kern.sgrowsiz 。

然后 init386() 准备全局描述符表（GDT）。 x86 上的每个任务都运行在自己的虚拟地址空间中，该空间由段：偏移对进行寻址。 例如，处理器即将执行的当前指令位于 CS：EIP，那么该指令的线性虚拟地址将是“代码段 CS 的虚拟地址”+ EIP。 为方便起见，段从虚拟地址 0 开始，并以 4GB 边界结束。 因此，对于此示例，指令的线性虚拟地址将仅为 EIP 的值。 诸如 CS、DS 等段寄存器是选择器，即 GDT 中的索引。 FreeBSD 的 GDT 为每个 CPU 保留了 15 个选择器的描述符：

```
sys/i386/i386/machdep.c:
union descriptor gdt0[NGDT];	/* initial global descriptor table */
union descriptor *gdt = gdt0;	/* global descriptor table */

sys/x86/include/segments.h:
/*
 * Entries in the Global Descriptor Table (GDT)
 */
#define	GNULL_SEL	0	/* Null Descriptor */
#define	GPRIV_SEL	1	/* SMP Per-Processor Private Data */
#define	GUFS_SEL	2	/* User %fs Descriptor (order critical: 1) */
#define	GUGS_SEL	3	/* User %gs Descriptor (order critical: 2) */
#define	GCODE_SEL	4	/* Kernel Code Descriptor (order critical: 1) */
#define	GDATA_SEL	5	/* Kernel Data Descriptor (order critical: 2) */
#define	GUCODE_SEL	6	/* User Code Descriptor (order critical: 3) */
#define	GUDATA_SEL	7	/* User Data Descriptor (order critical: 4) */
#define	GBIOSLOWMEM_SEL	8	/* BIOS low memory access (must be entry 8) */
#define	GPROC0_SEL	9	/* Task state process slot zero and up */
#define	GLDT_SEL	10	/* Default User LDT */
#define	GUSERLDT_SEL	11	/* User LDT */
#define	GPANIC_SEL	12	/* Task state to consider panic from */
#define	GBIOSCODE32_SEL	13	/* BIOS interface (32bit Code) */
#define	GBIOSCODE16_SEL	14	/* BIOS interface (16bit Code) */
#define	GBIOSDATA_SEL	15	/* BIOS interface (Data) */
#define	GBIOSUTIL_SEL	16	/* BIOS interface (Utility) */
#define	GBIOSARGS_SEL	17	/* BIOS interface (Arguments) */
#define	GNDIS_SEL	18	/* For the NDIS layer */
#define	NGDT		19
```

请注意，这些#define 本身并不是选择器，而只是选择器的字段 INDEX，因此它们恰好是 GDT 的索引。 例如，内核代码（GCODE_SEL）的实际选择器值为 0x20。

下一步是初始化中断描述符表（IDT）。 当软件或硬件中断发生时，处理器将引用此表。 例如，要进行系统调用，用户应用程序发出 INT 0x80 指令。 这是软件中断，因此处理器的硬件会查找 IDT 中索引为 0x80 的记录。 该记录指向处理此中断的例程，在这种特殊情况下，这将是内核的系统调用门。 IDT 最多可以有 256（0x100）个记录。 内核为 IDT 分配 NIDT 记录，其中 NIDT 为最大值（256）:

```
sys/i386/i386/machdep.c:
static struct gate_descriptor idt0[NIDT];
struct gate_descriptor *idt = &idt0[0];	/* interrupt descriptor table */
```

对于每个中断，都设置了适当的处理程序。 INT 0x80 的系统调用门也被设置:

```
sys/i386/i386/machdep.c:
	setidt(IDT_SYSCALL, &IDTVEC(int0x80_syscall),
			SDT_SYS386IGT, SEL_UPL, GSEL(GCODE_SEL, SEL_KPL));
```

因此，当用户应用程序发出 INT 0x80 指令时，控制权将转移到位于内核代码段中并以监督者特权执行的函数 _Xint0x80_syscall 。

然后初始化控制台和 DDB:

```
sys/i386/i386/machdep.c:
	cninit();
/* skipped */
  kdb_init();
#ifdef KDB
	if (boothowto & RB_KDB)
		kdb_enter(KDB_WHY_BOOTFLAGS, "Boot flags requested debugger");
#endif
```

任务状态段是另一个 x86 保护模式结构，TSS 用于在任务切换发生时由硬件存储任务信息。

本地描述符表用于引用用户空间代码和数据。定义了几个选择器指向 LDT，它们是系统调用门和用户代码和数据选择器：

```
sys/x86/include/segments.h:
#define	LSYS5CALLS_SEL	0	/* forced by intel BCS */
#define	LSYS5SIGR_SEL	1
#define	LUCODE_SEL	3
#define	LUDATA_SEL	5
#define	NLDT		(LUDATA_SEL + 1)
```

接下来，proc0 的进程控制块（ struct pcb ）结构被初始化。proc0 是一个 struct proc 结构，描述了一个内核进程。它在内核运行时始终存在，因此与 thread0 相关联：

```
sys/i386/i386/machdep.c:
register_t
init386(int first)
{
    /* ... skipped ... */

    proc_linkup0(&proc0, &thread0);
    /* ... skipped ... */
}
```

结构 struct pcb 是 proc 结构的一部分。它在 /usr/include/machine/pcb.h 中定义，包含特定于 i386 架构的进程信息，如寄存器值。

### 1.9.2. `mi_startup()`

此函数对所有系统初始化对象执行冒泡排序，然后依次调用每个对象的入口：

```
sys/kern/init_main.c:
	for (sipp = sysinit; sipp < sysinit_end; sipp++) {

		/* ... skipped ... */

		/* Call function */
		(*((*sipp)->func))((*sipp)->udata);
		/* ... skipped ... */
	}
```

尽管 sysinit 框架在开发者手册中有描述，我将讨论其中的内部。

每个系统初始化对象（sysinit 对象）都是通过调用 SYSINIT（）宏创建的。让我们以 announce sysinit 对象为例。此对象打印版权信息：

```
sys/kern/init_main.c:
static void
print_caddr_t(void *data __unused)
{
	printf("%s", (char *)data);
}
/* ... skipped ... */
SYSINIT(announce, SI_SUB_COPYRIGHT, SI_ORDER_FIRST, print_caddr_t, copyright);
```

此对象的子系统 ID 为 SI_SUB_COPYRIGHT（0x0800001）。因此，版权信息将在控制台初始化后首先打印出来。

让我们看看宏 SYSINIT() 实际上是做什么的。它扩展为 C_SYSINIT() 宏。然后 C_SYSINIT() 宏扩展为具有另一个 DATA_SET 宏调用的静态 struct sysinit 结构声明：

```
/usr/include/sys/kernel.h:
      #define C_SYSINIT(uniquifier, subsystem, order, func, ident) \
      static struct sysinit uniquifier ## _sys_init = { \ subsystem, \
      order, \ func, \ (ident) \ }; \ DATA_WSET(sysinit_set,uniquifier ##
      _sys_init);

#define	SYSINIT(uniquifier, subsystem, order, func, ident)	\
	C_SYSINIT(uniquifier, subsystem, order,			\
	(sysinit_cfunc_t)(sysinit_nfunc_t)func, (void *)(ident))
```

DATA_SET() 宏扩展为 _MAKE_SET() ，该宏是隐藏所有 sysinit 魔术的地方：

```
/usr/include/linker_set.h:
#define TEXT_SET(set, sym) _MAKE_SET(set, sym)
#define DATA_SET(set, sym) _MAKE_SET(set, sym)
```

执行这些宏后，在内核中创建了各种部分，包括 set.sysinit_set 。在内核二进制文件上运行 objdump，您可能会注意到这些小节的存在：

```
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

此屏幕转储显示 set.sysinit_set 部分的大小为 0x14d8 字节，因此 0x14d8/sizeof(void *) sysinit 对象被编译到内核中。其他部分，如 set.sysctl_set ，代表其他链接器集合。

通过定义一个类型为 struct sysinit 的变量， set.sysinit_set 部分的内容将被"收集"到该变量中：

```
sys/kern/init_main.c:
  SET_DECLARE(sysinit_set, struct sysinit);
```

struct sysinit 定义如下：

```
sys/sys/kernel.h:
  struct sysinit {
	enum sysinit_sub_id	subsystem;	/* subsystem identifier*/
	enum sysinit_elem_order	order;		/* init order within subsystem*/
	sysinit_cfunc_t func;			/* function		*/
	const void	*udata;			/* multiplexer/argument */
};
```

回到 mi_startup() 讨论，现在必须清楚了，sysinit 对象是如何被组织的。 mi_startup() 函数对它们进行排序并调用每一个。最后一个对象是系统调度程序：

```
/usr/include/sys/kernel.h:
enum sysinit_sub_id {
	SI_SUB_DUMMY		= 0x0000000,	/* not executed; for linker*/
	SI_SUB_DONE		= 0x0000001,	/* processed*/
	SI_SUB_TUNABLES		= 0x0700000,	/* establish tunable values */
	SI_SUB_COPYRIGHT	= 0x0800001,	/* first use of console*/
...
	SI_SUB_LAST		= 0xfffffff	/* final initialization */
};
```

系统调度程序 sysinit 对象在文件 sys/vm/vm_glue.c 中定义，该对象的入口点是 scheduler() 。该函数实际上是一个无限循环，代表进程 PID 0，即交换进程。之前提到的 thread0 结构用于描述它。

第一个用户进程称为 init，由 sysinit 对象 init 创建：

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
	/* divorce init's credentials from the kernel's */
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

函数 create_init() 通过调用 fork1() 分配一个新进程，但不标记为可运行。当调度程序安排执行此新进程时，将调用 start_init() 。该函数在 init_main.c 中定义。它尝试加载和执行 init 二进制文件，首先探测/sbin/init，然后是/sbin/oinit，/sbin/init.bak，最后是/rescue/init：

```
sys/kern/init_main.c:
static char init_path[MAXPATHLEN] =
#ifdef	INIT_PATH
    __XSTRING(INIT_PATH);
#else
    "/sbin/init:/sbin/oinit:/sbin/init.bak:/rescue/init";
#endif
```

---

1. 用户在 boot0 阶段选择要引导的操作系统后，如果按键，将出现此提示。
2. 如果有疑问，我们建议读者参考官方英特尔手册，其中描述了每条指令的确切语义。
3. 存在一个文件 /boot/boot1，但它并非写入到 FreeBSD 切片的开头。相反，它与 boot2 连接在一起形成 boot，后者写入到 FreeBSD 切片的开头，并在引导时读取。

实际上，我们确实向寄存器%si 中的切片条目传递了指针。 但是，boot1 不假设它是由 boot0 加载的（也许是其他某个 MBR 加载了它，并且没有传递这些信息），因此它不做任何假设。

在 16 位实模式的背景下，一个字是 2 个字节。

512*16=8192 字节，正好是 boot 的大小。

7. 历史上被称为磁盘标签。如果你曾经想知道 FreeBSD 将这些信息存储在哪里，它就在这个区域 - 参见 bsdlabel(8)
8. 由于传统原因，这是必要的。感兴趣的读者应参阅。
9. 当从保护模式切换回实模式时，根据英特尔手册的建议，实模式代码和数据是必要的。
