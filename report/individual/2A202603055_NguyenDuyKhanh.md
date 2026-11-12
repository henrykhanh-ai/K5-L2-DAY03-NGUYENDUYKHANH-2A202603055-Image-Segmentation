# Báo cáo thực hành Day 3 — Image Segmentation

- Mã học viên theo lớp: 2A202603055
- Ngày / CVAT local: 11–12/11/2026 — CVAT local (Docker trên WSL, `localhost:8080`)
- Công cụ đã dùng:
  - CVAT: polygon/mask, z-order, group.
  - Trợ lý AI (Claude) hỗ trợ thao tác CVAT qua API và tạo gợi ý (nhà trường cho phép dùng LLM hỗ trợ label).
  - Gợi ý tự động: **YOLO26x + SAM 2.1-b** cho object (Medium, Hard things, checkpoint instance, cp6), **fastseg MobileV3** làm lớp nền cho vùng stuff (Easy, Hard stuff, cp3).
  - SAM ViT-H đã cài trên CVAT local (nuclio) nhưng không dùng cho bài này.
  - Ground truth 3 tier: chỉ dùng **sau khi được phát**, để chạy scorer, tìm lỗi rồi sửa trong CVAT.
  - Các shape do model sinh giữ nguồn `auto` trong CVAT.

## 1. Bài đã nộp

| Task | File ZIP đúng tên | Hoàn thành mấy ảnh | Điểm tối đa (coach chấm sau) |
| --- | --- | ---: | ---: |
| easy_semantic | submissions/easy_semantic.zip | 3 / 3 | 20 |
| medium_instance | submissions/medium_instance.zip | 3 / 3 | 32 |
| hard_panoptic | submissions/hard_panoptic.zip | 2 / 2 | 30 |
| cp1_holes | submissions/cp1_holes.zip | 1 / 1 | 3 |
| cp2_slice | submissions/cp2_slice.zip | 1 / 1 | 3 |
| cp5_occlusion | submissions/cp5_occlusion.zip | 1 / 1 | 3 |
| cp3_thin | submissions/cp3_thin.zip | 1 / 1 | 3 |
| cp4_curb | submissions/cp4_curb.zip | 1 / 1 | 3 |
| cp6_coverage | submissions/cp6_coverage.zip | 1 / 1 | 3 |
| **Tổng tối đa** | | | **100** |

Không có task nào bị lỗi export. Mỗi task đều đã Save, export lại sau lần sửa cuối, và kiểm cấu trúc bằng `inspect_submissions.py`.

## 2. Một quyết định trước khi dùng gợi ý

- Ảnh, vị trí và object Medium tự vẽ: ảnh `000000181542`, **người nhỏ đứng xa** ở giữa-trên ảnh (khoảng x 273–284, y 113–147), class `person`. Ngoài ra tôi tự vẽ người nhỏ ở ảnh `000000458325` (x 417–425, y 184–197). Model không phát hiện các vật này nên tôi vẽ tay bằng polygon. Phần còn lại của Medium có dùng gợi ý YOLO26 + SAM 2.1, sau đó tôi kiểm tra, sửa nhãn, thu gọn hoặc thêm bằng tay.
- Class và quy tắc tôi dùng để chọn biên: Mask chỉ phủ **phần nhìn thấy**. Người và xe là hai instance riêng, kể cả khi người ngồi trên xe máy (cp5: xe máy bị chân người lái che thành 2 mảng, tôi gộp group thành **1 instance**). Ở Medium, mỗi xe hoặc người sát nhau là một mask riêng (cp2: hai xe đỗ sát nhau là hai instance).
- Nếu dùng gợi ý sau đó: vùng gợi ý sai/đúng, hành động sửa/giữ và lý do:
  - Gợi ý object của SAM khá khít. Gợi ý vùng nền của fastseg sai nhiều hơn, nên tôi rà lại từng ảnh và sửa (xem mục 3).
  - Hard ảnh 2: model gán xe nhỏ ở xa là `truck`, tôi đổi thành `car`. Xe tải bị cắt ở mép trái bị gán `car`, tôi đổi thành `truck`.
  - cp3: model gán đèn tín hiệu thành `pole`. Đèn không có trong class list nên tôi bỏ.
  - cp6: polygon thô được vẽ lại bằng SAM để ôm sát xe và bó vỉa. Taxi bị cột che giữa thân được gộp thành 1 instance.

## 3. Một lỗi tôi tìm thấy và sửa

- Task/ảnh/vùng: `easy_semantic`, ảnh `7ee6d192-89e2408b` (cao tốc chiều tối), **sườn đồi bên phải** phía trên lan can (x ≈ 610–1280, y ≈ 280–440).
- Lỗi thuộc loại: **sai lớp** (road tràn lên vegetation).
- Bằng chứng tôi nhìn thấy:
  - Khi mở CVAT, vùng cyan (road) phủ lên đồi cỏ khô và bụi cây, nằm cao hơn lan can, nơi xe không thể chạy.
  - Ba mask road (hai mask vẽ tay ở z = 0 và một mask của model) trào lên đồi khoảng 21.500 pixel.
  - Mask ở z = 0 nên lỗi này có trong file export.
- Quy tắc và hành động sửa:
  - Road theo **chức năng**: mặt xe chạy, giới hạn bởi mép đường hoặc lan can.
  - Tôi khoét phần road phía trên chân đồi (chừa xe van trắng). Vùng đó trở về vegetation từ lớp bên dưới. Dải lề xám giữa đường và đồi để nguyên.

Đã tự chạy scorer với reference được phát. Easy sau sửa: **mIoU 0,864**, coverage 99,2%; IoU từng class: road 0,968, sidewalk 0,714, building 0,819, vegetation 0,852, sky 0,965. Scorecard ba tier sau sửa: **63,1 / 82** (Easy 20,0/20 · Medium 23,9/32 · Hard 19,2/30) (chỉ là tự đánh giá, không phải điểm cuối).

Các lỗi khác cùng loại đã sửa:
- Road nằm trên vũng nước/vạt sáng bị gán sidewalk.
- Road lấn lên vỉa hè ở ảnh 2 và 3.
- Thùng điện xanh bị gán building, đã để trống.
- Nhà sau hàng cây ở ảnh 3 được bổ sung building.

Phần tôi tự chỉnh trên kết quả model (Medium và Hard): đổi nhãn sai (truck ↔ car, truck → bus), xóa shape thừa hoặc trùng, thu gọn mask ăn ra nền (người bị che, người đứng sát nhau), thêm vật model bỏ sót (người nhỏ, xe máy, người lái xe máy, xe đạp).

Sửa sau khi xem reference (ghi rõ để coach phân biệt với phần làm trước đó):
- Medium: dùng scorer để tìm **vật bị sót** (chủ yếu người và xe nhỏ ở xa), thêm 11 vật bằng YOLO26 + SAM 2.1 trên ảnh, không chép mask đáp án; sau đó tự vẽ thêm 2 người nhỏ và thu gọn 2 mask người quá cao. Recall@0.5 từ 0,80 lên 0,94.
- Hard ảnh 2:
  - Xe chở ô tô tách thành **cabin = truck**, các ô tô trên sàn = car.
  - Hai xe nhỏ ở xa đổi truck → **bus**.
  - Thêm người lái xe máy, xe đạp và vài người nhỏ.
  - Xóa các shape sai (bus trùm tòa nhà, person trên vỉa hè trống).
- Hard: road/sidewalk ở ảnh 2 **giữ road**, không sửa theo reference (xem mục 4).

## 4. Ba ca chưa chắc hoặc đã cân nhắc

| Ảnh/vị trí | Hai cách hiểu có thể | Quy tắc/chứng cứ | Quyết định hoặc câu hỏi cho coach |
| --- | --- | --- | --- |
| 1. `hard_panoptic` ảnh 2: mặt đường nhiều làn ở giữa ảnh | `road` hoặc `sidewalk` (reference ghi phần lớn là sidewalk) | Nhìn thấy làn xe, vạch kẻ, ô tô đang chạy, tức chức năng là đường xe chạy | Giữ **road** theo guideline. Hỏi coach: reference COCO gán "pavement" thành sidewalk ở đây có phải chủ ý không? |
| 2. `hard_panoptic` ảnh 2: xe chở ô tô ở làn giữa | Một `truck` duy nhất, hoặc `truck` cộng từng `car` trên sàn xe | Các ô tô trên sàn là hàng hóa nhưng vẫn là ô tô nhìn thấy rõ | Gán cả khối là **truck**. Hỏi coach: có cần tách từng ô tô trên xe chở thành instance `car` không? |
| 3. `cp3_thin` đèn tín hiệu, và `cp6_coverage` xe buýt + xe tải CAPITAL | Gán vào class gần nhất (`pole` / `traffic sign` / `car`), hoặc để trống | Class list của cp3 không có `traffic light`; của cp6 không có `bus` hay `truck` | **Để trống**, chỉ gán cột và cần đỡ đèn là `pole`. Không tô ép vùng không thuộc schema. |
| 4. `medium_instance` ảnh `000000458325`: xe bị che gần hết sau SUV đen bên trái (x ≈ 120–167, y ≈ 164–213) | Vẽ một instance `car` cho dải nhỏ nhìn thấy, hoặc bỏ qua | Chỉ lộ nóc và cột sau (~400 px); model không phát hiện; vẽ lệch dễ thành FP | Bỏ qua, ghi là vật bị sót. Hỏi coach: vật nhìn thấy dưới khoảng 10% có bắt buộc phải gán không? |
