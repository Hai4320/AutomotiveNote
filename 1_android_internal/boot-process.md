---
title: Boot Process & Init System
phase: 1 — Android Internals
tags:
  - AOSP
  - Boot
  - init
  - init.rc
---

# Boot Process & Init System — từ kernel đến system_server

> Thuộc **Phase 1 — Android Internals** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc sau [zygote.md](zygote.md) — mảnh cuối ghép kín bức tranh boot.

Boot process là **chuỗi bước từ lúc bấm nút nguồn đến khi màn hình launcher hiện ra** — mỗi bước dựng lên một tầng và bàn giao cho tầng kế tiếp. Nó phải chia thành nhiều giai đoạn vì lúc mới bật nguồn máy gần như chưa có gì: chưa có RAM được cấu hình, chưa có filesystem, chưa có process nào. Không thể nhảy thẳng vào chạy app, nên hệ thống dựng dần: phần cứng khởi động phần cứng, rồi mới nạp được kernel, kernel mới tạo được process đầu tiên, process đó mới dựng được phần còn lại.

Với app dev quen bắt đầu ở `onCreate()`, thì đây là tất cả những gì xảy ra **trước khi** code Java của bạn có cơ hội chạy. Ranh giới quan trọng nhất cần nhớ: **userspace bắt đầu từ `init`** (PID 1). Trước `init` là thế giới phần cứng và kernel (không gian nhân); từ `init` trở đi mới là các process bình thường — và mọi thứ Android mà ta biết (Zygote, system_server, app) đều là con cháu của `init`. Cứ hình dung `init` như hàm `main()` của cả userspace: nó là gốc, mọi process khác mọc ra từ nó.

## Toàn cảnh

```
Nguồn bật
 │
 ▼
1. Boot ROM          (code cứng trong SoC, load bootloader)
 │
 ▼
2. Bootloader        (ABL/U-Boot: init RAM, verify + load boot image; fastboot mode ở đây)
 │
 ▼
3. Kernel            (từ boot.img — GKI; mount ramdisk, start driver, tạo process đầu tiên)
 │
 ▼
4. init (PID 1)      (first stage → SELinux → second stage: parse init.rc, start services)
 │
 ├─▶ ueventd, logd, servicemanager, HAL services (vendor)
 │
 ▼
5. Zygote            (init ART VM + preload — xem zygote.md)
 │
 ▼
6. system_server     (fork đầu tiên từ Zygote — AMS, WMS, PMS... xem system-services.md)
 │
 ▼
7. Boot complete     (sys.boot_completed=1, ACTION_BOOT_COMPLETED, Launcher hiện)
```

## Chi tiết từng bước

### 1–2. Boot ROM & Bootloader
- Boot ROM: code trong silicon, không đổi được — verify + load bootloader.
- Bootloader: khởi tạo RAM/clock, **verify boot image (AVB — Android Verified Boot)** rồi nhảy vào kernel.
- Giữ nút → **fastboot mode** (chế độ flash partition qua USB bằng tool `fastboot`): flash từ đây ([android-gki.md](kernel/android-gki.md) — flash `boot`, `vendor_boot`).

### 3. Kernel
- Load từ `boot` partition (GKI) + vendor modules từ `vendor_boot`/`vendor_dlkm` ([android-lkm.md](kernel/android-lkm.md)).
- Mount ramdisk (filesystem nhỏ đóng trong boot image, nạp thẳng vào RAM — chứa `/init`), chạy xong khởi tạo → exec **`/init`**.

### 4. init (PID 1) — trái tim của userspace boot

Chạy 2 giai đoạn:

| Giai đoạn | Làm gì |
|-----------|--------|
| **First stage** | Mount partition cơ bản (`/dev`, `/proc`, `/sys`), load thêm kernel module, mount system/vendor qua dm-verity (kernel verify toàn vẹn partition mỗi lần đọc — xem [glossary.md](../glossary.md)) |
| **Second stage** | Load **SELinux policy** (permissive→enforcing), parse **init.rc**, xử lý property, start services |

#### init.rc — ngôn ngữ cấu hình

```
# Service: process init quản lý (tự restart nếu chết)
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart restart audioserver

# Action: chạy lệnh khi trigger bắn
on property:sys.boot_completed=1
    trigger load-bpf-programs
```

- File chính `/system/etc/init/hw/init.rc` + fragment từng module trong `/system/etc/init/`, `/vendor/etc/init/`.
- **Trigger** quan trọng: `early-init` → `init` → `late-init` → `boot`; property trigger `on property:x=y`.
- `ueventd` (cũng là init binary): tạo device node trong `/dev` theo uevent kernel.

### 5–7. Zygote → system_server → boot complete
- init start Zygote theo `init.zygote64.rc` — chi tiết ở [zygote.md](zygote.md).
- system_server start services 3 đợt — chi tiết ở [system-services.md](system-services.md).
- Xong hết: set `sys.boot_completed=1`, broadcast `ACTION_BOOT_COMPLETED`.

## Property system trong boot

| Property | Ý nghĩa |
|----------|---------|
| `ro.boot.*` | Từ kernel cmdline/bootconfig (bootloader truyền vào) |
| `ro.hardware` | Chọn file init.rc + HAL variant theo board |
| `sys.boot_completed` | `1` = boot xong |
| `ro.boottime.*` | Thời gian từng service start — đo boot time |

```bash
# Thực hành trên emulator
adb shell getprop sys.boot_completed
adb shell getprop | grep ro.boottime      # service nào chậm
adb logcat -b events | grep boot_progress # timeline boot
dmesg                                      # kernel stage
```

## 🚗 Liên hệ Automotive

- **Boot time = KPI hợp đồng** (OEM yêu cầu kiểu "cold boot → HMI (Human-Machine Interface — giao diện người lái thấy) < 10s, camera lùi < 2s").
- Camera lùi: đi đường **EVS (Extended View System)** — native service start bằng init.rc `early-init`/`boot` class sớm, **không chờ Zygote/Java**.
- Tune: sắp thứ tự service trong init.rc (class `early_hal` trước), giảm preload, dexpreopt ([art/art-configure.md](art/art-configure.md)), đo bằng `bootanalyze`/Perfetto boot trace.
- Vendor service tự viết = thêm file `.rc` vào `/vendor/etc/init/` + sepolicy — bài tập Phase 4.

## Câu hỏi tự kiểm tra

- [ ] Thứ tự 7 bước từ nguồn bật đến launcher?
- [ ] init first stage vs second stage làm gì?
- [ ] `service` vs `on` (action) trong init.rc khác nhau gì? Trigger chạy theo thứ tự nào?
- [ ] SELinux được load ở bước nào, chuyển từ mode gì sang gì?
- [ ] `sys.boot_completed` do ai set? Đo boot time bằng gì?
- [ ] Vì sao camera lùi không đi qua Java stack?

## 📚 Tài liệu

- 📖 [Android Init Language README](https://android.googlesource.com/platform/system/core/+/master/init/README.md) — spec init.rc chính thức
- 📖 [Android Verified Boot (AVB)](https://source.android.com/docs/security/features/verifiedboot)
- 📖 [Boot times optimization](https://source.android.com/docs/core/perf/boot-times) — docs chính thức đo & tune
- 📖 [init.rc thật (source)](https://cs.android.com/android/platform/superproject/main/+/main:system/core/rootdir/init.rc)

## Đọc tiếp

- [zygote.md](zygote.md) — note trong repo
- [system-services.md](system-services.md) — note trong repo
- [kernel/android-gki.md](kernel/android-gki.md) — boot partitions
