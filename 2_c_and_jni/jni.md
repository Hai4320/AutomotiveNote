---
title: JNI — cầu nối Kotlin/Java ↔ C/C++
phase: 2 — C/C++ & JNI
tags:
  - JNI
  - NDK
  - C
  - C++
  - Native
---

# JNI — cầu nối Kotlin/Java ↔ C/C++

> Thuộc **Phase 2 — C/C++ & JNI** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc **sau** [memory-management.md](memory-management.md) — JNI đứng giữa 2 thế giới bộ nhớ: bên có GC (Java) và bên tự quản (native), hiểu quản lý bộ nhớ trước thì reference của JNI mới thông.
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## JNI là gì và vì sao cần?

**JNI (Java Native Interface)** là chuẩn cho phép code chạy trong máy ảo (ART/JVM) **gọi** hàm C/C++ và ngược lại. Nó là **cửa khẩu** giữa 2 quốc gia dùng 2 loại "tiền" khác nhau: bên Java quản object bằng GC, bên native quản con trỏ bằng tay. JNI là phòng đổi tiền + hải quan ở biên giới.

Vì sao app Android cần bước xuống native?
- **Tái dùng thư viện C/C++ sẵn có** — codec, mã hóa, engine game, thư viện xử lý ảnh/âm thanh.
- **Hiệu năng** — vòng lặp tính toán nặng, xử lý tín hiệu.
- **Chạm phần cứng / tầng dưới** — đây là lý do chính với **Automotive**: HAL, driver, giao thức xe (CAN, VHAL) đều là C/C++. App muốn nói chuyện với chúng phải qua JNI.

```
┌─────────────┐   JNI    ┌──────────────┐
│ Kotlin/Java │ ───────► │  C/C++ (.so) │
│  (ART, GC)  │ ◄─────── │ tự quản memory│
└─────────────┘          └──────────────┘
     hộ chiếu = JNIEnv*
```

## Luồng tối thiểu: gọi C++ từ Kotlin

**1. Khai báo `external` bên Kotlin + load thư viện:**

```kotlin
class NativeBridge {
    external fun add(a: Int, b: Int): Int   // thân hàm nằm ở C++

    companion object {
        init { System.loadLibrary("native-bridge") }  // load libnative-bridge.so
    }
}
```

**2. Cài đặt bên C++ — tên hàm theo quy ước cố định:**

```cpp
#include <jni.h>

extern "C"                        // TẮT name mangling của C++, để JNI tìm đúng tên
JNIEXPORT jint JNICALL
Java_com_example_NativeBridge_add(JNIEnv* env, jobject thiz, jint a, jint b) {
    return a + b;                 // jint ↔ Int, tự map
}
```

Quy ước tên: `Java_` + package (`.`→`_`) + `_` + class + `_` + method. Sai một ký tự = `UnsatisfiedLinkError` lúc runtime (không lúc compile).

**3. Build bằng NDK/CMake** → sinh ra `libnative-bridge.so` nhét vào APK (chi tiết ở note NDK & CMake).

## `JNIEnv*` và `jobject thiz` — 2 tham số luôn có

Mọi hàm native đều nhận **thêm 2 tham số ẩn** trước các tham số thật:

| Tham số | Là gì |
|---------|-------|
| `JNIEnv* env` | **Hộ chiếu** — con trỏ tới bảng ~200 hàm để thao tác với thế giới Java (tạo string, gọi method, đọc field...). **Chỉ dùng trong đúng thread cấp nó**, không cache sang thread khác. |
| `jobject thiz` | `this` — object Java đã gọi hàm (hoặc `jclass` nếu method là `static`) |

Muốn làm gì với phía Java đều phải qua `env`:

```cpp
// tạo Java String từ C string
jstring js = env->NewStringUTF("hello");

// gọi method Java từ native (callback ngược)
jclass cls   = env->GetObjectClass(thiz);
jmethodID m  = env->GetMethodID(cls, "onDone", "(I)V");  // "(I)V" = nhận int, trả void
env->CallVoidMethod(thiz, m, 42);
```

## Type signature — thứ hay sai nhất

JNI mô tả kiểu bằng **chuỗi mã hóa**, không phải tên kiểu. `GetMethodID` sai signature = `NoSuchMethodError` runtime.

| Kiểu Java | Mã | | Kiểu Java | Mã |
|-----------|----|--|-----------|----|
| `boolean` | `Z` | | `long` | `J` |
| `byte` | `B` | | `float` | `F` |
| `char` | `C` | | `double` | `D` |
| `short` | `S` | | `void` | `V` |
| `int` | `I` | | `Object` | `Ljava/lang/Object;` |

- Mảng: thêm `[` phía trước — `int[]` = `[I`, `String[]` = `[Ljava/lang/String;`
- Method: `(tham_số)trả_về` — ví dụ `String f(int, long)` → `(IJ)Ljava/lang/String;`

Mẹo: chạy `javap -s YourClass.class` để in signature chuẩn, khỏi đoán tay.

## Local vs Global reference — cái bẫy rò rỉ của JNI

Mọi `jobject` mà `env` trả về là một **reference** tới object Java (không phải con trỏ trần). Có 3 loại:

| Loại | Sống bao lâu | Cách tạo |
|------|--------------|----------|
| **Local** (mặc định) | Hết hàm native là **tự thu hồi** | Mọi hàm `env->New*` / `Get*` trả về |
| **Global** | Tới khi bạn `DeleteGlobalRef` | `env->NewGlobalRef(x)` |
| **Weak global** | Như global nhưng không chặn GC | `env->NewWeakGlobalRef(x)` |

Hai lỗi kinh điển:
1. **Cache local ref sang lần gọi sau / thread khác** → nó đã bị thu hồi → **crash**. Muốn giữ lâu phải `NewGlobalRef`.
2. **Tạo nhiều local ref trong vòng lặp mà không xóa** → bảng local ref (mặc định chỉ ~512 slot) **tràn** → crash. Trong loop dài phải `env->DeleteLocalRef(x)` sau mỗi vòng.

```cpp
// SAI: giữ local ref để dùng ở callback sau này
myField = env->NewStringUTF("x");        // ❌ local — chết sau khi hàm return

// ĐÚNG: nâng lên global
myField = env->NewGlobalRef(env->NewStringUTF("x"));   // ✅ nhớ DeleteGlobalRef khi xong
```

Đây chính là lý do note này xếp **sau** [memory-management.md](memory-management.md): bên Java GC lo, bên native bạn lo, và **reference của JNI là biên giới bạn phải tự quản** — quên là leak hoặc crash, y hệt `malloc`/`free`.

## `RegisterNatives` — cách nối tên động (AOSP hay dùng)

Ngoài quy ước đặt tên `Java_...`, có cách 2: đăng ký tay bằng bảng ánh xạ trong `JNI_OnLoad` (gọi tự động khi `.so` được load). Ưu điểm: tên hàm C++ đặt tùy ý, nhanh hơn lúc link, và **giấu tên** (khó reverse hơn). Code framework Android phần lớn dùng cách này.

```cpp
static jint add(JNIEnv*, jclass, jint a, jint b) { return a + b; }

static const JNINativeMethod methods[] = {
    {"add", "(II)I", (void*)add},        // {tên Java, signature, con trỏ hàm C}
};

JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void*) {
    JNIEnv* env;
    vm->GetEnv((void**)&env, JNI_VERSION_1_6);
    jclass c = env->FindClass("com/example/NativeBridge");
    env->RegisterNatives(c, methods, 1);
    return JNI_VERSION_1_6;
}
```

## 🚗 Liên hệ Automotive

- **Đường xuống phần cứng đi qua JNI**: app Kotlin muốn đọc tốc độ, mức pin, trạng thái cửa → gọi **Car API** (Java) → xuống **VHAL** (C++) → driver. Chỗ nối Java↔C++ đó chính là JNI. Hiểu JNI mới debug được khi dữ liệu xe "không lên tới app".
- **`JavaVM` cache để callback từ thread native**: cảm biến/CAN bus đẩy dữ liệu từ **thread C++ riêng** (không phải thread Java gọi xuống). Muốn bắn event ngược lên app phải cache `JavaVM*` (an toàn xuyên thread, khác `JNIEnv*`) rồi `AttachCurrentThread` để lấy `env` hợp lệ cho thread đó:
  ```cpp
  JavaVM* g_vm;                               // lưu trong JNI_OnLoad, cache được
  // trong thread native:
  JNIEnv* env;
  g_vm->AttachCurrentThread(&env, nullptr);   // xin env cho thread này
  // ... gọi CallVoidMethod đẩy dữ liệu lên app ...
  g_vm->DetachCurrentThread();                // nhớ detach kẻo leak
  ```
- **Leak reference = leak trên daemon chạy suốt**: quên `DeleteGlobalRef` / không detach thread trong service xe chạy hàng tuần → tích tụ → OOM (xem lý do ở [memory-management.md](memory-management.md)).
- **Vượt biên JNI có phí**: mỗi lần gọi qua lại Java↔native tốn chi phí. Với luồng dữ liệu xe tần suất cao (CAN đọc liên tục), gom dữ liệu rồi vượt biên **một lần** (dùng mảng/`ByteBuffer` trực tiếp) thay vì gọi từng field.

## Câu hỏi tự kiểm tra

- [ ] JNI giải quyết vấn đề gì? Kể 3 lý do app Android bước xuống native.
- [ ] `JNIEnv*` và `jobject thiz` là gì? Vì sao `JNIEnv*` không được cache sang thread khác?
- [ ] Viết signature JNI cho method `String f(int, long)` và cho `int[]`.
- [ ] Local ref khác global ref chỗ nào? 2 lỗi kinh điển với local ref là gì?
- [ ] Vì sao trong vòng lặp dài phải `DeleteLocalRef`?
- [ ] `RegisterNatives` khác cách đặt tên `Java_...` ra sao? Vì sao AOSP thích cách này?
- [ ] Thread C++ (cảm biến xe) muốn callback lên app Java thì cần cache gì và gọi hàm nào? (bẫy: `JNIEnv*` hay `JavaVM*`?)
- [ ] Vì sao nên gom dữ liệu rồi vượt biên JNI một lần thay vì gọi nhiều lần?

## 📚 Tài liệu

- 📖 [JNI Tips (Google)](https://developer.android.com/training/articles/perf-jni) — best practice JNI trên Android, đọc kỹ phần reference & thread
- 📖 [JNI Specification (Oracle)](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/jniTOC.html) — đặc tả gốc, tra khi cần chi tiết
- 📖 [NDK — Concepts](https://developer.android.com/ndk/guides/concepts) — cách NDK build `.so` nhét vào APK
- 📖 [Car API overview](https://source.android.com/docs/automotive/vhal) — VHAL, đường Java↔C++ trong xe
- 📖 `javap -s` — công cụ JDK in signature chuẩn, khỏi đoán tay

## Đọc tiếp

- [README.md](README.md) — index Phase 2, thứ tự đọc các note tiếp theo
- [memory-management.md](memory-management.md) — nền quản lý bộ nhớ native, đọc trước note này nếu chưa
