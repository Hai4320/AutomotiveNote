---
title: HAL (Hardware Abstraction Layer)
source: https://source.android.com/docs/core/architecture/hal?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - HAL
  - AIDL
  - HIDL
  - Binder
---

# HAL (Hardware Abstraction Layer)

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/architecture/hal](https://source.android.com/docs/core/architecture/hal?hl=vi)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../../android-automotive-developer-roadmap.md). Đọc sau [android-architecture.md](../android-architecture.md).

## HAL là gì?

HAL (Hardware Abstraction Layer) là **lớp trung gian giữa Android framework và phần cứng**. Nó định nghĩa một bộ *interface chuẩn* mô tả "phần cứng này làm được gì" (đọc tốc độ xe, bật đèn, chụp ảnh...) mà **không nói phần cứng làm điều đó ra sao**. Google định nghĩa sẵn interface; hardware vendor có nhiệm vụ **điền phần implement** phía dưới cho đúng chip/board của mình. Có thể hình dung giống một `interface` trong Java: framework cầm cái interface và gọi method, còn vendor viết class implement thực tế.

Vì sao framework không gọi thẳng driver? Vì mỗi hãng phần cứng có driver, chip, cách giao tiếp khác nhau. Nếu framework gọi trực tiếp thì mỗi lần đổi phần cứng phải sửa lại framework — không thể scale cho hàng nghìn thiết bị của hàng trăm hãng. HAL cắt đứt sự phụ thuộc đó: framework chỉ biết **interface đứng yên**, còn phần implement bên dưới thay đổi tùy vendor. Đây cũng chính là nền tảng của Project Treble — tách phần Google maintain (framework) khỏi phần vendor maintain (HAL) để hai bên update độc lập.

Với người quen làm app: bạn gọi `CameraManager` mà không cần biết cảm biến Sony hay Samsung — HAL là lớp làm được điều tương tự ở tầng system. Framework gọi qua interface, vendor "điền vào chỗ trống" của interface có sẵn, và không bên nào phải biết chi tiết của bên kia.

```
Framework / System Service
        │  gọi qua interface (AIDL/HIDL)
        ▼
   HAL service (process riêng, chạy ở vendor)
        │
        ▼
   Driver / Kernel / Hardware
```

## 3 thành phần

| Thành phần | Vai trò |
|-----------|---------|
| **HAL client** | Process truy cập dịch vụ (framework, system service) |
| **HAL interface** | Chuẩn giao tiếp — định nghĩa bằng AIDL/HIDL |
| **HAL service** | Code hardware-specific do vendor viết |

⚠️ Quy tắc: mọi HAL bắt buộc nằm ở **vendor partition**, khai báo theo **compatibility matrix** (VINTF).

## Các loại HAL

### 1. Binderized HAL (chuẩn từ Android 8+)

- Chạy trong **process riêng**, giao tiếp qua **Binder IPC**.
- Đăng ký với service manager để client tìm thấy.
- Framework crash không kéo HAL chết theo và ngược lại — cô lập lỗi.

### 2. Same-Process (SP) HAL

- Chạy **trong process của client** (không qua IPC) — vì hiệu năng.
- Danh sách **giới hạn, Google kiểm soát**: OpenGL, Vulkan, stable C mapper libraries.

### 3. Wrapped HAL (legacy)

- HAL viết trước Android 8 (legacy `.so` libhardware).
- Được **bọc trong AIDL/HIDL wrapper** để chạy trong kiến trúc mới.

## AIDL vs HIDL

| | AIDL | HIDL |
|--|------|------|
| Trạng thái | ✅ Dùng cho HAL mới từ Android 11 | ❌ Chuẩn cũ Android 8–10, deprecated từ Android 13 |
| HAL mới | Bắt buộc viết bằng AIDL | Không dùng cho HAL mới |
| HAL cũ | — | Vẫn được hỗ trợ chạy |

Timeline thống nhất: HIDL chuẩn 8–10 → AIDL cho HAL từ 11 → HIDL deprecated 13.

## Vị trí trong source tree

- Interface chuẩn: `/hardware/interfaces/` (vd `/hardware/interfaces/automotive/vehicle/`)
- Legacy libhardware: `/hardware/libhardware/`

🚗 *Liên hệ Automotive:* **VHAL là 1 binderized AIDL HAL** — interface tại `/hardware/interfaces/automotive/vehicle/aidl/`. CarService (client) gọi VHAL service qua Binder; VHAL service do vendor implement, nói chuyện với CAN bus/ECU. Đây chính là pattern sẽ code ở Phase 4.

## Câu hỏi tự kiểm tra

- [ ] Vì sao framework không gọi thẳng driver mà phải qua HAL?
- [ ] Binderized vs Same-Process HAL — khác gì, khi nào dùng SP?
- [ ] Wrapped HAL giải quyết vấn đề gì?
- [ ] HAL mới từ Android 13 viết bằng gì? HIDL còn dùng được không?
- [ ] HAL bắt buộc nằm partition nào, khai báo qua cơ chế gì?

## 📚 Tài liệu

- 📖 [AIDL for HALs](https://source.android.com/docs/core/architecture/aidl/aidl-hals)
- 📖 [HIDL overview](https://source.android.com/docs/core/architecture/hidl)
- 📖 [VINTF / compatibility matrix](https://source.android.com/docs/core/architecture/vintf)
- 📖 [Vehicle HAL](https://source.android.com/docs/automotive/vhal)

## Đọc tiếp

- [aidl-hals.md](aidl-hals.md) — viết HAL bằng AIDL
- [vintf-objects.md](vintf-objects.md) — khai báo HAL với VINTF
- [android-vhal.md](android-vhal.md) — HAL quan trọng nhất của AAOS
