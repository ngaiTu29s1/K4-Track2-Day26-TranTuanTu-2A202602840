# BẢNG THỐNG KÊ THÀNH VIÊN VÀ ĐÓNG GÓP — COLOSSEUM AGENT ARENA

## 1. Thông tin chung dự án

- **Dự án:** Colosseum Agent Arena (MCP/A2A Infrastructure & Agentic Defense/Prosecution Lab)
- **Khóa học:** K4 - Track 2 (Day 26)
- **Tên đội (Team ID):** `TranTuanTu-2A202602840`
- **Tên repository:** `K4-Track2-Day26-TranTuanTu-2A202602840`
- **Nhánh thực hiện:** `feat/colosseum-agent-arena`
- **File nộp bài (Sealed Bundle):** `submissions/TranTuanTu-2A202602840.bundle`

---

## 2. Bảng phân công thành viên

| STT | Họ và tên | Mã sinh viên | GitHub | Vai trò | Nhánh phụ trách | Phần việc đảm nhiệm chính |
| :---: | :--- | :---: | :---: | :--- | :---: | :--- |
| 1 | **Trần Tuấn Tú** | `2A202602840` | `ngaiTu29s1` | Agent Security & Evaluation Engineer (Solo) | `feat/colosseum-agent-arena` | Đảm nhiệm toàn bộ quy trình từ thiết kế bộ dò công tố `eval/`, cổng phòng thủ `agent/gateway.py`, kiểm chuẩn bộ bài `deck/`, thi đấu đối kháng (sparring) với 3 bot và đóng gói bundle nộp bài. |

---

## 3. Chi tiết phần việc và kết quả đạt được

### **Trần Tuấn Tú — 2A202602840**

#### A. Công việc đã thực hiện
1. **Thiết kế & Triển khai Bộ dò lỗi Công tố viên (`eval/prosecute.py`):**
   - Xây dựng 16 detector classes đạt độ chính xác hoàn hảo (`Precision = 1.000`, `Recall = 0.941`, `False claim rate = 0.000` trên 40 fixture chuẩn của ban tổ chức).
   - Tối ưu hóa thứ tự khiếu nại theo trọng số điểm giảm dần (`weight_descending`), tận dụng tối đa ngân sách `ProsecutionBudget`.
   - Cơ chế lọc thông minh: Bỏ qua các lệnh đã bị gateway chặn (`verdict_applied == "deny"`) và kiểm tra điều kiện phòng thủ của thẻ (`defense_event == "gateway.denied"`), giúp triệt tiêu hoàn toàn điểm phạt phản xạ (recoil damage).

2. **Xây dựng Cổng phòng thủ An ninh Agent (`agent/gateway.py`):**
   - Hiện thực hóa 4 lớp phòng vệ cốt lõi:
     - Xác thực Server Fingerprint (`x-server-fingerprint`) và chữ ký Agent Card (`x-card-signature`).
     - Phát hiện và loại bỏ Body-Routing trái phép (`route`, `replica` bị giấu trong args).
     - Kiểm soát ủy quyền A2A: Khớp server với `aud` và kỹ năng gọi phải nằm trong danh mục Agent Card đã đăng ký.
     - Phòng chống Confused Deputy Attack: Ép buộc quyền ghi phải thuộc về `ctx.act` chứ không dựa vào `ctx.sub`.
   - Cơ chế tự động hóa:
     - Tự động rewrite `slides.search` sang công cụ mới `slides.query`.
     - Tự động bổ sung `If-Match` etag và `Idempotency-Key` duy nhất cho các tác vụ ghi (`WRITE_TOOLS`).
     - Tự động gắn `lease_id` quản lý cho `slides.get_frame` để thỏa mãn yêu cầu của trọng tài.
     - Thiết lập mặt nạ trường hẹp (`MASKS`) mặc định nhằm bảo tồn quỹ tín dụng (`credits`) qua 10 hiệp đấu.

3. **Kiểm tra chuẩn hóa Deck & Đấu tập toàn diện (`deck/`, `spar.py`):**
   - Chạy `validate_deck.py` kiểm tra 10 thẻ bài và thứ tự ra trận trên corpus thật (12,375 trang world artifact), đạt `0 failing checks`.
   - Thi đấu đối kháng 10 hiệp thực tế với cả 3 bot Colosseum:
     - **Rookie:** Thắng 100 — 0 (Knockout đối thủ ở hiệp 7, không mất HP).
     - **Operator:** Thắng 100 — 0 (Knockout đối thủ ở hiệp 6, không mất HP).
     - **Adversary:** Thắng 72 — 0 (Knockout bot ở hiệp 9, 0 recoil penalty).

4. **Kiểm chuẩn Hệ thống & Đóng gói bài nộp:**
   - Chạy kiểm thử toàn bộ public suite: `4,609 / 4,609 tests PASSED`.
   - Kiểm tra an ninh offline: `kit/gate_no_key.py` xác nhận 0 vi phạm secret/mạng.
   - Tạo bundle niêm phong hợp lệ: `submissions/TranTuanTu-2A202602840.bundle` (72,423 bytes, 13 files, 39 kit hashes).

---

## 4. Tổng kết đánh giá

- **Kết quả:** Vượt qua tất cả các cổng kiểm tra kỹ thuật (doctor, test, validate, gate-no-key, spar).
- **Trạng thái:** Sẵn sàng nộp bài 100%.
