---
title: JIT Compiler trong ART
source: https://source.android.com/docs/core/runtime/jit-compiler?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - ART
  - JIT
  - Profile-guided
---

# JIT Compiler trong ART

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/runtime/jit-compiler](https://source.android.com/docs/core/runtime/jit-compiler?hl=vi)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../../android-automotive-developer-roadmap.md). Đọc sau [android-art.md](android-art.md) và [art-configure.md](art-configure.md).

## JIT là gì? Khác AOT chỗ nào?

**JIT (Just-In-Time) compiler biến bytecode thành machine code *ngay trong lúc app đang chạy*** — cụ thể là chỉ compile những đoạn code được gọi nhiều ("hot"), còn phần ít dùng cứ để interpreter chạy. Đối lập với **AOT (Ahead-Of-Time)** compile *trước* khi chạy (lúc cài/lúc build): AOT như dịch sẵn cả cuốn sách trước khi đọc, JIT như dịch tại chỗ đúng những câu bạn thực sự đọc tới.

Điểm mấu chốt cho người mới: **JIT không thay thế AOT, nó bổ sung**. Ý tưởng là app **nhanh dần lên khi dùng** — lần đầu chạy thì interpreter + JIT lo, ART âm thầm ghi lại "method nào nóng" vào profile, rồi sau đó AOT lại đúng những method đó (profile-guided). Nhờ vậy không cần compile sẵn *toàn bộ* app (tốn chỗ) mà vẫn nhanh ở chỗ cần nhanh.

Vì thế JIT **chỉ bật cho app không compile bằng filter `speed`** — nếu đã AOT hết mọi method rồi thì chẳng còn gì cho JIT làm.

## Kiến trúc — 3 thành phần

| Thành phần | Vai trò |
|-----------|---------|
| **JIT code cache** | Lưu machine code compile tức thời (in-memory) |
| **Profiler** | Thu thập data runtime (method nóng) → ghi profile file |
| **dex2oat daemon** | AOT lại app **dựa trên profile** (chạy nền lúc idle/sạc) |

## Vòng đời tối ưu (profile-guided)

```
App chạy lần đầu
   │ interpreter chạy DEX
   ▼
Method "nóng" (vượt jitthreshold)
   │ JIT compile → code cache
   ▼
Profiler ghi method nóng → profile file
   │ (máy idle + đang sạc)
   ▼
dex2oat AOT theo profile (speed-profile)
   ▼
Lần chạy sau: code đã AOT sẵn, khởi động nhanh hơn
```

- Profile lưu tại: `/data/misc/profiles/cur/<user>/<package>/primary.prof`

## 3 trạng thái của method

1. **Interpreted** — chạy DEX qua interpreter
2. **JIT-compiled** — trong code cache
3. **AOT-compiled** — trong file `.oat`

⚠️ Cả JIT lẫn AOT cùng tồn tại → **JIT được ưu tiên** (code mới hơn, tối ưu theo runtime data thật).

## Bộ nhớ

- Code cache + metadata ổn định quanh **~4 MB** với app lớn.
- Tune qua `dalvik.vm.jitinitialsize` / `jitmaxsize` (xem [art-configure.md](art-configure.md)).

## Kiểm tra & thao tác

```bash
# Xem trạng thái compile của app
adb shell dumpsys package dexopt | grep -A2 <package>

# Ép compile theo profile ngay (không chờ idle)
adb shell cmd package compile -m speed-profile -f <package>

# Xóa profile + code đã compile (test lại từ đầu)
adb shell cmd package compile --reset <package>

# Tắt JIT (test)
adb shell setprop dalvik.vm.usejit false
```

🚗 *Liên hệ Automotive:* xe ít khi "idle + đang sạc" kiểu phone — background dexopt có thể không bao giờ chạy. OEM xử lý: ship sẵn **cloud profile / baseline profile** trong image (dexpreopt `speed-profile` với profile thu sẵn), hoặc trigger compile theo điều kiện riêng (xe đỗ, tắt màn). App AAOS quan trọng nên kèm baseline profile để khỏi phụ thuộc JIT warm-up.

## Câu hỏi tự kiểm tra

- [ ] JIT bật cho app compile bằng filter nào? Vì sao `speed` thì không?
- [ ] 3 thành phần kiến trúc JIT?
- [ ] Vòng đời: từ interpreter đến AOT theo profile diễn ra thế nào, khi nào dex2oat chạy?
- [ ] JIT code và AOT code cùng có — cái nào được ưu tiên, vì sao?
- [ ] Profile file nằm đâu? Lệnh nào ép compile theo profile ngay?

## 📚 Tài liệu

- 📖 [Baseline Profiles](https://developer.android.com/topic/performance/baselineprofiles/overview)

## Đọc tiếp

- [android-art.md](android-art.md) — ART tổng quan
- [art-configure.md](art-configure.md) — các knob tune
- [art-service.md](art-service.md) — dexopt từ Android 14
