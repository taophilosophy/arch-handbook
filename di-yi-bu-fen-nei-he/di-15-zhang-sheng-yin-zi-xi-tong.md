# 第 15 章 声音子系统


## 15.1. 介绍

FreeBSD 声音子系统清晰地将通用声音处理问题与特定设备问题分开。这使得为新硬件添加支持变得更容易。

pcm(4)框架是声音子系统的核心部分。它主要实现以下元素：

* 用于数字化声音和混音功能的系统调用接口（读取、写入、ioctls）。ioctl 命令集兼容传统的 OSS 或 Voxware 接口，允许常见多媒体应用程序在无需修改的情况下移植。
* 处理声音数据的通用代码（格式转换、虚拟通道）。
* 用于硬件特定音频接口模块的统一软件接口。
* 为一些常见硬件接口（ac97）提供额外支持，或者共享硬件特定代码（例如：ISA DMA 例程）。

特定声卡的支持是通过硬件特定驱动程序实现的，这些驱动程序提供通道和混音接口，以便插入通用 pcm 代码。

在本章中，pcm 一词将指代声卡驱动程序的中心通用部分，而不是硬件特定模块。

潜在的驱动程序编写人员当然希望从现有模块开始，并将代码用作最终参考。但是，尽管良好的代码很好而且干净，但它也大多没有注释。本文试图概述框架接口，并回答在调整现有代码时可能出现的一些问题。

作为另一种选择，或者除了从现有工作示例开始外，您也可以在 https://people.FreeBSD.org/~cg/template.c 找到一个有注释的驱动程序模板

## 15.2. 文件

所有相关的代码都位于 /usr/src/sys/dev/sound/ 中，除了公共的 ioctl 接口定义，可以在 /usr/src/sys/sys/soundcard.h 中找到。

在 /usr/src/sys/dev/sound/ 中，pcm/ 目录包含了核心代码，而 pci/、isa/ 和 usb/ 目录包含了用于 PCI 和 ISA 板卡以及 USB 音频设备的驱动程序。

## 15.3. 探测、附加等。

音频驱动程序的探测和附加方式几乎与任何硬件驱动程序模块相同。您可能希望查看手册中的 ISA 或 PCI 特定部分以获取更多信息。

然而，音频驱动程序在某些方面有所不同：

* 它们将自己声明为 pcm 类设备，带有一个 struct snddev_info 设备私有结构：

  ```
            static driver_t xxx_driver = {
                "pcm",
                xxx_methods,
                sizeof(struct snddev_info)
            };

            DRIVER_MODULE(snd_xxxpci, pci, xxx_driver, pcm_devclass, 0, 0);
            MODULE_DEPEND(snd_xxxpci, snd_pcm, PCM_MINVER, PCM_PREFVER,PCM_MAXVER);
  ```

  大多数声卡驱动程序需要存储有关其设备的其他私有信息。通常在附加例程中分配私有数据结构。其地址通过调用 pcm_register() 和 mixer_init() 传递给 pcm。稍后，pcm 会在调用声卡接口时将该地址作为参数传回。
* 声卡驱动程序附加例程应通过调用 mixer_init() 声明其 MIXER 或 AC97 接口到 pcm。对于 MIXER 接口，这将依次调用 xxxmixer_init() 。
* 声卡驱动程序附加例程通过调用 pcm_register(dev, sc, nplay, nrec) 声明其通道配置到 pcm，其中 sc 是设备数据结构的地址，在 pcm 的进一步调用中使用，而 nplay 和 nrec 是播放和记录通道的数量。
* 音频驱动程序附加例程通过调用 pcm_addchan() 声明其每个通道对象。这将在 pcm 中设置通道粘合剂，并依次调用 xxxchannel_init() 。
* 在释放资源之前，音频驱动程序分离例程应调用 pcm_unregister() 。

处理非 PnP 设备有两种可能的方法：

* 使用 device_identify() 方法（示例：sound/isa/es1888.c）。 device_identify() 方法会在已知地址处探测硬件，如果找到支持的设备，则创建一个新的 pcm 设备，然后传递给探测/附加。
* 使用定制内核配置，并为 pcm 设备提供适当的提示（示例：sound/isa/mss.c）。

pcm 驱动程序应实现 device_suspend ， device_resume 和 device_shutdown 例程，以便正确执行电源管理和模块卸载功能。

## 15.4. 接口

PCM 核心和声音驱动程序之间的接口是以内核对象的形式定义的。

通常，声音驱动程序提供两种主要接口：通道和混音器或 AC97。

AC97 接口是由具有 AC97 编解码器的硬件驱动程序实现的非常小的硬件访问（寄存器读/写）接口。在这种情况下，实际的 MIXER 接口由 pcm 中的共享 AC97 代码提供。

### 15.4.1. 通道接口

#### 15.4.1.1. 功能参数的通用注意事项

声音驱动程序通常具有一个私有数据结构来描述其设备，并为其支持的每个播放和记录数据通道提供一个结构。

对于所有 CHANNEL 接口函数，第一个参数是一个不透明指针。

第二个参数是指向私有通道数据结构的指针，除了 channel_init() ，它具有指向私有设备结构的指针（并返回通道指针以供 pcm 进一步使用）。

#### 15.4.1.2. 数据传输操作概述

为了实现音频数据传输，pcm 核心和音频驱动程序通过一个共享内存区域进行通信，该区域由 struct snd_dbuf 描述。

struct snd_dbuf 对 pcm 是私有的，音频驱动程序通过调用访问器函数（ sndbuf_getxxx() ）获取感兴趣的值。

共享内存区的大小为 sndbuf_getsize() ，分为大小为 sndbuf_getblksz() 字节的固定大小块。

在播放时，一般的传输机制如下（录制的想法相反）：

* pcm 最初填充缓冲区，然后调用声卡驱动程序的 xxxchannel_trigger() 函数，参数为 PCMTRIG_START。
* 声音驱动程序然后安排重复地将整个内存区域 ( sndbuf_getbuf() , sndbuf_getsize() ) 传输到设备，以 sndbuf_getblksz() 字节的块的形式。它会为每个传输的块回调 chn_intr() pcm 函数（这通常会发生在中断时期）。
* chn_intr() 安排将新数据复制到已传输到设备的区域（现在空闲），并对 snd_dbuf 结构进行适当更新。

#### 15.4.1.3. channel_init

xxxchannel_init() 被调用来初始化每个播放或录制通道。这些调用是从声音驱动程序附加例程启动的。（请参阅探测和附加部分）。

```
          static void *
          xxxchannel_init(kobj_t obj, void *data,
             struct snd_dbuf *b, struct pcm_channel *c, int dir) 
          {
              struct xxx_info *sc = data;
              struct xxx_chinfo *ch;
               ...
              return ch; 
           }
```

|  | b 是通道 struct snd_dbuf 的地址。应该在函数中通过调用 sndbuf_alloc() 来初始化它。要使用的缓冲区大小通常是您的设备的“典型”单元传输大小的一个小倍数。 c 是 pcm 通道控制结构指针。这是一个不透明对象。函数应该将它存储在本地通道结构中，以便在以后的 pcm 调用中使用（即： chn_intr(c) ）。 dir 表示通道方向（ PCMDIR_PLAY 或 PCMDIR_REC ）。 |
| -- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|  | 该函数应返回一个指向用于控制此通道的私有区域的指针。这将作为参数传递给其他通道接口调用。                                                                                                                                                                                                                                                    |

#### 15.4.1.4. channel_setformat

xxxchannel_setformat() 应该为指定的声道设置指定的声音格式。

```
          static int
          xxxchannel_setformat(kobj_t obj, void *data, u_int32_t format) 
          {
              struct xxx_chinfo *ch = data;
               ...
              return 0;
           }
```

|  | format 被指定为 AFMT_XXX value （soundcard.h）。 |
| -- | -------------------------------------------------- |

#### 15.4.1.5. channel_setspeed

xxxchannel_setspeed() 设置指定采样速度的通道硬件，并返回可能调整后的速度。

```
          static int
          xxxchannel_setspeed(kobj_t obj, void *data, u_int32_t speed)
          {
              struct xxx_chinfo *ch = data;
               ...
              return speed;
           }
```

#### 15.4.1.6. channel_setblocksize

xxxchannel_setblocksize() 设置块大小，这是 pcm 和声卡之间、声卡和设备之间单位事务的大小。通常，这将是在发生中断之前传输的字节数。在传输过程中，声卡应该每次传输这个大小时调用 pcm 的 chn_intr() 。

大多数声卡驱动程序只在这里注意块大小，在实际开始传输时使用。

```
          static int
          xxxchannel_setblocksize(kobj_t obj, void *data, u_int32_t blocksize)
          {
              struct xxx_chinfo *ch = data;
                ...
              return blocksize; 
           }
```

|  | 该函数返回可能调整后的块大小。如果块大小确实更改了，应调用 sndbuf_resize() 来调整缓冲区。 |
| -- | ------------------------------------------------------------------------------------------- |

#### 15.4.1.7. channel_trigger

`xxxchannel_trigger()` is called by pcm to control data transfer operations in the driver.

```
          static int
          xxxchannel_trigger(kobj_t obj, void *data, int go) 
          {
              struct xxx_chinfo *ch = data;
               ...
              return 0;
           }
```

|  | `go` defines the action for the current call. The possible values are: |
| -- | -------------------------------------------------------------------- |

|  | 如果驱动程序使用 ISA DMA，应在执行设备操作之前调用 sndbuf_isadma() ，并将处理 DMA 芯片方面的事务。 |
| -- | ---------------------------------------------------------------------------------------------------- |

#### 15.4.1.8. channel_getptr

xxxchannel_getptr() 返回传输缓冲区中的当前偏移量。通常会被 chn_intr() 调用，这是 pcm 知道它可以传输新数据的位置。

#### 15.4.1.9. channel_free

当驱动程序被卸载时， xxxchannel_free() 被调用来释放通道资源，如果通道数据结构是动态分配的或者未使用 sndbuf_alloc() 进行缓冲区分配，则应该实现它。

#### 15.4.1.10. channel_getcaps

```
          struct pcmchan_caps *
          xxxchannel_getcaps(kobj_t obj, void *data)
          {
              return &xxx_caps; 
           }
```

|  | 该例程返回指向（通常是静态定义的） pcmchan_caps 结构的指针（在 sound/pcm/channel.h 中定义）。该结构保存最小和最大采样频率，以及可接受的声音格式。查看任何声音驱动程序以获取示例。 |
| -- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

#### 15.4.1.11. 更多函数

channel_reset() ， channel_resetdone() 和 channel_notify() 用于特殊目的，不应在未经讨论的情况下在 FreeBSD 多媒体邮件列表上实现驱动程序中。

  channel_setdir() 已过时。

### 15.4.2. MIXER 接口

#### 15.4.2.1. mixer_init

xxxmixer_init() 初始化硬件并告诉 pcm 可用于播放和录音的混音设备

```
          static int
          xxxmixer_init(struct snd_mixer *m)
          {
              struct xxx_info   *sc = mix_getdevinfo(m);
              u_int32_t v;

              [Initialize hardware]

              [Set appropriate bits in v for play mixers] 
              mix_setdevs(m, v);
              [Set appropriate bits in v for record mixers]
              mix_setrecdevs(m, v)

              return 0;
          }
```

|  | 在整数值中设置位并调用 mix_setdevs() 和 mix_setrecdevs() 告诉 pcm 存在哪些设备。 |
| -- | ---------------------------------------------------------------------------------- |

混音器位定义可以在 soundcard.h 中找到（ SOUND_MASK_XXX 值和 SOUND_MIXER_XXX 位移）。

#### 15.4.2.2. mixer_set

xxxmixer_set() 设置一个混音设备的音量级别。

```
          static int
          xxxmixer_set(struct snd_mixer *m, unsigned dev,
                           unsigned left, unsigned right) 
          {
              struct sc_info *sc = mix_getdevinfo(m);
              [set volume level]
              return left | (right << 8); 
          }
```

|  | 设备被指定为一个 SOUND_MIXER_XXX 值。音量值在范围 [0-100] 内指定。零值应该将设备静音。                         |
| -- | ---------------------------------------------------------------------------------------------------------------- |
|  | 由于硬件级别可能不会与输入比例匹配，而且会发生一些四舍五入，所以该例程返回实际级别值（范围为 0-100），如所示。 |

#### 15.4.2.3. mixer_setrecsrc

xxxmixer_setrecsrc() 设置录音源设备。

```
          static int
          xxxmixer_setrecsrc(struct snd_mixer *m, u_int32_t src) 
          {
              struct xxx_info *sc = mix_getdevinfo(m);

              [look for non zero bit(s) in src, set up hardware]

              [update src to reflect actual action]
              return src; 
           }
```

|  | 指定的录音设备被指定为位字段                                                             |
| -- | ------------------------------------------------------------------------------------------ |
|  | 返回设置为录音的实际设备。一些驱动程序只能设置一个录音设备。如果发生错误，函数应返回-1。 |

#### 15.4.2.4. mixer_uninit，mixer_reinit

xxxmixer_uninit() 应确保所有声音静音，如有可能，混音器硬件应关闭电源。

xxxmixer_reinit() 应确保混音器硬件已上电，并恢复任何非由 mixer_set() 或 mixer_setrecsrc() 控制的设置。

### 15.4.3. AC97 接口

AC97 接口由带有 AC97 编解码器的驱动程序实现。它只有三种方法：

* xxxac97_init() 返回找到的 ac97 编解码器数量。
* ac97_read() 和 ac97_write() 读取或写入指定的寄存器。

AC97 接口由 pcm 中的 AC97 代码用于执行更高级别的操作。 查看 sound/pci/maestro3.c 或其他许多 sound/pci/下的示例。
