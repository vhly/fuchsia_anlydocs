# Magenta Boot

## magenta.bin

关于这个文件，需要查看 magenta/make/build.mk 这个文件<br/>
文件中配置了 $(OUTLKBIN): $(OUTLKELF) 这个依赖，<br/>
其中 OUTLKBIN 与 OUTLKELF 在 magenta/make/engine.mk 中定义<br/>
实际上就是 magenta.bin 与 magenta.elf<br/>

magenta.elf 是将 内核编译形成的，bin 是经过了 objcopy 指令生成的<br/>

关于 x86_64 平台的 magenta.bin 是直接通过 汇编编译的时候，进行生成的，<br/>
因此直接就是bootdata 数据格式，并且只有一个代码段；<br/>

关于 ARM64 平台的 magenta.bin 是通过 ELF 在进行链接时，<br/>
指定 magenta/kernel/arch/arm64/system-onesegment.ld 编译脚本进行设置，生成的。

### magenta.bin 内核入口
内核入口包含在特定的格式中，最好参考 x86_64平台的 header.S 文件定义即可。<br/>
生成的方式是  asm - ld处理 - ELF - objcopy 生成纯代码没有任何格式的内核文件，入口直接调用

#### ARM64 平台 magenta.bin 入口代码

代码入口在 magenta/kernel/arch/arm64/start.S 中，代码会调用 lk_main 进入 C 代码中

#### x86_64 平台 入口代码

代码入口在 magenta/kernel/arch/x86/header.S 中进行了判断，入口是 \_entry64 函数<br/>
这个函数在 magenta/kernel/arch/x86/64/start.S 中

### LK 内核入口 lk_main

magenta基于LK内核进行的制作，同时也增加了一些不同点，支持PC, Mobile, 实时设备等<br/>
两个平台在调用的时候，都直接调用了 lk_main 方法，进行内核功能实现；<br/>

方法在 magenta/kernel/top/main.c 文件中，注释明确说明是从arch代表调用到指定位置

#### lk_main 执行流程

1. 前期准备线程；
1. 前期构造，使用了外部定义方法数组，进行调用初始化；
1. 前期初始化主要CPU；
1. 前期平台初始化；
1. 前期目标初始化；
1. 初始化内核Heap堆；
1. 内核初始化；kernel_init() 外部方法；
1. 线程 bootstrap2 创建、启动；
1. 当前线程 IDLE；

#### 内核初始化 kernel_init() 调用

在文件 magenta/kernel/kernel/init.c 文件中，初始化 magenta的多线程和时间队列

#### bootstrap2线程执行流程

1. ARCH 平台初始化
1. 平台初始化；调用到实现初始化操作
1. 目标初始化；
1. apps 初始化：调用 apps_init(); 重要因为会调用到内置在内核文件中的应用程序了

### apps_init 初始化内核中的应用程序

apps_init() 可以认为是内核与应用的交汇，在这个地方就可以启动内核中的程序代码了<br/>
注意不是单独的程序文件，而是直接在内核代码与数据段中的代码；<br/>

方法在 magenta/kernel/app/app.c 文件中；
在这个文件中，定义了 apps_init() 方法，这个方法，<br/>
根据外部静态变量 <br/>

````
extern const struct app_descriptor __start_apps[] __WEAK;
extern const struct app_descriptor __stop_apps[] __WEAK;
````

来进行重复的调用，而实际的应用声明，都是在 app.h 中，使用 APP_START 宏来声明的<br/>
真正的程序都是在 magenta/kernel/app/ 中的子目录中，其中就是一个特殊的应用程序:<br/>
shell 这个程序，shell.c 文件中声明了应用到app.c中；可以直接开启线程启动对应的程序.

apps_init() 负责遍历声明的应用，为每一个应用开启一个线程，并且调用应用描述结构中的入口；<br/>
应用结构描述可以参考 shell.c 中的代码，如下：<br/>

````
static void shell_init(const struct app_descriptor *app)
{
    console_init();
}

static void shell_entry(const struct app_descriptor *app, void *args)
{
    console_start();
}

APP_START(shell)
.init = shell_init,
 .entry = shell_entry,
  APP_END
````

这种方式代表每一个应用，都应该有一个 app_descriptor 描述的结构，进行初始化，并且启动

### 关于 magenta 初始化

每一个 lk_main() 中的初始化，都会调用，例如初始化硬件，设备，内存等等。

## 关于 Fuchsia 的启动说明

参考了 frun ，实际上内部调用的是 mrun -x user.bootfs
其中 user.bootfs 代表包含了 magenta 以及 fuchsia 内部程序的内核映像。

### 启动流程分析

1. LK 进行初始化，初始化到 kernel/lib/userboot/userboot.cpp
1. 初始化调用 LK_INIT_HOOK(userboot, userboot_init, LK_INIT_LEVEL_APPS - 1);
1. userboot_init 执行 attempt_userboot 方法
1. attempt_userboot 查找 ramdisk，创建内存对象，创建线程，映射 UserbootImage
1. 创建线程，执行 userboot 的代码，加载 dso，可以理解为驱动，实际代码在 system/core/userboot/start.c 中

## magenta.bin 与 bootdata.bin

magenta.bin 是一个目前不超过1MB的文件，纯代码指令，用于由bootloader启动<br/>
bootdata.bin 是通过 magenta/make/engine.mk 定义，并且由 build.mk 进行执行操作<br/>
依赖于
1. $(MKBOOTFS)   host 工具，用于生成 bootdata 文件
1. $(USER_MANIFEST)   host 工具，生成添加到 bootdata中的文件列表
1. $(USER_MANIFEST_DEPS)
1. $(ADDITIONAL_BOOTDATA_ITEMS)

其中 USER_MANIFEST_DEPS 会在各个mk文件中进行添加，最终形成完整的内容
