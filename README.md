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
