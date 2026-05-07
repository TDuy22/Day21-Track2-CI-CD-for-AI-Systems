# AWS setup cho lab MLOps

File này ghi lại phần cần cấu hình thủ công khi dùng AWS thay cho Google Cloud.

## 1. Chọn region và bucket

Mục đích: S3 bucket dùng làm DVC remote và nơi lưu model mới nhất.

```powershell
$env:AWS_REGION="ap-southeast-1"
$env:BUCKET="mlops-lab-phamthanhduy-20260507"

aws s3 mb s3://$env:BUCKET --region $env:AWS_REGION
```

Nếu dùng region `us-east-1`, AWS CLI có thể yêu cầu cú pháp bucket khác:

```powershell
aws s3 mb s3://$env:BUCKET --region us-east-1
```

## 2. IAM permissions

Mục đích: GitHub Actions và DVC cần quyền đọc/ghi object trong bucket.

Policy tối thiểu cho IAM user/role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::YOUR_BUCKET"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::YOUR_BUCKET/*"
    }
  ]
}
```

Thay `YOUR_BUCKET` bằng tên bucket thật.

## 3. DVC remote

Mục đích: version dữ liệu bằng DVC nhưng lưu file thật trên S3.

```powershell
dvc init
dvc remote add -d myremote s3://$env:BUCKET/dvc
dvc add data/train_phase1.csv
dvc add data/eval.csv
dvc add data/train_phase2.csv
dvc push
```

Trong repo này remote đã được cấu hình tới:

```text
s3://mlops-lab-phamthanhduy-20260507/dvc
```

Commit các file pointer, không commit CSV:

```powershell
git add .dvc/config data/train_phase1.csv.dvc data/eval.csv.dvc data/train_phase2.csv.dvc .gitignore
git commit -m "feat: track datasets with DVC on S3"
```

## 4. EC2 setup

Mục đích: EC2 chạy FastAPI inference server.

Cài package trên EC2:

```bash
sudo apt update
sudo apt install -y python3-pip
pip3 install fastapi uvicorn scikit-learn joblib boto3 pandas
mkdir -p ~/models ~/src
```

Khuyến nghị: gắn IAM role cho EC2 có quyền `s3:GetObject` với bucket. Nếu chưa dùng IAM role, cấu hình AWS credentials trên EC2 bằng `aws configure`.

Copy `src/serve.py` lên EC2:

```powershell
scp -i <path-to-ec2-key.pem> src/serve.py ubuntu@<EC2_PUBLIC_IP>:~/src/serve.py
```

Tạo service trên EC2:

```bash
sudo tee /etc/systemd/system/mlops-serve.service > /dev/null <<EOF
[Unit]
Description=MLOps Model Inference Server
After=network.target

[Service]
User=$USER
WorkingDirectory=/home/$USER
Environment="S3_BUCKET=YOUR_BUCKET"
Environment="AWS_DEFAULT_REGION=YOUR_REGION"
ExecStart=/usr/bin/python3 /home/$USER/src/serve.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable mlops-serve
```

Thay `YOUR_BUCKET` và `YOUR_REGION` bằng giá trị thật.

## 5. GitHub Secrets

Mục đích: GitHub Actions cần credentials để pull DVC data, upload model và SSH deploy.

Thêm các secrets sau trong repo GitHub:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_REGION`
- `CLOUD_BUCKET`
- `VM_HOST`
- `VM_USER`
- `VM_SSH_KEY`

`CLOUD_BUCKET` là tên S3 bucket, không có tiền tố `s3://`.

Với bucket hiện tại:

```text
AWS_REGION=ap-southeast-1
CLOUD_BUCKET=mlops-lab-phamthanhduy-20260507
```

`VM_HOST` là public IP hoặc DNS của EC2.

`VM_USER` thường là `ubuntu` nếu dùng Ubuntu AMI.

`VM_SSH_KEY` là private key mà GitHub Actions dùng để SSH vào EC2.

## 6. Security group

Mục đích: cho phép gọi API từ ngoài internet.

Inbound rules cần có:

- TCP 22 từ IP của bạn hoặc GitHub Actions nếu muốn giới hạn chặt.
- TCP 8000 từ IP của bạn, hoặc `0.0.0.0/0` cho lab.

## 7. Kiểm tra

Sau khi pipeline upload model và deploy:

```bash
curl http://<EC2_PUBLIC_IP>:8000/health
curl -X POST http://<EC2_PUBLIC_IP>:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"features": [7.4, 0.70, 0.00, 1.9, 0.076, 11.0, 34.0, 0.9978, 3.51, 0.56, 9.4, 0]}'
```
