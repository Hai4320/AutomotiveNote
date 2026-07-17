# Phase 3 — Embedded Linux & AOSP Build: Tổng kết

> Phase 3 trong [roadmap](../android-automotive-developer-roadmap.md). Mục tiêu: lấp lỗ hổng lớn nhất của app dev — làm việc với **Linux thật** và **build AOSP thật**. **Học xong khi tự sync source, `lunch` target xe, `m` ra image, chạy AAOS emulator từ image tự build và sửa được một dòng trong SystemUI.**

## Thứ tự đọc

> 📖 [glossary.md](../glossary.md) — bảng thuật ngữ dùng chung mọi phase (repo, Soong, lunch, variant, partition, Treble, AVB, fastboot...). Mở song song, gặp từ lạ thì tra.

### 1. Linux nền tảng
1. [linux-fundamentals.md](linux-fundamentals.md) — filesystem/mount, permission `rwx` (DAC), process/PID/fork, signal; hai hàng rào permission của HAL
2. [shell-scripting.md](shell-scripting.md) — bash (quoting, `$?`, `set -euo pipefail`), pipe/redirect, grep/sed/awk — công cụ lọc log & viết script build
3. [kernel-concepts.md](kernel-concepts.md) — driver, kernel vs user space, device tree (`.dts`), driver phơi ra `/dev` `/sys` `/proc`

### 2. AOSP build system
4. [repo-tool.md](repo-tool.md) — `repo` + manifest quản ~1000 git repo, `repo init/sync`, upload lên Gerrit
5. [soong-make.md](soong-make.md) — `Android.bp` (Soong) vs `Android.mk` (Make), `envsetup`/`lunch`/variant, `m`/`mm`/`mmm`
6. [build-aaos-image.md](build-aaos-image.md) — build `sdk_car_x86_64`, chạy emulator từ source, vòng lặp sửa–`mmm`–sync ⭐ milestone phase
7. [partitions-flashing.md](partitions-flashing.md) — `m` ra bộ `.img`, partition nào chứa gì, `fastboot flash` lên board thật

### Đọc thêm (chưa có note riêng / đã có ở phase khác)
- **SELinux** — roadmap xếp ở Phase 3 nhưng đã có note đầy đủ: [selinux.md](../1_android_internal/selinux.md) (permissive/enforcing, `.te`, trị `avc: denied`). Đọc lại ở đây, là hàng rào MAC nối tiếp permission DAC ở note 1.
- **Kernel Android chi tiết** — ACK/GKI/KMI/vendor module đã có ở [kernel/](../1_android_internal/kernel/) (Phase 1).
- **Serial console** — debug khi board chưa boot nổi ADB; để dành [Phase 6](../android-automotive-developer-roadmap.md#phase-6-debugging--profiling-tools).

## 10 ý phải nhớ khi rời Phase 3

1. Android **là** Linux: filesystem một cây từ `/`, mọi thứ là file, `/system` `/vendor` mount **read-only**.
2. **Hai hàng rào permission**: Unix `rwx` (DAC) **và** SELinux (MAC). HAL fail có thể ở tầng nào — check cả `ls -l` lẫn `dmesg | grep avc`.
3. `SIGKILL`(9)/`SIGSTOP` không bắt được; `SIGTERM`(15) xin dừng lịch sự để flush trạng thái — quan trọng cho daemon xe chạy suốt.
4. grep = tìm dòng, sed = sửa text, awk = trích cột. `set -euo pipefail` + `$?` để script fail đúng, không giả pass.
5. Driver bug = **kernel panic** (chết cả xe); HAL crash chỉ chết một process → đẩy logic lên userspace HAL.
6. **Device tree** (`.dts`) mô tả phần cứng tĩnh cho kernel; `compatible` chọn driver. Khác "device tree" nghĩa AOSP.
7. AOSP = ~1000 git repo; **`repo` + manifest** ghim đúng version từng repo (build tái lập được). Review qua **Gerrit**, không phải GitHub PR.
8. `Android.bp` (Soong) khai báo module, không script; **`vendor: true`** đẩy HAL vào `/vendor` (Treble). `Android.mk` là hệ cũ.
9. `lunch <target>-<variant>` chọn thiết bị + variant (`user`/`userdebug`/`eng`); `adb root`/`setenforce 0` chỉ chạy userdebug/eng. `m` cả cây (giờ), `mmm` một module (phút) — dùng `mmm` cho vòng sửa–thử.
10. `m` ra nhiều `.img` = từng partition; **flash** = ghi image vào partition. `fastboot` (bootloader) khác `adb` (OS đã boot); image sai chữ ký AVB = không boot.

## Tự kiểm tra cuối phase

- [ ] Vẽ lại từ trí nhớ: từ `repo sync` → `lunch` → `m` → `.img` → flash/emulator, chỉ ra file `.img` nào là partition nào.
- [ ] Giải thích một HAL "chết lặng" có thể do những tầng permission nào, debug mỗi tầng bằng lệnh gì.
- [ ] Trả lời hết câu hỏi tự kiểm tra trong 7 note trên (+ glossary).

## 🎯 Thực hành khép phase (từ roadmap)

Sync AOSP → `lunch sdk_car_x86_64-userdebug` → `m` → chạy AAOS emulator từ image tự build → **sửa một dòng trong Car SystemUI, `mmm` module đó, đẩy lên, thấy thay đổi**:

```bash
source build/envsetup.sh
lunch sdk_car_x86_64-userdebug
m -j$(nproc)                                   # build đầu ~1–3 giờ
emulator -writable-system &
# sửa packages/apps/Car/SystemUI/...
mmm packages/apps/Car/SystemUI
adb root && adb remount && adb sync && adb reboot
```

## Tiếp theo → [Phase 4: Native & HAL Development](../android-automotive-developer-roadmap.md#phase-4-native--hal-development)

Phase 3 build được nền — Phase 4 dùng nền đó **viết HAL thật**: `Android.bp` cho service, AIDL interface xe, sepolicy, expose vehicle property mới lên `CarPropertyManager`.
