---
title: AIDL for HALs
source: https://source.android.com/docs/core/architecture/aidl/aidl-hals
phase: 1 — Android Internals (code thật ở Phase 4)
tags:
  - AOSP
  - AIDL
  - HAL
  - VINTF
  - Binder
---

# AIDL for HALs

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/architecture/aidl/aidl-hals](https://source.android.com/docs/core/architecture/aidl/aidl-hals)
> Chuỗi HAL trong [roadmap](../../android-automotive-developer-roadmap.md) — đọc sau [android-hal.md](android-hal.md). Nền tảng để viết HAL thật ở **Phase 4**.

## Vì sao AIDL thay HIDL? (Android 11+)

| Lợi ích | Ý nghĩa |
|---------|---------|
| **1 ngôn ngữ IPC duy nhất** | App, framework, HAL cùng dùng AIDL — dễ học, dễ debug, dễ audit bảo mật |
| **Versioning tốt hơn** | Thêm method/field mới **không cần recompile toàn bộ** client |
| **Dynamic extension** | Gắn interface mở rộng lúc runtime |

## Yêu cầu bắt buộc cho AIDL HAL

- [ ] Interface chú thích **`@VintfStability`**
- [ ] Khai báo trong **VINTF manifest** (device manifest / compatibility matrix)
- [ ] Vendor code dùng **NDK backend** — link `libbinder_ndk`, **không dùng `libbinder`** (giữ API stability)

## Vị trí trong source tree

Đặt trong thư mục `aidl/` song song với HIDL cũ:

| Path | Chứa |
|------|------|
| `hardware/interfaces/` | Interface phần cứng (VHAL ở đây) |
| `frameworks/hardware/interfaces/` | Interface cấp cao cho framework |
| `system/hardware/interfaces/` | Interface cấp thấp cho system |

## Build: `aidl_interface` trong Android.bp

```
aidl_interface {
    name: "android.hardware.light",
    vendor_available: true,
    srcs: ["android/hardware/light/*.aidl"],
    stability: "vintf",          // bắt buộc cho HAL
    backend: {
        ndk: { enabled: true },  // vendor dùng backend này
    },
}
```

- **Freeze version:** interface ổn định thì freeze — snapshot vào `aidl_api/<name>/<version>/`. Client/service tương thích theo version đã freeze; đổi interface = tạo version mới, không sửa version cũ.

## Đăng ký & tìm service

- Service đăng ký với **servicemanager**, tên dạng `<interface>/<instance>`, vd:
  `android.hardware.automotive.vehicle.IVehicle/default`
- Client lấy qua `AServiceManager_waitForService()` (NDK) hoặc `ServiceManager.waitForService()` (Java).

## Migrate từ HIDL

- Tool **`hidl2aidl`** — convert interface HIDL → AIDL tự động: sinh file `.aidl`, build rules, translate methods.
- Sau convert vẫn phải tự review: types map chưa hoàn hảo, semantics callback khác nhau.

🚗 *Liên hệ Automotive:* VHAL AIDL (`android.hardware.automotive.vehicle`) chính là ví dụ chuẩn của mọi thứ trong note này — `@VintfStability`, freeze version trong `aidl_api/`, service `IVehicle/default`, vendor implement bằng NDK backend. Đọc source tại `hardware/interfaces/automotive/vehicle/aidl/` để thấy pattern thật.

## Câu hỏi tự kiểm tra

- [ ] 3 lợi ích của AIDL so với HIDL?
- [ ] `@VintfStability` để làm gì? Thiếu thì sao?
- [ ] Vendor code phải link lib nào, vì sao không dùng `libbinder`?
- [ ] Freeze interface version nghĩa là gì, snapshot nằm đâu?
- [ ] Service AIDL HAL đặt tên theo format nào? Ví dụ với VHAL?

## 📚 Tài liệu

- 📖 [Stable AIDL / versioning](https://source.android.com/docs/core/architecture/aidl/stable-aidl)
- 📖 [VINTF manifest](https://source.android.com/docs/core/architecture/vintf/objects)
- 📖 [AIDL backends](https://source.android.com/docs/core/architecture/aidl/aidl-backends)

## Đọc tiếp

- [android-hal.md](android-hal.md) — HAL tổng quan
- [android-vhal.md](android-vhal.md) — ví dụ AIDL HAL thật
- [vintf-objects.md](vintf-objects.md) — phần khai báo VINTF chi tiết
