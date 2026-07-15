---
title: GKI (Generic Kernel Image)
source: https://source.android.com/docs/core/architecture/kernel/generic-kernel-image?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - Kernel
  - GKI
  - KMI
---

# GKI (Generic Kernel Image)

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/architecture/kernel/generic-kernel-image](https://source.android.com/docs/core/architecture/kernel/generic-kernel-image?hl=vi)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../../android-automotive-developer-roadmap.md). Đọc sau [android-kernel.md](android-kernel.md).

## GKI là gì?

**GKI (Generic Kernel Image) là một kernel binary *dùng chung* do Google build, giống hệt nhau trên mọi thiết bị cùng kiến trúc + cùng phiên bản** — phần driver riêng của từng chip/board bị tách ra thành **module nạp rời (loadable vendor modules)**. Nói cách khác: thay vì mỗi hãng có một kernel "trộn" cả lõi Android lẫn driver phần cứng của họ, GKI chẻ đôi — **lõi chung** một bên, **driver vendor** một bên, ghép lại lúc chạy.

> "GKI giải quyết kernel fragmentation bằng cách **gom core kernel về một mối**, đẩy toàn bộ SoC & board support ra **loadable vendor modules**."

Vì sao cần? Trước GKI, **~50% code kernel là code tự chế của OEM/SoC vendor nằm ngoài Linux upstream** — hậu quả là **fragmentation** (phân mảnh): mỗi thiết bị một kernel khác nhau, nên vá lỗi bảo mật cực chậm (phải chờ từng vendor merge rồi build lại) và nâng version LTS thì rất khổ. Tách lõi ra dùng chung giúp Google **vá kernel độc lập, đẩy thẳng tới mọi máy** mà không cần chờ vendor đụng tới phần driver của họ.

## Mục tiêu

- [ ] **1 kernel binary duy nhất** cho mỗi architecture (per LTS release) + loadable modules
- [ ] Google/partner phát hành **security patch độc lập**, không cần chờ vendor
- [ ] Giảm chi phí upgrade major kernel version
- [ ] Không degrade hiệu năng so với production kernel cũ

## Kiến trúc

```
┌──────────────────────────────┐
│  GKI kernel (core, chung)    │  ← Google build, KHÔNG chứa code SoC/board
├──────── KMI (stable) ────────┤  ← symbol list, interface ổn định
│  Vendor modules (.ko)        │  ← SoC vendor build (Qualcomm, NXP...)
└──────────────────────────────┘
```

- **Core kernel:** không có SoC-specific hay board-specific code.
- **KMI (Kernel Module Interface):** giữ binary stability — kernel và module update độc lập nhau.

## Partition liên quan (khi flash board)

| Partition | Chứa |
|-----------|------|
| `boot` | GKI kernel (Google build) |
| `vendor_boot` | Vendor ramdisk + vendor kernel modules boot sớm |
| `vendor_dlkm` | Vendor kernel modules load sau (dynamic loadable kernel modules) |

## Yêu cầu compliance

- Từ **Android 12**, device ship với kernel **5.10+** bắt buộc dùng GKI.
- GKI release build được update liên tục (LTS fix + critical patch) nhưng giữ **KMI binary stability**.
- Verify bằng **VTS** (Vendor Test Suite).

🚗 *Liên hệ Automotive:* bring-up board AAOS = flash GKI kernel vào `boot`, driver SoC (SA8155, i.MX, R-Car) vào `vendor_boot`/`vendor_dlkm`. Lỗi boot thường gặp: vendor module không khớp KMI version → check `dmesg` symbol mismatch.

## Câu hỏi tự kiểm tra

- [ ] Trước GKI, bao nhiêu % kernel code nằm ngoài upstream? Hệ quả?
- [ ] Core kernel GKI được phép chứa code SoC-specific không?
- [ ] KMI giữ ổn định thứ gì, cho phép điều gì?
- [ ] `boot` vs `vendor_boot` vs `vendor_dlkm` chứa gì?
- [ ] Từ Android bao nhiêu, kernel nào bắt buộc GKI?

## 📚 Tài liệu

- 📖 [KMI symbol lists](https://source.android.com/docs/core/architecture/kernel/symbol-lists)
- 📖 [GKI release builds](https://source.android.com/docs/core/architecture/kernel/gki-release-builds)
- 📖 [Boot partitions](https://source.android.com/docs/core/architecture/partitions)

## Đọc tiếp

- [android-kernel.md](android-kernel.md) — kernel tổng quan
- [android-lkm.md](android-lkm.md) — vendor modules chi tiết
- [android-ack.md](android-ack.md) — nhánh kernel & vòng đời
