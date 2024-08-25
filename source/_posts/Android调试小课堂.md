# Android 调试小课堂

***原文由 Nicholas Lim (niclimcy) 和 Nolen Johnson (npjohnson) 写于2023年10月10日，发表于[LineageOS博客]([Android Debugging Crash Course – LineageOS – LineageOS Android Distribution](https://lineageos.org/engineering/HowTo-Debugging/))***

*人工翻译，水平较差xD*

![](https://lineageos.org/images/engineering/hero_debugging.jpg)

## 术语表

- ADB： Android 调试桥。
- 缓冲区：内存中固定大小的存储区域。
- CLI： 命令行界面。
- Commits：对于代码库的原子更改，用于版本控制。
- 调试：找到并修复错误、bug和非预期行为的过程。
- 设备块文件：存在于/dev目录下的特殊文件，可与内核驱动进行标准化交互。
- DTS： 设备树源码。
- EDL：高通紧急下载模式。
- gdb：GNU 调试器。
- HAL：硬件抽象层。
- 内核空间：内核运行设备驱动并与之交互的地方。
- 记录日志：记录并存储运行软件时发生的事件，比如错误信息、警告和调试信息。
- 内存地址：内存中存储数据和指令的位置的独特的标识符。
- OEM：原始设备制造商（例如谷歌、Fairphone、三星等）。
- PID：进程ID。
- pstore：持久性存储。
- 变基：将 commit 从一个分支移动到另一个分支的过程。
- 堆栈跟踪：导致程序出现错误或非预期行为的时候的函数调用序列。
- TID：线程ID。
- UART：通用异步收发器。
- 用户空间：正常进程（比如应用程序）运行的地方。
## 什么是调试？

了解 Android 调试，更重要的是了解 Android 系统中的各个部分。总体看，Android 系统主要由三个部分组成：应用程序、平台和内核。

![](https://lineageos.org/images/engineering/content_android_stack.png)

## 用户空间调试

### ADB

Android 调试桥（ADB）允许我们访问设备的命令行界面（或者shell），让我们可以使用原生的调试工具比如 Logcat。欲知如何在你的设备上使用 ADB 和 fastboot，请访问我们的[wiki](https://wiki.lineageos.org/adb_fastboot_guide)

### logcat

`logcat` 是一个输出多种系统日志的命令行工具，可以输出包括你使用 Log 类从应用程序写入的消息。

`logcat` 进程存储了各种循环缓冲区，可以使用` -b` 选项访问它们，有以下选项：

- `radio`：查看包含了无线电/电话相关信息的缓冲区。

- `events`：查看已解释的二进制系统事件缓冲区信息。

- `main`：查看主日志缓冲区（默认），不包含 system 和 crash 日志信息。

- `system`：查看系统日志缓冲区（默认）。

- `crash`：查看崩溃日志缓冲区（默认）。

- `all`：查看所有缓冲区

- `default`：汇报main、system 和 crash 缓冲区。

你可以在 [Android Developers](https://developer.android.com/tools/logcat) 上找到更多关于如何使用 `logcat` 的信息。

这是用 `logcat` 获取 crash （崩溃）缓冲区的实例：

```shell
$ adb logcat -b crash
+--------------------+-----+-----+-------+-----------------------------------------------------------------------+
| Date  Time         | PID | TID | Level | ProcessName   : Message                                               |
+--------------------+-----+-----+-------+-----------------------------------------------------------------------+
| 04-14 11:22:34.256 | 5199| 5199| E     | AndroidRuntime: FATAL EXCEPTION: main                                 |
| 04-14 11:22:34.256 | 5199| 5199| E     | AndroidRuntime: Process: com.android.settings, PID: 5199              |
| 04-14 11:22:34.256 | 5199| 5199| E     | AndroidRuntime: java.lang.RuntimeException: Unable to resume activity |
|                    |     |     |       |            {com.android.settings/com.android.settings.SubSettings}:   |
|                    |     |     |       |            java.lang.ArrayIndexOutOfBoundsException length=7; index=7 |
+--------------------+-----+-----+-------+-----------------------------------------------------------------------+
```

崩溃缓冲区对于调试应用崩溃（比如*设置*应用停止运行）和识别运行时错误很有用。

### Tombstones

有时候 ADB 服务可能没有在运行（可能原因包括在 adb 启动之前一个系统进程出现问题导致重启）。在这种情况下，我们不能够访问 logcat 命令。不要担心，一个墓碑文件已被写入 `/data/tombstones` ，其中包括引起崩溃的堆栈跟踪。

Tombstones 也更详细，如果 logcat 输出不足，则它会提供更长的堆栈跟踪。因此，也可以使用以下命令从正在运行的进程中导出 tombstone ：

```shell
$ adb shell debuggerd {PID}
```

提示：将 {PID} 替换为实际的进程 ID。

### Stack

`stack` 是一个 Python 脚本，以人类可读的格式表示崩溃转储（符号化本机崩溃转储）。你可以在任何 LineageOS 存储库的本地同步的 ~/android/lineage/development/scripts/stack 的路径中找到 `stack`。可以使用 `stack < /path/to/tombstone_0` 在提取的 tombstone 上运行 stack。

本机崩溃转储通常如下所示：

```shell
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
Build fingerprint: 'Android/aosp_angler/angler:7.1.1/NYC/enh12211018:eng/test-keys'
Revision: '0'
ABI: 'arm'
pid: 17946, tid: 17949, name: crasher  >>> crasher <<<
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0xc
    r0 0000000c  r1 00000000  r2 00000000  r3 00000000
    r4 00000000  r5 0000000c  r6 eccdd920  r7 00000078
    r8 0000461a  r9 ffc78c19  sl ab209441  fp fffff924
    ip ed01b834  sp eccdd800  lr ecfa9a1f  pc ecfd693e  cpsr 600e0030

backtrace:
    #00 pc 0004793e  /system/lib/libc.so (pthread_mutex_lock+1)
    #01 pc 0001aa1b  /system/lib/libc.so (readdir+10)
    #02 pc 00001b91  /system/xbin/crasher (readdir_null+20)
    #03 pc 0000184b  /system/xbin/crasher (do_action+978)
    #04 pc 00001459  /system/xbin/crasher (thread_callback+24)
    #05 pc 00047317  /system/lib/libc.so (_ZL15__pthread_startPv+22)
    #06 pc 0001a7e5  /system/lib/libc.so (__start_thread+34)
Tombstone written to: /data/tombstones/tombstone_06
```

运行 `stack < /data/tombstones/tombstone_06` ，输出如下所示：

```shell
Revision: '0'
pid: 17946, tid: 17949, name: crasher  >>> crasher <<<
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0xc
     r0 0000000c  r1 00000000  r2 00000000  r3 00000000
     r4 00000000  r5 0000000c  r6 eccdd920  r7 00000078
     r8 0000461a  r9 ffc78c19  sl ab209441  fp fffff924
     ip ed01b834  sp eccdd800  lr ecfa9a1f  pc ecfd693e  cpsr 600e0030
Using arm toolchain from: ~/android/lineage/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin/

Stack Trace:
  RELADDR   FUNCTION                   FILE:LINE
  0004793e  pthread_mutex_lock+2       bionic/libc/bionic/pthread_mutex.cpp:515
  v------>  ScopedPthreadMutexLocker   bionic/libc/private/ScopedPthreadMutexLocker.h:27
  0001aa1b  readdir+10                 bionic/libc/bionic/dirent.cpp:120
  00001b91  readdir_null+20            system/core/debuggerd/crasher.cpp:131
  0000184b  do_action+978              system/core/debuggerd/crasher.cpp:228
  00001459  thread_callback+24         system/core/debuggerd/crasher.cpp:90
  00047317  __pthread_start(void*)+22  bionic/libc/bionic/pthread_create.cpp:202 (discriminator 1)
  0001a7e5  __start_thread+34          bionic/libc/bionic/clone.cpp:46 (discriminator 1)
```

`stack` 的工作原理与内核空间调试工具`decode_stacktrace.sh` 非常相似。它们都提供了堆栈跟踪所引用的原始代码的确切文件和行。继续阅读以了解有关如何使用`decode_stacktrace.sh` 的更多信息。

### ramoops-pmsg

ramoops-pmsg 是 ramoops 的用户空间可访问版本。要从 pstore 访问上次重启之前的这些日志，可以运行：

```sh
$ adb logcat -b all -L
```

关于ramoops内核特性更详细的解释可以在下面找到。

## 内核空间调试

内核空间调试帮助我们识别在内核内部的问题。设备制造商除了提供设备的内核驱动以外，还可能在发布内核源码时客制内核的其他部分。因此，当设备制造商将他们的驱动变基到新版本内核上的时候（为了跟上安全补丁），可能会出现回归。

### dmesg

`dmesg` 是一个显示内核缓冲区消息的命令行工具。它提供了内核级活动的详细视图，使设备维护者能够诊断系统崩溃、驱动问题并监视系统事件。请注意，所有 LineageOS 构建都默认 SELinux Enforcing，这要求你在使用 dmesg 之前 `adb root` 模式。

下面是 dmesg 的关于空指针引用的截断输出

```shell
# adb shell dmesg
| Unable to handle kernel NULL pointer dereference at virtual address 0000000000000010
| Internal error: Oops: 96000006 [#1] SMP
| Call trace:
| update_insn_emulation_mode+0xc0/0x148
| emulation_proc_handler+0x64/0xb8
| proc_sys_call_handler+0x9c/0xf8
| proc_sys_write+0x18/0x20
| __vfs_write+0x20/0x48
| vfs_write+0xe4/0x1d0
| ksys_write+0x70/0xf8
| __arm64_sys_write+0x20/0x28
| el0_svc_common.constprop.0+0x7c/0x1c0
| el0_svc_handler+0x2c/0xa0
| el0_svc+0x8/0x200
```

### ramoops

ramoops 是 [Linux 内核的一项功能](https://www.kernel.org/doc/html/next/admin-guide/ramoops.html)，可在系统崩溃前写入内存。ramoops 可在设备的内核设备树源码 (DTS) 中配置，方法是为 ramoops-pmsg 保留一个内存缓冲区。它与内核 pstore 驱动程序配合使用，在重新启动前将 ramoops 保存到 `/sys/fs/pstore` 的持久文件中。

分配了 pmsg 缓冲区的 ramoops 配置示例：

```c
/{
    reserved-memory {
        ramoops: ramoops@b0000000 {
            compatible = "ramoops";
            reg = <0 0xb0000000 0 0x00400000>;
            record-size = <0x40000>; /*256x1024*/
            console-size = <0x40000>;
            ftrace-size = <0x40000>;
            pmsg-size = <0x200000>;
            ecc-size = <0x0>;
        };
    };
};
```

在较新的内核上， ramoops 可以通过如下配置选项启用：

```
CONFIG_PSTORE=y
CONFIG_PSTORE_CONSOLE=y
CONFIG_PSTORE_RAM=y
```

pstore 通常默认压缩，这使得它难以在调试中使用。你可能想通过如下设置来禁用压缩：

````
# CONFIG_PSTORE_COMPRESS is not set
````

尽管 ramoops 和 pstore 都是强大的工具，但使用它们时仍有一些注意事项。由于 pstore 默认将数据写入缓冲区，并且我们通常只在系统即将崩溃时使用它，因此事后检索 pstore 时，我们往往会看到大量损坏。

### addr2line

dmesg 和 ramoops 经常在堆栈跟踪中产生加密的内存地址，例如`ffffff9405cebf10`来自：

```
CFI failure (target: [\<\ffffff9405cebf10\>] __typeid__ZTSFvP10net_deviceE_global_addr+0x170/0x17c):
```

在这种情况下，我们可以使用 Address To Line (addr2line) 来找到出现问题的文件和行数，使用：

```
$ addr2line -e /path/to/kernel-module.o ffffff9405cebf10
```

### decode_stacktrace.sh

`decode_stacktrace.sh` 是每个 Linux 内核附带脚本，使用`addr2line`，源码位于 [linux/blob/master/scripts/decode_stacktrace.sh](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/scripts/decode_stacktrace.sh) 。要使用它，首先你需要在内核配置文件中启用 `CONFIG_DEBUG_INFO=y` 并构建内核。

接下来，你需要从 dmesg 中提取要调试的内核恐慌的调用跟踪，并将其保存在文本文件中，例如此处的 dmesg.txt：

```
| update_insn_emulation_mode+0xc0/0x148
| emulation_proc_handler+0x64/0xb8
| proc_sys_call_handler+0x9c/0xf8
| proc_sys_write+0x18/0x20
| __vfs_write+0x20/0x48
| vfs_write+0xe4/0x1d0
| ksys_write+0x70/0xf8
| __arm64_sys_write+0x20/0x28
| el0_svc_common.constprop.0+0x7c/0x1c0
| el0_svc_handler+0x2c/0xa0
| el0_svc+0x8/0x200
```

最后，将 dmesg.txt 和构建好的内核提供给 `decode_stacktrace.sh` ：

```sh
$ ./scripts/decode_stacktrace.sh /path/to/vmlinux /path/to/kernel-source-dir < dmesg.txt
```

正如你在以下示例（不同的堆栈跟踪）中看到的，堆栈跟踪中每个调用的内存地址已被替换为特定文件和代码行，然后您可以在内核源码中找到它。

```
| dump_stack (lib/dump_stack.c:52)
| warn_slowpath_common (kernel/panic.c:418)
| warn_slowpath_null (kernel/panic.c:453)
| _oalloc_pages_slowpath+0x6a/0x7d0
| ? zone_watermark_ok (mm/page_alloc.c:1728)
| ? get_page_from_freelist (mm/page_alloc.c:1939)
| __alloc_pages_nodemask (mm/page_alloc.c:2766)
```

### 串口 / gdb

带有 UART 端口的设备（参见带有 3.5 毫米耳机端口的旧款 Nexus/Google Pixel 和带有 USB-C 调试器的新款 Google Pixel）可以使用 UART 线连接来查看内核控制台消息 (kgdb)。使用串口，你甚至可以调试内核启动前发生的问题。

当你的设备卡在 logo 上时，可以考虑使用串口。

### 绝望的调试

如果其他方法都失败了，你可以在要调试的内核部分使用 `panic()`。[SebaUbuntu 的补丁](https://github.com/xiaomi-sm8150-devs/android_kernel_xiaomi_sm8150-legacy/commit/9d8822a6967ee623790270539a929942b71f191b)在此处演示了如何使用 `panic()` 来捕获早期初始化问题。

## 芯片制造商 / OEM 特定的调试方法

这是我们这几年发现的非常有用的调试工具，它们由OEM开发。

### EDL memorydump （高通）

![](https://lineageos.org/images/engineering/content_qualcomm_crashdump.png)

一些高通设备启用了 CrashDump，允许你使用高通的 firehouse 工具来获取内存转储。由于 firehouse 是闭源的，我们建议使用由 Bjoern Kerler 重写的开源版，可以在 [bkerler/edl](https://github.com/bkerler/edl) 找到。你可以使用 `edl memorydump` 获取内存转储。

### /dev/block/by-name/debug （三星）

`/dev/block/by-name/debug` 是三星设备上一个特殊的设备块文件，它包含了XBL日志、内核日志以及更多东西。你可以执行 `adb pull /dev/block/by-name/debug debug.bin` 来转储日志流。

下面是一个截断的 debug.bin 文件的示例：

```
{340532} ** XBL(1) **
{340532}
Format: Log Type - Time(microsec) - Message - Optional Info
Log Type: B - Since Boot(Power On Reset),  D - Delta,  S - Statistic
S - QC_IMAGE_VERSION_STRING=BOOT.XF.2.1-00133-SDM710LZB-3
S - IMAGE_VARIANT_STRING=SDM670LA
S - OEM_IMAGE_VERSION_STRING=21DJFC21
S - Boot Interface: eMMC
S - Secure Boot: On
S - Boot Config @ 0x00786070 = 0x000000c9
S - JTAG ID @ 0x00786130 = 0x100910e1
S - OEM ID @ 0x00786138 = 0x00200000
S - Feature Config Row 0 @ 0x007841a0 = 0x08d020000b588420
S - Feature Config Row 1 @ 0x007841a8 = 0xe0140000000311a0
S - Core 0 Frequency, 1516 MHz
S - PBL Patch Ver: 0
S - PBL freq: 600 MHZ
S - I-cache: On
S - D-cache: On
```

我们之前讲解[高通的信任链](https://lineageos.org/engineering/Qualcomm-Firmware)的博客有关于 eXtensible Bootloader (XBL) 的更多细节。

## 常见错误 (以及如何解决)

现在我们对 Android 开发过程中使用的某些调试工具有了基本的了解，现在让我们学习如何识别和修复调试过程中遇到的常见错误。

### dlopen failed

许多设备都具有预构建库，这些库是使用旧版本的库编译的，这些旧版本的库缺失某些符号。可能发生如下所示的错误：

```
* java.lang.UnsatisfiedLinkError: dlopen failed: cannot locate symbol "_ZN7android21SurfaceComposerClient11Transaction5applyEb" referenced by "/product/lib64/libsecureuisvc_jni.so"...
```

为了解决此错误，我们需要插入我们俗称“垫片”的库。通过拦截对缺失函数的调用并提供替代的实现，我们可以从本质上模拟正在使用的预构建库构建的原始库的行为。

另外，一些现代设备会选择将更新库的旧 VNDK 版本复制到  `$libname-v$vndkVersion.so` ，然后对有问题的库打补丁，以加载该版本的库。

你可以[在此处](https://github.com/LineageOS/android_hardware_lineage_compat)查看我们通用的预先存在的垫片，这些垫片在 LineageOS 20 之前由各个设备维护者管理。对于上述示例，你可以参考[此补丁](https://review.lineageos.org/c/LineageOS/android_device_google_bonito/+/343466)，了解如何将垫片包用于需要它们的预构建库。

### Hidden dlopen failed

由于 dlopen 错误仅在运行时发生，因此某些故障不会立即显示，甚至不会记录在日志中。因此，我们想出了一个库钩子 ，` dlopen.so` ，你可以将其放置在 LD_PRELOAD 中，显示所有链接器操作，帮助我们查看哪些库当前缺少符号甚至缺少依赖项。

这是某个设备使用[此库](https://review.lineageos.org/c/LineageOS/android_hardware_lineage_compat/+/346648)的日志：

```
instantnoodlep / # LD_PRELOAD=dlopen.so /vendor/bin/hw/android.hardware.gnss\@2.1-service-qti
dlopen(libnetd_client.so) -> 0x0, errno: dlopen failed: library "libnetd_client.so" not found
dlopen(libgnss.so) -> 0xdc9f08e905187e63, errno: (null)
dlopen(liblbs_core.so) -> 0x618992fa4f6a1e6d, errno: (null)
dlopen(liblocdiagiface.so) -> 0x0, errno: dlopen failed: library "liblocdiagiface.so" not found
dlopen(libloc_net_iface.so) -> 0x0, errno: dlopen failed: library "libloc_net_iface.so" not found
dlopen(vendor.qti.gnss@4.0-service.so) -> 0xe8b09305c7a1c55f, errno: (null)
dlopen(libdataitems.so) -> 0xc4ba0f7c15946aef, errno: (null)
dlopen(android.hardware.gnss@2.1-impl-qti.so) -> 0xd950565f49bbcc01, errno: (null)
dlopen(libgnss.so) -> 0xdc9f08e905187e63, errno: (null)
dlopen(libxtadapter.so) -> 0x39eb3dbc835592b5, errno: (null)
dlopen(libcdfw.so) -> 0x59b12f3c0b5e3e1b, errno: (null)
dlopen(libloc_socket.so) -> 0x2229c8abec54dac9, errno: (null)
```

## One more thing

当调试非私有系统应用的时候，比如 Aperture ，你可以使用 [Android Studio](https://developer.android.com/studio) 来更轻松简单地进行调试！
