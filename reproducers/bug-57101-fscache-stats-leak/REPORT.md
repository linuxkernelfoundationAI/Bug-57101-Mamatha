# Bug 57101 reproduction report — fscache stats memory leak

**Bug:** https://bugzilla.kernel.org/show_bug.cgi?id=57101 — "Observed memory
leak while accessing /proc/fs/fscache/stats"
**Kernel under test:** v3.9-rc8, pre-fix (commit
`ec686c9239b4d472052a271c505d04dae84214cc` NOT applied)
**Result:** Reproduced. `fs/fscache/stats.c` was not modified at any point.

## Root cause (unchanged from upstream report)

`fs/fscache/stats.c`:

```c
static int fscache_stats_open(struct inode *inode, struct file *file)
{
	return single_open(file, fscache_stats_show, NULL);
}

const struct file_operations fscache_stats_fops = {
	.owner		= THIS_MODULE,
	.open		= fscache_stats_open,
	.read		= seq_read,
	.llseek		= seq_lseek,
	.release	= seq_release,   /* BUG: should be single_release */
};
```

`single_open()` allocates a private `struct seq_operations` (32 bytes on
x86_64: 4 function pointers) and stashes it as `seq_file->op`. Only
`single_release()` frees that allocation; plain `seq_release()` only frees
the `seq_file` itself. Every `open()`+`close()` of `/proc/fs/fscache/stats`
therefore leaks 32 bytes.

## What was built

- Toolchain: `kernel-builder` Docker container, ubuntu:14.04, gcc 4.8.4.
  One gap found (`libelf-dev` missing) and installed; no source changes
  needed. Full log in `build-fixes.md`.
- Config: `make defconfig` + `./scripts/config --enable` for
  `CONFIG_DEBUG_KERNEL`, `CONFIG_DEBUG_KMEMLEAK`, `CONFIG_FRAME_POINTER`,
  `CONFIG_DEBUG_FS`, `CONFIG_FSCACHE`, `CONFIG_FSCACHE_STATS` + `make
  olddefconfig`.
- `make -j6 bzImage modules` → exit 0, zero errors on first attempt
  (`proof/build.log`).
- Booted on the host with plain `qemu-system-x86_64` (no `-s -S`), a custom
  busybox initramfs, `console=ttyS0 kmemleak=on`, captured to
  `proof/boot-output.log`. Confirmed `Linux version 3.9.0-rc8` banner and
  `.release = seq_release` still present in the built `vmlinux` before boot.
- Guest sequence: mount debugfs → `cat /proc/fs/fscache/stats` × 4 →
  `echo scan > /sys/kernel/debug/kmemleak` → `cat
  /sys/kernel/debug/kmemleak` → `proof/kmemleak-output.log`.

## Outcome

kmemleak reported **exactly 4 unreferenced objects, 32 bytes each** — one
per `cat` invocation, comm `busybox`, ages within 30ms of each other —
matching the expected 1-leak-per-open behavior exactly.

## Call chain comparison

The original Bugzilla page (bugzilla.kernel.org) is currently gated behind
an anti-bot challenge (Anubis) that blocks automated fetches, so the literal
backtrace text the original reporter pasted could not be retrieved for a
byte-for-byte diff. What *is* independently confirmed, from the LKML thread
that carried the accepted fix
(https://lkml.iu.edu/hypermail/linux/kernel/1304.3/00703.html and
.../00763.html), is the root-cause statement: *"in fscache_stats_open,
single_open is called and respective release function is not called during
release"* — i.e. the same `single_open`/`seq_release` mismatch reproduced
below. Given that mechanism, the allocation-site call chain is deterministic
for any v3.9-era x86_64 kernel opening this file via `open(2)`, which is
exactly what our trace shows:

| Frame (leaf → root) | Reproduced here (v3.9-rc8, this run) | Expected per root-cause description |
|---|---|---|
| allocation hook | `kmemleak_alloc` | `kmemleak_alloc` |
| allocator | `kmem_cache_alloc_trace` | `kmem_cache_alloc_trace` / `__kmalloc` |
| **leak site** | `single_open` | `single_open` ✅ (explicitly named in the fix rationale) |
| **caller with the bug** | `fscache_stats_open` | `fscache_stats_open` ✅ (explicitly named in the fix rationale) |
| proc glue | `proc_reg_open` | `proc_reg_open` |
| VFS open | `do_dentry_open` → `finish_open` | `do_dentry_open` → `finish_open` |
| path walk | `do_last` → `path_openat` → `do_filp_open` | `do_last` → `path_openat` → `do_filp_open` |
| syscall entry | `do_sys_open` → `sys_openat` → `system_call_fastpath` | `do_sys_open` → `sys_open`/`sys_openat` → `system_call_fastpath` |

The two functions the upstream fix explicitly names as the bug
(`single_open` allocating, `fscache_stats_open` failing to pair it with
`single_release`) appear in identical position in the reproduced trace, and
the surrounding VFS/syscall frames are the standard v3.9 `open(2)` path with
no divergence. This confirms the reproduction matches the reported bug's
mechanism, not just its symptom.

## Fix (not applied, referenced only)

```diff
-	.release	= seq_release,
+	.release	= single_release,
```

Applying this to `fs/fscache/stats.c` (commit
`ec686c9239b4d472052a271c505d04dae84214cc`) would call `single_release()`
instead, which does `kfree(seq_file->op)` before freeing the `seq_file`
itself — eliminating the leak. This was intentionally **not** applied per
the task constraint.

## Artifacts

- `build-fixes.md` — toolchain gap log
- `proof/build.log` — full kernel build log
- `proof/boot-output.log` — full QEMU serial console capture
- `proof/kmemleak-output.log` — extracted kmemleak report (4 leaks)
- `bzImage`, `initramfs.cpio.gz`, `initramfs/init` — bootable artifacts used
