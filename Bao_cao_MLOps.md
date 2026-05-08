# BAO CAO THUC HANH MLOps: CI/CD CHO HE THONG AI

**Ho va ten:** Phạm Thành Duy

**Ma hoc vien:** 2A202600267

---

## 1. Lua chon bo sieu tham so tot nhat (Ket qua Buoc 1)

Trong qua trinh thuc nghiem cuc bo voi MLflow tren tap du lieu goc
(2998 mau), em da thu nghiem nhieu cau hinh sieu tham so khac nhau
cho mo hinh RandomForestClassifier. Sau khi phan tich va so sanh
tren MLflow UI, em da quyet dinh chon bo sieu tham so sau luu vao
params.yaml:

- n_estimators: 300
- max_depth: 20
- min_samples_split: 2

**Ly do chon:**

Bo tham so nay mang lai su can bang tot nhat giua kha nang hoc
(tranh underfitting) va kha nang tong quat hoa. Voi so luong cay
quyet dinh lon (300) va do sau cay vua du (20), mo hinh cho ket
qua on dinh va cao nhat tren tap danh gia (held-out eval set).

Mac du tren tap du lieu goc (2998 mau), accuracy cao nhat chi dat
xap xi 0.678. Tuy nhien, khi bo sung them du lieu moi o Buoc 3
(gop len 5996 mau), Accuracy da dat 0.756 va F1 Score dat 0.755,
vuot qua nguong chat luong an toan (>= 0.70) de cho phep CI/CD
deploy. Dieu nay chung minh them du lieu cai thien dang ke hieu
qua mo hinh.

---

## 2. So sanh ket qua Buoc 2 va Buoc 3

| Chi so    | Buoc 1 - cuc bo (2998 mau) | Buoc 3 - CI/CD (5996 mau) |
|-----------|---------------------------|---------------------------|
| Accuracy  | ~0.678                    | 0.756                     |
| F1 Score  | ~0.676                    | 0.755                     |

**Nhan xet:** Bo sung them 2998 mau du lieu moi (train_phase2) giup
mo hinh nang cao hieu suat dang ke (accuracy tang tu 0.678 len
0.756, tuong duong +11.5%). Eval gate (accuracy >= 0.70) trong
GitHub Actions chi cho phep deploy thanh cong khi mo hinh duoc
huan luyen tren tap du lieu lon hon. Day la hanh vi dung nhu mong
doi cua he thong CI/CD.

---

## 3. Kho khan gap phai va cach giai quyet

Trong qua trinh thuc hien bai Lab, em da gap mot so van de va da
xu ly nhu sau:

### Kho khan 1: Chuyen doi kien truc tu GCP sang AWS

- Van de: Bai lab huong dan cau hinh mac dinh bang GCP (Google
  Cloud), nhung em lua chon trien khai thuc te tren he sinh thai
  AWS (S3 + EC2), yeu cau su thay doi trong cach xac thuc DVC va
  viet pipeline Actions.
- Giai quyet: Em da thiet lap IAM Credentials an toan tren AWS.
  Doi voi DVC, em doi remote sang dang s3://... (su dung package
  dvc[s3]). Trong GitHub Actions, em su dung Action
  aws-actions/configure-aws-credentials@v4 thay cho GCP SA key,
  sau do su dung boto3 trong file serve.py va pipeline de tuong
  tac voi bucket S3 mot cach muot ma.

### Kho khan 2: Trien khai bao mat cao thong qua EC2 Instance Connect

- Van de: Viec su dung SSH key tinh (long-lived SSH key) de CI/CD
  truy cap vao VM tiem an rui ro bao mat neu lo Key tren GitHub
  Secrets.
- Giai quyet: Em da nang cap co che Deploy (Job 4) bang cach ap
  dung EC2 Instance Connect (aws ec2-instance-connect
  send-ssh-public-key). Pipeline se tu dong sinh ma SSH key dung
  mot lan (ed25519) ngay luc chay, day public key thang vao EC2
  instance, va thuc hien restart service mlops-serve. Cach nay
  giup qua trinh trien khai hoan toan tu dong ma van dat chuan
  bao mat ha tang Cloud.

### Kho khan 3: Accuracy chua vuot nguong tren tap du lieu goc

- Van de: Khi huan luyen cuc bo (Buoc 1) tren 2998 mau, accuracy
  chua dat nguong 0.70 (chi o muc ~0.68) de deploy. Do do MLflow
  chi luu cac run co acc ~0.68.
- Giai quyet: Loi nay chung minh tam quan trong cua vong lap ML.
  Khi em gia lap Buoc 3 (them du lieu, tong cong 5996 mau) va
  huan luyen lai, model moi vuot nguong (accuracy 0.756) va da
  duoc deploy tu dong thong qua CI/CD. Day la minh chung ro nhat
  cho gia tri cua quy trinh huan luyen lien tuc (Continuous
  Training) khi co them du lieu.
