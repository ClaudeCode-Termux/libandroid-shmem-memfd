# libandroid-shmem-memfd

System V shared memory (shmget/shmat/shmdt/shmctl) on Android — using **memfd** instead of ashmem.

## Why?

The original [libandroid-shmem](https://github.com/termux/libandroid-shmem) uses Android's ashmem subsystem. But on newer devices (Honor, Huawei, Android 13+), `ashmem SET_SIZE` returns `ENOTTY` — the kernel has deprecated ashmem.

This fork replaces ashmem with `memfd_create()` + `ftruncate()`, which works on **all Android 10+ devices** without any `/dev/ashmem` dependency.

## What changed

```diff
- ashmem_create_region() → open("/dev/ashmem") + ioctl(SET_SIZE)
+ memfd_create_region()  → memfd_create() + ftruncate()

- ashmem_get_size_region() → ioctl(ASHMEM_GET_SIZE)
+ memfd_get_size_region()  → fstat().st_size
```

Everything else (SCM_RIGHTS fd passing, mmap, cross-process sharing) is unchanged — it was already fd-based.

## Build

```bash
git clone https://github.com/2171628509a-cyber/libandroid-shmem-memfd.git
cd libandroid-shmem-memfd
make libandroid-shmem.so
# → libandroid-shmem.so (21KB)
```

## Test

```bash
cc -std=c11 -o test_shm test_shm.c -I. -L. -landroid-shmem -llog
LD_LIBRARY_PATH=. ./test_shm
# → shmget OK, shmat OK, write/read OK, shmdt OK, shmctl(RMID) OK
```

## Python

```python
from pyshmemfd import Shm

with Shm(65536) as shm:  # 64KB shared memory — memfd backed
    shm.write(b"HELLO FROM memfd!")
    print(shm.read(0, 21))
# → b'HELLO FROM memfd!'
```

## License

BSD-3-Clause (same as upstream)

## Credits

Forked from [termux/libandroid-shmem](https://github.com/termux/libandroid-shmem) — all credit to the original authors.
