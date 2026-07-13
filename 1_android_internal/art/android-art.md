---
title: ART (Android Runtime)
source: https://source.android.com/docs/core/runtime?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - ART
  - Dalvik
  - DEX
---

# ART (Android Runtime)

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/runtime](https://source.android.com/docs/core/runtime?hl=vi)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../../android-automotive-developer-roadmap.md). Đọc sau [android-architecture.md](../android-architecture.md).

## ART là gì?

**ART (Android Runtime)** là **managed runtime** — lớp phần mềm đứng giữa app của bạn và CPU/OS để chạy app Android. "Managed" nghĩa là runtime tự lo bộ nhớ (garbage collection), kiểm tra kiểu, xử lý exception… thay cho lập trình viên — giống JVM lo cho app Java trên desktop. Khi làm app, bạn viết Kotlin/Java rồi bấm Run; ART chính là cái thực sự nạp và thực thi code đó trên máy.

Vì sao Android cần một runtime *riêng* thay vì dùng thẳng JVM? Điện thoại/xe hơi có RAM ít, pin hạn chế và CPU yếu hơn máy chủ. Google thiết kế ART chạy **DEX bytecode** (Dalvik Executable) — một định dạng bytecode gọn hơn `.class` của JVM, tối ưu cho thiết bị nhiều app cùng chạy. ART **thay thế Dalvik** (runtime gốc của Android, dùng đến Android 4.4), điểm khác cốt lõi là cách nó biến bytecode thành machine code.

Ba chiến lược compile bạn sẽ gặp suốt chuỗi note này, ở mức khái quát: **AOT** (Ahead-Of-Time) compile *trước khi chạy* — như biên dịch sẵn ra file thực thi, chạy nhanh nhưng tốn dung lượng và thời gian compile; **JIT** (Just-In-Time) compile *ngay lúc chạy* những đoạn code hay dùng — như dịch tại chỗ khi cần; **interpreter** đọc và thực thi từng lệnh bytecode mà không compile — chậm nhất nhưng khởi động tức thì. ART kết hợp cả ba (chi tiết ở [art-jit.md](art-jit.md)).

## Luồng compile: Kotlin → chạy trên máy

```
Kotlin/Java source
   │ kotlinc/javac
   ▼
JVM bytecode (.class)
   │ D8/R8
   ▼
DEX bytecode (.dex, đóng gói trong APK)
   │ dex2oat (AOT — lúc cài hoặc chạy nền sau) + JIT (lúc chạy)
   ▼
Native machine code
```

Phân biệt 3 cái tên gần giống nhau (gặp suốt chuỗi ART):

| Tên | Là gì |
|-----|-------|
| **dex2oat** | *Tool* compile DEX → machine code |
| **dexopt** | *Việc* compile trên thiết bị (dex2oat thực hiện): lúc cài, lúc idle, sau OTA |
| **dexpreopt** | Compile sẵn *lúc build image* — app hệ thống lên máy đã có code compile |

- **Interpreter** = chạy bytecode bằng cách dịch từng lệnh lúc runtime, không compile trước — chậm nhưng chạy được ngay, là trạng thái đầu tiên của mọi method (xem [art-jit.md](art-jit.md)).

## ART vs Dalvik

| | Dalvik (cũ) | ART |
|--|-------------|-----|
| Compile | JIT thuần (mỗi lần chạy) | AOT + JIT + profile-guided |
| Tốc độ chạy | Chậm hơn | Nhanh hơn (đã compile sẵn) |
| GC | Pause dài | Concurrent, pause ngắn |
| Tương thích | — | App viết cho Dalvik chạy được trên ART |

## Tính năng chính

### AOT compilation
- **`dex2oat`** compile app thành binary tối ưu cho đúng thiết bị — lúc cài đặt **hoặc** chạy nền sau theo profile (mặc định hiện đại: lúc cài chỉ verify, AOT dần khi máy idle — xem [art-jit.md](art-jit.md)).
- Thực tế hiện đại: kết hợp AOT + JIT + **profile-guided compilation** — hot code được compile dần theo profile sử dụng.

### Garbage Collection cải tiến
- GC **chủ yếu concurrent** (chạy song song với app), concurrent copying.
- **Pause time không phụ thuộc heap size** — heap to không làm khựng lâu hơn.

### Debug hỗ trợ tốt hơn
- Sampling profiler chuyên dụng
- Lock tracking, class instance inspection
- Message lỗi chi tiết hơn cho `NullPointerException`, `ClassCastException` (chỉ rõ field/method nào null)

## ART là Mainline module

- Từ Android 12, ART phân phối qua **Mainline** (APEX module) → Google update ART **độc lập với OS release**, qua Play.

🚗 *Liên hệ Automotive:* AAOS boot chậm = vấn đề thật (driver ngồi chờ). Hiểu AOT/JIT giúp tối ưu: preload class trong Zygote, dexpreopt system app lúc build image (compile sẵn khi build AOSP thay vì lúc cài) — kỹ thuật OEM hay dùng để giảm cold start cho launcher/cluster app.

## Câu hỏi tự kiểm tra

- [ ] DEX bytecode là gì, khác JVM bytecode chỗ nào trong pipeline?
- [ ] ART khác Dalvik về chiến lược compile thế nào?
- [ ] `dex2oat` chạy lúc nào, sinh ra gì?
- [ ] GC của ART cải tiến gì so với Dalvik?
- [ ] ART update qua cơ chế nào từ Android 12?

## 📚 Tài liệu

- 📖 [Cấu hình ART](https://source.android.com/docs/core/runtime/configure)
- 📖 [JIT compiler trong ART](https://source.android.com/docs/core/runtime/jit-compiler)
- 📖 [Dexpreopt / boot image](https://source.android.com/docs/core/runtime/boot-image)

## Đọc tiếp

- [../android-architecture.md](../android-architecture.md) — vị trí ART trong stack
- [art-configure.md](art-configure.md) — compilation filters, dexpreopt
- [art-jit.md](art-jit.md) — JIT chi tiết
