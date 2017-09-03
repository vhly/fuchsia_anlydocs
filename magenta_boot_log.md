# Magenta 启动日志分析

## 日志分析

### 关于 LK 中的动态初始化

lk_init_level 函数主要用于内核模块的初始化，<br/>
文件在 magenta/kernel/top/init.c 文件中，会调用 lk_init_struct 结构体定义的初始化方法<br/>
每一个 lk_init_struct 是在代码中使用 LK_INIT_HOOK 宏来定义的，因此真正的内核初始化代码<br/>
分布在多个目录的文件中，这些宏定义在内核链接的时候，通过链接脚本的操作，被自动配置到 init.c 中声明的静态变量中；<br/>

#### 初始化实例

以下是两个初始化部分的对比：<br/>
* global_prng_seed
  位置 magenta/kernel/lib/crypto/global_prng.cpp，<br/>
  定于：LK_INIT_HOOK(global_prng_seed, crypto::GlobalPRNG::EarlyBootSeed, LK_INIT_LEVEL_TARGET_EARLY);<br/>
  代表在内核target初始化之前，初始化 global_prng_seed
* version
  位置 magenta/kernel/lib/version/version.cpp<br/>
  定义："LK_INIT_HOOK(version, (void \*)&print_version, LK_INIT_LEVEL_HEAP - 1);"<br/>
  代表在内核heap初始化前最后一个初始化回调，进行调用；

根据代码来看：magenta/kernel/top/main.c 内部的 lk_main 在调用时，
顺序是
````
target_early_init();   // 先实际执行初始化操作

dprintf(INFO, "\nwelcome to lk/MP\n\n");

// 执行完操作，打印日志之后，通过 cpu_init_level 方法，执行所有 target_early 的HOOK
// 以及所有 LK_INIT_LEVEL_HEAP - 1 的所有初始化
// bring up the kernel heap
lk_primary_cpu_init_level(LK_INIT_LEVEL_TARGET_EARLY, LK_INIT_LEVEL_HEAP - 1);
dprintf(SPEW, "initializing heap\n");
heap_init();

// initialize the kernel
lk_primary_cpu_init_level(LK_INIT_LEVEL_HEAP, LK_INIT_LEVEL_KERNEL - 1);
kernel_init();

lk_primary_cpu_init_level(LK_INIT_LEVEL_KERNEL, LK_INIT_LEVEL_THREADING - 1);

````

#### apps_init 初始化之后执行的操作

代码分析：<br/>

````
dprintf(SPEW, "calling apps_init()\n");
lk_primary_cpu_init_level(LK_INIT_LEVEL_TARGET, LK_INIT_LEVEL_APPS - 1);
apps_init();
````

打印完日志，执行所有 TARGET 到 APPS - 1 范围内的所有初始化HOOK，要想查找所有初始化的回调<br/>
只要查找 APPS 即可<br/>

实现APPS初始化的目前只有两个模块：ktrace 与 userboot<br/>

### userboot 入口分析

代码位置：magenta/kernel/lib/userboot/userboot.cpp<br/>
定义方式：LK_INIT_HOOK(userboot, userboot_init, LK_INIT_LEVEL_APPS - 1);
分析：将 userboot 定位在 LK_INIT_LEVEL_APPS - 1 代表这个代码将会在所有应用程序初始化之前<br/>

userboot同时也是系统从内核层代码，启动到用户层代码的入口；

userboot.cpp 会将 system/core/userboot 加载到内存，并且开启进程、线程分发器，
启动内部的代码.

system/core/userboot 是一个单独的模块，被认为是一个系统驱动，但并不是真正的驱动，<br/>
而是用于加载本地文件系统、并且启动用户空间的模块；入口点为 start.c 文件；<br/>

start.c 会遍历从 userboot.cpp 中传递过来的参数以及系统定义的各种 Handle，
找到包含 bootdata_vmo 的信息，这个就是 bootdata.bin 或者是 启动时指定的 user.bootfs<br/>
在这个启动数据文件中，查找 devmgr 和 libc。

代码如下：<br/>
````
// Locate the first bootfs bootdata section and decompress it.
// We need it to load devmgr and libc from.
// Later bootfs sections will be processed by devmgr.
````

## 关于 devmgr 启动

* start.c 中，在 parse_options 方法中，会查找对应的 OPTION_FILENAME 参数
* 这个参数的默认值是 #define OPTION_FILENAME_DEFAULT "bin/devmgr"
* 直接查找这个可执行文件；
* 查找到之后，创建进程、创建线程、调用 load_child_process 加载进程；
* load_child_process 会从bootfs中，找到指定的位置，并且加载到内存；
* 调用 elf_load_bootfs，内部会调用 bootfs_open(log, "program", fs, filename);
* 因此日志会出现： userboot: searching bootfs for program "bin/devmgr"
* bootfs_open 内部代码打印日志 print(log, "searching bootfs for ", purpose, " \"", filename, "\"\n", NULL);
* 解析 程序 devmgr 的ELF文件格式，查找使用的拦截器，发现 lib/ld.so.1
* 解析 程序 devmgr 使用的共享库 so，找到加载
* 加载成功之后，找到 devmgr 的代码入口地址，启动进程；
* loader-service 等待；

### devmgr 内部代码

位置：magenta/system/core/devmgr，这个程序主要的作用在于设备管理，以及 appmgr 的启动
appmgr 非常重要，用于内核之外的操作系统启动；

devmgr不属于内核模块，这个程序是一个用户空间模块。提供各种服务；

#### devmgr 内部模块划分

1. devmgr 程序，核心服务程序；
1. devhost 由 libdriver 提供的方法调用；参考 libdriver.abi.h 文件 真正的驱动代码入口
1. dmctl 桥接 devmgr 与 dm 指令

#### devmgr 启动代码

启动入口在 devmgr.c 文件的 main函数，初始化一些设备信息以及IO。<br/>

````
find_loadable_drivers("/boot/driver");
find_loadable_drivers("/boot/driver/test");
find_loadable_drivers("/boot/lib/driver");
````

其中如果附加 bootfs 准备好，即操作系统准备好，那么启动 appmgr<br/>

````
if (secondary_bootfs_ready()) {
    devmgr_start_appmgr(NULL);
}
````
如果 secondary_bootfs_ready，那么代表 /system 有效，因此启动 appmgr之前，会加载
所有在/system里面的驱动；<br/>

````
find_loadable_drivers("/system/driver");
find_loadable_drivers("/system/lib/driver");
````

之后启动 appmgr<br/>
````
devmgr_launch(fuchsia_job_handle, "appmgr", countof(argv_appmgr),
              argv_appmgr, NULL, -1, appmgr_hnds, appmgr_ids,
              appmgr_hnd_count, NULL);
````

代码查找指定位置 static const char* argv_appmgr[] = { "/system/bin/appmgr" };

devmgr 会加载 liblaunchpad 进行程序启动与运行；

## 以上的所有分析，都属于 magenta 范围

## 关于 appmgr

appmgr位置是 /system/bin/appmgr 这个程序，通过 launchpad 来启动的。<br/>
可以认为是Fuchsia 操作系统的第一个程序

appmgr工程位置 /fuchsia/application, <br/>
执行代码位置为 /fuchsia/application/src/manager/main.cc

也就是说，如果我们将 appmgr 替换为我们自己的，就可以基于 magenta 实现一套自己的操作系统了<br/>
因为magenta属于内核，我们可以在内核之上创建，实现特定的系统；

## 完整日志

````
+ echo CMDLINE: TERM=xterm-256color kernel.entropy-mixin=2c01cfd5ac07717992bf8ee37749222c53e37f0ea07e09a6b898e8615bd4cc1a kernel.halt-on-panic=true
CMDLINE: TERM=xterm-256color kernel.entropy-mixin=2c01cfd5ac07717992bf8ee37749222c53e37f0ea07e09a6b898e8615bd4cc1a kernel.halt-on-panic=true
+ exec /Users/vhly/Projects/fuchsia/fuchsia/buildtools/qemu/bin/qemu-system-aarch64 -m 2048 -serial stdio -vga none -device virtio-gpu-pci -net none -smp 4 -machine virt -kernel /Users/vhly/Projects/fuchsia/fuchsia/out/build-magenta/build-magenta-qemu-arm64/magenta.elf -cpu cortex-a53 -initrd /Users/vhly/Projects/fuchsia/fuchsia/out/debug-aarch64/user.bootfs -append 'TERM=xterm-256color kernel.entropy-mixin=2c01cfd5ac07717992bf8ee37749222c53e37f0ea07e09a6b898e8615bd4cc1a kernel.halt-on-panic=true '
[00000.000] 00000.00000> kernel command line: TERM=xterm-256color kernel.entropy-mixin=2c01cfd5ac07717992bf8ee37749222c53e37f0ea07e09a6b898e8615bd4cc1a
[00000.000] 00000.00000> kernel.halt-on-panic=true
[00000.000] 00000.00000> arm_gic_init max_irqs: 288
[00000.011] 00000.00000> test_time_conversion_check_result:264: FAIL, off by 72057594037927936
[00000.055] 00000.00000>
[00000.055] 00000.00000> welcome to lk/MP
[00000.055] 00000.00000>
[00000.055] 00000.00000> INIT: cpu 0, calling hook 0xffff000000074cc0 (global_prng_seed) at level 0x30000, flags 0x1
[00000.055] 00000.00000> INIT: cpu 0, calling hook 0xffff0000000517b0 (elf_build_id) at level 0x3fffe, flags 0x1
[00000.056] 00000.00000> INIT: cpu 0, calling hook 0xffff000000051708 (version) at level 0x3ffff, flags 0x1
[00000.056] 00000.00000> version:
[00000.056] 00000.00000> 	arch:     ARM64
[00000.056] 00000.00000> 	platform: GENERIC_ARM
[00000.056] 00000.00000> 	target:   QEMU_VIRT
[00000.056] 00000.00000> 	project:  MAGENTA_QEMU_ARM64
[00000.056] 00000.00000> 	buildid:  GIT_4DE2DECB5D9AE46A51EE038AA2EEC32930A33F30
[00000.056] 00000.00000> 	ELF build ID: 3da6b927f471052f8df54cff1deed1240b9436e4
[00000.056] 00000.00000> INIT: cpu 0, calling hook 0xffff000000054958 (vm_preheap) at level 0x3ffff, flags 0x1
[00000.059] 00000.00000> initializing heap
[00000.060] 00000.00000> INIT: cpu 0, calling hook 0xffff000000054c60 (vm) at level 0x50000, flags 0x1
[00000.060] 00000.00000> VM: reserving kernel region [ffff000000010000, ffff0000000b7000) flags 0x28 name 'kernel_code'
[00000.062] 00000.00000> VM: reserving kernel region [ffff0000000b7000, ffff0000000ea000) flags 0x8 name 'kernel_rodata'
[00000.062] 00000.00000> VM: reserving kernel region [ffff0000000ea000, ffff0000000ec000) flags 0x18 name 'kernel_data'
[00000.062] 00000.00000> VM: reserving kernel region [ffff0000000f0000, ffff000000149000) flags 0x18 name 'kernel_bss'
[00000.062] 00000.00000> VM: reserving kernel region [ffff000000149000, ffff00000114e000) flags 0x18 name 'kernel_bootalloc'
[00000.062] 00000.00000> initializing mp
[00000.062] 00000.00000> initializing threads
[00000.062] 00000.00000> initializing timers
[00000.062] 00000.00000> INIT: cpu 0, calling hook 0xffff000000035080 (debuglog) at level 0x6ffff, flags 0x1
[00000.062] 00000.00000> INIT: cpu 0, calling hook 0xffff0000000488f0 (thread_set_priority_experiment) at level 0x6ffff, flags 0x1
[00000.063] 00000.00000> thread set priority experiment is : DISABLED
[00000.063] 00000.00000> INIT: cpu 0, calling hook 0xffff000000074f78 (global_prng_thread_safe) at level 0x6ffff, flags 0x1
[00000.063] 00000.00000> creating bootstrap completion thread
[00000.069] 00000.00000> top of bootstrap2()
[00000.069] 00000.00000> INIT: cpu 0, calling hook 0xffff000000077688 (dpc) at level 0x70000, flags 0x1
[00000.069] 00000.00000> INIT: cpu 0, calling hook 0xffff00000009c870 (magenta) at level 0x70000, flags 0x1
[00000.072] 00000.00000> OOM: started thread
[00000.072] 00000.00000> ARM cpu 0: midr 0x410fd034 'ARM Cortex-a53 r0p4' mpidr 0x80000000 aff 0:0:0:0
[00000.073] 00000.00000> initializing platform
[00000.073] 00000.00000> Trying to start cpu1 returned: 0
[00000.073] 00000.00000> Trying to start cpu2 returned: 0
[00000.074] 00000.00000> Trying to start cpu3 returned: 0
[00000.074] 00000.00000> Trying to start cpu4 returned: fffffffe
[00000.074] 00000.00000> Trying to start cpu5 returned: fffffffe
[00000.074] 00000.00000> INIT: cpu 1, calling hook 0xffff00000002d0f0 (arm_generic_timer_init_secondary_cpu) at level 0x6ffff, flags 0x2
[00000.074] 00000.00000> Trying to start cpu6 returned: fffffffe
[00000.074] 00000.00000> INIT: cpu 2, calling hook 0xffff00000002d0f0 (arm_generic_timer_init_secondary_cpu) at level 0x6ffff, flags 0x2
[00000.074] 00000.00000> Trying to start cpu7 returned: fffffffe
[00000.074] 00000.00000> INIT: cpu 3, calling hook 0xffff00000002d0f0 (arm_generic_timer_init_secondary_cpu) at level 0x6ffff, flags 0x2
[00000.074] 00000.00000> ARM cpu 1: midr 0x410fd034 'ARM Cortex-a53 r0p4' mpidr 0x80000001 aff 0:0:0:1
[00000.074] 00000.00000> initializing target
[00000.074] 00000.00000> INIT: cpu 0, calling hook 0xffff00000002c468 (platform_dev_init) at level 0x90000, flags 0x1
[00000.074] 00000.00000> ARM cpu 2: midr 0x410fd034 'ARM Cortex-a53 r0p4' mpidr 0x80000002 aff 0:0:0:2
[00000.074] 00000.00000> entering scheduler on cpu 1
[00000.074] 00000.00000> entering scheduler on cpu 2
[00000.074] 00000.00000> ARM cpu 3: midr 0x410fd034 'ARM Cortex-a53 r0p4' mpidr 0x80000003 aff 0:0:0:3
[00000.074] 00000.00000> entering scheduler on cpu 3
[00000.091] 00000.00000> calling apps_init()
[00000.091] 00000.00000> INIT: cpu 0, calling hook 0xffff000000035a18 (ktrace) at level 0xaffff, flags 0x1
[00000.174] 00000.00000> ktrace: buffer at 0xffff000fcec5a000 (33554432 bytes)
[00000.174] 00000.00000> INIT: cpu 0, calling hook 0xffff0000000516e8 (userboot) at level 0xaffff, flags 0x1
[00000.174] 00000.00000> userboot: console init
[00000.175] 00000.00000> userboot: ramdisk      0x15f89000 @ 0xffff000008000000
[00000.340] 00000.00000> userboot: userboot rodata       0 @ [0xae92749a7000,0xae92749aa000)
[00000.341] 00000.00000> userboot: userboot code    0x3000 @ [0xae92749aa000,0xae92749b3000)
[00000.341] 00000.00000> userboot: vdso/full rodata       0 @ [0xae92749b3000,0xae92749b9000)
[00000.341] 00000.00000> userboot: vdso/full code    0x6000 @ [0xae92749b9000,0xae92749ba000)
[00000.344] 00000.00000> userboot: entry point             @ 0xae92749aab28
[00000.349] 01030.01037> userboot: option "TERM=xterm-256color"
[00000.350] 01030.01037> userboot: option "kernel.entropy-mixin.redacted=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
[00000.350] 01030.01037> userboot: option "kernel.halt-on-panic=true"
[00000.546] 01030.01037> userboot: searching bootfs for program "bin/devmgr"
[00000.548] 01030.01037> userboot: bin/devmgr has PT_INTERP "lib/ld.so.1"
[00000.549] 01030.01037> userboot: searching bootfs for dynamic linker "lib/ld.so.1"
[00000.555] 01030.01037> userboot: process bin/devmgr started.
[00000.555] 01030.01037> userboot: waiting for loader-service requests...
[00000.563] 01030.01037> userboot: searching bootfs for shared library "lib/libfs-management.so"
[00000.566] 01030.01037> userboot: searching bootfs for shared library "lib/liblaunchpad.so"
[00000.568] 01030.01037> userboot: searching bootfs for shared library "lib/libmxio.so"
[00000.573] 01044.01047> dso: id=c7f9f6598cbd9ce698c5d928565f4b3e2c401b74 base=0x00003aa524d38000 name=<application>
[00000.573] 01044.01047> dso: id=af2073c144bc0a11124f5f4ffd0f545c07d99599 base=0x0000f762bdb6f000 name=libfs-management.so
[00000.573] 01044.01047> dso: id=59498dd56510677a37cddfeda2acef3c95927cf8 base=0x000031a281c56000 name=liblaunchpad.so
[00000.573] 01044.01047> dso: id=0692828667c4d8193216e3ec432deed3bf7e7d00 base=0x00001ec612677000 name=libmxio.so
[00000.573] 01044.01047> dso: id=c460f882ad8a43e031e8024e6a2ddeb1556d9b72 base=0x0000a95e05c3a000 name=<vDSO>
[00000.573] 01044.01047> dso: id=fd84a4f1cafbf7e45f67e4e10e46975926390a47 base=0x000081147e4b5000 name=libc.so
[00000.588] 01030.01037> userboot: loader-service channel peer closed
[00000.591] 01030.01037> userboot: finished!
[00000.599] 01044.01047> devmgr: main()
[00000.600] 01044.01047> devmgr: init
[00000.601] 01044.01047> coordinator_init()
[00000.605] 01044.01047> devmgr: vfs init
[00017.212] 01044.01047> cmdline: TERM=xterm-256color
[00017.212] 01044.01047> cmdline: kernel.entropy-mixin.redacted=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
[00017.212] 01044.01047> cmdline: kernel.halt-on-panic=true
[00017.294] 01044.01047> devmgr: launch /boot/bin/crashlogger (crashlogger) OK
[00017.461] 01044.01245> devmgr: shell startup
[00017.465] 01044.01047> devmgr: launch /system/bin/appmgr (appmgr) OK
[00017.525] 01044.01047> devmgr: coordinator()
[00017.740] 01044.01309> devmgr: launch /boot/bin/netsvc (netsvc) OK
[00017.873] 01044.01309> devmgr: launch /boot/bin/virtual-console (virtual-console) OK
[00017.975] 01198.01237> crashlogger service ready
[00018.005] 01044.01047> for_each_note: pread(eh) failed
[00018.008] 01044.01047> devcoord: error reading info from '/boot/driver/test'
[00018.179] 01044.01047> devcoord: launch devhost 'devhost:root': pid=1719
[00018.258] 01044.01047> devcoord: launch devhost 'devhost:misc': pid=1763
[00018.387] 01044.01047> devcoord: launch devhost 'devhost:platform': pid=1812
[00018.615] 01044.01309> devmgr: block watch waiting...
[00018.916] 01249.01293> appmgr: Ignoring missing config file: /system/data/appmgr/startup.config
[00019.209] 01044.01047> devcoord: launch devhost 'devhost:pci#1:1af4:1050': pid=2164
[00019.292] 01044.01245> devmgr: launch /boot/bin/sh (sh:console) OK
[00019.411] 01763.01793> usb_virtual_bus_bind
[00019.820] 02164.02203> object 0xffff00003d99dd28 base 0x3f008000 size 0x1000 ref 1
$ [00019.990] 02164.02424> virtio-gpu: found display x 0 y 0 w 1024 h 768 flags 0x0
[00020.074] 02164.02203> fb: 1024 x 768 (stride=1024 pxlsz=4 format=5): 3145728 bytes @ 0x859d878c6000 SW
[00021.595] 02543.02563> file:///boot/bin/sh: 0: Can't open /system/autorun
[00022.642] 02691.02724> [INFO:main.cc(15)] time-service: starting
[00022.651] 02691.02724> [INFO:time_service.cc(22)] time-service: started
[00022.844] 02691.02724> [INFO:time_service.cc(49)] time-service: Updating system time, attempt: 1
[00022.996] 02780.02816> wlanstack: started
[00023.073] 02737.02765> netstack: started
[00023.220] 02737.02765> netstack: socket dispatcher started
[00023.242] 02780.03415> wlanstack: watching for wlan devices
[00023.251] 02737.02765> netstack: watching for ethernet devices
[00023.697] 02691.02724> [ERROR:apps/time-service/src/roughtime_server.cc(62)] time-service: Failed to resolve roughtime.sandbox.google.com:2002: Name does not resolve
[00023.698] 02691.02724> [INFO:time_service.cc(54)] time-service: Can't get time, sleeping for 10 sec
[00026.141] 04572.04597> [INFO:display_watcher.cc(45)] SceneManager: Acquired display /dev/class/display/000.
[00026.150] 04572.04597> [ERROR:apps/mozart/src/scene_manager/displays/display_watcher.cc(57)] IOCTL_DISPLAY_GET_FB failed: result=-2
[00026.151] 04572.04597> [ERROR:apps/mozart/src/scene_manager/main.cc(37)] No default display, SceneManager exiting
[00026.185] 04271.04291> [ERROR:apps/mozart/src/view_manager/view_registry.cc(117)] Exiting due to scene manager connection error.
[00026.191] 03832.03853> [ERROR:apps/mozart/src/root_presenter/app.cc(117)] SceneManager died, destroying view trees.
[00026.202] 02077.02271> [ERROR:application/src/bootstrap/app.cc(127)] Singleton scene_manager died
[00026.238] 02077.02271> [ERROR:application/src/bootstrap/app.cc(127)] Singleton view_manager died
[00027.418] 05022.05042> [INFO:display_watcher.cc(45)] SceneManager: Acquired display /dev/class/display/000.
[00027.426] 05022.05042> [ERROR:apps/mozart/src/scene_manager/displays/display_watcher.cc(57)] IOCTL_DISPLAY_GET_FB failed: result=-2
[00027.428] 05022.05042> [ERROR:apps/mozart/src/scene_manager/main.cc(37)] No default display, SceneManager exiting
[00027.466] 04868.04886> [ERROR:apps/mozart/src/view_manager/view_registry.cc(117)] Exiting due to scene manager connection error.
[00027.518] 02077.02271> [ERROR:application/src/bootstrap/app.cc(127)] Singleton scene_manager died
[00027.527] 02077.02271> [ERROR:application/src/bootstrap/app.cc(127)] Singleton view_manager died
[00027.753] 03863.04431> [INFO:vulkan_application.cc(77)] Vulkan call 'vk.CreateInstance(&create_info, nullptr, &instance)' failed with error VK_ERROR_EXTENSION_NOT_PRESENT
[00027.757] 03863.04431> [INFO:vulkan_application.cc(79)] Could not create application instance.
[00027.758] 03863.04431> [ERROR:flutter/content_handler/vulkan_surface_producer.cc(41)] Instance proc addresses have not been setup.
[00027.758] 03863.04431> [ERROR:flutter/content_handler/vulkan_surface_producer.cc(23)] Flutter engine: Vulkan surface producer initialization: Failed
[00027.774] 03863.04431> [FATAL:flutter/content_handler/session_connection.cc(38)] Check failed: false. Session connection was terminated.
[00027.779] 01198.01237> <== exception: process flutter:userpicker_device_shell[3863] thread pthread_t:0xaf19484b0000[4431]
[00027.780] 01198.01237> <== sw breakpoint, PC at 0x9de2cc0bad08
[00027.781] 01198.01237>  x0      0xd32ddb77dcf0 x1      0x9de2cc17d0b8 x2      0x9de2cc17d0b4 x3                   0
[00027.781] 01198.01237>  x4                   0 x5                   0 x6              0xffff x7             0x10000
[00027.781] 01198.01237>  x8                 0x3 x9                 0x8 x10                  0 x11     0x9de2cc17e520
[00027.781] 01198.01237>  x12                  0 x13     0x66dd68113c00 x14 0xb310c22f88001968 x15     0x66dd68114048
[00027.782] 01198.01237>  x16     0xfa8e2d3ede08 x17     0x9de2cc0bad08 x18               0x45 x19     0x66dd68114eb8
[00027.782] 01198.01237>  x20     0x66dd68114e80 x21     0xd32ddb77dcf0 x22     0xaf19484b04b8 x23     0x66dd68114ea0
[00027.782] 01198.01237>  x24 0xb310c22f88001968 x25     0x66dd68114e98 x26     0xaf19484b04b8 x27     0x66dd68115480
[00027.782] 01198.01237>  x28 0xb310c22f88001968 x29     0x81f6c8d94c40 lr      0xfa8e2c798d14 sp      0x81f6c8d94c40
[00027.782] 01198.01237>  pc      0x9de2cc0bad08 psr         0x60000000
[00027.783] 01198.01237> bottom of user stack:
[00027.784] 01198.01237> 0x000081f6c8d94c40: c8d94c90 000081f6 2c799440 0000fa8e |.L......@.y,....|
[00027.784] 01198.01237> 0x000081f6c8d94c50: 68115478 000066dd 68115478 000066dd |xT.h.f..xT.h.f..|
[00027.785] 01198.01237> 0x000081f6c8d94c60: 484b04b8 0000af19 681150f8 000066dd |..KH.....P.h.f..|
[00027.785] 01198.01237> 0x000081f6c8d94c70: 88001968 b310c22f 68115100 000066dd |h.../....Q.h.f..|
[00027.785] 01198.01237> 0x000081f6c8d94c80: 484b04b8 0000af19 68114eb8 000066dd |..KH.....N.h.f..|
[00027.785] 01198.01237> 0x000081f6c8d94c90: c8d94cd0 000081f6 2c6b9dc4 0000fa8e |.L........k,....|
[00027.785] 01198.01237> 0x000081f6c8d94ca0: 00000000 00000000 81d6b1f0 00007253 |............Sr..|
[00027.785] 01198.01237> 0x000081f6c8d94cb0: 00400004 00000000 68115100 000066dd |..@......Q.h.f..|
[00027.786] 01198.01237> 0x000081f6c8d94cc0: 00000001 00000000 81d6b1f0 00007253 |............Sr..|
[00027.786] 01198.01237> 0x000081f6c8d94cd0: c8d94d30 000081f6 2c795314 0000fa8e |0M.......Sy,....|
[00027.786] 01198.01237> 0x000081f6c8d94ce0: 00000000 00000000 681155a8 000066dd |.........U.h.f..|
[00027.786] 01198.01237> 0x000081f6c8d94cf0: 88001968 b310c22f 681155b0 000066dd |h.../....U.h.f..|
[00027.786] 01198.01237> 0x000081f6c8d94d00: 484b04b8 0000af19 81d720c0 00007253 |..KH..... ..Sr..|
[00027.786] 01198.01237> 0x000081f6c8d94d10: 68115480 000066dd 00400004 00000000 |.T.h.f....@.....|
[00027.787] 01198.01237> 0x000081f6c8d94d20: 81d720c0 00007253 00000001 00000000 |. ..Sr..........|
[00027.787] 01198.01237> 0x000081f6c8d94d30: c8d94d90 000081f6 2c7c71e8 0000fa8e |.M.......q|,....|
[00027.787] 01198.01237> arch: aarch64
[00027.877] 01198.01237> dso: id=d0a881555ec4bfc1 base=0xfa8e2bc2c000 name=app:flutter:userpicker_device_shell
[00027.878] 01198.01237> dso: id=c460f882ad8a43e031e8024e6a2ddeb1556d9b72 base=0xec63ce164000 name=<vDSO>
[00027.878] 01198.01237> dso: id=72e66f7b98db3468 base=0xe52b5cd08000 name=libicuuc.so
[00027.878] 01198.01237> dso: id=da0ad7bad5d5a34e base=0xe03445331000 name=libcrypto.so
[00027.878] 01198.01237> dso: id=d04c36b1f771fd32 base=0xd9b434537000 name=libc++abi.so.1
[00027.878] 01198.01237> dso: id=f5de27939da5b2f3 base=0xd84ad3b98000 name=libunwind.so.1
[00027.878] 01198.01237> dso: id=41a85b9ed04840e92fc3f44490744740cc90dc97 base=0xd7a0d18f5000 name=libasync-default.so
[00027.879] 01198.01237> dso: id=4785737e094a144a base=0xd32ddb69d000 name=libc++.so.2
[00027.879] 01198.01237> dso: id=f373cc27695dbddb base=0xc387edc33000 name=libicui18n.so
[00027.879] 01198.01237> dso: id=c3e038a0ada0677e base=0xc2552778d000 name=libssl.so
[00027.879] 01198.01237> dso: id=fd84a4f1cafbf7e45f67e4e10e46975926390a47 base=0x9de2cc09e000 name=libc.so
[00027.879] 01198.01237> dso: id=5a8d7c2b5f68a61c base=0x89da71da1000 name=libvulkan.so
[00027.880] 01198.01237> dso: id=8464175fe7f7e78d base=0x73e997c40000 name=libz.so
[00027.880] 01198.01237> dso: id=e604515cecc24d62 base=0x49cdbb2ab000 name=libminizip.so
[00027.881] 01198.01237> dso: id=59498dd56510677a37cddfeda2acef3c95927cf8 base=0x48cee145d000 name=liblaunchpad.so
[00027.881] 01198.01237> dso: id=ef6f325c4505c16d base=0x46c57a508000 name=libftl.so
[00027.881] 01198.01237> dso: id=aff53a726da535c9 base=0x35e6a0593000 name=libmtl.so
[00027.881] 01198.01237> dso: id=0692828667c4d8193216e3ec432deed3bf7e7d00 base=0xa048de5b000 name=libmxio.so
[00027.881] 01198.01237> dso: id=b37a9d82ae8c1187 base=0x4d108f7e000 name=libftl_logging.so
[00027.922] 01198.01237> bt#01: pc 0x9de2cc0bad08 sp 0x81f6c8d94c40 (libc.so,0x1cd08)
[00027.939] 01198.01237> bt#02: pc 0xfa8e2c798d14 sp 0x81f6c8d94c40 (app:flutter:userpicker_device_shell,0xb6cd14)
[00027.942] 01198.01237> bt#03: pc 0xfa8e2c799440 sp 0x81f6c8d94c50 (app:flutter:userpicker_device_shell,0xb6d440)
[00027.944] 01198.01237> bt#04: pc 0xfa8e2c6b9dc4 sp 0x81f6c8d94c60 (app:flutter:userpicker_device_shell,0xa8ddc4)
[00027.945] 01198.01237> bt#05: pc 0xfa8e2c795314 sp 0x81f6c8d94c70 (app:flutter:userpicker_device_shell,0xb69314)
[00027.946] 01198.01237> bt#06: pc 0xfa8e2c7c71e8 sp 0x81f6c8d94c80 (app:flutter:userpicker_device_shell,0xb9b1e8)
[00027.947] 01198.01237> bt#07: pc 0xfa8e2c7c086c sp 0x81f6c8d94c90 (app:flutter:userpicker_device_shell,0xb9486c)
[00027.948] 01198.01237> bt#08: pc 0xfa8e2c7c1e9c sp 0x81f6c8d94ca0 (app:flutter:userpicker_device_shell,0xb95e9c)
[00027.949] 01198.01237> bt#09: pc 0xfa8e2c7c10bc sp 0x81f6c8d94cb0 (app:flutter:userpicker_device_shell,0xb950bc)
[00027.950] 01198.01237> bt#10: pc 0xfa8e2c7c1a70 sp 0x81f6c8d94cc0 (app:flutter:userpicker_device_shell,0xb95a70)
[00027.951] 01198.01237> bt#11: pc 0xfa8e2c7c128c sp 0x81f6c8d94cd0 (app:flutter:userpicker_device_shell,0xb9528c)
[00027.952] 01198.01237> bt#12: pc 0xfa8e2c7c11d8 sp 0x81f6c8d94ce0 (app:flutter:userpicker_device_shell,0xb951d8)
[00027.953] 01198.01237> bt#13: pc 0xfa8e2c7c0124 sp 0x81f6c8d94cf0 (app:flutter:userpicker_device_shell,0xb94124)
[00027.956] 01198.01237> bt#14: pc 0xfa8e2c7c4344 sp 0x81f6c8d94d00 (app:flutter:userpicker_device_shell,0xb98344)
[00027.957] 01198.01237> bt#15: pc 0xfa8e2c7bb24c sp 0x81f6c8d94d10 (app:flutter:userpicker_device_shell,0xb8f24c)
[00027.958] 01198.01237> bt#16: pc 0x9de2cc0b764c sp 0x81f6c8d94d20 (libc.so,0x1964c)
[00027.962] 01198.01237> bt#17: end
[00029.084] 03467.03489> [INFO:netconnector_impl.cc(188)] mDNS started, host name fuchsia-1
[00029.098] 03467.03489> [INFO:mdns_interface_transceiver.cc(58)] Starting mDNS on interface en1, address 127.0.0.1
[00029.110] 02737.03938> netstack: convSockOpt: TODO IPPROTO_IP optname=32
[00029.113] 02737.03938> netstack: convSockOpt: TODO IPPROTO_IP optname=2
[00033.701] 02691.02724> [INFO:time_service.cc(49)] time-service: Updating system time, attempt: 2
[00033.716] 02691.02724> [ERROR:apps/time-service/src/roughtime_server.cc(62)] time-service: Failed to resolve roughtime.sandbox.google.com:2002: Name does not resolve
[00033.717] 02691.02724> [INFO:time_service.cc(54)] time-service: Can't get time, sleeping for 10 sec
[00034.172] 02583.02604> [WARNING:apps/modular/lib/util/filesystem.cc(49)] /data is not persistent. Did you forget to configure it?
[00043.718] 02691.02724> [INFO:time_service.cc(49)] time-service: Updating system time, attempt: 3
[00043.730] 02691.02724> [ERROR:apps/time-service/src/roughtime_server.cc(62)] time-service: Failed to resolve roughtime.sandbox.google.com:2002: Name does not resolve
````
