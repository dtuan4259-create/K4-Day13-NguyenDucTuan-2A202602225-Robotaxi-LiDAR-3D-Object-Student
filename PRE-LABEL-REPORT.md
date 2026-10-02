# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng:
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-group` / `executed-on-room-LC-machine` / `provided-results`.
- Người thực sự chạy; ngày/giờ; hệ máy/architecture:
- Image tag và image ID; phiên bản repo:
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp:
- Phạm vi: front-window; score threshold:
- Giả định kênh thứ tư/intensity và nguồn z_ground:

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | | |
| B | 1.73 | 0.16 | 13 | 1.034 | | |
| C | 1.73 | 0.32 | 6 | 1.091 | | |

- **A/B — chỉ đổi delta:** A có 1 hộp; B có 13 hộp, tăng 12 hộp. Đối chiếu số liệu tổng hợp của A/B cho thấy mean_z tăng từ 0.330 lên 1.034. Chưa có file và ảnh thực tế để chỉ ra vùng nào xuất hiện thêm hoặc mất hộp. Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ. Điều em còn chưa chắc là các hộp tăng thêm có khớp vật thể thật hay không và tọa độ đầu ra đã được đưa về hệ tọa độ gốc hay chưa. Chênh lệch mean_z không thể được coi là độ dịch của từng hộp vì hai lượt có tập hộp khác nhau.

- **B/C — chỉ đổi pillar:** B có 13 hộp; C có 6 hộp, giảm 7 hộp về tổng số. Pillar XY tăng từ 0.16 lên 0.32, còn delta giữ nguyên ở 1.73. mean_z tăng từ 1.034 lên 1.091. Chưa có JSON và ảnh đối chiếu để xác định lớp, vị trí hoặc vùng cụ thể thay đổi; cũng chưa thể khẳng định 6 hộp của C tương ứng với 6 trong 13 hộp của B. Không đủ bằng chứng để kết luận C tốt hơn: cần đối chiếu cùng ROI, cùng ngưỡng confidence và nhãn tham chiếu để phân biệt giảm false positive với tăng miss.

- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?** Chỉ đánh giá miss đối với vật thể nằm trong ROI được quy định và có đủ dữ liệu quan sát. Vật thể ngoài ROI hoặc bị cắt ở biên không nên tự động tính là model bỏ sót. Góc Side giúp kiểm tra độ cao z và mức độ hộp bám vào cụm điểm, nhưng có thể làm các vật thể chồng lên nhau, che khuất hộp hoặc gây khó đọc hướng. Không nên kết luận yaw đúng/sai chỉ từ ảnh Side; cần xem thêm góc BEV/Top và đối chiếu quy ước yaw.

- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?** Với bằng chứng hiện có, cả `A.json`, `B.json` và `C.json` đều chưa đủ cơ sở xác nhận sẵn sàng import. Đặc biệt, B/C cần kiểm tra cách xử lý delta và việc hoàn nguyên tọa độ nếu đầu ra còn nằm trong hệ tọa độ đã dịch. Tiếp theo cần kiểm tra: đúng frame và point cloud; schema phù hợp với định dạng import; ánh xạ class; đơn vị và thứ tự trục; tọa độ tâm và kích thước hộp; quy ước yaw; ROI; rồi chồng hộp lên point cloud ở cả Side và BEV để kiểm tra từng hộp. Giữ các ca này ở bước QC, chưa import CVAT.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
|---|---|---|---|---|---|
| case-correct | 0/20 | Δz = 0 m | Không đổi | Không dừng batch; đối chiếu không phát hiện lệch z | Giả lập: tọa độ z của 20/20 hộp khớp dữ liệu chuẩn |
| case-batch-z | 20/20 | Tất cả hộp tăng z thêm 0,50 m | Không đổi | Dừng batch; kiểm tra phép biến đổi tọa độ và offset z toàn batch | Giả lập: 20/20 hộp có Δz = +0,50 m; các trường còn lại giữ nguyên |
| case-one-box-z | 1/20 | Hộp `box_012` tăng z thêm 0,35 m | Không đổi | Kiểm từng hộp; sửa `box_012` và rà soát các hộp còn lại | Giả lập: `box_012` có Δz = +0,35 m; 19 hộp còn lại có Δz = 0 m |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục: vai trò đã làm; một quan sát A/B/C có dẫn file hoặc hộp/vùng; diễn giải phép z thuận/ngược; một quyết định lỗi batch và hành động; điều chưa chắc. Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
