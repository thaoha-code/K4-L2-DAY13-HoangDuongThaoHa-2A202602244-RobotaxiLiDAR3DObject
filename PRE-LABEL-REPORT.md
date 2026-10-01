# Báo cáo thực hành PointPillars — Day 13

> Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

* **Mã nhóm/phòng:** [ĐIỀN MÃ NHÓM / PHÒNG]
* **Thành viên:** xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
* **Trạng thái:** `executed-by-group`
* **Người thực sự chạy:** [ĐIỀN HỌ TÊN NGƯỜI CHẠY]
* **Ngày/giờ:** 01/10/2026, khoảng 15:02–15:03 (giờ máy)
* **Hệ máy/architecture:** Windows + Docker Linux container, `amd64`
* **Docker image:** `day13-pointpillars:lc-20261001-amd64`
* **Image ID:** `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
* **Phiên bản repo:** `0831856d921609312d42c7582c366e5a311bb7b1`
* **PCD được cấp / frame_id:** `pcd-source` / `demo`
* **Nơi thực hiện phép chạy:** máy nhóm, sử dụng Student Release bundle amd64
* **Input fingerprint:** `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
* **Checkpoint:** PointPillars KITTI có sẵn trong image, `epoch_160.pth`
* **Checkpoint SHA-256:** `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
* **Phạm vi:** `front-window`
* **Score threshold:** `0.3`
* **Dataset/model:** `KITTI`
* **Giả định kênh thứ tư/intensity:** Reflectance đã được loại khỏi input practice; RGB sử dụng placeholder 0 theo bundle, không coi đây là benchmark KITTI intensity thực.
* **Nguồn z_ground:** `z_ground = 0.075 m`, được ghi nhận từ dữ liệu/bundle.

## Kết quả smoke test

Smoke test hoàn tất thành công:

* `status: passed`
* `docker-load: passed`
* `run-A: passed`
* `run-B: passed`
* `run-C: passed`
* `qc-cases: passed`

Các output được tạo đầy đủ trong thư mục kết quả:

```text
C:\Lab13\ket-qua-nhom-01\
├── smoke.json
├── run-A\
├── run-B\
├── run-C\
└── qc-cases\
```

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV                                                                                              | Quan sát có bằng chứng                                                                                          |
| ---- | ----: | --------: | -----: | -----: | --------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| A    |     0 |      0.16 |      1 |  0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`; `run-A/side-demo-delta-0-voxel-0.16.png`; `run-A/summary.csv`       | 1 prediction, class `vehicle`. Đây là cấu hình không dịch Z trong thí nghiệm A/B.                               |
| B    |  1.73 |      0.16 |     13 |  1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`; `run-B/side-demo-delta-1.73-voxel-0.16.png`; `run-B/summary.csv` | 10 `vehicle`, 2 `pedestrian`, 1 `two-wheels`. So với A, chỉ thay đổi delta Z nhưng số prediction thay đổi mạnh. |
| C    |  1.73 |      0.32 |      6 |  1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`; `run-C/side-demo-delta-1.73-voxel-0.32.png`; `run-C/summary.csv` | 6 `pedestrian`. So với B, giữ delta Z và thay pillar XY từ 0.16 lên 0.32. Số prediction giảm từ 13 xuống 6.     |

### A/B: thay input trước model có làm khác dịch cùng một hằng số cho output không? Vì sao?

Có. Trong thí nghiệm A/B, pillar XY được giữ nguyên ở `0.16 m`, trong khi delta thay đổi từ `0` lên `+1.73 m`.

Kết quả quan sát được:

```text
A: 1 box, mean_z = 0.330 m
B: 13 boxes, mean_z = 1.034 m
```

Phân bố class cũng thay đổi:

```text
A: 1 vehicle
B: 10 vehicles + 2 pedestrians + 1 two-wheels
```

Do đó, thay đổi Z translation trước inference không chỉ làm thay đổi giá trị tọa độ mà còn làm prediction của model thay đổi mạnh. Đây là bằng chứng cho thấy pipeline/model phụ thuộc vào hệ tọa độ và phép biến đổi Z được sử dụng.

Không kết luận rằng 13 box ở B là ground truth hoặc tất cả prediction mới đều là object thật. Kết quả này được dùng để kiểm tra độ nhạy của pipeline đối với transform tọa độ.

### B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?

Có thay đổi rõ ràng khi giữ `delta = 1.73 m` và đổi pillar XY từ `0.16 m` sang `0.32 m`.

```text
B: voxel = 0.16 m → 13 boxes, mean_z = 1.034 m
C: voxel = 0.32 m → 6 boxes, mean_z = 1.091 m
```

Như vậy, thay đổi độ phân giải voxelization làm thay đổi số lượng và phân bố prediction.

Tuy nhiên, **chưa đủ bằng chứng để kết luận cấu hình B hoặc C tốt hơn**, vì thí nghiệm này không cung cấp ground truth để tính accuracy/precision/recall. Số lượng box nhiều hay ít không tự nó chứng minh chất lượng model.

### Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?

Thí nghiệm sử dụng phạm vi `front-window` và side-view được cung cấp bởi bundle. Vì vậy, quan sát từ một góc Side không đại diện đầy đủ cho toàn bộ không gian 3D.

Các object ở rìa ROI, object bị che khuất hoặc có hình học sparse có thể khó đánh giá chỉ bằng một góc nhìn. Đối với yaw hoặc trường hợp miss/extra, cần đối chiếu với point cloud và các view phù hợp thay vì kết luận chỉ từ số lượng box.

Do đó, các observation trong report được giới hạn ở những gì thể hiện trong output của controlled experiment, không dùng để claim chất lượng tổng thể của model.

### JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?

Các JSON trong `run-A`, `run-B`, `run-C` và `qc-cases` là output phục vụ controlled experiment/training practice.

Đặc biệt:

* Các file trong `qc-cases` được manifest đánh dấu `training_only`.
* `qc-cases/manifest.json` ghi rõ: **not ground truth, not for CVAT import or quality claims**.
* Không import `case-correct.json`, `case-batch-z.json` hoặc `case-one-box-z.json` vào CVAT.
* Không dùng các prediction demo này làm annotation ground truth cho Robotaxi.

Nếu cần đưa prediction vào một workflow annotation thực tế, cần kiểm tra schema, coordinate frame, class mapping, transform và xác nhận nguồn dữ liệu/ground truth trước khi import.

## Các ca QC có kiểm soát — không import CVAT

| Ca               | Số hộp / tóm tắt | Lượng lệch                                           | Class/x/y/yaw có đổi?                                               | Dạng batch, kiểm từng hộp hay chưa rõ?                                                          | Bằng chứng                                                                                                  |
| ---------------- | ---------------- | ---------------------------------------------------- | ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `case-correct`   | 13 box           | Không chỉnh sửa từ prediction nguồn                  | Prediction gốc                                                      | Baseline để đối chiếu                                                                           | `case-correct.json`, `side-correct.png`; manifest ghi rõ đây là model output, không phải accuracy reference |
| `case-batch-z`   | 13 box           | Mọi box bị dịch Z xuống `delta + z_ground = 1.805 m` | Không phải một inference mới; helper chỉ tạo biến thể từ prediction | Lỗi có tính batch/systematic → cần STOP và kiểm tra transform/pipeline                          | `case-batch-z.json`, `side-batch-z.png`, `manifest.json`                                                    |
| `case-one-box-z` | 13 box           | Chỉ box đầu tiên bị dịch Z xuống `1.805 m`           | Chỉ tạo biến thể Z của một object                                   | Lỗi object-local → cần kiểm tra object đó ở nhiều view/đối chiếu point cloud trước khi kết luận | `case-one-box-z.json`, `side-one-box-z.png`, `manifest.json`                                                |

### Diễn giải các ca QC

#### `case-correct`

Đây là bản sao không chỉnh sửa của prediction nguồn từ run B.

```text
source prediction: boxes-demo-delta-1.73-voxel-0.16.json
box_count: 13
delta: 1.73 m
z_ground: 0.075 m
```

Case này chỉ đóng vai trò baseline để so sánh. Nó **không phải ground truth** và không được dùng để claim accuracy.

#### `case-batch-z`

Manifest xác định mọi box bị dịch xuống:

```text
delta + z_ground
= 1.73 + 0.075
= 1.805 m
```

Vì toàn bộ batch bị tác động theo cùng một kiểu, đây là dấu hiệu của một lỗi có tính hệ thống trong transform/pipeline.

Quyết định QC:

> **STOP và kiểm tra transform/pipeline trước khi tiếp tục sửa từng box.**

Không nên sửa thủ công từng prediction khi chưa xác định nguyên nhân chung.

#### `case-one-box-z`

Chỉ box đầu tiên bị dịch Z xuống `1.805 m`.

Đây là trường hợp object-local. Không đủ cơ sở để kết luận toàn pipeline bị lỗi chỉ từ một box.

Cần:

1. Đối chiếu box đó với point cloud.
2. Kiểm tra object ở các view phù hợp.
3. So sánh với các object tương tự trong cùng frame.
4. Nếu bằng chứng chưa đủ, ghi nhận uncertainty và yêu cầu review thêm.

### Lưu ý về helper

Các file:

```text
case-correct.json
case-batch-z.json
case-one-box-z.json
```

là các controlled QC cases được tạo từ prediction nguồn.

Helper chỉ tạo biến thể phục vụ kiểm tra lỗi Z có kiểm soát; đây **không phải các lượt inference độc lập** và **không phải nhãn đúng/ground truth**.

## Nhận xét cá nhân

### Thành viên 1

* **Họ tên/MSSV:** [ĐIỀN]
* **Vai trò:** [ĐIỀN VAI TRÒ]
* **Đã làm:** Thực hiện/kiểm tra một phần workflow PointPillars A/B/C và kiểm tra output.
* **Quan sát có bằng chứng:** A có 1 box với mean_z `0.330 m`; B có 13 box với mean_z `1.034 m` khi delta Z thay đổi từ `0` lên `1.73 m`.
* **Diễn giải phép Z thuận/ngược:** Cần giữ nhất quán hệ tọa độ giữa input, ground và prediction; một transform sai có thể làm thay đổi prediction hàng loạt.
* **Quyết định lỗi batch:** Nếu nhiều box cùng bị lệch theo cùng một hướng/mức độ, ưu tiên dừng và kiểm tra transform/pipeline.
* **Điều chưa chắc:** Không dùng controlled output làm ground truth và không kết luận accuracy từ số box.

### Thành viên 2

* **Họ tên/MSSV:** [ĐIỀN]
* **Vai trò:** [ĐIỀN VAI TRÒ]
* **Đã làm:** [ĐIỀN CÔNG VIỆC THỰC TẾ]
* **Quan sát có bằng chứng:** B có 13 box tại voxel `0.16 m`, trong khi C có 6 box tại voxel `0.32 m`, với cùng delta `1.73 m`.
* **Diễn giải:** Thay đổi voxelization làm thay đổi prediction.
* **Quyết định lỗi batch:** Batch shift cần được kiểm tra ở transform/pipeline trước khi chỉnh từng object.
* **Điều chưa chắc:** Chưa có ground truth nên không thể kết luận cấu hình voxel nào có accuracy tốt hơn.

### Thành viên 3

* **Họ tên/MSSV:** [ĐIỀN]
* **Vai trò:** [ĐIỀN VAI TRÒ]
* **Đã làm:** [ĐIỀN CÔNG VIỆC THỰC TẾ]
* **Quan sát có bằng chứng:** `case-batch-z` làm tất cả 13 box bị dịch Z `1.805 m`, trong khi `case-one-box-z` chỉ tác động một box.
* **Quyết định:** Batch shift → inspect transform/pipeline; one-box shift → kiểm tra object/local evidence trước khi kết luận.
* **Điều chưa chắc:** Cần thêm bằng chứng từ point cloud/view để phân biệt lỗi annotation với lỗi transform trong trường hợp object-local.

### Thành viên 4 (nếu có)

* **Họ tên/MSSV:** [ĐIỀN]
* **Vai trò:** [ĐIỀN VAI TRÒ]
* **Đã làm:** [ĐIỀN CÔNG VIỆC THỰC TẾ]
* **Quan sát:** [ĐIỀN QUAN SÁT CÓ FILE/VÙNG CỤ THỂ]
* **Quyết định:** [ĐIỀN QUYẾT ĐỊNH]
* **Điều chưa chắc:** [ĐIỀN]

> Nếu nhóm chỉ có 3 thành viên thì xóa mục Thành viên 4.

## LC ghi nhận riêng

* **Quyền dùng PCD/image và đúng ca:** [LC ĐIỀN]
* **Có chạy thật / chỉ phân tích:** Nhóm đã thực hiện pipeline bằng Student Release bundle trên máy nhóm; smoke test và A/B/C/QC đều `passed`.
* **Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:** Đã tạo đầy đủ `run-A`, `run-B`, `run-C`, `qc-cases` và `smoke.json`. Các controlled QC cases được giữ riêng và không import vào CVAT.
* **Nhận xét từng thành viên và quyết định dừng pipeline:** [LC ĐIỀN]
* **Đồng ý chuyển sang chỉnh/QC / cần bổ sung:** [LC ĐIỀN]
* **Lý do:** [LC ĐIỀN]

## Kết luận

Pipeline PointPillars của nhóm đã chạy thành công trên Student Release bundle amd64.

Smoke test xác nhận:

```text
docker-load   = passed
run-A         = passed
run-B         = passed
run-C         = passed
qc-cases      = passed
```

Ba lượt controlled inference cho thấy:

```text
A: delta 0,    voxel 0.16 → 1 box,  mean_z 0.330
B: delta 1.73, voxel 0.16 → 13 boxes, mean_z 1.034
C: delta 1.73, voxel 0.32 → 6 boxes,  mean_z 1.091
```

Kết quả A/B cho thấy prediction nhạy với thay đổi Z transform. Kết quả B/C cho thấy prediction thay đổi khi thay đổi độ phân giải voxelization. Tuy nhiên, đây là controlled experiment trên dữ liệu KITTI demo và **không đủ để đánh giá accuracy hoặc kết luận cấu hình nào tốt hơn**.

Ba QC cases được tạo thành công. Trong đó, batch Z-shift được xem là tín hiệu cần dừng và kiểm tra transform/pipeline; one-box Z-shift cần kiểm tra object cụ thể và thu thập thêm evidence trước khi kết luận.

Các controlled QC cases chỉ phục vụ training/QC và **không phải ground truth, không import vào CVAT và không dùng để claim chất lượng annotation**.
