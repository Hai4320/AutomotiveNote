# Phase 1 — Android Internals: Tổng kết

> Phase 1 trong [roadmap](../android-automotive-developer-roadmap.md). Mục tiêu: hiểu Android hoạt động *bên dưới* — bước nhảy lớn nhất từ app dev sang system dev. **Phần này đã học xong khi trả lời được hết câu hỏi tự kiểm tra trong từng note.**

## Thứ tự đọc

### 1. Bức tranh lớn
1. [android-architecture.md](android-architecture.md) — 6 layer AOSP: Apps → Framework → ART → HAL → Native → Kernel
2. [java-native-relationship.md](java-native-relationship.md) — 2 cầu nối JNI vs Binder
3. [aaos-structure.md](aaos-structure.md) — AAOS = AOSP + car layers; build target

### 2. Giao tiếp & services
4. [binder-ipc.md](binder-ipc.md) — Binder driver, Proxy/Stub, thread pool, 3 domain
5. [system-services.md](system-services.md) — system_server, AMS/WMS/PMS, CarService process riêng

### 3. Runtime (ART)
6. [art/android-art.md](art/android-art.md) — DEX, AOT/JIT, ART vs Dalvik
7. [art/art-configure.md](art/art-configure.md) — compilation filters, dexpreopt
8. [art/art-jit.md](art/art-jit.md) — profile-guided, vòng đời interpreter→JIT→AOT
9. [art/art-service.md](art/art-service.md) — ART Service (14+), pm.dexopt.*

### 4. Boot
10. [zygote.md](zygote.md) — fork + copy-on-write, preload
11. [boot-process.md](boot-process.md) — 7 bước Boot ROM → launcher, init.rc

### 5. Kernel
12. [kernel/android-kernel.md](kernel/android-kernel.md) — ACK, GKI, KMI tổng quan
13. [kernel/android-ack.md](kernel/android-ack.md) — nhánh kernel, vòng đời, upstream-first
14. [kernel/android-gki.md](kernel/android-gki.md) — GKI chi tiết, boot partitions
15. [kernel/android-lkm.md](kernel/android-lkm.md) — vendor modules, modprobe, ABI/CRC

### 6. HAL
16. [hal/android-hal.md](hal/android-hal.md) — binderized/SP/wrapped, AIDL vs HIDL
17. [hal/aidl-hals.md](hal/aidl-hals.md) — @VintfStability, NDK backend, freeze version
18. [hal/vintf-objects.md](hal/vintf-objects.md) — manifest vs compatibility matrix
19. [hal/android-vhal.md](hal/android-vhal.md) — vehicle property, get/set/subscribe

### 7. Bảo mật & hiển thị
20. [selinux.md](selinux.md) — MAC, context/label, `.te`, trị `avc: denied` (nợ Phase 4 trả sớm)
21. [graphics-pipeline.md](graphics-pipeline.md) — BufferQueue, VSync, SurfaceFlinger, HWC, multi-display

### 8. Update & tổng hợp
22. [mainline/android-mainline.md](mainline/android-mainline.md) — APEX, update ngoài chu kỳ OS
23. [carservice-vhal-roles.md](carservice-vhal-roles.md) — phân vai CarService vs VHAL ⭐ note khép phase

### Đọc thêm (chưa có note, tự tìm hiểu nếu dư thời gian)
- **App sandbox & permission internals** — mỗi app 1 UID, permission map sang GID; nền của multi-user theo driver profile trên AAOS. Docs: [Application sandbox](https://source.android.com/docs/security/app-sandbox)
- **Process lifecycle & LMK** — `oom_adj_score`, vì sao app bị kill; xe RAM cố định nên hay gặp. Docs: [Low Memory Killer](https://source.android.com/docs/core/perf/lmkd)

## 10 ý phải nhớ khi rời Phase 1

1. Android = 6 layer; framework không bao giờ gọi thẳng driver — luôn qua HAL.
2. Java ↔ Native chỉ có 2 cầu: **JNI** (cùng process) và **Binder** (khác process).
3. Binder cho RPC + calling identity (UID/PID) → permission check tại boundary; buffer 1MB, pool ~16 thread.
4. Manager class trong app chỉ là proxy; service thật sống trong `system_server` (fork từ Zygote).
5. Zygote = preload 1 lần + fork + copy-on-write → app start nhanh, RAM share.
6. Boot: ROM → bootloader (AVB) → kernel (GKI) → init (PID 1, init.rc) → Zygote → system_server.
7. ART hybrid: phone = AOT dần theo profile; **car = AOT sẵn từ nhà máy** (dexpreopt, không có idle+sạc).
8. Kernel: GKI (Google) + vendor modules (.ko), ghép qua KMI — lệch version = `disagrees about version of symbol`.
9. Treble: vendor ↔ framework tách bằng VINTF (manifest "có" match matrix "cần").
10. **VHAL giữ dữ kiện, CarService giữ chính sách** — app chỉ thấy Car API, không bao giờ thấy HAL.

## Tự kiểm tra cuối phase

- [ ] Vẽ lại từ trí nhớ: luồng `App → CarPropertyManager → CarService → VHAL → CAN` và chỉ ra chỗ nào Binder, chỗ nào permission check
- [ ] Giải thích được vì sao camera lùi không đi qua Java stack
- [ ] Trả lời hết câu hỏi tự kiểm tra trong 23 note trên

## 🎯 Thực hành khép phase (từ roadmap)

Trên AAOS emulator: dùng `dumpsys`, `ps`, đọc log boot — map từng note vào thực tế:

```bash
adb shell ps -A | grep -E 'zygote|system_server|com.android.car'
adb shell service list | head -30
adb shell dumpsys car_service | head -50
adb shell getprop | grep ro.boottime
adb logcat -b events | grep boot_progress
```

## Tiếp theo → [Phase 2: C/C++ & JNI](../android-automotive-developer-roadmap.md#phase-2-cc--jni-cho-native-layer)

Phase 1 đọc hiểu — Phase 2 bắt đầu **viết code** ở tầng native: C/C++, JNI, rồi AIDL interface thật.
