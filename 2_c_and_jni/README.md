# Phase 2 — C/C++ & JNI: Tổng kết

> Phase 2 trong [roadmap](../android-automotive-developer-roadmap.md). Mục tiêu: viết được code tầng **native** — C/C++ (ngôn ngữ của HAL), gọi qua **JNI** từ Kotlin/Java, build bằng **NDK/CMake**. **Học xong khi trả lời hết câu hỏi tự kiểm tra trong từng note + gọi được C++ từ app qua JNI.**

## Thứ tự đọc

> 📖 [glossary.md](../glossary.md) — bảng thuật ngữ dùng chung mọi phase. Gặp từ lạ thì tra.

### 1. C/C++ nền tảng
1. [memory-management.md](memory-management.md) — 4 vùng bộ nhớ, stack/heap, malloc/free, RAII + smart pointer, 5 lỗi kinh điển
2. [pointers-struct.md](pointers-struct.md) — `&`/`*`, số học con trỏ, function pointer, struct/union, padding/alignment; nền của mọi API C
3. [cpp-basics.md](cpp-basics.md) — class + destructor, reference vs pointer, STL (vector/map/string), template, const/namespace

### 2. Cầu nối Java ↔ native
4. [jni.md](jni.md) — JNIEnv/jobject, quy ước tên `Java_...`, type signature, local/global ref, RegisterNatives, callback xuyên thread
5. [jni-tips.md](jni-tips.md) — cạm bẫy thực tế: cache ID, FindClass sai classloader, exception check, modified UTF-8, mảng region, jlong con trỏ

### 3. Build tầng native
6. [ndk-cmake.md](ndk-cmake.md) — NDK cross-compile, CMakeLists, nối Gradle, ABI, logcat/symbol, Android.bp vs CMake

### 4. Ôn framework
7. [java-binder-aidl.md](java-binder-aidl.md) — IPC tầng framework: khai báo `.aidl`, Stub/Proxy, bindService, `in`/`out`/`oneway`, threading, stable AIDL cho HAL

## Ý phải nhớ khi rời Phase 2

1. Native **không có GC** — ai cấp phát người đó tự trả; leak/dangling nổ lúc runtime, không lúc compile.
2. Mặc định dùng stack; lên heap chỉ khi cần sống lâu hơn scope. Mỗi `new`/`malloc` khớp đúng 1 `delete`/`free`.
3. C++ hiện đại: **RAII + smart pointer** (`unique_ptr` mặc định), gần như không `new`/`delete` tay.
4. Leak trên HAL xe nguy hiểm hơn app điện thoại — daemon chạy suốt, không reboot cho nhanh.

## 🎯 Thực hành khép phase (từ roadmap)

App Kotlin gọi native C++ qua JNI; viết 1 AIDL interface giữa 2 app. Bật ASan khi build để bắt lỗi bộ nhớ.

## Tiếp theo → [Phase 3: Embedded Linux & AOSP Build](../android-automotive-developer-roadmap.md#phase-3-embedded-linux--aosp-build-system-)
