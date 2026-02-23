---
title: "Memory Reclamation Causing Slow OverlayFS Mount"
draft: false
date: 2024-02-04
tags:
  - Linux
  - perf
---

## The Problem

In our production environment, we observed that OverlayFS mount times during container creation would occasionally spike from milliseconds to **over 10 seconds** — roughly 0.2% of the time. Our optimized secure container creation pipeline normally completes in about 100ms, so a 10-second stall on a single `mount` call is completely unacceptable.

## Investigation

### Correlating with Memory Pressure

The first clue came from correlating the latency spikes with system-level metrics. The mount jitter only appeared when **memory reclamation** was actively running.

Using `sar -B`, we could see that `pgscan` (page scan rate) was extremely high during the spikes. The root cause: our system runs batch workloads that trigger massive bursts of container creation at scheduled times. These bursts push system memory below the low watermark, forcing the kernel into aggressive memory reclamation via `kswapd`.

To reproduce this in a test environment, we simply exhausted available memory using `dd` to force the system into reclamation:

```bash
dd if=/dev/zero of=big003.txt bs=1MB count=1000000
```

With memory pressure applied, the mount latency spikes became reliably reproducible.

### Tracing the Slow Kernel Path

With a reproducible case in hand, we used the [funcslower](https://github.com/iovisor/bcc/blob/master/tools/funcslower.py) tool from BCC to walk down the kernel call stack and identify which function was actually stalling:

```c
// https://elixir.bootlin.com/linux/v5.4.251/source/fs/overlayfs/super.c#L1749
ovl_mount
 => mount_nodev
  => sget
    => alloc_super
      => prealloc_shrinker
        => prealloc_memcg_shrinker
          => down_write(&shrinker_rwsem);  // High latency here
```

```bash
./funcslower -P down_write 10000
```

The bottleneck: acquiring the **write lock** on `shrinker_rwsem`. During `mount` (and `umount`), the kernel registers (or unregisters) a shrinker callback, which requires taking this write lock. Under normal conditions, this is nearly instantaneous. But during memory reclamation, something else is holding the lock — and holding it for a long time.

### Identifying the Lock Holder with eBPF

There's a subtlety that makes this tricky to debug: the memory reclamation path acquires `shrinker_rwsem` using `down_read_trylock` rather than a blocking `down_read`. This means standard lock contention tools won't easily catch it. Instead, we need to measure how long the **read lock is held** by tracing both `down_read_trylock` (acquire) and `up_read` (release).

Here's the `down_read_trylock` signature for reference:

```c
// https://elixir.bootlin.com/linux/v5.4.251/source/kernel/locking/rwsem.c#L1544
int down_read_trylock(struct rw_semaphore *sem)
{
	int ret = __down_read_trylock(sem);
	if (ret == 1)
		rwsem_acquire_read(&sem->dep_map, 0, 1, _RET_IP_);
	return ret;
}
EXPORT_SYMBOL(down_read_trylock);
```

The first argument is the semaphore address. We can use bpftrace's `kaddr()` function to resolve the address of `shrinker_rwsem` and filter only the events we care about:

```bpftrace
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

The output confirmed our suspicion — **`kswapd` was holding the read lock for hundreds of milliseconds** at a time:

```text
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

## Root Cause

During memory reclamation, `kswapd` acquires the **read lock** of `shrinker_rwsem` and holds it for an extended time while scanning and reclaiming pages. Meanwhile, `mount` needs the **write lock** on the same semaphore to register its shrinker callback. Since a write lock cannot be acquired while any read lock is held, the mount operation stalls until `kswapd` finishes its reclamation pass.

## Fixes

### 1. Kernel Fix

A [community patch series](https://www.mail-archive.com/linux-erofs@lists.ozlabs.org/msg08935.html) removes the `shrinker_rwsem` read lock from the `shrink_node` path entirely, replacing it with a reference-counting scheme and a `shrinker_mutex`. With this change, `kswapd`'s reclamation no longer blocks the write lock acquisition during `mount`.

This fix was [merged into mainline Linux 6.7](https://lore.kernel.org/all/20231101145447.60320c9044e7db4dba2d93e3@linux-foundation.org/) in October 2023.

### 2. Application-Level Workaround

As an alternative for systems that cannot upgrade kernels, we bypass the host-side `mount` entirely by injecting all overlay layer files directly into the VM via `virtiofs`. This avoids the problematic `mount` call altogether.
