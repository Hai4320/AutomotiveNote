---
title: Kiến trúc Android (AOSP Architecture)
source: https://source.android.com/docs/core/architecture?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - Architecture
  - HAL
  - Kernel
---

# Kiến trúc Android (AOSP)

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/architecture](https://source.android.com/docs/core/architecture?hl=vi)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../android-automotive-developer-roadmap.md).

**AOSP architecture** là cách Android tổ chức toàn bộ hệ điều hành thành nhiều **tầng (layer) xếp chồng**, từ app trên cùng xuống Linux kernel dưới đáy. Là app dev bạn quen chỉ nhìn thấy tầng trên (gọi API `android.*`), nhưng bên dưới còn ART, HAL, native libs và kernel — hiểu chúng mới sửa được platform.

Vì sao phải phân tầng thay vì viết một khối liền? Để mỗi phần đổi độc lập mà không kéo theo phần khác: vendor thay driver phần cứng không cần Google sửa framework, framework đổi không cần vá lại kernel. Nguyên tắc là **mỗi layer chỉ nói chuyện với layer kề nó** — app gọi framework, framework gọi HAL, HAL gọi kernel — chứ không nhảy cóc. Giống như công ty có phòng ban: phòng kinh doanh không tự xuống xưởng sửa máy, mà đặt yêu cầu qua phòng kế bên. Cách tách bạch này chính là nền cho Project Treble, HAL và GKI ở các mục sau.

## Sơ đồ các layer

```
┌─────────────────────────────────────────┐
│  Apps (public API / privileged / OEM)   │
├─────────────────────────────────────────┤
│  Android Framework + System Services    │
│  (system_server, SurfaceFlinger, ...)   │
├─────────────────────────────────────────┤
│  Android Runtime (ART)                  │
├─────────────────────────────────────────┤
│  HAL (Hardware Abstraction Layer)       │
├──────────────────┬──────────────────────┤
│  Native libs     │  Native daemons      │
│  (libc, libbinder│  (init, healthd,     │
│   libselinux...) │   logd...)           │
├──────────────────┴──────────────────────┤
│  Linux Kernel (GKI + vendor modules)    │
└─────────────────────────────────────────┘
```

## Các layer chi tiết

### 1. Application Layer

3 loại app, khác nhau ở mức truy cập:

| Loại | Truy cập |
|------|----------|
| App thường | Public API (developer.android.com) |
| Privileged app | Public API + System API |
| OEM app | Truy cập thẳng framework implementation |

### 2. Framework & System Services

- **Android Framework** — classes, interfaces, pre-compiled code mà app build lên trên.
- **System services** — process nền xử lý chức năng lõi: `system_server` (chứa ActivityManager, WindowManager, PackageManager...), `SurfaceFlinger` (compositing UI), `MediaService`.
- 🚗 *Liên hệ Automotive:* Car Service (`CarPropertyManager`...) cũng là system service, chạy như system app trong AAOS.

### 3. Android Runtime (ART)

- Dịch application bytecode (DEX) → processor-specific instructions.
- Kotlin/Java compile → DEX → ART chạy (AOT + JIT compilation).

### 4. HAL (Hardware Abstraction Layer)

- Interface chuẩn để **hardware vendor implement** — Android không phụ thuộc chi tiết driver tầng thấp.
- Framework gọi HAL qua interface đã định nghĩa (AIDL/HIDL), vendor tự lo phần implementation.
- 🚗 *Liên hệ Automotive:* **Vehicle HAL (VHAL)** chính là 1 HAL — cầu nối giữa CarService và phần cứng xe (CAN bus, ECU).

### 5. Native Layer

Tương tác thẳng với kernel, **không qua HAL**:

- **Native libraries:** `libc`, `liblog`, `libbinder`, `libselinux`
- **Native daemons:** `init` (process đầu tiên sau kernel), `healthd`, `logd` — *daemon* = process chạy nền không UI.

### 6. Kernel

- Giao tiếp trực tiếp với phần cứng.
- Được module hóa: phần **hardware-independent** (GKI) + **vendor-specific modules**.

## Khái niệm quan trọng

### Project Treble (Android 8+)
- Kiến trúc tách **vendor** (code phần cứng) khỏi **framework** (code Google) qua interface ổn định → 2 bên update độc lập.
- Lý do tồn tại của HAL binderized, VINTF, GKI... trong các note sau — đọc kỹ ở [glossary.md](../glossary.md) trước khi vào chuỗi HAL/kernel.

### HIDL / AIDL
- Cơ chế IPC giữa framework ↔ HAL.
- Timeline: **HIDL** là chuẩn HAL Android 8–10 → **AIDL** dùng cho HAL mới từ Android 11 → HIDL chính thức deprecated từ Android 13 (HAL cũ vẫn chạy được).

### Modular System Components (Mainline)
- Các component update được qua Google Play, không cần OTA full: Media, Networking, NFC...
- Mục đích: vá lỗi/bảo mật nhanh, không chờ OEM.

### GKI (Generic Kernel Image)
- Kernel hardware-agnostic dùng chung — tách kernel core khỏi vendor driver.
- Vendor driver load dạng kernel module riêng → giảm fragmentation.

### Mức tương thích (Compatibility)

| Mức | Yêu cầu |
|-----|---------|
| **AOSP Compatibility** | Đạt CDD (Compatibility Definition Document — định nghĩa "thế nào là thiết bị Android hợp lệ") |
| **Android Compatibility** | CDD + VSR (Vendor Software Requirements) + pass VTS (Vendor Test Suite — test tầng vendor/HAL) & CTS (Compatibility Test Suite — test tầng app/API) |

Chi tiết 4 chữ viết tắt này: [glossary.md](../glossary.md).

## Câu hỏi tự kiểm tra

- [ ] App thường vs privileged app vs OEM app khác nhau chỗ nào?
- [ ] Vì sao cần HAL thay vì framework gọi thẳng driver?
- [ ] `system_server` chứa những service nào?
- [ ] GKI giải quyết vấn đề gì?
- [ ] VHAL nằm ở layer nào trong sơ đồ trên?

## 📚 Tài liệu

- 📖 [HAL overview](https://source.android.com/docs/core/architecture/hal)
- 📖 [Android Runtime (ART)](https://source.android.com/docs/core/runtime)
- 📖 [Kernel overview](https://source.android.com/docs/core/architecture/kernel)
- 📖 [Modular System Components](https://source.android.com/docs/core/ota/modular-system)

## Đọc tiếp

- [java-native-relationship.md](java-native-relationship.md) — 2 cầu nối giữa các layer
- [hal/android-hal.md](hal/android-hal.md) — layer HAL chi tiết
- [kernel/android-kernel.md](kernel/android-kernel.md) — layer kernel chi tiết
- [art/android-art.md](art/android-art.md) — layer runtime chi tiết
