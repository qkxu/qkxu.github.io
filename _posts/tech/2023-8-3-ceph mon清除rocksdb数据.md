---
layout: post
title: ceph mon清除rocksdb数据
category: ceph
---

# ceph mon清除rocksdb数据

导出ceph mon的数据可以看出，占用比较大的prefix为logm以及paxos，rocksdb本身并没有主动删除数据（ttl=0），随着数据的增加，rocksdb空间也逐渐增大，那么ceph mon如何清理数据？

### PaxosService的数据清理

每个PaxosService服务删除rocksdb中的值都在maybe_trim函数中，这个在mon执行的时候会被tick调用

而tick会在monitor初始化时会周期性执行

```
int Monitor::init()
{
  dout(2) << "init" << dendl;
  std::lock_guard l(lock);

  finisher.start();

  // start ticker
  timer.init();
  new_tick();
  ....
  }
void Monitor::new_tick()
{
  timer.add_event_after(g_conf()->mon_tick_interval, new C_MonContext(this, [this](int) {
	tick();
      }));
}
void Monitor::tick()
{
  ...
  for (auto& svc : paxos_service) {
    svc->tick();
    svc->maybe_trim();
  }
  ...
}
```

获取要删除的边界版本，然后根据配置，达到删除条件时就会触发写入删除，由于删除（也是写入）是在leader上进行，因为需要通过paxos提案让其他poen也进行进行删除

```c++
void PaxosService::maybe_trim()
{
  if (!is_writeable())
    return;
  //获取删除的边界，例如log的mon_max_log_epochs为500
  version_t trim_to = get_trim_to();
  if (trim_to < get_first_committed())
    return;

  version_t to_remove = trim_to - get_first_committed();
  const version_t trim_min = g_conf().get_val<version_t>("paxos_service_trim_min");
  //当log的跨度版本大于500时还不会立即触发，因为paxos_service_trim_min为250，因此大于750时才会触发
  if (trim_min > 0 &&
      to_remove < trim_min) {
    dout(10) << __func__ << " trim_to " << trim_to << " would only trim " << to_remove
	     << " < paxos_service_trim_min " << trim_min << dendl;
    return;
  }

  to_remove = [to_remove, this] {
    const version_t trim_max = g_conf().get_val<version_t>("paxos_service_trim_max");
    if (trim_max == 0 || to_remove < trim_max) {
      return to_remove;
    }
    ...
  trim_to = get_first_committed() + to_remove;

  dout(10) << __func__ << " trimming to " << trim_to << ", " << to_remove << " states" << dendl;
  MonitorDBStore::TransactionRef t = paxos->get_pending_transaction();
  //写入删除标记
  trim(t, get_first_committed(), trim_to);
  //更新first commit值
  put_first_committed(t, trim_to);
  cached_first_committed = trim_to;

  // let the service add any extra stuff
  encode_trim_extra(t, trim_to);
  //发起paxos协商
  paxos->trigger_propose();
}
```

而言把debug日志级别提高则能看到相关的日志

```
2023-07-27 16:11:42.674 7f09a1a50700 10 mon.ceph-1@0(leader).paxosservice(logm 249720..250473) maybe_trim trimming to 249973, 253 states
2023-07-27 16:11:42.674 7f09a1a50700 10 mon.ceph-1@0(leader).paxosservice(logm 249720..250473) trim from 249720 to 249973
2023-07-27 16:11:42.674 7f09a1a50700 20 mon.ceph-1@0(leader).paxosservice(logm 249720..250473) trim 249720
2023-07-27 16:11:42.674 7f09a1a50700 20 mon.ceph-1@0(leader).paxosservice(logm 249720..250473) trim full_249720
2023-07-27 16:11:42.674 7f09a1a50700 20 mon.ceph-1@0(leader).paxosservice(logm 249720..250473) trim 249721
```

通知其他mon提案的内容大致为:

```json
2023-07-28 17:00:58.415 7f13cf83f700 10 mon.ceph-2@1(peon).paxos(paxos updating c 517061..517812) handle_commit on 517813
2023-07-28 17:00:58.415 7f13cf83f700 10 mon.ceph-2@1(peon).paxos(paxos updating c 517061..517813) store_state [517813..517813]
2023-07-28 17:00:58.415 7f13cf83f700 30 mon.ceph-2@1(peon).paxos(paxos updating c 517061..517813) store_state transaction dump:
{
    "ops": [
        {
            "op_num": 0,
            "type": "PUT",
            "prefix": "paxos",
            "key": "last_committed",
            "length": 8
        },
        {
            "op_num": 1,
            "type": "PUT",
            "prefix": "paxos",
            "key": "517813",
            "length": 11392
        },
        ...
        {
            "op_num": 5,
            "type": "ERASE",
            "prefix": "paxos",
            "key": "517061"
        },
        {
            "op_num": 6,
            "type": "ERASE",
            "prefix": "paxos,
            "key": "517062"
        }
        ....
```

### Paxos的数据清理

Paxos删除rocksdb中的值（prefix为paxos）在commit_finish-->finish_round中执行

```c++
void Paxos::finish_round()
{
  dout(10) << __func__ << dendl;
  ceph_assert(mon->is_leader());

  // ok, now go active!
  state = STATE_ACTIVE;

  ...
  //判定是否满足删除条件
  if (should_trim()) {
    trim();
  }

  if (is_active() && pending_proposal) {
  //同样会触发提案，让poen也删除
    propose_pending();
  }
}

void Paxos::trim()
{
  ceph_assert(should_trim());
  version_t end = std::min(get_version() - g_conf()->paxos_min,
		      get_first_committed() + g_conf()->paxos_trim_max);

  if (first_committed >= end)
    return;

  dout(10) << "trim to " << end << " (was " << first_committed << ")" << dendl;

  MonitorDBStore::TransactionRef t = get_pending_transaction();
  //删除多余的paxos
  for (version_t v = first_committed; v < end; ++v) {
    dout(10) << "trim " << v << dendl;
    t->erase(get_name(), v);
  }
  t->put(get_name(), "first_committed", end);
    //根据配置是否强制触发rocskdb的compact
  if (g_conf()->mon_compact_on_trim) {
    dout(10) << " compacting trimmed range" << dendl;
    t->compact_range(get_name(), stringify(first_committed - 1), stringify(end));
  }

  trimming = true;
  queue_pending_finisher(new C_Trimmed(this));
}
```

当然以上的删除只是写入erase标记，当rocksdb执行compact时才是真正的删除

### 异常场景下的rocksdb分析

异常场景下（例如只留一个poen，其余节点下电，过一段时间再上电），因为堆积了大量的消息日志，开始恢复时rocksdb占用的空间比较多，导出数据可以查看到占用的空间的主要是logm以及paxos，其实key的数量并不多，从上述分析可以看到当key过多时会trim掉

**logm / full_xxx** ： 由于配置了mon_log_max_summary，full中的每个channel最大50条，因此不会占用太多空间

**logm / xxx** ：过多的消息堆积导致一次处理的日志条数特别多，这个占用了主要的空间

**paxos / xxx**：log日志的paxos同样占用特别大的空间，因为paxos带有encode后的待处理的消息，paxos中的数据量很大，但是通过kv tool看到的很少，这是因为dump的时候默认不打印出bf的内容

```

    void dump(ceph::Formatter *f, bool dump_val=false) const {
      f->open_object_section("transaction");
      f->open_array_section("ops");
      list<Op>::const_iterator it;
      int op_num = 0;
      for (it = ops.begin(); it != ops.end(); ++it) {
	const Op& op = *it;
	f->open_object_section("op");
	f->dump_int("op_num", op_num++);
	switch (op.type) {
	case OP_PUT:
	  {
	    f->dump_string("type", "PUT");
	    f->dump_string("prefix", op.prefix);
	    f->dump_string("key", op.key);
	    f->dump_unsigned("length", op.bl.length());
	    if (dump_val) {
	      ostringstream os;
	      op.bl.hexdump(os);
	      f->dump_string("bl", os.str());
	    }
	  }
	  break;
```





