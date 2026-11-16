# dataco-supply-chain-data-mining

Khai phá luật "rò rỉ lợi nhuận" (Apriori, FP-Growth) và phân khúc khách hàng có xét chất lượng dịch vụ (RFM mở rộng với K-Means, DBSCAN) trên bộ dữ liệu DataCo Smart Supply Chain.

## 1. Tổng quan dự án

Dự án áp dụng các kỹ thuật khai phá dữ liệu lên dữ liệu giao dịch bán lẻ và chuỗi cung ứng. Thay vì lặp lại các phân tích phổ biến trên bộ dữ liệu này (dự đoán giao trễ, market basket cơ bản, RFM thuần), dự án đặt lại cả hai bài toán xoay quanh **lợi nhuận** và **chất lượng dịch vụ**.

- **Bài toán 1 — Khai phá luật rò rỉ lợi nhuận (Luật kết hợp):** Tìm các tổ hợp điều kiện (mức giảm giá, nhóm hàng, thị trường, hình thức vận chuyển, hình thức thanh toán, phân khúc khách) thường đi kèm đơn hàng bị lỗ, và xếp hạng luật theo số tiền thực tế bị mất.
- **Bài toán 2 — Phân khúc khách hàng có xét chất lượng dịch vụ (Phân cụm):** Mở rộng RFM truyền thống bằng các đặc trưng về chất lượng giao hàng và lợi nhuận, sau đó so sánh các phân khúc thu được với RFM thuần để thấy những gì RFM không phát hiện được.
- **Bước liên kết:** Đối chiếu các luật lỗ của Bài toán 1 với các cụm khách của Bài toán 2 để xem nhóm khách nào gây ra nhiều thiệt hại nhất.

## 2. Bộ dữ liệu

- **Tên:** DataCo SMART SUPPLY CHAIN FOR BIG DATA ANALYSIS
- **Nguồn:** [Mendeley Data (DOI: 10.17632/8gx2fvg2k6.5)](https://doi.org/10.17632/8gx2fvg2k6.5)
- **Quy mô:** 180.519 bản ghi, 53 thuộc tính, 65.752 đơn hàng và 20.652 khách hàng (2015–2018).
- **Mức chi tiết:** Mỗi dòng là một order item, nên một đơn hàng có thể gồm nhiều dòng. Phân tích ở mức đơn hàng cần gom theo `Order Id`, phân tích ở mức khách hàng cần gom theo `Order Customer Id`.
- **Lưu ý:** File CSV gốc bị loại khỏi repo bằng `.gitignore` do giới hạn 100 MB của GitHub. Hãy tải `DataCoSupplyChainDataset.csv` từ Mendeley Data và đặt vào `data/raw/`.

### Các cột chính được sử dụng

| Mục đích | Cột |
|---|---|
| Nhãn lỗ và lợi nhuận | `Order Profit Per Order`, `Order Item Profit Ratio` |
| Điều kiện đơn hàng | `Order Item Discount Rate`, `Category Name`, `Market`, `Order Region`, `Shipping Mode`, `Type`, `Customer Segment`, `Order Status` |
| Khách hàng và thời gian | `Order Customer Id`, `order date (DateOrders)`, `Order Id`, `Sales` |
| Chất lượng giao hàng | `Days for shipping (real)`, `Days for shipment (scheduled)`, `Delivery Status`, `Late_delivery_risk` |

Các cột chứa thông tin cá nhân, cột đã bị mask hoặc cột không mang thông tin (ví dụ tên, email, mật khẩu, URL hình ảnh) sẽ bị loại ở bước tiền xử lý.

## 3. Cấu trúc repository

```
dataco-supply-chain-data-mining/
├── data/
│   ├── raw/                  # Dữ liệu gốc từ Mendeley Data (không được theo dõi bởi git)
│   └── processed/            # Dữ liệu đã làm sạch, transaction mức đơn hàng, đặc trưng mức khách hàng
├── docs/                     # Từ điển dữ liệu, ghi chú phương pháp và báo cáo
├── notebooks/
│   ├── 01_eda_preprocessing.ipynb
│   ├── 02_profit_leakage_rules.ipynb
│   ├── 03_extended_rfm_clustering.ipynb
│   └── 04_linking_rules_and_segments.ipynb
├── result/                   # Luật, nhãn cụm và hình ảnh trực quan hóa
├── .gitignore
├── README.md
└── requirements.txt
```

## 4. Bài toán 1 — Khai phá luật rò rỉ lợi nhuận

### 4.1 Câu hỏi nghiên cứu

Những tổ hợp điều kiện đơn hàng nào đi kèm với đơn hàng bị lỗ, và tổ hợp nào gây ra tổng thiệt hại tài chính lớn nhất?

### 4.2 Điểm khác so với cách làm phổ biến

| Khía cạnh | Cách làm phổ biến | Dự án này |
|---|---|---|
| Mục tiêu | Sản phẩm mua kèm, hoặc giao trễ | Đơn hàng bị lỗ (kết quả kinh doanh, không phải vận hành) |
| Xếp hạng | Support, confidence, lift | Bổ sung **loss coverage**: tổng tiền lỗ của các đơn khớp với luật |
| Kết quả | Một danh sách luật dài | Xếp hạng Pareto các luật gây lỗ kèm phân tích what-if |
| Đánh giá | Một thuật toán | So sánh Apriori và FP-Growth trên cùng một vế phải cố định |

### 4.3 Phương pháp

1. **Gom theo đơn hàng:** Nhóm các dòng theo `Order Id` và tính lợi nhuận của đơn.
2. **Gán nhãn:** `loss = 1` nếu lợi nhuận đơn hàng nhỏ hơn 0.
3. **Rời rạc hóa:** Chia các thuộc tính liên tục (mức giảm giá, doanh số, số lượng) thành các khoảng dễ diễn giải.
4. **Mã hóa transaction:** Mỗi đơn hàng là một transaction gồm các item dạng `thuộc_tính=giá_trị` (one-hot cho biến phân loại và biến đã chia khoảng) cùng item `loss`.
5. **Khai phá:** Chạy Apriori và FP-Growth với `min_support` thấp (đơn lỗ là lớp thiểu số), rồi chỉ giữ các luật có vế phải là `loss`.
6. **Xếp hạng:** Tính support, confidence, lift và **loss coverage** (tổng tiền lỗ của các đơn khớp với vế trái). Xếp hạng theo loss coverage.
7. **Phân tích what-if:** Ước lượng mức cải thiện lợi nhuận nếu giới hạn giảm giá trong các nhóm luật gây lỗ nhiều nhất.
8. **So sánh thuật toán:** So sánh thời gian chạy, bộ nhớ và số luật giữa Apriori và FP-Growth với nhiều giá trị `min_support`.

### 4.4 Sản phẩm đầu ra dự kiến

- Bảng các luật gây lỗ hàng đầu cùng các độ đo nêu trên.
- Biểu đồ Pareto về mức đóng góp lỗ tích lũy theo luật.
- Bảng what-if cho các kịch bản giới hạn giảm giá.
- So sánh thời gian chạy và số luật giữa Apriori và FP-Growth.

### 4.5 Rủi ro đã biết

Lợi nhuận đơn hàng có thể gắn chặt với giá và mức giảm giá, nên một số luật (ví dụ "giảm giá rất cao thì lỗ") có thể hiển nhiên. Bước EDA cần kiểm tra mối quan hệ này trước, và phân tích nên làm nổi bật các luật vượt ra ngoài hiệu ứng giảm giá.

## 5. Bài toán 2 — Phân khúc khách hàng có xét chất lượng dịch vụ

### 5.1 Câu hỏi nghiên cứu

Khi bổ sung chất lượng giao hàng và lợi nhuận vào RFM, xuất hiện những phân khúc khách hàng nào mà RFM thuần không thấy?

### 5.2 Điểm khác so với cách làm phổ biến

| Khía cạnh | Cách làm phổ biến | Dự án này |
|---|---|---|
| Đặc trưng | Chỉ Recency, Frequency, Monetary | RFM cộng thêm đặc trưng chất lượng dịch vụ và lợi nhuận |
| Đánh giá | Chọn k, so sánh thuật toán | Bổ sung so sánh cụm RFM thuần với cụm RFM mở rộng (ARI, bảng chéo) |
| Diễn giải | Hồ sơ cụm dựa trên chi tiêu | Hồ sơ như "khách giá trị cao nhưng hay bị giao trễ" hoặc "khách săn giảm giá, lợi nhuận âm" |

### 5.3 Đặc trưng (mức khách hàng)

| Nhóm | Đặc trưng |
|---|---|
| RFM | Recency (số ngày kể từ đơn gần nhất), Frequency (số đơn), Monetary (tổng doanh số) |
| Chất lượng dịch vụ | Tỷ lệ giao trễ, độ trễ trung bình (số ngày thực tế trừ số ngày dự kiến), tỷ lệ hủy hoặc nghi gian lận |
| Tác động kinh doanh | Mức giảm giá trung bình, tổng lợi nhuận trên mỗi khách |

### 5.4 Phương pháp

1. **Tạo đặc trưng:** Gom theo `Order Customer Id` với một ngày tham chiếu cố định, sau ngày đặt hàng cuối cùng trong dữ liệu.
2. **Tiền xử lý:** Log-transform các biến lệch rồi chuẩn hóa.
3. **K-Means:** Chọn k bằng phương pháp elbow, silhouette score và chỉ số Davies-Bouldin.
4. **DBSCAN:** Chọn `eps` bằng đồ thị k-distance và điều chỉnh `min_samples`. Các điểm nhiễu được phân tích như nhóm khách bất thường.
5. **So sánh với baseline:** Chạy cùng các thuật toán trên RFM thuần, rồi so sánh hai bộ nhãn bằng Adjusted Rand Index và bảng chéo.
6. **Mô tả cụm:** Mô tả từng cụm bằng giá trị trung bình các đặc trưng và diễn giải kinh doanh.

### 5.5 Sản phẩm đầu ra dự kiến

- Các chỉ số chất lượng cụm của K-Means và DBSCAN trên cả hai bộ đặc trưng.
- Bảng chéo cho thấy các phân khúc RFM thuần bị tách ra thế nào khi dùng RFM mở rộng.
- Hồ sơ từng cụm kèm diễn giải kinh doanh.

### 5.6 Rủi ro đã biết

Bộ dữ liệu có thể mang tính tổng hợp, nên các cụm có thể tách yếu (silhouette thấp). Nếu vậy, điều này sẽ được báo cáo trung thực như một kết quả, không che giấu.

## 6. Liên kết hai bài toán

Sau khi phân cụm, mỗi đơn hàng được gán cụm của khách hàng tương ứng. Các luật lỗ của Bài toán 1 sau đó được đánh giá theo từng cụm để trả lời: phân khúc nào đóng góp phần lớn nhất vào tổng tiền lỗ, và các luật hàng đầu có tập trung ở phân khúc cụ thể nào không.

## 7. Lộ trình dự án

- [x] Khởi tạo repository và cấu trúc `README.md`
- [ ] Thiết lập cấu trúc thư mục và `requirements.txt`
- [ ] Phân tích khám phá dữ liệu (EDA) và tiền xử lý
- [ ] Bài toán 1: Khai phá luật rò rỉ lợi nhuận (Apriori và FP-Growth)
- [ ] Bài toán 2: Tạo đặc trưng RFM mở rộng và phân cụm (K-Means và DBSCAN)
- [ ] Phân tích liên kết: luật lỗ theo phân khúc khách hàng
- [ ] Đánh giá kết quả, trực quan hóa và báo cáo cuối cùng

## 8. Cài đặt và sử dụng

### Yêu cầu

Python 3.10+ và các gói trong `requirements.txt`. Danh sách dự kiến:

```
pandas
numpy
scikit-learn
mlxtend
scipy
matplotlib
seaborn
jupyter
```

### Các bước

```bash
# 1. Clone repository
git clone https://github.com/buihuuthien/dataco-supply-chain-data-mining.git
cd dataco-supply-chain-data-mining

# 2. Tạo môi trường và cài đặt thư viện
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

# 3. Tải dataset từ Mendeley Data và đặt tại:
#    data/raw/DataCoSupplyChainDataset.csv

# 4. Chạy các notebook theo thứ tự
jupyter notebook notebooks/
```

> **Lưu ý:** Khi đọc CSV bằng `pandas.read_csv` có thể cần `encoding="latin-1"`.

## 9. Khả năng tái lập

- Cố định random seed cho K-Means và các bước lấy mẫu.
- Giữ toàn bộ tiền xử lý trong notebook để mọi file trong `data/processed/` đều có thể tạo lại từ CSV gốc.
- Lưu luật, nhãn cụm và hình ảnh cuối cùng trong `result/`.
