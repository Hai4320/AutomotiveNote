---
title: ART Service (Android 14+)
source: https://source.android.com/docs/core/runtime/configure/art-service
phase: 1 — Android Internals
tags:
  - AOSP
  - ART
  - dexopt
  - ART Service
---

# ART Service (Android 14+)

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/runtime/configure/art-service](https://source.android.com/docs/core/runtime/configure/art-service)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../../android-automotive-developer-roadmap.md). Đọc sau [art-jit.md](art-jit.md).

## ART Service là gì?

- Từ **Android 14**: mọi **on-device AOT compilation (dexopt)** do ART Service đảm nhiệm.
- **Thay thế** cơ chế dexopt cũ nằm trong Package Manager (`BackgroundDexOptService`).
- Là một phần của **ART Mainline module** → update qua Play, có API + system property để customize.

## Compilation reasons

Mỗi lần dexopt có 1 "reason" — filter mặc định cấu hình theo reason:

| Reason | Khi nào |
|--------|---------|
| `first-boot` | Boot lần đầu sau factory reset |
| `boot-after-ota` | Boot đầu sau OTA update |
| `boot-after-mainline-update` | Boot sau update Mainline module |
| `bg-dexopt` | Background dexopt job (idle + sạc) |
| `inactive` | App lâu không dùng → downgrade |
| `cmdline` | Chạy tay qua `adb shell cmd package compile` |

## System properties (`pm.dexopt.*`)

| Property | Ý nghĩa |
|----------|---------|
| `pm.dexopt.<reason>` | Filter mặc định cho từng reason (vd `pm.dexopt.bg-dexopt=speed-profile`) |
| `pm.dexopt.shared` | Fallback cho app bị app khác dùng chung, không có profile (default `speed`) |
| `pm.dexopt.<reason>.concurrency` | Số dex2oat chạy song song |
| `pm.dexopt.downgrade_after_inactive_days` | App không dùng N ngày → hạ filter, tiết kiệm storage |
| `pm.dexopt.disable_bg_dexopt` | Tắt background job (test) |

## Background dexopt job

- Chạy khi **idle + đang sạc**, schedule qua **JobScheduler, job ID `27873780`**.
- Nguyên tắc: ART Service ưu tiên **`speed-profile`** cho mọi app khi có profile — thường trong bg-dexopt.
- Debug: `JobScheduler.getPendingJobReason()` xem vì sao job chưa chạy.

## Java API cho OEM (`ArtManagerLocal`)

Singleton trong system server — điểm móc customize:

| API | Dùng để |
|-----|---------|
| `dexoptPackage()` | Trigger dexopt 1 app với param tùy chỉnh |
| `setBatchDexoptStartCallback()` | Sửa danh sách package + param cho boot/bg dexopt |
| `addDexoptDoneCallback()` | Nghe sự kiện dexopt xong |
| `setAdjustCompilerFilterCallback()` | Override filter per-package (Android 15+) |
| `setScheduleBackgroundDexoptJobCallback()` | Đổi điều kiện schedule bg job |

🚗 *Liên hệ Automotive:* đây chính là chỗ giải bài toán "xe không có idle + sạc" (xem [art-jit.md](art-jit.md)) — OEM dùng `setScheduleBackgroundDexoptJobCallback()` đổi điều kiện chạy job (vd: xe đỗ, tắt màn, điện ắc quy đủ) và `setBatchDexoptStartCallback()` ưu tiên app cockpit. Không cần fork ART, chỉ cần hook API trong car service riêng.

## Câu hỏi tự kiểm tra

- [ ] ART Service thay thế cơ chế nào, từ Android bao nhiêu?
- [ ] Kể 4 compilation reason và ý nghĩa?
- [ ] `pm.dexopt.shared` dùng khi nào, default gì?
- [ ] Background dexopt job chạy điều kiện nào, job ID bao nhiêu?
- [ ] OEM muốn đổi điều kiện chạy bg dexopt thì dùng API nào?

## 📚 Tài liệu

- 📖 [ART Service docs](https://source.android.com/docs/core/runtime/configure/art-service)
- 📖 [Background dexopt](https://source.android.com/docs/core/runtime/configure#how_art_service_works)

## Đọc tiếp

- [art-configure.md](art-configure.md) — filter & dexpreopt
- [art-jit.md](art-jit.md) — profile-guided flow
