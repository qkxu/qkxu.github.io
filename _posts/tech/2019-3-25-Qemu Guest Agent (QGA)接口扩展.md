---
layout: post
title: Qemu Guest Agent (QGA)接口扩展
category: VMM
description: Qemu Guest Agent (QGA)接口扩展
---
#  Qemu Guest Agent (QGA)接口扩展

### 1.**制作rpm包**

虚拟机系统采用的centos7，因为采用rpm包的方式安装qemu-guest-agent比较方便，因此在扩展QGA之前要了解熟悉下如何制作rpm包。

1. 从官网上下载qemu-guest-agent的src.rpm包，以qemu-guest-agent-2.12.0为例：

   ```sh
   wget http://www.rpmfind.net/linux/centos/7.6.1810/os/x86_64/Packages/qemu-guest-agent-2.12.0-2.el7.x86_64.rpm
   ```

2. 安装src.rpm包，即在用户目录下生成相关的rpmbuild目录

   ```sh
   rpm -i qemu-guest-agent-2.12.0-2.el7.src.rpm
   ```

3. 安装后可以看到用户目录下的相关文件

   ```
   cd ~
   ll rpmbuild/
   total 0
   drwxr-xr-x 2 root root 274 Mar 26 08:16 SOURCES
   drwxr-xr-x 2 root root  35 Mar 26 08:16 SPECS
   ```


4. 编译前准备，即解压源码以及建立相应的目录，如果需要相应的rpm包，用yum安装所需依赖包。

   ```sh
    rpmbuild -bp ~/rpmbuild/SPECS/qemu-guest-agent.spec
   ```

5. 修改源代码用于生成patch，由于patch是基于git diff生成的，因此建议把源代码复制单独的目录，并建立git库，主要是用于修改源代码后，使用git diff生成patch。

   ```sh
   mkdir ~/tmp
   cp -r ~/rpmbuild/BUILD/qemu-2.12.0/ ~/tmp/
   cd ~/tmp/qemu-2.12.0/
   git init
   git add .
   git commit -m "first commit"
   ```

6. 修改相关的文件，即扩展QGA，主要是修改qapi-schema.json与commands-posix.c，具体实现见段二。

   ```
   vim ~/tmp/qemu-2.12.0/qga/qapi-schema.json
   vim ~/tmp/qemu-2.12.0/qga/commands-posix.c
   ```

7. 生成patch文件到rpmbuild/SOURCES目录下

   ```sh
   cd ~/tmp/qemu-2.12.0/
   git diff > ~/rpmbuild/SOURCES/qemu-ga-new-cmd.patch
   ```

8. 修改spec文件，用于生成新的rpm包

   ```
   vim ~/rpmbuild/SPECS/qemu-guest-agent.spec
   ```

   1. 找到Patchxxx这样的字符串，xxx表示数字，在其中xxx最大一行下面添加：

      Patchxxy: patch_file

      假设xxx是最大的数字，则xxy=xxx+1。

      例如本例中：

      ```
      # For bz#1567041 - qemu-guest-agent does not parse PCI bridge links in "build_guest_fsinfo_for_real_device" (q35)
      Patch2: qemuga-qga-fix-driver-leak-in-guest-get-fsinfo.patch
      Patch3: qemu-ga-new-cmd.patch
      ```

   2. 找到%patchxxx -pn这样的字符串，xxx，n表示数字。

      其中xxx表示在上步骤中的patch号，n表示忽略patch的第n层目录，例如：

      %patch -p1 使用前面定义的Patch补丁进行，-p1是忽略patch的第一层目录，具体可以查看git diff生成patch中的内容即可理解。

      例如本例中：

      ```
      %prep
      %setup -q -n qemu-%{version}
      %patch1 -p1
      %patch2 -p1
      %patch3 -p1
      ```



9. 生成新的rpm包

   ```
   rpmbuild -bb ~/rpmbuild/SPECS/qemu-guest-agent.spec
   ```

10. 然后卸载原有的包，安装新包即可。




### 2.**扩展QGA接口**

定制qga命令本质是基于QAPI框架实现QMP命令。QGA代码框架对于开发者来说，扩展较为容易，简单命令只需要在qapi-schema.json实现头文件说明以及commands-posix.c中实现具体的功能即可。

以设定watchdog的timeout为例来说明：

1. 修改qga/qapi-schema.json，增加函数声明

   ```
   ##
   # @guest-get-osinfo:
   #
   # Retrieve guest operating system information
   #
   # Returns: @GuestOSInfo
   #
   # Since: 2.10
   ##
   { 'command': 'guest-get-osinfo',
     'returns': 'GuestOSInfo' }
   # 以下是新增接口信息
   # @guest-set-watchdog-timeout:
   #
   # Set watchdog time for guest operating system information
   #
   #
   # Since: 2.10
   ##
   { 'command': 'guest-set-watchdog-timeout','data': { 'timeout': 'int' }}
   ```
   说明：

   1. 关于注释，注释的内容很重要，按照要求的格式填写，后续用于生成函数说明，@后面跟函数名字

   2. command指命令名称，即在virsh环境中执行的命令，用中划线分割。

      如果有参数，则使用data，如果多个参数用","分割，例如：

      ```
      'data':    { 'path': 'str', '*arg': ['str'], }
      ```

      如果有返回值，则用returns，例如：
      ```
      'returns': 'GuestOSInfo'
      ```
      需要注意的是高版本中不建议直接返回内置类型（int，string等），用QGA官方说法是为了扩展性，后续版本在返回数据时会附带一些信息，字典类型数据便于添加和扩展。

      如果直接返回内置类型会编译报错，例如：
      ```
      'returns': 'int'
      ```
      如果想返回内置类型，则必须把该函数添加到白名单中，即在qapi-schema.json开头中添加：
      ```
      # Whitelists to permit QAPI rule violations; think twice before you
      # add to them!
      { 'pragma': {
          # Commands allowed to return a non-dictionary:
          'returns-whitelist': [
              'guest-file-open',
              'guest-fsfreeze-freeze',
              'guest-fsfreeze-freeze-list',
              'guest-fsfreeze-status',
              'guest-fsfreeze-thaw',
              'guest-get-time',
              'guest-set-vcpus',
              'guest-sync',
              'guest-sync-delimited',
              'guest-get-cpu-usage-rate' ] } }
          
      ```

      因此推荐使用自定义结构体进行包装，例如：

      ```
      { 'command': 'guest-get-cpu-usage-rate',
        'returns': 'number' }
      
      ```

   3. QGA 中内置了多种数据类型，与C语言中的数据类型对应，并支持自定义结构体，在qapi-schema.json有诸多样例，例如：

      ```
      { 'struct': 'GuestHostName',
        'data':   { 'host-name': 'str' } }
      ```

   4. QGA为了兼容性，对于qapi-schema.json定义的相关类型会进行转化，例如上例子中的timeoout为int类型，编译后为int64_t timeout，

      可以参考commands-posix.c中的其他函数定义，或者查看编译后的头文件（~/rpmbuild/BUILD/qemu-2.12.0/qga/qapi-generated/qga-qapi-commands.h）

2. 修改qga/qapi-schema.json，实现声明的函数。

   ```
   void qmp_guest_set_watchdog_timeout(int64_t timeout, Error **errp)
   {
       char msg[1024] = {'\0'};
       int ret;
       sprintf(msg,"echo \"options  i6300esb heartbeat=%" PRId64 "\" > /etc/modprobe.d/watchdog.conf", timeout);
       ret=system(msg);
       ret=ret;
       return;
   }
   ```
   说明：

   1. 关于形参类型上文已经说明，如果定义不对应，编译将报错，最后一个参数为Error **errp，不可省略，用于返回给libvirt调用报错的信息，

      可以参考其他函数实现返回错误信息。

   2. 关于函数名称，要加上qmp_，这也时编译头文件时自动加上的。命令中划线修改为下划线，遵守QGA统一命名转化规范。

   3. ret=ret;主要是为了编译通过，定义的变量要使用。如果直接使用system(msg)不定义ret，则编译报system(msg)返回值未引用错误。

   4. 对于写文件操作的命令，关闭SELinux，否则会报权限错误。

3. 修改后用git diff将补丁生成，重新制作新的rpm包即可。



参考文档：

[定制qga(qemu-guest-agent)命令](https://blog.csdn.net/dwdwdw2/article/details/79313684)

[How to use the QAPI code generator](<https://github.com/coreos/qemu/blob/master/docs/qapi-code-gen.txt>)