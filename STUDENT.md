# Thông tin học viên

- Họ và tên: Trần Thọ Lâm
- MSHV: 2A202603025
- Lớp (L2 / L3 / khác): L2
- Tài khoản GitHub: trantholam2k4
- Repo bài nộp (URL): https://github.com/trantholam2k4/K5-L23-TranThoLam-2A202603025-Day08-AI-Assisted-Labeling
- Commit chốt bài (hash): "Complete Day 08 AI-Assisted Labeling lab - 2A202603025 TranThoLam"
- Máy: window 11, RTX3050
- CVAT (Docker): v2.74.0

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 497 (95.4%) / 8 (1.5%) / 16 (3.1%) (mẫu số: 521 box máy); added: 13 (2.5%) (mẫu số: 518 đối tượng sau duyệt)
- So với gold: precision / recall của **máy**: 43.2% (32/74) / 78.0% (32/41 đối tượng gold)
- Bắt bẫy: 10/12 = 83.3%
- Thời gian: gán tay 3.0 phút/ảnh; duyệt có pre-label 1.25 phút/ảnh
- Lỗi QC: 100.0% theo ảnh (5/5 ảnh mẫu), 4.8% theo đối tượng (5/105 đối tượng)
- **Quyết định:** Rework — lý do một dòng: Tỷ lệ ảnh có lỗi trong mẫu QC đạt 100.0% (5/5 > ngưỡng 2%) kèm lỗi lặp hệ thống (box ma `person` và gán nhầm `three_wheeler`), cần sửa lại lô theo `guideline_v1.1_calibrated.md` trước khi phát hành.