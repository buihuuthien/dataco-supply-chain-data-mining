# dataco-supply-chain-data-mining

Khai phá luật về cam kết giao hàng và giao trễ (Apriori, FP-Growth) và phân khúc khách hàng có xét chất lượng dịch vụ (RFM mở rộng với K-Means, DBSCAN) trên bộ dữ liệu DataCo Smart Supply Chain.

## 1. Tổng quan dự án

Dự án áp dụng các kỹ thuật khai phá dữ liệu lên dữ liệu giao dịch bán lẻ và chuỗi cung ứng. Thay vì lặp lại các phân tích phổ biến, dự án đặt lại hai bài toán xoay quanh **vận hành cam kết giao hàng** và **phân khúc hành vi rủi ro của khách hàng**.

- **Bài toán 1 — Khai phá luật cam kết giao hàng và giao trễ (Luật kết hợp):** Tìm các tổ hợp điều kiện vận hành thường đi kèm đơn hàng bị giao trễ (`late`), đồng thời phân tích nghịch lý tỷ lệ trễ cao ở các dịch vụ vận chuyển cao cấp.
- **Bài toán 2 — Phân khúc hành vi rủi ro và rò rỉ lợi nhuận (Phân cụm):** Mở rộng RFM truyền thống bằng cách tích hợp các đặc trưng về tỷ lệ đơn lỗ, mức giảm giá, và rủi ro giao hàng trễ, sau đó áp dụng K-Means và DBSCAN để phân loại khách hàng.

## 2. Bộ dữ liệu

- **Tên:** DataCo SMART SUPPLY CHAIN FOR BIG DATA ANALYSIS
- **Nguồn:** [Mendeley Data (DOI: 10.17632/8gx2fvg2k6.5)](https://doi.org/10.17632/8gx2fvg2k6.5)
- **Quy mô:** 180.519 bản ghi, 53 thuộc tính, 65.752 đơn hàng và 20.652 khách hàng (2015–2018).
- **Mức chi tiết:** Mỗi dòng là một order item, nên một đơn hàng có thể gồm nhiều dòng. Phân tích ở mức đơn hàng cần gom theo `Order Id`, phân tích ở mức khách hàng cần gom theo `Order Customer Id`.
- **Lưu ý:** File CSV gốc bị loại khỏi repo bằng `.gitignore` do giới hạn 100 MB của GitHub. Hãy tải `DataCoSupplyChainDataset.csv` từ Mendeley Data và đặt vào `data/raw/`.

### Các cột chính được sử dụng

| Mục đích | Cột |
|---|---|
| Nhãn rủi ro và lợi nhuận | `Order Profit Per Order`, `Order Item Profit Ratio`, `Late_delivery_risk`, `Delivery Status` |
| Điều kiện đơn hàng | `Order Item Discount Rate`, `Category Name`, `Market`, `Order Region`, `Shipping Mode`, `Type`, `Customer Segment`, `Order Status` |
| Khách hàng và thời gian | `Order Customer Id`, `order date (DateOrders)`, `Order Id`, `Sales` |
| Chất lượng giao hàng | `Days for shipping (real)`, `Days for shipment (scheduled)`, `shipping date (DateOrders)` |

Các cột chứa thông tin cá nhân hoặc không mang thông tin sẽ bị loại ở bước tiền xử lý.

## 3. Cấu trúc repository

```
dataco-supply-chain-data-mining/
├── data/
│   ├── raw/                  # Dữ liệu gốc từ Mendeley Data (không được theo dõi bởi git)
│   └── processed/            # Dữ liệu đã làm sạch, transaction mức đơn hàng, đặc trưng mức khách hàng
├── docs/                     # Từ điển dữ liệu, ghi chú phương pháp và báo cáo
├── notebooks/
│   ├── 01_eda_preprocessing.ipynb
│   ├── 02_delivery_promise_rules.ipynb
│   ├── 03_extended_rfm_clustering.ipynb
│   └── 04_linking_rules_and_segments.ipynb
├── result/                   # Luật, nhãn cụm và hình ảnh trực quan hóa
├── .gitignore
├── README.md
└── requirements.txt
```

## 4. Bài toán 1 — Khai phá luật cam kết giao hàng và giao trễ

### 4.1 Câu hỏi nghiên cứu

Những điều kiện vận hành nào (hình thức vận chuyển, khu vực, thị trường, danh mục) thường đi kèm với việc đơn hàng bị giao trễ (`late`)?

### 4.2 Thực trạng vận hành và nghịch lý dịch vụ

- Tỷ lệ giao trễ phụ thuộc rất mạnh vào hình thức vận chuyển: First Class trễ 95%, Second Class 77%, Same Day 46%, Standard Class 38%.
- **Điểm bất thường quan trọng:** Sau khi lọc bỏ các đơn bị hủy (`Shipping canceled`), toàn bộ các đơn hàng sử dụng phương thức `First Class` còn lại đều bị giao trễ thực tế so với lịch trình cam kết (tỷ lệ trễ thực tế gần như 100% đối với các đơn hợp lệ). Nhãn `Late_delivery_risk` được suy ra từ ngày thực tế so với ngày cam kết, dẫn đến luật "thực tế > cam kết ⇒ trễ" là hiển nhiên và được làm sạch/xử lý phù hợp trong quá trình khai phá.

### 4.3 Phương pháp

1. **Gom theo đơn hàng:** Nhóm các dòng theo `Order Id` và xác định nhãn giao trễ (`late`) cho đơn hàng (loại bỏ các đơn `Shipping canceled`).
2. **Rời rạc hóa:** Chia các thuộc tính liên tục (mức giảm giá, doanh số) thành các khoảng dễ diễn giải.
3. **Mã hóa transaction:** Mỗi đơn hàng là một transaction gồm các item dạng `thuộc_tính=giá_trị` cùng item `late`.
4. **Khai phá:** Chạy Apriori và FP-Growth với `min_support` phù hợp, chỉ giữ các luật có vế phải là `late`.
5. **Xếp hạng:** Tính support, confidence, lift và số lượng đơn trễ được giải thích.
6. **Phân tích What-if:** Đánh giá các kịch bản điều chỉnh thời gian cam kết vận chuyển (ví dụ: mở rộng thời gian cam kết của First Class từ 1 lên 2 ngày để hạ tỷ lệ vi phạm cam kết).

### 4.4 Các hướng đã thử nghiệm nhưng không có tín hiệu (Ghi nhận trung thực)
- **Rò rỉ lợi nhuận (Lỗ tài chính):** Tỷ lệ đơn lỗ ở mức item là 18,7% và mức đơn là 21,2%. Tuy nhiên, kiểm định chi-square cho mọi thuộc tính đều có p > 0,05, Cramér’s V ≤ 0,025 và hệ số tương quan giữa giảm giá với tỷ suất lợi nhuận chỉ là −0,003. Lợi nhuận gần như độc lập với các điều kiện đơn hàng, do đó hướng khai phá luật lỗ không đem lại tín hiệu tin cậy.
- **Market Basket Analysis (Sản phẩm mua kèm):** Không tạo ra các luật kết hợp sản phẩm có lift nổi bật đáng kể so với mức ngẫu nhiên (lift trung vị xấp xỉ 1,04).

## 5. Bài toán 2 — Phân cụm hành vi rủi ro và rò rỉ lợi nhuận của khách hàng

### 5.1 Câu hỏi nghiên cứu

Khi mở rộng RFM bằng cách tích hợp các đặc trưng về rủi ro tài chính (tỷ lệ đơn lỗ, mức giảm giá) và chất lượng dịch vụ (tỷ lệ trễ), ta thu được những phân khúc khách hàng nào mà RFM thuần không thể phát hiện?

### 5.2 Đặc trưng (mức khách hàng)

| Nhóm | Đặc trưng |
|---|---|
| RFM chuẩn | Recency, Frequency, Monetary (đã log-transform) |
| Rủi ro & Vận hành | Tỷ lệ đơn lỗ (`Loss_Order_Ratio`), mức giảm giá trung bình (`Avg_Discount_Rate`), tỷ lệ rủi ro giao trễ (`Late_Risk_Ratio`) |

### 5.3 Phương pháp

1. **Tách nhóm khách hàng:** Xử lý riêng nhóm khách mua một lần (chiếm khoảng 43% tổng số khách) vì tần suất $F=1$ chi phối mạnh mô hình RFM.
2. **Tiền xử lý:** Chuẩn hóa các biến đặc trưng (loại bỏ tương quan cơ học giữa M và tổng lợi nhuận).
3. **Phân cụm K-Means & DBSCAN:** Áp dụng trên nhóm khách quay lại ($F \ge 2$ hoặc $F \ge 3$) để tìm ra các phân khúc hành vi đặc thù (như nhóm săn sale gây lỗ, nhóm rủi ro vận hành cao, nhóm khách hàng chuẩn mực).
4. **Kiểm định thống kê & So sánh:** So sánh với kết quả RFM thuần bằng chỉ số Adjusted Rand Index (ARI) và kiểm định permutation test để chứng minh sự khác biệt cấu trúc cụm.

## 6. Liên kết hai bài toán

Sau khi hoàn thành phân cụm khách hàng ở Bài toán 2, các luật giao trễ từ Bài toán 1 được đối chiếu theo từng phân khúc khách hàng để xem nhóm khách hàng nào chịu ảnh hưởng hoặc đóng góp lớn nhất vào rủi ro vận hành của hệ thống.

## 7. Cài đặt và sử dụng

### Yêu cầu

Python 3.10+ và các gói trong `requirements.txt`:
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

### Các bước thực hiện

```bash
# 1. Clone repository
git clone https://github.com/buihuuthien/dataco-supply-chain-data-mining.git
cd dataco-supply-chain-data-mining

# 2. Tạo môi trường và cài đặt thư viện
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

# 3. Đặt file dữ liệu tại: data/raw/DataCoSupplyChainDataset.csv

# 4. Chạy các notebook
jupyter notebook notebooks/
```

> **Lưu ý:** Khi đọc file CSV bằng `pandas.read_csv`, nhớ sử dụng `encoding="latin-1"`.
