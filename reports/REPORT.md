# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Lê Hùng Cường

Công cụ gán nhãn đã dùng: CVAT

## 1. Dữ liệu và cách chia tập

Camera đứng cố định một chỗ, nên một chiếc xe nằm trong hình vài giây liên tiếp. Vì vậy tập chưa gán nhãn
(pool) và tập kiểm thử (test) được chia theo trục thời gian, có một vùng đệm ở giữa, thay vì chia ngẫu nhiên.
Nếu chia ngẫu nhiên, cùng một chiếc xe có thể xuất hiện ở cả hai ảnh gần nhau trong hình: một ảnh rơi vào tập
học, ảnh kề đó rơi vào tập kiểm thử. Khi đó mô hình vừa được học trên chiếc xe đó, vừa được chấm đúng trên gần
như chính chiếc xe đó, điểm số đo trên tập kiểm thử sẽ bị lệch cao hơn sự thật (rò rỉ dữ liệu) chứ không phản
ánh đúng khả năng nhận xe ở những cảnh hoàn toàn mới.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Số chính là điểm khớp khung AP50 = `0.771`. Độ phủ (recall) theo kích thước xe cho thấy xe lớn chỉ được tìm
thấy `0.561`, xe vừa `0.547`, còn xe nhỏ chỉ `0.182` — nghĩa là xe ở xa (trong ảnh hiện ra nhỏ) bị bỏ sót nhiều
hơn hẳn xe ở gần. Trên ảnh `outputs/compare_round0.jpg`, có chỗ khung AI lệch hoặc bỏ sót ở những xe xa, chỉ
thấy hai chấm đèn hoặc xe bị cắt ở mép ảnh; mô hình hay vẽ khung thiếu hoặc lệch những xe này.

Một trường hợp cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai: nhãn tham chiếu dùng để chấm
cũng do một mô hình khác vẽ tự động, chưa có người xem từng khung, nên có thể có chỗ nhãn chấm sai (thiếu xe,
khung lệch hoặc khoanh nhầm vệt đèn) chứ không phải mô hình của tôi sai.

## 3. Chiến lược chọn mẫu

Mỗi ảnh trong pool được chấm một điểm theo công thức `score = W_U·U + W_A·A + W_D·D` (mặc định 0.5 / 0.3 / 0.2):

- `U` là độ bất định (uncertainty), tính từ trung bình 5 box có conf gần 0.5 nhất — model lưỡng lự nhất; chiếm
  một nửa điểm.
- `A` là số box "mập mờ" có conf 0.15–0.50 (những box model vẽ mà chưa tự tin), chuẩn hoá theo max của pool;
  chiếm ba phần mười điểm.
- `D` là khoảng cách thời gian tới frame đã gán gần nhất (chặn ở 10s), khuyến khích ảnh khác thời điểm; chiếm
  hai phần mười điểm (vòng đầu chưa có frame nào đã gán nên D = 1 cho mọi ảnh).

`MIN_GAP_S = 2s` chỉ yêu cầu hai ảnh trong cùng lô phải cách nhau ít nhất 2 giây, vì camera đứng yên, hai ảnh
sát nhau gần như giống hệt nhau — gán cả hai tốn công mà model học thêm rất ít.

Dẫn ba frame trong `reports/SELECTION.md`: `frame_0182.jpg` (0.9591, U = 0.9182, 18 box mập mờ), `frame_0099.jpg`
(0.9063, U = 0.946, 14 box mập mờ) và `frame_0107.jpg` (0.8876, U = 0.8752, 15 box mập mờ) — đều bất định cao
và nhiều box mập mờ. Một frame khác để cân nhắc: `frame_0372.jpg` (hạng 6, 0.9101) cao hơn nhiều ảnh được chọn
nhưng bị loại vì chỉ cách `frame_0369.jpg` 1.2 giây (< MIN_GAP_S); sửa cả hai thì tốn công mà ít học thêm.

Điểm bất định (score) cao không chứng minh ảnh đó sẽ cải thiện mô hình: nó chỉ nghĩa là model đang phân vân.
Việc sửa ảnh xong có làm model nhận xe tốt hơn hay không chỉ biết được sau khi fine-tune và đo lại trên tập test.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 328 | 0.528 | -0.243 | 1.000 | 0.084 | 0.156 | 0.000 | 0.085 | 0.220 |

Vòng 1, tôi đã sửa pre-label trên 12 ảnh rất nhiều (theo `outputs/round1_diff.md`): model đề xuất 169 box,
sau khi sửa còn 328 box — giữ nguyên 149 box (accepted), kéo lại 8 box (edited), xoá 12 box sai (deleted),
thêm 171 box model bỏ sót (added), accept rate 88%. AP50 giảm từ `0.771` (vòng 0) xuống `0.528` (vòng 1), tức
giảm `-0.243` so với cold start. Precision@0.25 thì tăng lên 1.000 nhưng recall tụt còn 0.084 (từ 0.489): model
sau fine-tune chỉ dự đoán rất ít box (34 TP, 0 FP) và bỏ sót gần như toàn bộ xe — R small về 0.000, R medium
0.085, R large 0.220, nhóm nào cũng tệ hơn vòng 0.

Phân biệt ba nguồn: quan sát độc lập của tôi ghi ở `reports/BLIND_SCAN.md` (ảnh `frame_0099.jpg`, thấy xe hoà
vào bóng tối chỉ hiện đèn pha và xe bị mép ảnh cắt); các khung tôi đã sửa ghi ở `reports/REVIEW_LOG.csv`
(`frame_0187.jpg` xe bị mờ thêm box, `frame_0227.jpg` xe buýt chỉnh box, `frame_0270.jpg` xe tối sát mép trái
thêm box); kết quả model sau train nằm ở `outputs/compare_round1.jpg` và `outputs/round1_diff.md`. Một ca kết
quả đổi sau fine-tune: precision tăng lên 1.00 nhưng recall tụt từ 0.489 xuống 0.084 — model trở nên "kén" hơn,
chỉ giữ lại rất ít box tự tin thay vì vẽ nhiều như vòng 0; đây là chỗ có thể kiểm bằng cách so `compare_round0.jpg`
với `compare_round1.jpg` (trên cùng ảnh test, số box dự đoán giảm rõ rệt).

## 5. Kết luận và giới hạn

Vòng 1 AP50 = 0.528, thấp hơn cold start 0.771 (−0.243), nên tôi chọn dừng học thêm chứ không chạy tiếp vòng 2
bằng cùng cách này: 12 ảnh đã phải sửa rất nhiều (171 box thêm mới, accept rate 88%) nhưng điểm lại giảm, cho
thấy việc chỉ thêm một lô nhỏ chưa đủ và nhãn tôi sửa có thể chưa nhất quán. Hai ca còn yếu hoặc bất định cho
vòng sau: (1) xe ở rất xa chỉ còn hai chấm đèn, thân gần như chìm trong bóng tối; (2) xe bị cắt ở mép ảnh chỉ
còn phần đuôi/đèn. Sửa thêm những ca này mất thời gian, và không nên chọn hai ảnh sát nhau (< 2 giây) vì chúng
gần như một cảnh. Tập kiểm thử chỉ 20 ảnh, có luật bỏ qua box cao dưới 16 px (14 box bị bỏ qua) và nhãn tham
chiếu do mô hình tạo chưa được người rà — các giới hạn đó làm kết luận "điểm giảm" kém chắc chắn hơn: một phần
sai số có thể đến từ nhãn chấm chưa đúng, hoặc do tập test quá nhỏ. Vì AP50 giảm, trước khi train thêm tôi sẽ
xem lại các khung đã sửa trong `labels/round1/` và `REVIEW_LOG.csv` để đảm bảo chưa vẽ sai/thiếu và giữ cách gán
nhãn nhất quán, rồi mới quyết định có chạy vòng tiếp theo hay không.