---
title: Graphics Pipeline (SurfaceFlinger, BufferQueue, VSync)
phase: 1 — Android Internals
tags:
  - AOSP
  - SurfaceFlinger
  - Graphics
  - HWC
  - Display
---

# Graphics Pipeline — SurfaceFlinger, BufferQueue, VSync

> Thuộc **Phase 1 — Android Internals** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc sau [system-services.md](system-services.md) — SurfaceFlinger xuất hiện 3 note rồi, giờ mổ xẻ nó.

Graphics pipeline là **chuỗi bước đưa hình ảnh từ app ra tới điểm ảnh trên màn hình** — từ lúc app vẽ nội dung của nó cho đến khi màn refresh và bạn nhìn thấy. Điều đầu tiên cần gỡ khỏi đầu (nhất là với app dev): **app không bao giờ vẽ thẳng ra màn hình**. Mỗi app chỉ vẽ vào một vùng nhớ riêng (buffer) của nó, rồi giao buffer đó cho hệ thống. Lý do là màn hình tại một thời điểm bị **nhiều thứ cùng tranh nhau**: app của bạn, status bar, thanh navigation, notification, các app khác ở chế độ chia đôi màn hình... Nếu ai cũng được ghi trực tiếp lên màn thì hình sẽ xé nát và giẫm đè lên nhau.

Vì thế Android tách vai trò ra: mỗi app là một **producer** vẽ hình vào buffer của mình, còn một service trung tâm tên **SurfaceFlinger** đóng vai **consumer tổng** — nó gom buffer của tất cả các app cùng những layer hệ thống rồi **compose** (ghép lớp, quyết cái nào đè cái nào) thành một frame cuối cùng duy nhất, đúng nhịp refresh của màn. Cứ hình dung SurfaceFlinger như **một tổng đạo diễn dựng phim**: mỗi app quay cảnh riêng của mình rồi nộp thước phim; đạo diễn xếp lớp, cắt ghép thành một khung hình hoàn chỉnh trước khi chiếu lên màn. Sơ đồ dưới đi qua từng chặng của hành trình đó.

## Toàn cảnh: 1 frame từ app ra màn hình

```
App (UI thread + RenderThread)
 │ vẽ vào buffer (GPU/Skia)
 ▼
BufferQueue  (producer = app, consumer = SurfaceFlinger)
 │ queue buffer đã vẽ xong
 ▼
SurfaceFlinger  (native service, process riêng)
 │ compose các layer (mọi app + status bar + ...)
 ▼
HWC (Hardware Composer HAL)  ← vendor implement
 │ overlay plane phần cứng / fallback GPU
 ▼
Display(s)
```

## Từng mảnh

### Surface & BufferQueue
- **Surface** = producer đầu ghi của 1 BufferQueue; mỗi window 1 cái.
- **BufferQueue** = vòng 2-3 buffer luân chuyển producer ↔ consumer (double/triple buffering) — app vẽ buffer này trong khi màn hiện buffer kia.
- App không bao giờ vẽ thẳng ra màn — luôn qua buffer.

### VSync & Choreographer
- **VSync** — tick phần cứng mỗi lần màn refresh (60Hz = 16.6ms/frame).
- **Choreographer** (trong app) nhận VSync → gọi `doFrame()`: input → animation → measure/layout → draw.
- Trễ deadline 1 tick = **jank** (giật): frame cũ hiện lại thêm 16.6ms.

### SurfaceFlinger
- Native service ([system-services.md](system-services.md)) — **consumer tổng**: gom layer mọi app, quyết cái gì đè cái gì, compose ra frame cuối.
- Nhận VSync riêng (offset với app VSync để pipeline gối nhau).

### HWC (Hardware Composer HAL)
- Vendor HAL ([hal/android-hal.md](hal/android-hal.md)) — SurfaceFlinger hỏi: "mấy layer này, phần cứng tự overlay được không?"
- Được → display controller ghép trực tiếp, **không tốn GPU**; không → SurfaceFlinger fallback compose bằng GPU (`Client` composition).
- Xem thực tế: `adb shell dumpsys SurfaceFlinger` — bảng layer cuối có cột `HWC`/`Client`.

## Lệnh thực hành

```bash
adb shell dumpsys SurfaceFlinger            # layer stack, HWC vs Client
adb shell dumpsys gfxinfo <package>         # histogram frame time app
adb shell dumpsys SurfaceFlinger --latency  # timing từng frame
# Perfetto: bật track "SurfaceFlinger" + "VSync" → thấy jank trực quan
```

## 🚗 Liên hệ Automotive

- **Multi-display là chuyện thường**: IVI (màn giữa) + instrument cluster (màn đồng hồ sau vô lăng) + HUD (Head-Up Display — chiếu lên kính lái) + màn ghế sau — 1 SurfaceFlinger quản nhiều display, mỗi display 1 layer stack riêng.
- **Cluster ≠ được phép jank**: kim tốc độ giật là lỗi an toàn (nhiều OEM tách cluster sang chip/OS riêng hoặc guest VM riêng vì thế — xem hypervisor Phase 7).
- **Camera lùi (EVS)** đi đường riêng cắm gần display, **bỏ qua SurfaceFlinger** — vì deadline < 2s và độ trễ frame thấp ([boot-process.md](boot-process.md)).
- GPU trên SoC xe yếu hơn phone flagship → tối đa hóa **HWC overlay**, tránh Client composition là việc tune thật của platform team.

## Câu hỏi tự kiểm tra

- [ ] Vẽ lại đường đi 1 frame từ app ra màn hình?
- [ ] BufferQueue có mấy buffer, vì sao không phải 1?
- [ ] VSync là gì, jank xảy ra khi nào?
- [ ] HWC composition vs Client composition — khác gì, cái nào rẻ hơn?
- [ ] Vì sao EVS camera lùi bỏ qua SurfaceFlinger?
- [ ] Lệnh nào xem layer đang HWC hay Client?

## 📚 Tài liệu

- 📖 [Graphics architecture](https://source.android.com/docs/core/graphics/architecture) — docs chính thức, đọc kỹ nhất
- 📖 [BufferQueue & Surface](https://source.android.com/docs/core/graphics/arch-bq-gralloc)
- 📖 [SurfaceFlinger & HWC](https://source.android.com/docs/core/graphics/arch-sf-hwc)
- 📖 [VSync](https://source.android.com/docs/core/graphics/implement-vsync)
- 📖 [Multi-display AAOS](https://source.android.com/docs/automotive/displays/multi_display)

## Đọc tiếp

- [system-services.md](system-services.md) — SurfaceFlinger là native service
- [hal/android-hal.md](hal/android-hal.md) — HWC là 1 HAL
- [boot-process.md](boot-process.md) — EVS start sớm
