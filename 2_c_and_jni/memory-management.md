---
title: Quản lý bộ nhớ trong C/C++
phase: 2 — C/C++ & JNI
tags:
  - C
  - C++
  - Memory
  - Pointer
  - RAII
---

# Quản lý bộ nhớ trong C/C++

> Thuộc **Phase 2 — C/C++ & JNI** trong [roadmap](../android-automotive-developer-roadmap.md). Note **mở đầu Phase 2**: đọc trước khi động vào pointer/struct và JNI.
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## Quản lý bộ nhớ là gì và vì sao C/C++ khác Kotlin/Java?

Ở Kotlin/Java (tầng app/framework) có **Garbage Collector (GC)**: cấp phát object thoải mái, ART tự dọn khi không còn ai tham chiếu. Bạn gần như không bao giờ nghĩ tới "giải phóng bộ nhớ".

Xuống tầng **native/HAL** — nơi C/C++ sống — **không có GC**. Ai cấp phát, người đó phải tự trả. Cấp phát mà quên trả = **memory leak**; trả rồi còn dùng = **crash**. Đây là bước nhảy tư duy lớn nhất khi từ app dev xuống system dev: trình biên dịch không có lưới an toàn, lỗi bộ nhớ không báo lúc compile mà **nổ lúc runtime**, có khi sau nhiều ngày chạy.

Ví dụ dễ hình dung: Java như **thuê khách sạn** — dùng xong cứ đi, dọn phòng có người lo. C/C++ như **thuê nhà** — tự trả tiền, tự dọn, tự khóa cửa; quên cái nào là mất tiền hoặc mất đồ.

## Bản đồ bộ nhớ 1 process

Mọi biến/cấp phát rơi vào 1 trong 4 vùng:

```
Địa chỉ cao
┌───────────────────────────┐
│  Stack                    │  biến local, tham số hàm
│  (mọc xuống ↓)            │  tự cấp/tự thu theo scope hàm — NHANH
├───────────────────────────┤
│         ↓                 │
│         (khoảng trống)    │
│         ↑                 │
├───────────────────────────┤
│  Heap                     │  malloc/new — TỰ TAY quản lý
│  (mọc lên ↑)              │  sống tới khi free/delete
├───────────────────────────┤
│  BSS / Data (static)      │  biến global, static — sống suốt process
├───────────────────────────┤
│  Text / Code              │  mã máy, read-only
└───────────────────────────┘
Địa chỉ thấp
```

## Stack vs Heap — chọn cái nào?

| | Stack | Heap |
|---|-------|------|
| Cấp phát | Tự động khi vào hàm | Tay: `malloc`/`new` |
| Thu hồi | Tự động khi ra hàm | Tay: `free`/`delete` |
| Tốc độ | Rất nhanh (dời con trỏ) | Chậm hơn (tìm block trống) |
| Kích thước | Nhỏ, cố định (vài MB) | Lớn, giới hạn = RAM |
| Vòng đời | Hết scope là mất | Do bạn quyết |
| Bẫy | Trả con trỏ tới biến stack sau khi hàm return = **dangling** | Quên `free` = **leak** |

Nguyên tắc: **mặc định dùng stack**; chỉ lên heap khi cần object sống lâu hơn hàm, hoặc kích thước quá lớn / chưa biết lúc compile.

## Cấp phát tay: C vs C++

```c
// C — cặp malloc/free
int *p = malloc(sizeof(int) * 10);   // cấp 10 int trên heap
if (!p) { /* malloc trả NULL khi hết RAM — PHẢI check */ }
free(p);                              // trả lại
p = NULL;                             // tránh dùng lại con trỏ chết
```

```cpp
// C++ — cặp new/delete (new còn gọi constructor)
int *p   = new int[10];   delete[] p;   // mảng: new[]  ↔  delete[]
Foo *f   = new Foo();     delete f;     // object đơn: new ↔ delete
```

**Quy tắc vàng: mỗi `malloc`/`new` phải có đúng 1 `free`/`delete` khớp cặp** (`new[]` ↔ `delete[]`, không lẫn `free` với `delete`).

## 5 lỗi bộ nhớ kinh điển

| Lỗi | Là gì | Hậu quả |
|-----|-------|---------|
| **Memory leak** | Cấp mà không bao giờ `free` | RAM phình dần → daemon chạy lâu bị kill/crash |
| **Dangling pointer** | Con trỏ trỏ tới vùng đã `free`/đã hết scope | Đọc/ghi rác, crash ngẫu nhiên |
| **Double free** | `free` 2 lần cùng con trỏ | Hỏng heap → crash hoặc lỗ hổng bảo mật |
| **Use-after-free** | Dùng con trỏ sau khi `free` | Undefined behavior, thường bị khai thác |
| **Buffer overflow** | Ghi quá kích thước cấp phát | Đè dữ liệu/stack → crash, RCE |

Điểm chung: **không lỗi nào báo lúc compile**. Phải bắt bằng công cụ (mục Automotive).

## Lối thoát của C++: RAII + smart pointer

**RAII** (Resource Acquisition Is Initialization): buộc vòng đời tài nguyên vào vòng đời một object trên stack. Object hết scope → destructor tự chạy → tự `free`. Biến việc "nhớ giải phóng" thành **tự động**, kể cả khi có exception.

```cpp
#include <memory>

// unique_ptr — sở hữu DUY NHẤT, tự delete khi ra scope. Zero overhead.
std::unique_ptr<Foo> a = std::make_unique<Foo>();
// hết scope → ~Foo() tự chạy, không cần delete

// shared_ptr — sở hữu CHUNG, đếm tham chiếu; về 0 mới delete
std::shared_ptr<Foo> b = std::make_shared<Foo>();

// weak_ptr — "mượn xem" không giữ; phá vòng lặp shared_ptr ↔ shared_ptr (chu trình = leak)
std::weak_ptr<Foo> w = b;
```

| Smart pointer | Sở hữu | Dùng khi |
|---------------|--------|----------|
| `unique_ptr` | 1 chủ | Mặc định — hầu hết trường hợp |
| `shared_ptr` | Nhiều chủ | Nhiều nơi cùng cần giữ object sống |
| `weak_ptr`  | Không giữ | Bẻ chu trình tham chiếu, cache |

Quy tắc hiện đại (C++11+): **ưu tiên `make_unique`/`make_shared`, gần như không `new`/`delete` tay**. Code AOSP native cũng theo hướng này (thêm `sp<>`/`wp<>` của libutils — smart pointer riêng của Android).

## 🚗 Liên hệ Automotive

- **HAL/daemon chạy suốt đời xe**: VHAL, EVS, audio HAL... là process C++ chạy liên tục hàng **tuần/tháng** không restart. Leak vài KB mỗi giây trong app điện thoại thì user reboot là hết; trên xe thì tích tụ → OOM kill → **mất tính năng an toàn** (camera lùi, cảnh báo). Native leak nguy hiểm hơn nhiều vì **không có GC dọn hộ**.
- **Không có "reboot cho nhanh"**: xe kỳ vọng uptime cao; bug bộ nhớ phải diệt từ gốc, không dựa vào restart.
- **Công cụ bắt lỗi bắt buộc biết**:
  ```bash
  # AddressSanitizer — bật khi build NDK/AOSP, bắt use-after-free / overflow tại runtime
  # Android.bp:  sanitize: { address: true }
  # Valgrind (host/emulator) tìm leak:
  valgrind --leak-check=full ./my_native_binary
  ```
- **NDK build**: native code Android biên dịch bằng NDK + CMake/`Android.bp`; smart pointer + ASan là combo chuẩn để native HAL không rò rỉ.

## Câu hỏi tự kiểm tra

- [ ] Vì sao C/C++ không có GC còn Kotlin/Java có? Hệ quả với dev?
- [ ] 4 vùng bộ nhớ của process là gì? Biến local nằm đâu, `new` nằm đâu?
- [ ] Khi nào dùng stack, khi nào lên heap?
- [ ] Kể 5 lỗi bộ nhớ kinh điển — lỗi nào báo lúc compile? (bẫy)
- [ ] RAII giải quyết vấn đề gì? `unique_ptr` khác `shared_ptr` chỗ nào?
- [ ] `weak_ptr` sinh ra để chữa bệnh gì của `shared_ptr`?
- [ ] Vì sao leak trên HAL xe nguy hiểm hơn leak trong app điện thoại?

## 📚 Tài liệu

- 📖 [learncpp.com — Dynamic memory](https://www.learncpp.com/cpp-tutorial/dynamic-memory-allocation-with-new-and-delete/) — new/delete từ zero
- 📖 [learncpp.com — Smart pointers & RAII](https://www.learncpp.com/cpp-tutorial/introduction-to-smart-pointers-move-semantics/)
- 📖 [cppreference — std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr) / [shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)
- 📖 Sách **"The C Programming Language"** (K&R) — chương 5 (con trỏ) + 8.7 (malloc)
- 📖 [AddressSanitizer trên Android](https://source.android.com/docs/security/test/asan) — docs chính thức bắt lỗi bộ nhớ native
- 📖 [Android sp<>/wp<> (libutils StrongPointer)](https://cs.android.com/android/platform/superproject/main/+/main:system/core/libutils/include/utils/StrongPointer.h) — smart pointer riêng của AOSP

## Đọc tiếp

- [README.md](README.md) — index Phase 2, thứ tự đọc các note tiếp theo
- [zygote.md](../1_android_internal/zygote.md) — note Phase 1, copy-on-write cũng là chuyện quản lý page bộ nhớ ở tầng OS
