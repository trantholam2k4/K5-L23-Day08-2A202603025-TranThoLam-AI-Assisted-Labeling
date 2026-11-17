# Báo cáo Day 08 — 2A202603025 - Trần Thọ Lâm

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / car / bus / truck | `yolo26-m` | `0.3` (ngưỡng mặc định chung khi gộp) | `r01`, `r02`, `r05`: `yolo26-m` nhận diện tốt các lớp COCO phổ biến ở tiền cảnh và trung cảnh với tốc độ `0.12s/ảnh` trên GPU RTX 3050. Qua `threshold_sweep.md`, các lớp có ít mẫu trên tập Gold (`bicycle`, `bus`, `truck` < 5 vật) hoặc không đạt recall 90% ở mức ≤ 0.5 (`motorbike` tối đa 88% trên 8 vật, `car` tối đa 86% trên 7 vật) nên giữ nguyên ngưỡng mặc định `0.3`. |
| three_wheeler | `yoloe-s` (prompt: `three_wheeler`) | `0.15` | `r03`, `r06`: `yolo26-m` không có lớp `three_wheeler` trong bộ nhãn COCO gốc nên luôn gán nhầm tuk-tuk/xe ba gác thành `car` hoặc `truck`. Dùng `yoloe-s` chạy riêng cho lớp `three_wheeler` (`0.07s/ảnh`) và chọn ngưỡng `0.15` từ `threshold_sweep.md` giúp bắt được phương tiện 3 bánh đặc thù, sau đó ưu tiên đè lên box trùng của `yolo26-m`. |

- **Lệnh `merge_prelabels.py` đã dùng (dán nguyên):**
  `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --thr "three_wheeler=0.15" --conf 0.3 --iou 0.6 --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json`
- **Phiên bản SAM bạn chọn (SAM 1 / 2.1 / 3) và lý do:** Chọn **`sam2.1_b.pt` (SAM 2.1 Base)** thay vì `mobile_sam.pt` (SAM 1). Trên cùng ảnh `img_013.jpg` và cùng tọa độ 3 điểm click, `sam2.1_b.pt` tạo ra **1 polygon liền mạch** phủ `317,259 px` (`45.4%` diện tích ảnh), ôm trọn khung xe tuk-tuk và phần mái che; trong khi `mobile_sam.pt` bị vỡ thành **2 polygon rời rạc** (`245,482 px`, `35.2%` diện tích ảnh) và hụt mất phần khung mái che phía trên.
- **Lỗi nào không sửa được bằng đổi ngưỡng? Sửa thế nào?** Lỗi lệch miền từ vựng (domain shift) khi `yolo26-m` gọi nhầm xe tuk-tuk/ba gác là `car` hoặc `truck` (ví dụ ở `img_013.jpg` và `img_004.jpg`). Dù hạ ngưỡng xuống `0.05` (`r05`), `yolo26-m` vẫn không thể sinh nhãn `three_wheeler`. Cách khắc phục là chạy bổ sung detector từ vựng mở `yoloe-s` với prompt `three_wheeler` rồi gộp với `yolo26-m` bằng `merge_prelabels.py` (ưu tiên đè box trùng ở ngưỡng IoU `0.6`).
- **Box điểm cao mà sai hoặc điểm thấp mà đúng (ảnh nào):**
  - *Điểm cao mà sai:* `yolo26-m` sinh box `person` với độ tin cậy vượt ngưỡng `0.3` tại vùng trống không có người ở `img_006.jpg` (box `#94`) và `img_035.jpg` (box `#497`), hoặc nhận nhầm mũ/vật thể nền thành `person` ở `img_031.jpg` (box `#452`).
  - *Điểm thấp mà đúng:* `yoloe-s` bắt đúng xe tuk-tuk ở mức điểm tự tin thấp (`0.15 ≤ conf < 0.3`), giúp giữ lại các xe ba bánh bị che khuất một phần khi hạ ngưỡng lớp `three_wheeler` xuống `0.15`.
- **Giới hạn:** Việc chọn ngưỡng qua `threshold_sweep.py` được thực hiện trên chính 4 ảnh Gold (`41` đối tượng, in-sample) nên chỉ số của máy trên tập Gold mang tính lạc quan, đồng thời số lượng mẫu của một số lớp quá ít (`bus`: 0 vật, `bicycle` và `truck`: < 5 vật) chưa đủ ý nghĩa thống kê để tối ưu ngưỡng riêng.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 497 | 95.4% (497/521) | 521 box máy |
| edited | 8 | 1.5% (8/521) | 521 box máy |
| deleted | 16 | 3.1% (16/521) | 521 box máy |
| added | 13 | 2.5% (13/518) | 518 đối tượng sau duyệt |

*(Lưu ý: Trong tổng số `deleted + added`, có 1 trường hợp là đổi lớp tại chỗ nên số vật thể máy bỏ sót thực tế là `13 - 1 = 12` vật).*

- **Lớp nào máy tệ nhất? Vì sao (dẫn ảnh)?**
  - Xét theo tỷ lệ box máy bị sửa/xóa, **`three_wheeler`** tệ nhất: trong `41` box máy tạo ra (`24` accepted, `5` edited, `12` deleted), có tới **`17/41` (`41.5%`)** phải chỉnh sửa hoặc xóa bỏ do `yoloe-s` ở ngưỡng `0.15` thường sinh box trùng lặp hoặc gán nhầm xe máy chở hàng cồng kềnh thành `three_wheeler` (như ca box `#17` ở `img_003.jpg`).
  - Xét theo số lượng bỏ sót (`added`), **`motorbike`** có nhiều vật bị máy sót nhất (`7` box vẽ thêm trên tổng `144` đối tượng sau duyệt), chủ yếu do xe máy bị che khuất một phần hoặc kích thước nhỏ ở trung cảnh.
- **So với gold:** Precision của máy là **`43.2%` (`32/74`**, KTC 95%: `33–55%`), Recall là **`78.0%` (`32/41`**, KTC 95%: `63–88%`) trên mẫu số `41` đối tượng Gold (`TP=32, FP=42, FN=9`).

## 3. Thiên kiến tự động hóa và ảnh bẫy

- **Bắt bẫy:** Đạt **`10/12` (`83.3%`**, KTC 95%: `55–95%`, vượt ngưỡng đạt `80%`).
  - Chi tiết theo loại: `extra`: `3/3`, `wrong_class`: `3/3`, `loose`: `2/3`, `missing`: `2/3`.
  - Hai bẫy bị lọt: **`img_025.jpg` (`missing`)** và **`img_029.jpg` (`loose`)**.
  - Nguyên nhân: Ở vòng duyệt đầu tiên trên CVAT, do chưa thực hiện triệt để bước ẩn toàn bộ layer nhãn máy để tự quét ảnh trước, tỷ lệ bắt bẫy ban đầu chỉ đạt `2/12` (`16.7%`). Sau khi siết lại quy trình rà từng ảnh, tỷ lệ tăng lên `10/12`, nhưng vẫn lọt 1 ca `missing` ở `img_025.jpg` (vật thể nằm ở vùng đông chi tiết nên mắt quét qua nhanh) và 1 ca `loose` ở `img_029.jpg` (độ hở viền box không quá lộ liễu nên bị bỏ qua).
- **Accepted cao mà bắt bẫy thấp nói gì về cách duyệt?** Tỷ lệ `accepted` lên tới `95.4%` (`497/521`) trong khi vòng duyệt đầu chỉ bắt được `2/12` bẫy (và kể cả sau khi rà lại vẫn còn sót lỗi ở 5 ảnh mẫu QC) cho thấy hiện tượng **thiên kiến tự động hóa (automation bias)** rõ rệt: người duyệt có xu hướng tin tưởng mặc định vào các khung bao đã có sẵn trên màn hình, chỉ tập trung xóa các box thừa quá lộ hoặc sửa tên lớp sai mà thiếu thao tác tự quét vùng trống (dẫn tới sót `missing`) và kiểm tra độ khít 4 cạnh từng box (dẫn tới sót `loose`).

## 4. Máy có đáng dùng không?

- **Thời gian (từ `submission/time_log.csv`):** Gán tay tập Gold (4 ảnh, 41 đối tượng) mất **`[điền tổng phút gold] / 4 = [điền phút/ảnh]` phút/ảnh**; duyệt tập Batch có pre-label (24 ảnh, 527 đối tượng) mất **`[điền tổng phút batch] / 24 = [điền phút/ảnh]` phút/ảnh**.
  - *Thiên lệch cần lưu ý:* Tập Gold được gán tay trước nên đã làm quen với taxonomy và bối cảnh phố châu Á trước khi duyệt Batch; mặt khác, mật độ vật thể trung bình của Batch cao hơn Gold (`527/24 ≈ 22.0` vật/ảnh so với `41/4 = 10.25` vật/ảnh), nên nếu gán tay hoàn toàn từ đầu cho 24 ảnh Batch sẽ tốn thời gian hơn nhiều so với tốc độ trung bình trên 4 ảnh Gold.
- **Kết luận:** **Có đáng dùng**, đặc biệt hiệu quả cho các lớp phổ biến có tỷ lệ `accepted` cao như `person` (`268/270` box máy giữ nguyên), `car` (`55/55`) và `motorbike` (`135/138`), giúp tiết kiệm phần lớn thao tác kéo khung từ đầu. Tuy nhiên với lớp đặc thù `three_wheeler`, pre-label chỉ đóng vai trò gợi ý vị trí và bắt buộc phải kiểm tra lại bằng mắt.
- **Ba quy tắc giảm thiên kiến cá nhân:**
  1. **Ẩn layer nhãn máy trước khi nhìn ảnh:** Khi mở từng ảnh mới trong CVAT, bấm tắt biểu tượng con mắt trên bảng `Objects` để tự đếm số phương tiện và người đi bộ từ trái sang phải trước khi bật lại nhãn máy đối chiếu (giảm lỗi `missing`).
  2. **Lọc kiểm tra riêng theo từng lớp (Filter by class):** Sau khi duyệt xong theo thứ tự ảnh, dùng bộ lọc của CVAT lọc riêng toàn bộ nhãn `three_wheeler` và `bicycle` để rà lại một lượt tất cả các box được gán lớp hiếm (loại bỏ lỗi `yoloe-s` gán nhầm xe máy chở hàng).
  3. **Phóng to kiểm tra viền và vùng tối đối với các box nhỏ/mờ:** Với các box `person` đứng đơn lẻ ở vùng nền tối hoặc khuất bóng, phóng to kiểm tra cấu trúc đầu–vai trước khi nhấn `Accept` để loại bỏ box ma (`thua`) và kéo sát 4 cạnh viền chống lỗi `loose`.

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| `img_013.jpg` | `sam2.1_b.pt` (SAM 2.1) | 3 (`2 pos`, `1 neg`) | Còn dính một phần đầu và chân người lái do chỉ đặt 1 điểm âm ở lưng (`720,300`) | Tạo 1 polygon liền mạch (`317,259 px`, `45.4%` ảnh), ôm sát khung xe tuk-tuk nhưng chưa khoét tách hoàn toàn người lái | `submission/masks/tuktuk_img013_sam2.jpg` |
| `img_013.jpg` | `mobile_sam.pt` (SAM 1) | 3 (`2 pos`, `1 neg`) | Tách được thân xe chính nhưng bị vỡ vùng mái che | Mask bị chia thành 2 polygon rời rạc (`245,482 px`, `35.2%` ảnh), mất phần khung mái che phía trên | `submission/masks/tuktuk_img013_sam1.jpg` |
| `img_004.jpg` | `sam2.1_b.pt` (SAM 2.1) | 3 (`2 pos`, `1 neg`) | Không gồm người lái (điểm âm ở `465,450` tách tốt vùng người) | Tạo 1 polygon gọn (`93,192 px`, `13.3%` ảnh), viền bám sát thân xe | `submission/masks/tuktuk_img004_sam2.jpg` |
| `img_017.jpg` | `sam2.1_b.pt` (SAM 2.1) | 3 (`2 pos`, `1 neg`) | Không gồm người lái (điểm âm ở `645,355` khoét tách vùng người ngồi) | Tạo 1 polygon liền mạch (`84,531 px`, `12.1%` ảnh), biên mượt | `submission/masks/tuktuk_img017_sam2.jpg` |

- **Đối chiếu guideline mục 4.2:** Quy tắc yêu cầu tách riêng người lái ra khỏi phương tiện (`motorbike`, `three_wheeler`) bằng điểm âm (`--neg`) giúp xác định rõ phạm vi mask xe. Tuy nhiên, guideline mục 4.2 chưa quy định cụ thể số lượng điểm âm tối thiểu khi người lái ngồi lọt bên trong khung xe có mái che (như tuk-tuk ở `img_013.jpg`), dẫn đến việc dùng 1 điểm âm chưa tách hết phần đầu và chân người lái; nội dung này đã được bổ sung vào `guideline_v1.1_calibrated.md` (`CAL-04`).

## 6. Calibration (Part 6)

- **Số quyết định:** **`4` quyết định** trong `submission/qc/calibration_decisions.csv` (`CAL-01` đến `CAL-04`).
  - **Quan trọng nhất (`CAL-01`):** Quy định tiêu chí phân biệt chặt chẽ giữa `three_wheeler` (bắt buộc thấy kết cấu 3 bánh) và `motorbike` chở hàng cồng kềnh, kết hợp kiểm tra cấu trúc thân người để loại bỏ box ma `person` phát sinh từ pre-label (`img_003.jpg`, `img_006.jpg`, `img_035.jpg`).
- **Quy tắc mới đã vào `guideline_v1.1_calibrated.md`:**
  1. Không gán nhãn cá thể cho đối tượng đơn lẻ cao `< 20px` ở nền xa; chỉ gộp chung thành `ignore_region` nếu tụ thành đám đông sát nhau (`CAL-02`).
  2. Tách riêng người lái và hành khách bằng tối thiểu 2 điểm âm (`--neg`) khi tạo mask phương tiện bằng SAM (`CAL-04`).
  3. Chỉ gán `three_wheeler` khi nhận diện rõ cấu trúc 3 bánh; kiểm tra cấu trúc đầu–vai trước khi giữ lại box `person` ở vùng tối (`CAL-01`).
  4. Bắt buộc tắt hiển thị layer nhãn máy để quét mắt toàn ảnh từ trái qua phải trước khi đối chiếu, và kéo sát 4 cạnh viền từng box (`CAL-03`).
- **Nguyên nhân gốc:** Kết hợp của **lỗi hệ thống từ model pre-label** (`yoloe-s` gán nhầm xe chở hàng, `yolo26-m` sinh box ma ở vùng nền tối và bắt nhiều vật `< 20px`) và **thiên kiến tự động hóa khi duyệt tay** (chưa áp dụng triệt để quy trình ẩn nhãn máy và rà độ khít viền box).

## 7. QC và quyết định (Part 6)

- **Mẫu:** `5` ảnh (`0` ảnh `can_review` + `5` ảnh ngẫu nhiên với `seed 42`: `img_003.jpg`, `img_006.jpg`, `img_014.jpg`, `img_031.jpg`, `img_035.jpg`), tổng cộng `105` đối tượng (từ `submission/qc/qc_sample.csv`).
  - *Hạn chế:* Người kiểm QC cũng chính là người trực tiếp duyệt lô `batch-review`, nên khó tránh khỏi điểm mù nhận thức ở các ca biên mà bản thân đã bỏ qua trước đó.
- **Lỗi theo đối tượng:** **`5/105 = 4.8%`** · **Lỗi theo ảnh:** **`5/5 = 100.0%`** (vượt xa ngưỡng Release `≤ 2%`, nguồn: `submission/qc/qc_report_auto.md`).
- **Top lỗi (từ `submission/qc/qc_review.csv` và `qc_report_auto.md`):**
  1. `sai_lop` (`2` lỗi): `img_003.jpg` (box `#17` tô nhầm `motorbike` thành `three_wheeler`), `img_031.jpg` (box `#452` gán nhầm mũ/vật thể nền thành `person`).
  2. `thua` (`2` lỗi): `img_006.jpg` (box `#94` box ma `person` tại vị trí không có người), `img_035.jpg` (box `#497` box ma `person` tại vị trí không có người).
  3. `sai_vi_tri_kich_thuoc` (`1` lỗi): `img_014.jpg` (box `#146` lệch vị trí/kích thước chưa ôm sát vật thể).
- **Giới hạn mẫu nhỏ:** Với `5` ảnh lỗi trên `5` ảnh mẫu, khoảng tin cậy 95% (Wilson) có cận trên lên tới **`100%`**. Ngay cả khi mẫu sạch lỗi (`0/5`), cận trên Wilson 95% vẫn là `43%`, do đó kích thước mẫu `n=5` quá nhỏ để chứng minh đạt ngưỡng `≤ 2%` về mặt thống kê; kết quả này chỉ đủ cơ sở để phát hiện lỗi hệ thống và yêu cầu sửa lại lô.
- **Script kiểm được / không kiểm được:**
  - `validate_labels.py` và `qc_report.py` kiểm tự động được cấu trúc JSON, ID hợp lệ, ngưỡng kích thước tối thiểu (`< 20px` — đã cảnh báo 11 box trên 6 ảnh) và tổng hợp tỷ lệ lỗi từ phiếu kiểm.
  - Tuy nhiên, script không thể tự phát hiện box ma (`thua`), gán nhầm lớp (`sai_lop`), độ ôm sát viền box (`sai_vi_tri_kich_thuoc`) hay vật bị bỏ sót (`bo_sot`) nếu không có con người soi trực tiếp từng ảnh bằng mắt.

**Quyết định: REWORK**, ngày `2026-11-17`.
- **Lý do (theo cây quyết định Hold → Rework → Release):** Không rơi vào **Hold** vì bộ dữ liệu ảnh không hỏng và các vướng mắc về tiêu chí đã được làm rõ trong `guideline_v1.1_calibrated.md`. Tuy nhiên, bắt buộc chọn **Rework** thay vì Release vì tỷ lệ ảnh có lỗi trong mẫu QC lên tới **`100.0%` (`5/5` ảnh) > ngưỡng `2%`**, đồng thời xuất hiện lỗi lặp có tính hệ thống (`thua` do box ma `person` và `sai_lop` nhầm `motorbike` thành `three_wheeler`), kèm theo `11` cảnh báo box kích thước `< 20px` trong `validate_labels.py`.
- **Việc cần làm tiếp (kế hoạch Rework):**
  1. Sửa ngay 5 lỗi cụ thể đã ghi nhận trong `qc_review.csv` (`#17` ở `img_003.jpg`, `#94` ở `img_006.jpg`, `#146` ở `img_014.jpg`, `#452` ở `img_031.jpg`, `#497` ở `img_035.jpg`).
  2. Áp dụng `guideline_v1.1_calibrated.md` để rà lại toàn bộ 24 ảnh trong `batch-review`: dùng bộ lọc lớp kiểm tra toàn bộ nhãn `three_wheeler` và các box `person` kích thước nhỏ ở nền tối, đồng thời xử lý dứt điểm `11` cảnh báo `< 20px` (xóa box đơn lẻ hoặc gom thành `ignore_region`).
  3. Xuất lại bản `release` mới, rút mẫu QC vòng 2 độc lập (khác seed) và chỉ chuyển sang **Release** khi tỷ lệ ảnh lỗi đạt `≤ 2%` và không còn lỗi lặp hệ thống.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài:** Sử dụng trợ lý AI (Gemini) để tra cứu lệnh vận hành CVAT Docker trên WSL/Windows, cấu trúc lệnh chạy các script Python (`merge_prelabels.py`, `inject_traps.py`, `sam2_points.py`, `qc_report.py`), giải thích các mã lỗi QC và rà soát tính nhất quán số liệu giữa các file báo cáo. Toàn bộ việc gán nhãn tập Gold, duyệt 24 ảnh Batch trên CVAT, kiểm tra 5 ảnh mẫu QC bằng mắt và đối chiếu số liệu từ terminal đều tự thực hiện và kiểm chứng trực tiếp.
- **Gold có bị nhiễm không:** **Không.** Tập Gold (4 ảnh `001`, `012`, `020`, `036`) được gán tay hoàn toàn từ đầu trên CVAT trước khi chạy bất kỳ mô hình pre-label nào (`yolo26`, `yoloe`, `sam2.1`) và không sử dụng công cụ AI hỗ trợ gán nhãn tự động trong lúc làm Gold.
- **Máy / GPU / CVAT dùng; vướng mắc cài đặt:** Sử dụng máy tính cá nhân chạy Windows kết hợp Ubuntu 24.04 (WSL2), GPU NVIDIA GeForce **RTX 3050** với môi trường Python 3.13 và PyTorch hỗ trợ CUDA (`+cu12x`). Cài đặt CVAT chạy qua Docker local tại `http://localhost:8080`. Vướng mắc gặp phải: lúc đầu thư mục `work/tasks/` chưa được tạo do chưa chạy `prepare_workdirs.py`, và việc lấy file `.zip` kết quả export từ CVAT Docker cần dùng lệnh `docker exec` kết hợp `docker cp` từ `/home/django/data/cache/export/` sang thư mục làm việc trên Windows.
- **Dùng cache `data/cache/` thay vì tự chạy?** **Không.** Toàn bộ quá trình chạy suy luận `yolo26-m`, `yoloe-s`, quét ngưỡng `threshold_sweep.py`, gộp nhãn và tách mask `sam2.1_b.pt` / `mobile_sam.pt` đều được chạy trực tiếp trên GPU RTX 3050 cục bộ.
- **Lỗi / bug script đã gặp (nếu có):** Không gặp lỗi bug từ mã nguồn script của bài lab; chỉ gặp lưu ý về quy trình thao tác khi lúc đầu quên chạy `prepare_workdirs.py` trước khi tìm ảnh trong `work/tasks/review/` và cần tải bổ sung trọng số `mobile_sam.pt` khi so sánh SAM 1 với SAM 2.1.
- **Một điều bạn làm khác lần sau:** Ngay từ lượt duyệt `batch-review` đầu tiên trên CVAT, sẽ tắt toàn bộ hiển thị layer nhãn máy để tự quét mắt đếm vật thể trên từng ảnh trước, đồng thời dùng bộ lọc theo lớp (`Filter`) rà riêng các lớp hiếm (`three_wheeler`, `bicycle`) và kiểm tra chiều cao box `< 20px` trước khi export, thay vì duyệt tuần tự trên nền nhãn máy có sẵn gây thiên kiến tự động hóa.