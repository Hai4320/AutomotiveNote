---
title: NDK & CMake — build tầng native cho Android
phase: 2 — C/C++ & JNI
tags:
  - NDK
  - CMake
  - Build
  - Native
  - ABI
---

# NDK & CMake — build tầng native cho Android

> Thuộc **Phase 2 — C/C++ & JNI** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc **sau** [jni.md](jni.md) — JNI cho biết *viết* code C++ nối với Java thế nào; note này trả lời *biên dịch* code đó thành `.so` chạy được trên máy Android ra sao.
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## NDK là gì và vì sao không dùng gcc thường?

**NDK (Native Development Kit)** là bộ công cụ Google đóng gói để biên dịch C/C++ **cho Android**. Code C++ trên máy bạn (x86_64 Linux/Mac) không chạy được trên điện thoại/xe (CPU ARM, thư viện C khác). NDK là **bộ đồ nghề cross-compile**: build trên máy này, ra binary chạy máy kia.

Trong NDK có:
- **Toolchain Clang/LLVM** đã cấu hình sẵn cho từng kiến trúc Android.
- **Sysroot** — headers + thư viện hệ thống Android (`libc` = **Bionic**, không phải glibc như Linux desktop).
- **Build helper** — tích hợp với CMake và ndk-build.

```
Máy bạn (x86_64)          NDK cross-compile          Thiết bị (ARM64)
  code C++      ──────────────────────────────►  libnative.so
                Clang + sysroot Bionic
```

## CMake là gì và vai trò ở đây?

**CMake** là công cụ **mô tả cách build** (build system generator). Bạn viết `CMakeLists.txt` khai báo "file nào biên dịch thành thư viện gì, link với ai" — CMake sinh lệnh build thật. Android Gradle plugin gọi CMake với đúng cờ NDK để ra `.so`. Đây là con đường **chính thức, mặc định** hiện nay (ndk-build là cách cũ, ít dùng cho dự án mới).

Luồng tổng thể:

```
build.gradle ──gọi──► CMake ──đọc──► CMakeLists.txt ──build──► libnative.so ──nhét──► APK/AAB
   (khai báo)          (điều phối)     (mô tả build)            (mỗi ABI 1 file)
```

## `CMakeLists.txt` tối thiểu

```cmake
cmake_minimum_required(VERSION 3.22)
project(native-bridge)

# gom file .cpp thành 1 shared library → sinh ra libnative-bridge.so
add_library(native-bridge SHARED
    native-bridge.cpp
)

# tìm thư viện log của Android (in logcat từ C++)
find_library(log-lib log)

# link libnative-bridge với liblog
target_link_libraries(native-bridge ${log-lib})
```

- `SHARED` → `.so` (dynamic, load lúc runtime bằng `System.loadLibrary`). `STATIC` → `.a` (nhúng thẳng, hiếm dùng cho JNI).
- Tên trong `add_library` là `native-bridge` → file ra là `libnative-bridge.so` → Kotlin gọi `System.loadLibrary("native-bridge")` (không có `lib` và `.so`). Ba chỗ này **phải khớp**.

## Nối CMake vào Gradle

```gradle
// build.gradle (module app)
android {
    defaultConfig {
        ndk {
            // chỉ build ABI cần — mỗi ABI là 1 file .so, càng nhiều APK càng nặng
            abiFilters += listOf("arm64-v8a", "x86_64")
        }
    }
    externalNativeBuild {
        cmake {
            path = file("src/main/cpp/CMakeLists.txt")
            version = "3.22.1"
        }
    }
}
```

## ABI — vì sao 1 app có nhiều bản `.so`?

**ABI (Application Binary Interface)** = "chuẩn giao tiếp binary" của một kiến trúc CPU. Binary build cho ARM64 không chạy trên x86. Nên APK phải chứa **1 bản `.so` cho mỗi ABI** muốn hỗ trợ, gói trong `lib/<abi>/`:

| ABI | Dùng ở đâu |
|-----|-----------|
| `arm64-v8a` | 64-bit ARM — hầu hết điện thoại + **head unit xe** ngày nay. **Bắt buộc.** |
| `armeabi-v7a` | 32-bit ARM — thiết bị cũ |
| `x86_64` | Emulator, một số thiết bị Intel |
| `x86` | Emulator 32-bit cũ |

Build nhiều ABI → APK phình. Cách gọn: `abiFilters` chỉ build ABI cần, hoặc dùng **App Bundle (AAB)** để Play/hệ thống chỉ giao đúng ABI của máy.

## Debug native: logcat + symbol

In log từ C++ ra logcat:

```cpp
#include <android/log.h>
#define LOG_TAG "NativeBridge"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)

LOGI("giá trị = %d", x);   // xem bằng: adb logcat -s NativeBridge
```

Crash native in ra **tombstone** (địa chỉ thô). Muốn đọc được thành tên hàm + số dòng phải giữ **symbol chưa strip** và dùng `ndk-stack` / `addr2line`. Build `Debug` giữ symbol; build `Release` strip mất → nhớ lưu file symbol để giải mã crash sau này.

## 🚗 Liên hệ Automotive

- **`arm64-v8a` gần như là ABI của xe**: head unit AAOS chạy ARM64. Build native cho xe thường chỉ cần đúng ABI này — khỏi phình như app đa thiết bị.
- **AOSP không dùng CMake+Gradle mà dùng `Android.bp` + Soong**: khi code **nằm trong** cây nguồn AOSP (HAL, service hệ thống), build system là **Soong** đọc file `Android.bp` (cú pháp riêng, không phải CMake). CMake/Gradle là cho **app** cài lên máy; `Android.bp` là cho **thành phần hệ thống** build cùng OS. Cùng ra `.so` nhưng 2 đường build khác nhau — biết cả hai để không lẫn.
  ```
  // Android.bp — build native TRONG cây AOSP
  cc_library_shared {
      name: "libmy_hal",
      srcs: ["my_hal.cpp"],
      shared_libs: ["liblog", "libutils"],
      sanitize: { address: true },   // bật ASan ngay trong build
  }
  ```
- **Bật ASan lúc build là combo chuẩn**: nối tiếp [memory-management.md](memory-management.md) — HAL/daemon xe chạy suốt, phải bắt lỗi bộ nhớ lúc build/test. CMake thêm `-fsanitize=address`, `Android.bp` thêm `sanitize: { address: true }`.
- **Bionic ≠ glibc**: libc của Android (Bionic) nhỏ hơn, thiếu vài API POSIX của glibc. Thư viện C++ chép từ Linux desktop có thể không build thẳng cho Android — đây là lý do phải cross-compile bằng NDK chứ không dùng gcc host.

## Câu hỏi tự kiểm tra

- [ ] NDK là gì? Vì sao không dùng gcc/clang thường của máy để build cho Android?
- [ ] CMake đóng vai trò gì trong luồng build native Android?
- [ ] Trong `add_library(foo SHARED ...)`, file ra tên gì và Kotlin gọi `loadLibrary` với chuỗi nào? (bẫy khớp tên)
- [ ] ABI là gì? Vì sao 1 APK có thể chứa nhiều bản `.so`? Kể ABI bắt buộc cho xe.
- [ ] `abiFilters` / AAB giải quyết vấn đề gì?
- [ ] Build `Release` strip symbol thì đọc crash native kiểu gì?
- [ ] CMake+Gradle khác `Android.bp`+Soong ở chỗ nào — khi nào dùng cái nào?
- [ ] Vì sao thư viện C++ từ Linux desktop chưa chắc build thẳng cho Android?

## 📚 Tài liệu

- 📖 [NDK Guides (Google)](https://developer.android.com/ndk/guides) — docs chính thức, đọc phần Concepts trước
- 📖 [Add C/C++ to project (CMake + Gradle)](https://developer.android.com/studio/projects/add-native-code) — hướng dẫn nối CMake vào Android Studio
- 📖 [Android ABIs](https://developer.android.com/ndk/guides/abis) — bảng ABI đầy đủ, cách đóng gói `.so`
- 📖 [CMake documentation](https://cmake.org/cmake/help/latest/) — tra cú pháp CMakeLists
- 📖 [Soong / Android.bp](https://source.android.com/docs/setup/build) — build system của AOSP (dùng khi code nằm trong cây nguồn)
- 📖 [ndk-stack — giải mã crash native](https://developer.android.com/ndk/guides/ndk-stack)

## Đọc tiếp

- [README.md](README.md) — index Phase 2, thứ tự đọc các note tiếp theo
- [jni.md](jni.md) — code C++ nối Java (build ra `.so` bằng note này)
- [memory-management.md](memory-management.md) — ASan bật lúc build để bắt lỗi bộ nhớ native
