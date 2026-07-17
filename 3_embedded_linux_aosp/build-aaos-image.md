---
title: Build & chạy AAOS emulator image từ source
source: https://source.android.com/docs/automotive/start/avd/android_virtual_device
phase: 3 — Embedded Linux & AOSP Build
tags:
  - AOSP
  - AAOS
  - Emulator
  - Build
---

# Build & chạy AAOS emulator image từ source

> Thuộc **Phase 3 — Embedded Linux & AOSP Build** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc **sau** [repo-tool.md](repo-tool.md) (đã sync source) và [soong-make.md](soong-make.md) (biết lunch/m). Đây là **milestone của phase**: tự build một AAOS chạy được và sửa được. Nối với [aaos-structure.md](../1_android_internal/aaos-structure.md) (AAOS = AOSP + car layers).
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## Mục tiêu note này

Ghép ba bước trước thành một milestone chạy được: **sync → lunch target xe → build → chạy emulator từ image tự build → sửa một dòng → rebuild → thấy thay đổi**. Làm xong vòng này là qua được cửa ải lớn nhất của app dev chuyển sang platform: "tôi build được AOSP thật".

## Vì sao build từ source thay vì tải image sẵn?

Android Studio có sẵn AAOS emulator image tải về — dùng để **viết app** cho xe thì đủ. Nhưng platform engineer cần image **build từ source** vì:

- Sửa được **framework, SystemUI, CarService, sepolicy, VHAL** — thứ image đóng gói sẵn không cho đụng.
- Đây là chính xác quy trình OEM/Tier-1 làm với board xe thật, chỉ khác target.
- Debug system-level cần symbol + source khớp image đang chạy.

## Quy trình đầy đủ

```bash
# 0. Đã: repo sync xong (repo-tool.md), máy ≥250GB đĩa + RAM khá
cd aosp

# 1. Nạp hàm build + chọn target AAOS emulator
source build/envsetup.sh
lunch sdk_car_x86_64-userdebug     # target xe, variant userdebug (root/debug được)

# 2. Build toàn bộ image (lần đầu ~1–3 giờ tùy máy)
m -j$(nproc)

# 3. Chạy emulator từ image vừa build
emulator                            # tự lấy image trong out/ của target đã lunch
```

- `sdk_car_x86_64` = target emulator AAOS (car), `x86_64` chạy nhanh trên máy PC (không emulate ARM).
- `emulator` sau khi `lunch` + `m` tự trỏ vào `$OUT` (thư mục output của target hiện tại).
- Cờ hay dùng: `emulator -writable-system` (cho `adb remount` sửa `/system`), `-no-snapshot`, `-verbose`.

## Vòng lặp sửa–thử (quan trọng nhất)

Đừng `m` cả cây cho mỗi thay đổi nhỏ. Sửa một module → build **riêng module đó** → đẩy lên emulator đang chạy:

```bash
# Ví dụ: sửa 1 dòng trong Car SystemUI
# 1. sửa file trong packages/apps/Car/SystemUI/...
mmm packages/apps/Car/SystemUI      # build riêng module (phút, không phải giờ)

# 2. đẩy lên emulator đang chạy
adb root && adb remount             # cần -writable-system khi khởi động emulator
adb sync                            # đồng bộ artifact mới vào /system, /vendor...
adb reboot                          # hoặc restart riêng component nếu được
```

| Cần | Làm |
|-----|-----|
| Sửa & thử một module app/service | `mmm <path>` → `adb sync` → reboot |
| Sửa nhiều nơi / đổi cấu hình image | `m` lại toàn bộ → chạy `emulator` mới |
| Build lỗi lạ sau khi sửa | `m installclean` (dọn nhẹ) → build lại |

⚠️ `adb sync`/`remount` cần variant **userdebug/eng** + emulator chạy `-writable-system`. `user` build khóa chặt, không sửa được kiểu này.

## Bài tập milestone (từ roadmap)

1. `repo sync` source AOSP.
2. `lunch sdk_car_x86_64-userdebug`, `m` → build image.
3. Chạy AAOS emulator từ image tự build.
4. **Sửa một dòng hiển thị trong Car SystemUI** (vd đổi text/màu status bar), `mmm` module đó, đẩy lên, thấy thay đổi trên emulator.

Làm được bước 4 = nắm trọn vòng build–sửa–verify của platform dev.

## 🚗 Liên hệ Automotive

- **Emulator giờ, board sau**: `sdk_car_x86_64` để học miễn phí. Dự án thật đổi thành `lunch <oem_board>-userdebug` rồi flash `.img` lên bench board bằng fastboot (xem [partitions-flashing.md](partitions-flashing.md)) — **cùng quy trình, khác target**.
- **CarService/VHAL cũng chỉ là module**: muốn thử một vehicle property mới (Phase 4/7) → sửa VHAL default, `mmm` module VHAL, sync, thấy property mới qua `dumpsys car_service`. Chính vòng lặp này.
- **Chu kỳ dài là bình thường** (Phase 8): build đầu tính bằng giờ; đây là lý do máy build mạnh + `mmm` đúng module là kỹ năng năng suất cốt lõi, không phải mẹo vặt.
- **dexpreopt trên xe**: image car build sẵn compile AOT toàn bộ system app ([art-configure.md](../1_android_internal/art/art-configure.md)) → build lâu hơn nhưng boot/chạy nhanh, hợp xe (không có idle+sạc để compile sau).

## Câu hỏi tự kiểm tra

- [ ] Vì sao platform engineer build image từ source thay vì dùng image AS tải sẵn?
- [ ] Ba lệnh cốt lõi để ra emulator AAOS chạy được là gì?
- [ ] `sdk_car_x86_64` nghĩa là gì? Vì sao chọn `x86_64` chứ không ARM để học?
- [ ] Sửa một dòng SystemUI: vì sao dùng `mmm` chứ không `m`? Đẩy lên emulator bằng gì?
- [ ] `adb sync`/`remount` cần điều kiện gì (variant + cờ emulator)?
- [ ] Chuyển từ emulator sang board xe thật đổi cái gì trong quy trình?
- [ ] Vì sao image car build lâu hơn (gợi ý: dexpreopt)?

## 📚 Tài liệu

- 📖 [AAOS on Android Virtual Device](https://source.android.com/docs/automotive/start/avd/android_virtual_device) — build & chạy AAOS emulator
- 📖 [Build Android](https://source.android.com/docs/setup/build) — lunch/m chi tiết
- 📖 [Run builds (emulator)](https://source.android.com/docs/setup/test/run) — chạy image tự build
- 📖 [aaos-structure.md](../1_android_internal/aaos-structure.md) — AAOS layer & build target (Phase 1)

## Đọc tiếp

- [partitions-flashing.md](partitions-flashing.md) — `m` ra `.img`; chương này flash lên board thật.
- [soong-make.md](soong-make.md) — chi tiết lunch/m/mmm dùng ở đây.
- [aaos-structure.md](../1_android_internal/aaos-structure.md) — thứ bạn vừa build gồm những layer nào.
- [hal/android-vhal.md](../1_android_internal/hal/android-vhal.md) — thử property mới trên image này (Phase 4).
