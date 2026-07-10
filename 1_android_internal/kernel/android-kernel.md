---
title: Android Kernel
source: https://source.android.com/docs/core/architecture/kernel?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - Kernel
  - GKI
  - ACK
---

# Android Kernel

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/architecture/kernel](https://source.android.com/docs/core/architecture/kernel?hl=vi)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc sau [android-architecture.md](android-architecture.md).

## Android Kernel là gì?

- Dựa trên **Linux LTS** (Long-Term Support) kernel.
- Google lấy LTS + thêm các patch riêng cho Android → **ACK (Android Common Kernel)**.
- Patch Android-specific chưa được merge vào Linux mainline (upstream).

```
Linux mainline → Linux LTS → ACK (android-mainline / androidXX-Y.Y)
                                  → GKI kernel + vendor modules → device kernel
```

## Các nhánh kernel (ACK branches)

| Nhánh | Vai trò |
|-------|---------|
| **android-mainline** | Nhánh phát triển chính — feature Android mới vào đây trước |
| **KMI ACK branches** | Đặt tên theo `android<phiên bản>-<kernel>`, vd `android15-6.6` — nhánh ổn định cho device ship |

## GKI (Generic Kernel Image)

Giải quyết **kernel fragmentation** — trước đây mỗi OEM/SoC fork kernel riêng, vá bảo mật chậm.

- Áp dụng từ kernel **5.10+**, chỉ **aarch64**.
- Tách **core kernel chung** (Google build, dùng chung mọi device) khỏi **hardware-specific modules** (load động).
- Device mới ship với GKI kernel do Google cung cấp, vendor không sửa core kernel nữa.

### Kernel Modules — 2 loại

| Loại | Ai làm | Chứa gì |
|------|--------|---------|
| **GKI modules** | Google | Module chung, phân phối kèm GKI |
| **Vendor modules** | SoC vendor (Qualcomm, NXP...) | Driver riêng cho phần cứng |

### KMI (Kernel Module Interface)

- Interface ổn định giữa **GKI kernel ↔ vendor modules**.
- Định nghĩa qua **symbol list** — các function/global data mà vendor module được phép dùng.
- Nhờ KMI ổn định → Google update GKI kernel độc lập, vendor module không cần build lại.
- 🚗 *Liên hệ Automotive:* SoC xe (Qualcomm SA8155/8295, NXP i.MX, Renesas R-Car) đều ship driver dạng vendor module trên nền GKI/ACK — hiểu KMI để debug lỗi load module khi bring-up board.

## So sánh nhanh: trước vs sau GKI

| | Trước GKI | Sau GKI |
|--|-----------|---------|
| Kernel | Mỗi OEM fork riêng | 1 kernel chung Google build |
| Driver | Trộn trong kernel | Tách thành vendor module |
| Security patch | Chờ OEM merge, chậm | Google update GKI trực tiếp |
| Fragmentation | Nặng | Giảm mạnh |

## Câu hỏi tự kiểm tra

- [ ] ACK khác Linux LTS chỗ nào?
- [ ] GKI giải quyết vấn đề gì? Áp dụng từ kernel version nào?
- [ ] GKI module vs vendor module — ai build, chứa gì?
- [ ] KMI là gì, vì sao cần symbol list?
- [ ] Nhánh `android15-6.6` nghĩa là gì?

## 📚 Tài liệu

- 📖 [GKI overview](https://source.android.com/docs/core/architecture/kernel/generic-kernel-image)
- 📖 [Vendor modules](https://source.android.com/docs/core/architecture/kernel/loadable-kernel-modules)
- 📖 [ACK branches](https://source.android.com/docs/core/architecture/kernel/android-common)

## Đọc tiếp

- [android-ack.md](android-ack.md) — nhánh & vòng đời chi tiết
- [android-gki.md](android-gki.md) — GKI chi tiết
- [android-lkm.md](android-lkm.md) — vendor modules chi tiết
