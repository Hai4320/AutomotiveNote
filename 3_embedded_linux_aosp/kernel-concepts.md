---
title: Kernel concepts — device tree, module, driver
source: https://bootlin.com/docs/
phase: 3 — Embedded Linux & AOSP Build
tags:
  - Linux
  - Kernel
  - Device Tree
  - Driver
  - Kernel Module
---

# Kernel concepts — device tree, module, driver

> Thuộc **Phase 3 — Embedded Linux & AOSP Build** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc **sau** [linux-fundamentals.md](linux-fundamentals.md) (device node là file trong `/dev`) và **sau** loạt kernel Phase 1 ([android-kernel.md](../1_android_internal/kernel/android-kernel.md), [android-gki.md](../1_android_internal/kernel/android-gki.md), [android-lkm.md](../1_android_internal/kernel/android-lkm.md)) — note này bổ khái niệm **driver / device tree / driver expose ra userspace thế nào**, thứ Phase 1 chưa nói.
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## Note này khớp với Phase 1 chỗ nào?

Phase 1 đã nói **kiến trúc kernel Android**: ACK, GKI, KMI, vendor module (`.ko`). Note này lùi một bước về **kiến thức kernel Linux nền**: driver là gì, phần cứng được mô tả cho kernel thế nào (**device tree**), và driver phơi mình ra userspace ra sao (**`/dev`, `/sys`**). Đây là mảnh nối giữa "kernel Android build thế nào" (Phase 1) và "HAL nói chuyện với phần cứng thế nào" (Phase 4).

```
[phần cứng: CAN controller]
        │  device tree (.dts) mô tả: địa chỉ, IRQ, "dùng driver nào"
        ▼
[driver trong kernel]  ──expose──►  /dev/can0  (userspace mở/đọc/ghi)
        │                           /sys/class/net/can0/  (trạng thái/config)
        ▼
[VHAL userspace mở /dev/can0]  → CarService → App   (Phase 4)
```

## Driver là gì?

**Driver** là đoạn code trong kernel biết cách nói chuyện với **một loại phần cứng cụ thể** (CAN controller, cảm biến, màn hình, GPU...). Kernel không biết chi tiết từng chip — nó cung cấp khung, driver điền phần "biết chip này ghi thanh ghi nào". Driver che phần cứng sau một **interface thống nhất** để userspace không cần biết chip nào bên dưới — đọc `/dev/can0` giống nhau dù chip Bosch hay chip khác.

Hai kiểu tồn tại:

- **Built-in** — biên dịch thẳng vào kernel image (boot có ngay).
- **Loadable Kernel Module (LKM, `.ko`)** — nạp/gỡ lúc runtime. Trên Android, vendor driver là **vendor module** nạp từ `vendor_dlkm` (xem [android-lkm.md](../1_android_internal/kernel/android-lkm.md)).

## Kernel space vs user space

Ranh giới quan trọng nhất của OS:

| | Kernel space | User space |
|--|-------------|-----------|
| Quyền | Full, đụng phần cứng trực tiếp | Bị giới hạn, sandbox |
| Chứa | Kernel, driver, module `.ko` | App, HAL, daemon, service |
| Crash | **Kernel panic** — chết cả máy | Process chết, máy sống |
| Vào nhau qua | **syscall** (user→kernel) | — |

- Userspace **không đụng phần cứng trực tiếp** — phải qua **syscall** (`open`, `read`, `write`, `ioctl`...) tới driver.
- Bug driver = **kernel panic** (chết cả xe) — nguy hiểm hơn hẳn crash một HAL userspace. Lý do Android đẩy càng nhiều logic lên userspace HAL càng tốt.

## Device tree — mô tả phần cứng cho kernel

PC dùng ACPI để kernel tự dò phần cứng. Hệ embedded (ARM, xe) **không tự dò được** — phần cứng cố định trên board, phải **mô tả tĩnh** cho kernel qua **Device Tree**:

```dts
/* file .dts — Device Tree Source */
can0: can@40006400 {
    compatible = "bosch,m_can";   /* "dùng driver nào" — kernel match chuỗi này */
    reg = <0x40006400 0x400>;     /* địa chỉ + kích thước vùng thanh ghi */
    interrupts = <21>;            /* dùng IRQ số mấy */
    status = "okay";              /* bật thiết bị này */
};
```

- `.dts` (source) → compile bằng `dtc` → `.dtb` (blob nhị phân) → bootloader nạp cho kernel lúc boot.
- **`compatible`** là chìa khóa: kernel so chuỗi này với bảng driver → chọn driver đúng.
- ⚠️ **"Device tree" nghĩa Linux kernel** (file `.dts` này) **khác** "device tree" nghĩa AOSP (`device/<vendor>/<board>/` chứa config build) — cùng tên, hai thứ. Xem [glossary.md](../glossary.md).

## Driver phơi ra userspace thế nào

Driver không hữu dụng nếu userspace không gọi được. Ba cửa chính:

| Cửa | Là gì | Ví dụ |
|-----|-------|-------|
| **`/dev/*`** (device node) | File đại diện thiết bị; userspace `open/read/write/ioctl` | `/dev/can0`, `/dev/ttyS0` (serial) |
| **`/sys/*`** (sysfs) | Trạng thái/config driver dạng file text, đọc/ghi từng thuộc tính | `/sys/class/net/can0/statistics/` |
| **`/proc/*`** (procfs) | Thông tin kernel/process | `/proc/interrupts`, `/proc/meminfo` |

```bash
adb shell ls -l /dev/ | grep -E '^c|^b'      # liệt kê character/block device
adb shell cat /proc/interrupts               # IRQ nào đang được driver nào dùng
adb shell cat /sys/class/net/can0/statistics/rx_packets   # đếm frame CAN nhận
adb shell dmesg | grep -i m_can              # log driver lúc probe (nạp)
```

Device node trong `/dev` được **ueventd** tạo dựa trên uevent kernel phát ra khi driver probe — và **ueventd gán owner/group** cho node (nối về permission ở [linux-fundamentals.md](linux-fundamentals.md)).

## 🚗 Liên hệ Automotive

- **Luồng CAN đầy đủ** (Phase 5 nối tiếp): device tree khai CAN controller → kernel nạp driver `m_can`/SocketCAN → tạo `/dev/can0` hoặc network interface `can0` → **VHAL userspace** đọc frame → CarService → App. Note này là mắt xích kernel của luồng đó.
- **Bring-up board xe**: sai một dòng trong `.dts` (địa chỉ, IRQ, `compatible`) = driver không probe = thiết bị "không tồn tại" với userspace. `dmesg` lúc boot là nơi đầu tiên nhìn khi bring-up.
- **Panic vs crash**: bug ở driver = kernel panic = xe treo, phải reset — nghiêm trọng với safety. Đây là lý do logic phức tạp đặt ở HAL userspace, không nhét vào driver.
- **Vendor module trên xe**: driver của SoC vendor là `.ko` nạp từ `vendor_dlkm`; lệch KMI với GKI = `disagrees about version of symbol` (xem [android-lkm.md](../1_android_internal/kernel/android-lkm.md)).

## Câu hỏi tự kiểm tra

- [ ] Driver là gì? Built-in vs loadable module (`.ko`) khác nhau chỗ nào?
- [ ] Kernel space vs user space — crash mỗi bên hậu quả ra sao? Userspace đụng phần cứng qua gì?
- [ ] Device tree giải quyết vấn đề gì mà PC (ACPI) không cần? Trường `compatible` để làm gì?
- [ ] `.dts` → `.dtb` qua công cụ nào, ai nạp nó?
- [ ] Ba cửa driver phơi ra userspace là gì? Cái nào để đọc/ghi data, cái nào xem trạng thái?
- [ ] Ai tạo device node trong `/dev` và gán owner/group cho nó?
- [ ] "Device tree" nghĩa kernel khác "device tree" nghĩa AOSP thế nào?

## 📚 Tài liệu

- 📖 [Bootlin — Linux kernel & driver development](https://bootlin.com/docs/) — slides chuẩn ngành về driver/device tree
- 📖 [Device Tree — kernel.org](https://www.kernel.org/doc/html/latest/devicetree/usage-model.html) — usage model chính thức
- 📖 [Linux Device Drivers (LDD3)](https://lwn.net/Kernel/LDD3/) — sách kinh điển, free online
- 📖 [SocketCAN kernel docs](https://www.kernel.org/doc/html/latest/networking/can.html) — driver CAN trên Linux (Phase 5)

## Đọc tiếp

- [linux-fundamentals.md](linux-fundamentals.md) — device node là file; ueventd gán permission cho nó.
- [kernel/android-lkm.md](../1_android_internal/kernel/android-lkm.md) — vendor module `.ko` trên Android, KMI.
- [kernel/android-gki.md](../1_android_internal/kernel/android-gki.md) — tách kernel core (Google) / vendor module.
- [hal/android-vhal.md](../1_android_internal/hal/android-vhal.md) — HAL userspace ngồi trên driver này (Phase 4).
