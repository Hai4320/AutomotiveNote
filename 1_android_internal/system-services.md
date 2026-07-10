---
title: System Services
phase: 1 — Android Internals
tags:
  - AOSP
  - system_server
  - System Services
  - CarService
---

# System Services

> Thuộc **Phase 1 — Android Internals** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc sau [android-architecture.md](android-architecture.md) và [java-native-relationship.md](java-native-relationship.md).

## System service là gì?

- Process/thành phần nền cung cấp **chức năng lõi của OS** cho mọi app: quản lý activity, cửa sổ, package, pin, input...
- App **không gọi thẳng** — đi qua manager class (`Context.getSystemService()`) → Binder → service thật.

```
App
 │ getSystemService(Context.ACTIVITY_SERVICE)
 ▼
ActivityManager (manager class, chạy trong app)
 │ Binder IPC
 ▼
ActivityManagerService (chạy trong system_server)
```

## system_server — nhà của đa số service

- Process Java **lớn nhất hệ thống**, fork từ Zygote lúc boot.
- Chứa hàng chục service chạy chung 1 process (mỗi service là 1 object + thread, không phải process riêng).
- Khởi động trong `SystemServer.java` theo 3 đợt:

| Đợt | Ví dụ |
|-----|-------|
| `startBootstrapServices()` | ActivityManagerService, PowerManagerService, PackageManagerService |
| `startCoreServices()` | BatteryService, UsageStatsService |
| `startOtherServices()` | WindowManagerService, InputManagerService, ConnectivityService... |

## Service Java vs Native

| | Chạy ở đâu | Ví dụ |
|--|-----------|-------|
| Java service | Trong `system_server` | AMS, WMS, PMS |
| Native service | **Process riêng** | `SurfaceFlinger`, `MediaServer`, HAL services |

- Tất cả đăng ký với **servicemanager** (danh bạ Binder) để client tra cứu.

## Các service quan trọng cần biết

| Service | Vai trò |
|---------|---------|
| **ActivityManagerService (AMS)** | Vòng đời activity/process, task stack |
| **WindowManagerService (WMS)** | Quản lý cửa sổ, phối hợp SurfaceFlinger render |
| **PackageManagerService (PMS)** | Cài đặt, query package, permission |
| **PowerManagerService** | Wakelock, suspend, màn hình |
| **InputManagerService** | Sự kiện touch/key từ driver lên app |

## 🚗 CarService — điểm đặc biệt của AAOS

- **KHÔNG nằm trong system_server** — chạy trong **process riêng** của system app `com.android.car` (`CarService`).
- `system_server` chỉ chứa `CarServiceHelperService` nhỏ để start + kết nối CarService.
- Lý do tách: cô lập lỗi (CarService crash không sập cả system_server), vendor extend dễ hơn.
- Bên trong CarService: `CarPropertyService`, `CarPowerManagementService`, `CarAudioService`... — app gọi qua `Car.createCar()` → `CarPropertyManager` v.v.

```
App → CarPropertyManager ─Binder→ CarService (com.android.car) ─Binder→ VHAL (C++)
```

## Lệnh thực hành

```bash
# Liệt kê mọi service đã đăng ký với servicemanager
adb shell service list

# Dump trạng thái 1 service
adb shell dumpsys activity
adb shell dumpsys window
adb shell dumpsys car_service        # AAOS

# Xem system_server chứa gì
adb shell ps -A | grep system_server
```

## Câu hỏi tự kiểm tra

- [ ] `ActivityManager` (trong app) vs `ActivityManagerService` — khác gì, nối nhau qua đâu?
- [ ] system_server fork từ đâu, chứa service dạng process hay object?
- [ ] 3 đợt khởi động service trong `SystemServer.java`?
- [ ] SurfaceFlinger có nằm trong system_server không?
- [ ] CarService chạy ở process nào, vì sao không ở system_server?

## 📚 Tài liệu

- 📖 [Android architecture — Services](https://source.android.com/docs/core/architecture) — docs chính thức, section System services
- 📖 [SystemServer.java (source)](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/java/com/android/server/SystemServer.java) — đọc 3 đợt start service thật
- 📖 [dumpsys docs](https://developer.android.com/tools/dumpsys)
- 📖 [Car docs — What is Automotive](https://source.android.com/docs/automotive/start/what_automotive) — vị trí CarService

## Đọc tiếp

- [android-architecture.md](android-architecture.md) — note trong repo
- [java-native-relationship.md](java-native-relationship.md) — note trong repo, 2 cầu JNI/Binder
- [hal/android-vhal.md](hal/android-vhal.md) — note trong repo, phía dưới CarService
