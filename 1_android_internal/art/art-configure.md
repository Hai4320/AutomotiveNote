---
title: Cấu hình ART (Configure ART)
source: https://source.android.com/docs/core/runtime/configure?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - ART
  - dex2oat
  - dexpreopt
  - Performance
---

# Cấu hình ART

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/runtime/configure](https://source.android.com/docs/core/runtime/configure?hl=vi)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../../android-automotive-developer-roadmap.md). Đọc sau [android-art.md](android-art.md).

## "Cấu hình ART" nghĩa là gì?

ART không compile mọi app theo một kiểu cứng nhắc — nó có **nhiều núm vặn (knob)** để quyết định: app này compile sâu tới đâu, compile lúc nào, dùng bao nhiêu CPU để compile. "Cấu hình ART" chính là chỉnh những núm đó qua build variable (lúc build image) và system property (lúc chạy).

Vì sao phải chỉnh? Vì luôn có **đánh đổi tay ba**: compile càng nhiều thì app chạy càng nhanh, nhưng **tốn dung lượng** (file `.oat` to) và **tốn thời gian compile** (cài lâu, hoặc ngốn CPU lúc chạy nền). Không cấu hình nào tối ưu cho mọi app: launcher cần mở tức thì nên compile sẵn hết, còn app hiếm dùng thì để nhẹ cho đỡ phí chỗ. OEm (đặc biệt xe hơi — xem mục 🚗 cuối) tinh chỉnh mấy núm này để cân boot time, dung lượng và độ mượt.

Cụ thể có 2 nhóm núm: **compilation filter** (compile *sâu* tới mức nào — mục ngay dưới) và **dexpreopt/threading** (compile *khi nào* và bằng *bao nhiêu* CPU).

## Compilation filters (dex2oat)

4 filter chính — quyết định DEX được xử lý sâu đến đâu:

| Filter | Làm gì | Trade-off |
|--------|--------|-----------|
| `verify` | Chỉ verify DEX, **không AOT** | Cài nhanh, chạy chậm (interpreter/JIT) |
| `quicken` (≤ Android 11) | Verify + tối ưu instruction cho interpreter | Đã bỏ từ Android 12 |
| `speed` | Verify + **AOT toàn bộ method** | Chạy nhanh nhất, tốn dung lượng + thời gian compile |
| `speed-profile` | Verify + AOT **chỉ method trong profile** + tối ưu class-loading | Cân bằng tốt nhất — mặc định hiện đại |

## Dexpreopt — compile sẵn lúc build image

App trong system image được **precompile lúc build AOSP** (không phải lúc cài trên máy):

| Biến build | Vai trò |
|-----------|---------|
| `WITH_DEXPREOPT` | Bật/tắt dex2oat khi build image (**mặc định bật**) |
| `PRODUCT_DEX_PREOPT_DEFAULT_COMPILER_FILTER` | Filter mặc định cho app (Android 9+): `speed-profile`, fallback `verify` nếu không có profile |
| `PRODUCT_SYSTEM_SERVER_COMPILER_FILTER` | Filter cho system server JARs — Android 14+: `speed-profile` (hoặc `speed` nếu không profile); trước đó: `speed` |
| `LOCAL_DEX_PREOPT` | Bật/tắt per-module: `true` / `false` / `nostripping` |

## JIT runtime properties (`dalvik.vm.*`)

| Property | Default | Ý nghĩa |
|----------|---------|---------|
| `dalvik.vm.usejit` | true | Bật/tắt JIT |
| `dalvik.vm.jitthreshold` | 10000 | Hotness counter — method "nóng" đến ngưỡng này thì JIT compile |
| `dalvik.vm.jitinitialsize` | 64K | Code cache khởi tạo |
| `dalvik.vm.jitmaxsize` | 64M | Code cache tối đa |

## Dex2oat threading

| Property | Ý nghĩa |
|----------|---------|
| `dalvik.vm.dex2oat-threads` | Số thread compile |
| `dalvik.vm.dex2oat-cpu-set` | Ghim compile vào core cụ thể, vd `0,1,2,3` |

- Set cpu-set khớp số thread → tránh contention với app đang chạy.

🚗 *Liên hệ Automotive:* đây là các knob OEM tune boot time & cold start cho AAOS:
- System app quan trọng (launcher, cluster) → dexpreopt `speed-profile`/`speed` để không phải compile trên xe.
- `dex2oat-cpu-set` ghim compile vào core nhỏ (little cores) → update app nền không giật UI khi đang lái.
- Đổi các property này trong `device.mk` / `BoardConfig.mk` của device tree.

## Câu hỏi tự kiểm tra

- [ ] 4 compilation filter — khác nhau gì, mặc định hiện đại là gì?
- [ ] Dexpreopt khác gì compile lúc cài đặt? Biến nào bật/tắt?
- [ ] System server dùng filter gì từ Android 14?
- [ ] `jitthreshold` hoạt động thế nào?
- [ ] Vì sao cần `dex2oat-cpu-set`?

## 📚 Tài liệu

- 📖 [JIT compiler](https://source.android.com/docs/core/runtime/jit-compiler)
- 📖 [Boot image / dexpreopt](https://source.android.com/docs/core/runtime/boot-image)

## Đọc tiếp

- [android-art.md](android-art.md) — ART tổng quan
- [art-jit.md](art-jit.md) — JIT & profile-guided
- [art-service.md](art-service.md) — dexopt từ Android 14
