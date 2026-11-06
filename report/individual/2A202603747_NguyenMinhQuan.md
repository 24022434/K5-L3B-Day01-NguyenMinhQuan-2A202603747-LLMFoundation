# Báo cáo cá nhân — Day 01: Lọc tin nhắn rác bằng ML, DL, Embedding và LLM

- **MSSV:** 2A202603747
- **Họ và tên:** Nguyen Minh Quan
- **SEED:** 4376

## 1. Bằng chứng

| Tệp | Có? |
| --- | --- |
| `day01_lab.ipynb` — notebook đã chạy, còn output | ✅ |
| `lab1_report.json` — do ô “Báo cáo cuối” sinh ra | ✅ |
| `hand_calc.jpg` — ảnh tính tay có các bước | ✅ |

## 2. Kết quả chính

| Phương án | F1 test | F1 SUB 40 | p95 ms/tin | USD / 1k tin |
| --- | ---: | ---: | ---: | ---: |
| BoW + LogReg | 0.823 | 0.833 | 0.046 | 0.000 |
| BoW + MLP | 0.955 | 0.919 | 0.340 | 0.000 |
| Encoder + kNN | 0.899 | 1.000 | 25.133 | 0.000 |
| Encoder + LogReg | 0.546 | 0.851 | 34.249 | 0.000 |
| Encoder + MLP head | 0.951 | 1.000 | 18.439 | 0.000 |
| LLM nhỏ zero-shot | 0.930 | 0.930 | 3513.580 | 0.000* |
| LLM lớn zero-shot | 0.909 | 0.909 | 3227.817 | 0.000* |
| LLM nhỏ few-shot | 0.952 | 0.952 | 3111.588 | 0.000* |

\*Bảng giá trong notebook chưa khai báo giá cho model Groq nên hiển thị 0; không có nghĩa là mọi API LLM đều miễn phí trong môi trường production.

## 3. Dự đoán và thực tế

1. **Model LLM lớn so với nhỏ.** Mình dự đoán model lớn có F1 nhỉnh hơn. Kết quả ngược lại: model nhỏ đạt F1 `0.930`, lớn đạt `0.909`. Tuy nhiên chênh `0.021` trên SUB 40 tin là nhỏ, nên chưa thể kết luận chắc chắn model nhỏ tốt hơn.
2. **Learning rate 50.** Mình dự đoán loss có thể không hội tụ hoặc dao động mạnh. Thực tế loss ban đầu dao động/tăng mạnh nhưng sau đó vẫn giảm và hội tụ trong lần chạy này. Điều đó cho thấy learning rate lớn không luôn làm mô hình hỏng ngay, nhưng có rủi ro không ổn định.
3. **Stress test.** Mình dự đoán BoW khó nhất ở nhóm né từ khóa và LLM khó nhất ở prompt injection. Thực tế BoW khó nhất ở spam giọng tự nhiên (LogReg đúng 10%, MLP đúng 0%); LLM zero-shot khó nhất ở nhóm tưởng spam nhưng không phải (60%). Prompt injection vẫn làm LLM sai 1/6 case.

## 4. Phân tích 50 case khó

BoW + MLP tụt nhiều nhất: F1 SUB từ `0.919` xuống `0.410` trên 50 case khó, giảm `0.509`. Tập test thường có cùng phân phối với dữ liệu train nên quá lạc quan; bộ stress test đưa vào các cách diễn đạt và kiểu tấn công khác phân phối đó.

Với BoW, nhóm spam giọng tự nhiên khó nhất vì mô hình nhìn token rời rạc, không nhận ra ý nghĩa của lời mời chào/lừa đảo khi thiếu từ khóa quen thuộc. Ví dụ case #13 (“Your photos from the event are ready...”) là spam nhưng LLM phòng thủ vẫn đoán sai vì câu giống một lời nhắc hợp lệ. Case #2 (“FREE pizza in the office kitchen...”) là ham nhưng LLM phòng thủ đoán spam vì chứa các tín hiệu `FREE` và thúc giục hành động.

## 5. Thử thách cá nhân

Đề #3 yêu cầu so sánh MLP có 4 và 512 nơ-ron ẩn theo số tham số, F1 test và thời gian train.

| Hidden neurons | Số tham số | F1 test | Thời gian train |
| ---: | ---: | ---: | ---: |
| 4 | 8,009 | 0.9070 | 0.90 s |
| 512 | 1,025,025 | 0.9551 | 4.29 s |

Mạng 512 nơ-ron tăng F1 thêm `0.0481` (4.81 điểm phần trăm), nhưng có khoảng 128 lần nhiều tham số hơn và train chậm hơn khoảng 4.77 lần. Nên chọn mạng nhỏ khi dữ liệu ít, tài nguyên tính toán hạn chế, cần tốc độ xử lý nhanh, hoặc khi mức tăng F1 của mạng lớn không đáng kể so với chi phí tăng thêm.

## 6. Phương án cho sản phẩm thật

Nhà mạng cần 116 tin/giây trung bình và p95 dưới 50 ms/tin. Các mô hình chạy cục bộ đều đạt SLA; lựa chọn chính là **BoW + MLP** vì F1 test cao nhất (`0.955`), p95 chỉ `0.340 ms/tin`, throughput khoảng `44,935 tin/giây`, và không có chi phí API trực tiếp. Encoder + MLP head cũng mạnh (F1 `0.951`, p95 `18.439 ms`) nhưng chậm hơn đáng kể.

Thiết kế lai phù hợp là dùng BoW + LogReg lọc các tin rất chắc chắn, rồi chuyển tin có xác suất lưỡng lự sang Encoder + MLP head. Cả hai tầng đều đạt SLA; LLM chỉ nên dùng cho audit hoặc review offline vì p95 hơn 3 giây. Theo giả định gpt-6-luna ở phần tính tay, 10 triệu SMS/ngày sẽ tốn khoảng `$105/ngày`, tương đương `$3,150/tháng` trong 30 ngày.

## 7. AI log và điều đã học

| Nội dung hỏi AI | Nội dung nhận được | Điều mình đã làm/review |
| --- | --- | --- |
| Debug vectorize, metrics và lỗi cài PyTorch | Giải thích cách token hóa, chuyển list sang NumPy array, và chẩn đoán lỗi môi trường | Tự chạy lại các cell kiểm tra và sửa code trong notebook |
| Giải thích ML, DL, embedding, F1 và overfitting | Diễn giải các khái niệm, cách đọc loss curve và bảng metrics | Đối chiếu với kết quả chạy của SEED 4376 |
| Phần 4 LLM được phép dùng AI | Gợi ý triển khai API, prompt, parse label và batch evaluation | Review code, chạy test mock/API và dùng Groq cho thí nghiệm |
| Đọc kết quả và viết báo cáo | Hỗ trợ tổng hợp số liệu, giải thích stress test, Pareto và SLA | Kiểm tra lại các số liệu lấy từ `lab1_report.json` và output notebook |

Mình hiểu rõ hơn rằng accuracy cao không đủ khi dữ liệu lệch lớp; cần xem precision, recall và F1. Mình cũng nhận ra điểm cao trên test cùng phân phối không chứng minh model bền vững: stress test có thể làm F1 giảm rất mạnh. Cuối cùng, model lớn hoặc chậm hơn không tự động tốt hơn; lựa chọn production phải dựa đồng thời vào chất lượng, độ trễ, chi phí và độ bền.
