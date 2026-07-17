---
title: Linux fundamentals — filesystem, permission, process, signal
source: https://bootlin.com/docs/
phase: 3 — Embedded Linux & AOSP Build
tags:
  - Linux
  - Filesystem
  - Process
  - Signal
  - Permission
---

# Linux fundamentals — filesystem, permission, process, signal

> Thuộc **Phase 3 — Embedded Linux & AOSP Build** trong [roadmap](../android-automotive-developer-roadmap.md). Note mở đầu phase: Android **là** Linux, nên mọi thứ ở đây áp thẳng vào thiết bị xe. Đọc **trước** [shell-scripting.md](shell-scripting.md) (dùng lệnh thao tác các khái niệm ở đây) và [kernel-concepts.md](kernel-concepts.md) (device node là file, sinh ra từ mô hình filesystem này).
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## Vì sao app dev phải học lại Linux?

Làm app Android bạn không bao giờ thấy filesystem thật, process table hay signal — framework giấu hết sau Activity/Service. Xuống system-level thì ngược lại: bạn `adb shell` vào một máy Linux thật, một HAL "chết lặng" thường là do **permission file sai**, một daemon không khởi động lại là do **signal handler**, một mount `/vendor` sai là do **filesystem layout**. Đây là nền không thể bỏ qua trước khi đụng AOSP build và HAL.

Android khác Linux desktop ở vài điểm cốt lõi — nhớ sớm để khỏi ngạc nhiên:

- **Không có `glibc`** — Android dùng **Bionic** (libc riêng, nhỏ, license BSD).
- **Không có systemd** — process khởi động bởi **init** của Android (`init.rc`, xem [boot-process.md](../1_android_internal/boot-process.md)).
- **Mỗi app một UID riêng** — sandbox bằng chính cơ chế permission Unix bên dưới.
- **Nhiều partition mount read-only** (`/system`, `/vendor`) — bảo vệ bởi dm-verity ([glossary.md](../glossary.md)).

## Filesystem — mọi thứ là file

Triết lý Unix: thiết bị, socket, pipe, thư mục... đều biểu diễn dưới dạng **file** trong một cây thư mục **duy nhất** bắt đầu từ `/` (root). Không có "ổ C:/D:" như Windows — ổ mới thì **mount** vào một thư mục.

Cây thư mục quan trọng trên thiết bị Android:

| Path | Chứa |
|------|------|
| `/system` | Framework, system app, ART — partition Google (read-only) |
| `/vendor` | HAL, driver userspace, sepolicy vendor — partition SoC vendor (read-only) |
| `/product`, `/odm` | Tùy biến OEM/ODM |
| `/data` | Data người dùng, app cài thêm (read-write) — đây là chỗ duy nhất ghi thoải mái |
| `/dev` | **Device node** — cửa vào phần cứng (`/dev/can0`, `/dev/ttyS0`) → [kernel-concepts.md](kernel-concepts.md) |
| `/sys` | sysfs — trạng thái kernel/driver phơi ra dạng file |
| `/proc` | procfs — thông tin process & kernel (`/proc/<pid>/`, `/proc/meminfo`) |

**Mount** = gắn nội dung một partition/filesystem vào một điểm trong cây:

```bash
adb shell mount                    # xem mọi filesystem đang mount + ro/rw
adb shell mount -o rw,remount /    # remount / thành ghi được (userdebug, để sửa /system tạm)
adb root && adb remount            # cách nhanh remount các partition system để đẩy file test
```

⚠️ `/system` và `/vendor` mount **read-only** trên production. Muốn sửa file trong đó lúc dev → `adb remount` (chỉ userdebug/eng, và AVB/dm-verity phải tắt hoặc disable-verity).

## Permission — `rwx`, owner/group, và tại sao HAL cần đúng

Mỗi file có **owner**, **group** và 3 nhóm quyền `rwx` (read/write/execute) cho owner / group / others:

```bash
$ ls -l /dev/can0
crw-rw---- 1 root can 511, 0 ... /dev/can0
│└┬┘└┬┘└┬┘   │    │
│ │  │  │    │    └─ group = can
│ │  │  │    └────── owner = root
│ │  │  └─────────── others: --- (không quyền)
│ │  └────────────── group : rw-
│ └───────────────── owner : rw-
└─ 'c' = character device (không phải '-' file thường, 'd' dir, 'l' symlink)
```

- Số octal: `rwx` = 4+2+1. `chmod 660` = `rw-rw----`, `chmod 755` = `rwxr-xr-x`.
- `chown owner:group file`, `chmod`, `chgrp` để đổi.
- Ký tự đầu **loại file**: `-` thường, `d` dir, `l` symlink, `c` char device, `b` block device, `s` socket, `p` pipe.

Vì sao quan trọng khi làm HAL: process VHAL muốn mở `/dev/can0` thì **UID/GID của nó phải khớp** owner/group của node đó (thường qua `ueventd.rc` gán group cho device node) — sai group = `permission denied` **ngay cả trước khi SELinux vào cuộc**. Nhớ: có **hai hàng rào** — Unix permission (DAC) **và** SELinux (MAC, [selinux.md](../1_android_internal/selinux.md)). Fail có thể ở bất kỳ tầng nào.

## Process — chương trình đang chạy

**Process** = một instance của chương trình đang chạy, có **PID** (số định danh), một **parent** (PPID), một UID/GID, và bộ nhớ riêng.

```bash
adb shell ps -A                       # liệt kê mọi process (PID, UID, tên)
adb shell ps -A | grep car            # tìm process CarService/VHAL
adb shell cat /proc/<pid>/status      # trạng thái chi tiết 1 process
adb shell top                         # process theo CPU/RAM realtime
```

- **PID 1 = init** — process đầu tiên, tổ tiên của tất cả (trên Android là Android init, không phải systemd). Xem [boot-process.md](../1_android_internal/boot-process.md).
- Process mới sinh bằng **fork** (nhân bản parent) rồi **exec** (nạp chương trình mới). Zygote fork ra app process chính là mô hình này ([zygote.md](../1_android_internal/zygote.md)).
- **Zombie** = process đã chết nhưng parent chưa `wait()` thu exit code. **Orphan** = parent chết trước → init nhận nuôi.

## Signal — cách gửi tín hiệu bất đồng bộ cho process

**Signal** là thông điệp ngắn kernel/process gửi cho một process để báo sự kiện (bị kill, con chết, ngắt từ bàn phím...). Process có thể **cài handler** để xử lý, hoặc để hành vi mặc định.

| Signal | Số | Nghĩa | Bắt được? |
|--------|----|-------|-----------|
| `SIGTERM` | 15 | Xin dừng lịch sự (để dọn dẹp) | có |
| `SIGKILL` | 9 | Kill cứng, không thương lượng | **không** — kernel giết thẳng |
| `SIGSEGV` | 11 | Truy cập bộ nhớ sai (segfault) | có (để dump) |
| `SIGINT` | 2 | Ngắt từ terminal (Ctrl-C) | có |
| `SIGCHLD` | 17 | Con vừa chết | có |
| `SIGHUP` | 1 | Terminal đóng / "reload config" (quy ước) | có |

```bash
adb shell kill -TERM <pid>     # xin dừng lịch sự
adb shell kill -9 <pid>        # kill cứng (SIGKILL)
```

- `SIGKILL` (9) và `SIGSTOP` **không thể bắt hay bỏ qua** — đây là cách duy nhất chắc chắn dừng một process treo.
- `SIGSEGV` (11) là "crash native" kinh điển — con trỏ hỏng ([pointers-struct.md](../2_c_and_jni/pointers-struct.md)); kernel sinh **tombstone** để debug.

## 🚗 Liên hệ Automotive

- **HAL chết lặng** thường là chuỗi: sai owner/group của device node (DAC) **hoặc** thiếu `allow` trong sepolicy (MAC). Check cả hai: `ls -l /dev/xxx` **và** `dmesg | grep avc`.
- **Daemon xe chạy suốt** — không có luxury reboot cho nhanh như app điện thoại. Signal handling đúng (`SIGTERM` để flush trạng thái trước khi dừng, tránh mất data) quan trọng hơn hẳn trên xe.
- **`/vendor` read-only**: đẩy binary HAL test lên phải `adb remount` (userdebug). Trên production không sửa được — phải rebuild image (xem [build-aaos-image.md](build-aaos-image.md)).
- **`/proc` & `/sys`** là bạn khi bench-debug: đọc trạng thái CAN interface (`/sys/class/net/can0/`), thông số nhiệt, RAM... khi chưa có tool profiling.

```bash
adb shell dmesg | tail -50            # kernel log — nguồn chân lý khi HAL/driver fail
adb shell ls -Z /dev/can0             # xem SELinux label (-Z) của device node
adb shell ip -details link show can0  # trạng thái CAN interface (Phase 5)
```

## Câu hỏi tự kiểm tra

- [ ] Vì sao Android không có ổ "C:/D:"? "Mount" nghĩa là gì?
- [ ] `crw-rw----` đọc ra sao — loại file gì, ai có quyền gì?
- [ ] Hai hàng rào permission khi HAL mở `/dev/can0` là gì? Debug mỗi cái bằng lệnh nào?
- [ ] PID 1 trên Android là gì? Process mới sinh ra bằng cơ chế nào?
- [ ] `SIGTERM` vs `SIGKILL` khác gì? Signal nào không thể bắt?
- [ ] `/proc` và `/sys` chứa gì, khác `/data` chỗ nào?
- [ ] Vì sao `/system`, `/vendor` mount read-only? Sửa tạm lúc dev thế nào?

## 📚 Tài liệu

- 📖 [Linux Journey](https://linuxjourney.com/) — filesystem, permission, process từ zero
- 📖 [Bootlin — Embedded Linux training](https://bootlin.com/docs/) — slides chuẩn ngành, miễn phí
- 📖 [Bionic (Android libc) README](https://android.googlesource.com/platform/bionic/+/master/README.md) — vì sao Android không dùng glibc
- 📖 [Filesystem Hierarchy Standard](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html) — chuẩn cây thư mục `/`

## Đọc tiếp

- [shell-scripting.md](shell-scripting.md) — dùng bash để thao tác file/process/signal vừa học.
- [kernel-concepts.md](kernel-concepts.md) — device node trong `/dev` từ đâu ra, driver expose qua `/sys`.
- [boot-process.md](../1_android_internal/boot-process.md) — init (PID 1) khởi động daemon thế nào, `init.rc`.
- [selinux.md](../1_android_internal/selinux.md) — hàng rào permission thứ hai (MAC) đè lên `rwx`.
