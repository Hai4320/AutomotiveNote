---
title: Java Binder & AIDL — IPC tầng framework
source: https://developer.android.com/develop/background-work/services/aidl
phase: 2 — C/C++ & JNI
tags:
  - Binder
  - AIDL
  - IPC
  - Framework
  - Kotlin
---

# Java Binder & AIDL — IPC tầng framework

> Ghi chú từ tài liệu chính thức: [Android Interface Definition Language (AIDL)](https://developer.android.com/develop/background-work/services/aidl)
> Thuộc **Phase 2 — C/C++ & JNI** trong [roadmap](../android-automotive-developer-roadmap.md). Đây là phần **ôn framework** — đọc **sau** khi đã nắm cơ chế Binder ở tầng thấp trong [binder-ipc.md](../1_android_internal/binder-ipc.md). Phase 1 giải thích Binder *hoạt động thế nào*; note này là *cách bạn dùng nó* trong Kotlin/Java.
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## AIDL là gì và vì sao cần?

**AIDL (Android Interface Definition Language)** là ngôn ngữ khai báo interface để 2 **process** khác nhau gọi hàm của nhau qua Binder. Bạn viết 1 file `.aidl` mô tả các method + kiểu dữ liệu; trình biên dịch AIDL tự sinh ra 2 lớp Java: **Stub** (bên service cài đặt thật) và **Proxy** (bên client gọi, trông như gọi hàm local).

Vì sao cần AIDL thay vì tự viết Binder tay? Vì tự đóng gói tham số vào `Parcel`, gọi `transact()`, giải nén ở `onTransact()` là việc lặp đi lặp lại và dễ sai. AIDL sinh toàn bộ boilerplate đó — bạn chỉ khai báo interface, phần **marshalling** (đóng gói dữ liệu qua ranh giới process) máy lo.

Khi nào **không** cần AIDL:
- Chỉ giao tiếp trong **cùng process** → dùng interface Kotlin bình thường, hoặc `Binder` local (extend `Binder`, trả về qua `onBind`). Không tốn chi phí marshalling.
- Cần gọi service từ process khác nhưng **không cần đa luồng** → `Messenger` đủ (nó xếp hàng request về 1 thread, đơn giản hơn).
- **Cần AIDL khi**: nhiều app/process gọi đồng thời VÀ service phải xử lý song song đa luồng.

## Luồng tối thiểu: 1 AIDL interface giữa 2 app

**1. Khai báo file `.aidl`** (đặt trong `src/main/aidl/<package>/`, cùng package 2 app phải khớp):

```java
// IRemoteAdder.aidl
package com.example.shared;

interface IRemoteAdder {
    int add(int a, int b);
}
```

Build 1 lần → Gradle sinh `IRemoteAdder.java` chứa `IRemoteAdder.Stub` (abstract) và Proxy ẩn bên trong.

**2. Bên SERVICE — cài đặt Stub, trả về qua `onBind`:**

```kotlin
class AdderService : Service() {
    private val binder = object : IRemoteAdder.Stub() {   // Stub = Binder + interface
        override fun add(a: Int, b: Int): Int = a + b     // chạy trên thread của Binder pool
    }
    override fun onBind(intent: Intent): IBinder = binder
}
```

Khai báo service `exported` trong manifest để app khác bind được:

```xml
<service android:name=".AdderService" android:exported="true">
    <intent-filter><action android:name="com.example.ADD"/></intent-filter>
</service>
```

**3. Bên CLIENT — bind rồi ép kiểu bằng `asInterface`:**

```kotlin
private var adder: IRemoteAdder? = null

private val conn = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName, service: IBinder) {
        adder = IRemoteAdder.Stub.asInterface(service)   // KHÔNG cast trực tiếp — asInterface xử lý cả cross-process lẫn local
    }
    override fun onServiceDisconnected(name: ComponentName) { adder = null }
}

// bind:
val intent = Intent("com.example.ADD").setPackage("com.example.serviceapp")
bindService(intent, conn, Context.BIND_AUTO_CREATE)

// gọi (trông như local, thực ra là Binder transaction sang process kia):
val sum = adder?.add(2, 3)   // = 5
```

## Kiểu dữ liệu AIDL hỗ trợ

| Nhóm | Kiểu | Ghi chú |
|------|------|---------|
| Nguyên thủy | `int`, `long`, `boolean`, `float`, `double`, `char`, `byte` | map thẳng |
| Chuỗi/mảng | `String`, `CharSequence`, `List`, `Map`, mảng `T[]` | `List`/`Map` phần tử phải là kiểu AIDL hỗ trợ |
| Object tùy biến | class implement `Parcelable` | phải khai báo `parcelable Foo;` trong 1 file `.aidl` riêng |
| Interface khác | `IBinder`, interface AIDL khác | truyền được callback qua đây |

**Hướng tham số** — với kiểu không nguyên thủy phải ghi rõ `in` / `out` / `inout`:

```java
void update(in Config cfg, out Result res);
```

- `in` — chỉ gửi đi (mặc định, rẻ nhất).
- `out` — chỉ nhận về (client cấp object rỗng, service điền vào).
- `inout` — cả hai chiều (đắt: marshal 2 lần). Chỉ dùng khi thật cần.

## `oneway` — gọi không chờ

Mặc định Binder call **đồng bộ**: client block cho tới khi service trả về. Đánh dấu `oneway` để gọi bất đồng bộ (fire-and-forget), client trả về ngay:

```java
oneway interface INotifier {
    void onEvent(int code);   // không trả giá trị được (void), không throw về client
}
```

Dùng cho **callback** service → client (đăng ký 1 `oneway` interface từ client, service gọi ngược lại) và cho lệnh không cần kết quả. Lưu ý: `oneway` từ *cùng process* vẫn chạy đồng bộ.

## Threading — bẫy hay quên

- Method Stub chạy trên **thread của Binder thread pool** (mặc định tối đa 16 thread/process), **KHÔNG** phải main thread. → Đụng UI phải `post` về main thread; dữ liệu chia sẻ phải tự đồng bộ (`synchronized`/lock).
- Nhiều client gọi cùng lúc = nhiều thread vào Stub song song. Code trong Stub phải **thread-safe**.
- Binder call **đồng bộ block thread gọi**. Gọi từ main thread của client mà service xử lý lâu = ANR. Gọi trong background thread hoặc dùng `oneway`.

## 🚗 Liên hệ Automotive

- **`CarService` và mọi Car API dùng AIDL.** Khi app gọi `CarPropertyManager.getProperty(...)` để đọc tốc độ/nhiệt độ, bên dưới là 1 AIDL Binder call bay sang process `com.android.car` (CarService). Roadmap Phase 3–4 sẽ tự viết property mới xuyên qua chuỗi này.
- **AIDL for HALs** — từ Android 11, HAL đời mới (VHAL v2+) khai báo bằng **stable AIDL** thay cho HIDL. Cùng ngôn ngữ `.aidl` nhưng versioned và chạy được qua ranh giới vendor/system. Chi tiết ở Phase 4 → [AIDL for HALs](https://source.android.com/docs/core/architecture/aidl/aidl-hals).
- **Bẫy version**: xe cập nhật OTA từng phần — app (system partition) và HAL (vendor partition) có thể lệch version. AIDL thường không đổi thì hỏng ngầm; **stable AIDL** bắt buộc versioning để tránh. Đây là lý do Automotive dùng stable AIDL chứ không AIDL "app thường".
- Debug ai đang giữ Binder / service nào đang chạy:

```bash
adb shell dumpsys activity services   # list service đang bind + client
adb shell service list                # mọi Binder service đã đăng ký với servicemanager
adb shell dumpsys car_service         # trạng thái CarService + property
```

## Câu hỏi tự kiểm tra

- [ ] AIDL sinh ra 2 lớp nào, lớp nào bên client lớp nào bên service?
- [ ] Vì sao dùng `IRemoteAdder.Stub.asInterface(service)` thay vì cast thẳng `service as IRemoteAdder`?
- [ ] Method trong Stub chạy trên thread nào? Hệ quả gì khi đụng UI hoặc dữ liệu chia sẻ?
- [ ] `in` / `out` / `inout` khác nhau ra sao, cái nào đắt nhất và vì sao?
- [ ] `oneway` thay đổi hành vi gọi thế nào, dùng cho tình huống nào?
- [ ] Khi nào KHÔNG cần AIDL mà dùng local `Binder` hoặc `Messenger`?
- [ ] Vì sao Automotive dùng *stable* AIDL cho HAL thay vì AIDL app thường?

## 📚 Tài liệu

- [Android Interface Definition Language (AIDL)](https://developer.android.com/develop/background-work/services/aidl) — docs chính thức, có ví dụ đầy đủ
- [Bound services](https://developer.android.com/develop/background-work/services/bound-services) — vòng đời bind/unbind, `ServiceConnection`
- [AIDL for HALs](https://source.android.com/docs/core/architecture/aidl/aidl-hals) — stable AIDL cho HAL (dùng ở Phase 4)
- [Parcelable](https://developer.android.com/reference/android/os/Parcelable) — truyền object tùy biến qua Binder

## Đọc tiếp

- [binder-ipc.md](../1_android_internal/binder-ipc.md) — cơ chế Binder tầng thấp (Parcel, transact/onTransact, servicemanager) mà AIDL đóng gói giùm.
- [jni.md](jni.md) — cầu Java↔native; AIDL là cầu process↔process. Hai loại "biên giới" khác nhau trong Android.
- [carservice-vhal-roles.md](../1_android_internal/carservice-vhal-roles.md) — CarService dùng AIDL để expose Car API; nền cho thực hành Phase 4.
