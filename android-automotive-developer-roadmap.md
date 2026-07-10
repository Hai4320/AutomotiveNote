---
title: Android Automotive Developer Roadmap
tags:
  - Android Automotive
  - AAOS
  - AOSP
  - Embedded Linux
  - Kotlin
  - Roadmap
---

# 🚗 Android Automotive Developer Roadmap (cho Android Dev)

Lộ trình dành cho người **đã là Android developer**, muốn chuyển sang Android Automotive — từ system-level (AOSP, HAL, Vehicle HAL) đến kiến thức ngành automotive (CAN bus, chuẩn an toàn, hạ tầng cockpit).

## Tổng quan lộ trình

| Phase | Nội dung | Thời gian gợi ý |
|-------|----------|-----------------|
| 0 | Android App Review (đã có nền — ôn nhanh) | 1–2 tuần |
| 1 | Android Internals (System Understanding) | 1–2 tháng |
| 2 | C/C++ & JNI cho Native Layer | 1–2 tháng |
| 3 | Embedded Linux & AOSP Build System | 1–2 tháng |
| 4 | Native & HAL Development | 2–3 tháng |
| 5 | Automotive Domain (CAN, UDS, SOME/IP) | 1 tháng |
| 6 | Debugging & Profiling Tools | 1 tháng |
| 7 | AAOS Specialization & Cockpit Platform | 2–3 tháng |
| 8 | Chuẩn ngành & Workflow (học song song) | 2–3 tuần |

```
[Đã có: Android App] → Internals → C/C++ & JNI → Embedded Linux & AOSP Build
        → HAL & Native → Automotive Domain → Debugging → AAOS & Cockpit
        (song song: ASPICE / ISO 26262 / Gerrit workflow)
```

> 💡 **Hai hướng nghề nghiệp:**
> - **Automotive App Developer** — chỉ cần Phase 0, 7 (mục Car App Library). Vào nghề nhanh.
> - **Platform / HAL / Framework Engineer** — cần toàn bộ roadmap. Đa số job automotive tại VN (outsource cho OEM/supplier) là dạng này.

---

## PHASE 0: Android App Review ✅ (đã có nền)

**Mục tiêu:** Ôn nhanh, lấp lỗ hổng nếu có. Không cần học lại từ đầu.

Checklist tự đánh giá — mục nào chưa chắc thì ôn lại:

- [ ] Activity/Fragment lifecycle, Intents, PendingIntent
- [ ] Services & Bound Services, BroadcastReceiver, ContentProvider
- [ ] Jetpack Compose + ViewModel + Flow
- [ ] Kotlin Coroutines & Flow nâng cao
- [ ] Permissions, Manifest, Gradle build types

**📚 Tài liệu:**
- 📖 [Android Basics with Compose](https://developer.android.com/courses/android-basics-compose/course) — course chính thức của Google
- 📖 [Kotlin Coroutines & Flow docs](https://kotlinlang.org/docs/coroutines-overview.html)

**✅ Kết quả:** Tự tin phần app — nền để so sánh khi học system side.

---

## PHASE 1: Android Internals (System Understanding)

**Mục tiêu:** Hiểu Android hoạt động *bên dưới* thế nào — nền tảng cho AAOS và HAL-level development. **Đây là bước nhảy lớn nhất từ app dev sang system dev.**

### Android System Architecture

- [ ] Các layer của Android OS (Kernel → HAL → Native Libraries → ART → Framework → Apps)
- [ ] Vai trò từng layer trong hệ thống
- [ ] Quan hệ giữa Java Framework & Native Components

**📚 Tài liệu:**
- 📖 [Android Architecture overview](https://source.android.com/docs/core/architecture) — docs chính thức AOSP

### Key Android Internals

- [ ] System Services: `ActivityManager`, `WindowManager`, `PackageManager`, `CarPropertyManager`
- [ ] **Binder IPC** — cách các process giao tiếp trong Android
- [ ] **Android Runtime (ART)** — cách Kotlin/Java được compile và chạy
- [ ] **Zygote Process** — tạo app process và preloading
- [ ] **Boot Process & Init System** — từ kernel đến system_server
- [ ] **AAOS Structure & Build System** — AOSP + car-specific layers
- [ ] Vai trò của Car Services và Vehicle HAL

**📚 Tài liệu:**
- 📖 [Android Runtime (ART)](https://source.android.com/docs/core/runtime) — docs chính thức
- 📖 [Android Init README](https://android.googlesource.com/platform/system/core/+/master/init/README.md) — init system từ source
- 📖 Sách **"Embedded Android"** — Karim Yaghmour (O'Reilly), kinh điển về internals
- 📖 Sách **"Android Internals"** — Jonathan Levin ([newandroidbook.com](http://newandroidbook.com/))

**🎯 Thực hành:** Dùng `dumpsys`, `ps`, đọc log boot trên emulator — map lý thuyết vào thực tế.

**✅ Kết quả:** Hiểu OS boot, chạy và giao tiếp nội bộ thế nào.

---

## PHASE 2: C/C++ & JNI cho Native Layer

**Mục tiêu:** Kotlin/Java đã có — phase này tập trung **C/C++**, ngôn ngữ của tầng HAL.

| Ngôn ngữ | Layer | Trạng thái |
|----------|-------|-----------|
| Kotlin | App | ✅ đã có |
| Java | Framework (Binder, AIDL) | ôn lại nếu cần |
| C/C++ | Native/HAL | ❗ trọng tâm phase này |

- [ ] C fundamentals: pointers, memory management, struct
- [ ] C++ cơ bản: class, RAII, smart pointers, STL
- [ ] JNI — gọi C/C++ từ Kotlin/Java và ngược lại
- [ ] NDK, CMakeLists cho native builds
- [ ] Java Binder & AIDL (ôn phần framework)

**📚 Tài liệu:**
- 📖 [learncpp.com](https://www.learncpp.com/) — học C++ từ zero, miễn phí, rất kỹ
- 📖 Sách **"The C Programming Language"** — K&R, kinh điển cho C
- 📖 [NDK Guides](https://developer.android.com/ndk/guides) — docs chính thức
- 📖 [JNI Tips](https://developer.android.com/training/articles/perf-jni) — best practices JNI từ Google
- 📖 [AIDL docs](https://developer.android.com/develop/background-work/services/aidl)

**🎯 Thực hành:** App Kotlin gọi native C++ qua JNI; viết 1 AIDL interface giữa 2 app.

**✅ Kết quả:** Di chuyển thoải mái giữa layer Kotlin/Java (framework) và C++ (HAL).

---

## PHASE 3: Embedded Linux & AOSP Build System 🆕

**Mục tiêu:** Lấp lỗ hổng lớn nhất của app dev — làm việc với Linux và build AOSP thật. Job platform nào cũng cần.

### Linux Fundamentals

- [ ] Shell scripting (bash): pipe, grep, sed, awk cơ bản
- [ ] File system, permissions, process, signals
- [ ] Kernel concepts: device tree, kernel module, driver là gì
- [ ] SELinux cơ bản (permissive/enforcing, sepolicy — nguồn lỗi kinh điển khi làm HAL)

**📚 Tài liệu:**
- 📖 [MIT Missing Semester](https://missing.csail.mit.edu/) — shell, git, tooling (kèm video bài giảng)
- 📖 [Linux Journey](https://linuxjourney.com/) — học Linux từ zero, miễn phí
- 📖 [Bootlin training materials](https://bootlin.com/docs/) — slides embedded Linux/kernel miễn phí, chuẩn ngành
- 📖 [SELinux for Android](https://source.android.com/docs/security/features/selinux) — docs chính thức

### AOSP Build System

- [ ] `repo` tool — sync source AOSP (hàng trăm GB)
- [ ] **Soong** (`Android.bp`) & Make (`Android.mk`) — hệ build của AOSP
- [ ] `lunch` targets, `m`, `mm`, `mmm`
- [ ] Build AAOS emulator image từ source
- [ ] Partition layout: system, vendor, product, odm
- [ ] Flash image vào thiết bị thật / bench board (nếu có)

**📚 Tài liệu:**
- 📖 [AOSP — Set up & download source](https://source.android.com/docs/setup/start) — docs chính thức, làm theo từng bước
- 📖 [repo command reference](https://source.android.com/docs/setup/reference/repo)
- 📖 [Soong build system](https://source.android.com/docs/setup/build) — Android.bp
- 📖 [Android partitions](https://source.android.com/docs/core/architecture/partitions)

**🎯 Thực hành:** Sync AOSP, build target `sdk_car_x86_64`, chạy AAOS emulator từ image tự build, sửa 1 dòng trong SystemUI rồi rebuild.

**✅ Kết quả:** Tự build, sửa và flash AOSP — kỹ năng bắt buộc của platform engineer.

---

## PHASE 4: Native & HAL Development

**Mục tiêu:** Hiểu cách Android giao tiếp với phần cứng xe.

### 1. HAL (Hardware Abstraction Layer)

- [ ] HAL là gì, mục đích
- [ ] Viết và tích hợp custom HAL
- [ ] Cấu trúc thư mục HAL trong AOSP (`/hardware/interfaces/`)

**📚 Tài liệu:**
- 📖 [HAL overview](https://source.android.com/docs/core/architecture/hal) — docs chính thức

### 2. HIDL & AIDL

- [ ] **HIDL** — dùng ở Android cũ (≤ Android 10)
- [ ] **AIDL** — cơ chế IPC mới cho HAL
- [ ] Define, compile và dùng AIDL interfaces
- [ ] Luồng giao tiếp: System Service ↔ HAL ↔ App

**📚 Tài liệu:**
- 📖 [AIDL for HALs](https://source.android.com/docs/core/architecture/aidl/aidl-hals) — docs chính thức
- 📖 [HIDL overview](https://source.android.com/docs/core/architecture/hidl) — cho code cũ

### 3. Vendor Interface & Project Treble

- [ ] Treble Architecture là gì
- [ ] Vendor partition separation
- [ ] VINTF (Vendor Interface Manifest)

**📚 Tài liệu:**
- 📖 [VINTF overview](https://source.android.com/docs/core/architecture/vintf) — docs chính thức

### 4. Car Service & Car API

- [ ] `CarPropertyManager` — đọc/ghi vehicle properties
- [ ] `CarInfoManager`, `CarSensorManager`, `CarAppFocusManager`
- [ ] Car Service như một System App trong AAOS

**📚 Tài liệu:**
- 📖 [Car API reference](https://developer.android.com/reference/android/car/Car)
- 📖 [CarPropertyManager](https://developer.android.com/reference/android/car/hardware/property/CarPropertyManager)

### 5. Android Properties System

- [ ] `getprop`, `setprop`
- [ ] Định nghĩa system và vendor properties

**📚 Tài liệu:**
- 📖 [Add system properties](https://source.android.com/docs/core/architecture/configuration/add-system-properties) — docs chính thức

**🎯 Thực hành:** Viết 1 AIDL HAL đơn giản, expose 1 property mới qua `CarPropertyManager` trên AAOS emulator (image tự build ở Phase 3).

**✅ Kết quả:** Hiểu software layer giao tiếp với ECU và sensor của xe thế nào.

---

## PHASE 5: Automotive Domain Knowledge 🆕

**Mục tiêu:** Kiến thức ngành xe — để làm việc được với team ECU/embedded, thứ Android dev thuần không có.

### In-Vehicle Networking

- [ ] **CAN / CAN-FD** — frame format, arbitration, DBC file
- [ ] **LIN** — mạng phụ giá rẻ (khái niệm)
- [ ] **Automotive Ethernet** & **SOME/IP** — xu hướng xe mới
- [ ] Đọc/phân tích CAN log (candump, SavvyCAN)

**📚 Tài liệu:**
- 📖 [CAN Bus tutorial — CSS Electronics](https://www.csselectronics.com/pages/can-bus-simple-intro-tutorial) — intro tốt nhất cho người mới
- 📖 [SocketCAN kernel docs](https://www.kernel.org/doc/html/latest/networking/can.html) — CAN trên Linux
- 📖 [SOME/IP official](https://some-ip.com/) — spec từ BMW
- 📖 [SavvyCAN](https://www.savvycan.com/) — tool phân tích CAN log, miễn phí

### Diagnostics

- [ ] **UDS (ISO 14229)** — dịch vụ chẩn đoán, DID, DTC
- [ ] **DoIP** — diagnostics over IP

**📚 Tài liệu:**
- 📖 [UDS tutorial — CSS Electronics](https://www.csselectronics.com/pages/uds-protocol-tutorial-unified-diagnostic-services)

### Kiến trúc ECU

- [ ] **AUTOSAR Classic vs Adaptive** — khái niệm, vị trí so với AAOS
- [ ] ECU, domain controller, zonal architecture là gì
- [ ] Vehicle HAL nhận data từ CAN thế nào (luồng CAN → VHAL → CarService → App)

**📚 Tài liệu:**
- 📖 [AUTOSAR official](https://www.autosar.org/) — spec chính thức

**🎯 Thực hành:** Dựng virtual CAN (`vcan`) trên Linux, gửi/nhận frame bằng `can-utils`, parse bằng DBC.

**✅ Kết quả:** Nói chuyện được với team ECU, hiểu data xe đi từ sensor đến app thế nào.

---

## PHASE 6: Debugging & Profiling Tools

**Mục tiêu:** Inspect, debug và profile cả app lẫn system-level component.

### Command-Line Tools

- [ ] **ADB** — connect, push, log, shell
- [ ] **Fastboot** — flash image, recovery mode
- [ ] **logcat** — system logs
- [ ] **dmesg** — kernel logs
- [ ] **Serial console** — debug khi board chưa boot được ADB (chuẩn khi làm bench)

**📚 Tài liệu:**
- 📖 [ADB docs](https://developer.android.com/tools/adb)
- 📖 [logcat docs](https://developer.android.com/tools/logcat)

### Profiling & Tracing

- [ ] **Perfetto / Systrace** — system performance tracing
- [ ] **Android Studio Profiler** — CPU, memory, network
- [ ] **GDB / LLDB** — debug native code
- [ ] **strace** — system call tracing

**📚 Tài liệu:**
- 📖 [Perfetto docs](https://perfetto.dev/docs/) — kèm tutorial trace Android
- 📖 [Android Studio Profiler](https://developer.android.com/studio/profile)
- 📖 [Debug native code](https://developer.android.com/studio/debug)

### System Diagnostics

- [ ] **dumpsys** — dump system service info
- [ ] **dumpstate** — device state snapshot
- [ ] **bugreport** — full issue capture (chuẩn trong Automotive debugging)

**📚 Tài liệu:**
- 📖 [dumpsys docs](https://developer.android.com/tools/dumpsys)
- 📖 [Capture & read bug reports](https://developer.android.com/studio/debug/bug-report)

**🎯 Thực hành:** Trace 1 lần boot bằng Perfetto, phân tích 1 bugreport thật, debug 1 crash native bằng LLDB.

**✅ Kết quả:** Troubleshoot được mọi thứ — từ app crash đến system-level issue.

---

## PHASE 7: AAOS Specialization & Cockpit Platform

**Mục tiêu:** Áp dụng toàn bộ kiến thức vào Android Automotive OS và hạ tầng cockpit thực tế.

### 1. AAOS Overview

- [ ] AAOS là gì, khác Android Auto thế nào
- [ ] Các layer AAOS: Framework → Car Service → Vehicle HAL
- [ ] OEM customization & branding (Volvo, Polestar, Stellantis)

**📚 Tài liệu:**
- 📖 [What is Android Automotive](https://source.android.com/docs/automotive/start/what_automotive) — docs chính thức

### 2. Automotive Application Development

- [ ] **Car App Library**
- [ ] Develop Media, Navigation, POI apps
- [ ] **CarContext**, **Template APIs**, **Car UX Guidelines**
- [ ] Voice Commands & Google Assistant integration
- [ ] Driver Distraction rules (UX bị giới hạn khi xe chạy)

**📚 Tài liệu:**
- 📖 [Build apps for cars](https://developer.android.com/training/cars) — docs chính thức, có codelab
- 📖 [Design for cars](https://developers.google.com/cars/design) — UX guidelines
- 📖 [Car App Library samples (GitHub)](https://github.com/android/car-samples) — code mẫu chính thức

### 3. Vehicle Integration

- [ ] **Vehicle HAL** — đọc/ghi car data
- [ ] Implement CarProperty values mới
- [ ] Luồng end-to-end: CAN → VHAL → CarService → App

**📚 Tài liệu:**
- 📖 [Vehicle HAL docs](https://source.android.com/docs/automotive/vhal) — docs chính thức
- 📖 [Vehicle properties](https://source.android.com/docs/automotive/vhal/properties)

### 4. OEM Customization & System UI

- [ ] Custom launchers
- [ ] SystemUI modifications cho automotive
- [ ] Runtime Resource Overlays (RRO) cho branding

**📚 Tài liệu:**
- 📖 [Runtime Resource Overlays](https://source.android.com/docs/core/runtime/rros) — docs chính thức
- 📖 [Car System UI customization](https://source.android.com/docs/automotive/hmi/car_ui) — Car UI library

### 5. Cockpit Platform thực tế 🆕

- [ ] **Hypervisor** — xe thật hiếm chạy AAOS trần: QNX/Hypervisor host + AAOS guest
- [ ] SoC phổ biến: Qualcomm SA8155/SA8295, NXP i.MX, Renesas R-Car
- [ ] **A/B (seamless) OTA update** — cách xe update phần mềm
- [ ] Instrument cluster vs IVI — phân vùng safety

**📚 Tài liệu:**
- 📖 [A/B (seamless) system updates](https://source.android.com/docs/core/ota/ab) — docs chính thức

**🎯 Project tổng kết:** Media app bằng Car App Library trên AAOS emulator + custom 1 vehicle property từ VHAL lên UI, mô phỏng data từ vcan.

**✅ Kết quả:** Sẵn sàng cho vai trò **Automotive App Developer**, **System Developer**, hoặc **AAOS Framework Engineer**.

---

## PHASE 8: Chuẩn ngành & Workflow 🆕 (học song song, mức khái niệm)

**Mục tiêu:** Những thứ phỏng vấn automotive hay hỏi và ngày nào đi làm cũng gặp. Không cần sâu — cần biết là gì và vì sao tồn tại.

### Chuẩn ngành

- [ ] **ASPICE** — quy trình phát triển phần mềm automotive (gần như bắt buộc ở supplier)
- [ ] **ISO 26262** — functional safety, các mức ASIL (A→D), QM
- [ ] **ISO 21434** & **UNECE R155/R156** — cybersecurity và OTA regulation

**📚 Tài liệu:**
- 📖 [Automotive SPICE official](https://vda-qmc.de/en/automotive-spice/) — spec chính thức từ VDA
- 📖 [ISO 26262 — Wikipedia](https://en.wikipedia.org/wiki/ISO_26262) — overview đủ dùng cho khái niệm

### Workflow đặc thù

- [ ] **Gerrit** — code review của AOSP (thay cho GitHub PR)
- [ ] Làm việc với requirement/spec từ OEM (DOORS, Polarion — biết khái niệm)
- [ ] Chu kỳ build dài, test trên bench/HIL thay vì chỉ emulator

**📚 Tài liệu:**
- 📖 [Gerrit walkthrough](https://gerrit-review.googlesource.com/Documentation/intro-gerrit-walkthrough.html) — hướng dẫn chính thức
- 📖 [Contribute to AOSP](https://source.android.com/docs/setup/contribute) — workflow Gerrit thực tế

### Testing kiểu automotive

- [ ] **VTS / CTS** — test suite của AOSP
- [ ] **HIL / SIL** — hardware/software-in-the-loop là gì

**📚 Tài liệu:**
- 📖 [VTS (Vendor Test Suite)](https://source.android.com/docs/core/tests/vts) — docs chính thức

**✅ Kết quả:** Qua vòng phỏng vấn automotive, không bỡ ngỡ với quy trình OEM/supplier.

---

## 📚 Resources tổng hợp

- [Android Automotive OS docs](https://source.android.com/docs/automotive)
- [Car App Library](https://developer.android.com/training/cars)
- [AOSP source & build](https://source.android.com/docs/setup)
- [Vehicle HAL reference](https://source.android.com/docs/automotive/vhal)
- [Perfetto docs](https://perfetto.dev/)
- [can-utils (SocketCAN)](https://github.com/linux-can/can-utils)
- [CSS Electronics — CAN/UDS tutorials](https://www.csselectronics.com/pages/can-bus-simple-intro-tutorial)
- [ASPICE overview](https://vda-qmc.de/en/automotive-spice/)
