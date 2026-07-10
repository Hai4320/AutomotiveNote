---
title: Vai trò của Car Services và Vehicle HAL
phase: 1 — Android Internals
tags:
  - AAOS
  - CarService
  - VHAL
  - Car API
---

# Vai trò của Car Services và Vehicle HAL

> Thuộc **Phase 1 — Android Internals** trong [roadmap](../android-automotive-developer-roadmap.md) — mục cuối Phase 1, tổng hợp từ [system-services.md](system-services.md), [hal/android-vhal.md](hal/android-vhal.md), [aaos-structure.md](aaos-structure.md).

## Phân vai trong 1 câu

- **VHAL** = *phiên dịch viên phần cứng*: biến tín hiệu xe (CAN/ECU) thành **vehicle properties** chuẩn hóa.
- **CarService** = *người gác cổng + điều phối*: quản lý property, **check permission**, cấp API an toàn cho app.

App không bao giờ nói chuyện thẳng với VHAL — luôn qua CarService.

```
┌─────────────┐   Car API      ┌──────────────────┐   AIDL Binder   ┌────────────┐   CAN/ECU
│  App        │ ─────────────▶ │  CarService       │ ──────────────▶ │  VHAL      │ ─────────▶ 🚗
│ (Kotlin)    │  permission ✋  │ (com.android.car) │  property ↕     │ (C++ vendor)│
└─────────────┘                └──────────────────┘                 └────────────┘
     Google/OEM quản              Google + OEM extend                  Vendor implement
```

## Vai trò của VHAL

| Vai trò | Chi tiết |
|---------|----------|
| **Chuẩn hóa** | Mỗi tín hiệu xe → 1 property có ID, type, area, change mode ([android-vhal.md](hal/android-vhal.md)) — mọi OEM cùng ngôn ngữ |
| **Trừu tượng hóa vendor** | Framework không biết xe dùng CAN, LIN hay Ethernet — vendor giấu hết sau `IVehicle.aidl` |
| **Get / Set / Subscribe** | 3 thao tác duy nhất — mọi tương tác xe quy về đây |
| **Ranh giới Treble** | VHAL ở vendor partition — update Android không đụng, update vendor không sợ vỡ framework ([hal/vintf-objects.md](hal/vintf-objects.md)) |

## Vai trò của CarService

Chạy process riêng `com.android.car` (không trong system_server — [system-services.md](system-services.md)), bên trong là **bó service con**:

| Service con | Quản gì |
|-------------|---------|
| `CarPropertyService` | Get/set/subscribe property — mặt tiền của VHAL |
| `CarPowerManagementService` | Power state xe (STR — suspend to RAM, garage mode) |
| `CarAudioService` | Audio zone (driver/passenger riêng), ducking, volume theo zone |
| `CarInputService` | Nút bấm vô lăng, rotary controller |
| `CarUxRestrictionsManagerService` | **Driver distraction** — khóa UI khi xe chạy |
| `CarDrivingStateService` | Trạng thái lái (parked / idling / moving) |

Vai trò chung:

1. **Permission gatekeeper** — check `Car.PERMISSION_*` bằng Binder calling identity ([binder-ipc.md](binder-ipc.md)) trước khi cho đụng property.
2. **Policy layer** — VHAL báo "xe đang chạy" (dữ kiện), CarService quyết "khóa keyboard, giới hạn list 6 item" (chính sách).
3. **API ổn định cho app** — app dùng `Car.createCar()` → `CarPropertyManager`, `CarAudioManager`... không cần biết HAL.
4. **Điểm OEM extend** — thêm service con / property riêng mà không vỡ Car API chuẩn.

## Ví dụ xuyên suốt: hiện tốc độ + khóa UI

```
ECU tốc độ ──CAN──▶ VHAL: property PERF_VEHICLE_SPEED (CONTINUOUS, 10Hz)
                      │ subscribe callback
                      ▼
        CarPropertyService ──▶ app cluster hiện 87 km/h
                      │
        CarDrivingStateService: state = MOVING
                      ▼
        CarUxRestrictionsManagerService ──▶ app media tự ẩn keyboard search
```

VHAL chỉ đưa số 87. Mọi quyết định "87 nghĩa là gì với UX" nằm ở CarService — **đó là ranh giới vai trò**.

## Câu hỏi tự kiểm tra

- [ ] Vì sao app không được nói chuyện thẳng với VHAL?
- [ ] VHAL chuẩn hóa tín hiệu xe thành gì? 3 thao tác?
- [ ] CarService check permission bằng cơ chế Binder nào?
- [ ] Kể 4 service con trong CarService và vai trò?
- [ ] "Dữ kiện vs chính sách" — bên nào giữ bên nào? Ví dụ?
- [ ] OEM muốn thêm tính năng riêng thì extend ở tầng nào?

## 📚 Tài liệu

- 📖 [Car Service overview](https://source.android.com/docs/automotive/start/what_automotive) — docs chính thức
- 📖 [VHAL docs](https://source.android.com/docs/automotive/vhal)
- 📖 [UX Restrictions](https://source.android.com/docs/automotive/hmi/driver_distraction/driver_ux) — driver distraction chính thức
- 📖 [CarPropertyService (source)](https://cs.android.com/android/platform/superproject/main/+/main:packages/services/Car/service/src/com/android/car/CarPropertyService.java)

## Đọc tiếp

- [hal/android-vhal.md](hal/android-vhal.md) — property/area/change mode chi tiết
- [system-services.md](system-services.md) — vì sao CarService process riêng
- [aaos-structure.md](aaos-structure.md) — vị trí trong source tree
