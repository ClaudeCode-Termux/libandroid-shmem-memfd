# libandroid-shmem-memfd
[中文](./README_CN.md)

System V shared memory on Android — **memfd backend**. No `/dev/ashmem` dependency.

## Why this exists

The original [pelya/android-shmem](https://github.com/pelya/android-shmem) (222⭐) and its Termux port [termux/libandroid-shmem](https://github.com/termux/libandroid-shmem) (150⭐) both use **ashmem** as the backend. On newer devices (Honor ALT-AN00, kernel 5.10.226, Android 14), the kernel has deprecated ashmem — `ioctl(SET_SIZE)` returns `ENOTTY`.

This fork replaces the ashmem backend with `memfd_create()` + `ftruncate()`, preserving the exact same System V API (shmget/shmat/shmdt/shmctl). Cross-process fd sharing via SCM_RIGHTS is unchanged.

## Credits

This project is **not a competitor** — it's a backend swap on top of the excellent work by:

- [**pelya/android-shmem**](https://github.com/pelya/android-shmem) — original System V shm emulation for Android
- [**termux/libandroid-shmem**](https://github.com/termux/libandroid-shmem) — Termux packaging and maintenance

All credit for the System V shm API layer, the SCM_RIGHTS cross-process fd passing, and the UNIX socket listener architecture belongs to the original authors. This fork changes only the memory allocation backend.

## What changed

```diff
- ashmem_create_region() → /dev/ashmem + ioctl(SET_SIZE) 
+ memfd_create_region()  → memfd_create()   + ftruncate()

- ashmem_get_size_region() → ioctl(ASHMEM_GET_SIZE)
+ memfd_get_size_region()  → fstat()
```

2 functions changed. 18,000 lines unchanged. API identical.

## License

BSD-3-Clause (same as [pelya/android-shmem](https://github.com/pelya/android-shmem))

## Why memfd over ashmem? (updated findings)

Testing on Honor ALT-AN00 (kernel 5.10.226, Android 14):

| Issue | ashmem | memfd |
|-------|--------|-------|
| ioctl number | 64-bit `SET_SIZE` returns `ENOTTY`; only 32-bit compat ioctl works | `ftruncate()` — no ioctl guessing |
| SELinux | `shell` domain blocked from `/dev/ashmem` entirely (`EACCES`) | No device path needed |
| Device dependency | Requires `/dev/ashmem` node | Anonymous — no FS path |
| Cross-UID access | Only `untrusted_app` domain can open | Any process with fd access |

## Test matrix (real hardware)

| UID | Context | `/dev/ashmem` | `SET_SIZE` (32-bit) | `memfd` |
|-----|---------|:---:|:---:|:---:|
| 2000 | ADB shell | ❌ SELinux | — | ✅ |
| 10228 | Termux app | ✅ | ✅ | ✅ |
| 10318 | Device Owner | — | — | ✅ |

## Building

```bash
git clone https://github.com/ClaudeCode-Termux/libandroid-shmem-memfd.git
cd libandroid-shmem-memfd
make libandroid-shmem.so
# → libandroid-shmem.so (21KB)
```

## Testing

Full test suite passes on real hardware:

```bash
cc -std=c11 -o stress_shm test/stress_shm.c -I. -L. -landroid-shmem -llog
LD_LIBRARY_PATH=. ./stress_shm
# 🧪 10 test categories → ✅ ALL PASS

cc -std=c11 -o fallback_test test/fallback_test.c -I. -L. -landroid-shmem -llog  
LD_LIBRARY_PATH=. ./fallback_test
# 🧪 Backend fallback test → ✅ 6/6 PASS
```

## ashmem autopsy: Android 14 (kernel 5.10.226)

Full ioctl-level testing reveals that ashmem on modern Android kernels is a **zombie** — only the bare minimum allocation path survives:

### Alive
| Feature | Status |
|---------|:---:|
| SET_SIZE (allocation) | ✅ |
| mmap (R/W + R/O + SEGV guard) | ✅ |
| 10 concurrent fds | ✅ |
| 32MB large allocation | ✅ |

### Dead
| Feature | Status |
|---------|:---:|
| SET\_NAME / GET\_NAME | 💀 ENOTTY |
| GET\_SIZE | 💀 ENOTTY |
| SET\_PROT\_MASK / GET\_PROT\_MASK | 💀 ENOTTY |
| PIN / UNPIN | 💀 EINVAL |
| GET\_PIN\_STATUS | 💀 ENOTTY |
| PURGE\_ALL\_CACHES | 💀 ENOTTY |

### Bottom line

ashmem can allocate and map memory, but **every management function is dead**. You can't name a region, can't query its size, can't pin pages, can't purge caches, can't set protection masks. The kernel has deliberately gutted all advanced features, leaving only the minimal allocation path for legacy app compatibility.

memfd provides all of this — naming (`/proc/self/fd/N`), sizing (`fstat`), sealing (`F_SEAL_WRITE`), and more — with a single syscall and no device dependency.

## Memory comparison

| Feature | ashmem (Honor 5.10.226) | memfd |
|---------|:---:|:---:|
| Allocate | ✅ SET\_SIZE (32-bit compat) | ✅ ftruncate |
| mmap R/W | ✅ | ✅ |
| Name | ❌ ENOTTY | ✅ /proc/self/fd/N |
| Query size | ❌ ENOTTY | ✅ fstat |
| Pin pages | ❌ EINVAL | ✅ via mlock |
| Sealing | ❌ | ✅ F\_SEAL\_\* |
| SELinux shell access | ❌ EACCES | ✅ |
| Cross-process via SCM_RIGHTS | ✅ | ✅ |
