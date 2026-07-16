---
title: Pointers & struct — cơ khí của C
source: https://en.cppreference.com/w/c/language/pointer
phase: 2 — C/C++ & JNI
tags:
  - C
  - Pointer
  - Struct
  - Union
  - Native
---

# Pointers & struct — cơ khí của C

> Thuộc **Phase 2 — C/C++ & JNI** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc **sau** [memory-management.md](memory-management.md) (đã nói vùng nhớ, dangling, `malloc`/`free` — note này không lặp) và **trước** [jni.md](jni.md): con trỏ + struct là thứ JNI và mọi API C của HAL trao đổi qua lại.
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## Con trỏ là gì và vì sao C không sống thiếu nó?

**Con trỏ (pointer) là biến chứa địa chỉ của biến khác** — không chứa giá trị, mà chứa *chỗ ngồi* của giá trị trong bộ nhớ. C dùng con trỏ ở khắp nơi vì nó là cách duy nhất để: sửa biến của hàm gọi (không có "pass by reference" như C++), duyệt mảng/chuỗi, cấp phát heap, và truyền cấu trúc lớn mà không copy.

Java giấu hết chuyện này (mọi object là reference ẩn, không số học địa chỉ). C phơi bày → mạnh nhưng dễ tự bắn chân. Đây là lý do phải nắm chắc trước khi đụng JNI: bên native, mỗi `jobject`, mỗi mảng, mỗi handle phần cứng đều là con trỏ.

## `&` và `*` — hai chiều ngược nhau

```c
int x = 42;
int* p = &x;      // & = "địa chỉ của" → p giữ chỗ ngồi của x
printf("%d", *p); // * = "giá trị tại" (dereference) → in 42
*p = 99;          // ghi qua con trỏ → x giờ = 99
```

- `&x` = **lấy địa chỉ** của x.
- `*p` = **truy cập giá trị** tại địa chỉ p đang trỏ.
- `int* p` — dấu `*` lúc khai báo nghĩa là "p là con trỏ tới int", khác `*p` lúc dùng (deref). Cùng ký hiệu, 2 vai.

**Sửa biến hàm gọi** — dùng con trỏ vì C truyền tham số theo copy:

```c
void inc(int* n) { (*n)++; }   // nhận địa chỉ, sửa giá trị tại đó
int a = 5; inc(&a);            // a = 6. Nếu void inc(int n) thì a vẫn = 5
```

## Con trỏ + mảng: số học địa chỉ

Tên mảng ≈ con trỏ tới phần tử đầu. `p + 1` **không** cộng 1 byte — cộng `sizeof(phần tử)`:

```c
int arr[3] = {10, 20, 30};
int* p = arr;        // p → arr[0]
*(p + 2);            // = 30. Trình dịch tự nhân 2 * sizeof(int)
arr[2] == *(arr + 2) // arr[i] chỉ là đường ngắn của *(arr + i)
```

Bẫy kinh điển: đi quá cuối mảng (`p[3]` ở trên) = **buffer overflow**, đọc/ghi rác. Không có bound check như Java. → [memory-management.md](memory-management.md) mục 5 lỗi.

## Function pointer — trỏ tới hàm

Con trỏ giữ được cả **địa chỉ hàm** → truyền hàm như dữ liệu. Đây là cách C làm **callback**:

```c
int add(int a, int b) { return a + b; }

int (*op)(int, int) = add;   // op = con trỏ hàm nhận (int,int) trả int
op(2, 3);                    // = 5, gọi qua con trỏ

// dùng thật: bảng callback
void on_event(int code, void (*handler)(int)) {
    handler(code);           // gọi lại hàm caller đưa vào
}
```

Cú pháp `int (*op)(int, int)`: dấu ngoặc quanh `*op` bắt buộc — thiếu nó thành "hàm trả về con trỏ int". Struct chứa nhiều function pointer = **vtable/ops table** — chính là cách HAL kiểu C (HIDL/legacy) expose API cho framework.

## struct — gom nhiều field thành 1 kiểu

```c
struct Point { int x; int y; };

struct Point pt = {3, 4};
pt.x;                        // truy cập field qua "." khi có object
struct Point* pp = &pt;
pp->y;                       // qua con trỏ dùng "->" ; pp->y ≡ (*pp).y
```

- `.` khi bạn cầm **object**, `->` khi cầm **con trỏ tới object**. Nhầm = lỗi compile.
- Truyền struct lớn nên truyền **con trỏ** (`struct Point*`) để khỏi copy cả khối.
- `sizeof(struct)` **≥ tổng sizeof field** vì trình dịch chèn **padding** cho đúng alignment. Đừng giả định layout khít — quan trọng khi map struct qua ranh giới (JNI, đọc/ghi binary, khung CAN).

## union — nhiều field chung 1 ô nhớ

```c
union Value {
    int32_t i;
    float f;
    int64_t l;
};   // sizeof = field lớn nhất (8), KHÔNG phải tổng
```

Mọi field **đè lên cùng vùng nhớ** — ghi `i` rồi đọc `f` = đọc lại cùng bit dưới dạng khác. Dùng khi 1 giá trị có thể là 1 trong nhiều kiểu, tiết kiệm nhớ. Người dùng phải tự nhớ đang giữ kiểu nào (thường kèm 1 field `type` bên cạnh → gọi là *tagged union*).

## 🚗 Liên hệ Automotive

- **VHAL `RawPropValues`** chứa nhiều mảng (`int32Values`, `floatValues`, `int64Values`, `byteValues`) — 1 property có thể mang kiểu khác nhau, đúng tinh thần tagged union/struct. Đọc/ghi property xe = điền đúng field trong struct này.
- **Khung CAN** là struct nhị phân cố định (id + dlc + 8 byte data). Parse sai **padding/alignment** = đọc lệch field → hiểu sai tín hiệu xe. Dùng `__attribute__((packed))` để ép struct khít khi map trực tiếp lên buffer phần cứng.
- **`jlong` ôm con trỏ**: JNI hay giấu con trỏ C++ trong 1 `long` bên Kotlin (đối tượng native tồn tại qua nhiều lời gọi). Ép kiểu con trỏ ↔ số nguyên là chuyện thường ở đây — xem [jni-tips.md](jni-tips.md).
- **Ops table kiểu function pointer**: HAL C cũ expose struct đầy con trỏ hàm cho framework gọi. AIDL/HIDL đời mới thay bằng interface có kiểu (xem [java-binder-aidl.md](java-binder-aidl.md)), nhưng đọc code HAL cũ vẫn gặp.

Xem địa chỉ/con trỏ khi debug native:

```bash
adb shell setprop libc.debug.malloc.options guard   # bật guard bắt overflow con trỏ
# hoặc build với ASan (xem ndk-cmake.md) — bắt use-after-free / out-of-bounds
```

## Câu hỏi tự kiểm tra

- [ ] `&x` và `*p` khác nhau thế nào? `int* p` lúc khai báo so với `*p` lúc dùng?
- [ ] Vì sao muốn 1 hàm sửa được biến của hàm gọi, C bắt truyền con trỏ?
- [ ] `p + 1` dịch con trỏ đi bao nhiêu byte? `arr[i]` viết lại bằng con trỏ ra sao?
- [ ] Khi nào dùng `.`, khi nào dùng `->`?
- [ ] `sizeof(struct)` có bằng tổng sizeof các field không? Vì sao? Ảnh hưởng gì khi map struct qua ranh giới?
- [ ] union khác struct ở chỗ nào về bộ nhớ? Vì sao cần thêm field `type` đi kèm?
- [ ] Function pointer khai báo thế nào, dùng để làm gì trong HAL?

## 📚 Tài liệu

- [cppreference — Pointers](https://en.cppreference.com/w/c/language/pointer) — spec con trỏ C
- [cppreference — struct](https://en.cppreference.com/w/c/language/struct) & [union](https://en.cppreference.com/w/c/language/union)
- 📖 Sách **"The C Programming Language"** (K&R) — chương 5 (con trỏ & mảng) + chương 6 (struct/union)
- [VHAL types (cs.android.com)](https://cs.android.com/android/platform/superproject/main/+/main:hardware/interfaces/automotive/vehicle/) — struct property thật của xe

## Đọc tiếp

- [jni.md](jni.md) — JNI trao đổi toàn con trỏ (`JNIEnv*`, `jobject`); note này là nền để đọc chữ ký hàm JNI.
- [jni-tips.md](jni-tips.md) — mẹo ôm con trỏ C++ trong `jlong`, mảng region — số học con trỏ áp dụng thẳng.
- [cpp-basics.md](cpp-basics.md) — C++ thêm **reference** (`Foo&`) bên cạnh con trỏ; so sánh 2 cách "trỏ".
