---
title: Soong & Make — hệ build AOSP (Android.bp, lunch, m)
source: https://source.android.com/docs/setup/build
phase: 3 — Embedded Linux & AOSP Build
tags:
  - AOSP
  - Soong
  - Android.bp
  - Make
  - Build
---

# Soong & Make — hệ build AOSP (Android.bp, lunch, m)

> Thuộc **Phase 3 — Embedded Linux & AOSP Build** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc **sau** [repo-tool.md](repo-tool.md) (phải có source) và **trước** [build-aaos-image.md](build-aaos-image.md) (dùng chính lệnh ở đây để ra image). Nối với [ndk-cmake.md](../2_c_and_jni/ndk-cmake.md) — Phase 2 build native ngoài cây bằng CMake, đây là build **trong cây** AOSP bằng Soong.
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## Ba hệ build, đừng lẫn

App dev quen **Gradle**. Trong AOSP có ba thứ khác:

| Hệ | File khai báo | Dùng cho |
|----|---------------|----------|
| **Soong** | `Android.bp` | Chuẩn hiện tại của AOSP — mọi module mới |
| **Make (Kati)** | `Android.mk` | Cũ, đang bị thay dần bởi Soong; vẫn còn nhiều |
| **CMake** | `CMakeLists.txt` | Build native **ngoài** cây (app NDK) — [ndk-cmake.md](../2_c_and_jni/ndk-cmake.md) |

Note này lo **Soong + Make** (build **trong** cây AOSP). `Android.bp` cho AOSP giống `build.gradle` cho app — khai báo module, không viết lệnh build.

## Android.bp — cú pháp khai báo (không phải script)

`Android.bp` là **khai báo thuần** (JSON-like), **không có logic/if/loop** — cố ý, để build phân tích được và tái lập:

```
cc_binary {                          // build một binary C/C++
    name: "android.hardware.automotive.vehicle@2.0-service",
    srcs: ["VehicleService.cpp"],
    shared_libs: [
        "libbase",
        "libbinder_ndk",
        "android.hardware.automotive.vehicle-V2-ndk",
    ],
    init_rc: ["vhal.rc"],            // file init.rc đi kèm service
    vendor: true,                    // cài vào /vendor (HAL bắt buộc — Treble)
}
```

Loại module hay gặp:

| Module type | Ra cái gì |
|-------------|-----------|
| `cc_binary` | Binary C/C++ (vd service HAL) |
| `cc_library_shared` / `_static` | `.so` / `.a` |
| `android_app` | APK |
| `java_library` | `.jar` |
| `aidl_interface` | Sinh code từ `.aidl` (HAL AIDL — Phase 4) |
| `cc_test` | Test native |

- **`vendor: true`** — cài module vào `/vendor`. HAL **bắt buộc** cờ này (ranh giới Treble, xem [glossary.md](../glossary.md)).
- Không viết lệnh compile — chỉ khai srcs + deps; Soong tự dựng lệnh.

`Android.mk` (Make) làm việc tương đương nhưng cú pháp Makefile cũ (`LOCAL_SRC_FILES`, `include $(BUILD_EXECUTABLE)`) — đọc được là đủ, module mới viết `.bp`.

## Envsetup, lunch, m — quy trình build

```bash
source build/envsetup.sh     # 1. nạp hàm build vào shell: lunch, m, mm, mmm, mma...
lunch sdk_car_x86_64-userdebug   # 2. chọn TARGET: thiết bị + build variant
m                            # 3. build toàn bộ image cho target đã chọn
```

**`lunch <target>-<variant>`** chọn hai thứ:

- **Target** = thiết bị/board (`sdk_car_x86_64` = emulator AAOS x86_64).
- **Variant** = `user` (bản ship, khóa) / `userdebug` (như user + root/debug) / `eng` (dev, bật hết). Lệnh như `adb root`, `setenforce 0` **chỉ chạy trên userdebug/eng**.

`lunch` không tham số → menu chọn.

## m / mm / mmm — build cả cây vs một module

| Lệnh | Build gì | Chạy từ đâu |
|------|----------|-------------|
| `m` | **Toàn bộ** image cho target | Bất kỳ đâu (root cây) |
| `mm` | Module **trong thư mục hiện tại** | Trong thư mục module |
| `mmm <path>` | Module ở **path chỉ định** | Bất kỳ đâu |
| `mma` / `mmma` | Module hiện tại/path **+ mọi dependency** | — |
| `m <module>` | Chỉ một module theo tên | Root cây |

```bash
m -j$(nproc)                       # build toàn bộ, dùng hết core
mmm packages/apps/Car/SystemUI     # rebuild chỉ Car SystemUI (nhanh, khi sửa 1 chỗ)
m installclean                     # dọn output cài đặt khi build lỗi lạ
```

⚠️ **Đừng `m` cả cây cho mỗi sửa nhỏ** — build đầu ~1–3 giờ. Sửa một module → `mmm <path>` rồi đẩy artifact lên (`adb sync` / flash partition liên quan). `m` toàn bộ chỉ khi cần image hoàn chỉnh.

## 🚗 Liên hệ Automotive

- **Viết HAL = viết `Android.bp`** (Phase 4): khai `cc_binary` cho service, `aidl_interface` cho interface xe, `vendor: true` để vào `/vendor`, `init_rc` để init khởi động. Note này là ngữ pháp bạn sẽ dùng hằng ngày ở Phase 4.
- **Target xe**: `sdk_car_x86_64` (emulator) để học; dự án thật lunch target board xe của OEM (`lunch <oem_board>-userdebug`). Cùng cơ chế `lunch`, khác tên target.
- **Chu kỳ build dài** (Phase 8): `m` cả cây AAOS + vendor tính bằng giờ trên máy mạnh. Kỹ năng sống còn: biết `mmm` đúng module để vòng lặp sửa–thử tính bằng phút, không phải giờ.
- **Soong vs CMake**: app NDK của bạn (Phase 2) build ngoài cây bằng CMake; HAL/system module build trong cây bằng Soong. Cùng là C/C++, hai hệ build khác nhau — nhớ kẻo lẫn.

## Câu hỏi tự kiểm tra

- [ ] Ba hệ build trong/quanh AOSP là gì, mỗi cái dùng file nào?
- [ ] Vì sao `Android.bp` không có if/loop? So với `build.gradle` thì giống gì?
- [ ] `cc_binary`, `cc_library_shared`, `aidl_interface` ra output gì?
- [ ] `vendor: true` làm gì, vì sao HAL bắt buộc?
- [ ] `lunch sdk_car_x86_64-userdebug` chọn hai thứ gì? Variant nào cho phép `adb root`?
- [ ] `m` vs `mm` vs `mmm` khác nhau chỗ nào? Khi nào dùng cái nào?
- [ ] `source build/envsetup.sh` để làm gì?

## 📚 Tài liệu

- 📖 [Build Android](https://source.android.com/docs/setup/build) — envsetup, lunch, m chính thức
- 📖 [Soong reference](https://source.android.com/docs/setup/build/build-system) — hệ build Soong
- 📖 [Soong Modules Reference](https://ci.android.com/builds/latest/branches/aosp-build-tools/targets/linux/view/soong_build.html) — mọi module type + property
- 📖 [Android.bp file format](https://android.googlesource.com/platform/build/soong/+/master/README.md) — cú pháp Blueprint

## Đọc tiếp

- [build-aaos-image.md](build-aaos-image.md) — dùng lunch + m để ra image AAOS chạy được.
- [repo-tool.md](repo-tool.md) — bước trước: sync source để có cây build.
- [ndk-cmake.md](../2_c_and_jni/ndk-cmake.md) — build native ngoài cây bằng CMake (đối chiếu Soong).
- [partitions-flashing.md](partitions-flashing.md) — `m` ra `.img`, chương này flash chúng.
