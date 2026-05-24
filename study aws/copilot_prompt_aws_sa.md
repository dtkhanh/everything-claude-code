# 📚 Prompt Template cho GitHub Copilot - AWS Solutions Architect Associate

## Hướng dẫn sử dụng:
1. Copy từng phần nội dung từ Notion của bạn
2. Paste vào GitHub Copilot kèm theo prompt tương ứng
3. Copilot sẽ viết lại thành ví dụ cụ thể, chi tiết, dễ học

---

## 🎯 PROMPT CHÍNH (Sử dụng cho mỗi phần chuyên đề)

```
Tôi đang học AWS Solutions Architect Associate. Đây là nội dung từ bài học:

[PASTE NỘI DUNG TỪ NOTION TẠI ĐÂY]

Hãy giúp tôi:
1. 📝 Viết lại phần này bằng các ví dụ THỰC TẾ, cụ thể
2. 💡 Giải thích từng khái niệm bằng kinh doanh/ngữ cảnh thực tế
3. 🔧 Tạo mã ví dụ (Python/CLI) nếu có thể áp dụng
4. ⚠️ Liệt kê các lỗi thường gặp và cách tránh
5. 📊 Tạo bảng so sánh các tùy chọn
6. ✅ Viết câu hỏi ôn tập để kiểm tra hiểu biết

Format output: Markdown với đầu mục rõ ràng
Ngôn ngữ: Tiếng Việt
Độ chi tiết: Trung cấp (không quá cơ bản, không quá nâng cao)
```

---

## 🏢 PROMPT CHO PHẦN EC2 (Compute)

```
Nội dung AWS cần học:
[PASTE PHẦN EC2 TỪ NOTION]

Hãy tạo một bài học chi tiết về EC2 với:

**1. Định nghĩa đơn giản:**
- Giải thích EC2 như là gì bằng 2-3 câu

**2. Các loại instance:**
- Liệt kê 5 loại chính (t2, m5, c5, r5, g4)
- Khi nào dùng loại nào?

**3. Ví dụ thực tế:**
```bash
# Ví dụ 1: Tạo web server đơn giản
aws ec2 run-instances --image-id ami-0c55b159cbfafe1f0 --count 1 --instance-type t2.micro

# Ví dụ 2: Tạo database server
aws ec2 run-instances --image-id ami-0c55b159cbfafe1f0 --count 1 --instance-type r5.large
```

**4. So sánh (Bảng):**
| Khía cạnh | On-Premises | EC2 | Chi phí |
|-----------|------------|-----|--------|
| Setup | Tuần | Phút | Cao |
| Bảo trì | Tự | AWS | Thấp |

**5. Lỗi thường gặp:**
- Quên đóng instance (tiền điện lãng phí)
- Chọn nhầm AZ (latency cao)
- Không backup EBS volume

**6. Câu hỏi ôn tập:**
- Câu 1: ...?
- Câu 2: ...?
```

Viết đơn giản, dễ hiểu, không dùng thuật ngữ quá phức tạp.
```

---

## 💾 PROMPT CHO PHẦN STORAGE (S3, EBS, EFS)

```
Nội dung AWS cần học:
[PASTE PHẦN STORAGE TỪ NOTION]

Hãy giải thích 3 dịch vụ lưu trữ chính với:

**1. S3 (Simple Storage Service):**
- Là gì? (1 câu)
- Khi nào dùng? (ví dụ: ảnh web, backup file)
- Mã Python ví dụ:
```python
import boto3
s3 = boto3.client('s3')
# Upload file
s3.upload_file('local.txt', 'my-bucket', 'remote.txt')
# Download file
s3.download_file('my-bucket', 'remote.txt', 'local.txt')
```

**2. EBS (Elastic Block Storage):**
- Là gì? (1 câu)
- Giống cứng trong máy tính như thế nào?
- Khi nào dùng? (database, application files)
- Lệnh gắn EBS:
```bash
# Attach volume to EC2
aws ec2 attach-volume --volume-id vol-1234567 --instance-id i-1234567 --device /dev/sdf
```

**3. EFS (Elastic File System):**
- Là gì? (1 câu)
- Khác gì so với EBS?
- Khi nào dùng? (nhiều EC2 cùng truy cập)

**So sánh 3 dịch vụ:**
| Tiêu chí | S3 | EBS | EFS |
|---------|-----|-----|-----|
| Loại | Object | Block | File |
| Tốc độ | Chậm | Nhanh | Trung bình |
| Chia sẻ | Public có | Không | Có (NFS) |
| Giá | Rẻ | Trung bình | Đắt |

Viết dễ hiểu, dùng phép so sánh thực tế.
```

---

## 🌐 PROMPT CHO PHẦN NETWORKING (VPC, Security Groups)

```
Nội dung AWS cần học:
[PASTE PHẦN NETWORKING TỪ NOTION]

Hãy tạo hướng dẫn VPC & Security Groups với:

**1. VPC (Virtual Private Cloud) là gì?**
- Giới thiệu: VPC = mạng riêng ảo trong AWS
- Diagram nền (dạng ASCII):
```
┌─────────────────────────────────────┐
│         VPC (10.0.0.0/16)           │
├─────────────────────────────────────┤
│  ┌────────────┐    ┌────────────┐   │
│  │ Subnet 1   │    │ Subnet 2   │   │
│  │(10.0.1.0)  │    │(10.0.2.0)  │   │
│  └────────────┘    └────────────┘   │
└─────────────────────────────────────┘
```

**2. Security Group là gì?**
- Như tường lửa cho từng EC2
- Luật Inbound (vào) & Outbound (ra)

**3. Ví dụ thực tế:**
```bash
# Tạo Security Group
aws ec2 create-security-group --group-name web-sg --description "Web server SG"

# Cho phép HTTP (port 80)
aws ec2 authorize-security-group-ingress --group-id sg-123456 \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# Cho phép HTTPS (port 443)
aws ec2 authorize-security-group-ingress --group-id sg-123456 \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

# Cho phép SSH từ IP cụ thể
aws ec2 authorize-security-group-ingress --group-id sg-123456 \
  --protocol tcp --port 22 --cidr 203.0.113.0/32
```

**4. Các lỗi thường gặp:**
- Quên mở port 22 (không vào được EC2 qua SSH)
- Cho phép tất cả traffic (0.0.0.0/0) không an toàn
- Không hiểu sự khác nhau Subnet & Security Group

**5. Câu hỏi kiểm tra:**
- CIDR 10.0.0.0/16 có bao nhiêu IP?
- Security Group nào được áp dụng cho instance?
```

Giải thích rõ, dùng diagram & ví dụ CLI.
```

---

## 🗄️ PROMPT CHO PHẦN DATABASE (RDS, DynamoDB)

```
Nội dung AWS cần học:
[PASTE PHẦN DATABASE TỪ NOTION]

Hãy so sánh 2 loại database chính:

**1. RDS (Relational Database Service):**
- Là gì? Database SQL được quản lý (MySQL, PostgreSQL, Oracle...)
- Ưu điểm:
  - AWS quản lý backup, patch, replication
  - Không lo về infrastructure
  - Multi-AZ cho high availability

- Mã ví dụ:
```bash
# Tạo RDS instance
aws rds create-db-instance --db-instance-identifier mydb \
  --db-instance-class db.t2.micro \
  --engine postgres \
  --allocated-storage 20
```

**2. DynamoDB:**
- Là gì? NoSQL database (dạng key-value)
- Khi nào dùng?
  - Mobile apps (sync offline)
  - Real-time data (chat, scores)
  - Cần auto-scaling
  
- Mã ví dụ (Python):
```python
import boto3
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

# Lưu user
table.put_item(Item={'UserID': '123', 'Name': 'John', 'Email': 'john@example.com'})

# Lấy user
response = table.get_item(Key={'UserID': '123'})
print(response['Item'])

# Query
response = table.query(KeyConditionExpression='UserID = :id', ExpressionAttributeValues={':id': '123'})
```

**3. So sánh chi tiết:**
| Tính năng | RDS | DynamoDB |
|-----------|-----|----------|
| Cấu trúc | SQL (bảng) | NoSQL (key-value) |
| Join | Có | Không |
| Giá | Theo giờ | Theo request |
| Scaling | Thủ công | Tự động |
| Học | Dễ | Khó |
| Real-time | Chậm | Nhanh |

**4. Khi nào dùng cái nào?**
- RDS: Website truyền thống, phức tạp query, quan hệ dữ liệu
- DynamoDB: Mobile app, real-time, không cần join

Viết dễ hiểu, dùng ví dụ code thực tế.
```

---

## ⚡ PROMPT CHO PHẦN SCALABILITY & LOAD BALANCING

```
Nội dung AWS cần học:
[PASTE PHẦN SCALABILITY TỪ NOTION]

Hãy giải thích cách làm web app chịu tải tốt:

**1. Vấn đề:**
- Website bạn có 100 user → OK
- Tăng lên 10,000 user → Chết!
- Cần scaling (mở rộng)

**2. 2 cách scaling:**

**Vertical Scaling (nâng cấp máy):**
```
t2.micro → t2.small → t2.medium
(dễ nhưng có giới hạn)
```

**Horizontal Scaling (thêm máy):**
```
1 máy chủ → 3 máy chủ → 10 máy chủ
(tốt hơn, nhưng phức tạp)
```

**3. Giải pháp AWS:**

**a) Auto Scaling Group:**
```bash
# Tạo ASG
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --min-size 2 --max-size 10 --desired-capacity 5 \
  --availability-zones us-east-1a us-east-1b
```

**b) Elastic Load Balancer (ELB):**
```
┌──────────────┐
│ User Request │
└──────┬───────┘
       ↓
   ┌─────────┐
   │ ELB     │ (phân phối tải)
   └─────────┘
   ↙    ↓    ↘
┌────┐┌────┐┌────┐
│EC2 ││EC2 ││EC2 │
└────┘└────┘└────┘
```

**4. Ví dụ đơn giản:**
```bash
# Tạo Load Balancer
aws elbv2 create-load-balancer \
  --name web-lb \
  --subnets subnet-123 subnet-456 \
  --security-groups sg-789

# Tạo Target Group
aws elbv2 create-target-group \
  --name web-targets \
  --protocol HTTP --port 80 \
  --vpc-id vpc-123
```

**5. Lỗi thường gặp:**
- Chỉ scale lên, không scale xuống (tiền lãng phí)
- Health check không đúng (request đi tới server chết)
- Không test session replication

**6. Câu hỏi:**
- Khi nào nên dùng Auto Scaling?
- ELB và Security Group khác gì?
- Session data lưu ở đâu khi scale?

Viết đơn giản, dùng diagram & flow.
```

---

## 🔒 PROMPT CHO PHẦN SECURITY & IAM

```
Nội dung AWS cần học:
[PASTE PHẦN SECURITY TỪ NOTION]

Hãy giải thích AWS Security (IAM, Encryption):

**1. IAM (Identity Access Management):**
- Là gì? Kiểm soát ai có quyền làm gì
- Root account (siêu user) ≠ IAM user

**2. Ví dụ thực tế:**
```
Công ty "TechCorp":
├─ CEO (Admin - toàn quyền)
├─ Kỹ sư A (chỉ deploy EC2)
├─ Kỹ sư B (chỉ quản lý database)
└─ Intern (chỉ xem logs)

→ Mỗi người 1 IAM user với quyền khác nhau
```

**3. Tạo IAM User:**
```bash
# Tạo user
aws iam create-user --user-name engineer-a

# Gán policy (quyền)
aws iam attach-user-policy --user-name engineer-a \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

# Tạo access key
aws iam create-access-key --user-name engineer-a
```

**4. IAM Policy (JSON):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": "ec2:TerminateInstances",
      "Resource": "*"
    }
  ]
}
```
(Cho phép tất cả EC2, nhưng cấm terminate)

**5. Encryption:**
- S3: SSE-S3, SSE-KMS
- EBS: Mã hóa volume
- RDS: Enable encryption

**6. Best Practices:**
- ✅ Dùng IAM user, không dùng root
- ✅ Mỗi user 1 quyền cụ thể (Principle of Least Privilege)
- ✅ Rotate access key định kỳ
- ✅ Enable MFA (2FA)
- ❌ Không share credentials

**7. Câu hỏi:**
- Root account dùng để làm gì?
- Ai nên có quyền terminate EC2?
```

Giải thích an toàn, dùng ví dụ policy thực tế.
```

---

## 📊 PROMPT TỔNG HỢP (Sau khi học xong)

```
Tôi đã học xong AWS Solutions Architect Associate với các phần:
[LIỆT KÊ TẤT CẢ PHẦN ĐÃ HỌC]

Hãy giúp tôi:
1. Tạo một project AWS thực tế kết hợp tất cả dịch vụ
2. Vẽ kiến trúc (ASCII diagram)
3. Viết step-by-step cách setup
4. Tính chi phí ước tính
5. Liệt kê các lỗi cần tránh
6. Tạo 50 câu hỏi trắc nghiệm

Mục tiêu: Chuẩn bị cho kỳ thi SAA
```

---

## 🎓 MẸO DÙNG COPILOT HIỆU QUẢ

### 1. **Prompt Chi Tiết**
- ❌ Tệ: "Giải thích EC2"
- ✅ Tốt: "Giải thích EC2 kèm ví dụ CLI, diagram, lỗi thường gặp, so sánh"

### 2. **Context Đủ**
- Luôn cho biết bạn đang học gì, mục tiêu là gì
- Ví dụ: "Tôi học AWS SAA, chuẩn bị thi tháng 6"

### 3. **Yêu cầu Format**
- Markdown
- Sections rõ ràng (heading, bullet points)
- Code blocks với comment

### 4. **Hỏi Follow-up**
- "Giải thích thêm phần [X]"
- "Tạo ví dụ nâng cao"
- "Làm bài tập khó hơn"

### 5. **Dùng System Prompt (GitHub Copilot Settings)**
```
Bạn là một giáo viên AWS tuyệt vời.
Giải thích luôn chi tiết, cụ thể, dễ hiểu.
Dùng ví dụ thực tế, code, diagram.
Liệt kê lỗi thường gặp.
Luôn dùng tiếng Việt.
```

---

## 📝 TEMPLATE ĐƠN GIẢN (Copy-Paste Ready)

```
**Chủ đề:** [Tên chủ đề]
**Mục tiêu:** Học [cái gì] cho AWS SAA
**Nội dung gốc:**
[PASTE TỪ NOTION]

**Yêu cầu output:**
1. Định nghĩa đơn giản (1 câu)
2. Ví dụ thực tế (2-3 case)
3. Mã ví dụ (Python/CLI)
4. So sánh (bảng)
5. Lỗi thường gặp
6. 5 câu hỏi ôn tập

Format: Markdown
Ngôn ngữ: Tiếng Việt
```

---

## 🚀 QUY TRÌNH HỌC ĐỀ XUẤT

1. **Tuần 1-2:** EC2, Storage (S3, EBS, EFS)
2. **Tuần 3-4:** Networking (VPC, SG), Database (RDS, DynamoDB)
3. **Tuần 5:** Scalability (ASG, ELB), Monitoring (CloudWatch)
4. **Tuần 6:** Security (IAM, Encryption), Cost Optimization
5. **Tuần 7:** Tổng hợp, làm bài tập
6. **Tuần 8:** Ôn tập, thi thử

---

**Chúc bạn học tập hiệu quả! 🎯**
