---
title: VHAL (Vehicle HAL)
source: https://source.android.com/docs/automotive/vhal?hl=vi
phase: 1 — Android Internals (dùng lại ở Phase 4 & 7)
tags:
  - AAOS
  - VHAL
  - HAL
  - AIDL
  - CarService
---

# VHAL (Vehicle HAL)

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/automotive/vhal](https://source.android.com/docs/automotive/vhal?hl=vi)
> Thuộc chuỗi HAL trong [roadmap](../../android-automotive-developer-roadmap.md) — đọc sau [android-hal.md](android-hal.md). Sẽ code thật ở **Phase 4** và **Phase 7**.

## VHAL là gì?

**VHAL (Vehicle HAL) là HAL đặc biệt dành cho *chức năng của chiếc xe*** — cái cầu duy nhất nối Android với phần cứng xe. Mọi HAL khác trừu tượng hóa một thiết bị (camera, âm thanh...); VHAL trừu tượng hóa *cả chiếc xe*: tốc độ, cửa, ghế, điều hòa, đèn, trạng thái lái... Nó biến mớ tín hiệu hỗn tạp từ **CAN bus** (mạng nội bộ nối các bộ điều khiển trong xe) và **ECU** (Electronic Control Unit — máy tính con điều khiển từng chức năng như động cơ, cửa, điều hòa) thành một danh sách **vehicle property** chuẩn hóa mà Android hiểu được.

Vì sao cần chuẩn hóa như vậy? Vì mỗi hãng xe đi dây, đặt tín hiệu CAN một kiểu khác nhau — nếu framework phải biết chi tiết đó thì không thể chạy chung trên xe của nhiều hãng. VHAL định nghĩa một *interface* (`IVehicle.aidl`): mỗi tín hiệu xe → một **property** có ID, kiểu dữ liệu, vùng áp dụng (area) và cách thay đổi (change mode) thống nhất. OEM/vendor điền phần implement phía dưới (nói chuyện với CAN/ECU thật), còn Android chỉ làm việc với property. Toàn bộ dữ liệu xe — tốc độ, HVAC, ADAS — đều chảy qua đây, nên VHAL là component quan trọng nhất với một platform engineer AAOS.

## Vị trí trong stack

```
App (Car API)
   │
CarPropertyManager          ← API tầng app
   │
CarService (system app)     ← quản lý property, permission
   │  Binder (AIDL)
VHAL service                ← vendor implement (IVehicle.aidl)
   │
CAN bus / ECU / phần cứng xe
```

## 3 thao tác cốt lõi

| Thao tác | Ý nghĩa | Ví dụ |
|----------|---------|-------|
| **Get (read)** | Đọc giá trị property hiện tại | tốc độ xe, nhiệt độ |
| **Set (write)** | Ghi giá trị property | bật điều hòa, chỉnh ghế |
| **Subscribe** | Đăng ký nhận thay đổi | update tốc độ liên tục |

## Vehicle Property

- Mỗi property có **ID** (int) — encode sẵn: property group, data type, area type.
- **Area** — property áp cho vùng nào của xe: `GLOBAL`, `SEAT`, `DOOR`, `WINDOW`, `WHEEL`, `MIRROR`...
- **Change mode:**

| Mode | Ý nghĩa |
|------|---------|
| `STATIC` | Không đổi (vd VIN — số khung xe) |
| `ON_CHANGE` | Báo khi giá trị đổi (vd cửa mở) |
| `CONTINUOUS` | Stream liên tục theo sample rate (vd tốc độ) |

- Nhóm property đặc biệt: **SEAT**, **STEERING_WHEEL**, **ADAS** (hỗ trợ lái: giữ làn, cảnh báo va chạm), HVAC (điều hòa)...

## HIDL vs AIDL VHAL

- **VHAL mới bắt buộc dùng AIDL** (`IVehicle.aidl`) — từ Android 13.
- CarService hỗ trợ cả 2, **ưu tiên AIDL** khi có.
- OEM còn HIDL VHAL (Android ≤ 12) nên migrate sang AIDL.

## Reference implementation & debug

- AOSP có **reference VHAL** (fake/emulator implementation) — dùng để học và test không cần xe thật, chạy trên AAOS emulator.
- Debug: `dumpsys android.hardware.automotive.vehicle.IVehicle/default` hoặc tool `vhal_emulator`, xem property qua **KitchenSink app** trên emulator.

🚗 *Đây là component quan trọng nhất của AAOS platform engineer* — mọi data xe (tốc độ, cửa, HVAC, ADAS) đều chảy qua VHAL. Project Phase 4: thêm 1 custom property vào reference VHAL → expose qua `CarPropertyManager` → hiện lên app.

## Câu hỏi tự kiểm tra

- [ ] VHAL nằm đâu trong luồng App → phần cứng xe?
- [ ] 3 thao tác cốt lõi của VHAL?
- [ ] Property ID encode những thông tin gì? Area type là gì?
- [ ] 3 change mode — property tốc độ xe dùng mode nào?
- [ ] VHAL mới phải viết bằng AIDL hay HIDL? File interface tên gì?

## 📚 Tài liệu

- 📖 [VHAL property configs](https://source.android.com/docs/automotive/vhal/property-configs)
- 📖 [System properties list](https://source.android.com/docs/automotive/vhal/system-properties)
- 📖 [VHAL reference implementation](https://source.android.com/docs/automotive/vhal/reference-implementation)
- 📖 [VHAL debugging](https://source.android.com/docs/automotive/vhal/vhal-debug)

## Đọc tiếp

- [android-hal.md](android-hal.md) — HAL tổng quan
- [aidl-hals.md](aidl-hals.md) — pattern AIDL HAL mà VHAL dùng
- [../carservice-vhal-roles.md](../carservice-vhal-roles.md) — phân vai với CarService
