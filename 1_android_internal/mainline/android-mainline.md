---
title: Mainline (Modular System Components)
source: https://source.android.com/docs/core/ota/modular-system?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - Mainline
  - APEX
  - OTA
---

# Mainline (Modular System Components)

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/ota/modular-system](https://source.android.com/docs/core/ota/modular-system?hl=vi)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../../android-automotive-developer-roadmap.md). Đọc sau [android-architecture.md](../android-architecture.md).

## Mainline là gì?

- Từ **Android 10**: cho phép **update các thành phần hệ thống ngoài chu kỳ Android release** — không cần OTA (Over-The-Air update) full image, không đụng vendor implementation hay app.
- Mục đích chính: vá bảo mật + fix bug **nhanh, đồng loạt mọi device**, không chờ OEM.

## Nguyên tắc kiến trúc

- Module tách khỏi system, update độc lập.
- Chỉ dùng **SDK/API được CTS đảm bảo** + giao tiếp qua **stable AIDL** → module mới chạy được trên system cũ.
- Cài đặt **atomic** — cả gói update thành công hoặc rollback, không nửa vời.

## 2 định dạng phân phối

| Format | Đặc điểm |
|--------|----------|
| **APEX** | Container chính (Android 10+) — chứa native lib, binary, service hệ thống; mount lúc boot sớm |
| **APK** | Format app truyền thống — cho module dạng app (vd MediaProvider) |

- Tên package: `com.android.*` (AOSP) / `com.google.android.*` (bản Google).

## Kênh update

1. **Google Play system updates** (hạ tầng Play Store)
2. **OTA của partner/OEM** — cho device không có Play

## Module tiêu biểu (30+)

`ART`, `Bluetooth`, `Wi-Fi`, `Network Stack`, `DNS Resolver`, `MediaProvider`, `Media Codecs`, `Timezone Data`, `Permission Controller`, `NNAPI`...

- Đã gặp trong chuỗi note: **ART là Mainline module** ([android-art.md](../art/android-art.md)) — update runtime qua Play không cần OTA.

🚗 *Liên hệ Automotive:* AAOS trên xe thường **không có GMS/Play đầy đủ** hoặc OEM tự quản kênh update → Mainline update đi qua **OTA của OEM** thay vì Play. Khi làm platform, cần biết image mình build bundle module version nào (`adb shell cmd apexservice` / `pm list packages --apex-only`) — bug ở Bluetooth/Media có khi fix bằng update APEX, không cần rebuild image.

## Câu hỏi tự kiểm tra

- [ ] Mainline giải quyết vấn đề gì, có từ Android nào?
- [ ] Vì sao module bắt buộc dùng CTS-guaranteed API + stable AIDL?
- [ ] APEX khác APK chỗ nào, loại nào cho native component?
- [ ] "Atomic install" nghĩa là gì?
- [ ] Device không có Play (như nhiều xe AAOS) nhận Mainline update qua đâu?

## 📚 Tài liệu

- 📖 [APEX file format](https://source.android.com/docs/core/ota/apex)
- 📖 [Danh sách module](https://source.android.com/docs/core/ota/modular-system#available-modules)

## Đọc tiếp

- [../android-architecture.md](../android-architecture.md) — vị trí Mainline trong stack
- [../art/android-art.md](../art/android-art.md) — ART là Mainline module
