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

**LKM (Loadable Kernel Module) là một mẩu code kernel đóng gói thành file `.ko`, nạp vào kernel *lúc runtime* thay vì compile cứng vào trong kernel.** Điển hình là driver phần cứng: driver màn hình, cảm biến, CAN... mỗi cái là một module `.ko` riêng, kernel cần thì `insmod`/`modprobe` nạp vào, không cần thì thôi. Giống như plugin của một ứng dụng — lõi giữ nguyên, tính năng cắm thêm rời.

Vì sao Android bắt buộc cơ chế này (từ **Android 8.0** mọi SoC kernel phải hỗ trợ LKM)? Vì nó là **nền tảng cho Treble và GKI**: muốn có một kernel lõi (GKI) dùng chung cho mọi máy thì phần driver riêng của từng chip *bắt buộc* phải tách ra nạp rời — và LKM chính là cách tách đó. Không có LKM thì không thể "một kernel chạy mọi thiết bị". Note này đi vào chi tiết: module nằm partition nào, nạp ra sao, và cách kiểm tra tương thích ABI để module không nạp nhầm kernel.

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
