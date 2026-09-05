# Thi LAN — Hệ thống thi trắc nghiệm qua mạng LAN

> **Phần mềm quản lý và tổ chức thi trắc nghiệm cho phòng máy tính nội bộ (LAN), không cần Internet.**
> Kiến trúc client–server TCP, đồng bộ đề theo thời gian thực, chống gian lận ở phía client.

---

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Kiến trúc](#2-kiến-trúc)
3. [Cấu trúc thư mục](#3-cấu-trúc-thư-mục)
4. [Công nghệ sử dụng](#4-công-nghệ-sử-dụng)
5. [Workflow tổng thể](#5-workflow-tổng-thể)
6. [Cài đặt & chạy](#6-cài-đặt--chạy)
7. [Build EXE](#7-build-exe)
8. [Tự động cập nhật](#8-tự-động-cập-nhật)
9. [Phát hành bản mới](#9-phát-hành-bản-mới)
10. [Quy ước code](#10-quy-ước-code)
11. [Câu hỏi thường gặp](#11-câu-hỏi-thường-gặp)

---

## 1. Tổng quan

**Thi LAN** là phần mềm giúp giáo viên tổ chức thi trắc nghiệm cho học sinh ngay trong phòng máy tính, **chỉ dùng mạng LAN nội bộ** — không phụ thuộc Internet.

### Tính năng chính

| Vai trò | Tính năng |
|---------|-----------|
| **Giáo viên (Admin)** | • Tạo đề từ file Word (.docx), PDF, hoặc Gemini AI<br>• Quản lý ngân hàng câu hỏi, lớp học<br>• Phát đề đến học sinh qua LAN<br>• Theo dõi tiến độ làm bài real-time<br>• Xem bảng điểm, xuất Excel<br>• Giám thị online, đóng/mở ca thi |
| **Học sinh (Student)** | • Vào phòng thi theo mã phòng<br>• Làm bài với giao diện giống Azota<br>• Tự động lưu bài khi mất kết nối<br>• Nộp bài khi hết giờ |

### Đặc điểm kỹ thuật

- ✅ **Offline**: Hoạt động hoàn toàn trong mạng LAN — không cần Internet (trừ lúc check update).
- ✅ **Real-time**: Dữ liệu đồng bộ qua TCP socket, cập nhật bảng điểm trong < 1 giây.
- ✅ **Auto-discovery**: Học sinh tự tìm thấy phòng thi mà không cần nhập IP.
- ✅ **Chống gian lận**: Chặn phím tắt, USB, copy, screenshot, kill Chrome/task manager (Windows).
- ✅ **Đề an toàn**: Đề mã hoá khi truyền, xáo trộn câu/đáp án theo từng học sinh.
- ✅ **Tự cập nhật**: Tự động phát hiện bản mới qua GitHub Releases.

---

## 2. Kiến trúc

```
┌───────────────────────────┐                ┌───────────────────────────┐
│  MÁY GIÁO VIÊN            │                │  MÁY HỌC SINH (x N)       │
│                           │                │                           │
│  ┌─────────────────────┐ │    TCP LAN      │  ┌─────────────────────┐  │
│  │ ThiLAN-thi-lan-admin-Setup.exe │ │ ◄────────────► │  │ ThiLAN-thi-lan-student-Setup.exe  │  │
│  │ (QMainWindow)       │ │  cổng 47821     │  │ (QMainWindow)       │  │
│  └──────────┬──────────┘ │                │  └──────────┬──────────┘  │
│             │             │                │             │             │
│  ┌──────────▼──────────┐ │                │  ┌──────────▼──────────┐  │
│  │ ExamServer (TCP)    │ │  Broadcast UDP │  │ Anti-cheat stack     │  │
│  │ • Quản lý phòng    │ │  cổng 47822    │  │ • Keyboard hook      │  │
│  │ • Phát đề mã hoá   │ │ ─────────────► │  │ • USB guard          │  │
│  │ • Thu bài           │ │                │  │ • VM detect          │  │
│  └─────────────────────┘ │                │  │ • Process scan       │  │
└───────────────────────────┘                │  └─────────────────────┘  │
                                            └───────────────────────────┘
```

### Luồng dữ liệu chính

1. **Auto-discovery** (UDP broadcast):
   - Server broadcast `"THILAN_HUB:<port>:<room_code>:<title>"` mỗi 2 giây
   - Client nhận được → hiện phòng trong danh sách "Tìm phòng"

2. **Kết nối** (TCP `localhost` hoặc IP):
   - Client gửi `{"op":"join","room":"ABC123","name":"Nguyễn Văn A"}`
   - Server validate → trả `{"op":"joined","session_id":"..."}`
   - Server broadcast cho cả phòng biết có thành viên mới

3. **Phát đề**:
   - Giáo viên bấm "Phát đề" → server đọc file đề → mã hoá → gửi `{"op":"paper","data":<encrypted_bytes>,"paper_meta":{...}}`
   - Mỗi client nhận → giải mã → render đề

4. **Làm bài**:
   - Client lưu đáp án local mỗi 5 giây + autosave khi mất mạng
   - Server không biết đáp án cho đến khi nộp

5. **Nộp bài**:
   - Client gửi `{"op":"submit","answers":{1:2,2:0,...}}`
   - Server chấm → lưu SQLite → broadcast `{"op":"ranking","rows":[...]}`

---

## 3. Cấu trúc thư mục

```
project/
├── README.md                                ← File này
├── RELEASE.md                               ← Hướng dẫn deploy chi tiết
├── setup.cmd                                ← Setup git + GitHub (chạy 1 lần)
├── deploy.cmd                               ← Deploy day du (bump + build + git + push)
├── build.bat                                ← Script build EXE (gọi bởi deploy.cmd)
├── run-admin.bat                            ← Chạy admin nhanh (dev mode, KHÔNG build)
├── run-student.bat                          ← Chạy student nhanh (dev mode, KHÔNG build)
├── run-both.bat                             ← Chạy cả 2 cùng lúc (test client-server)
│
├── admin/                                   ← App giáo viên
│   ├── main.py                              ← Entry point: `python admin/main.py`
│   ├── assets/
│   │   ├── thi-lan-admin.ico               ← Icon cho EXE + PyInstaller
│   │   └── thi-lan-admin.png               ← Logo hiển thị trong app
│   ├── packaging/
│   │   └── thilan-admin.spec                ← PyInstaller spec
│   └── thi_lan/                             ← Python package
│       ├── __init__.py                      ← __version__
│       ├── core/                            ← Thuật toán, parser, storage
│       │   ├── __init__.py
│       │   ├── answer_marks.py              ← Chấm điểm theo đáp án
│       │   ├── azota_spec.py                ← Chuẩn format Azota
│       │   ├── child_exam.py                ← Đề con cho học sinh (shuffle)
│       │   ├── crypto_util.py               ← Mã hoá AES cho đề
│       │   ├── docx_exam.py                 ← Parser file .docx
│       │   ├── exam_algorithm.py            ← Thuật toán chấm điểm
│       │   ├── exam_source.py               ← Lưu trữ đề gốc
│       │   ├── gemini_answers.py            ← Tạo đáp án bằng Gemini AI
│       │   ├── import_files.py              ← Import đề từ nhiều format
│       │   ├── logging_util.py              ← Logger chuẩn
│       │   ├── paper_zip.py                 ← Nén đề + đáp án thành .zip
│       │   ├── parse_exam.py                ← Parser tổng quát
│       │   ├── pdf_azota.py                 ← Parser PDF Azota
│       │   ├── pdf_exam.py                  ← Parser PDF thường
│       │   ├── protocol.py                  ← Protocol constants + helpers
│       │   ├── roster.py                    ← Quản lý danh sách lớp
│       │   ├── session_persist.py           ← Lưu ca thi (resume khi crash)
│       │   ├── shuffle.py                   ← Xáo trộn câu/đáp án
│       │   ├── store.py                     ← SQLite wrapper
│       │   └── updater.py                   ← Auto-update logic (MỚI v1.1.0)
│       ├── net/                             ← Network layer
│       │   ├── __init__.py
│       │   ├── discovery.py                 ← UDP broadcast discovery
│       │   └── server.py                    ← TCP server + session manager
│       └── ui_qt/                           ← Giao diện PySide6
│           ├── __init__.py
│           ├── admin_window.py              ← Main window: login, dashboard, 5 trang
│           ├── brand.py                     ← App icon, theme colors
│           ├── help_guide.py                ← Trang trợ giúp nội bộ
│           ├── icons.py                     ← Icon helper
│           ├── num_input.py                 ← QLineEdit chỉ nhập số
│           ├── styles.py                    ← QSS stylesheet
│           ├── updater_ui.py                ← Dialog cập nhật (MỚI v1.1.0)
│           ├── widgets.py                   ← Reusable widgets
│           └── pages/                       ← 5 trang chức năng
│               ├── __init__.py
│               ├── bang_diem.py             ← Bảng điểm
│               ├── de_thi.py                ← Đề thi (wizard 3 bước)
│               ├── lop.py                   ← Lớp học
│               ├── ngan_hang.py             ← Ngân hàng câu hỏi
│               └── phat_de.py               ← Phát đề + giám thị
│
└── student/                                 ← App học sinh (mirror cấu trúc admin)
    ├── main.py
    ├── assets/
    │   ├── thi-lan-student.ico
    │   └── thi-lan-student.png
    ├── packaging/
    │   └── thilan-hocsinh.spec
    └── thi_lan/
        ├── __init__.py                      ← __version__ (PHẢI khớp admin)
        ├── anti_cheat/                     ← Chống gian lận (chỉ có ở student)
        │   ├── __init__.py
        │   ├── process_scan.py              ← Kill Chrome, task manager...
        │   ├── usb_guard.py                 ← Block USB
        │   ├── vm_detect.py                 ← Phát hiện máy ảo
        │   └── windows_guard.py             ← Keyboard hook + kiosk mode
        ├── core/                            ← Tương tự admin (subset)
        │   ├── crypto_util.py
        │   ├── logging_util.py
        │   ├── paper_zip.py                 ← Giải nén + giải mã đề
        │   ├── protocol.py
        │   └── updater.py                   ← (MỚI v1.1.0)
        ├── net/
        │   ├── client.py                    ← TCP client
        │   └── discovery.py
        └── ui_qt/
            ├── brand.py
            ├── styles.py
            ├── student_window.py            ← Main window
            ├── updater_ui.py                ← (MỚI v1.1.0)
            └── window_spy.py                ← Debug tool (optional)
```

### Điểm khác biệt admin vs student

| Thư mục | Admin | Student | Ghi chú |
|---------|-------|---------|---------|
| `core/` | 19 file | 4 file | Student chỉ cần parse + chấm điểm |
| `net/` | `server.py` | `client.py` | TCP socket |
| `ui_qt/pages/` | 5 trang | 0 trang | Student là single-window |
| `anti_cheat/` | ❌ | ✅ | Chỉ áp dụng cho máy học sinh |

---

## 4. Công nghệ sử dụng

### Core
- **Python 3.10+** — Ngôn ngữ chính
- **PySide6 6.5+** — Qt6 GUI framework
- **SQLite3** — Embedded database (có sẵn trong Python)

### Network
- **socket** (stdlib) — TCP server/client
- **UDP broadcast** — Auto-discovery

### Xử lý file
- **python-docx** — Đọc file Word (.docx)
- **pdfplumber / PyPDF2** — Đọc file PDF
- **openpyxl** — Xuất bảng điểm Excel

### AI (tuỳ chọn)
- **Google Gemini API** — Tạo đáp án tự động cho câu hỏi

### Build
- **PyInstaller** — Đóng gói thành `.exe`
- **UPX** (optional) — Nén `.exe`

### Auto-update (v1.1.0+)
- **GitHub Releases API** — Check version mới
- **Registry Run key** (Windows) — Apply update khi đăng nhập lại

---

## 5. Workflow tổng thể

### 5.1. Giáo viên tạo đề

```
[Đề Word/PDF] ──import_files──► [JSON thô] ──exam_algorithm──► [Đề chuẩn hoá]
                                                                   │
                                                                   ├─► [Lưu SQLite store.py]
                                                                   ├─► [Gemini tạo đáp án?]
                                                                   └─► [Xáo trộn shuffle.py]
                                                                              │
                                                                              ▼
                                                                  [Đề + key mã hoá crypto_util.py]
                                                                              │
                                                                              ▼
                                                                  [paper_zip.py đóng gói .zip]
```

### 5.2. Giáo viên phát đề

```
Giáo viên bấm "Mở ca thi"
    │
    ▼
Tạo phòng (mã 6 ký tự) → UDP broadcast mỗi 2s
    │
    ▼
Học sinh thấy phòng trong danh sách "Tìm phòng thi"
    │
    ▼
Học sinh nhập tên → TCP join → server broadcast "Nguyễn Văn A đã vào"
    │
    ▼
Giáo viên bấm "Phát đề" → server chọn đề → child_exam.py shuffle
    │
    ▼
Server gửi paper bytes mã hoá cho từng client
    │
    ▼
Client giải mã → render UI → bắt đầu đếm giờ
    │
    ▼
Hết giờ HOẶC học sinh bấm "Nộp" → gửi answers về server
    │
    ▼
Server chấm → lưu SQLite → broadcast ranking
    │
    ▼
Giáo viên xem bảng điểm, xuất Excel
```

### 5.3. Học sinh làm bài

```
App mở → AutoUpdater check (background) ──► [Popup "Có update?"]
    │
    ▼
Client UDP discovery → thấy phòng → nhập tên → join
    │
    ▼
Nhận paper → render UI 4 cột (Azota-style)
    │
    ▼
Anti-cheat bật: chặn Alt+Tab, Win, Ctrl+C, USB...
    │
    ▼
Auto-save answers mỗi 5s vào local (chống crash)
    │
    ▼
Mất mạng? → vẫn làm bài, queue local, gửi khi reconnect
    │
    ▼
Bấm "Nộp bài" → confirm dialog → gửi answers
    │
    ▼
Server trả kết quả + hiện bảng xếp hạng phòng
```

---

## 6. Cài đặt & chạy

### 6.1. Yêu cầu

- Windows 10/11 (đã test, chưa test macOS/Linux)
- Python 3.10 trở lên: https://www.python.org/downloads/
- Cả giáo viên và học sinh **cùng mạng LAN** (cùng subnet, vd 192.168.1.x)

### 6.2. Cài thư viện

Mỗi bên (admin/student) cài riêng:

```powershell
cd C:\Users\X Tien\Desktop\project\admin
python -m pip install -r requirements.txt

cd C:\Users\X Tien\Desktop\project\student
python -m pip install -r requirements.txt
```

### 6.3. Chạy từ source

**Cách nhanh** (khuyến nghị khi dev):
```powershell
# Tai root project, chay 1 trong 3:
.\run-admin.bat       # Chi admin
.\run-student.bat     # Chi student
.\run-both.bat        # Ca 2 cung luc (test client-server)
```

**Cách thủ công**:
```powershell
# Máy giáo viên
cd C:\Users\X Tien\Desktop\project\admin
python main.py

# Máy học sinh
cd C:\Users\X Tien\Desktop\project\student
python main.py
```

### 6.4. Chạy từ EXE

Sau khi build:

```
build\
  ├── ThiLAN-thi-lan-admin-Setup.exe   ← Copy sang máy giáo viên
  └── ThiLAN-thi-lan-student-Setup.exe    ← Copy sang máy học sinh
```

Double-click để chạy. Không cần cài Python.

---

## 7. Build EXE

### 7.1. Build bằng script

```powershell
cd C:\Users\X Tien\Desktop\project
.\build.bat
```

Script tự động:
1. Cài thư viện (PyInstaller)
2. Build `ThiLAN-thi-lan-admin-Setup.exe` từ `admin/packaging/thilan-admin.spec`
3. Build `ThiLAN-thi-lan-student-Setup.exe` từ `student/packaging/thilan-hocsinh.spec`
4. Copy cả 2 vào thư mục `build/`

### 7.2. Build thủ công

```powershell
cd admin
python -m PyInstaller --clean packaging/thilan-admin.spec

cd ..\student
python -m PyInstaller --clean packaging/thilan-hocsinh.spec
```

### 7.3. File spec là gì?

`*.spec` là file config PyInstaller, định nghĩa:
- **Entry point**: `main.py`
- **Resources**: `assets/*.ico`, `assets/*.png`
- **Hidden imports**: `thi_lan.core.updater`, `thi_lan.ui_qt.updater_ui` (PyInstaller không tự detect các module import động)
- **Excludes**: Bỏ Qt3D, QtCharts, QtWebEngine... (giảm 50MB)
- **Output name**: `ThiLAN-thi-lan-admin-Setup` / `ThiLAN-thi-lan-student-Setup`
- **Icon**: `.ico` cho app Windows

### 7.4. Kích thước EXE

| File | Size | Ghi chú |
|------|------|---------|
| `ThiLAN-thi-lan-admin-Setup.exe` | ~80 MB | PySide6 chiếm phần lớn |
| `ThiLAN-thi-lan-student-Setup.exe` | ~75 MB | Nhỏ hơn (không có wizard tạo đề) |

---

## 8. Tự động cập nhật

### 8.1. Cách hoạt động

App tự động check bản mới qua **GitHub Releases API**.

```
App mở (v1.1.0)
    │
    ▼
Sau 30s, AutoUpdater chạy thread nền
    │
    ▼
GET https://api.github.com/repos/<owner>/thi-lan/releases/latest
    │
    ▼
Server trả về: {"tag_name": "v1.2.0", "assets": [...], "body": "changelog..."}
    │
    ▼
So sánh: 1.2.0 > 1.1.0 → có update!
    │
    ▼
Popup hiện: "Có bản cập nhật v1.2.0" + changelog
    │
    ▼
User bấm "Tải về":
  ├─ DownloadDialog mở, progress bar
  ├─ urllib.request tải file .exe về %TEMP%\thi-lan-update\
  └─ schedule_apply_on_exit() ghi Registry Run key
    │
    ▼
User đóng app → Windows apply update khi đăng nhập lại
    │
    ▼
App tự mở lại với version mới
```

### 8.2. File liên quan

| File | Vai trò |
|------|---------|
| `thi_lan/core/updater.py` | Logic check GitHub API, tải file, schedule apply |
| `thi_lan/ui_qt/updater_ui.py` | Dialog popup, progress bar |
| `__init__.py` | Chứa `__version__` — phải khớp tag release |

### 8.3. Cấu hình GitHub repo

Mở file `thi_lan/core/updater.py` (cả admin và student), sửa:

```python
GITHUB_REPO = "your-username/thi-lan"  # ← Sửa chỗ này
APP_NAME = "thi-lan-admin"            # admin
APP_NAME = "thi-lan-student"           # student
```

Biến `APP_NAME` dùng để match asset trên GitHub. Asset phải chứa tên này trong filename, vd:
- `ThiLAN-thi-lan-admin-Setup.exe` chứa `thi-lan-admin` ✓ (sau khi match không phân biệt hoa/thường)
- `ThiLAN-thi-lan-student-Setup.exe` chứa `thi-lan-student` ✓

### 8.4. Auto-update có bắt buộc không?

**Không.** Nếu:
- Không có GitHub repo → `fetch_manifest()` trả `None` → skip silently
- Mất mạng → catch exception → skip
- User tắt popup "Để sau" → vẫn dùng bản hiện tại

App **không bao giờ** block user khỏi dùng app vì lý do update.

---

## 9. Phát hành bản mới

### 9.1. Quy trình nhanh

```powershell
cd C:\Users\X Tien\Desktop\project

# Lần đầu tiên (chạy 1 lần):
.\setup.cmd xtien
# → Tao git repo + ket noi GitHub remote + push

# Mỗi lần phát hành (chỉ 1 lệnh):
.\deploy.cmd 1.2.0 "Them cham diem tu dong"
# → Tự bump version + build + commit + tag + push + mo GitHub
```

Sau đó lên web: **kéo thả 2 file `.exe`** vào Release → **Publish**.

User nhận popup update trong vòng 30s–6h (hoặc khi khởi động lại app).

### 9.2. Chi tiết

**Script làm gì** (`deploy.cmd`):

| Bước | Chi tiết |
|------|----------|
| 1. Bump | Sửa `__version__` trong `admin/__init__.py` + `student/__init__.py` |
| 2. Build | Gọi `build.bat` → tạo 2 file `.exe` |
| 3. Commit | `git add build/` + 2 file `__init__.py` |
| 4. Tag + Push | `git tag v1.2.0` → push code → push tag |
| 5. Mở GitHub | Trình duyệt mở trang Release mới |

**Output sau build**:
```
build\
  ├── ThiLAN-thi-lan-admin-Setup.exe   (~80 MB)
  └── ThiLAN-thi-lan-student-Setup.exe (~75 MB)
```

**Quy tắc tăng version**:
- `1.1.0 → 1.1.1` PATCH — sửa bug nhỏ
- `1.1.0 → 1.2.0` MINOR — thêm tính năng
- `1.2.0 → 2.0.0` MAJOR — breaking change

### 9.3. Rollback

```powershell
# Tao release fix moi
git revert HEAD
git push

# Hoac reset ve tag cu (nguy hiem, mat lich su)
git reset --hard v1.1.0
git push --force

# Roi deploy lai
.\deploy.cmd 1.2.1 "Fix bug"
```

---

## 10. Quy ước code

### 10.1. Naming

- File: `snake_case.py`
- Class: `PascalCase`
- Function/variable: `snake_case`
- Constant: `UPPER_SNAKE`
- Private: prefix `_` (vd `_parse_ver`)

### 10.2. Imports

```python
# 1. Standard library
from __future__ import annotations
import json
import sys
from pathlib import Path

# 2. Third-party
from PySide6.QtCore import Qt, Signal

# 3. Local (relative hoặc absolute)
from thi_lan.core.logging_util import get_logger
```

### 10.3. Logging

```python
from thi_lan.core.logging_util import get_logger
logger = get_logger("thi_lan.module.name")

logger.debug("chi tiết kỹ thuật")
logger.info("sự kiện bình thường")
logger.warning("có vấn đề nhưng không crash")
logger.error("lỗi nghiêm trọng")
```

**Không** dùng `print()` trong code production.

### 10.4. Threading

- Main thread: chỉ xử lý UI
- Background: network I/O, file parsing, AI generation
- Cross-thread UI update: dùng `QTimer.singleShot(0, ...)` hoặc `QMetaObject.invokeMethod()` với `QueuedConnection`

### 10.5. Error handling

```python
try:
    risky_operation()
except SpecificError as exc:
    logger.error("operation failed: %s", exc)
    # User-friendly fallback
    QMessageBox.warning(self, "Lỗi", "Không thể thực hiện thao tác.")
except Exception:
    logger.exception("unexpected")
    # Catch-all để app không crash
```

### 10.6. Type hints

Dùng cho function public:

```python
def fetch_manifest(repo: str | None = None, timeout: float = 10.0) -> Optional[UpdateInfo]:
    ...
```

Private function có thể bỏ qua type hints.

---

## 11. Câu hỏi thường gặp

### App không thấy phòng thi?

**Nguyên nhân**: Tường lửa Windows chặn UDP broadcast.

**Cách sửa**:
1. Mở **Windows Defender Firewall with Advanced Security**
2. **Inbound Rules** → **New Rule** → **Port** → **UDP 47822**
3. Allow the connection → Apply to all profiles
4. Đặt tên: `Thi LAN Discovery`

Hoặc tắt tường lửa tạm thời (chỉ dùng cho dev):
```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

### Học sinh vào phòng nhưng không nhận được đề?

**Nguyên nhân**: TCP bị chặn (cổng 47821).

**Cách sửa**: Tương tự trên, mở **TCP 47821** (Inbound).

### App học sinh báo "Đang chạy trong máy ảo"?

Anti-cheat phát hiện VM (VMware/VirtualBox/Hyper-V). Học sinh cần dùng máy thật. Nếu test trên VM là cố ý, sửa `student/thi_lan/anti_cheat/vm_detect.py` để bypass.

### Build EXE bị lỗi `ModuleNotFoundError: thi_lan.core.updater`?

PyInstaller không detect được module import động. Đã fix trong file `.spec`:

```python
hiddenimports=[
    'thi_lan.core.updater',
    'thi_lan.ui_qt.updater_ui',
],
```

### Làm sao test update mà không cần GitHub thật?

Tạo file `manifest.json` local:

```json
{
  "tag_name": "v1.2.0",
  "body": "Test changelog",
  "assets": [
    {
      "name": "ThiLAN-thi-lan-admin-Setup.exe",
      "browser_download_url": "file:///C:/path/to/ThiLAN-thi-lan-admin-Setup.exe",
      "size": 83886080
    }
  ]
}
```

Sửa tạm `fetch_manifest()` để đọc từ file này trong dev.

### Có thể chạy 2 giáo viên cùng lúc trên 1 mạng LAN không?

**Không nên**. Vì cả 2 sẽ cùng broadcast phòng gây nhiễu. Thiết kế hiện tại giả định 1 giáo viên = 1 phòng thi = 1 ca.

### Học sinh copy đề bằng cách screenshot?

Hiện tại app chỉ chặn phím tắt + USB. Chưa chặn screenshot/print screen. Nếu cần, bổ sung trong `student/thi_lan/anti_cheat/`.

### Có chạy được trên macOS/Linux không?

Phần lớn code là Python nên OK, nhưng:
- `anti_cheat/` chỉ hoạt động trên Windows
- `windows_guard.py` dùng `SetWindowsHookEx` API
- File `.spec` PyInstaller đang build `.exe` cho Windows

Nếu muốn cross-platform, cần viết spec riêng cho macOS (`.app`) và Linux (AppImage).

---

## Giấy phép

Internal use only. © 2026 Thi LAN.

---

## Tác giả / Liên hệ

- **Developer**: XTien
- **Repo**: https://github.com/&lt;username&gt;/thi-lan (private)
- **Email**: (tbd)
