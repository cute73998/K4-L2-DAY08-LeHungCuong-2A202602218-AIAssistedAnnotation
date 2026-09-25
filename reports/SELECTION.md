# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, nếu chỉ có ngân sách rà năm ảnh, tôi ưu tiên:
`frame_0182.jpg` (hạng 1, điểm 0.9591, t = 72.8s), `frame_0369.jpg` (hạng 2, điểm 0.9324, t = 147.6s),
`frame_0380.jpg` (hạng 3, điểm 0.917, t = 152.0s), `frame_0326.jpg` (hạng 4, điểm 0.9155, t = 130.4s)
và `frame_0331.jpg` (hạng 5, điểm 0.9154, t = 132.4s). Năm ảnh này đứng đầu danh sách theo cột `score`.
`frame_0182.jpg` và `frame_0187.jpg` (hạng 10, t = 74.8s) cách nhau khoảng 2 giây, cảnh gần giống nhau,
nên nếu chỉ được năm ảnh tôi không lấy cả hai.

Ba frame thuộc lô 12 ảnh model chọn, tôi xem `frame_0182.jpg` (điểm 0.9591, U = 0.9182, 18 box mập mờ),
`frame_0099.jpg` (điểm 0.9063, U = 0.946, 14 box mập mờ) và `frame_0107.jpg` (điểm 0.8876, U = 0.8752,
15 box mập mờ). Cả ba đều có điểm trên 0.88, model khoanh nhiều xe nhưng còn nhiều khung chưa chắc
(cột `n_ambiguous` cao), nên sửa ba ảnh này giúp model học thêm chỗ nó đang lưỡng lự.

Một frame có điểm cao nhưng không chọn: `frame_0372.jpg` (hạng 6, điểm 0.9101, t = 148.8s) cao hơn nhiều
ảnh đã được chọn (như `frame_0187.jpg` 0.8995, `frame_0312.jpg` 0.91, `frame_0392.jpg` 0.8874), nhưng model
bỏ qua vì nó sát giờ với `frame_0369.jpg` (t = 147.6s, chỉ cách 1.2 giây, nhỏ hơn MIN_GAP_S = 2s). Hai ảnh
gần như cùng một cảnh, sửa cả hai thì tốn công mà model ít học thêm được gì.

Phép chọn này chưa chứng minh về chất lượng mô hình: điểm cao chỉ nghĩa là AI đang phân vân (bất định
cao, nhiều box mập mờ). Nó chưa chứng minh sửa ảnh đó xong thì AI sẽ nhận xe tốt hơn.