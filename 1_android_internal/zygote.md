---
title: Zygote Process
phase: 1 — Android Internals
tags:
  - AOSP
  - Zygote
  - Process
  - Boot
---

# Zygote Process

> Thuộc **Phase 1 — Android Internals** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc sau [system-services.md](system-services.md), trước Boot Process.

## Zygote là gì?

Zygote là một process **"khuôn mẫu" (template) mà mọi app process trên Android đều fork ra từ đó** — kể cả `system_server`. Tên nó nghĩa đen là "hợp tử": giống một tế bào gốc, mọi process con "sinh ra" từ nó bằng `fork()` và thừa hưởng toàn bộ trạng thái đã chuẩn bị sẵn. Nó khởi động một lần lúc boot bởi `init` (từ `init.zygote64.rc`): chạy binary `app_process` → khởi tạo ART VM → vào `ZygoteInit.java` rồi ngồi chờ lệnh fork.

Vì sao lại cần một process trung gian như vậy thay vì để mỗi app tự khởi động từ đầu? Vì khởi tạo một máy ảo Java và nạp cả framework Android là **rất tốn thời gian và RAM**. Nếu quen với app dev, hãy nghĩ tới việc mỗi lần mở Activity mà phải khởi động lại toàn bộ JVM + nạp lại mọi class `android.*` — mở app nào cũng lag vài giây. Zygote làm phần nặng đó **đúng một lần** rồi cho con cái copy lại (chi tiết ở mục dưới).

Ví dụ dễ hình dung: Zygote như **cục bột đã nhào sẵn men và bột nở**; mỗi khi cần một ổ bánh (app mới) chỉ việc **véo một cục ra nướng**, không phải nhào lại từ đầu — nhanh hơn nhiều so với trộn bột mới mỗi lần.

## Vì sao cần Zygote? — bài toán cold start

Không có Zygote, mỗi app phải: khởi tạo VM + load hàng nghìn framework class + resource → **vài giây mỗi lần mở app**.

Giải pháp: làm 1 lần trong Zygote, mọi app fork ra **thừa hưởng miễn phí**:

```
Boot:
init → Zygote start
        ├─ khởi tạo ART VM
        ├─ preload classes    (hàng nghìn class framework — frameworks/base/config/preloaded-classes)
        ├─ preload resources  (theme, drawable hệ thống)
        └─ lắng nghe socket /dev/socket/zygote

Mở app:
AMS (system_server) ──socket──▶ Zygote
                                  │ fork()
                                  ▼
                             App process mới
                                  │ (đã có sẵn VM + preloaded classes)
                                  ▼
                             ActivityThread.main() → chạy app
```

## Cơ chế then chốt: fork + Copy-on-Write (COW)

- `fork()` tạo bản sao process — nhưng Linux **không copy RAM thật**, các page memory **share** giữa Zygote và app con.
- Chỉ khi ai đó **ghi** vào page thì page mới bị copy (copy-on-write).
- Preloaded classes/resources là **read-only** → hàng chục app cùng share 1 bản trong RAM.
- Kết quả: **khởi động app nhanh** + **tiết kiệm RAM** toàn hệ thống.

## Điểm cần nhớ

| Điều | Chi tiết |
|------|----------|
| Ai yêu cầu fork? | `ActivityManagerService` (qua socket `/dev/socket/zygote`, **không phải Binder** — vì fork không an toàn với thread Binder) |
| system_server | Cũng fork từ Zygote (đầu tiên, đặc biệt) |
| Zygote64/32 | Device 64-bit thường chạy 2 Zygote: `zygote64` (chính) + `zygote_secondary` (app 32-bit) |
| Zygote chết? | init restart Zygote → **system_server chết theo → cả hệ thống UI restart** ("soft reboot") |
| USAP pool | Unspecialized App Process — pool process fork sẵn chờ dùng, giảm latency thêm (Android 10+) |

## 🚗 Liên hệ Automotive

- **Boot time KPI**: preload list là knob tune — OEM thêm class Car API (`android.car.*`) vào preload để app cockpit mở nhanh; nhưng preload nhiều = Zygote khởi động chậm → trade-off boot time vs app start.
- Camera lùi phải lên < 2s (quy định an toàn) — thường làm ở tầng native/EVS (Extended View System — stack camera native riêng của AAOS) **không chờ Zygote/Java stack**.
- `ro.boot.` + `adb shell ps -A | grep zygote` xem Zygote trên AAOS emulator.

## Câu hỏi tự kiểm tra

- [ ] Zygote giải quyết bài toán gì? Không có nó thì sao?
- [ ] Preload gồm những gì, khai báo ở đâu?
- [ ] Copy-on-Write hoạt động thế nào, tiết kiệm gì?
- [ ] AMS yêu cầu Zygote fork qua socket hay Binder? Vì sao?
- [ ] Zygote chết thì chuyện gì xảy ra?
- [ ] system_server sinh ra từ đâu?

## 📚 Tài liệu

- 📖 [Overview of memory management](https://developer.android.com/topic/performance/memory-overview) — docs chính thức, giải thích Zygote share RAM
- 📖 [ZygoteInit.java (source)](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java) — preload + fork loop thật
- 📖 [Preloaded classes list (source)](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/config/preloaded-classes)

## Đọc tiếp

- [system-services.md](system-services.md) — note trong repo
- [art/android-art.md](art/android-art.md) — note trong repo, VM mà Zygote khởi tạo
- [binder-ipc.md](binder-ipc.md) — note trong repo
