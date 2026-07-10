---
title: Loadable Kernel Modules (LKM)
source: https://source.android.com/docs/core/architecture/kernel/loadable-kernel-modules?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - Kernel
  - LKM
  - Vendor Modules
---

# Loadable Kernel Modules (LKM)

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/architecture/kernel/loadable-kernel-modules](https://source.android.com/docs/core/architecture/kernel/loadable-kernel-modules?hl=vi)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../../android-automotive-developer-roadmap.md). Đọc sau [android-gki.md](android-gki.md).

## LKM là gì?

- Kernel module (`.ko`) **load động lúc runtime**, không compile cứng vào kernel.
- Từ **Android 8.0**: mọi SoC kernel **bắt buộc hỗ trợ LKM** — nền tảng cho Treble & GKI.

### Kernel config bắt buộc

```
CONFIG_MODULES=y          # bật hỗ trợ module
CONFIG_MODULE_UNLOAD=y    # cho phép unload
CONFIG_MODVERSIONS=y      # CRC check ABI compatibility
```

## Phân loại

| Loại | Ai build | Ghi chú |
|------|----------|---------|
| **GKI modules** | Google | Module chung, **ký** từ GKI build |
| **Vendor modules** | SoC vendor / ODM | Riêng từng thiết bị, không ký |

## Vị trí lưu (partition)

| Module | Path |
|--------|------|
| GKI modules | `/system/lib/modules` → symlink `/system_dlkm/lib/modules` |
| Vendor (SoC) | `/vendor/lib/modules` |
| ODM | `/odm/lib/modules` (hoặc `/vendor/lib/modules`) |
| Recovery | `/lib/modules` trong recovery ramfs |

⚠️ Quy tắc: module phải load từ **partition read-only**; vendor module không được nằm trong `/system`.

## Quá trình load

- Load **tất cả module 1 lần** từ `init.rc` bằng `modprobe -a`.
- Lý do: tránh spawn C runtime nhiều lần → boot nhanh hơn.
- Build system tự chạy `depmod` → sinh `modules.dep` (thứ tự dependency).

## Ký module & bảo vệ

- Vendor module trên GKI: **không dùng module signing**.
- Thay bằng **dm-verity** (verified boot) bảo vệ partition chứa module.
- Module chưa ký load được nếu chỉ dùng symbol trong **allowlist (KMI symbol list)**.

## ABI compatibility (CONFIG_MODVERSIONS)

- Tính **CRC checksum** cho từng symbol prototype.
- Load module: so CRC — khớp thì load, lệch thì **fail**.
- ⚠️ Hạn chế: MODVERSIONS không fix được ABI đổi thật sự — struct như `task_struct` đổi layout thì module cũ **phải recompile**.

## Tích hợp vào Android build

Khai báo trong `BoardConfig.mk`:

```makefile
BOARD_VENDOR_KERNEL_MODULES := <danh sách .ko cho vendor image>
BOARD_RECOVERY_KERNEL_MODULES := <subset cho recovery>
```

🚗 *Liên hệ Automotive:* bring-up board AAOS phần lớn là chỉnh danh sách vendor module (CAN driver, display, touch...) trong `BoardConfig.mk` + debug lỗi load. Lỗi kinh điển: `disagrees about version of symbol` trong `dmesg` = CRC mismatch → module build lệch kernel version, cần rebuild đúng KMI.

## Câu hỏi tự kiểm tra

- [ ] Từ Android nào LKM bắt buộc? 3 config kernel nào cần bật?
- [ ] GKI module vs vendor module: khác nhau về ký (signing) thế nào?
- [ ] Vendor module nằm partition nào? Vì sao không được ở `/system`?
- [ ] Vì sao load module bằng `modprobe -a` 1 lần từ `init.rc`?
- [ ] CRC mismatch khi load module — nguyên nhân và cách xử lý?

## 📚 Tài liệu

- 📖 [KMI symbol lists](https://source.android.com/docs/core/architecture/kernel/symbol-lists)
- 📖 [dm-verity / Verified Boot](https://source.android.com/docs/security/features/verifiedboot)

## Đọc tiếp

- [android-gki.md](android-gki.md) — GKI mà module cắm vào
- [android-kernel.md](android-kernel.md) — kernel tổng quan
