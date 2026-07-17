---
title: repo tool — sync source AOSP
source: https://source.android.com/docs/setup/download
phase: 3 — Embedded Linux & AOSP Build
tags:
  - AOSP
  - repo
  - git
  - manifest
---

# repo tool — sync source AOSP

> Thuộc **Phase 3 — Embedded Linux & AOSP Build** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc **sau** [shell-scripting.md](shell-scripting.md) (repo chạy trong terminal) và **trước** [soong-make.md](soong-make.md) — có source rồi mới build được. Bước thực tế đầu tiên chạm vào AOSP.
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## Vấn đề repo giải quyết

AOSP **không phải một git repo** — nó là **~1000 git repo con** (framework, mỗi HAL, kernel, mỗi vendor lib...). Không ai clone tay 1000 repo và giữ chúng đồng bộ đúng nhánh. **`repo`** là tool Python của Google bọc quanh git, quản lý cả rừng repo đó như **một cây source duy nhất**, dựa trên một file **manifest** liệt kê "cần repo nào, ở nhánh nào, đặt vào thư mục nào".

`repo` **không thay git** — mỗi thư mục con vẫn là git repo bình thường (`git log`, `git diff` chạy như thường). `repo` chỉ điều phối thao tác **xuyên nhiều repo cùng lúc**.

## Manifest — bản thiết kế của cây source

```
.repo/manifests/default.xml   ← file XML liệt kê mọi project
```

```xml
<manifest>
  <remote name="aosp" fetch="https://android.googlesource.com/" />
  <default revision="android-14.0.0_r1" remote="aosp" />

  <project path="frameworks/base" name="platform/frameworks/base" />
  <project path="hardware/interfaces" name="platform/hardware/interfaces" />
  <!-- ...~1000 dòng project... -->
</manifest>
```

- **`revision`** = nhánh/tag → đây là cách **ghim đúng một phiên bản Android**.
- Mỗi `<project>` map một git repo (`name`) vào một đường dẫn trong cây (`path`).
- OEM/vendor thường có **manifest riêng** (thêm repo device/vendor của họ) — đây là cách một dự án xe mở rộng AOSP gốc.

## Workflow chuẩn — init rồi sync

```bash
# 1. Cài repo tool
mkdir -p ~/.bin && curl https://storage.googleapis.com/git-repo-downloads/repo > ~/.bin/repo
chmod a+x ~/.bin/repo && export PATH=~/.bin:$PATH

# 2. Tạo thư mục làm việc & init — chọn phiên bản qua -b (branch)
mkdir aosp && cd aosp
repo init -u https://android.googlesource.com/platform/manifest -b android-14.0.0_r1

# 3. Sync — tải source thật (hàng trăm GB, hàng giờ)
repo sync -c -j8
```

- `-b <branch>` chọn phiên bản Android — dùng tag release ([bảng branch chính thức](https://source.android.com/docs/setup/reference/build-numbers)).
- `-c` (current-branch) chỉ tải nhánh cần → nhẹ hơn nhiều so với tải cả lịch sử mọi nhánh.
- `-j8` = 8 luồng song song. Cần **≥250 GB đĩa trống** + RAM khá; sync lần đầu rất lâu.
- Sync lại sau này: `repo sync -c -j8` lấy cập nhật mới nhất theo manifest.

## Lệnh repo hay dùng

| Lệnh | Việc |
|------|------|
| `repo sync` | Tải/cập nhật mọi repo theo manifest |
| `repo status` | Trạng thái git **xuyên tất cả** repo có thay đổi |
| `repo start <topic> .` | Tạo branch làm việc trong repo hiện tại (chuẩn trước khi sửa code) |
| `repo forall -c '<cmd>'` | Chạy một lệnh shell trong **mọi** repo (vd `repo forall -c 'git log -1'`) |
| `repo upload` | Đẩy commit lên **Gerrit** review (không phải `git push`) — xem [glossary.md](../glossary.md) |
| `repo diff` | Diff xuyên các repo đã đổi |

⚠️ AOSP review qua **Gerrit**, không phải GitHub PR. Sửa code → `repo start` → commit → `repo upload` tạo một **change** trên Gerrit (Phase 8 đi sâu).

## 🚗 Liên hệ Automotive

- **Manifest OEM**: dự án xe thật hiếm khi sync manifest AOSP trần. OEM/Tier-1 phát một **manifest riêng** trỏ tới AOSP gốc + thêm repo `device/<oem>/...`, `vendor/...`, kernel riêng, HAL riêng. Onboard job automotive thường bắt đầu bằng "đây là URL manifest của dự án, `repo init` cái này".
- **Ghim version**: safety/ASPICE đòi build **tái lập được** — manifest ghim đúng revision từng repo là cơ chế đảm bảo "build hôm nay giống build 6 tháng trước". Không ai để nhánh trôi tự do.
- **Đĩa & thời gian thật**: source AAOS + vendor có thể **>300 GB**, sync + build đầu tiên tính bằng giờ. Đây là cú sốc đầu tiên của app dev — chu kỳ dài là bình thường ở platform (Phase 8).
- **`repo forall`** cứu mạng khi cần patch/kiểm tra xuyên hàng trăm repo cùng lúc (vd tìm repo nào có commit chưa upload).

## Câu hỏi tự kiểm tra

- [ ] Vì sao AOSP cần `repo` thay vì một git clone? repo có thay git không?
- [ ] Manifest là gì, chứa thông tin gì? `revision` để làm gì?
- [ ] `repo init -b <branch>` chọn cái gì? Tìm tên branch ở đâu?
- [ ] `-c` và `-j8` trong `repo sync` làm gì?
- [ ] Sửa code AOSP xong đẩy lên đâu, bằng lệnh gì? Khác GitHub PR chỗ nào?
- [ ] Dự án xe mở rộng AOSP gốc bằng cách nào (gợi ý: manifest)?
- [ ] `repo forall` dùng khi nào?

## 📚 Tài liệu

- 📖 [Download the Android source](https://source.android.com/docs/setup/download) — làm theo từng bước
- 📖 [repo command reference](https://source.android.com/docs/setup/reference/repo) — mọi lệnh repo
- 📖 [Codelines, branches, and releases](https://source.android.com/docs/setup/reference/build-numbers) — tra tag branch
- 📖 [Contribute (Gerrit workflow)](https://source.android.com/docs/setup/contribute) — repo upload → Gerrit

## Đọc tiếp

- [soong-make.md](soong-make.md) — có source rồi: `source envsetup.sh`, `lunch`, `m` để build.
- [build-aaos-image.md](build-aaos-image.md) — build image AAOS từ source vừa sync.
- [shell-scripting.md](shell-scripting.md) — cả workflow repo/build sống trong terminal.
