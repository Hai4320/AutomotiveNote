---
title: Quan hệ Java Framework & Native Components
phase: 1 — Android Internals
tags:
  - AOSP
  - JNI
  - Binder
  - Framework
  - Native
---

# Quan hệ Java Framework & Native Components

> Thuộc **Phase 1 — Android Internals** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc sau [android-architecture.md](android-architecture.md).

## Ý chính

**Java framework là bộ mặt API, native components là nơi làm việc nặng.** Hai bên nối nhau qua đúng 2 cây cầu:

| Cầu | Khi nào | Cơ chế |
|-----|---------|--------|
| **JNI** | Native lib **cùng process** | Gọi hàm trực tiếp |
| **Binder IPC** | Native service **process khác** | Proxy → binder driver (kernel) → process đích |

```
Java Framework (android.*, system_server)
   │
   ├── JNI ──────────────▶ Native lib cùng process
   │                       (libandroid_runtime, libmedia...)
   │
   └── Binder IPC ───────▶ Native service process khác
                           (SurfaceFlinger, MediaServer, HAL services)
```

## Cầu 1: JNI — cùng process

- Nhiều class framework chỉ là "vỏ" Java: logic thật ở C++.
- Method đánh dấu `native`, implement trong `libandroid_runtime.so`.
- Ví dụ: `Bitmap`, `Canvas`, `MediaPlayer`.
- Ưu: nhanh (không IPC). Nhược: crash ở native kéo chết cả process.

```java
public final class Bitmap {
    private static native long nativeCreate(...);  // implement ở C++
}
```

## Cầu 2: Binder IPC — khác process

- Logic nằm ở process khác → gọi qua **proxy (AIDL stub)** → binder driver trong kernel → process đích.
- Ví dụ:
  - App gọi `WindowManager` (Java, system_server) → render thật do **SurfaceFlinger** (native, process riêng)
  - `SensorManager` (Java) → JNI → `libsensorservice` → sensor HAL
- Ưu: cô lập lỗi (native service crash không kéo framework), có permission check tại boundary.

## Hai chiều

Native gọi ngược lên Java qua JNI (`CallVoidMethod`, `CallObjectMethod`...):
- HAL callback đẩy event lên framework
- Zygote fork xong gọi vào `ActivityThread.main()` (Java) từ native

## 🚗 Ví dụ AAOS — luồng đọc tốc độ xe

```
App (Kotlin)
  → CarPropertyManager (Java)          ← Java framework
  → Binder → CarService (Java, system app)
  → Binder AIDL → VHAL service (C++)   ← native component
  → CAN bus → ECU
```

Cả 2 cầu xuất hiện: Binder giữa App ↔ CarService ↔ VHAL; JNI bên trong các service khi cần gọi native lib.

## Vì sao roadmap bắt học C++/JNI + Binder/AIDL (Phase 2)

- Làm platform = đứng giữa 2 thế giới: sửa framework Java, debug xuống native service.
- Stack trace crash thường xuyên nhảy Java ↔ native — không đọc được cả 2 bên thì không debug nổi.

## Câu hỏi tự kiểm tra

- [ ] JNI vs Binder — khi nào dùng cầu nào?
- [ ] Vì sao `Bitmap` là class Java nhưng logic ở C++? Trade-off?
- [ ] SurfaceFlinger là Java hay native? App nói chuyện với nó qua gì?
- [ ] Native gọi ngược lên Java bằng cách nào? Ví dụ?
- [ ] Trong luồng CarPropertyManager → VHAL, chỗ nào là Binder, chỗ nào là JNI?

## 📚 Tài liệu

- 📖 [JNI Tips](https://developer.android.com/training/articles/perf-jni) — best practices chính thức
- 📖 [JNI specification (Oracle)](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/jniTOC.html) — spec gốc
- 📖 [NDK Guides](https://developer.android.com/ndk/guides)
- 📖 [Binder IPC — AOSP docs](https://source.android.com/docs/core/architecture/hidl/binder-ipc)

## Đọc tiếp

- [android-architecture.md](android-architecture.md) — note trong repo
- [hal/android-hal.md](hal/android-hal.md) — note trong repo
- [hal/aidl-hals.md](hal/aidl-hals.md) — note trong repo
