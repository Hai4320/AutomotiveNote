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

- **Managed runtime** — môi trường thực thi của app Android.
- Chạy **DEX bytecode** (Dalvik Executable).
- Thay thế **Dalvik** (runtime gốc của Android, đến Android 4.4).

## Luồng compile: Kotlin → chạy trên máy

```
Kotlin/Java source
   │ kotlinc/javac
   ▼
JVM bytecode (.class)
   │ D8/R8
   ▼
DEX bytecode (.dex, đóng gói trong APK)
   │ dex2oat (AOT, lúc cài đặt) + JIT (lúc chạy)
   ▼
Native machine code
```

## ART vs Dalvik

| | Dalvik (cũ) | ART |
|--|-------------|-----|
| Compile | JIT thuần (mỗi lần chạy) | AOT + JIT + profile-guided |
| Tốc độ chạy | Chậm hơn | Nhanh hơn (đã compile sẵn) |
| GC | Pause dài | Concurrent, pause ngắn |
| Tương thích | — | App viết cho Dalvik chạy được trên ART |

## Tính năng chính

### AOT compilation
- **`dex2oat`** compile app **lúc cài đặt** → binary tối ưu cho đúng thiết bị.
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
