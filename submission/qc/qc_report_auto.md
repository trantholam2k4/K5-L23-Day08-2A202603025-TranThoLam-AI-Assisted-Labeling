# Báo cáo QC tự động (nháp)

> **Gợi ý** do script tính; bạn xác nhận. Script chỉ đếm theo phiếu kiểm bằng mắt người.

## Mẫu
- 5 ảnh (0 ca khó + 5 ngẫu nhiên), 105 đối tượng.
- taxonomy v1.0

## Số đo
| Chỉ số | Giá trị | Mẫu số |
|---|---|---|
| Lỗi **theo đối tượng** | 5/105 = 4.8% | 105 đối tượng trong mẫu |
| **Ảnh có lỗi** (dùng cho ngưỡng) | 5/5 = 100.0% | 5 ảnh trong mẫu |
| Máy: accepted / edited / deleted | 497/521 = 95.4% / 8/521 = 1.5% / 16/521 = 3.1% | 521 box máy |
| Máy: added | 13/518 = 2.5% | 518 đối tượng sau duyệt |
| Bắt lỗi bẫy | 10/12 = 83.3% | 12 bẫy |

## Top 3 loại lỗi
| Loại lỗi | Số lỗi | Ảnh ví dụ |
|---|---|---|
| sai_lop | 2 | img_003.jpg, img_031.jpg |
| thua | 2 | img_006.jpg, img_035.jpg |
| sai_vi_tri_kich_thuoc | 1 | img_014.jpg |

## Giới hạn thống kê
- 5 ảnh lỗi trong 5 ảnh mẫu: cận trên 95% (Wilson) của tỉ lệ ảnh lỗi thật là **100%**. Ngưỡng 2% không thể được **chứng minh** bằng mẫu nhỏ như vậy: hãy nói rõ giới hạn này trong báo cáo.

## Gợi ý quyết định
**REWORK**

- ảnh có lỗi 100.0% > 2%

Cây quyết định: Hold (guideline chưa rõ / dữ liệu hỏng) -> Rework (> ngưỡng, lỗi lặp, bắt bẫy / nhất quán thấp) -> Release.
