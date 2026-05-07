# REFLECTION – Lab MLOps: CI/CD for AI Systems

**Họ và tên:** Dương Quang Đông  
**MSSV:** 2A202600445  
**Course:** AIInAction – VinUni | Day 21 – CI/CD cho AI Systems

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

Mô hình sử dụng: **RandomForestClassifier** (scikit-learn)

| Tham số | Giá trị | Lý do |
|---|---|---|
| `n_estimators` | 1000 | Tăng số cây để giảm phương sai, cải thiện độ ổn định dự đoán |
| `max_depth` | null (không giới hạn) | Cho phép cây học sâu, phù hợp với dữ liệu có nhiều đặc trưng tương tác |
| `criterion` | entropy | Information Gain thường cho kết quả tốt hơn Gini trên bài toán phân loại đa lớp |
| `max_features` | sqrt | Giảm tương quan giữa các cây, chuẩn tắc của Random Forest |
| `class_weight` | balanced | Xử lý mất cân bằng nhãn (imbalanced classes) trong tập dữ liệu wine quality |
| `min_samples_split` | 2 | Giá trị mặc định, cho phép cây phát triển linh hoạt |
| `min_samples_leaf` | 1 | Giá trị mặc định, không hạn chế kích thước lá |

### Kết quả thực nghiệm (MLflow)

| Lần chạy | Dữ liệu | Accuracy | F1-score |
|---|---|---|---|
| Bước 2 (`indecisive-fox-460`) | 2998 mẫu (phase 1) | 0.67 | 0.67 |
| Bước 3 (`placid-grub-633`) | 5996 mẫu (phase 1 + phase 2) | **0.75** | **0.75** |

Kết quả cho thấy việc **bổ sung dữ liệu** (`train_phase2.csv`) có tác động lớn hơn so với tinh chỉnh tham số: accuracy tăng từ 0.67 lên 0.75 chỉ bằng cách tăng gấp đôi lượng dữ liệu huấn luyện.

---

## 2. Khó Khăn Gặp Phải và Cách Giải Quyết

### 2.1 Lệnh Bash không tương thích với PowerShell (Windows)

**Vấn đề:** Các lệnh trong hướng dẫn dùng cú pháp Linux (`export VAR=value`, `touch file`) không chạy được trên PowerShell.

**Giải pháp:** Chuyển đổi sang cú pháp PowerShell:
- `export VAR=value` → `$env:VAR = "value"`
- `touch file` → `New-Item -ItemType File -Path file -Force`

---

### 2.2 DVC pull thất bại với lỗi 401 (Invalid Credentials)

**Vấn đề:** File `.dvc/config` có `credentialpath = sa-key.json` (đường dẫn tương đối), nhưng trong GitHub Actions runner file này không tồn tại, dù biến môi trường `GOOGLE_APPLICATION_CREDENTIALS=/tmp/sa-key.json` đã được set đúng.

**Giải pháp:** Xóa `credentialpath` khỏi `.dvc/config` bằng lệnh:
```bash
dvc remote modify myremote --unset credentialpath
```
DVC sau đó tự động đọc `GOOGLE_APPLICATION_CREDENTIALS` từ biến môi trường, credential được thiết lập đúng trong workflow.

---

### 2.3 Job Deploy fail do health check timeout quá ngắn

**Vấn đề:** Script deploy chỉ đợi 5 giây (`sleep 5`) rồi gọi `curl /health`, nhưng server mất ~16 giây để tải mô hình Random Forest (1000 cây) và khởi động Uvicorn.

**Giải pháp:** Thay thế `sleep 5` bằng vòng lặp retry (10 lần, mỗi lần 5 giây):
```bash
for i in {1..10}; do
  curl -sf http://localhost:8000/health && echo "Health check passed." && exit 0
  sleep 5
done
exit 1
```

---

### 2.4 `pkg_resources` không tồn tại với Python 3.12

**Vấn đề:** `pytest` báo lỗi `ModuleNotFoundError: No module named 'pkg_resources'` vì `setuptools >= 74` đã tách `pkg_resources` ra.

**Giải pháp:** Hạ phiên bản setuptools:
```bash
pip install "setuptools<74"
```

---

## 3. Tổng Kết

Pipeline CI/CD hoàn chỉnh đã được thiết lập và hoạt động ổn định với 4 jobs tự động: **Unit Test → Train → Eval (ngưỡng ≥ 0.70) → Deploy**. Mô hình cuối đạt accuracy **0.75**, vượt ngưỡng deploy và được phục vụ qua REST API (FastAPI) trên Google Cloud VM.
