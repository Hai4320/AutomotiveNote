---
title: VINTF Objects (Vendor Interface)
source: https://source.android.com/docs/core/architecture/vintf/objects?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - VINTF
  - Treble
  - HAL
  - Manifest
---

# VINTF Objects (Vendor Interface)

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/architecture/vintf/objects](https://source.android.com/docs/core/architecture/vintf/objects?hl=vi)
> Thuộc chuỗi HAL trong [roadmap](../../android-automotive-developer-roadmap.md) — đọc sau [aidl-hals.md](aidl-hals.md). Đây là phần "khai báo VINTF manifest" mà AIDL HAL bắt buộc.

## VINTF là gì?

- Hệ thống **đảm bảo tương thích device (vendor) ↔ framework (system)** — trái tim của Project Treble.
- Tổng hợp data từ **manifest** (bên này *có* gì) và **compatibility matrix** (bên kia *cần* gì), rồi match chéo.

```
Device side                     Framework side
─────────────                   ──────────────
Device manifest      ──match──▶ Framework Compatibility Matrix (FCM)
(vendor có HAL gì)              (framework cần HAL gì)

Device Compat Matrix ◀──match── Framework manifest
(vendor cần gì)                 (framework có gì)
```

## 2 loại manifest

### Device manifest (vendor/ODM viết)
- Liệt kê **HAL vendor cung cấp**, SELinux policy version, khả năng phần cứng.
- Load theo thứ tự: **vendor manifest → vendor fragments → ODM manifest → ODM fragments** (ODM override được vendor).
- Path chính: `/vendor/etc/vintf/manifest.xml`

### Framework manifest (Google/system viết)
- HAL mà **system/product/system_ext cung cấp**, VNDK version (VNDK = bộ native lib ổn định framework cho vendor dùng), system SDK version.

## Thuộc tính XML quan trọng

| Attribute | Ý nghĩa |
|-----------|---------|
| `manifest.version` | Version của metadata format |
| `manifest.type` | `device` hoặc `framework` |
| `manifest.target-level` | **Bắt buộc với device manifest** — FCM version device nhắm tới |
| `manifest.hal.override="true"` | Manifest sau đè khai báo HAL của manifest trước |

## Manifest fragments (Android 10+)

- Gắn khai báo VINTF **trực tiếp vào HAL module** thay vì dồn 1 file to:

```
cc_binary {
    name: "android.hardware.light-service",
    vintf_fragments: ["light-manifest.xml"],
    ...
}
```

- Lợi: module có mặt trong build thì entry mới có mặt trong manifest — **conditional include** tự nhiên.

## Thứ tự load device manifest (runtime)

1. Vendor manifest + vendor fragments + ODM manifest + ODM fragments
2. Nếu thiếu vendor: ODM manifest + fragments
3. Legacy `/vendor/manifest.xml`
4. Cuối cùng: fragments từ **APEX vendor packages**

→ Nhiều product dùng chung 1 vendor image, khác biệt để ở ODM.

## Kiểm tra tương thích khi nào?

- **Boot**: mismatch nghiêm trọng có thể chặn boot
- **OTA**: check trước khi update — tránh flash framework mới lên vendor không tương thích
- **VTS**: test `vts_treble_vintf` bắt manifest khai đúng với HAL thực chạy

🚗 *Liên hệ Automotive:* thêm VHAL/custom HAL ở Phase 4 = bắt buộc thêm **vintf fragment** cho service đó, không thì service chạy nhưng client không được phép thấy (denied by servicemanager) hoặc fail VTS. Lỗi kinh điển khi port AAOS: `target-level` không khớp FCM của Android version mới.

## Câu hỏi tự kiểm tra

- [ ] Manifest vs compatibility matrix — bên nào khai "có", bên nào khai "cần"?
- [ ] Device manifest load theo thứ tự nào? ODM đè vendor được không?
- [ ] `target-level` là gì, bắt buộc với manifest nào?
- [ ] `vintf_fragments` giải quyết vấn đề gì so với 1 file manifest to?
- [ ] VINTF được check ở những thời điểm nào?

## 📚 Tài liệu

- 📖 [Compatibility matrices](https://source.android.com/docs/core/architecture/vintf/comp-matrices)
- 📖 [FCM lifecycle](https://source.android.com/docs/core/architecture/vintf/fcm)
- 📖 [Match rules](https://source.android.com/docs/core/architecture/vintf/match-rules)

## Đọc tiếp

- [aidl-hals.md](aidl-hals.md) — @VintfStability, freeze version
- [android-hal.md](android-hal.md) — HAL tổng quan
