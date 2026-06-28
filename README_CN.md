# libandroid-shmem-memfd
[English](./README.md)

Android 上的 System V 共享内存——**memfd 驱动**。告别 `/dev/ashmem`。

## 为什么写这个

[原版 pelya/android-shmem](https://github.com/pelya/android-shmem) (222⭐) 和 [Termux 移植版](https://github.com/termux/libandroid-shmem) (150⭐) 都基于 Android 的 **ashmem** 子系统。但在较新的设备上（实测 Honor ALT-AN00, kernel 5.10.226, Android 14），内核已弃用 ashmem——`ioctl(SET_SIZE)` 直接返回 `ENOTTY` 😭💀

本项目把底层从 ashmem 换成 **memfd_create() + ftruncate()**，对上层保持完全相同的 System V API（shmget/shmat/shmdt/shmctl）。跨进程 fd 传递（SCM_RIGHTS）原封不动。

## 实测环境

| 设备 | 内核 | Android | ashmem SET_SIZE | memfd |
|------|------|:---:|:---:|:---:|
| Honor ALT-AN00 | 5.10.226 | 14 | ❌ ENOTTY | ✅ |
| 其他 Android 10+ | — | — | 看脸 | ✅ |

## 改动内容

```diff
- ashmem_create_region() → /dev/ashmem + ioctl(SET_SIZE) 
+ memfd_create_region()  → memfd_create()   + ftruncate()

- ashmem_get_size_region() → ioctl(ASHMEM_GET_SIZE)
+ memfd_get_size_region()  → fstat()
```

**改了 2 个函数。其余 18,000 行没动。**

## 编译

```bash
git clone https://github.com/2171628509a-cyber/libandroid-shmem-memfd.git
cd libandroid-shmem-memfd
make libandroid-shmem.so
# → libandroid-shmem.so (21KB)
```

## 测试

```bash
cc -std=c11 -o test_shm test_shm.c -I. -L. -landroid-shmem -llog
LD_LIBRARY_PATH=. ./test_shm
# shmget ✅ → shmat ✅ → write/read ✅ → shmdt ✅ → shmctl(RMID) ✅
```

## Python 接口

```python
from pyshmemfd import Shm

with Shm(65536) as shm:  # 64KB memfd 共享内存
    shm.write(b"你好 memfd!")
    print(shm.read(0, 13))
# → b'你好 memfd!'
```

## 致谢

本项目**不是竞争者**——仅在以下杰出项目的基础上替换了后端：

- [**pelya/android-shmem**](https://github.com/pelya/android-shmem) —— Android System V shm 的原始实现
- [**termux/libandroid-shmem**](https://github.com/termux/libandroid-shmem) —— Termux 的移植与维护

System V shm 的 API 层、SCM_RIGHTS 跨进程 fd 传递、UNIX socket 监听架构，全部归功于原作者。

## 协议

BSD-3-Clause（与原版 [pelya/android-shmem](https://github.com/pelya/android-shmem) 一致）
