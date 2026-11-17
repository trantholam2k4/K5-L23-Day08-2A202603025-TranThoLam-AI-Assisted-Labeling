# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202603025 - Trần Thọ Lâm

> **Chỉ viết phần thay đổi so với v1.0** (không copy cả file). Chốt các câu hỏi mở ở mục 6 của v1.0 (hoặc nói rõ câu nào chưa chốt, vì sao)
> và thêm quy tắc mới rút ra từ lỗi của máy / lỗi của bạn. Mỗi ca khó **có ảnh ví dụ** (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 1 | Vật thể nhỏ ở nền xa có chiều cao dưới 20px (từ 14.3px đến 18.6px) có gán nhãn cá thể hay bỏ qua? | Không gán nhãn cá thể cho đối tượng đơn lẻ cao `< 20px` (xóa bỏ để tránh nhiễu); nếu nhiều người/xe tụ thành đám đông sát nhau ở nền xa thì gộp chung thành 1 box `ignore_region`. | `img_018.jpg`, `img_027.jpg` | `CAL-02` |
| 2 | Khi tách mask bằng điểm (SAM 2.1) cho phương tiện có người ngồi trên (`three_wheeler`, `motorbike`), người lái có nằm trong mask xe không? | Không. Mask phương tiện chỉ bao gồm khung xe, mái che và bánh xe; người lái và hành khách phải được tách riêng bằng tối thiểu 2 điểm âm (`--neg`) đặt lên vùng đầu và thân người. | `img_013.jpg`, `img_017.jpg` | `CAL-04` |

*(Lưu ý: Các câu hỏi mở khác trong mục 6 của v1.0 như phân biệt xe đạp điện và xe máy tay ga khi không thấy bàn đạp chưa gặp ca biên rõ rệt trong 28 ảnh đã gán nên giữ nguyên quy tắc mặc định của v1.0).*

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| `img_003.jpg` | Xe máy chở hàng cồng kềnh phía sau dễ bị nhìn nhầm thành xe ba bánh | `yoloe-s` (ngưỡng 0.15) gán nhầm thành `three_wheeler` (box `#17` bị lọt qua review, phát hiện lúc QC) | Sửa về lớp `motorbike` | **Mục 2 (Định nghĩa lớp):** Chỉ gán `three_wheeler` khi thấy rõ kết cấu 3 bánh (tuk-tuk, xe ba gác, xích lô); xe máy 2 bánh chở thùng hàng lớn vẫn là `motorbike` (`CAL-01`). |
| `img_006.jpg`, `img_035.jpg`, `img_031.jpg` | Đốm sáng/bóng đổ trên đường hoặc mũ bảo hiểm/vật thể nền ở xa có hình dáng giống đầu người | `yolo26-m` sinh box giả `person` tại vùng trống (box `#94`, `#497`) hoặc gán nhầm mũ/vật nền thành `person` (box `#452`) | Xóa bỏ các box ma (`thua`) và đổi đúng lớp cho vật thể nền (`sai_lop`) | **Mục 3 (Chống dương tính giả):** Không giữ box `person` nếu không xác định được cấu trúc thân người tối thiểu (đầu + vai/thân); phóng to kiểm tra vùng nền tối trước khi `accept` (`CAL-01`). |
| `img_025.jpg`, `img_029.jpg` | Người/xe bị che khuất một phần ở trung cảnh và box có viền kéo rộng hở nền nhiều | Pre-label có cài bẫy `missing` (`img_025.jpg`) và `loose` (`img_029.jpg`) bị lọt qua vòng review đầu do thiên kiến tự động hóa (tỷ lệ `accepted` 95.4%) | Bổ sung bước ẩn toàn bộ layer nhãn máy để tự đếm vật trước khi bật lại và kéo sát 4 cạnh viền box | **Mục 5 (Quy trình duyệt QC):** Bắt buộc tắt hiển thị nhãn máy, quét mắt toàn ảnh từ trái qua phải trước khi bật lại đối chiếu; mọi box phải ôm khít phần nhìn thấy, không chừa khoảng trống nền (`CAL-03`). |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Chốt quy tắc xử lý vật nhỏ `< 20px` (`ignore_region`), quy tắc tách người lái khi tạo mask SAM, bổ sung tiêu chí phân biệt `three_wheeler` vs `motorbike` chở hàng, siết quy trình ẩn layer nhãn máy để chống box ma và bẫy `missing`/`loose`. | Rút ra từ 11 cảnh báo kích thước ở `img_018.jpg` (`CAL-02`), ca tách mask `img_013.jpg` (`CAL-04`), 5 lỗi phát hiện qua QC ở `img_003/006/014/031/035` (`CAL-01`) và 2 bẫy bị lọt ở `img_025/029` (`CAL-03`). |