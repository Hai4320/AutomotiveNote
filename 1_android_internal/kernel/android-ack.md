---
title: Android Common Kernel (ACK)
source: https://source.android.com/docs/core/architecture/kernel/android-common?hl=vi
phase: 1 — Android Internals
tags:
  - AOSP
  - Kernel
  - ACK
  - KMI
---

# Android Common Kernel (ACK)

> Ghi chú từ tài liệu chính thức: [source.android.com/docs/core/architecture/kernel/android-common](https://source.android.com/docs/core/architecture/kernel/android-common?hl=vi)
> Thuộc **Phase 1 — Android Internals** trong [roadmap](../../android-automotive-developer-roadmap.md). Đọc sau [android-kernel.md](android-kernel.md).

## ACK là gì?

- Kernel dựa trên **kernel.org (upstream)**, Google duy trì + thêm patch cộng đồng Android cần.
- Patch gồm: chức năng chưa kịp upstream, feature đang phát triển, vendor tooling.

## Cấu trúc nhánh

```
Linux mainline (Linus release)
      │ merge liên tục
      ▼
android-mainline          ← nhánh phát triển chính
      │ fork theo release
      ▼
androidRELEASE-KERNEL     ← nhánh GKI/KMI, vd: android15-6.6, android14-6.1
```

| Nhánh | Vai trò |
|-------|---------|
| **android-mainline** | Dev chính — merge Linux mới ngay khi Linus release, feature Android vào đây trước |
| **`androidXX-Y.Y`** (GKI branches) | Nhánh ổn định device ship, đặt tên `ANDROID_RELEASE-KERNEL_VERSION` |

## Vòng đời nhánh GKI (3 giai đoạn)

| Giai đoạn | Nhận gì |
|-----------|---------|
| 1. **Development** | Feature mới thoải mái |
| 2. **Stabilization** | KMI được giám sát thay đổi interface |
| 3. **KMI frozen** | Chỉ nhận security patch nghiêm trọng |

- Hỗ trợ **4–6 năm** tùy version. Hết EOL = coi như **vulnerable**, phải chuyển nhánh mới.

## Tương thích KMI

- GKI kernel **tương thích ngược** với mọi Android release hỗ trợ version đó.
- KMI **ổn định giữa các patch cùng nhánh** (vd mọi build của `android15-6.6`).
- ⚠️ **KHÔNG** tương thích KMI **giữa các nhánh GKI khác nhau** (`android14-6.1` ↛ `android15-6.6`) — vendor module phải rebuild khi đổi nhánh.

## Quy trình đóng góp (upstream-first)

1. Phát triển feature trên **Linux mainline trước** (upstream-first policy).
2. Được upstream chấp nhận → backport về nhánh ACK nếu cần.
3. Gửi patch qua **Gerrit**: [android-review.googlesource.com](https://android-review.googlesource.com)

🚗 *Liên hệ Automotive:* dự án xe chọn 1 nhánh ACK (vd `android14-6.1`) và **gắn chặt nhiều năm** — vòng đời xe dài hơn consumer phone, nên EOL 4–6 năm của ACK là ràng buộc thật khi chọn platform. Đổi nhánh kernel giữa chừng = rebuild toàn bộ vendor module (KMI không tương thích chéo nhánh).

## Câu hỏi tự kiểm tra

- [ ] ACK khác kernel.org chỗ nào? Patch gồm những loại gì?
- [ ] `android-mainline` vs `android15-6.6` — vai trò từng nhánh?
- [ ] 3 giai đoạn vòng đời nhánh GKI? KMI frozen nhận gì?
- [ ] KMI có tương thích giữa 2 nhánh GKI khác nhau không? Hệ quả với vendor module?
- [ ] Upstream-first policy là gì, vì sao Google ép?

## 📚 Tài liệu

- 📖 [GKI release builds](https://source.android.com/docs/core/architecture/kernel/gki-release-builds)
- 📖 [Contribute to AOSP kernel](https://source.android.com/docs/setup/contribute)

## Đọc tiếp

- [android-gki.md](android-gki.md) — GKI chi tiết
- [android-lkm.md](android-lkm.md) — vendor modules
