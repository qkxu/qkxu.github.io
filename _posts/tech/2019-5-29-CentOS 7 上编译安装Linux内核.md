---
layout: post
title: CentOS 7 上编译安装Linux内核
category: VMM
description: CentOS 7 上编译安装Linux内核
---
#  CentOS 7 上编译安装Linux内核

前段时间碰到一些内核bug，需要回合到目前版本，因此稍微研究下内核编译，都是些基本的操作，供后续自己参考。编译分为两种方式，一种是官网src.rpm编译，另外一种用源代码编译。

centos版本：CentOS Linux release 7.4.1708 (Core)

当前系统内核版本：3.10.0-693.5.2.el7.x86_64

待编译的内核版本：3.10.0-693.21.1.el7.x86_64

编译内核需要配置config文件，最好取相同版本的.config文件做修改（/boot/config-*），以匹配当前的内核编译，避免不必要的错误，例如多余的参数或者少参数。

### 1.**官网src.rpm编译**

rpm包的编译和其他的rpm编译类似，基本没有区别。

1. rpm包编译

   ```
   下载：
   wget http://vault.centos.org/7.4.1708/updates/Source/SPackages/kernel-3.10.0-693.21.1.el7.src.rpm
   安装源码包：
   rpm -ivh kernel-3.10.0-693.21.1.el7.src.rpm
   编译准备：
   rpmbuild -bp /root/rpmbuild/SPECS/kernel.spec
   编译内核（时间比较长，生产与内核相关的rpm包，如果生成src包，则执行-ba）：
   rpmbuild -bb /root/rpmbuild/SPECS/kernel.spec
   ```

   

2. 注意点

   1. 编译准备时，需要相应的软件包，例如：

      ```
      rpmbuild -bp /root/rpmbuild/SPECS/kernel.spec
      error: Failed build dependencies:
      	bc is needed by kernel-3.10.0-693.21.1.el7.x86_64
      	xmlto is needed by kernel-3.10.0-693.21.1.el7.x86_64
      	asciidoc is needed by kernel-3.10.0-693.21.1.el7.x86_64
      	hmaccalc is needed by kernel-3.10.0-693.21.1.el7.x86_64
      	newt-devel is needed by kernel-3.10.0-693.21.1.el7.x86_64
      	perl(ExtUtils::Embed) is needed by kernel-3.10.0-693.21.1.el7.x86_64
      	pesign >= 0.109-4 is needed by kernel-3.10.0-693.21.1.el7.x86_64
      	elfutils-devel is needed by kernel-3.10.0-693.21.1.el7.x86_64
      	binutils-devel is needed by kernel-3.10.0-693.21.1.el7.x86_64
      	bison is needed by kernel-3.10.0-693.21.1.el7.x86_64
      	java-devel is needed by kernel-3.10.0-693.21.1.el7.x86_64
      ```

      安装相应的包即可，其中perl(ExtUtils::Embed)此依赖安装perl-ExtUtils-Embed即可。

      ```
      yum install perl-ExtUtils-Embed
      ```

   2. 执行bp后，在~/rpmbuild/SOURCES中可以看到相关的默认配置文件

      ```sh
      -rw-rw-r--. 1 root root   123782 Mar  6  2018 kernel-3.10.0-ppc64.config
      -rw-rw-r--. 1 root root   124062 Mar  6  2018 kernel-3.10.0-ppc64-debug.config
      -rw-rw-r--. 1 root root   123388 Mar  6  2018 kernel-3.10.0-ppc64le.config
      -rw-rw-r--. 1 root root   123679 Mar  6  2018 kernel-3.10.0-ppc64le-debug.config
      -rw-rw-r--. 1 root root    59564 Mar  6  2018 kernel-3.10.0-s390x.config
      -rw-rw-r--. 1 root root    59170 Mar  6  2018 kernel-3.10.0-s390x-debug.config
      -rw-rw-r--. 1 root root    31654 Mar  6  2018 kernel-3.10.0-s390x-kdump.config
      -rw-rw-r--. 1 root root   140960 Mar  6  2018 kernel-3.10.0-x86_64.config
      -rw-rw-r--. 1 root root   141164 Mar  6  2018 kernel-3.10.0-x86_64-debug.config
      ```

      内核编译所需.config文件的修改不需要通过生成patch来进行修改，上述配置文件用于生成编译内核所需要的配置文件即：

      ```sh
      ~/rpmbuild/BUILD/kernel-3.10.0-693.21.1.el7/linux-3.10.0-693.21.1.el7.x86_64/.config
      ```

      编译时根据不同的平台会选择相应的配置。本文属于x86_64，因此在执行bp或者bb后会根据以下两种文件生成配置，修改内核参数修改以下两个文件即可。
      ```sh
      -rw-rw-r--. 1 root root   140960 Mar  6  2018 kernel-3.10.0-x86_64.config
      -rw-rw-r--. 1 root root   141164 Mar  6  2018 kernel-3.10.0-x86_64-debug.config
      ```

   3. config文件的配置项是有依赖关系的，在执行build生成config时会做检测。

      例如：CONFIG_STRICT_DEVMEM参数是控制使用mmap映射/dev/mem文件功能的选项，依赖于CONFIG_X86_PAT，如果不取消CONFIG_X86_PAT，调用mmap()的时候会出现Invalid Parameter错误。但是CONFIG_X86_PAT又依赖于CONFIG_EXPERT，开启CONFIG_EXPERT需要更改一系列的参数，因此

      在修改config文件时需要注意下依赖关系，这也是推荐使用make menuconfig图形化修改参数的原因，图形化页面会处理依赖。

   4. patch制作后，修改spec文件：

      ```sh
      # empty final patch to facilitate testing of kernel patches
      Patch999999: linux-kernel-test.patch
      Patch1000: debrand-single-cpu.patch
      Patch1001: debrand-rh_taint.patch
      Patch1002: debrand-rh-i686-cpu.patch
   Patch1003: xxx.patch
      ...
   ApplyOptionalPatch linux-kernel-test.patch
      ApplyOptionalPatch debrand-single-cpu.patch
   ApplyOptionalPatch debrand-rh_taint.patch
      ApplyOptionalPatch debrand-rh-i686-cpu.patch
   ApplyOptionalPatch xxx.patch
      ```
   
   5. 编译报错compile不支持

      ```sh
      make -s ARCH=x86_64 V=1 -j4 KCFLAGS= 'WITH_GCOV=  0' bzImage
      arch/x86/Makefile:166: *** CONFIG_RETPOLINE=y, but not supported by the compiler. Toolchain update recommended..  Stop.
      error: Bad exit status from /var/tmp/rpm-tmp.vbZbJO (%build)
      ```
      错误是因为gcc的版本太低，不支持CONFIG_RETPOLINE=y选项，升级gcc即可。
   
   6. CONFIG_STRICT_DEVMEM的解决方式
   
      由于配置项的依赖关系，暂时不做修改CONFIG_EXPERT以及CONFIG_X86_PAT，仅仅修改
   
      ```sh
      # CONFIG_STRICT_DEVMEM is not set
      ```
   
      在编译完内核后，在新内核的启动项中关闭CONFIG_X86_PAT，即在/boot/grub2/grub.cfg中内核启动选项中添加 nopat选项即可。
   
      ```sh
              linux16 /vmlinuz-3.10.0-693.21.1.el7.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb nopat quiet LANG=en_US.UTF-8
              initrd16 /initramfs-3.10.0-693.21.1.el7.x86_64.img
      
      ```
   
   7. 编译后的内核支持启动参数的方式更改默认行为，与编译时更改配置效果一致，支持的功能参数可以参见参考文档，内核启动参数修改后，重启生效。




### 2.源代码编译

从[内核官网](<https://www.kernel.org/>)下载内核的源代码，切换到相应的分支或者tag：

```
git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
```

或者从src.rpm安装包，执行

```
rpmbuild -bp /root/rpmbuild/SPECS/kernel.spec
```

进入目录

```
~/rpmbuild/BUILD/kernel-3.10.0-693.21.1.el7/linux-3.10.0-693.21.1.el7.x86_64
```

1. 准备.config文件

   编译内核需要配置config文件，最好取相同版本的.config文件做修改（/boot/config-*），以匹配当前的内核编译，避免不必要的错误，例如多余的参数或者少参数。.config中的参数含义：

   ```sh
   =y：打到核心里，以后放在vmlinux中
   =m：模块方式，以后就表现为 ko文件
   not set：该功能不启用
   ```

2. 配置内核选项，执行

   ```
    make menuconfig 
   ```

   如果缺少包，安装即可。图形化的config配置并不是显示config中的配置项，而是以更有好的文字说明配置功能，可以按"/"快捷键来搜索config中的配置项，然后输入搜索到的配置项前面的数字即可。例如搜索配置项CONFIG_ASYNC_RAID6_TEST，显示
   
   ```sh
   Symbol: BUILD_DOCSRC [=y]  
       x Type  : boolean  
       x Prompt: Build targets in Documentation/ tree  
       x   Location:  
       x (1) -> Kernel hacking  
       x   Defined at lib/Kconfig.debug:1438  
       x   Depends on: HEADERS_CHECK [=y] 
   ```
   
   输入1即可调转到相应的配置。
   
   General setup --->的子菜单
      Local version - append to kernel release 进入这一项可以写自己编译安装后的内核版本名，为了后面编译出来的包不与当前内核包冲突。


3. 编译

   ```sh 
   make -j 16
   ```

   j参数后面跟的是cpu核数，用几个cpu来编译，编译时间稍微长些，初次编译缺少包，安装即可。

4. 安装module

   ```
   make modules_install
   ```

   默认会安装到/usr/lib/modules目录，当然也可以指定相应的安装目录

   ```sh
   make modules_install INSTALL_MOD_PATH=/home/test/build/
   ```

5. 安装内核

   ```
   make install
   ```

   安装后到/boot目录下查看相应的内核文件

6. 验证新内核

   重启主机在菜单页面选择新安装的内核进入即可。

7. 生成rpm安装包

   在源代码包中直接执行下面命令也可以生成相应的rpm包，用于安装到其他主机。这个时间比较长，因为会成成与内核相关的所有包。

   ```
   make rpm
   ```

   但是从centos下载的3.10.0-693.21.1.el7.x86_64源代码 中执行报错，估计是版本的缺陷。

参考文档：

[CentOS 6/7 上编译安装Linux内核](<https://www.linuxidc.com/Linux/2017-11/148276.htm>)

[Linux 内核引导选项简介](<http://www.jinbuguo.com/kernel/boot_parameters.html>)