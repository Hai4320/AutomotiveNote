---
title: Partition layout & flashing image
source: https://source.android.com/docs/core/architecture/partitions
phase: 3 — Embedded Linux & AOSP Build
tags:
  - AOSP
  - Partition
  - fastboot
  - Image
  - Treble
---

# Partition layout & flashing image

> Thuộc **Phase 3 — Embedded Linux & AOSP Build** trong [roadmap](../android-automotive-developer-roadmap.md). Note khép phase: `m` ([soong-make.md](soong-make.md), [build-aaos-image.md](build-aaos-image.md)) ra một **bộ `.img`** — note này giải thích **mỗi image là partition gì và flash lên thiết bị thế nào**. Khái niệm partition đã có ở [glossary.md](../glossary.md) — đây là góc **build output + flash**.
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## Từ `m` tới thiết bị: image là gì?

`m` không ra một file — nó ra **nhiều `.img`**, mỗi cái là nội dung của **một partition** trên thiết bị:

```
out/target/product/<board>/
├── system.img      → partition /system   (framework, Google)
├── vendor.img      → partition /vendor    (HAL, driver, SoC vendor)
├── product.img     → partition /product   (app OEM)
├── boot.img        → kernel (GKI) + ramdisk
├── vbmeta.img      → metadata AVB (chữ ký verify boot)
└── super.img       → gộp system+vendor+product (dynamic partitions)
```

**Flash** = ghi một `.img` vào đúng partition trên thiết bị. Mỗi partition có chủ + vòng đời update riêng — đây là ranh giới **Treble** ở dạng vật lý (xem [glossary.md](../glossary.md)): update Android = thay `system.img`, không đụng `vendor.img`.

## Partition nào chứa gì (góc build)

| Image | Partition | Chứa | Ai build |
|-------|-----------|------|----------|
| `system.img` | `/system` | Framework, system app, ART | Google/AOSP |
| `vendor.img` | `/vendor` | HAL service, driver userspace, sepolicy vendor | SoC vendor |
| `product.img` | `/product` | App/config theo dòng sản phẩm | OEM |
| `odm.img` | `/odm` | Tùy biến từng thiết bị (đè vendor) | ODM/OEM |
| `boot.img` | `boot` | Kernel + ramdisk | Google (GKI) |
| `vendor_boot.img`, `vendor_dlkm.img` | — | Vendor kernel modules `.ko` | SoC vendor |
| `vbmeta.img` | `vbmeta` | Chữ ký AVB verify các partition trên | — |

- **Nhớ luật Treble**: HAL → `vendor.img`, không bao giờ `system.img`. Đây là lý do `Android.bp` của HAL cần `vendor: true` ([soong-make.md](soong-make.md)).
- **super.img (dynamic partitions)**: Android 10+ gộp system/vendor/product vào một partition `super` co giãn được, thay cho phân vùng cứng cố định.

## fastboot — flash qua USB

**fastboot** = chế độ bootloader + tool ghi partition qua USB (khác `adb` chạy khi OS đã boot). Vào bằng lệnh hoặc giữ nút lúc boot:

```bash
adb reboot bootloader              # vào fastboot mode
fastboot devices                   # kiểm tra thấy thiết bị

fastboot flash boot boot.img       # flash 1 partition cụ thể
fastboot flash vendor vendor.img
fastboot flashall                  # flash mọi image trong $OUT (theo target đã lunch)
fastboot reboot
```

- **`fastboot flashall`** (hoặc `fastboot -w flashall`) flash cả bộ image của target đang lunch — cách nhanh nhất đưa build lên board.
- Muốn flash được thường phải **unlock bootloader** (`fastboot flashing unlock`) — mở khóa xóa sạch userdata (bảo mật).
- **AVB/dm-verity**: flash image không khớp chữ ký → **không boot** ([glossary.md](../glossary.md)). Lúc dev thường `fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img` để bỏ qua.

## fastboot vs adb — hai chế độ khác nhau

| | fastboot | adb |
|--|----------|-----|
| Chạy khi | Bootloader (OS chưa boot) | OS đã boot |
| Việc chính | Flash partition, unlock | Shell, push file, logcat, debug |
| Vào bằng | `adb reboot bootloader` / giữ nút | Tự có khi OS chạy |

Bring-up board mới: khi OS chưa boot nổi, **fastboot** (và serial console — Phase 6) là cứu cánh; `adb` chưa dùng được.

## 🚗 Liên hệ Automotive

- **Bench board thật**: quy trình chuẩn khi làm HAL/framework trên xe — `mmm` module → `m` partition liên quan → `fastboot flash vendor vendor.img` → reboot → test. Emulator (`adb sync`, [build-aaos-image.md](build-aaos-image.md)) chỉ là bước tập; board thật flash partition thật.
- **Chỉ flash partition đổi**: sửa HAL → chỉ `fastboot flash vendor vendor.img`, khỏi flash cả bộ → nhanh hơn nhiều. Biết code mình sửa rơi vào partition nào là kỹ năng thực dụng.
- **A/B (seamless) update trên xe**: xe production update qua **hai slot A/B** (Phase 7) — flash slot nền trong lúc xe chạy slot kia, reboot đổi slot. fastboot ở đây là công cụ dev/bring-up, không phải cơ chế update ngoài đường.
- **AVB nghiêm ở xe**: safety đòi verify boot chặt. Flash image lỗi/chưa ký = board không boot — lúc bench nhớ xử lý verity, đừng nghi code.

## Câu hỏi tự kiểm tra

- [ ] `m` ra một file hay nhiều file? Mỗi `.img` tương ứng cái gì?
- [ ] "Flash" nghĩa là gì? Vì sao update Android chỉ thay `system.img`?
- [ ] HAL nằm ở image nào, vì sao? Liên hệ với `vendor: true`?
- [ ] super.img (dynamic partition) giải quyết vấn đề gì?
- [ ] fastboot vs adb — chạy ở giai đoạn nào, làm việc gì?
- [ ] `fastboot flashall` khác `fastboot flash boot boot.img` chỗ nào?
- [ ] Vì sao flash image sai làm thiết bị không boot? Lúc dev xử lý sao?

## 📚 Tài liệu

- 📖 [Partitions overview](https://source.android.com/docs/core/architecture/partitions) — layout chính thức
- 📖 [Dynamic partitions](https://source.android.com/docs/core/ota/dynamic_partitions) — super.img
- 📖 [fastboot](https://source.android.com/docs/setup/test/running#booting-into-fastboot-mode) — flash qua USB
- 📖 [Verified Boot (AVB + dm-verity)](https://source.android.com/docs/security/features/verifiedboot) — vì sao image sai không boot

## Đọc tiếp

- [build-aaos-image.md](build-aaos-image.md) — bước trước: `m` ra chính các `.img` này.
- [soong-make.md](soong-make.md) — `vendor: true` quyết định module vào partition nào.
- [kernel/android-gki.md](../1_android_internal/kernel/android-gki.md) — boot.img/vendor_boot chi tiết.
- [glossary.md](../glossary.md) — Treble, partition, AVB, dm-verity, fastboot tra nhanh.
