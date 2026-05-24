# 📚 AWS Solutions Architect Associate — Tài Liệu Ôn Tập

> 📖 Tổng hợp từ ghi chú Notion + khóa Udemy của Stephane Maarek  
> 🗓️ Cập nhật: 22/05/2026  
> 🎯 Mục tiêu: Pass kỳ thi SAA-C03

---

## 📋 Mục Lục

1. [IAM — Quản lý Quyền Truy Cập](#1-iam--quản-lý-quyền-truy-cập)
2. [EC2 Fundamentals — Máy Chủ Ảo](#2-ec2-fundamentals--máy-chủ-ảo)
3. [EC2 Advanced — IP, Placement, ENI, Hibernate](#3-ec2-advanced--ip-placement-eni-hibernate)
4. [EC2 Instance Storage — Lưu Trữ](#4-ec2-instance-storage--lưu-trữ)
5. [High Availability & Scalability — ELB & ASG](#5-high-availability--scalability--elb--asg)
6. [📝 Câu Hỏi Ôn Tập Tổng Hợp](#6--câu-hỏi-ôn-tập-tổng-hợp)

---

## 1. IAM — Quản lý Quyền Truy Cập

> 💡 **IAM = Identity and Access Management** — Kiểm soát *ai* có thể làm *gì* trong AWS. Là **Global Service** (không phụ thuộc Region).

### 🏗️ Kiến Trúc IAM

```
AWS Account
├── Root Account (⚠️ NGUY HIỂM — chỉ dùng để tạo lần đầu)
├── IAM Users (người dùng cụ thể)
│   ├── User A → thuộc Group Dev + Group QA
│   ├── User B → thuộc Group Dev
│   └── User C → không thuộc group nào (cũng được)
└── IAM Groups (nhóm)
    ├── Dev Group → có Policy: EC2FullAccess
    ├── Admin Group → có Policy: AdministratorAccess
    └── ReadOnly Group → có Policy: ReadOnlyAccess
```

**Quy tắc quan trọng:**
- Groups chỉ chứa Users, **KHÔNG** chứa Groups khác
- User có thể thuộc nhiều Groups
- User KHÔNG bắt buộc phải thuộc Group nào

### 🔐 IAM Policies — JSON Permission

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2ButNotTerminate",
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

| Trường | Ý nghĩa | Ví dụ |
|--------|---------|-------|
| `Version` | Phiên bản ngôn ngữ policy | `"2012-10-17"` (luôn dùng cái này) |
| `Sid` | Statement ID — tên tùy chọn | `"AllowS3Access"` |
| `Effect` | Cho phép hay từ chối | `"Allow"` hoặc `"Deny"` |
| `Principal` | Ai được áp dụng | Account, User, Role |
| `Action` | API call nào được phép | `"s3:GetObject"`, `"ec2:*"` |
| `Resource` | Tài nguyên nào | `"*"` hoặc ARN cụ thể |
| `Condition` | Điều kiện thêm (tùy chọn) | IP, thời gian, MFA... |

> ⚠️ **Nguyên tắc vàng:** **Least Privilege** — chỉ cấp đúng quyền cần thiết, không hơn!

### 🛡️ Bảo Mật IAM

#### Password Policy
- Đặt độ dài tối thiểu
- Yêu cầu ký tự đặc biệt, số, chữ hoa/thường
- Buộc đổi mật khẩu định kỳ
- Ngăn dùng lại mật khẩu cũ

#### MFA (Multi-Factor Authentication)
**MFA = Mật khẩu + Thiết bị vật lý/app**

| Loại MFA | Ví dụ | Ghi chú |
|----------|-------|---------|
| Virtual MFA App | Google Authenticator, Authy | Phổ biến nhất |
| U2F Security Key | YubiKey (Yubico) | USB vật lý |
| Hardware Key Fob | Gemalto device | Thiết bị vật lý |
| GovCloud Key Fob | SurePassID | Chỉ dùng cho AWS GovCloud |

> 🔑 **Bắt buộc bật MFA cho Root Account và tài khoản quan trọng!**

### 🖥️ 3 Cách Truy Cập AWS

| Cách | Bảo vệ bởi | Dùng khi |
|------|-----------|---------|
| AWS Console (Web UI) | Password + MFA | Quản lý thủ công |
| AWS CLI | Access Key | Script, tự động hóa |
| AWS SDK (code) | Access Key | Lập trình ứng dụng |

```
Access Key ID     ~= Username
Secret Access Key ~= Password
⚠️ Không bao giờ chia sẻ Access Key với ai!
```

### ⚡ IAM Best Practices (thi thường hỏi)

- ✅ **Không** dùng Root Account cho công việc hàng ngày
- ✅ Tạo IAM User riêng cho mỗi người
- ✅ Dùng **Groups** để gán permissions (không gán trực tiếp cho User)
- ✅ Áp dụng nguyên tắc **Least Privilege**
- ✅ Bật **MFA** cho tất cả user quan trọng
- ✅ Rotate Access Keys định kỳ
- ✅ Dùng **IAM Roles** cho EC2/Lambda thay vì hardcode credentials

### ❓ Câu Hỏi Ôn Tập IAM

1. IAM là Global hay Regional service?
2. Root account có nên dùng thường xuyên không? Tại sao?
3. Một user có thể thuộc bao nhiêu Groups?
4. Sự khác biệt giữa IAM User, Group, Role, và Policy là gì?
5. Khi User thuộc 2 Groups có policies khác nhau, quyền nào được áp dụng?
6. MFA bảo vệ thêm được gì so với chỉ dùng password?
7. Access Key ID và Secret Access Key dùng để làm gì?

<details>
<summary>📌 Đáp án gợi ý</summary>

1. **Global** — không gắn với Region cụ thể
2. **Không** — Root có toàn quyền, nếu bị hack = mất toàn bộ account
3. Không giới hạn — user có thể thuộc nhiều groups
4. User = người dùng; Group = nhóm user; Role = quyền tạm thời cho services/apps; Policy = JSON định nghĩa permissions
5. Permissions hợp nhất (union), nhưng **Deny** luôn thắng **Allow**
6. Bảo vệ ngay cả khi mật khẩu bị lộ — cần thêm thiết bị vật lý
7. Dùng để xác thực khi truy cập AWS qua CLI hoặc SDK

</details>

---

## 2. EC2 Fundamentals — Máy Chủ Ảo

> 💡 **EC2 = Elastic Compute Cloud** = IaaS (Infrastructure as a Service)  
> Hiểu EC2 = hiểu nền tảng của toàn bộ AWS Cloud

### 🔧 EC2 Gồm Những Gì?

```
EC2 Instance = Máy chủ ảo trong AWS
├── Compute: CPU + RAM
├── Storage: EBS / EFS / Instance Store
├── Network: VPC, Security Group, IP
└── Bootstrap: EC2 User Data (chạy lần đầu khởi động)
```

### 🏷️ Naming Convention — Cách Đọc Tên Instance

```
m 5 . 2xlarge
│ │   └─── Size (nhỏ → lớn: nano < micro < small < medium < large < xlarge < 2xlarge...)
│ └─────── Generation (số càng cao = phần cứng càng mới)
└───────── Instance class (loại máy)
```

### 📊 Các Loại EC2 Instance (cần nhớ cho thi)

| Family | Letters | Đặc điểm | Use Cases | Ví dụ |
|--------|---------|----------|-----------|-------|
| General Purpose | T, M, A | Cân bằng CPU/RAM/Network | Web server, App server | t2.micro (Free Tier!) |
| Compute Optimized | C | CPU cao, RAM thấp | Batch processing, Gaming, HPC, ML inference | c5.4xlarge |
| Memory Optimized | R, X, Z | RAM cực lớn | Database, Redis cache, Real-time analytics | r5.16xlarge |
| Storage Optimized | I, D, H | IOPS cao, local SSD | NoSQL DB, Data warehouse, OLTP | i3.4xlarge |
| Accelerated (GPU) | P, G, F, Inf | GPU/FPGA | ML training, Video rendering | p3.8xlarge |

> 🎯 **Tip thi:** Thấy câu hỏi về database lớn trong RAM → **R series**. ML training nhanh → **P/G series**. High IOPS local storage → **I series**.

### 🔥 EC2 User Data — Tự Động Hóa Khi Khởi Động

```bash
#!/bin/bash
# Script này chạy DUY NHẤT 1 LẦN khi instance boot lần đầu
# Dùng để: cài phần mềm, tải files, cấu hình server

# Ví dụ: Tự động tạo web server
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from EC2!</h1>" > /var/www/html/index.html
```

> ⚠️ User Data script chạy với quyền **root** và chỉ chạy **1 lần duy nhất** khi khởi động lần đầu.

### 🔒 Security Groups — Tường Lửa Cho EC2

```
Internet → [Security Group] → EC2 Instance

Inbound Rules (vào):  Mặc định CHẶN TẤT CẢ
Outbound Rules (ra):  Mặc định CHO PHÉP TẤT CẢ
```

**Đặc điểm quan trọng:**
- Security Group chỉ có rule **ALLOW**, **KHÔNG có DENY**
- Nếu không có rule nào match → traffic bị block
- Có thể reference SG khác (không chỉ IP)
- Áp dụng cho từng EC2 instance, **không phải subnet**
- 1 EC2 có thể có nhiều SG; 1 SG có thể gán cho nhiều EC2

### 🌐 Classic Ports Cần Nhớ

| Port | Protocol | Dùng cho | Ghi chú |
|------|----------|---------|---------|
| 22 | SSH (TCP) | Linux EC2 | Chỉ mở cho IP của bạn! |
| 3389 | RDP (TCP) | Windows EC2 | Remote Desktop |
| 21 | FTP (TCP) | File transfer | |
| 22 | SFTP (TCP) | Secure FTP | Dùng SSH |
| 80 | HTTP (TCP) | Web server | Public |
| 443 | HTTPS (TCP) | Web server secure | Public |

### 💰 EC2 Purchasing Options — Cách Mua/Thuê

| Loại | Giảm giá | Cam kết | Khi nào dùng |
|------|---------|---------|-------------|
| **On-Demand** | 0% | Không | Thử nghiệm, workload không đoán được |
| **Reserved (1 năm)** | ~40% | 1 năm | App chạy liên tục, predictable usage |
| **Reserved (3 năm)** | ~60% | 3 năm | App ổn định lâu dài |
| **Savings Plans** | ~66% | 1-3 năm | Linh hoạt hơn Reserved |
| **Spot Instances** | ~90% | Không | Batch jobs, có thể bị interrupt |
| **Dedicated Hosts** | - | On-demand/Reserved | Compliance, software licensing |
| **Dedicated Instances** | - | - | Isolation cần thiết |

> 🎯 **Tip thi thường hỏi:**
> - Tiết kiệm nhất cho workload liên tục = **Reserved**
> - Tiết kiệm nhất tuyệt đối = **Spot** (nhưng có thể bị AWS lấy lại)
> - Cần phần cứng riêng = **Dedicated Host** (BYOL - Bring Your Own License)

### ❓ Câu Hỏi Ôn Tập EC2

1. t2.micro thuộc family nào? Có bao nhiêu vCPU và RAM?
2. EC2 User Data chạy bao nhiêu lần?
3. Security Group chặn traffic bằng cách nào?
4. Bạn cần chạy batch jobs có thể interrupt, loại instance nào rẻ nhất?
5. Ứng dụng cần 512GB RAM, loại instance family nào?
6. Port nào dùng để SSH vào Linux EC2?
7. Sự khác biệt giữa Dedicated Host và Dedicated Instance?

<details>
<summary>📌 Đáp án gợi ý</summary>

1. **General Purpose (T family)**, 1 vCPU, 1 GB RAM — Free Tier 750h/tháng
2. **Chỉ 1 lần** — lần đầu tiên instance boot
3. **Không có rule** = block; mọi traffic đều bị chặn nếu không được allow
4. **Spot Instances** — rẻ nhất ~90%, nhưng có thể bị interrupt
5. **Memory Optimized (R, X series)** — r5.16xlarge (512GB RAM)
6. **Port 22**
7. Dedicated Host = toàn bộ server vật lý riêng (control socket level); Dedicated Instance = instance chạy trên hardware không share với customer khác nhưng có thể share với instance khác cùng account

</details>

---

## 3. EC2 Advanced — IP, Placement, ENI, Hibernate

### 🌍 Public vs Private vs Elastic IP

```
Internet
   │
   ├── Public IP: Địa chỉ thay đổi khi restart instance
   │              Duy nhất trên Internet
   │
   └── Private IP: Chỉ trong VPC internal
                   Cố định, không thay đổi khi restart
                   Connect ra Internet qua NAT Gateway

Elastic IP = Public IP tĩnh (không đổi khi restart)
           = Bạn "sở hữu" IP này cho đến khi release
           ⚠️ TỐN TIỀN nếu không attach vào instance đang chạy!
```

**Thực tế:**
- Khi **Stop → Start** EC2: Public IP thay đổi, Private IP giữ nguyên
- Elastic IP: giải quyết vấn đề IP thay đổi, nhưng khuyến khích dùng **DNS hoặc Load Balancer** thay thế

### 🎯 EC2 Placement Groups — Chiến Lược Đặt Máy

| Strategy | Cách đặt | Ưu điểm | Nhược điểm | Dùng khi |
|----------|----------|---------|-----------|---------|
| **Cluster** | Cùng rack, cùng AZ | Latency thấp nhất, networking 10Gbps+ | Nếu rack hỏng = mất hết | HPC, Big Data jobs nhanh, cần low latency |
| **Spread** | Mỗi instance 1 rack khác nhau | Fault isolation tối đa | Max 7 instances/AZ | App quan trọng, cần high availability |
| **Partition** | Chia theo partitions (racks) | Scale lớn + fault isolation theo partition | Phức tạp hơn | Hadoop, Kafka, Cassandra |

```
Cluster:  [EC2][EC2][EC2] ← cùng rack → nhanh nhưng rủi ro

Spread:   [rack1: EC2] [rack2: EC2] [rack3: EC2] ← isolated

Partition: [Partition1: EC2,EC2,EC2] [Partition2: EC2,EC2,EC2]
            ↕ rack khác nhau
```

### 🌐 ENI — Elastic Network Interface

> ENI = **Card mạng ảo** trong VPC

**Mỗi ENI có:**
- Primary Private IPv4 (luôn có)
- Secondary Private IPv4 (tùy chọn)
- Elastic/Public IP (tùy chọn)
- Security Groups
- MAC Address

**Đặc điểm:**
- Mỗi EC2 có 1 ENI mặc định (eth0)
- Có thể tạo thêm ENI và gắn vào EC2 (eth1, eth2...)
- **ENI bound to AZ** — không thể di chuyển qua AZ khác
- **Có thể chuyển ENI giữa các EC2** trong cùng AZ → dùng cho failover

```
Failover Use Case:
EC2-Primary (eth0: 10.0.1.5) chết
→ Detach ENI từ EC2-Primary
→ Attach ENI vào EC2-Backup
→ EC2-Backup giờ có IP 10.0.1.5 (seamless failover!)
```

### 💤 EC2 Hibernate — Ngủ Đông Instance

| Trạng thái | RAM | Disk | Boot time |
|-----------|-----|------|-----------|
| **Stop** | Mất | Giữ (EBS) | Bình thường (chậm) |
| **Terminate** | Mất | Mất (mặc định) | N/A |
| **Hibernate** | **Lưu vào EBS** | Giữ (EBS) | **Rất nhanh!** |

**Yêu cầu để dùng Hibernate:**
- RAM < 150 GB
- Root volume phải là **EBS encrypted**
- Không dùng được cho Bare Metal instances
- Thời gian hibernate tối đa: **60 ngày**
- OS: Linux, Windows

**Khi nào dùng Hibernate?**
- Processes lâu dài không muốn dừng
- Tránh warm-up time khi restart app
- Save chi phí ban đêm mà không mất state

### ❓ Câu Hỏi Ôn Tập EC2 Advanced

1. Khi Stop rồi Start lại EC2, cái gì thay đổi?
2. Bạn có 3 EC2 instances chạy ứng dụng quan trọng, cần đảm bảo nếu 1 rack server hỏng thì còn 2 cái chạy. Dùng Placement Group nào?
3. ENI có thể di chuyển qua AZ khác không?
4. EC2 Hibernate lưu RAM vào đâu?
5. Tại sao Elastic IP tốn tiền khi không dùng?

<details>
<summary>📌 Đáp án gợi ý</summary>

1. **Public IP thay đổi**, Private IP và EBS data giữ nguyên
2. **Spread Placement Group** — isolates instance trên different hardware
3. **Không** — ENI bound to specific AZ
4. **Root EBS volume** (phải được encrypted)
5. AWS tính tiền để hạn chế lãng phí IP public (có giới hạn số lượng toàn cầu)

</details>

---

## 4. EC2 Instance Storage — Lưu Trữ

### 💾 So Sánh Các Loại Storage

| | EBS | Instance Store | EFS | S3 |
|--|-----|---------------|-----|-----|
| **Loại** | Block | Block (local) | File (NFS) | Object |
| **Vị trí** | Network drive | Physical disk | Network share | Managed |
| **Persistent** | ✅ Có | ❌ Ephemeral | ✅ Có | ✅ Có |
| **Gắn vào** | 1 EC2 (mặc định) | Chỉ 1 EC2 | Nhiều EC2 | Không gắn (API) |
| **Performance** | Tốt | Rất cao (NVMe) | Trung bình | Thấp (latency) |
| **Cross-AZ** | ❌ (cần snapshot) | ❌ | ✅ | ✅ |
| **Giá** | Trung bình | Miễn phí (built-in) | Cao nhất | Rẻ nhất |
| **Dùng khi** | OS disk, DB | Buffer/Cache/Temp | Shared filesystem | Backup, static files |

### 📀 EBS — Elastic Block Store

> EBS = Ổ cứng mạng (network drive) gắn vào EC2

```
EC2 Instance ←──network──→ EBS Volume
                            (us-east-1a)

⚠️ EBS locked to AZ — cannot attach to EC2 in different AZ
✅ Solution: Snapshot → Copy to another AZ → Create new EBS
```

**EBS Volume Types:**

| Type | IOPS | Throughput | Dùng khi |
|------|------|-----------|---------|
| gp3 (SSD) | 3,000-16,000 | 125-1,000 MB/s | General purpose (mới nhất, recommend) |
| gp2 (SSD) | Up to 16,000 | - | General purpose (cũ hơn) |
| io1/io2 (SSD) | Up to 64,000 | High | Database cần high IOPS |
| st1 (HDD) | - | 500 MB/s | Big data, Data warehouse |
| sc1 (HDD) | - | 250 MB/s | Cold storage, infrequent access |

> 🎯 **Tip thi:** HDD (st1, sc1) **không thể** là boot volume. Chỉ SSD (gp, io) mới boot được.

**Delete on Termination:**
- Root EBS: mặc định **XÓA** khi terminate
- Additional EBS: mặc định **GIỮ** khi terminate
- Có thể thay đổi setting này

### 📸 EBS Snapshots

```
EBS Volume ──snapshot──→ S3 (stored behind the scenes)
                         ↓
                    Dùng để:
                    1. Backup/Restore
                    2. Copy sang AZ/Region khác
                    3. Tạo EBS volume mới
```

**Tính năng Snapshot:**

| Feature | Mô tả | Chi phí |
|---------|-------|---------|
| **Standard Snapshot** | Backup thông thường | Bình thường |
| **Archive Tier** | Lưu trữ lâu dài, rẻ hơn 75% | Rẻ hơn, nhưng restore mất 24-72h |
| **Recycle Bin** | Giữ snapshot đã xóa (1 ngày - 1 năm) | Nhỏ |
| **Fast Snapshot Restore (FSR)** | Restore ngay không cần warmup | Đắt tiền |

### 🖼️ AMI — Amazon Machine Image

> AMI = Template để tạo EC2 instances

```
EC2 (đã setup) → [Stop] → [Create AMI] → AMI

AMI chứa:
- OS (Linux, Windows, macOS)
- Cài đặt phần mềm
- Cấu hình
- EBS snapshot

Dùng AMI để:
→ Launch instance mới nhanh hơn (preinstalled)
→ Clone instance
→ Copy sang Region khác
```

**3 loại AMI:**

| Loại | Ai tạo | Ví dụ |
|------|--------|-------|
| **Public AMI** | AWS | Amazon Linux 2 |
| **Custom AMI** | Bạn | "My-App-Server-v2" |
| **Marketplace AMI** | Third party (mua/bán) | WordPress, MongoDB |

### ⚡ EC2 Instance Store

> Instance Store = Ổ cứng **vật lý** gắn trực tiếp vào server

```
Ưu điểm:  IOPS cực cao (lên đến 3.3 triệu IOPS!)
               vs EBS gp2 max chỉ 32,000 IOPS

Nhược điểm: EPHEMERAL — mất dữ liệu khi Stop/Terminate!

Dùng khi:
✅ Buffer, Cache
✅ Scratch data / Temp files
✅ Cần I/O cực cao
❌ KHÔNG dùng để lưu data lâu dài!
```

### ❓ Câu Hỏi Ôn Tập Storage

1. EBS có thể gắn vào EC2 ở AZ khác không?
2. Bạn muốn backup EBS rồi restore sang AZ khác, phải làm gì?
3. Khi terminate EC2, EBS nào bị xóa mặc định?
4. Instance Store có bị xóa khi reboot không? Khi Stop thì sao?
5. Loại EBS nào rẻ nhất cho data ít xuyên?
6. AMI có thể copy sang Region khác không?
7. EBS Snapshot Archive giúp tiết kiệm bao nhiêu % chi phí?

<details>
<summary>📌 Đáp án gợi ý</summary>

1. **Không** — EBS locked to AZ. Muốn move phải snapshot → copy → tạo mới
2. **Snapshot EBS** → Copy snapshot to target AZ/Region → Create EBS volume from snapshot
3. **Root EBS** (delete on termination = true mặc định); Additional EBS volume giữ nguyên
4. **Reboot: Không mất** (data giữ); **Stop: Mất tất cả** Instance Store data
5. **sc1 (Cold HDD)** — rẻ nhất, dành cho infrequent access
6. **Có** — AMI có thể copy across regions
7. **75% rẻ hơn** — nhưng restore mất 24-72 giờ

</details>

---

## 5. High Availability & Scalability — ELB & ASG

### 📈 Scalability vs High Availability

```
Scalability = Xử lý được nhiều tải hơn
├── Vertical (Scale Up): Nâng cấp 1 máy (t2.micro → t2.large)
│   ✅ Đơn giản | ❌ Có giới hạn phần cứng | ❌ Downtime khi nâng cấp
└── Horizontal (Scale Out): Thêm máy (1 EC2 → 10 EC2)
    ✅ Không giới hạn | ✅ Không downtime | ✅ Modern apps

High Availability = Tồn tại ngay cả khi 1 AZ/DC sập
└── Cần chạy tối thiểu ở 2 AZ
    ✅ Passive HA: RDS Multi-AZ (standby chờ sẵn)
    ✅ Active HA: Multiple EC2 + Load Balancer (tất cả cùng phục vụ)
```

> 🎯 Scalability ≠ High Availability, nhưng thường kết hợp cùng nhau!

### ⚖️ Elastic Load Balancer (ELB)

> ELB = Cổng vào điều phối traffic đến nhiều instances

```
Users (internet)
      │
  [ELB / Load Balancer]  ← single endpoint
      │
  ┌───┼───┐
 EC2  EC2  EC2    ← target group
```

**Lợi ích ELB:**
- Một endpoint duy nhất cho users
- Health checks — không gửi traffic đến instance bị lỗi
- SSL Termination (HTTPS)
- Cross-zone load balancing
- Tách public traffic / private traffic

### 🔑 4 Loại Load Balancer (Quan Trọng Cho Thi!)

| LB Type | Layer OSI | Protocols | Khi nào dùng | Launch năm |
|---------|-----------|-----------|-------------|-----------|
| **CLB** (Classic) | 4 & 7 | HTTP, HTTPS, TCP | ⚠️ Đã deprecated, không dùng nữa | 2009 |
| **ALB** (Application) | 7 (HTTP) | HTTP, HTTPS, WebSocket | Web apps, Microservices, Docker/ECS | 2016 |
| **NLB** (Network) | 4 (TCP) | TCP, UDP, TLS | Real-time, Gaming, IoT, cần static IP | 2017 |
| **GWLB** (Gateway) | 3 (IP) | IP | Security appliances, Firewalls, IDS/IPS | 2020 |

### 🌐 Application Load Balancer (ALB) — Chi Tiết

**ALB routing dựa trên:**

```
1. Path-based:     example.com/users    → Target Group A (User Service)
                   example.com/products → Target Group B (Product Service)

2. Host-based:     users.example.com    → Target Group A
                   api.example.com      → Target Group B

3. Query string:   example.com?type=mobile → Target Group A (mobile)
                   example.com?type=desktop → Target Group B (desktop)
```

**Target Groups của ALB:**
- EC2 instances (managed by ASG)
- ECS tasks
- Lambda functions
- IP Addresses (private IPs)

**Lưu ý quan trọng:**
- ALB có **fixed hostname** (xxx.region.elb.amazonaws.com)
- Instance **KHÔNG thấy IP thật** của client — phải đọc từ header `X-Forwarded-For`
- Dùng `X-Forwarded-Port` và `X-Forwarded-Proto` để lấy port và protocol gốc

### 🔀 Network Load Balancer (NLB) — Chi Tiết

```
Đặc điểm NLB:
✅ Hiệu năng cao: xử lý hàng triệu requests/giây
✅ Latency thấp (~100ms vs ~400ms của ALB)
✅ 1 Static IP per AZ (hỗ trợ Elastic IP)
✅ Không dùng SSL termination
❌ Không route theo URL/header như ALB

Dùng khi: TCP/UDP traffic, Gaming, IoT, Low latency cực cần
```

### ⚡ Auto Scaling Group (ASG)

> ASG = Tự động thêm/bớt EC2 instances theo load

```
Target: Giữ CPU usage ≈ 40%

     Low traffic:          High traffic:
     [EC2] [EC2]           [EC2] [EC2] [EC2] [EC2] [EC2]
           ↑                              ↑
       Scale In                      Scale Out
    (giảm instance)               (thêm instance)
```

**ASG Configuration:**
```
Min Size:      Số instance tối thiểu   (VD: 2)
Max Size:      Số instance tối đa      (VD: 10)
Desired:       Số instance mong muốn   (VD: 4)
```

**ASG Scaling Policies:**

| Policy | Mô tả | Ví dụ |
|--------|-------|-------|
| **Target Tracking** | Giữ metric ở target | Giữ Average CPU = 40% |
| **Simple/Step Scaling** | Scale khi alarm trigger | CPU > 70% → thêm 2 instance |
| **Scheduled Scaling** | Scale theo lịch | 8am thêm, 10pm giảm |
| **Predictive Scaling** | ML dự đoán trước | Tự học pattern, scale trước |

> 🔑 **Scaling Cooldown:** Sau khi scale, ASG chờ 300s (mặc định) trước khi scale tiếp — tránh scale liên tục.

**ASG + ELB Integration:**
```
Users → ELB → ASG (manages EC2 fleet)
              ↓
         Health Check failure → ASG terminates unhealthy, launches new
         Load increases        → ASG launches more
         Load decreases        → ASG terminates excess
```

### 🔒 Security Group Best Practice Cho ELB

```
ELB Security Group:
  Inbound:  Allow 0.0.0.0/0 on 80/443 (public internet)
  Outbound: Allow to EC2 SG

EC2 Security Group:
  Inbound:  Allow ONLY from ELB Security Group ← QUAN TRỌNG!
  ❌ KHÔNG bao giờ allow 0.0.0.0/0 trực tiếp vào EC2!
```

### ❓ Câu Hỏi Ôn Tập HA & Scalability

1. Vertical scaling và Horizontal scaling khác gì?
2. High Availability tối thiểu cần bao nhiêu AZ?
3. Bạn cần load balancer cho microservices với Docker/ECS, dùng loại nào?
4. NLB có ưu điểm gì hơn ALB?
5. ASG Min=2, Max=10, Desired=4 — khi CPU tăng cao sẽ xảy ra gì?
6. Scaling Cooldown Period là gì và tại sao cần?
7. Client IP thật ở đâu khi dùng ALB?

<details>
<summary>📌 Đáp án gợi ý</summary>

1. Vertical = nâng cấp 1 máy to hơn (có giới hạn); Horizontal = thêm nhiều máy (không giới hạn)
2. Tối thiểu **2 AZ**
3. **ALB** — layer 7, route theo path/host, hỗ trợ ECS/Docker
4. NLB: Low latency (~100ms), Static IP, xử lý TCP/UDP, hàng triệu req/s
5. ASG sẽ **Scale Out** (thêm instances) tối đa đến 10
6. Thời gian chờ sau mỗi scaling action để metrics ổn định trước khi scale tiếp — tránh thrashing
7. Trong HTTP header **`X-Forwarded-For`**

</details>

---

## 6. 📝 Câu Hỏi Ôn Tập Tổng Hợp

### 🎯 30 Câu Hỏi Trắc Nghiệm Kiểu Thi SAA

**1.** Một công ty muốn đảm bảo IAM user chỉ có thể access AWS từ văn phòng (IP cố định). Cách tốt nhất là?
- A. Tạo VPN riêng
- B. Dùng IAM Policy với Condition key `aws:SourceIp`
- C. Dùng Security Group
- D. Enable MFA

**2.** EC2 instance bị stop rồi start lại. Điều gì xảy ra?
- A. Public IP và Private IP đều thay đổi
- B. Public IP thay đổi, Private IP giữ nguyên ✅
- C. Cả hai đều giữ nguyên
- D. Private IP thay đổi, Public IP giữ nguyên

**3.** Ứng dụng cần xử lý 2 triệu IOPS cho database. Loại storage nào phù hợp?
- A. EBS gp2
- B. EBS st1
- C. EC2 Instance Store ✅
- D. EFS

**4.** Bạn cần chạy 7 EC2 instances quan trọng, đảm bảo không bao giờ 2 instances trên cùng 1 physical rack. Dùng Placement Group nào?
- A. Cluster
- B. Spread ✅
- C. Partition
- D. Không cần Placement Group

**5.** Cần load balancer hỗ trợ WebSocket và route traffic dựa trên URL path. Dùng loại nào?
- A. Classic Load Balancer
- B. Network Load Balancer
- C. Application Load Balancer ✅
- D. Gateway Load Balancer

**6.** Auto Scaling Group đang chạy 4 instances (min=2, max=8). CPU tăng lên 90%. Điều gì xảy ra?
- A. Không có gì — đã ở desired
- B. Thêm instances cho đến khi CPU xuống, tối đa 8 ✅
- C. Thay EC2 to hơn
- D. Tắt ASG alerts

**7.** Muốn copy EBS volume sang AZ khác, cần làm gì?
- A. Không thể, EBS không copy được
- B. Detach và move trực tiếp
- C. Tạo Snapshot → Copy snapshot → Create volume từ snapshot ✅
- D. Dùng EFS thay thế

**8.** IAM Role khác gì IAM User?
- A. Role không có credentials lâu dài ✅
- B. Role chỉ dùng được cho AWS services
- C. Role mạnh hơn User
- D. Role và User như nhau

**9.** Bạn đang dùng ALB, application muốn lấy IP thật của client. Tìm ở đâu?
- A. IP của request trực tiếp
- B. Header `X-Real-IP`
- C. Header `X-Forwarded-For` ✅
- D. Không lấy được

**10.** EBS Snapshot Archive giúp tiết kiệm bao nhiêu % chi phí? Và restore mất bao lâu?
- A. 50%, 1 giờ
- B. 75%, 24-72 giờ ✅
- C. 90%, 1 ngày
- D. 60%, 12 giờ

**11.** EC2 Hibernate yêu cầu root EBS volume phải như thế nào?
- A. Phải là io1 volume
- B. Phải được **Encrypted** ✅
- C. Phải ≥ 500 GB
- D. Phải là instance store

**12.** Startup đang dùng Reserved Instances 1 năm cho web server. Traffic tăng đột biến vào Black Friday. Giải pháp nào tốt nhất cho phần traffic tăng thêm?
- A. Mua thêm Reserved Instances
- B. Scale vertical lên instance lớn hơn
- C. Dùng Spot Instances ✅
- D. Dùng Dedicated Hosts

**13.** Security Group chứa loại rules nào?
- A. Chỉ Allow rules ✅
- B. Chỉ Deny rules
- C. Cả Allow và Deny
- D. Tùy cấu hình

**14.** NLB hoạt động ở layer nào của OSI?
- A. Layer 7 (Application)
- B. Layer 6
- C. Layer 4 (Transport) ✅
- D. Layer 3 (Network)

**15.** Bạn có 3 EC2 instances chạy Hadoop. Cần đảm bảo fault isolation theo partitions. Dùng gì?
- A. Cluster Placement Group
- B. Spread Placement Group
- C. Partition Placement Group ✅
- D. Multi-AZ

### 📊 Quick Reference Cards

#### IAM Cheat Sheet
```
Root Account    → Tạo lần đầu, enable MFA, KHÔNG dùng hàng ngày
IAM User        → Người dùng thực, có credentials cố định
IAM Group       → Nhóm users, gán permissions hàng loạt
IAM Policy      → JSON permissions, gán cho User/Group/Role
IAM Role        → Credentials tạm, cho services (EC2, Lambda...)
MFA             → Always enable cho Root và Admin users
Least Privilege → Chỉ cấp quyền cần thiết!
```

#### EC2 Instance Types Cheat Sheet
```
T, M, A → General Purpose (web, app servers)
C       → Compute (batch, gaming, ML inference)
R, X, Z → Memory (DB, Redis, analytics)
I, D, H → Storage (NoSQL, data warehouse)
P, G    → GPU (ML training, video)
```

#### Storage Cheat Sheet
```
EBS          → Block storage, 1 EC2, same AZ
Instance Store → Local disk, highest IOPS, ephemeral!
EFS          → NFS, multi-EC2 share, expensive
S3           → Object storage, global, cheapest per GB

EBS Types:
gp3/gp2 → General (boot volumes, default choice)
io1/io2 → High IOPS databases
st1     → Big data (HDD, throughput)
sc1     → Cold/Archive (HDD, cheapest)
```

#### Load Balancer Cheat Sheet
```
ALB → Layer 7, HTTP/HTTPS, URL routing, microservices, ECS
NLB → Layer 4, TCP/UDP, static IP, low latency, real-time
GWLB → Layer 3, security appliances, firewall
CLB → Deprecated, đừng dùng

ALB Tips:
- X-Forwarded-For = real client IP
- Fixed Hostname
- Route by: path / host / query string / header
```

#### ASG Cheat Sheet
```
Min / Desired / Max = boundaries của fleet

Scaling Policies:
- Target Tracking → dễ nhất, giữ metric ở target
- Step Scaling    → step-by-step khi alarm trigger
- Scheduled       → theo lịch (morning hours)
- Predictive      → ML-based, scale trước traffic peak

Cooldown = 300s mặc định, tránh thrashing
```

---

## 🗓️ Lịch Học Đề Xuất

| Tuần | Chủ đề | Mục tiêu |
|------|--------|---------|
| Tuần 1 | ✅ IAM + EC2 Basics | Hiểu rõ IAM, Security Groups, Instance types |
| Tuần 2 | ✅ EC2 Storage (EBS, AMI, Instance Store) | Biết khi nào dùng loại storage nào |
| Tuần 3 | ✅ HA, ELB, ASG | Thiết kế scalable architecture |
| Tuần 4 | VPC, Route 53 | Networking nâng cao |
| Tuần 5 | S3, Database (RDS, DynamoDB, ElastiCache) | Storage & Data services |
| Tuần 6 | Serverless (Lambda, API Gateway, SQS, SNS) | Modern architectures |
| Tuần 7 | Security (KMS, Shield, WAF), Monitoring | Enterprise features |
| Tuần 8 | Practice exams + Review | Pass kỳ thi! 🎯 |

---

> 💪 **Chúc bạn học tốt và pass AWS SAA!**  
> 📌 File này được tổng hợp từ ghi chú Notion cá nhân — cập nhật thêm khi học tiếp các phần mới.

