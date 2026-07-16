---
title: C++ cơ bản — class, STL, template
phase: 2 — C/C++ & JNI
tags:
  - C++
  - Class
  - STL
  - Template
  - OOP
---

# C++ cơ bản — class, STL, template

> Thuộc **Phase 2 — C/C++ & JNI** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc **sau** [memory-management.md](memory-management.md) (đã nói kỹ RAII + smart pointer — note này không lặp, chỉ nối). Là nền để đọc code C++ của HAL/AOSP và viết được lớp native gọi qua [jni.md](jni.md).
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## C++ khác Kotlin/Java ở tư duy nào?

Bạn đã quen OOP từ Kotlin/Java. C++ cũng OOP nhưng **thêm 3 khác biệt cốt lõi** khiến code trông lạ:

1. **Không GC** — object có thể nằm trên **stack** (tự chết cuối scope) chứ không phải luôn trên heap như Java. Đây là gốc của RAII (xem [memory-management.md](memory-management.md)).
2. **Value semantics mặc định** — gán/truyền object là **copy** cả nội dung, không copy reference như Java. Muốn chia sẻ phải chủ động dùng con trỏ/reference.
3. **Header/source tách đôi** — khai báo ở `.h`, cài đặt ở `.cpp`. Java gộp làm một.

Ví dụ khác biệt value vs reference:

```cpp
Foo a;
Foo b = a;   // b là BẢN SAO độc lập của a (Java: b trỏ cùng object với a)
```

## Class: khai báo (.h) + cài đặt (.cpp)

```cpp
// foo.h — khai báo
class Foo {
public:
    Foo(int x);          // constructor
    ~Foo();              // DESTRUCTOR — Java không có; chạy tự động khi object chết
    int getValue() const;   // const = hàm không sửa object

private:
    int value_;          // quy ước AOSP: field private có hậu tố _
};
```

```cpp
// foo.cpp — cài đặt
#include "foo.h"

Foo::Foo(int x) : value_(x) {}   // : value_(x) = member initializer list (khởi tạo field)
Foo::~Foo() {}                    // dọn tài nguyên ở đây (đóng file, free...)
int Foo::getValue() const { return value_; }
```

**Destructor `~Foo()`** là điểm khác Java lớn nhất: nó chạy **tự động** khi object hết scope (stack) hoặc bị `delete` (heap). Chính cơ chế này làm RAII hoạt động.

## Reference vs Pointer — 2 cách "trỏ"

| | Pointer `Foo*` | Reference `Foo&` |
|---|---------------|------------------|
| Có thể null | Có | Không (luôn trỏ vật thật) |
| Đổi target | Được (`p = &other`) | Không, gắn 1 lần |
| Cú pháp | `p->x`, `*p` | `r.x` như biến thường |
| Dùng khi | Có thể vắng mặt, cấp phát heap | Tham số/trả về "mượn xem", không copy |

```cpp
void print(const Foo& f);   // truyền reference + const = không copy, không sửa. KIỂU chuẩn truyền object
```

Mặc định truyền object nặng bằng `const&` để tránh copy tốn kém.

## STL — thư viện chuẩn (như Collections của Java)

Không tự viết mảng động/map. STL có sẵn:

| STL | Tương đương Java | Dùng khi |
|-----|------------------|----------|
| `std::vector<T>` | `ArrayList<T>` | Mảng động — mặc định chọn cái này |
| `std::string` | `String` | Chuỗi (đừng dùng `char*` kiểu C nếu tránh được) |
| `std::map<K,V>` | `TreeMap` | Map có thứ tự (cây, O(log n)) |
| `std::unordered_map<K,V>` | `HashMap` | Map băm, O(1) trung bình |
| `std::array<T,N>` | mảng cố định | Kích thước biết lúc compile |

```cpp
#include <vector>
#include <string>

std::vector<int> v = {1, 2, 3};
v.push_back(4);
for (int x : v) { /* range-based for, giống for(x : list) Java */ }

std::string s = "xe";
s += " hơi";                 // nối chuỗi thoải mái, tự quản bộ nhớ (RAII bên trong)
```

STL container tự quản bộ nhớ heap bên trong bằng RAII → dùng `vector`/`string` thì **gần như không phải `new`/`delete` tay**.

## Template — generic của C++

`template` = viết code cho **kiểu bất kỳ**, giống generic `<T>` Java. Khác lớn: template **sinh code riêng cho mỗi kiểu lúc compile** (không xóa kiểu như Java erasure) → nhanh, nhưng lỗi báo dài dòng.

```cpp
template <typename T>
T maxOf(T a, T b) { return a > b ? a : b; }

maxOf(3, 5);        // compiler sinh bản int
maxOf(1.2, 3.4);    // sinh bản double
```

Giai đoạn này chỉ cần **đọc hiểu** template (STL đầy template); viết template phức tạp để sau.

## const & namespace — 2 thứ gặp liên tục trong code AOSP

```cpp
const int MAX = 10;          // hằng
void f(const Foo& x);        // x không bị sửa trong f
int getValue() const;        // method không sửa object

namespace android {          // AOSP gói mọi thứ trong namespace, tránh trùng tên
    class Foo { ... };
}
android::Foo obj;            // gọi qua ::
```

`const` không chỉ là "hằng" — nó là **lời hứa không sửa**, compiler ép tuân. Code HAL/AOSP rải `const` khắp nơi; hiểu nó mới đọc trôi.

## 🚗 Liên hệ Automotive

- **Code HAL/framework xe là C++ hiện đại**: VHAL, EVS, audio HAL viết bằng class + STL + smart pointer + `const` dày đặc. Đọc được các thành phần trên là điều kiện để sửa/mở rộng chúng.
- **AOSP có STL/smart pointer riêng**: ngoài STL chuẩn, libutils của Android có `Vector`, `String8`, và `sp<>`/`wp<>` (strong/weak pointer — xem [memory-management.md](memory-management.md)). Cùng ý tưởng RAII, khác tên. Biết STL chuẩn trước thì đọc bản Android dễ.
- **Value semantics + xe = cẩn thận copy**: object cấu hình/frame dữ liệu lớn nếu vô tình copy (quên `&`) sẽ tốn CPU/bộ nhớ trên head unit tài nguyên hạn chế. Truyền `const&` là phản xạ.
- **Destructor = nơi trả tài nguyên phần cứng**: mở handle cảm biến/bus trong constructor, đóng trong destructor (RAII) → không rò handle khi service xe chạy dài.

## Câu hỏi tự kiểm tra

- [ ] 3 khác biệt tư duy giữa C++ và Kotlin/Java là gì?
- [ ] `Foo b = a;` trong C++ làm gì? Khác Java chỗ nào? (bẫy value semantics)
- [ ] Destructor `~Foo()` chạy khi nào? Vì sao nó là gốc của RAII?
- [ ] Pointer khác reference ở điểm nào? Khi truyền object nặng nên dùng gì?
- [ ] `vector`, `unordered_map`, `string` tương đương gì bên Java?
- [ ] Vì sao dùng STL thì hầu như không phải `new`/`delete` tay?
- [ ] Template khác generic Java ra sao (erasure vs sinh code)?
- [ ] `const` sau chữ ký method (`int getValue() const`) nghĩa là gì?
- [ ] Vì sao truyền `const&` là phản xạ khi làm việc trên head unit xe?

## 📚 Tài liệu

- 📖 [learncpp.com](https://www.learncpp.com/) — học C++ từ zero, chương Classes + STL đọc kỹ
- 📖 [cppreference — Containers library](https://en.cppreference.com/w/cpp/container) — tra `vector`/`map`/`string`
- 📖 [cppreference — Templates](https://en.cppreference.com/w/cpp/language/templates) — đặc tả template
- 📖 [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html) — AOSP theo style này (naming, const, header)
- 📖 [Android libutils (String8, Vector, sp<>)](https://cs.android.com/android/platform/superproject/main/+/main:system/core/libutils/) — bản STL/smart pointer riêng của AOSP

## Đọc tiếp

- [README.md](README.md) — index Phase 2, thứ tự đọc các note tiếp theo
- [memory-management.md](memory-management.md) — RAII + smart pointer (nối thẳng destructor ở note này)
- [jni.md](jni.md) — dùng lớp C++ vừa học để nối với Java
