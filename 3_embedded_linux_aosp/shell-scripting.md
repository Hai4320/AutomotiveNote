---
title: Shell scripting — bash, pipe, grep/sed/awk
source: https://missing.csail.mit.edu/
phase: 3 — Embedded Linux & AOSP Build
tags:
  - Linux
  - Bash
  - Shell
  - grep
  - sed
  - awk
---

# Shell scripting — bash, pipe, grep/sed/awk

> Thuộc **Phase 3 — Embedded Linux & AOSP Build** trong [roadmap](../android-automotive-developer-roadmap.md). Đọc **sau** [linux-fundamentals.md](linux-fundamentals.md) (file/process/signal là thứ shell thao tác) và **trước** [repo-tool.md](repo-tool.md), [soong-make.md](soong-make.md) — build AOSP là hàng loạt shell command, envsetup là script bash.
> Thuật ngữ lạ tra ở [glossary.md](../glossary.md).

## Vì sao shell là kỹ năng nền của platform dev?

App dev bấm nút Run trong Android Studio. Platform dev sống trong terminal: sync source (`repo`), set build env (`source build/envsetup.sh`), chọn target (`lunch`), build (`m`), flash, rồi lọc hàng vạn dòng log để tìm một `avc: denied`. Không có GUI cho những việc này. Thạo bash + `grep`/`sed`/`awk` biến việc "đọc log 10 phút" thành "một dòng lệnh". Đây là bội số năng suất, không phải kiến thức trang trí.

## Bash cơ bản — biến, quoting, exit code

```bash
name="can0"                # KHÔNG có khoảng trắng quanh dấu =
echo "iface=$name"         # "  → nội suy biến; '  → literal, không nội suy
echo 'iface=$name'         # in nguyên $name

files=$(ls *.img)          # $(...) = command substitution: lấy output lệnh làm giá trị
echo "exit code: $?"       # $? = exit code lệnh vừa chạy (0 = OK, khác 0 = fail)
```

- **Quoting**: `"$var"` (nháy kép) gần như luôn đúng — không quote thì khoảng trắng trong tên file làm vỡ lệnh. Thói quen: **luôn quote biến**.
- **Exit code** `$?`: `0` = thành công. Mọi script/CI dựa vào quy ước này.
- `&&` chạy lệnh sau nếu lệnh trước OK; `||` nếu fail: `m && echo done || echo BUILD FAILED`.

Script tối thiểu:

```bash
#!/bin/bash               # shebang — nói dùng bash chạy file này
set -euo pipefail         # -e: dừng khi lệnh fail; -u: lỗi khi dùng biến chưa set;
                          # pipefail: pipe fail nếu bất kỳ khâu nào fail
for img in *.img; do
    echo "flashing $img"
done
```

`set -euo pipefail` là dòng đầu chuẩn của script nghiêm túc — không có nó, script "chạy tiếp bất chấp lỗi" và giấu bug.

## Pipe & redirect — ghép lệnh nhỏ thành công cụ lớn

Triết lý Unix: **mỗi lệnh làm một việc, ghép lại qua pipe**. `|` đẩy stdout lệnh trái vào stdin lệnh phải:

```bash
adb logcat | grep -i error | head -20      # log → lọc error → 20 dòng đầu
adb shell ps -A | grep car                 # process → chỉ dòng có "car"
```

Redirect (chuyển hướng luồng):

| Cú pháp | Nghĩa |
|---------|-------|
| `cmd > f` | ghi stdout ra file `f` (đè) |
| `cmd >> f` | nối thêm vào `f` |
| `cmd 2> f` | ghi **stderr** (luồng lỗi) ra `f` |
| `cmd > f 2>&1` | cả stdout **và** stderr vào `f` |
| `cmd < f` | lấy `f` làm stdin |

⚠️ **stdout (1) và stderr (2) là hai luồng riêng**. Build log lỗi thường ra stderr — `m > log.txt` **không** bắt lỗi; cần `m > log.txt 2>&1`.

## grep — tìm dòng khớp pattern

```bash
grep error log.txt              # dòng chứa "error"
grep -i error log.txt           # -i: không phân biệt hoa/thường
grep -r "CarService" .          # -r: đệ quy trong thư mục
grep -n "denied" log.txt        # -n: kèm số dòng
grep -v debug log.txt           # -v: đảo — dòng KHÔNG chứa "debug"
grep -E "avc.*denied" log.txt   # -E: regex mở rộng
dmesg | grep -i "can\|vhal"     # nhiều pattern (\| = hoặc)
```

Lệnh dùng nhiều nhất khi debug system. `grep avc` trong `dmesg` là phản xạ trị SELinux ([selinux.md](../1_android_internal/selinux.md)).

## sed — sửa text theo dòng (stream editor)

```bash
sed 's/foo/bar/'  file          # thay foo→bar, mỗi dòng lần đầu
sed 's/foo/bar/g' file          # g = global, mọi lần trong dòng
sed -i 's/foo/bar/g' file       # -i = sửa thẳng file (in-place)
sed -n '10,20p' file            # in dòng 10–20
sed '/^#/d' file                # xóa dòng comment (bắt đầu bằng #)
```

Hay dùng để sửa hàng loạt config/`.mk`/`.rc` bằng script thay vì mở tay từng file.

## awk — xử lý dữ liệu theo cột

`awk` chia mỗi dòng thành field (`$1`, `$2`, ... theo khoảng trắng), `$0` = cả dòng:

```bash
adb shell ps -A | awk '{print $2, $9}'          # in cột PID và tên process
df -h | awk '$5+0 > 80 {print $6}'              # mount point dùng >80% dung lượng
awk -F: '{print $1}' /etc/passwd                # -F: đổi delimiter thành ":"
adb shell dumpsys meminfo | awk '/TOTAL/{print $2}'  # lọc dòng khớp + in field
```

Quy tắc: `awk 'PATTERN {ACTION}'` — chạy ACTION cho dòng khớp PATTERN. Mạnh khi cần **trích cột số** rồi tính/lọc, thứ `grep` không làm được.

## grep vs sed vs awk — chọn cái nào

| Cần | Dùng |
|-----|------|
| Tìm/lọc dòng theo pattern | `grep` |
| Thay thế / xóa / sửa text | `sed` |
| Trích cột, tính toán trên field | `awk` |

Ghép được: `dmesg | grep avc | awk '{print $NF}'` (in field cuối `$NF` mỗi dòng denied).

## 🚗 Liên hệ Automotive

- **Build AOSP là bash**: `source build/envsetup.sh` (định nghĩa hàm `lunch`, `m`, `mm`...), rồi cả pipeline build/flash chạy trong terminal. Xem [soong-make.md](soong-make.md), [build-aaos-image.md](build-aaos-image.md).
- **Lọc log xe khổng lồ**: bench xe đổ ra hàng vạn dòng logcat/dmesg. Một pipe `logcat | grep -i vhal | grep -v verbose` khoanh vùng bug trong giây.
- **Bắt SELinux denial**: `adb shell dmesg | grep 'avc:.*denied' | awk '{print $6, $7}'` → gom nhanh scontext/tcontext bị chặn.
- **CI/HIL script**: pipeline test tự động trên bench viết bằng bash — `set -euo pipefail` + `$?` để fail đúng khi một bước hỏng, không giả "pass".

```bash
# gom mọi denial của 1 domain HAL trong 1 lần boot
adb shell dmesg | grep 'avc:.*denied' | grep hal_vehicle | sort | uniq -c
```

## Câu hỏi tự kiểm tra

- [ ] `"$var"` vs `'$var'` khác gì? Vì sao nên luôn quote biến?
- [ ] `$?` là gì, quy ước 0 nghĩa là gì? `set -euo pipefail` làm gì?
- [ ] stdout vs stderr — vì sao `m > log.txt` không bắt được lỗi build?
- [ ] Pipe `|` làm gì? Viết lệnh lọc process có "car" từ `ps -A`.
- [ ] grep vs sed vs awk — mỗi cái mạnh việc gì?
- [ ] `sed -i` khác `sed` thường chỗ nào? Nguy hiểm gì?
- [ ] `awk '{print $2}'` in cái gì? `$0` và `$NF` là gì?

## 📚 Tài liệu

- 📖 [MIT Missing Semester](https://missing.csail.mit.edu/) — shell, pipe, tooling (có bài giảng)
- 📖 [GNU Bash manual](https://www.gnu.org/software/bash/manual/bash.html) — tham chiếu chính thức
- 📖 [grep](https://www.gnu.org/software/grep/manual/grep.html) / [sed](https://www.gnu.org/software/sed/manual/sed.html) / [gawk](https://www.gnu.org/software/gawk/manual/gawk.html) manuals
- 📖 [ShellCheck](https://www.shellcheck.net/) — linter bắt lỗi bash phổ biến (quoting, `$?`...)

## Đọc tiếp

- [repo-tool.md](repo-tool.md) — `repo` là script Python nhưng cả workflow sync là shell.
- [soong-make.md](soong-make.md) — `envsetup.sh`, `lunch`, `m` đều là hàm/lệnh shell.
- [linux-fundamentals.md](linux-fundamentals.md) — file/process/signal mà lệnh shell thao tác.
