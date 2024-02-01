## 现象

生产环境在创建容器时 overlay mount 的时间会抖动到秒级，出现概率0.2%左右。现在我们优化的安全容器创建只要 100ms 左右，但是这个 mount 的抖动会到十几秒，简直不能忍。

## 分析

### 复现条件

分析发现抖动发生的时间点，跟内存回收事件强关联，同样的创建频率，在内存回收的时候 ovl mount 才会发生抖动。
通过 `sar -B` 命令可以看到，pgscan非常高。这是因为我们系统会有一些定时任务，集中在某个时刻会有大量创建，导致系统内存击穿 low 水位，发生剧烈回收。

测试环境通过将内存水位压低，并且通过 dd 施加内存压力，可以复现。

```
dd if=/dev/zero of=big003.txt bs=1MB count=1000000
```

接下来继续分析内存回收是如何影响到 mount 操作的。
### 调用栈

进一步分析内核 mount 调用耗时高的函数
通过 [funcslower 工具](https://github.com/iovisor/bcc/blob/master/tools/funcslower.py)一步步分析调用栈中慢的函数，最终定位是 down_write 的这个调用慢。

```
// https://elixir.bootlin.com/linux/v5.4.251/source/fs/overlayfs/super.c#L1749
ovl_mount
 => mount_nodev
  => sget
    => alloc_super
      => prealloc_shrinker
        => prealloc_memcg_shrinker
          => down_write(&shrinker_rwsem);  // 这里耗时高
```

分析命令
```bash
./funcslower -P down_write 10000
```

mount 或者 umount 在注册 shrinker回调的时候，获取写锁很慢。

mount 或者 umount 是一直在进行的，但是没有内存回收就没问题，所以一定是内存回收占用了锁。

## 捕获实际锁的占用者

通过 ebpf 脚本分析锁的持有时间，这里最坑的是内核用的是 trylock，而不是直接的 lock。

```
#!/usr/bin/env bpftrace

kprobe:down_read_trylock
{
    if (arg0 != kaddr("shrinker_rwsem")) {
        return;
    }
    @start[tid] = nsecs;
}

kretprobe:up_read /@start[tid]/
{
    $cost = (nsecs - @start[tid]) / 1000 / 1000;

    if ($cost > 1) {
        printf("kswapd lock, cost=%dms, comm=%s, ustack %s\n", $cost, comm, kstack(perf, 10));
    }

    delete(@start[tid]);
}
```

**确定是 kswapd 持有读锁时间过长导致**

```
 acquire lock, cost=180ms, comm=kswapd0, kstack 
        ffffffff81061720 kretprobe_trampoline+0
        ffffffff8123fbc8 rmap_walk+72
        ffffffff8123fd2c page_referenced+332
        ffffffff812037ef shrink_page_list+1583
        ffffffff81204a78 shrink_inactive_list+552
        ffffffff812051aa shrink_list+74
        ffffffff81205c10 shrink_lruvec+768
        ffffffff81206178 shrink_node+408
        ffffffff812076d6 balance_pgdat+854
        ffffffff81207b84 kswapd+532
```


## 结论

kswapd 在内存回收时，持有shrinker_rwsem读锁时间过长，影响到了 mount alloc_super 时获取shrinker_rwsem写锁的操作。

## 修复办法

### 1. 内核侧

社区最新的一个 patch，通过引用计数的方式，去除了 shrink_node 时候 shrinker_rwsem 的读锁。并将其完全改成了 shrinker_mutex。按照我们的分析，kswapd 的行为应该就不会影响到 mount 的操作了。

[https://www.mail-archive.com/linux-erofs@lists.ozlabs.org/msg08935.html](https://www.mail-archive.com/linux-erofs@lists.ozlabs.org/msg08935.html)

该 patch 在 linux 6.7 合入主线，非常新，10 月底合入的

[https://lore.kernel.org/all/20231101145447.60320c9044e7db4dba2d93e3@linux-foundation.org/](https://lore.kernel.org/all/20231101145447.60320c9044e7db4dba2d93e3@linux-foundation.org/)

### 2. 应用侧

把 overlay 各层的文件，直接通过 virtiofs 捅进 vm，主机侧完全没有 mount
