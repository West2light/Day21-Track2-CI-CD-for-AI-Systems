# Kế Hoạch Triển Khai Bước 2: CI/CD Pipeline & Triển Khai Mô Hình

Dựa trên yêu cầu của `tasks/buoc-2.md` và hiện trạng môi trường của bạn (Windows, chưa cài Cloud SDK), dưới đây là kế hoạch chi tiết từng bước (Milestones) tập trung vào nền tảng **Google Cloud Platform (GCP)** (được khuyến nghị theo mặc định của lab).

---

## Milestone 1: Cài đặt Google Cloud SDK trên Windows
**Mục tiêu:** Cài đặt công cụ dòng lệnh `gcloud` để tương tác với tài nguyên GCP từ máy của bạn.

- **Bước 1.1:** Truy cập trang web chính thức: [Cài đặt Google Cloud CLI cho Windows](https://cloud.google.com/sdk/docs/install#windows) và tải file cài đặt (GoogleCloudSDKInstaller.exe).
- **Bước 1.2:** Chạy file cài đặt, giữ các tùy chọn mặc định (đảm bảo cài đặt cả gói Python đi kèm của Google nếu có, hoặc để gcloud tự cấu hình).
- **Bước 1.3:** Mở PowerShell (có thể cần mở lại một cửa sổ mới để nhận biến môi trường) và chạy lệnh `$ gcloud init`. 
- **Bước 1.4:** Trình duyệt sẽ mở ra, hãy đăng nhập bằng tài khoản Google của bạn và chọn Project sử dụng cho bài lab này. Kiểm tra bằng lệnh: `$ gcloud config get-value project`.

---

## Milestone 2: Thiết lập Bucket và Tài Khoản Dịch Vụ (Service Account)
**Mục tiêu:** Có nơi lưu trữ đám mây cho DVC (Cloud Storage) và khóa truy cập an toàn.

- **Bước 2.1:** Tạo bucket trên GCP (tên duy nhất, ví dụ: `my-mlops-bucket-123`):
  `$ gsutil mb -p <YOUR_PROJECT> -l us-central1 gs://<YOUR_BUCKET_NAME>`
- **Bước 2.2:** Kích hoạt Storage API (nếu chưa):
  `$ gcloud services enable storage.googleapis.com`
- **Bước 2.3:** Tạo Service Account (`mlops-lab-sa`) và cấp quyền `roles/storage.objectAdmin` cho bucket vừa tạo (tham khảo Step 2.2 trong file `buoc-2.md`).
- **Bước 2.4:** Xuất file khóa (key) JSON và lưu dưới dạng `sa-key.json` tại thư mục gốc của project `Day21-Track2-CI-CD-for-AI-Systems`. (Đã có sẵn trong `.gitignore` nên an toàn).

---

## Milestone 3: Cấu hình DVC và Đồng Bộ Dữ Liệu
**Mục tiêu:** Đưa tập dữ liệu Wine Quality lên Cloud Storage thông qua DVC.

- **Bước 3.1:** Khởi tạo DVC (nếu chưa): `$ dvc init`
- **Bước 3.2:** Thêm remote storage trỏ tới GCS bucket:
  `$ dvc remote add -d myremote gs://<YOUR_BUCKET_NAME>/dvc`
- **Bước 3.3:** Khai báo file credential cho DVC:
  `$ dvc remote modify myremote credentialpath sa-key.json`
- **Bước 3.4:** Đẩy file lên Cloud: `$ dvc push`
- **Bước 3.5:** Commit `.dvc/config` và các file `*.csv.dvc` vào git.

---

## Milestone 4: Khởi tạo Máy chủ ảo (VM) và Hoàn thiện mã nguồn API
**Mục tiêu:** Tạo môi trường cho server suy luận và viết code API `src/serve.py`.

- **Bước 4.1:** Dùng `gcloud compute instances create mlops-serve ...` để tạo Ubuntu VM. Tạo cấu hình tường lửa (firewall rule) cho phép cổng 8000.
- **Bước 4.2:** Lấy IP công khai của VM và lưu lại.
- **Bước 4.3:** Viết mã nguồn hoàn chỉnh cho file `src/serve.py`, giải quyết các `TODO` để kết nối GCS tải `model.pkl` và phục vụ model bằng FastAPI.
- **Bước 4.4:** Đẩy file `serve.py` và `sa-key.json` từ Windows lên VM thông qua lệnh `gcloud compute scp`.
- **Bước 4.5:** Truy cập VM bằng `gcloud compute ssh`, cài đặt các package cần thiết, thiết lập systemd service để giữ API Server luôn chạy (run in background).

---

## Milestone 5: Thiết lập CI/CD GitHub Actions
**Mục tiêu:** Cấu hình luồng kiểm thử, huấn luyện tự động và triển khai.

- **Bước 5.1:** Tạo key SSH cho Github Actions (`ssh-keygen` trên Windows) và đưa Public key (file `.pub`) vào `~/.ssh/authorized_keys` của VM.
- **Bước 5.2:** Cấu hình 5 Secrets trên Repo Github: `CLOUD_CREDENTIALS`, `CLOUD_BUCKET`, `VM_HOST`, `VM_USER`, `VM_SSH_KEY` (xem chi tiết step 2.9).
- **Bước 5.3:** Đảm bảo `tests/test_train.py` chạy thành công bằng pytest.
- **Bước 5.4:** Viết và hoàn thiện file `.github/workflows/mlops.yml` (định nghĩa 4 jobs: test, train, eval, deploy).

---

## Milestone 6: Chạy Pipeline & Đánh Giá
**Mục tiêu:** Kích hoạt tự động quá trình và nghiệm thu.

- **Bước 6.1:** Commit toàn bộ setup, pipeline yaml, test và source code lên GitHub `main` branch.
- **Bước 6.2:** Theo dõi Actions tab trên GitHub. Sau khi job train đẩy model lên bucket thả vào `models/latest/model.pkl`, job deploy sẽ khởi động lại API trên VM.
- **Bước 6.3:** Kết nối thử từ local (Windows PowerShell):
  `$ Invoke-WebRequest -Uri http://<VM_IP>:8000/health `
  và dùng `Invoke-RestMethod` hoặc `curl` để POST nghiệm thu hàm `/predict`.
