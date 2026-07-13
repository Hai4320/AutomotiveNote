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

**Binder là cơ chế IPC (Inter-Process Communication — cách hai process nói chuyện với nhau) chính của Android.** Vì lý do bảo mật, mỗi app và mỗi service chạy trong process riêng, không đọc được bộ nhớ của nhau; muốn phối hợp thì phải có một đường truyền qua ranh giới process — đường đó là Binder. Gần như mọi giao tiếp trong máy đều đi qua nó: app ↔ system service, framework ↔ HAL, app ↔ app.

Vì sao không dùng socket/pipe như Linux truyền thống? Vì Android cần thêm ba thứ mà socket không cho sẵn: **gọi hàm ở process khác trông y như gọi hàm local (RPC)** nên lập trình dễ, truyền được **object reference** giữa các process, và tự động kèm **identity người gọi** (UID/PID) để service kiểm tra quyền. Là app dev bạn đã xài Binder mà không biết: mỗi lần gọi `getSystemService()` rồi gọi method trên nó, bên dưới là một Binder call bay sang `system_server`.

## Kiến trúc

```
Process A (client)                Process B (service)
┌─────────────────┐              ┌──────────────────┐
│ Proxy            │              │ Stub              │
│ (AIDL interface) │              │ (implement thật)  │
└───────┬─────────┘              └────────▲─────────┘
        │ transact()                       │ onTransact()
        ▼                                  │
┌──────────────────────────────────────────┴─────────┐
│           /dev/binder (binder driver, kernel)       │
└─────────────────────────────────────────────────────┘
```

- **Binder driver** (`/dev/binder`) — module kernel, trung chuyển mọi transaction, copy data giữa 2 process (chỉ **1 lần copy**: driver ghi thẳng vào vùng nhớ mà process đích đã mmap — map sẵn vào address space — nên không cần copy lần 2).
- **Proxy/Stub** — sinh tự động từ file **AIDL**. **Proxy ở phía client**: client gọi method như hàm local, proxy marshal (serialize) tham số thành **Parcel**, gửi qua driver. **Stub ở phía service**: nhận Parcel, unmarshal, gọi implementation thật.
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
- SELinux cần bộ rule riêng cho vndbinder (`vndbinder_use`, `binder_call` giữa các cặp process, quyền `{add, find}` service). Các khái niệm domain/label/`.te` học ở [selinux.md](selinux.md) (note 20) — đọc xong quay lại đây sẽ hiểu trọn đoạn này.

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
