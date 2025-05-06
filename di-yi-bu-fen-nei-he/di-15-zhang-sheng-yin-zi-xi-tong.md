# 第 15 章 声音子系统

## 15.1. 介绍

FreeBSD 的声音子系统将通用的声音处理问题与设备特定问题分开，这使得为新硬件添加支持变得更容易。

[pcm(4)](https://man.freebsd.org/cgi/man.cgi?query=pcm&sektion=4&format=html) 框架是声音子系统的核心部分。它主要实现以下几个元素：

* 一个系统调用接口（读、写、ioctl）用于数字化声音和混音功能。ioctl 命令集与传统的 *OSS* 或 *Voxware* 接口兼容，允许常见的多媒体应用程序无需修改即可移植。
* 用于处理声音数据的通用代码（格式转换、虚拟通道）。
* 一个统一的软件接口，用于硬件特定的音频接口模块。
* 对一些常见硬件接口（如 ac97）或共享硬件特定代码（例如：ISA DMA 例程）的额外支持。

特定声音卡的支持由硬件特定的驱动程序实现，这些驱动程序提供通道和混音器接口，以便接入通用的 **pcm** 代码。

在本章中，**pcm** 术语将指代声音驱动程序的核心通用部分，而不是硬件特定的模块。

预计的驱动程序编写者当然希望从现有模块开始，并使用代码作为最终参考。尽管声音代码整洁且清晰，但它也几乎没有注释。本文档尝试概述框架接口，并回答在适配现有代码时可能出现的一些问题。

作为替代方案，或者作为对现有示例的补充，您可以在 [https://people.FreeBSD.org/\~cg/template.c](https://people.FreeBSD.org/~cg/template.c) 上找到一个带注释的驱动程序模板。

## 15.2. 文件

所有相关代码都位于 **/usr/src/sys/dev/sound/**，除公共 ioctl 接口定义外，后者位于 **/usr/src/sys/sys/soundcard.h**。

在 **/usr/src/sys/dev/sound/** 下，**pcm/** 目录包含核心代码，而 **pci/**、**isa/** 和 **usb/** 目录则包含 PCI 和 ISA 板卡的驱动程序，以及 USB 音频设备的驱动程序。

## 15.3. 探测、附加等

声音驱动程序几乎与任何硬件驱动模块一样进行探测和附加。您可能想查看手册中的 [ISA](https://docs.freebsd.org/en/books/arch-handbook/isa-driver/#isa-driver) 或 [PCI](https://docs.freebsd.org/en/books/arch-handbook/pci/#pci) 特定部分以获取更多信息。

然而，声音驱动程序在一些方面有所不同：

* 它们声明自己为 **pcm** 类设备，并使用 `struct snddev_info` 设备私有结构：

  ```c
  static driver_t xxx_driver = {
                "pcm",
                xxx_methods,
                sizeof(struct snddev_info)
            };

            DRIVER_MODULE(snd_xxxpci, pci, xxx_driver, pcm_devclass, 0, 0);
            MODULE_DEPEND(snd_xxxpci, snd_pcm, PCM_MINVER, PCM_PREFVER,PCM_MAXVER);
  ```

  大多数声音驱动程序需要存储有关其设备的额外私有信息。私有数据结构通常在附加例程中分配。其地址通过对 `pcm_register()` 和 `mixer_init()` 的调用传递给 **pcm**。随后，**pcm** 在调用声音驱动程序接口时将该地址作为参数传递回去。
* 声音驱动程序的附加例程应通过调用 `mixer_init()` 向 **pcm** 声明其 MIXER 或 AC97 接口。对于 MIXER 接口，这会进一步调用 [`xxxmixer_init()`](https://docs.freebsd.org/en/books/arch-handbook/sound/#xxxmixer-init)。
* 声音驱动程序的附加例程通过调用 `pcm_register(dev, sc, nplay, nrec)` 向 **pcm** 声明其一般 CHANNEL 配置，其中 `sc` 是设备数据结构的地址，用于后续 **pcm** 的调用，`nplay` 和 `nrec` 分别是播放和录音通道的数量。
* 声音驱动程序的附加例程通过调用 `pcm_addchan()` 声明每个通道对象。这会在 **pcm** 中设置通道连接，并进一步调用 [`xxxchannel_init()`](https://docs.freebsd.org/en/books/arch-handbook/sound/#xxxchannel-init)。
* 声音驱动程序的拆卸例程应在释放资源之前调用 `pcm_unregister()`。

有两种处理非 PnP 设备的方法：

* 使用 `device_identify()` 方法（例如：**sound/isa/es1888.c**）。`device_identify()` 方法在已知地址探测硬件，如果发现受支持的设备，则创建一个新的 pcm 设备，并将其传递给探测/附加过程。
* 使用带有适当提示的自定义内核配置（例如：**sound/isa/mss.c**）。

**pcm** 驱动程序应实现 `device_suspend`、`device_resume` 和 `device_shutdown` 例程，以便电源管理和模块卸载能够正常工作。

## 15.4. 接口

**pcm** 核心与声音驱动之间的接口是通过 [内核对象](https://docs.freebsd.org/en/books/arch-handbook/kobj/#kernel-objects) 定义的。

声音驱动通常会提供两个主要接口：*CHANNEL* 和 *MIXER* 或 *AC97*。

*AC97* 接口是一个非常小的硬件访问（寄存器读写）接口，适用于具有 AC97 编解码器的硬件驱动。在这种情况下，实际的 MIXER 接口是由 **pcm** 中共享的 AC97 代码提供的。

### 15.4.1. CHANNEL 接口

#### 15.4.1.1. 函数参数的常见说明

声音驱动通常具有一个私有数据结构，用于描述其设备，并为其支持的每个播放和录音数据通道提供一个结构。

对于所有 CHANNEL 接口函数，第一个参数是一个不透明指针。

第二个参数是指向私有通道数据结构的指针，`channel_init()` 除外，后者具有指向私有设备结构的指针（并返回通道指针，供 **pcm** 进一步使用）。

#### 15.4.1.2. 数据传输操作概述

对于声音数据传输，**pcm** 核心与声音驱动通过一个共享内存区域进行通信，该区域由 `struct snd_dbuf` 描述。

`struct snd_dbuf` 是 **pcm** 私有的，声音驱动通过调用访问器函数（`sndbuf_getxxx()`）获取感兴趣的值。

共享内存区域的大小由 `sndbuf_getsize()` 确定，并且被划分为固定大小的块，每块为 `sndbuf_getblksz()` 字节。

播放时，一般的传输机制如下（录音时则反向操作）：

* **pcm** 首先填充缓冲区，然后调用声音驱动的 [`xxxchannel_trigger()`](https://docs.freebsd.org/en/books/arch-handbook/sound/#channel-trigger) 函数，参数为 PCMTRIG\_START。
* 然后，声音驱动将整个内存区域（`sndbuf_getbuf()`，`sndbuf_getsize()`）以 `sndbuf_getblksz()` 字节为单位重复传输到设备。对于每个传输的块，它会回调 `chn_intr()`pcm 函数（通常在中断时发生）。
* `chn_intr()` 会将新数据复制到已传输到设备的区域（现在为空闲），并对 `snd_dbuf` 结构进行适当的更新。

#### 15.4.1.3. channel\_init

`xxxchannel_init()` 用于初始化每个播放或录音通道。该调用由声音驱动的 attach 例程发起。（参见 crossref\:sound）。

```c
static void *
          xxxchannel_init(kobj_t obj, void *data,
             struct snd_dbuf *b, struct pcm_channel *c, int dir) ①
          {
              struct xxx_info *sc = data;
              struct xxx_chinfo *ch;
               ...
              return ch; ②
           }
```

- ①  `b` 是指向通道 `struct snd_dbuf` 的地址。函数中应通过调用 `sndbuf_alloc()` 初始化该缓冲区。要使用的缓冲区大小通常是设备“典型”单位传输大小的小倍数。`c` 是 **pcm** 通道控制结构的指针，这是一个不透明对象。函数应将其存储在本地通道结构中，以供后续 **pcm** 调用使用（例如：`chn_intr(c)`）。`dir` 表示通道的方向（`PCMDIR_PLAY` 或 `PCMDIR_REC`）。 
- ② 该函数应返回指向用于控制此通道的私有区域的指针。该指针将作为参数传递给其他通道接口调用。  

#### 15.4.1.4. channel\_setformat

`xxxchannel_setformat()` 应该为指定的通道和指定的声音格式设置硬件。

```c
static int
          xxxchannel_setformat(kobj_t obj, void *data, u_int32_t format) ①
          {
              struct xxx_chinfo *ch = data;
               ...
              return 0;
           }
```

- ①  `format` 作为 `AFMT_XXX` 值（**soundcard.h**）指定。 

#### 15.4.1.5. channel\_setspeed

`xxxchannel_setspeed()` 为指定的采样速度设置通道硬件，并返回可能调整后的速度。

```c
static int
          xxxchannel_setspeed(kobj_t obj, void *data, u_int32_t speed)
          {
              struct xxx_chinfo *ch = data;
               ...
              return speed;
           }
```

#### 15.4.1.6. channel\_setblocksize

`xxxchannel_setblocksize()` 设置块大小，即 **pcm** 和声音驱动之间、以及声音驱动和设备之间的单位事务的大小。通常，这是传输发生中断前传输的字节数。在传输过程中，每当传输该大小的数据时，声音驱动应调用 **pcm** 的 `chn_intr()`。

大多数声音驱动仅在此处注意到块大小，以便在实际传输开始时使用。

```c
static int
          xxxchannel_setblocksize(kobj_t obj, void *data, u_int32_t blocksize)
          {
              struct xxx_chinfo *ch = data;
                ...
              return blocksize; ①
           }
```

- ① 函数返回可能调整后的块大小。如果块大小确实更改，则应调用 `sndbuf_resize()` 来调整缓冲区。
  
#### 15.4.1.7. channel\_trigger

`xxxchannel_trigger()` 由 **pcm** 调用以控制驱动中的数据传输操作。

```c
static int
          xxxchannel_trigger(kobj_t obj, void *data, int go) ①
          {
              struct xxx_chinfo *ch = data;
               ...
              return 0;
           }
```

- ① `go` 定义当前调用的操作。可能的值为：

>**注意**
>
>如果驱动程序使用 ISA DMA，应该在对设备执行操作之前调用 `sndbuf_isadma()`，它会处理 DMA 芯片的相关操作。

#### 15.4.1.8. channel\_getptr

`xxxchannel_getptr()` 返回传输缓冲区中的当前偏移量。通常这将由 `chn_intr()` 调用，这样 **pcm** 就能知道可以在哪里传输新数据。

#### 15.4.1.9. channel\_free

`xxxchannel_free()` 用于释放通道资源，例如在卸载驱动程序时。如果通道数据结构是动态分配的，或者 `sndbuf_alloc()` 没有用于缓冲区分配，则应该实现此函数。

#### 15.4.1.10. channel\_getcaps

```c
struct pcmchan_caps *
          xxxchannel_getcaps(kobj_t obj, void *data)
          {
              return &xxx_caps; ①
           }
```

- ① 该例程返回指向（通常是静态定义的）`pcmchan_caps` 结构的指针（定义在 **sound/pcm/channel.h**）。该结构包含最小和最大采样频率，以及接受的声音格式。可以查看任何声音驱动程序的示例。 

#### 15.4.1.11. 更多函数

`channel_reset()`、`channel_resetdone()` 和 `channel_notify()` 是特殊用途的函数，应该在不讨论的情况下不要在驱动程序中实现，建议通过 [FreeBSD 多媒体邮件列表](https://lists.freebsd.org/subscription/freebsd-multimedia)讨论。

`channel_setdir()` 已弃用。

### 15.4.2. MIXER 接口

#### 15.4.2.1. mixer\_init

`xxxmixer_init()` 用于初始化硬件，并告诉 **pcm** 可用的播放和录音混音设备。

```c
static int
          xxxmixer_init(struct snd_mixer *m)
          {
              struct xxx_info   *sc = mix_getdevinfo(m);
              u_int32_t v;

              [初始化硬件]

              [设置适当的位，表示播放混音设备] ①
              mix_setdevs(m, v);
              [设置适当的位，表示录音混音设备]
              mix_setrecdevs(m, v)

              return 0;
          }
```

- ① 在整数值中设置位，并调用 `mix_setdevs()` 和 `mix_setrecdevs()` 来告诉 **pcm** 存在哪些设备。

混音器位定义可以在 **soundcard.h** 中找到（`SOUND_MASK_XXX` 值和 `SOUND_MIXER_XXX` 位移）。

#### 15.4.2.2. mixer\_set

`xxxmixer_set()` 设置一个混音器设备的音量级别。

```c
static int
          xxxmixer_set(struct snd_mixer *m, unsigned dev,
                           unsigned left, unsigned right) ①
          {
              struct sc_info *sc = mix_getdevinfo(m);
              [设置音量级别]
              return left | (right << 8); 燃堂
          }
```

- ① 设备由 `SOUND_MIXER_XXX` 值指定。音量值的范围是 \[0-100]。值为零时应该静音设备。 
- ② 由于硬件级别可能与输入比例不匹配，并且可能会发生一些四舍五入，函数返回实际的音量级别（范围为 0-100）。 

#### 15.4.2.3. mixer\_setrecsrc

`xxxmixer_setrecsrc()` 设置录音源设备。

```c
static int
          xxxmixer_setrecsrc(struct snd_mixer *m, u_int32_t src) ①
          {
              struct xxx_info *sc = mix_getdevinfo(m);

              [查找 src 中的非零位，设置硬件]

              [更新 src 以反映实际操作]
              return src; ②
           }
```

- ① 所需的录音设备作为一个位字段指定。                        
- ② 返回实际设置为录音的设备。有些驱动程序只能为录音设置一个设备。如果发生错误，函数应返回 `-1`。 

#### 15.4.2.4. mixer\_uninit, mixer\_reinit

`xxxmixer_uninit()` 应确保所有声音被静音，并且如果可能，混音器硬件应该关闭电源。

`xxxmixer_reinit()` 应确保混音器硬件重新启动，并恢复任何未通过 `mixer_set()` 或 `mixer_setrecsrc()` 控制的设置。

### 15.4.3. The AC97 Interface

*AC97* 接口由带有 AC97 编解码器的驱动程序实现。它只有三个方法：

* `xxxac97_init()` 返回找到的 AC97 编解码器的数量。
* `ac97_read()` 和 `ac97_write()` 分别读取或写入指定的寄存器。

*AC97* 接口由 **pcm** 中的 AC97 代码用于执行更高级别的操作。可以参考 **sound/pci/maestro3.c** 或 **sound/pci/** 下的许多其他示例。
