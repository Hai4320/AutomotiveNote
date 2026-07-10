---
title: SELinux trên Android
phase: 1 — Android Internals (trả nợ sớm cho Phase 4)
tags:
  - AOSP
  - SELinux
  - sepolicy
  - Security
---

# SELinux trên Android

> Thuộc **Phase 1 — Android Internals** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc sau [boot-process.md](boot-process.md). **Phase 4 viết HAL sẽ đập mặt vào `avc: denied` — học trước ở đây.**

## SELinux là gì?

- **MAC (Mandatory Access Control)** — khác DAC (rwx truyền thống): quyền do **policy hệ thống** quyết, root cũng không vượt được.
- Nguyên tắc Android: **deny by default** — không có rule allow = cấm.
- 2 mode: **permissive** (chỉ log vi phạm) / **enforcing** (chặn thật). Production luôn enforcing.

```bash
adb shell getenforce              # Enforcing / Permissive
adb shell setenforce 0            # tạm permissive (userdebug, để test)
```

## Khái niệm lõi: label & type enforcement

Mọi thứ (process, file, socket, property) đều có **security context**:

```
u:r:hal_vehicle_default:s0
│  │  │                  └─ level (MLS, ít dùng)
│  │  └─ type/domain  ← QUAN TRỌNG NHẤT
│  └─ role
└─ user
```

- Process → gọi là **domain**; file/resource → **type**.
- Policy = tập rule `allow <domain> <type>:<class> {permissions};`

```
# Cho VHAL đọc/ghi node CAN
allow hal_vehicle_default can_device:chr_file { read write open };
```

## File policy nằm đâu

| File | Vai trò |
|------|---------|
| `*.te` | Rule chính (type enforcement) — mỗi domain 1 file |
| `file_contexts` | Gán label cho file theo path |
| `service_contexts` / `vndservice_contexts` | Label cho Binder service ([binder-ipc.md](binder-ipc.md)) |
| `property_contexts` | Label cho system property |

**Tách theo Treble** (như VINTF): platform policy (`system/sepolicy/`) Google giữ — vendor policy (`device/<vendor>/.../sepolicy/`) OEM viết. Vendor **không sửa** platform policy.

## Workflow trị `avc: denied` (Phase 4 dùng hằng ngày)

```
1. Chạy service → fail khó hiểu
2. adb shell dmesg | grep avc      (hoặc logcat -b events)
   avc: denied { open } for path="/dev/can0"
   scontext=u:r:hal_vehicle_default:s0
   tcontext=u:object_r:device:s0 tclass=chr_file
3. Đọc: domain hal_vehicle_default bị cấm open file type device
4. audit2allow -i denial.txt   → sinh rule gợi ý
5. KHÔNG copy mù: xem lại — thiếu label riêng? (thêm file_contexts
   cho /dev/can0 thay vì allow cả type device chung chung)
6. Thêm rule vào .te trong vendor sepolicy → rebuild → thử lại
```

⚠️ Bẫy: `audit2allow` luôn cho rule "chạy được" nhưng thường **quá rộng** — reviewer/VTS (`neverallow` rules) sẽ chặn. Label đúng > allow rộng.

## Macro hay gặp

```
init_daemon_domain(hal_vehicle_default)   # init start service này vào domain riêng
hal_server_domain(my_hal, hal_vehicle)    # đánh dấu là HAL server
binder_call(carservice_app, hal_vehicle_default)
```

## 🚗 Liên hệ Automotive

Viết 1 HAL mới (Phase 4) = bộ sepolicy tối thiểu:
1. `.te` mới khai domain cho service
2. `file_contexts` — label binary `/vendor/bin/hw/...`
3. `vndservice_contexts` / VINTF nếu đăng ký Binder service
4. `binder_call` giữa CarService ↔ HAL domain
5. Quyền đụng device node (CAN, tty...)

Thiếu bất kỳ mảnh nào → service chết lặng hoặc client không tìm thấy — **check `dmesg | grep avc` TRƯỚC khi nghi code sai**.

## Câu hỏi tự kiểm tra

- [ ] MAC khác DAC gì? Root có vượt được SELinux không?
- [ ] Đọc context `u:r:hal_vehicle_default:s0` — phần nào quan trọng nhất?
- [ ] Policy load ở bước nào của boot? ([boot-process.md](boot-process.md))
- [ ] `.te` vs `file_contexts` vs `service_contexts` — mỗi file làm gì?
- [ ] Vì sao không copy nguyên output `audit2allow`?
- [ ] Platform vs vendor sepolicy — ai được sửa gì?

## 📚 Tài liệu

- 📖 [SELinux for Android](https://source.android.com/docs/security/features/selinux) — docs chính thức
- 📖 [Writing SELinux policy](https://source.android.com/docs/security/features/selinux/device-policy)
- 📖 [Validate SELinux](https://source.android.com/docs/security/features/selinux/validate) — trị denial chính thức
- 📖 [system/sepolicy (source)](https://cs.android.com/android/platform/superproject/main/+/main:system/sepolicy/)

## Đọc tiếp

- [boot-process.md](boot-process.md) — SELinux load ở init second stage
- [binder-ipc.md](binder-ipc.md) — vndservice_contexts
- [hal/android-hal.md](hal/android-hal.md) — HAL cần domain riêng
