---
layout: post
title: libvirt中blockcopy使用
category: VMM
description: libvirt中blockcopy使用
---
#  libvirt中blockcopy使用

Libvirt支持blockcopy操作，将虚拟机某个盘同步到主机文件，使得主机文件与虚拟机磁盘实现动态复制功能。虚拟机热迁移场景下的存储热迁移也是基于此原理，在目的节点与源节点的块设备同步后，动态切换到目的节点。

### 实验环境

宿主机系统（host）：CentOS Linux release 7.4.1708 (Core)

libvirt：3.2.0

qemu: 2.9.0

### blockcopy过程

1. copy phase

   这个阶段主要是保证虚拟机磁盘与主机目标同步，时间较长，取决于磁盘大小，在copy阶段中对虚拟机的修改写入，也会被记为脏页，一并同步到目标文件。

   ```sh
   virsh # help blockcopy
     NAME
       blockcopy - Start a block copy operation.
   
     ...
   
     DESCRIPTION
       Copy a disk backing image chain to dest.
   
     OPTIONS
       [--domain] <string>  domain name, id or uuid
       [--path] <string>  fully-qualified path of source disk
       --dest <string>  path of the copy to create
       --bandwidth <number>  bandwidth limit in MiB/s
       --shallow        make the copy share a backing chain
       --reuse-external  reuse existing destination 使用已有文件
       --blockdev       copy destination is block device instead of regular file
       --wait           wait for job to reach mirroring phase
       --verbose        with --wait, display the progress
       --timeout <number>  implies --wait, abort if copy exceeds timeout (in seconds)
       --pivot          implies --wait, pivot when mirroring starts
                        当mirroring starts时切换到目的卷
       --finish         implies --wait, quit when mirroring starts
                        当mirroring starts时结束mirroring任务
       --async          with --wait, don't wait for cancel to finish
       --xml <string>   filename containing XML description of the copy destination
       --format <string>  format of the destination file
       --granularity <number>  power-of-two granularity to use during the copy
       --buf-size <number>  maximum amount of in-flight data during the copy
       --bytes          the bandwidth limit is in bytes/s rather than MiB/s
       --transient-job  the copy job is not persisted if VM is turned off
                        对于持久化的虚拟机必须加上此参数，当vm关机时，copy job无效。
   
   ```

   命令行示例：

   ```sh
   virsh blockcopy drive_mirror vda --dest /home/xqk/vm_work/drive_mirror/test.qcow2 --wait --verbose  --transient-job --reuse-external
   
   ```

   在执行第一阶段时：

   ```
   Block Copy: [75 %]
   ```

   或者通过命令查询

   ```sh
   virsh qemu-monitor-command --hmp drive_mirror  'info  block-jobs'
   ```

   执行完毕时：

   ```
   Block Copy: [100%]
   ```

   对虚拟机event监听：

   ```sh
   virsh # event 4 –all
   event 'block-job' for domain drive_mirror:
   Block Copy for /home/xqk/vm_work/drive_mirror/drive_mirror.qcow2 ready
   ```

   

2. mirroring phase

   当copy结束后进入mirroring阶段，这个阶段对磁盘的写入会持续写入目标文件。

   ```xml
   dumpxml drive_mirror
   <mirror type='file' file='/home/xqk/vm_work/drive_mirror/test.qcow2' format='qcow2' job='copy' ready='yes'>
           <format type='qcow2'/>
           <source file='/home/xqk/vm_work/drive_mirror/test.qcow2'/>
   </mirror>
   ```

   当虚拟机磁盘已经执行了blockcopy操作后，不能再对该磁盘再次执行blockcopy，除非中断当前的blcok任务。

   ```sh
   virsh qemu-monitor-command --hmp drive_mirror  'block_job_complete drive-virtio-disk0'
   virsh qemu-monitor-command --hmp drive_mirror  'block_job_cancel drive-virtio-disk0'
   ```

   再次查询该磁盘的blockjob则为空。

   ```
   virsh qemu-monitor-command --hmp drive_mirror  'info  block-jobs'
   ```

   

libvirt对于blockcopy进行了封装，但是只支持本地文件备份，加了些查询参数，方便用户查看进度，并对本地文件进行校验之类的。在底层是调用则是调用qemu的**drive_mirror**命令以及查询**info  block-jobs**命令。

### libvirt对于blockcopy实现

解析virsh命令

```c
static bool
cmdBlockCopy(vshControl *ctl, const vshCmd *cmd)
{
    virDomainPtr dom = NULL;
    ...
    //对于传入的disk，如果不是xml类型，会拼装成xml，传入virDomainBlockCopy
    if (!xmlstr) {
            virBuffer buf = VIR_BUFFER_INITIALIZER;
            virBufferAsprintf(&buf, "<disk type='%s'>\n",
                              blockdev ? "block" : "file");
            virBufferAdjustIndent(&buf, 2);
            virBufferAsprintf(&buf, "<source %s", blockdev ? "dev" : "file");
            virBufferEscapeString(&buf, "='%s'/>\n", dest);
            virBufferEscapeString(&buf, "<driver type='%s'/>\n", format);
            virBufferAdjustIndent(&buf, -2);
            virBufferAddLit(&buf, "</disk>\n");
            if (virBufferCheckError(&buf) < 0)
                goto cleanup;
            xmlstr = virBufferContentAndReset(&buf);
        }
    if (virDomainBlockCopy(dom, path, xmlstr, params, nparams, flags) < 0)
            goto cleanup;
```

最后调用服务端中的函数

```
.domainBlockCopy = qemuDomainBlockCopy, 
```

即：

```c
static int
qemuDomainBlockCopyCommon(virDomainObjPtr vm,
                          virConnectPtr conn,
                          const char *path,
                          virStorageSourcePtr mirror,
                          unsigned long long bandwidth,
                          unsigned int granularity,
                          unsigned long long buf_size,
                          unsigned int flags,
                          bool keepParentLabel)
{
    ...
    /* Prepare the destination file. 只支持本地文件，以/根目录开始的文件*/
    /* XXX Allow non-file mirror destinations */
    if (!virStorageSourceIsLocalStorage(mirror)) {
        virReportError(VIR_ERR_ARGUMENT_UNSUPPORTED, "%s",
                       _("non-file destination not supported yet"));
        goto endjob;
    }
    ...
        /* pre-create the image file （对于新创建文件）*/
    if (!(flags & VIR_DOMAIN_BLOCK_COPY_REUSE_EXT)) {
        int fd = qemuOpenFile(driver, vm, mirror->path,
                              O_WRONLY | O_TRUNC | O_CREAT,
                              &need_unlink, NULL);
        if (fd < 0)
            goto endjob;
        VIR_FORCE_CLOSE(fd);
    ...
    /*调用qemu driver-mirror*/
    ret = qemuMonitorDriveMirror(priv->mon, device, mirror->path, format,
                                 bandwidth, granularity, buf_size, flags);
    virDomainAuditDisk(vm, NULL, mirror, "mirror", ret >= 0);
    }
```

### qemu中driver-mirror实现

无论是qmp或者hmp，最终都会调用qmp函数处理

```c
void qmp_drive_mirror(DriveMirror *arg, Error **errp)
{
    ...
    /* Mirroring takes care of copy-on-write using the source's backing
     * file.
     */
    /*在open函数中，会解析filename，根据filename获取后端driver中协议以及带有的参数
     *bdrv_open->bdrv_open_inherit
     *bdrv_open_inherit中会调用bdrv_fill_options获取driver以及所带参数
     */
    target_bs = bdrv_open(arg->target, NULL, options,
                          flags | BDRV_O_NO_BACKING, errp);
    if (!target_bs) {
        goto out;
    }
    ...
    /*调用mirror_common具体实现*/
    blockdev_mirror_common(...)
}
```

进入block_job

```c
blockdev_mirror_common->mirror_start->mirror_start_job;
/*注意此时传入的job_driver为mirror_job_driver*/
mirror_start_job(job_id, bs, BLOCK_JOB_DEFAULT, target, replaces,
                     speed, granularity, buf_size, backing_mode,
                     on_source_error, on_target_error, unmap, NULL, NULL, errp,
                     &mirror_job_driver, is_none_mode, base, false,
                     filter_node_name);
static void mirror_start_job(...)
{
    ...
    /* In the case of active commit, add dummy driver to provide consistent
     * reads on the top, while disabling it in the intermediate nodes, and make
     * the backing chain writable. */
    /*bdrv_mirror_top这个驱动注册为：
     * Dummy node that provides consistent read to its users without requiring it
     * from its backing file and that allows writes on the backing file chain.
     static BlockDriver bdrv_mirror_top = {
    .format_name                = "mirror_top",
    .bdrv_co_preadv             = bdrv_mirror_top_preadv,
    .bdrv_co_pwritev            = bdrv_mirror_top_pwritev,
    .bdrv_co_pwrite_zeroes      = bdrv_mirror_top_pwrite_zeroes,
    .bdrv_co_pdiscard           = bdrv_mirror_top_pdiscard,
    .bdrv_co_flush              = bdrv_mirror_top_flush,
    .bdrv_co_get_block_status   = bdrv_mirror_top_get_block_status,
    .bdrv_refresh_filename      = bdrv_mirror_top_refresh_filename,
    .bdrv_close                 = bdrv_mirror_top_close,
    .bdrv_child_perm            = bdrv_mirror_top_child_perm,
    };
    */
    mirror_top_bs = bdrv_new_open_driver(&bdrv_mirror_top, filter_node_name,
                                         BDRV_O_RDWR, errp);
    ...
    /*把bs的aio赋值于bdrv_mirror_top，当源bs有写入动作时，会同时触发bdrv_mirror_top中的
     *bdrv_co_pwritev函数(这个是驱动规范，所有的驱动注册都会有上述的函数子成员），
     *相当于一份数据写入两份驱动来实现数据同步*/
    bdrv_set_aio_context(mirror_top_bs, bdrv_get_aio_context(bs));
    ...
    /*创建job对象，将参数driver作为job的dirver*/
    s = block_job_create(job_id, driver, mirror_top_bs,
                         BLK_PERM_CONSISTENT_READ,
                         BLK_PERM_CONSISTENT_READ | BLK_PERM_WRITE_UNCHANGED |
                         BLK_PERM_WRITE | BLK_PERM_GRAPH_MOD, speed,
                         creation_flags, cb, opaque, errp);
    ...
    /*初始化脏页对象*/
    s->dirty_bitmap = bdrv_create_dirty_bitmap(bs, granularity, NULL, errp);
    /* Required permissions are already taken with blk_new() */
    /*添加job*/
    block_job_add_bdrv(&s->common, "target", target, 0, BLK_PERM_ALL,
                       &error_abort);
    ...
    /*启动job*/
    block_job_start(&s->common);
```

启动block_job

```c
void block_job_start(BlockJob *job)
{
    assert(job && !block_job_started(job) && job->paused &&
           job->driver && job->driver->start);
    /*创建回调，执行bdrv_coroutine_enter后会触发此回调*/
    job->co = qemu_coroutine_create(block_job_co_entry, job);
    job->pause_count--;
    job->busy = true;
    job->paused = false;
    bdrv_coroutine_enter(blk_bs(job->blk), job->co);
}
static void coroutine_fn block_job_co_entry(void *opaque)
{
    BlockJob *job = opaque;

    assert(job && job->driver && job->driver->start);
    block_job_pause_point(job);
    /*触发注册driver的start*/
    job->driver->start(job);
}
/*job的driver是在创建的时候由参数driver带入的，而driver参数是由
 *mirror_start_job传入，即mirror_job_driver*/
static const BlockJobDriver mirror_job_driver = {
    .instance_size          = sizeof(MirrorBlockJob),
    .job_type               = BLOCK_JOB_TYPE_MIRROR,
    .set_speed              = mirror_set_speed,
    .start                  = mirror_run,
    .complete               = mirror_complete,
    .pause                  = mirror_pause,
    .attached_aio_context   = mirror_attached_aio_context,
    .drain                  = mirror_drain,
};
/*即执行mirror_run函数*/
```

driver_mirror的实现

```c
static void coroutine_fn mirror_run(void *opaque)
{
    MirrorBlockJob *s = opaque;
    ...
    s->last_pause_ns = qemu_clock_get_ns(QEMU_CLOCK_REALTIME);
    if (!s->is_none_mode) {
        /*第一次初始化dirty，所以copy阶段比较长，要copy较多的脏页*/
        ret = mirror_dirty_init(s);
        if (ret < 0 || block_job_is_cancelled(&s->common)) {
            goto immediate_exit;
        }
    }
    ...
    /*在这个循环中不断完成dirty的copy*/
    for (;;) {
        uint64_t delay_ns = 0;
        int64_t cnt, delta;
        bool should_complete;

        if (s->ret < 0) {
            ret = s->ret;
            goto immediate_exit;
        }
        ...
        if (delta < SLICE_TIME &&
            s->common.iostatus == BLOCK_DEVICE_IO_STATUS_OK) {
            if (s->in_flight >= MAX_IN_FLIGHT || s->buf_free_count == 0 ||
                (cnt == 0 && s->in_flight > 0)) {
                trace_mirror_yield(s, cnt, s->buf_free_count, s->in_flight);
                mirror_wait_for_io(s);
                continue;
            } else if (cnt != 0) {
                /*脏页数目不为0时，继续迭代copy*/
                delay_ns = mirror_iteration(s);
            }
        }

        should_complete = false;
        if (s->in_flight == 0 && cnt == 0) {
            trace_mirror_before_flush(s);
            if (!s->synced) {
                if (mirror_flush(s) < 0) {
                    /* Go check s->ret.  */
                    continue;
                }
                /* We're out of the streaming phase.  From now on, if the job
                 * is cancelled we will actually complete all pending I/O and
                 * report completion.  This way, block-job-cancel will leave
                 * the target in a consistent state.
                 */
                /*s->synced标识第一次copy完毕，此时copy阶段已经完成，向libvirt发送
                 *block job ready 函数，后续进入mirroring阶段
                 */
                block_job_event_ready(&s->common);
                s->synced = true;
            }

            /*当手动调用block_job_complete或者block_job_cancel时
             *s->should_complete为true
             */
            should_complete = s->should_complete ||
                block_job_is_cancelled(&s->common);
            cnt = bdrv_get_dirty_count(s->dirty_bitmap);
        }

```

dirty_bitmap的copy

```
mirror_iteration
  --》case MIRROR_METHOD_COPY:
            io_sectors = mirror_do_read(s, sector_num, io_sectors);
			  --》blk_aio_preadv(source, sector_num * BDRV_SECTOR_SIZE, &op->qiov, 0,
                   mirror_read_complete, op);
				     --》mirror_read_complete
					   --》blk_aio_pwritev
					     --》mirror_read_complete
						   --》mirror_write_complete
						   
将脏页写入
```

现在看到drive_mirror在循环中不断读取dirty_bitmap，并写入文件，当一次读取完毕时会触发event，表明此时原卷与目的卷已经同步完毕，进入mirroring阶段，后续在循环中继续读取dirty_bitmap并写入文件，直到用户手动调用block_job_complete或者block_job_cancel来结束任务。

那么就剩一个问题了dirty_bitmap是怎样被设置的？

bdrv_mirror_top这个驱动已经注册过了，当原卷有数据写入时，会同时触发bdrv_mirror_top中的bdrv_co_pwritev，大致过程：

```
bdrv_mirror_top.bdrv_co_pwritev
                --》bdrv_mirror_top_pwritev
                  --》bdrv_co_pwritev
				    --》bdrv_aligned_pwritev
					  --》bdrv_set_dirty
```

### 非主机文件的blcokcopy

libvirt只支持根目录文件，对于非本地文件暂且不支持，解决方法：

- 可以通过网络文件系统例如NFS，挂载在到本地。

- 直接使用hmp或者qmp透传到qemu，例如，

  虚拟机热迁移中，本地磁盘热迁移调用的nbd协议

  ```sh
  {"execute":"drive-mirror","arguments":{"device":"drive-virtio-disk0","target":"nbd:xuh-208:49153:exportname=drive-virtio-disk0","speed":9223372036853727232,"sync":"full","mode":"existing","format":"raw"}}
  ```

  ceph卷采用rbd协议（开启cephx认证）

  ```sh
  {"execute":"drive-mirror","arguments":{"device":"drive-virtio-disk0","target":"rbd:volumes/volume-4965215d-6d64-4e07-9611-a3c4ff03ce81:id=cinder:auth_supported=cephx","sync":"full","mode":"existing","format":"raw"}}
  ```

  ceph卷采用rbd协议（关闭cephx认证）

  ```sh
  {"execute":"drive-mirror","arguments":{"device":"drive-virtio-disk0","target":"rbd:volumes/volume-4965215d-6d64-4e07-9611-a3c4ff03ce81:id=cinder","sync":"full","mode":"existing","format":"raw"}}
  ```

### python调用

python调用的接口为：

```c
def blockCopy(self, disk, destxml, params=None, flags=0):
        """Copy the guest-visible contents of a disk image to a new file described by destxml """
        ret = libvirtmod.virDomainBlockCopy(self._o, disk, destxml, params, flags)
        if ret == -1: raise libvirtError ('virDomainBlockCopy() failed', dom=self)
        return ret

```

destxm与attach-devcie用的xml一样，存放磁盘的信息，params存放bandwidth等参数信息，flag存放—shallow --reuse-external  --transient-job 等标志位。

```c
/**
 * virDomainBlockCopyFlags:
 *
 * Flags available for virDomainBlockCopy().
 */
typedef enum {
    VIR_DOMAIN_BLOCK_COPY_SHALLOW   = 1 << 0, /* Limit copy to top of source
                                                 backing chain */
    VIR_DOMAIN_BLOCK_COPY_REUSE_EXT = 1 << 1, /* Reuse existing external
                                                 file for a copy */
    VIR_DOMAIN_BLOCK_COPY_TRANSIENT_JOB = 1 << 2, /* Don't force usage of
                                                     recoverable job for the
                                                     copy operation */
} virDomainBlockCopyFlags;
```

flag的取值即根据上述三个标记位来设定，示例代码：

```python
import libvirt
conn = libvirt.open("qemu+tcp://127.0.0.1/system")
dom = conn.lookupByUUIDString("aaba00b2-e2cf-4354-b81f-82e7e3eb775e")
dom.blockCopy("vda","<disk type='network' device='disk'><driver name='qemu' type='raw' cache='writeback' discard='unmap'/><auth username='cinder'><secret type='ceph' uuid='457eb676-33da-42ec-9a8c-9293d545c337'/></auth><source protocol='rbd' name='cinder-volumes/485362c9-53cb-46e5-acdd-3936755f4521'><host name='192.168.204.2' port='6789'/><host name='192.168.204.3' port='6789'/><host name='192.168.204.168' port='6789'/></source></disk>","",7)
```

