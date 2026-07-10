---
title: Binder IPC
source: https://source.android.com/docs/core/architecture/hidl/binder-ipc?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - Binder
  - IPC
  - AIDL
---

# Binder IPC

> Thuộc **Phase 1 — Android Internals** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc sau [system-services.md](system-services.md) và [java-native-relationship.md](java-native-relationship.md).

## Binder là gì?

- Cơ chế **IPC (Inter-Process Communication) chính của Android** — gần như mọi giao tiếp giữa process đi qua nó: app ↔ system service, framework ↔ HAL, app ↔ app.
- Không dùng socket/pipe truyền thống làm chuẩn chung vì Binder cho: **gọi hàm như local (RPC)**, truyền object reference, kèm **identity người gọi** (UID/PID) để check permission.

## Kiến trúc

```
Process A (client)                Process B (service)
┌─────────────────┐              ┌──────────────────┐
│ Proxy (Stub)     │              │ Binder object     │
│ interface AIDL   │              │ (implement thật)  │
└───────┬─────────┘              └────────▲─────────┘
        │ transact()                       │ onTransact()
        ▼                                  │
┌──────────────────────────────────────────┴─────────┐
│           /dev/binder (binder driver, kernel)       │
└─────────────────────────────────────────────────────┘
```

- **Binder driver** (`/dev/binder`) — module kernel, trung chuyển mọi transaction, copy data giữa 2 process (1 lần copy, qua mmap).
- **Proxy/Stub** — sinh tự động từ file **AIDL**; client gọi method như hàm local, proxy marshal tham số thành **Parcel**, gửi qua driver.
- **servicemanager** — "danh bạ": service đăng ký tên, client tra tên lấy handle. Bản thân servicemanager cũng nói chuyện qua Binder (handle 0 đặc biệt).

## Luồng 1 lần gọi

1. Client gọi `proxy.getSpeed()`
2. Proxy marshal args → **Parcel** → `transact()` xuống `/dev/binder`
3. Driver tìm process đích, copy data, đánh thức **binder thread** bên service
4. Stub `onTransact()` unmarshal → gọi implementation thật
5. Kết quả đi ngược lại; client **block chờ** (synchronous mặc định; `oneway` = fire-and-forget)

## Điểm cần nhớ

| Khái niệm | Ý nghĩa |
|-----------|---------|
| **Parcel** | Container serialize data qua Binder; object phải `Parcelable` |
| **Binder thread pool** | Mỗi process có pool (mặc định max ~15-16 thread) nhận call đến — service chậm thì pool cạn, caller treo |
| **oneway** | Call async, không chờ kết quả — dùng cho callback/notify |
| **Death recipient** | `linkToDeath()` — client được báo khi process service chết |
| **Calling identity** | `Binder.getCallingUid()/Pid()` — service check quyền người gọi; `clearCallingIdentity()` khi cần gọi tiếp bằng danh nghĩa mình |
| **Transaction buffer 1MB** | Mỗi process ~1MB cho mọi transaction đang bay — gửi data to = `TransactionTooLargeException` |

## 3 biến thể domain

| Domain | Node | Dùng cho |
|--------|------|----------|
| Binder | `/dev/binder` | Framework ↔ app, dùng AIDL |
| **hwbinder** | `/dev/hwbinder` | Framework ↔ vendor dùng HIDL (legacy) |
| **vndbinder** | `/dev/vndbinder` | Vendor ↔ vendor dùng AIDL |

- Lý do tách (Android 8, "binder contexts"): **cô lập traffic framework (device-independent) khỏi traffic vendor (device-specific)** — framework không bị vendor spam chung 1 driver context.
- AIDL HAL hiện đại (stable AIDL) chạy trên `/dev/binder` thường — xem [hal/aidl-hals.md](hal/aidl-hals.md).

### Vendor dùng vndbinder

```cpp
// PHẢI gọi trước mọi giao tiếp Binder trong process
ProcessState::initWithDriver("/dev/vndbinder");
```

- Danh bạ riêng: **vndservicemanager** (thay servicemanager), context khai trong `vndservice_contexts`.
- SELinux cần đủ 4 thứ: `vndbinder_use(domain)` (mở `/dev/vndbinder`), quyền transfer/call với vndservicemanager, `binder_call(A, B)` cho từng cặp domain, quyền `{add, find}` service — label dùng `vndservice_manager_type` trong `vndservice.te`.

## Tối ưu hiệu năng (Android 8+)

| Cải tiến | Ý nghĩa |
|----------|---------|
| **Scatter-gather** | Giảm copy data 3 lần → **1 lần** — truyền thẳng, bỏ serialize trung gian |
| **Fine-grained locking** | Bỏ global lock trong driver → giảm latency, hết giật khi nhiều process gọi đồng thời |
| **Binder contexts** | Tách domain framework/vendor như trên |

Bối cảnh: từ Android 8 (Treble), framework ↔ HAL chuyển hết sang Binder → traffic tăng vọt, phải tối ưu driver.

## 🚗 Liên hệ Automotive

Luồng `App → CarPropertyManager → CarService → VHAL` có **2 hop Binder**:
- App ↔ CarService: Binder thường + permission check (`Car.PERMISSION_SPEED`...) bằng calling identity
- CarService ↔ VHAL: AIDL HAL binder; VHAL subscribe property `CONTINUOUS` đẩy event ngược bằng callback `oneway` — tốc độ xe update liên tục mà không block VHAL thread

Debug: `dumpsys binder_calls_stats`, xem transaction chậm; treo UI hay do binder call synchronous vào service đang bận.

## Câu hỏi tự kiểm tra

- [ ] Vì sao Android chọn Binder thay socket/pipe cho IPC chuẩn?
- [ ] Proxy/Stub sinh từ đâu? Parcel là gì?
- [ ] Binder call mặc định sync hay async? `oneway` khác gì?
- [ ] Binder thread pool cạn thì hiện tượng gì?
- [ ] `TransactionTooLargeException` do đâu, giới hạn bao nhiêu?
- [ ] hwbinder vs vndbinder dùng cho ai? Vì sao Android 8 tách 3 context?
- [ ] Vendor process muốn dùng vndbinder phải gọi gì trước tiên? Danh bạ nào?
- [ ] Scatter-gather giảm copy từ mấy xuống mấy?

## 📚 Tài liệu

- 📖 [Binder IPC — AOSP docs](https://source.android.com/docs/core/architecture/hidl/binder-ipc) — docs chính thức
- 📖 [AIDL docs](https://developer.android.com/develop/background-work/services/aidl)
- 📖 [Binder API reference](https://developer.android.com/reference/android/os/Binder) — calling identity, linkToDeath
- 📖 [Parcel API reference](https://developer.android.com/reference/android/os/Parcel)

## Đọc tiếp

- [system-services.md](system-services.md) — note trong repo
- [java-native-relationship.md](java-native-relationship.md) — note trong repo
- [hal/aidl-hals.md](hal/aidl-hals.md) — note trong repo
