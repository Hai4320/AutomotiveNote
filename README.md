# AutomotiveNote 🚗

Ghi chú và lộ trình học **Android Automotive OS (AAOS)** — từ Android app đến system-level (AOSP, HAL, Vehicle HAL).

## Nội dung

| | |
|--|--|
| 📍 [Roadmap 9 phase](android-automotive-developer-roadmap.md) | Lộ trình cho Android dev chuyển sang Automotive — checkbox tiến độ, project mỗi phase, tài liệu từng mục |
| 📖 [Glossary](glossary.md) | Bảng thuật ngữ tra nhanh **dùng chung mọi phase** — Treble, partition, OEM/ODM, CAN/ECU, dexopt... |
| 📂 [Phase 1 — Android Internals](1_android_internal/README.md) | 23 note: architecture, Binder, system services, ART, boot, kernel/GKI, HAL/VHAL — kèm thứ tự đọc & tổng kết |
| 📂 [Phase 2 — C/C++ & JNI](2_c_and_jni/README.md) | Native layer: quản lý bộ nhớ, con trỏ, RAII, JNI, NDK — kèm thứ tự đọc & tổng kết |
| 📂 [Phase 3 — Embedded Linux & AOSP Build](3_embedded_linux_aosp/README.md) | 7 note: Linux (filesystem/permission/process/signal), bash & grep/sed/awk, kernel/driver/device tree, repo, Soong/`Android.bp`, build AAOS emulator, partition & fastboot |

## Lộ trình

```
Phase 0  Android App Review          ✅ đã có nền
Phase 1  Android Internals           ✅ notes xong
Phase 2  C/C++ & JNI                 ✅ notes xong
Phase 3  Embedded Linux & AOSP Build ✅ notes xong
Phase 4  Native & HAL
Phase 5  Automotive Domain (CAN/UDS)
Phase 6  Debugging & Profiling
Phase 7  AAOS & Cockpit Platform
Phase 8  Chuẩn ngành & Workflow      (song song)
```

## Cách dùng

Format note theo [TEMPLATE.md](TEMPLATE.md). Học theo thứ tự phase trong roadmap → tick checkbox `- [x]` từng mục → làm project 🎯 cuối phase rồi mới qua phase sau. Mỗi phase học xong sẽ có thư mục note riêng (`N_ten_phase/`) với README tổng kết.
