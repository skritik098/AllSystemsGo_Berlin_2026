# From fsync to Block I/O: Tracing storage with eBPF

**All Systems Go! 2026 · Berlin · 30 September 2026**
Kritik Sachdeva · IBM

📄 [Slides (PDF)](slides/ebpf-storage-tracing-asg2026.pdf)

## Abstract

"The disk is slow," yet `iostat` shows a device that is never idle and never behind: a queue under one, writes in 0.31 ms, flushes in 0.62 ms. Meanwhile the commit the user is waiting on takes milliseconds.

Device-level tools only measure the last hop of an I/O. This talk follows a request from `write()` / `fsync()` through the page cache, filesystem and journal, writeback, cgroup throttling and the block queue down to the device. It uses eBPF to find out where the time actually goes.

## What `iostat` cannot tell you

- **Who waited.** iostat counts per device, not per process or per cgroup.
- **Where it waited.** `await` starts at issue, so everything before that point goes unmeasured.
- **How bad it got.** The numbers are averages over a window, so the tail latency is missing.

## Why it matters now

For years the disk was obviously the slow part. Compute is now the part we keep adding, and storage is what holds it up. In training runs, GPU utilisation often sits below 50% while the GPUs wait on data.

## The I/O path

```
write() · fsync() → page cache → fs · journal → writeback → cgroup throttle → block queue → device
└──────────── the application waits for all of this ────────────┘               └─ iostat ─┘
```

## eBPF attach points

| Type | What it is | Trade-off |
|------|------------|-----------|
| **tracepoint** | Static events the kernel already exposes, e.g. `block:block_rq_issue` | Stable across kernels and the cheapest option. Start here. |
| **fentry / fexit** | Entry and return of a kernel function via a BPF trampoline | `fexit` gives arguments and return value together. Needs kernel 5.5+ and BTF. |
| **kprobe / kretprobe** | Dynamic probe on any instruction in a function | Works without BTF, but it is a breakpoint trap, so it costs more and depends on kernel internals. |

## Demos

### Demo 1: cold run vs. warm run

🎬 [demos/demo1-cold-vs-warm-cache.mov](demos/demo1-cold-vs-warm-cache.mov)

```bash
# 1. Drop the page cache
sync; echo 3 > /proc/sys/vm/drop_caches

# 2. Cold run: reads hit the disk
dd if=/data/testfile of=/dev/null

# 3. Same read again: the block layer goes silent
#    Observe with: biosnoop · cachestat · blocklat-raw.bt
```

**Watch for:** the block trace goes quiet on the second run because the page cache serves everything.

### Demo 2: throttled on purpose

🎬 [demos/demo2-cgroup-io-throttle.mov](demos/demo2-cgroup-io-throttle.mov)

```bash
# 1. Cap the cgroup's read IOPS
echo "259:0 riops=200" > io.max

# 2. Run the same dd inside that cgroup
dd if=/dev/zero of=/data/out

# 3. The device is still fast, but the wait moved up the stack
#    Observe with: biolatency -m · blocklat-cgroup.bt
```

**Watch for:** the latency histogram shifts even though the device stays fast.

## Requirements

- Linux kernel with BPF support (5.5+ with BTF for fentry/fexit)
- [bpftrace](https://github.com/bpftrace/bpftrace)
- [BCC tools](https://github.com/iovisor/bcc) (`biosnoop`, `biolatency`, `cachestat`)
- cgroup v2 for the `io.max` throttling demo
- Root privileges

## Takeaway

> Trace the path, correlate the wait, then go look at the device.

- **Who waited:** this process, this cgroup, not "the host".
- **Where it waited:** page cache, journal, block queue or cgroup throttle.
- **How long:** per layer and per request, not one device average.
