---
title: Bảng thuật ngữ (mọi phase)
phase: chung — dùng cho toàn roadmap
tags:
  - Glossary
  - AOSP
  - Treble
  - Automotive
---

# Bảng thuật ngữ — dùng chung mọi phase

> Thuật ngữ dùng xuyên suốt [roadmap](android-automotive-developer-roadmap.md), không riêng phase nào. Không cần đọc một mạch — **mở song song khi đọc note bất kỳ phase**, gặp thuật ngữ lạ thì tra ở đây. Phase mới thêm thuật ngữ mới thì bổ sung vào file này.

## 2 khái niệm nền (đọc kỹ — nửa sau Phase 1 và cả Phase 4/7 xây trên chúng)

### Project Treble (Android 8+)

Kiến trúc tách **vendor** (code phần cứng — hãng chip/hãng xe viết) khỏi **framework** (code Android — Google viết) qua các interface ổn định. Trước Treble: mỗi lần update Android, vendor phải port lại toàn bộ code → update chậm hoặc không bao giờ. Sau Treble: 2 bên update độc lập, miễn giữ đúng interface đã cam kết.

Nhiều thứ trong Phase 1 là hệ quả trực tiếp của Treble:

- HAL bắt buộc nằm ở vendor partition — [hal/android-hal.md](1_android_internal/hal/android-hal.md)
- VINTF match giữa "vendor có gì" và "framework cần gì" — [hal/vintf-objects.md](1_android_internal/hal/vintf-objects.md)
- Binder tách 3 context (binder / hwbinder / vndbinder) — [binder-ipc.md](1_android_internal/binder-ipc.md)
- SELinux policy tách đôi platform / vendor — [selinux.md](1_android_internal/selinux.md)
- GKI tách kernel core / vendor module — [kernel/android-gki.md](1_android_internal/kernel/android-gki.md)

### Partition — Android chia bộ nhớ lưu trữ thế nào

Ổ lưu trữ của thiết bị Android chia thành nhiều **partition** (phân vùng), mỗi cái có chủ sở hữu và vòng đời update riêng:

| Partition | Chứa | Ai sở hữu |
|-----------|------|-----------|
| `system` | Framework, system app, ART | Google/AOSP |
| `vendor` | HAL services, driver userspace, sepolicy vendor | SoC vendor |
| `odm` | Tùy biến theo từng thiết bị (đè được vendor) | ODM/OEM |
| `product` | App/config theo dòng sản phẩm | OEM |
| `boot` | Kernel (GKI) + ramdisk | Google |
| `vendor_boot`, `vendor_dlkm` | Vendor kernel modules | SoC vendor |
| `userdata` | Data người dùng, app cài thêm | — |

- Ranh giới Treble là ranh giới **vật lý**: update Android = thay `system`, không đụng `vendor` — vì thế mới có quy tắc "HAL phải nằm ở vendor partition".
- **Image** = file chứa nội dung 1 partition (`system.img`, `boot.img`...), sinh ra khi build AOSP, flash vào máy bằng fastboot. "Build image" trong các note = build ra bộ file này.

## Vai trò các bên

| Thuật ngữ | Nghĩa |
|-----------|-------|
| **OEM** | Original Equipment Manufacturer — hãng bán sản phẩm cuối. Bên phone: Samsung; bên xe: Volvo, GM |
| **ODM** | Original Design Manufacturer — hãng thiết kế/gia công thiết bị cho OEM |
| **SoC vendor** | Hãng chip (Qualcomm, NXP, Renesas) — viết driver + vendor modules cho chip của mình |
| **Tier-1** | Nhà cung cấp cấp 1 trong ngành xe (Bosch, LG, Harman) — làm hộp IVI/phần mềm cho OEM |
| **Vendor** (trong docs AOSP) | Chỉ chung "phía làm phần cứng" — thường là SoC vendor + ODM, đối lập với "framework" của Google |

## Build & phát triển AOSP

| Thuật ngữ | Nghĩa |
|-----------|-------|
| **repo** | Tool của Google quản lý ~1000 git repo con của AOSP như 1 cây source |
| **Soong / `Android.bp`** | Hệ build của AOSP; `Android.bp` là file khai báo module (như `build.gradle` của app) |
| **lunch** | Lệnh chọn target build (thiết bị nào + variant nào), vd `lunch sdk_car_x86_64-userdebug` |
| **Build variant** | `user` = bản ship (khóa chặt) / `userdebug` = như user + root & debug được / `eng` = bản dev, bật hết. Lệnh như `setenforce 0` chỉ chạy trên userdebug/eng |
| **Device tree** (nghĩa AOSP) | Thư mục `device/<vendor>/<board>/` chứa config build cho 1 thiết bị (`device.mk`, `BoardConfig.mk`). ⚠️ Đừng nhầm với "device tree" của Linux kernel (file `.dts` mô tả phần cứng) |
| **Upstream** | Repo gốc phía trên (Linux = kernel.org). "Upstream-first" = gửi code lên gốc trước, rồi mới đem về nhánh riêng |
| **Backport** | Đem fix/feature từ nhánh mới về nhánh cũ đang duy trì |
| **Gerrit** | Hệ code review của AOSP ([android-review.googlesource.com](https://android-review.googlesource.com)) |
| **Bring-up** | Quá trình đưa Android boot lần đầu trên board mới: flash kernel, ghép driver, trị lỗi từng tầng |
| **Ramdisk** | Filesystem nhỏ đóng trong boot image, nạp thẳng vào RAM để chạy giai đoạn boot sớm (chứa `/init` first stage) |
| **OTA** | Over-The-Air — update hệ điều hành gửi qua mạng xuống thiết bị |
| **Daemon** | Process chạy nền, không có UI (`logd`, `ueventd`...) |

## Kiểm định & tương thích

| Thuật ngữ | Nghĩa |
|-----------|-------|
| **CDD** | Compatibility Definition Document — tài liệu định nghĩa "thế nào là thiết bị Android hợp lệ" |
| **CTS** | Compatibility Test Suite — bộ test tự động verify thiết bị đạt CDD (chạy ở tầng app/API) |
| **VTS** | Vendor Test Suite — bộ test verify tầng vendor: HAL, kernel, VINTF khai đúng |
| **VSR** | Vendor Software Requirements — yêu cầu bổ sung cho vendor muốn ship GMS |
| **GMS** | Google Mobile Services — gói app/service độc quyền Google (Play, Maps...) OEM phải license; bên xe gói tương đương là **GAS** ([aaos-structure.md](1_android_internal/aaos-structure.md)) |

## Bảo mật & boot

| Thuật ngữ | Nghĩa |
|-----------|-------|
| **AVB** | Android Verified Boot — bootloader verify chữ ký boot image **trước khi boot**; sửa image = không boot |
| **dm-verity** | Cơ chế kernel verify tính toàn vẹn partition **trong lúc chạy** (mỗi lần đọc block) — AVB bảo vệ lúc boot, dm-verity bảo vệ runtime |
| **fastboot** | Chế độ + tool flash partition qua USB (vào bằng giữ nút lúc boot), vd `fastboot flash boot boot.img` |

## Binder & ART — mấy cái tên dễ lẫn

| Thuật ngữ | Nghĩa |
|-----------|-------|
| **Marshal / unmarshal** | = serialize / deserialize — đóng gói tham số thành bytes (Parcel) để gửi qua process khác, và mở ngược lại |
| **"Domain" 2 nghĩa** | Trong [binder-ipc.md](1_android_internal/binder-ipc.md): domain = biến thể binder driver (binder/hwbinder/vndbinder). Trong [selinux.md](1_android_internal/selinux.md): domain = type của process. Cùng từ, 2 khái niệm không liên quan |
| **dex2oat** | Tool compile DEX → machine code (binary `dex2oat` chạy trên máy) |
| **dexopt** | *Việc* compile DEX trên thiết bị (do dex2oat thực hiện) — lúc cài, lúc idle, sau OTA... |
| **dexpreopt** | Compile sẵn **lúc build image** trên máy build — app hệ thống lên xe đã có sẵn code compile |
| **Interpreter** | Chạy bytecode bằng cách dịch từng lệnh lúc runtime, không compile trước — chậm nhưng khởi động ngay |

## Thuật ngữ ô tô

| Thuật ngữ | Nghĩa |
|-----------|-------|
| **ECU** | Electronic Control Unit — máy tính con điều khiển 1 chức năng (động cơ, phanh, cửa...). Xe hiện đại có hàng chục ECU |
| **CAN bus** | Controller Area Network — mạng nội bộ trong xe nối các ECU với nhau (ngoài ra còn **LIN** rẻ hơn/chậm hơn, **Automotive Ethernet** nhanh hơn) |
| **IVI** | In-Vehicle Infotainment — hệ thống giải trí/thông tin trung tâm (màn giữa xe) — chỗ AAOS chạy |
| **Cluster / instrument cluster** | Màn đồng hồ sau vô lăng: tốc độ, vòng tua, cảnh báo |
| **HUD** | Head-Up Display — chiếu thông tin lên kính lái |
| **HMI** | Human-Machine Interface — gọi chung toàn bộ giao diện người lái nhìn/chạm (IVI + cluster + HUD) |
| **HVAC** | Heating, Ventilation, Air Conditioning — hệ điều hòa |
| **ADAS** | Advanced Driver Assistance Systems — hỗ trợ lái: giữ làn, cảnh báo va chạm, cruise control thích ứng |
| **EVS** | Extended View System — stack camera native của AAOS (camera lùi, surround view), đi đường riêng không chờ Java stack ([boot-process.md](1_android_internal/boot-process.md)) |
| **VIN** | Vehicle Identification Number — số khung, định danh duy nhất của xe |
| **SDV** | Software-Defined Vehicle — xe mà tính năng chủ yếu do phần mềm quyết định, update qua OTA |
| **Garage mode** | Cửa sổ bảo trì khi xe đỗ + tắt máy: hệ thống thức dậy ngầm chạy update, dexopt... rồi ngủ lại |
| **Audio ducking** | Tự hạ volume nguồn âm đang phát khi có âm ưu tiên hơn (nhạc nhỏ lại khi navigation nói) |
| **Rotary controller** | Núm xoay điều khiển UI (BMW iDrive...) — input chính khi không dùng cảm ứng |

## Câu hỏi tự kiểm tra

- [ ] Treble tách gì khỏi gì? Kể 3 hệ quả của Treble gặp trong Phase 1?
- [ ] `system` vs `vendor` partition — ai sở hữu, vì sao HAL phải nằm ở vendor?
- [ ] OEM vs ODM vs SoC vendor vs Tier-1 — trong dự án xe ai làm gì?
- [ ] dex2oat / dexopt / dexpreopt khác nhau thế nào?
- [ ] AVB vs dm-verity — cái nào bảo vệ lúc boot, cái nào lúc runtime?

## 📚 Tài liệu

- 📖 [Android architecture (Treble/HIDL overview)](https://source.android.com/docs/core/architecture)
- 📖 [Partitions overview](https://source.android.com/docs/core/architecture/partitions)
- 📖 [CTS](https://source.android.com/docs/compatibility/cts) / [VTS](https://source.android.com/docs/core/tests/vts)
- 📖 [Verified Boot (AVB + dm-verity)](https://source.android.com/docs/security/features/verifiedboot)

## Đọc tiếp

- [android-architecture.md](1_android_internal/android-architecture.md) — note mở đầu Phase 1, dùng ngay CDD/CTS/VTS
- [hal/vintf-objects.md](1_android_internal/hal/vintf-objects.md) — Treble hiện thân cụ thể nhất
