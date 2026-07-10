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

## Tổng quan

- JIT **bổ sung cho AOT**, không thay thế — kết hợp với code profiling để app **nhanh dần lên khi dùng**.
- Chỉ bật cho app **không** compile bằng filter `speed` (đã AOT hết thì JIT không cần).

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
