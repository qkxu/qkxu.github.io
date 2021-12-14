---
layout: post
title: libvirt找不到支持kvm的guest
category: VMM
tags: VMM
description: libvirt找不到支持kvm的guest
---
#  libvirt找不到支持kvm的guest

在物理主机上创建虚拟机报错，不支持domaintype=kvm的虚拟机（偶现）。

```sh
error: Failed to create domain from test.xml
error: invalid argument: could not find capabilities for arch=x86_64 domaintype=kvm
```

主机bios已经打开了支持kvm配置，并且查看kvm model，也已经加载。

### 问题环境

宿主机系统（host）：CentOS Linux release 7.7.1908 (Core)

虚拟化软件版本：

libvirt：5.0.0

qemu：2.12.0

### virsh capabilities 分析

执行virsh capabilities查看结果：

```
 <guest>
    <os_type>hvm</os_type>
    <arch name='x86_64'>
      <wordsize>64</wordsize>
      <emulator>/usr/libexec/qemu-kvm</emulator>
      <machine maxCpus='240'>pc-i440fx-rhel7.6.0</machine>
      <machine canonical='pc-i440fx-rhel7.6.0' maxCpus='240'>pc</machine>
      ...
      <domain type='qemu'/>
      #<domain type='kvm'/> 没有此选项
    </arch>
```

流程分析：

```c
(gdb) bt
#0  virQEMUCapsInit (cache=0x7f8c4c10ad00) at qemu/qemu_capabilities.c:900
#1  0x00007f8c550c49b0 in virQEMUDriverCreateCapabilities (driver=driver@entry=0x7f8c4c11b0a0) at qemu/qemu_conf.c:1098
#2  0x00007f8c550c4c83 in virQEMUDriverGetCapabilities (driver=0x7f8c4c11b0a0, refresh=<optimized out>) at qemu/qemu_conf.c:1168
#3  0x00007f8c55128443 in qemuConnectGetCapabilities (conn=<optimized out>) at qemu/qemu_driver.c:1434
#4  0x00007f8c823ebec3 in virConnectGetCapabilities (conn=0x7f8c3d8efa50) at libvirt-host.c:403
#5  0x000055e7846f0b97 in remoteDispatchConnectGetCapabilities (server=0x55e785f6f0b0, msg=0x55e785fa5600, ret=0x7f8c3d814b70, rerr=0x7f8c71392c10, client=0x55e785fa5130) at remote/remote_daemon_dispatch_stubs.h:790
#6  remoteDispatchConnectGetCapabilitiesHelper (server=0x55e785f6f0b0, client=0x55e785fa5130, msg=0x55e785fa5600, rerr=0x7f8c71392c10, args=0x7f8c3d8aa390, ret=0x7f8c3d814b70) at remote/remote_daemon_dispatch_stubs.h:769
#7  0x00007f8c82308655 in virNetServerProgramDispatchCall (msg=0x55e785fa5600, client=0x55e785fa5130, server=0x55e785f6f0b0, prog=0x55e785f96610) at rpc/virnetserverprogram.c:435
#8  virNetServerProgramDispatch (prog=0x55e785f96610, server=server@entry=0x55e785f6f0b0, client=0x55e785fa5130, msg=0x55e785fa5600) at rpc/virnetserverprogram.c:302
#9  0x00007f8c8230ee8d in virNetServerProcessMsg (msg=<optimized out>, prog=<optimized out>, client=<optimized out>, srv=0x55e785f6f0b0) at rpc/virnetserver.c:142
#10 virNetServerHandleJob (jobOpaque=<optimized out>, opaque=0x55e785f6f0b0) at rpc/virnetserver.c:163
#11 0x00007f8c8223c3e1 in virThreadPoolWorker (opaque=opaque@entry=0x55e785f4e0e0) at util/virthreadpool.c:163
#12 0x00007f8c8223b768 in virThreadHelper (data=<optimized out>) at util/virthread.c:206
#13 0x00007f8c7fa1ee65 in start_thread () from /lib64/libpthread.so.0
#14 0x00007f8c7f34088d in clone () from /lib64/libc.so.6

```

其中virQEMUCapsInit(driver->qemuCapsCache)，cache是从driver中取得：

在此函数中会进行一些host主机基本信息的获取，然后执行virQEMUCapsInitGuest来获取guest的配置，根据

qemuCaps即 cache来判断是否添加kvm类型的guest os.

```c
//virQEMUCapsInitGuest
/* Ignore binary if extracting version info fails */
    if (binary) {
    //在cache中查看相关的属性值
        if (!(qemuCaps = virQEMUCapsCacheLookup(cache, binary))) {
            virResetLastError();
            VIR_FREE(binary);
        }
    }
    ret = virQEMUCapsInitGuestFromBinary(caps,
                                         binary, qemuCaps,
                                         guestarch);
...
//virQEMUCapsInitGuestFromBinary
//若支持QEMU_CAPS_KVM，则添加，出问题的环境没有添加，问题则出在这里
if (virQEMUCapsGet(qemuCaps, QEMU_CAPS_KVM)) {
        if (virCapabilitiesAddGuestDomain(guest,
                                          VIR_DOMAIN_VIRT_KVM,
                                          NULL,
                                          NULL,
                                          0,
                                          NULL) == NULL) {
            goto cleanup;
        }
    }
```

### cache的初始化

查询过程中可以看到cache就一直存在，查看cache的源头，只能从libvirt初始化的定位查询

```sh
Breakpoint 1, virQEMUCapsCacheLookup (cache=cache@entry=0x7fffc0136b50, binary=0x7fffc0155b40 "/usr/libexec/qemu-kvm") at qemu/qemu_capabilities.c:4855
4855	{
(gdb) bt
#0  virQEMUCapsCacheLookup (cache=cache@entry=0x7fffc0136b50, binary=0x7fffc0155b40 "/usr/libexec/qemu-kvm") at qemu/qemu_capabilities.c:4855
#1  0x00007fffcabdd864 in virQEMUCapsInitGuest (guestarch=VIR_ARCH_I686, hostarch=VIR_ARCH_X86_64, cache=0x7fffc0136b50, caps=0x7fffc0154760) at qemu/qemu_capabilities.c:781
#2  virQEMUCapsInit (cache=0x7fffc0136b50) at qemu/qemu_capabilities.c:944
#3  0x00007fffcac2f600 in virQEMUDriverCreateCapabilities (driver=driver@entry=0x7fffc0114190) at qemu/qemu_conf.c:1098
#4  0x00007fffcac791ba in qemuStateInitialize (privileged=true, callback=<optimized out>, opaque=<optimized out>) at qemu/qemu_driver.c:927
#5  0x00007ffff7661f1f in virStateInitialize (privileged=true, callback=callback@entry=0x555555577c30 <daemonInhibitCallback>, opaque=opaque@entry=0x5555557f6720) at libvirt.c:657
#6  0x0000555555577c8b in daemonRunStateInit (opaque=0x5555557f6720) at remote/remote_daemon.c:796
#7  0x00007ffff74d5d82 in virThreadHelper (data=<optimized out>) at util/virthread.c:206
#8  0x00007ffff4a8ae65 in start_thread () from /lib64/libpthread.so.0
#9  0x00007ffff43ac88d in clone () from /lib64/libc.so.6
```

在libvirt初始化时，会执行virQEMUCapsCacheLookup，进而执行virFileCacheValidate

```c
static void
virFileCacheValidate(virFileCachePtr cache,
                     const char *name,
                     void **data)
{
    //调用驱动isValid函数
    if (*data && !cache->handlers.isValid(*data, cache->priv)) {
        VIR_DEBUG("Cached data '%p' no longer valid for '%s'",
                  *data, NULLSTR(name));
        if (name)
            virHashRemoveEntry(cache->table, name);
        *data = NULL;
    }
    //失效重新生成data
    if (!*data && name) {
        VIR_DEBUG("Creating data for '%s'", name);
        *data = virFileCacheNewData(cache, name);
        if (*data) {
            VIR_DEBUG("Caching data '%p' for '%s'", *data, name);
            if (virHashAddEntry(cache->table, name, *data) < 0) {
                virObjectUnref(*data);
                *data = NULL;
            }
        }
    }
}
```

cache的生成

```c
static void *
virFileCacheNewData(virFileCachePtr cache,
                    const char *name)
{
    void *data = NULL;
    int rv;
    //从文件缓存中获取，同样会进行isValid校验是否有效
    if ((rv = virFileCacheLoad(cache, name, &data)) < 0)
        return NULL;

    if (rv == 0) {
        //调用驱动newData函数重新生成
        if (!(data = cache->handlers.newData(name, cache->priv)))
            return NULL;

        if (virFileCacheSave(cache, name, data) < 0) {
            virObjectUnref(data);
            data = NULL;
        }
    }

    return data;
}
```

当初始化libvirt会检查/var/cache/libvirt/qemu/capabilities/*.xml是否有效或者存在，当存在并且有效时，则从文件中加载，否则重新生成，生成的代码块都在virQEMUCapsNewData中，其中virQEMUCapsInitQMP是关键一步

大致过程：

1. 创建qemu进程

   ```sh
   /usr/libexec/qemu-kvm -S -no-user-config -nodefaults -nographic -machine none,accel=kvm:tcg -qmp unix:/var/lib/libvirt/qemu/capabilities.monitor.sock,server,nowait -pidfile /var/lib/libvirt/qemu/capabilities.pidfile -daemonize
   ```

   其中accel=kvm:tcg表示采用kvm与tcg加速器，如果kvm支持，优先采用kvm，不支持则qemu全模拟

2. qmp命令查询capability

   查询执行的命令比较多，功能点查询较多，以查询是否支持kvm为例，采用的qmp命令：

   ```sh
   {"execute":"query-kvm"}
   ```

   通过返回值来判定是否支持kvm

   ```c
   static int
   virQEMUCapsProbeQMPKVMState(virQEMUCapsPtr qemuCaps,
                               qemuMonitorPtr mon)
   {
       bool enabled = false;
       bool present = false;
   
       if (qemuMonitorGetKVMState(mon, &enabled, &present) < 0)
           return -1;
   
       if (present && enabled)
           virQEMUCapsSet(qemuCaps, QEMU_CAPS_KVM);
   
       return 0;
   }
   ```

3. 当查询完毕时会把进程关闭，做一些清理资源的动作

把各种查询结果保存到cache，并记录文件。



### cache系统的抽象

libvirt代码对于底层虚拟化的capabilities做了cache，cache中有一套完整的流程，在virfilecache.c可以看到各个接口中的细节

大致包含如下：

1. 创建cache 

   首先校验文件是否存在，如果存在并且有效（isValid），则从文件加载（loadFile）。否则重新生成cache（newData），并将cache保存到文件（saveFile），方便下次加载。

2. 查询 cache

   查询cache时，首先会判断cache是否有效（isValid），如果无效则重新走创建cache流程，如果有效，则直接返回对应结果。

当然只有再libvirt对接qemu时才采用这套流程（对接qemu时实现了这套接口），在qemu_capabilities.c中可以看到各个接口具体的实现

```c
virFileCacheHandlers qemuCapsCacheHandlers = {
    .isValid = virQEMUCapsIsValid,
    .newData = virQEMUCapsNewData,
    .loadFile = virQEMUCapsLoadFile,
    .saveFile = virQEMUCapsSaveFile,
    .privFree = virQEMUCapsCachePrivFree,
};
```

virsh capabilities中返回的项目比较多，当遇到具体的capabilities中某个项时可以根据这套流程，针对性的查看代码细节。

### 回归问题本身

了解了cache以后就按部就班的定位问题了，qemuMonitorJSONGetKVMState中执行了qpm命令（query-kvm）进行查询，手动模拟libvirt重启时的进程启动过程

```sh
/usr/libexec/qemu-kvm -S -no-user-config -nodefaults -nographic -machine none,accel=kvm:tcg -qmp unix:/var/lib/libvirt/qemu/capabilities.monitor.sock,server,nowait -pidfile /var/lib/libvirt/qemu/capabilities.pidfile -daemonize
```

针对qemu monitor启动的方式不同，有不同的连接方式：

```sh
# qemu monitor采用tcp方式，监听在127.0.0.1上，端口为4444
/usr/libexec/qemu-kvm -qmp tcp:127.0.0.1:4444,server,nowait

# qemu monitor采用unix socket，socket文件生成于/opt/qmp.socket
/usr/libexec/qemu-kvm -qmp unix:/opt/qmp.socket,server,nowait
```

连接qemu monitor：

```sh
# tcp可以通过telnet进行连接，方法如下
> telnet 127.0.0.1 1234
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
{"QMP": {"version": {"qemu": {"micro": 0, "minor": 12, "major": 2}, "package": "qemu-kvm-ev-2.12.0-33.1.fh.3.4.el7"}, "capabilities": []}}
# unix socket可以通过nc -U进行连接，方法如下
> nc -U /opt/qmp.socket
{"QMP": {"version": {"qemu": {"micro": 0, "minor": 12, "major": 2}, "package": "qemu-kvm-ev-2.12.0-33.1.fh.3.4.el7"}, "capabilities": []}}
```

连接后就处于等待状态，但是还不能使用，必须先执行下如下命令，然后再执行其他的qmp命令即可。

```sh
{ "execute" : "qmp_capabilities" }
```

执行命令查看返回值:

```sh
{"execute":"query-kvm"}
{"return": {"enabled": false, "present": true}}
```

返回的enabled为false，导致libvirt认为底层不支持kvm，继续跟踪qemu代码分析定位。

```c
KvmInfo *qmp_query_kvm(Error **errp)
{
    KvmInfo *info = g_malloc0(sizeof(*info));
    //返回kvm_allowed此变量的值，重点在这里
    info->enabled = kvm_enabled();
    //编译时定义了CONFIG_KVM就返回为true
    info->present = kvm_available();

    return info;
}
```

查看kvm_allowed变量的使用

```c
//kvm_all.c
static void kvm_accel_class_init(ObjectClass *oc, void *data)
{
    AccelClass *ac = ACCEL_CLASS(oc);
    ac->name = "KVM";
    ac->init_machine = kvm_init;
    ac->allowed = &kvm_allowed;
}
```

kvm_allowed即ac->allowed。

qemu启动过程比较复杂，这里只关注改变量的变化，摘取主要代码：

```c
//accel.c
//accel_init_machine方法在qemu初始化加速器（vl.c中configure_accelerator)时会被调动
static int accel_init_machine(AccelClass *acc, MachineState *ms)
{
    ObjectClass *oc = OBJECT_CLASS(acc);
    const char *cname = object_class_get_name(oc);
    AccelState *accel = ACCEL(object_new(cname));
    int ret;
    ms->accelerator = accel;
    *(acc->allowed) = true;
    //调用加速器的init_machine方法
    ret = acc->init_machine(ms);
    if (ret < 0) {
        ms->accelerator = NULL;
        *(acc->allowed) = false;
        object_unref(OBJECT(accel));
    } else {
        accel_register_compat_props(ms->accelerator);
    }
    return ret;
}
```

这里只是一个抽象接口，实现都在具体的accel（加速器）中，启动时指定了accel=kvm:tcg设定了两个accel，qemu会优先选择第一个，当第一个失败时会尝试第二个。对于kvm accel来说，即kvm_init函数，在kvm_init函数中

```c
static int kvm_init(MachineState *ms)
{
    MachineClass *mc = MACHINE_GET_CLASS(ms);
    ...
    //打开kvm设备
    s->fd = qemu_open("/dev/kvm", O_RDWR);
    if (s->fd == -1) {
        fprintf(stderr, "Could not access KVM kernel module: %m\n");
        ret = -errno;
        goto err;
    }
    //ioctl获取kvm一些信息
    ret = kvm_ioctl(s, KVM_GET_API_VERSION, 0);
    ...
    do {
        //尝试创建虚拟机
        ret = kvm_ioctl(s, KVM_CREATE_VM, type);
    } while (ret == -EINTR);
```

如果其中有返回错误，即ret < 0  则将allowed置为false，即返回给libvirt的enabled值为false。

打开libvirt debug级别日志，在日志中可以看到：

```
21587 Could not access KVM kernel module: Permission denied
21588 qemu-kvm: failed to initialize KVM: Permission denied
21589 qemu-kvm: Back to tcg accelerator
```

即打开kvm时没有权限。

```sh
ll /dev/kvm 
crw-rw-rw-+ 1 root kvm 10, 232 Jul 14 00:32 /dev/kvm
```

可以看到文件权限最后有个+号，即启用了linux acl控制（可用getfacl与getfacl管理）：

```sh
getfacl /dev/kvm
getfacl: Removing leading '/' from absolute path names
# file: dev/kvm
# owner: root
# group: kvm
user::rw-
group::---
mask::rw-
other::rw-
```

可以看到kvm所属的组的权限为空，而qemu属于kvm group所以导致了没有权限。

而/dev/kvm设备是由内核模块kvm_intel加载而来的，ll可以看到kvm的主次设备号为：10,232，查看主设备对应的关系为

```
cat /proc/devices 
Character devices:
  1 mem
  4 /dev/vc/0
  4 tty
  4 ttyS
  5 /dev/tty
  5 /dev/console
  5 /dev/ptmx
  7 vcs
 10 misc
 13 input
 21 sg
...
```

即kvm设备是个misc设备，查看内核代码可以看到kvm确实是注册到misc字符设备驱动中的，并在include/linux/miscdevice.h中定义了从设备号#define KVM_MINOR		232

默认加载kvm module 时的权限为：

```
ll /dev/kvm 
crw------- 1 root root 10, 232 Jul 18 23:06 /dev/kvm
```

当安装后qemu后，在spec安装脚本中可以看到，对于kvm设备权限做了几件事：

1. 添加相关组与权限

   ```
   getent group kvm >/dev/null || groupadd -g 36 -r kvm
   getent group qemu >/dev/null || groupadd -g 107 -r qemu
   getent passwd qemu >/dev/null || \
   useradd -r -u 107 -g qemu -G kvm -d / -s /sbin/nologin \
     -c "qemu user" qemu
   ```

2. 将80-kvm.rules写入udev文件中，这个后续重启自动生效，本地生效参见步骤3，而80-kvm.rules内容为：

    ```
    KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"
    ```

   即把kvm设备的group修改为kvm，并将权限置为666

3. 然后执行udev更新命令,因为步骤2中已经写入了新的rules，所以加载后生效

   ```
   %udev_rules_update
   sh %{_sysconfdir}/sysconfig/modules/kvm.modules &> /dev/null || :
       udevadm trigger --subsystem-match=misc --sysname-match=kvm --action=add || :
   ```

   当然从rpm包中可以直接看到

   ```sh
   rpm -qp --scripts qemu-kvm-ev-2.12.0-33.1.fh.3.4.el7.x86_64.rpm
   postinstall scriptlet (using /bin/sh):
   # load kvm modules now, so we can make sure no reboot is needed.
   # If there's already a kvm module installed, we don't mess with it
   
   udevadm control --reload >/dev/null 2>&1 || : 
   
   sh /etc/sysconfig/modules/kvm.modules &> /dev/null || :
       udevadm trigger --subsystem-match=misc --sysname-match=kvm --action=add || :
   ```

从整个过程来说，并没有主动添加acl权限的动作，又没办法重启调试（重启现象消失），所以怀疑是硬件与内核的兼容性不好，采用规避性方案：

根据分析，重新加载kvm module 

```sh
modprobe -r kvm_intel
modprobe kvm_intel
```

或者手动执行setfacl将acl规则去掉，当然也可以添加组权限

```
setfacl -b /dev/kvm
or
setfacl -m g::rw /dev/kvm
```

重新执行virsh capabilities可以看到\<domain type='kvm'/>（不需要重启libvirt，原因是获取cache时，isValid会校验/dev/kvm的属性变化，导致校验失败，会触发重新生成cache）

这里其实只是暂时规避方案，因为发生的概率特别小，不容易复现，真正具体的原因还需要研究内核代码以及服务器厂商配合定位。



参考文档

[基于QMP实现对qemu虚拟机进行交互](https://blog.csdn.net/loveychent/article/details/89811758)

