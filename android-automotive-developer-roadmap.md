---
title: Android Automotive Developer Roadmap
tags:
  - Android Automotive
  - AAOS
  - Kotlin
  - Roadmap
---

# 🚗 Android Automotive Developer Roadmap

Lộ trình từng bước từ Android cơ bản đến system-level development (AOSP, HAL, Vehicle HAL) để trở thành Android Automotive Developer.

## Tổng quan lộ trình

| Phase | Nội dung | Thời gian gợi ý |
|-------|----------|-----------------|
| 1 | Android Application Fundamentals | 2–3 tháng |
| 2 | Android Internals (System Understanding) | 1–2 tháng |
| 3 | Programming Languages for Automotive | 1–2 tháng |
| 4 | Native & HAL Development | 2–3 tháng |
| 5 | Debugging & Profiling Tools | 1 tháng |
| 6 | Automotive Specialization (AAOS) | 2–3 tháng |

```
App Fundamentals → Android Internals → Languages (Kotlin/Java/C++)
        → HAL & Native → Debugging Tools → AAOS Specialization
```

---

## PHASE 1: Android Application Fundamentals

**Mục tiêu:** Nền tảng vững về Android app development trước khi vào Automotive.

### Android Basics

- [ ] Android Application Structure (Manifest, Gradle, Resource folders, Build types)
- [ ] Activities, Fragments & Activity Lifecycle
- [ ] Activity Embedding (multi-window layouts)
- [ ] Intents (explicit & implicit), PendingIntent
- [ ] Services & Bound Services
- [ ] BroadcastReceiver & ContentProvider
- [ ] Permissions & App Manifest setup

### UI Layer

- [ ] Jetpack Compose (modern UI toolkit)
- [ ] Composable functions & State management
- [ ] Layouts & Navigation
- [ ] Material 3 theming
- [ ] Compose + ViewModel + Flow integration

**🎯 Project thực hành:** Xây 1 app hoàn chỉnh (vd: note app) dùng Compose + ViewModel + Flow, có Service và BroadcastReceiver.

**✅ Kết quả:** Build được app Android hoàn chỉnh, hiểu cách từng component tương tác với hệ thống.

---

## PHASE 2: Android Internals (System Understanding)

**Mục tiêu:** Hiểu Android hoạt động *bên dưới* thế nào — nền tảng cho AAOS và HAL-level development.

### Android System Architecture

- [ ] Các layer của Android OS (Kernel → HAL → Native Libraries → ART → Framework → Apps)
- [ ] Vai trò từng layer trong hệ thống
- [ ] Quan hệ giữa Java Framework & Native Components

### Key Android Internals

- [ ] System Services: `ActivityManager`, `WindowManager`, `PackageManager`, `CarPropertyManager`
- [ ] **Binder IPC** — cách các process giao tiếp trong Android
- [ ] **Android Runtime (ART)** — cách Kotlin/Java được compile và chạy
- [ ] **Zygote Process** — tạo app process và preloading
- [ ] **Boot Process & Init System** — từ kernel đến system_server
- [ ] **AAOS Structure & Build System** — AOSP + car-specific layers
- [ ] Vai trò của Car Services và Vehicle HAL

**🎯 Thực hành:** Build AOSP emulator image, đọc log boot process, dùng `dumpsys` khám phá system services.

**✅ Kết quả:** Hiểu OS boot, chạy và giao tiếp nội bộ thế nào — bắt buộc cho vai trò Automotive system.

---

## PHASE 3: Programming Languages for Automotive

**Mục tiêu:** Thành thạo 3 ngôn ngữ chính ở các layer khác nhau.

| Ngôn ngữ | Layer | Dùng cho |
|----------|-------|----------|
| Kotlin | App | Automotive apps, UI |
| Java | Framework | System services, Binder, AIDL |
| C/C++ | Native/HAL | HAL, Vehicle HAL, native libs |

- [ ] Kotlin nâng cao: Coroutines, Flow, DSLs
- [ ] Java cho framework work: Binder, AIDL
- [ ] C/C++ fundamentals (pointers, memory, build)
- [ ] JNI — gọi C/C++ từ Java/Kotlin
- [ ] Android.mk & CMakeLists cho native builds

**🎯 Thực hành:** Viết 1 app Kotlin gọi native C++ function qua JNI.

**✅ Kết quả:** Di chuyển thoải mái giữa layer Java (framework) và C++ (HAL).

---

## PHASE 4: Native & HAL Development

**Mục tiêu:** Hiểu cách Android giao tiếp với phần cứng xe.

### 1. HAL (Hardware Abstraction Layer)

- [ ] HAL là gì, mục đích
- [ ] Viết và tích hợp custom HAL
- [ ] Cấu trúc thư mục HAL trong AOSP (`/hardware/interfaces/`)

### 2. HIDL & AIDL

- [ ] **HIDL** — dùng ở Android cũ (≤ Android 10)
- [ ] **AIDL** — cơ chế IPC mới cho HAL
- [ ] Define, compile và dùng AIDL interfaces
- [ ] Luồng giao tiếp: System Service ↔ HAL ↔ App

### 3. Vendor Interface & Project Treble

- [ ] Treble Architecture là gì
- [ ] Vendor partition separation
- [ ] VINTF (Vendor Interface Manifest)

### 4. Car Service & Car API

- [ ] `CarPropertyManager` — đọc/ghi vehicle properties
- [ ] `CarInfoManager`, `CarSensorManager`, `CarAppFocusManager`
- [ ] Car Service như một System App trong AAOS

### 5. Android Properties System

- [ ] `getprop`, `setprop`
- [ ] Định nghĩa system và vendor properties

**🎯 Thực hành:** Viết 1 AIDL HAL đơn giản, expose 1 property mới qua `CarPropertyManager` trên AAOS emulator.

**✅ Kết quả:** Hiểu software layer giao tiếp với ECU và sensor của xe thế nào.

---

## PHASE 5: Debugging & Profiling Tools

**Mục tiêu:** Inspect, debug và profile cả app lẫn system-level component.

### Command-Line Tools

- [ ] **ADB** — connect, push, log, shell
- [ ] **Fastboot** — flash image, recovery mode
- [ ] **logcat** — system logs
- [ ] **dmesg** — kernel logs

### Profiling & Tracing

- [ ] **Perfetto / Systrace** — system performance tracing
- [ ] **Android Studio Profiler** — CPU, memory, network
- [ ] **GDB / LLDB** — debug native code
- [ ] **strace** — system call tracing

### System Diagnostics

- [ ] **dumpsys** — dump system service info
- [ ] **dumpstate** — device state snapshot
- [ ] **bugreport** — full issue capture (chuẩn trong Automotive debugging)

**🎯 Thực hành:** Trace 1 lần boot bằng Perfetto, phân tích 1 bugreport thật.

**✅ Kết quả:** Troubleshoot được mọi thứ — từ app crash đến system-level issue.

---

## PHASE 6: Automotive Specialization (AAOS)

**Mục tiêu:** Áp dụng toàn bộ kiến thức vào Android Automotive OS.

### 1. AAOS Overview

- [ ] AAOS là gì, khác Android Auto thế nào
- [ ] Các layer AAOS: Framework → Car Service → Vehicle HAL
- [ ] OEM customization & branding (Volvo, Polestar, Stellantis)

### 2. Automotive Application Development

- [ ] **Car App Library**
- [ ] Develop Media, Navigation, POI apps
- [ ] **CarContext**, **Template APIs**, **Car UX Guidelines**
- [ ] Voice Commands & Google Assistant integration

### 3. Vehicle Integration

- [ ] **Vehicle HAL** — đọc/ghi car data
- [ ] **CAN Bus / ECU communication**
- [ ] Implement CarProperty values mới

### 4. OEM Customization & System UI

- [ ] Custom launchers
- [ ] SystemUI modifications cho automotive
- [ ] Runtime Resource Overlays (RRO) cho branding

**🎯 Project tổng kết:** Build 1 media app bằng Car App Library chạy trên AAOS emulator + custom 1 vehicle property từ VHAL lên UI.

**✅ Kết quả:** Sẵn sàng cho vai trò **Automotive App Developer**, **System Developer**, hoặc **AAOS Framework Engineer**.

---

## 📚 Resources

- [Android Automotive OS docs](https://source.android.com/docs/automotive)
- [Car App Library](https://developer.android.com/training/cars)
- [AOSP source](https://source.android.com/)
- [Vehicle HAL reference](https://source.android.com/docs/automotive/vhal)
- [Perfetto docs](https://perfetto.dev/)
