---
title: JNI Tips — cạm bẫy & best practice
phase: 2 — C/C++ & JNI
tags:
  - JNI
  - Best-Practice
  - Thread
  - Exception
  - Performance
---

# JNI Tips — cạm bẫy & best practice

> Thuộc **Phase 2 — C/C++ & JNI** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc **sau** [jni.md](jni.md) — note kia dạy JNI *hoạt động thế nào*; note này chắt lọc [JNI Tips (Google)](https://developer.android.com/training/articles/perf-jni) về những chỗ **dễ sai nhất trong thực tế** mà bản cơ bản chưa đào sâu. Không lặp lại cơ bản.
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## Vì sao cần note riêng cho "tips"?

JNI biên dịch trót lọt nhưng **lỗi chỉ nổ lúc runtime**, thường ngẫu nhiên và khó lần. Google gom các cạm bẫy này thành một trang best-practice vì đa số bug JNI đến từ **cùng vài lỗi lặp đi lặp lại**. Nắm trước thì tiết kiệm hàng giờ debug tombstone.

## Cache ID (`jclass`/`jmethodID`/`jfieldID`) — đừng tra lại mỗi lần

`GetMethodID` / `GetFieldID` / `FindClass` **tốn kém** (tra bảng theo tên + signature). ID **không đổi** trong suốt đời class → tra **một lần**, cache lại.

```cpp
// ĐÚNG: cache trong JNI_OnLoad hoặc lần gọi đầu
static jmethodID g_onData;   // ID là số, cache thẳng an toàn
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void*) {
    JNIEnv* env; vm->GetEnv((void**)&env, JNI_VERSION_1_6);
    jclass c = env->FindClass("com/example/Bridge");
    g_onData = env->GetMethodID(c, "onData", "(I)V");   // tra 1 lần
    return JNI_VERSION_1_6;
}
```

**Bẫy khớp với reference**: `jmethodID`/`jfieldID` là **số ID** → cache thẳng OK. Nhưng `jclass` là một **object reference** → cache local ref là chết (xem [jni.md](jni.md)); muốn giữ `jclass` phải `NewGlobalRef`.

## `FindClass` trong thread native → sai classloader

Cạm bẫy nặng và khó đoán: `FindClass` tìm class bằng **classloader của frame đang gọi**. Thread do Java gọi xuống thì đúng. Nhưng thread **C++ tự tạo** (cảm biến, CAN — xem callback xuyên thread ở [jni.md](jni.md)) chỉ có classloader hệ thống → **không thấy class app** → `ClassNotFoundException`.

Cách chữa: **tra + cache `jclass` (global ref) lúc `JNI_OnLoad`** — khi đó classloader còn đúng — rồi dùng ID/class đã cache trong thread native.

## Exception: PHẢI check, không tự nhảy như Java

JNI **không có try/catch**. Hàm Java gọi qua JNI ném exception thì exception **treo lơ lửng** — code C++ vẫn chạy tiếp dòng sau. Gọi tiếp hầu hết hàm JNI khi có exception treo = **crash**.

```cpp
env->CallVoidMethod(obj, g_method);
if (env->ExceptionCheck()) {          // PHẢI tự hỏi: vừa rồi có ném không?
    env->ExceptionDescribe();         // in ra logcat (debug)
    env->ExceptionClear();            // xóa trước khi gọi JNI tiếp
    // ... xử lý lỗi ...
}
```

Chỉ vài hàm được phép gọi khi có exception treo (`ExceptionCheck`, `ExceptionClear`, các `Release*`, `DeleteLocalRef`...). Ngoài ra phải clear trước.

## String: cẩn thận "modified UTF-8"

`GetStringUTFChars` **không trả UTF-8 chuẩn** mà là **modified UTF-8** của JVM (mã hóa khác ở ký tự NUL và ngoài BMP). Nhét thẳng vào thư viện C mong UTF-8 chuẩn = **sai với emoji / ký tự ngoài BMP**.

```cpp
const char* s = env->GetStringUTFChars(js, nullptr);
// ... dùng s ...
env->ReleaseStringUTFChars(js, s);   // BẮT BUỘC release, kẻo leak
```

- Cần chuẩn Unicode thật → dùng `GetStringChars` (UTF-16) rồi tự chuyển.
- `NewStringUTF` với chuỗi **không phải modified UTF-8 hợp lệ** → **abort cả process**. Đừng đưa dữ liệu nhị phân thô vào.
- Mọi `Get...Chars` phải có `Release...` khớp cặp (giống malloc/free của [memory-management.md](memory-management.md)).

## Mảng: pin/copy và ưu tiên "Region"

`GetIntArrayElements` trả con trỏ tới phần tử — runtime **có thể copy hoặc pin** (giữ nguyên chỗ). Phải `Release` với mode đúng:

| Mode release | Nghĩa |
|--------------|-------|
| `0` | Copy ngược về Java + giải phóng bản C |
| `JNI_COMMIT` | Copy ngược nhưng **không** giải phóng (dùng giữa chừng) |
| `JNI_ABORT` | Bỏ thay đổi, giải phóng |

Với đọc/ghi **một khúc nhỏ**, ưu tiên **Region call** — gọn, không lo pin, không phải release:

```cpp
jint buf[4];
env->GetIntArrayRegion(arr, 0, 4, buf);   // copy 4 phần tử ra buf, xong luôn
// env->SetIntArrayRegion(...) để ghi ngược
```

## Thread & reference — nhắc lại kỷ luật

Đã nói ở [jni.md](jni.md), gom lại thành checklist vì đây là nguồn crash số 1:

- `JNIEnv*` **theo thread** — không truyền/cache sang thread khác. `JavaVM*` mới cache được.
- Thread C++ tự tạo: `AttachCurrentThread` để lấy `env`, xong **`DetachCurrentThread`** (quên = leak, có khi treo lúc thoát). Mẹo tự động: `pthread_key_create` với destructor gọi `DetachCurrentThread` khi thread chết.
- Local ref hết hàm là mất; giữ lâu → `NewGlobalRef` (nhớ `DeleteGlobalRef`).
- Loop dài tạo nhiều local ref → `DeleteLocalRef` từng vòng, hoặc `EnsureLocalCapacity`/`PushLocalFrame`+`PopLocalFrame` báo trước.
- **Không so reference bằng `==`** — 2 ref khác giá trị vẫn có thể trỏ cùng object. Dùng `env->IsSameObject(a, b)`.

## Visibility: `-fvisibility=hidden` giấu mất hàm

Nếu build với `-fvisibility=hidden` (phổ biến để `.so` gọn/nhanh) mà hàm JNI **không có `JNIEXPORT`**, symbol bị ẩn → `UnsatisfiedLinkError` runtime dù đặt tên đúng. Luôn gắn `extern "C" JNIEXPORT` cho hàm nối tên `Java_...`.

## Con trỏ native lưu trong Java → dùng `jlong`

Muốn giữ con trỏ C++ (đối tượng native) qua các lần gọi, lưu nó ở field Java kiểu **`long`** (`jlong`), không phải `int` — `int` 32-bit **cắt cụt con trỏ trên máy 64-bit** (mọi thiết bị xe/điện thoại hiện đại đều 64-bit).

```cpp
// C++: trả con trỏ về Java giữ hộ
jlong handle = reinterpret_cast<jlong>(new Engine());
// lần sau lấy lại:
Engine* e = reinterpret_cast<Engine*>(handle);
// nhớ có đường delete: destroy(handle) → delete e;  (không GC lo hộ)
```

## Bật CheckJNI khi phát triển

CheckJNI là "chế độ nghiêm" của runtime: bắt **ngay** khi phạm luật (ref sai, `env` nhầm thread, Get/Release lệch cặp, UTF-8 hỏng, exception treo) thay vì để crash lạc chỗ sau đó. **Bật suốt lúc dev.**

```bash
adb shell setprop debug.checkjni 1          # áp cho lần khởi chạy sau
# hoặc bật toàn runtime:
adb shell stop && adb shell setprop dalvik.vm.checkjni true && adb shell start
```

Emulator thường bật sẵn. Nó chậm hơn → tắt khi đo hiệu năng/release.

## 🚗 Liên hệ Automotive

- **`FindClass` sai classloader = bug kinh điển khi đẩy dữ liệu xe lên app**: luồng CAN/cảm biến chạy thread C++ riêng; quên cache `jclass` lúc `JNI_OnLoad` → callback lên app "class not found" tuy code đúng. Đây là lý do phải cache sớm.
- **Quên `DetachCurrentThread` trên daemon xe**: service chạy hàng tuần, mỗi lần attach mà không detach tích tụ → leak/treo lúc shutdown. Nối tiếp bài học leak ở [memory-management.md](memory-management.md).
- **Bỏ qua `ExceptionCheck` = crash ngẫu nhiên khó tái hiện**: dữ liệu xe lỗi làm method Java ném; native không check, chạy tiếp rồi sập ở chỗ khác → tombstone lạc hướng, tốn giờ lần.
- **`jlong` cho con trỏ**: head unit AAOS 64-bit — lưu handle native bằng `int` là cắt cụt, crash chắc chắn.

## Câu hỏi tự kiểm tra

- [ ] Vì sao cache `jmethodID`? `jclass` cache khác `jmethodID` chỗ nào? (bẫy reference)
- [ ] `FindClass` trong thread C++ tự tạo hỏng vì sao? Chữa thế nào?
- [ ] JNI không có try/catch — sau khi gọi method Java có thể ném, phải làm gì trước khi gọi JNI tiếp?
- [ ] "Modified UTF-8" là gì? `GetStringUTFChars` cho emoji có an toàn không?
- [ ] `NewStringUTF` với dữ liệu nhị phân thô → chuyện gì xảy ra?
- [ ] Region call (`GetIntArrayRegion`) hơn `GetIntArrayElements` ở điểm nào?
- [ ] Thread C++ callback lên Java: attach xong quên gì thì leak?
- [ ] Vì sao con trỏ native lưu trong Java phải là `long` chứ không `int`?
- [ ] `JNIEXPORT` + `-fvisibility=hidden` liên quan gì tới `UnsatisfiedLinkError`?
- [ ] Vì sao không so 2 reference bằng `==`? Dùng gì thay?
- [ ] CheckJNI bắt loại lỗi nào? Vì sao bật lúc dev, tắt lúc đo hiệu năng?

## 📚 Tài liệu

- 📖 [JNI Tips (Google)](https://developer.android.com/training/articles/perf-jni) — trang gốc note này chắt lọc, đọc trọn ít nhất 1 lần
- 📖 [JNI Specification — Exceptions](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/design.html) — quy tắc exception & reference chi tiết
- 📖 [Modified UTF-8 (JNI spec)](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/types.html#modified_utf_8_strings) — vì sao khác UTF-8 chuẩn
- 📖 [CheckJNI](https://developer.android.com/ndk/guides/checkjni) — bật để runtime bắt sai reference/exception/thread ngay khi phạm

## Đọc tiếp

- [README.md](README.md) — index Phase 2, thứ tự đọc các note tiếp theo
- [jni.md](jni.md) — cơ bản JNI (đọc trước note này)
- [ndk-cmake.md](ndk-cmake.md) — bật CheckJNI/ASan lúc build để bắt các lỗi trên
