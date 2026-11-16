# Kernel Fuzzing Notes

A learning log of my path into Linux kernel security: building a debuggable
kernel, running it in QEMU, and working toward fuzzing with Syzkaller.
I'm a beginner and writing this to understand each step, not just copy it.

## Environment
- Laptop: ThinkPad X1 Carbon Gen 8 (Intel i7-10710U, 16 GB RAM)
- OS: Ubuntu [version]
- Virtualization: KVM enabled (VT-x), QEMU [version]
- Kernel source: Linux v6.6

## Progress
- [x] Verified KVM works (`vmx` flag, `/dev/kvm`)
- [x] Stage 1: booted the host kernel in QEMU with a BusyBox initramfs
- [ ] Stage 2: build Linux v6.6 with KCOV, KASAN and DWARF debug info, boot it in QEMU
- [ ] Stage 3: install and run Syzkaller against that kernel
- [ ] Stage 4: read and reproduce a crash report

## Stage 1: minimal QEMU boot
Goal: understand what a kernel needs in order to boot and what PID 1 is.

What I built:
- A tiny root filesystem (initramfs) containing a static `busybox` and an `init` script.
- `init` mounts `/proc`, `/sys` and `/dev`, prints a message, then `exec`s a shell.

Boot command:

```bash
qemu-system-x86_64 -enable-kvm -cpu host -m 1G -smp 2 \
  -kernel vmlinuz -initrd initramfs.cpio.gz \
  -append "console=ttyS0" -nographic
```

What I learned:
- The kernel runs `/init` as PID 1. `exec /bin/sh` replaces the program inside
  that same process, so `ps` shows `/bin/sh` as PID 1. If PID 1 exits, the kernel panics.
- `mount` showing `none on /proc` is normal: `none` is just the placeholder for
  the source of a filesystem with no backing device.
- BusyBox must be statically linked, because the initramfs has no shared libraries.

### Debugging log
**Problem:** the VM showed only a blinking cursor and accepted no input.
**Cause:** typo in the kernel command line: `console=tty50` instead of `console=ttyS0`.
**Why it was silent:** the kernel ignores unknown console names and sent its output
to the default VGA console, which doesn't exist with `-nographic`. The VM was
running fine, but its output went where I couldn't see it.
**Lesson:** when a VM is blank, check the `-append` string first, then the files,
then the CPU/KVM options.

## Stage 2 (in progress): instrumented kernel
- KCOV: coverage feedback, exposed at `/sys/kernel/debug/kcov`
- KASAN: detects use-after-free and out-of-bounds accesses
- DWARF debug info: maps crash addresses to source lines

Config used on top of `defconfig` + `kvm_guest.config`:
`DEBUG_KERNEL, DEBUG_INFO_DWARF4, KCOV, KASAN, KASAN_INLINE, DEBUG_FS, CONFIGFS_FS, SECURITYFS`

## Next
- Install Go and build Syzkaller
- Create a VM disk image and write the Syzkaller config
- Run it and study the first crash report

## References
- Syzkaller docs: `docs/linux/` in https://github.com/google/syzkaller
- Linux kernel documentation on KASAN and KCOV (kernel.org)
