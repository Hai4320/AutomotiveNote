# Chuẩn format note & phase (dựa trên Phase 1)

> Áp dụng cho mọi phase từ Phase 2 trở đi. Phase 1 (`1_android_internal/`) là ví dụ mẫu hoàn chỉnh.

## Cấu trúc thư mục

```
N_ten_phase/
├── README.md          ← index + tổng kết phase (bắt buộc)
├── chu-de-a.md        ← note đơn lẻ
└── nhom-chu-de/       ← subdir khi 1 chủ đề có 3+ note (như kernel/, hal/, art/)
    └── chu-de-b.md
```

- README gốc của repo chỉ link tới roadmap + [glossary.md](glossary.md) + README từng phase — **không** index từng note.
- **glossary.md ở root dùng chung mọi phase**: thuật ngữ mới của phase → thêm vào đây (không tạo glossary riêng từng phase). Note link về bằng đường dẫn tương đối (`../glossary.md` nếu note nằm trong `N_ten_phase/`).

## Format 1 note

```markdown
---
title: <Tên chủ đề>
source: <URL docs chính thức — chỉ khi note tóm tắt từ 1 trang cụ thể>
phase: <N — Tên phase>
tags:
  - <3-5 tag>
---

# <Tên chủ đề>

> Ghi chú từ tài liệu chính thức: [<link>](<url>)        ← nếu có source
> Thuộc **Phase N** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc sau [note-truoc.md](note-truoc.md).

## <Khái niệm là gì?>          ← mở đầu: định nghĩa + vì sao tồn tại

## <Các section nội dung>      ← tự do, nhưng ưu tiên:
                                  - sơ đồ ASCII cho kiến trúc/luồng
                                  - bảng cho so sánh/liệt kê
                                  - code block cho lệnh & config thật

## 🚗 Liên hệ Automotive       ← BẮT BUỘC (section riêng hoặc đoạn 🚗 inline):
                                  áp dụng thật trên xe/AAOS, bẫy kinh điển, lệnh debug

## Câu hỏi tự kiểm tra         ← 4-6 câu checkbox, trả lời được = hiểu note

## 📚 Tài liệu                 ← CHỈ link ngoài: docs chính thức (source.android.com,
                                  developer.android.com), source code (cs.android.com),
                                  sách, spec. KHÔNG video/YouTube.

## Đọc tiếp                    ← CHỈ link note nội bộ trong repo + 1 dòng vì sao đọc
```

## Format README phase

```markdown
# Phase N — <Tên>: Tổng kết

> Mục tiêu phase + điều kiện "học xong".

## Thứ tự đọc                  ← note đánh số, nhóm theo chủ đề
### 1. <Nhóm A>
1. [note.md](note.md) — 1 dòng tóm tắt
### Đọc thêm                   ← chủ đề chưa có note, kèm link docs

## 10 ý phải nhớ khi rời phase ← mỗi note chắt 1 câu

## Tự kiểm tra cuối phase      ← 2-3 bài tổng hợp (vẽ lại luồng từ trí nhớ...)

## 🎯 Thực hành khép phase     ← project/lệnh từ roadmap

## Tiếp theo → [Phase N+1](../android-automotive-developer-roadmap.md#...)
```

## Quy tắc chung

1. **Tiếng Việt, thuật ngữ kỹ thuật giữ tiếng Anh** (Binder, property, freeze...).
2. Mỗi note **1 chủ đề, đọc 5–10 phút**; dài hơn thì tách.
3. Link chéo tối đa giữa các note — kiến thức là đồ thị, không phải list.
4. Có lệnh chạy được (`adb shell ...`) ở mọi note có thể thực hành.
5. Sau khi tạo note: thêm vào **README của phase** (thứ tự đọc + đếm lại số note).
6. Move file = sửa relative links ngay (roadmap `../` hay `../../` theo độ sâu).
```
