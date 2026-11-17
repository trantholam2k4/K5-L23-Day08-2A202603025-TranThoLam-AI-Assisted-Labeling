# Báo cáo Day 08 — 2A202603025 - Trần Thọ Lâm

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / car / bus / truck | `yolo26-m` | `0.3` | `r01`, `r02`, `r05`: `yolo26-m` chạy nhanh (`0.12s/ảnh` trên RTX 3050) và bắt ổn các lớp phổ thông. Theo `threshold_sweep.md`, các lớp ít mẫu trên Gold (`bicycle`, `bus`, `truck` < 5 vật) hoặc không đạt recall 90% ở mức ≤ 0.5 (`motorbike` max 88% trên 8 vật, `car` max 86% trên 7 vật) nên mình giữ ngưỡng mặc định `0.3`. |
| three_wheeler | `yoloe-s` (prompt: `three_wheeler`) | `0.15` | `r03`, `r06`: Bộ nhãn COCO của `yolo26-m` không có lớp `three_wheeler` nên toàn gọi nhầm xe tuk-tuk thành `car` hoặc `truck`. Mình chạy riêng `yoloe-s` (`0.07s/ảnh`) ở ngưỡng `0.15` để bắt xe ba bánh rồi cho đè lên box trùng của `yolo26-m`. |

- **Lệnh `merge_prelabels.py` đã dùng (dán nguyên):**
  `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json`
- **Phiên bản SAM bạn chọn (SAM 1 / 2.1 / 3) và lý do:** Chọn **`sam2.1_b.pt` (SAM 2.1)** thay vì `mobile_sam.pt` (SAM 1). Test trên cùng ảnh `img_013.jpg` với 3 điểm click giống nhau: `sam2.1_b.pt` ra 1 polygon liền khối (`317,259 px`, `45.4%` ảnh), ôm trọn khung xe và mái che; còn `mobile_sam.pt` bị vỡ làm 2 mảnh rời (`245,482 px`, `35.2%` ảnh) và hụt mất phần mái che.
- **Lỗi nào không sửa được bằng đổi ngưỡng? Sửa thế nào?** Lỗi `yolo26-m` không có lớp `three_wheeler` nên nhận nhầm tuk-tuk thành `car`/`truck` (như ở `img_013.jpg`, `img_004.jpg`). Dù hạ ngưỡng xuống `0.05` (`r05`) thì model vẫn không sinh ra nhãn `three_wheeler` được. Phải dùng thêm detector mở `yoloe-s` với prompt `three_wheeler` rồi gộp lại bằng `merge_prelabels.py`.
- **Box điểm cao mà sai hoặc điểm thấp mà đúng (ảnh nào):**
  - *Điểm cao mà sai:* `yolo26-m` vẽ nhầm box `person` ở chỗ trống không có người tại `img_006.jpg` (box `#94`) và `img_035.jpg` (box `#497`), hoặc nhận nhầm mũ/vật ở xa thành `person` tại `img_031.jpg` (box `#452`).
  - *Điểm thấp mà đúng:* `yoloe-s` bắt trúng mấy xe tuk-tuk bị che một phần ở mức điểm tự tin thấp (`0.15 ≤ conf < 0.3`).
- **Giới hạn:** Chọn ngưỡng trên chính 4 ảnh Gold (`41` đối tượng) nên số đo trên Gold bị lạc quan hơn thực tế. Ngoài ra có vài lớp quá ít mẫu (`bus`: 0 vật, `bicycle` và `truck`: < 5 vật) nên chưa đủ cơ sở để tối ưu ngưỡng riêng.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 497 | 95.4% (497/521) | 521 box máy |
| edited | 8 | 1.5% (8/521) | 521 box máy |
| deleted | 16 | 3.1% (16/521) | 521 box máy |
| added | 13 | 2.5% (13/518) | 518 đối tượng sau duyệt |

*(Lưu ý: Trong `deleted + added` có 1 ca đổi lớp tại chỗ nên số vật máy bỏ sót thật là `13 - 1 = 12` vật).*

- **Lớp nào máy tệ nhất? Vì sao (dẫn ảnh)?**
  - Tính theo tỷ lệ phải sửa/xóa thì **`three_wheeler`** tệ nhất: trong `41` box máy tạo ra (`24` accepted, `5` edited, `12` deleted), phải sửa/xóa tới **`17/41` (`41.5%`)**. Do đặt ngưỡng `0.15` thấp nên `yoloe-s` hay vẽ trùng box hoặc nhận nhầm xe máy chở hàng cồng kềnh thành `three_wheeler` (như box `#17` ở `img_003.jpg`).
  - Tính theo số vật bỏ sót (`added`) thì **`motorbike`** sót nhiều nhất (`7` box vẽ thêm trên `144` vật sau duyệt), chủ yếu là xe ở xa hoặc bị khuất một phần.
- **So với gold:** Precision của máy là **`43.2%` (`32/74`**, KTC 95%: `33–55%`), Recall là **`78.0%` (`32/41`**, KTC 95%: `63–88%`) trên `41` đối tượng Gold (`TP=32, FP=42, FN=9`).

## 3. Thiên kiến tự động hóa và ảnh bẫy

- **Bắt bẫy:** Đạt **`10/12` (`83.3%`**, KTC 95%: `55–95%`, qua ngưỡng `80%`).
  - Chi tiết: `extra`: `3/3`, `wrong_class`: `3/3`, `loose`: `2/3`, `missing`: `2/3`.
  - Hai bẫy bị lọt: **`img_025.jpg` (`missing`)** và **`img_029.jpg` (`loose`)**.
  - Lý do: Lượt duyệt đầu tiên mình chưa tắt hết layer nhãn máy để tự nhìn ảnh trước nên chỉ bắt được `2/12` (`16.7%`). Lúc rà lại thì lên được `10/12`, nhưng vẫn lọt 1 bẫy `missing` ở `img_025.jpg` (do cảnh đông nên nhìn lướt qua) và 1 bẫy `loose` ở `img_029.jpg` (do viền hở không quá rõ).
- **Accepted cao mà bắt bẫy thấp nói gì về cách duyệt?** Tỷ lệ `accepted` tới `95.4%` (`497/521`) nhưng lượt đầu chỉ bắt được `2/12` bẫy (và lúc QC vẫn lọt thêm 5 lỗi) cho thấy mình bị phụ thuộc vào box máy vẽ sẵn: thấy khung bao hiện lên là mặc định tin tưởng, chỉ xóa mấy box thừa quá lộ chứ chưa tự rà vùng trống (nên sót `missing`) và chưa soi kỹ 4 cạnh box (nên sót `loose`).

## 4. Máy có đáng dùng không?

- **Thời gian (từ `submission/time_log.csv`):** Gán tay tập Gold (4 ảnh, 41 vật) hết **`12 / 4 = 3.0` phút/ảnh**; duyệt tập Batch có pre-label (24 ảnh, 527 vật) hết **`30 / 24 = 1.25` phút/ảnh**.
  - *Thiên lệch:* Mình làm Gold trước nên đã quen mắt với bộ nhãn và cảnh phố; hơn nữa mật độ vật thể ở Batch dày hơn Gold gấp đôi (`527/24 ≈ 22.0` vật/ảnh so với `41/4 = 10.25` vật/ảnh), nên nếu gán tay từ đầu cho 24 ảnh Batch chắc chắn sẽ chậm hơn mức `3.0` phút/ảnh nhiều.
- **Kết luận:** **Đáng dùng**, nhất là với các lớp phổ thông như `person` (`268/270` box giữ nguyên), `car` (`55/55`) và `motorbike` (`135/138`) vì đỡ tốn công kéo tay từ đầu. Riêng lớp `three_wheeler` thì chỉ dùng làm gợi ý vị trí, bắt buộc phải soi lại.
- **Ba quy tắc giảm thiên kiến cá nhân:**
  1. **Tắt con mắt ẩn layer nhãn máy trước khi làm:** Mở ảnh mới trên CVAT là ẩn hết box máy, tự đếm người/xe từ trái qua phải trước rồi mới bật lại để đối chiếu (giảm lỗi `missing`).
  2. **Dùng Filter lọc riêng từng lớp hiếm:** Duyệt xong theo thứ tự ảnh thì lọc riêng nhãn `three_wheeler` và `bicycle` để rà lại một lượt, tránh lỗi `yoloe-s` gán nhầm xe máy chở hàng.
  3. **Zoom kỹ các box nhỏ ở vùng tối:** Với mấy box `person` đứng lẻ ở góc tối hoặc nền xa, phóng to xem có đúng dáng đầu–vai không rồi mới giữ (tránh box ma `thua` và kéo sát viền chống `loose`).

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| `img_013.jpg` | `sam2.1_b.pt` (SAM 2.1) | 3 (`2 pos`, `1 neg`) | Còn dính một ít đầu và chân người lái do mới đặt 1 điểm âm ở lưng (`720,300`) | Ra 1 polygon liền (`317,259 px`, `45.4%` ảnh), ôm sát khung tuk-tuk nhưng chưa khoét sạch người lái | `submission/masks/tuktuk_img013_sam2.jpg` |
| `img_013.jpg` | `mobile_sam.pt` (SAM 1) | 3 (`2 pos`, `1 neg`) | Tách được thân xe nhưng bị mất phần mái che | Mask vỡ thành 2 mảnh rời (`245,482 px`, `35.2%` ảnh), hụt khung mái phía trên | `submission/masks/tuktuk_img013_sam1.jpg` |
| `img_004.jpg` | `sam2.1_b.pt` (SAM 2.1) | 3 (`2 pos`, `1 neg`) | Không dính người lái (điểm âm ở `465,450` tách ổn) | Ra 1 polygon gọn (`93,192 px`, `13.3%` ảnh), bám sát thân xe | `submission/masks/tuktuk_img004_sam2.jpg` |
| `img_017.jpg` | `sam2.1_b.pt` (SAM 2.1) | 3 (`2 pos`, `1 neg`) | Không dính người lái (điểm âm ở `645,355` tách sạch người ngồi) | Ra 1 polygon liền (`84,531 px`, `12.1%` ảnh), viền mượt | `submission/masks/tuktuk_img017_sam2.jpg` |

- **Đối chiếu guideline mục 4.2:** Mục 4.2 yêu cầu dùng điểm âm (`--neg`) để tách người lái ra khỏi phương tiện (`motorbike`, `three_wheeler`). Tuy nhiên chưa nói rõ phải đặt ít nhất bao nhiêu điểm âm với xe có mái che kín như tuk-tuk (`img_013.jpg`), làm mình đặt 1 điểm nên vẫn dính đầu và chân người lái; mình đã bổ sung quy tắc đặt tối thiểu 2 điểm âm vào `guideline_v1.1_calibrated.md` (`CAL-04`).

## 6. Calibration (Part 6)

- **Số quyết định:** **`4` quyết định** trong `submission/qc/calibration_decisions.csv` (`CAL-01` đến `CAL-04`).
  - **Quan trọng nhất (`CAL-01`):** Chốt rõ chỉ gán `three_wheeler` khi thấy rõ kết cấu 3 bánh (không gán cho xe máy chở hàng cồng kềnh) và phải kiểm tra dáng người trước khi giữ box `person` để loại box ma (`img_003.jpg`, `img_006.jpg`, `img_035.jpg`).
- **Quy tắc mới đã vào `guideline_v1.1_calibrated.md`:**
  1. Vật đơn lẻ cao `< 20px` ở xa thì bỏ không gán; nếu tụ thành đám đông sát nhau thì gộp chung 1 box `ignore_region` (`CAL-02`).
  2. Đặt tối thiểu 2 điểm âm (`--neg`) lên người lái/khách khi tách mask xe bằng SAM (`CAL-04`).
  3. Chỉ gán `three_wheeler` khi thấy rõ 3 bánh; kiểm tra kỹ vùng tối trước khi giữ box `person` (`CAL-01`).
  4. Tắt hiển thị layer nhãn máy để tự quét mắt toàn ảnh trước khi đối chiếu, kéo sát 4 cạnh box (`CAL-03`).
- **Nguyên nhân gốc:** Do cả lỗi từ model pre-label (`yoloe-s` nhầm xe chở hàng, `yolo26-m` sinh box ma ở góc tối và bắt nhiều vật `< 20px`) lẫn thói quen duyệt hơi ẩu theo khung máy có sẵn.

## 7. QC và quyết định (Part 6)

- **Mẫu:** `5` ảnh (`0` ảnh `can_review` + `5` ảnh ngẫu nhiên `seed 42`: `img_003.jpg`, `img_006.jpg`, `img_014.jpg`, `img_031.jpg`, `img_035.jpg`), tổng `105` đối tượng (từ `submission/qc/qc_sample.csv`).
  - *Hạn chế:* Chính mình vừa duyệt lô vừa tự kiểm QC nên dễ bị quen mắt với những ca biên mà lúc trước mình đã lướt qua.
- **Lỗi theo đối tượng:** **`5/105 = 4.8%`** · **Lỗi theo ảnh:** **`5/5 = 100.0%`** (vượt ngưỡng Release `≤ 2%`, nguồn: `submission/qc/qc_report_auto.md`).
- **Top lỗi (từ `submission/qc/qc_review.csv` và `qc_report_auto.md`):**
  1. `sai_lop` (`2` lỗi): `img_003.jpg` (box `#17` tô nhầm `motorbike` thành `three_wheeler`), `img_031.jpg` (box `#452` nhận nhầm mũ/vật nền thành `person`).
  2. `thua` (`2` lỗi): `img_006.jpg` (box `#94` box ma `person` chỗ không có người), `img_035.jpg` (box `#497` box ma `person` chỗ không có người).
  3. `sai_vi_tri_kich_thuoc` (`1` lỗi): `img_014.jpg` (box `#146` box lệch/chưa ôm sát vật).
- **Giới hạn mẫu nhỏ:** Với `5/5` ảnh lỗi, cận trên 95% (Wilson) lên tới **`100%`**. Kể cả khi mẫu sạch (`0/5`) thì cận trên Wilson 95% vẫn là `43%`, nên mẫu `n=5` quá nhỏ để chứng minh đạt ngưỡng `≤ 2%` về mặt thống kê; kết quả này chủ yếu giúp chỉ ra lỗi lặp để sửa lại lô.
- **Script kiểm được / không kiểm được:**
  - `validate_labels.py` và `qc_report.py` bắt được lỗi định dạng JSON, trùng ID, cảnh báo box nhỏ `< 20px` (báo 11 box trên 6 ảnh) và tính tỷ lệ lỗi từ file CSV.
  - Script không thể tự nhìn ảnh để biết đâu là box ma (`thua`), gán nhầm lớp (`sai_lop`), box hở viền (`sai_vi_tri_kich_thuoc`) hay sót vật (`bo_sot`).

**Quyết định: REWORK**, ngày `2026-11-17`.
- **Lý do (theo cây quyết định Hold → Rework → Release):** Không chọn **Hold** vì ảnh không hỏng và quy tắc đã chốt rõ ở `guideline_v1.1_calibrated.md`. Phải chọn **Rework** vì tỷ lệ ảnh có lỗi ở mẫu QC lên tới **`100.0%` (`5/5` ảnh) > `2%`**, bị lặp lỗi hệ thống (`thua` box ma `person` và `sai_lop` nhầm `three_wheeler`), cộng thêm `11` cảnh báo box `< 20px` từ `validate_labels.py`.
- **Việc cần làm tiếp:**
  1. Sửa ngay 5 box lỗi ghi trong `qc_review.csv` (`#17` ở `img_003.jpg`, `#94` ở `img_006.jpg`, `#146` ở `img_014.jpg`, `#452` ở `img_031.jpg`, `#497` ở `img_035.jpg`).
  2. Dùng `guideline_v1.1_calibrated.md` rà lại toàn bộ 24 ảnh Batch: lọc riêng lớp `three_wheeler` và các box `person` nhỏ ở góc tối, xử lý hết `11` cảnh báo `< 20px` (xóa box lẻ hoặc gom thành `ignore_region`).
  3. Xuất lại bản `release` mới, rút mẫu QC vòng 2 khác seed để kiểm tra lại trước khi chốt Release.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài:** Dùng Gemini để tra cứu lệnh Docker CVAT trên WSL, cú pháp chạy mấy script Python (`merge_prelabels.py`, `inject_traps.py`, `sam2_points.py`, `qc_report.py`), giải thích mã lỗi QC và rà lại số liệu cho khớp giữa các file. Phần gán tay Gold, duyệt 24 ảnh Batch trên CVAT, soi 5 ảnh mẫu QC và chốt số liệu đều tự làm trực tiếp.
- **Gold có bị nhiễm không:** **Không.** Mình gán tay 4 ảnh Gold (`001`, `012`, `020`, `036`) từ đầu trên CVAT trước khi chạy bất kỳ model pre-label nào.
- **Máy / GPU / CVAT dùng; vướng mắc cài đặt:** Chạy trên máy cá nhân Windows + Ubuntu 24.04 (WSL2), GPU **RTX 3050**, Python 3.13 + PyTorch CUDA. Cài CVAT qua Docker local (`http://localhost:8080`). Lúc đầu quên chạy `prepare_workdirs.py` nên không thấy thư mục `work/tasks/`, với lúc export từ CVAT Docker thì phải dùng `docker exec` + `docker cp` từ `/home/django/data/cache/export/` ra Windows.
- **Dùng cache `data/cache/` thay vì tự chạy?** **Không.** Tự chạy hết `yolo26-m`, `yoloe-s`, `threshold_sweep.py` và `sam2.1_b.pt` / `mobile_sam.pt` trên GPU RTX 3050.
- **Lỗi / bug script đã gặp (nếu có):** Script của repo chạy ổn không bug, chỉ phải tải thêm file trọng số `mobile_sam.pt` lúc làm phần so sánh SAM 1 với SAM 2.1.
- **Một điều bạn làm khác lần sau:** Ngay từ đầu lúc duyệt Batch sẽ tắt hẳn layer nhãn máy để tự đếm vật trước rồi mới bật lên đối chiếu, đồng thời dùng bộ lọc theo lớp để rà riêng `three_wheeler` và kiểm tra kích thước `< 20px` trước khi export.