---
title: AAOS Structure & Build System
source: https://source.android.com/docs/automotive/start/what_automotive
phase: 1 — Android Internals (đào sâu ở Phase 3 & 7)
tags:
  - AAOS
  - AOSP
  - CarService
  - Build
---

# AAOS Structure & Build System — AOSP + car-specific layers

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/automotive/start/what_automotive](https://source.android.com/docs/automotive/start/what_automotive)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../android-automotive-developer-roadmap.md). Build thật ở **Phase 3**, chuyên sâu ở **Phase 7**.

## AAOS là gì?

**AAOS (Android Automotive OS) là chính Android bạn đã biết, nhưng chạy trực tiếp trên phần cứng xe làm hệ điều hành của xe.** Điểm mấu chốt cho người mới: nó **không phải một fork riêng biệt**, mà là **cùng một codebase Android** trên phone/tablet được mở rộng thêm các layer chuyên cho xe. Kỹ năng Android bạn có vẫn dùng được — chỉ học thêm phần "xe".

Vì sao Google làm nó? Vì màn hình trung tâm trên xe ngày càng cần một OS đầy đủ thay vì phần mềm nhúng rời rạc của từng hãng. AAOS gánh 2 nhiệm vụ: lo **IVI** (In-Vehicle Infotainment — hệ thống thông tin giải trí trung tâm: nhạc, bản đồ, điều hòa) ngay bây giờ, và làm nền cho **SDV** (Software-Defined Vehicle — xe mà tính năng do phần mềm quyết, cập nhật qua OTA) trong tương lai.

Để tránh nhầm ngay từ đầu: AAOS khác hẳn Android Auto — một cái *là* OS của xe, cái kia chỉ chiếu màn hình điện thoại lên xe (xem bảng ngay dưới).

### AAOS vs Android Auto — hay nhầm

| | Android Auto | Android Automotive (AAOS) |
|--|--------------|---------------------------|
| Chạy ở đâu | **Trên điện thoại**, cast UI qua USB/wireless lên màn xe | **Trên phần cứng xe**, là OS của xe |
| Xe không có phone | Không dùng được | Chạy độc lập |
| Ví dụ | Đa số xe đời cũ | Polestar, Volvo, GM, Renault... |

### GAS (Google Automotive Services)
- Gói app/service Google (Maps, Assistant, Play Store) OEM **license riêng** để nhúng vào IVI — như GMS (Google Mobile Services) bên phone.
- AAOS thuần AOSP không có GAS → OEM tự làm store/map hoặc license.

## AAOS = AOSP + các layer car-specific

```
┌──────────────────────────────────────────────┐
│ Car Apps (Launcher, HVAC, Media, Cluster...)  │  packages/apps/Car/*
├──────────────────────────────────────────────┤
│ Car API (android.car.*)                       │  packages/services/Car/car-lib
├──────────────────────────────────────────────┤
│ CarService (com.android.car)                  │  packages/services/Car
├──────────────────────────────────────────────┤
│ Car HALs: VHAL, Audio Control, EVS...         │  hardware/interfaces/automotive/*
├──────────────────────────────────────────────┤
│         AOSP chuẩn (framework, ART,           │
│         system_server, kernel...)             │
└──────────────────────────────────────────────┘
```

Vị trí trong source tree:

| Layer | Path trong AOSP |
|-------|-----------------|
| Car API (client lib) | `packages/services/Car/car-lib/` |
| CarService + các service con | `packages/services/Car/service/` |
| Car apps mẫu (Launcher, Settings, HVAC, KitchenSink) | `packages/apps/Car/` |
| CarSystemUI | `frameworks/base/packages/CarSystemUI` |
| Car HAL interfaces (VHAL, EVS, audio control) | `hardware/interfaces/automotive/` |
| Device config emulator | `device/generic/car/` |

- Các mảnh đã học: [system-services.md](system-services.md) (CarService process riêng), [hal/android-vhal.md](hal/android-vhal.md), [hal/android-hal.md](hal/android-hal.md).

## Build system cho AAOS

Cùng hệ build AOSP (`repo` + Soong/`Android.bp`), khác **lunch target**:

```bash
source build/envsetup.sh
lunch sdk_car_x86_64-trunk_staging-userdebug   # AAOS emulator x86_64
m -j$(nproc)                                    # build
emulator                                        # chạy AAOS emulator
```

| Target | Dùng cho |
|--------|----------|
| `sdk_car_x86_64` | AAOS emulator trên máy dev (bài tập Phase 3) |
| `sdk_car_arm64` | Emulator ARM (máy Apple Silicon) |
| `aosp_cf_x86_64_auto` | Cuttlefish auto (Android ảo chạy trên cloud/CI) |
| Target OEM (vd `sa8155_car`...) | Board thật của vendor |

- Feature flag phân biệt car build: `PRODUCT_CHARACTERISTICS := automotive`, package `android.hardware.type.automotive`.
- App check runtime: `PackageManager.FEATURE_AUTOMOTIVE`.

## 🚗 Điều platform engineer cần nắm

- Sửa behavior xe = thường đụng **3 tầng cùng lúc**: app (`packages/apps/Car`) → CarService (`packages/services/Car`) → HAL (`hardware/interfaces/automotive`) — dự án thật chia team theo đúng 3 tầng này.
- **KitchenSink** (`packages/apps/Car/Kitchensink`) — app test mọi Car API, công cụ khám phá nhanh nhất trên emulator.
- OEM đổi branding/UI qua **RRO (Runtime Resource Overlay — đè resource như theme/layout mà không sửa APK gốc) + CarSystemUI**, không sửa framework gốc (giữ khả năng merge AOSP mới).

## Câu hỏi tự kiểm tra

- [ ] AAOS vs Android Auto — khác nhau căn bản gì?
- [ ] GAS là gì, AOSP thuần có không?
- [ ] Car API, CarService, Car HAL nằm đâu trong source tree?
- [ ] Lunch target nào build AAOS emulator? Lệnh đầy đủ?
- [ ] Build phân biệt "đây là xe" qua flag nào?
- [ ] KitchenSink dùng làm gì?

## 📚 Tài liệu

- 📖 [What is Android Automotive](https://source.android.com/docs/automotive/start/what_automotive) — docs chính thức
- 📖 [AAOS developer guide](https://source.android.com/docs/automotive/start/avd/android_virtual_device) — build & chạy AAOS emulator
- 📖 [packages/services/Car (source)](https://cs.android.com/android/platform/superproject/main/+/main:packages/services/Car/) — đọc CarService thật
- 📖 [device/generic/car (source)](https://cs.android.com/android/platform/superproject/main/+/main:device/generic/car/) — device config emulator

## Đọc tiếp

- [system-services.md](system-services.md) — CarService process riêng
- [hal/android-vhal.md](hal/android-vhal.md) — Vehicle HAL
- [android-architecture.md](android-architecture.md) — AOSP layers gốc
