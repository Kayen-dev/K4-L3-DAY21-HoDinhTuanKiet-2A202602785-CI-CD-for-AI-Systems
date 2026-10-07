# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Hồ Đình Tuấn Kiệt |
| MSSV | 2A202602785 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/Kayen-dev/K4-L3-DAY21-HoDinhTuanKiet-2A202602785-CI-CD-for-AI-Systems |
| Ngày nộp | 08/10/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---:|---:|---:|---:|---:|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |

Tôi chọn `n_estimators=200`, `learning_rate=0.1`, `max_depth=5` vì F1-score 0.7149 là cao nhất và vượt quality gate 0.65. Lần 1 có accuracy cao hơn một chút (0.8780) nhưng F1 thấp hơn; vì vậy accuracy không phản ánh khả năng nhận diện lớp thu nhập cao. Lần 2 giảm đồng thời số cây và learning rate nên mô hình học chưa đủ, dẫn đến F1 thấp nhất. Kết quả cũng cho thấy learning rate thấp thường cần nhiều estimators hơn để bù khả năng học.

## 2. Vì Sao Quality Gate Đặt Trên F1 Chứ Không Phải Accuracy

Lớp dương, tức thu nhập trên 50K, chỉ chiếm khoảng 24,8% dữ liệu. Một mô hình luôn dự đoán “thu nhập thấp” vẫn có thể đạt accuracy xấp xỉ 75,2%, nhưng không phát hiện được bất kỳ trường hợp thu nhập cao nào. Vì thế dùng accuracy làm quality gate sẽ dễ chấp nhận một mô hình không hữu ích cho lớp cần quan tâm. F1 của lớp dương kết hợp precision và recall, vừa phạt dự đoán dương sai vừa phạt bỏ sót người thu nhập cao. Tôi dùng `f1_score(y_true, y_pred)` mặc định cho nhãn dương 1, không dùng `average="weighted"` hoặc `average="macro"`, vì các cách trung bình này có thể làm ảnh hưởng của lớp âm lớn che khuất chất lượng nhận diện lớp dương.

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| Không liệt kê được bucket S3 | IAM user thiếu quyền `s3:ListAllMyBuckets` | Bổ sung policy list bucket và policy giới hạn quyền tại bucket `ai-lab-cicd`. |
| Train trên GitHub Actions lỗi quyền `/D:` | Metadata MLflow local chứa đường dẫn artifact tuyệt đối của Windows | Đặt `MLFLOW_TRACKING_URI=sqlite:///mlflow-ci.db` riêng cho job Train trên Linux. |
| Release cần xác thực và địa chỉ ổn định | Runner GitHub phải SSH vào VM, IP công cộng có thể thay đổi | Tạo EC2 role đọc artifact S3, Elastic IP, security group và lưu SSH key/host dưới GitHub Secrets. |

## 4. So Sánh Bước 2 và Bước 3

| | f1_score | accuracy |
|---|---:|---:|
| Bước 2 (chỉ `train_batch1`) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2`) | 0.7354 | 0.8820 |

Sau khi ghép thêm 22.361 mẫu, F1-score tăng 0.0205 và accuracy tăng 0.0080. Hai batch đến từ cùng nguồn nên cải thiện không quá lớn; kết quả xác nhận DVC đã phiên bản hóa dữ liệu và pipeline đã tự huấn luyện, kiểm tra chất lượng, upload model rồi triển khai lại API thành công.
