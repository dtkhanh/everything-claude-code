# 📚 AWS Solutions Architect Associate — Tài Liệu Ôn Tập HOÀN CHỈNH

> 📖 Tổng hợp từ Notion cá nhân + khóa Udemy Stephane Maarek
> 🗓️ Cập nhật: 22/05/2026 | 🎯 Mục tiêu: Pass SAA-C03

---

## 📋 Mục Lục

1. [IAM — Quản lý Quyền Truy Cập](#1-iam)
2. [EC2 Fundamentals](#2-ec2-fundamentals)
3. [EC2 Advanced — IP, Placement, ENI, Hibernate](#3-ec2-advanced)
4. [EC2 Instance Storage](#4-ec2-storage)
5. [High Availability — ELB & ASG](#5-high-availability--elb--asg)
6. [RDS, Aurora & ElastiCache](#6-rds-aurora--elasticache)
7. [Route 53](#7-route-53)
8. [Amazon S3](#8-amazon-s3)
9. [CloudFront & Global Accelerator](#9-cloudfront--global-accelerator)
10. [Storage Extras — FSx, Storage Gateway, Snow](#10-storage-extras)
11. [SQS, SNS & Kinesis — Messaging](#11-sqs-sns--kinesis)
12. [Containers — ECS, Fargate, ECR, EKS](#12-containers)
13. [Serverless — Lambda, API Gateway, DynamoDB](#13-serverless)
14. [VPC & Networking](#14-vpc--networking)
15. [Security — KMS, SSM, Shield, WAF](#15-security)
16. [Monitoring — CloudWatch, CloudTrail, Config](#16-monitoring)
17. [IAM Advanced & Organizations](#17-iam-advanced--organizations)
18. [Disaster Recovery & Migrations](#18-disaster-recovery--migrations)
19. [Other Services](#19-other-services)
20. [Câu Hỏi Tổng Hợp 50 Câu](#20-câu-hỏi-tổng-hợp-50-câu)

---


## 1. IAM

> **IAM = Identity and Access Management** — **Global Service**. Kiểm soát AI có quyền làm GÌ trong AWS.

### Kiến Trúc IAM

```
Root Account (tạo lần đầu, enable MFA, KHÔNG dùng hàng ngày)
├── IAM Users  (người dùng thực tế, có credentials dài hạn)
│   └── thuộc IAM Groups
├── IAM Groups (nhóm users, chỉ chứa Users — không chứa Groups)
│   └── gán IAM Policies
└── IAM Roles  (dùng cho AWS services: EC2, Lambda, cross-account)
    └── Temporary credentials (STS)

IAM Policies: JSON document định nghĩa permissions
```

### IAM Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "TênTùyChọn",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::123456:user/alice"},
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
  }]
}
```

| Trường | Ý nghĩa |
|--------|---------|
| Version | `"2012-10-17"` — luôn dùng cái này |
| Effect | `Allow` hoặc `Deny` |
| Principal | Ai được áp dụng (user, account, service) |
| Action | API calls được phép/từ chối |
| Resource | ARN của tài nguyên |
| Condition | Điều kiện thêm (IP, time, MFA...) |

> ⚠️ **DENY luôn override ALLOW**. Nếu có bất kỳ Deny nào = bị từ chối dù có nhiều Allow.

### MFA Options

| Loại | Ví dụ |
|------|-------|
| Virtual MFA | Google Authenticator, Authy |
| U2F Security Key | YubiKey (Yubico) |
| Hardware Key Fob | Gemalto |
| GovCloud Key Fob | SurePassID |

### 3 Cách Access AWS

| Cách | Bảo vệ bởi | Dùng khi |
|------|-----------|---------|
| Console (Web) | Password + MFA | Quản lý thủ công |
| CLI | Access Key ID + Secret | Scripts, automation |
| SDK (code) | Access Key ID + Secret | Applications |

### IAM Best Practices (Thi hay hỏi!)

- ✅ Không dùng Root cho công việc hàng ngày
- ✅ Tạo IAM user riêng cho mỗi người
- ✅ Dùng **Groups** để gán permissions (không gán trực tiếp cho users)
- ✅ Nguyên tắc **Least Privilege** — chỉ cấp quyền cần thiết
- ✅ Enable **MFA** cho Root và tất cả Admin users
- ✅ Rotate Access Keys định kỳ
- ✅ Dùng **IAM Roles** cho EC2/Lambda thay vì hardcode credentials

### Câu Hỏi IAM

1. IAM là Global hay Regional service?
2. Một Deny policy có thể override Allow không?
3. IAM Role khác IAM User ở điểm nào?
4. Least Privilege nghĩa là gì?
5. Root account có nên dùng hàng ngày không?

<details><summary>📌 Đáp án</summary>

1. **Global** — không phụ thuộc Region
2. **Có** — Explicit Deny luôn thắng
3. Role dùng temporary credentials (STS), không có username/password dài hạn; thường dùng cho AWS services
4. Chỉ cấp đúng quyền cần thiết, không thừa
5. **Không** — chỉ dùng để setup ban đầu, luôn dùng IAM users

</details>

---


## 2. EC2 Fundamentals

> **EC2 = Elastic Compute Cloud** = IaaS. Máy chủ ảo cho thuê.

### Instance Types (Phải nhớ cho thi!)

| Family | Letters | Dùng khi |
|--------|---------|---------|
| **General Purpose** | T, M, A | Web server, App server (t2.micro = Free Tier!) |
| **Compute Optimized** | C | Batch processing, Gaming, ML inference, HPC |
| **Memory Optimized** | R, X, Z | Database lớn, Redis, Real-time analytics |
| **Storage Optimized** | I, D, H | NoSQL, Data warehouse, IOPS cao |
| **GPU/Accelerated** | P, G, F, Inf | ML training, Video rendering |

```
Naming: m5.2xlarge
        │ │  └── Size (nano < micro < small < medium < large < xlarge < 2xlarge)
        │ └──── Generation (5 = thế hệ 5, cao hơn = mới hơn)
        └────── Class (m = General Purpose)
```

### EC2 User Data

```bash
#!/bin/bash
# Chạy DUY NHẤT 1 LẦN khi instance boot lần đầu (quyền root)
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from EC2!</h1>" > /var/www/html/index.html
```

### Security Groups

```
Internet ──→ [Security Group] ──→ EC2 Instance
              ↑ Inbound Rules: BLOCK ALL by default
              ↓ Outbound Rules: ALLOW ALL by default

Chỉ có ALLOW rules! Không có DENY.
Nếu không có rule match → traffic bị chặn.
Stateful: return traffic tự động allowed.
```

### Classic Ports Cần Nhớ

| Port | Protocol | Dùng cho |
|------|----------|---------|
| 22 | SSH (TCP) | Linux EC2 remote access |
| 3389 | RDP (TCP) | Windows EC2 Remote Desktop |
| 80 | HTTP | Web server |
| 443 | HTTPS | Secure web server |
| 3306 | MySQL/Aurora | Database |
| 5432 | PostgreSQL | Database |
| 6379 | Redis | Cache |

### EC2 Purchasing Options

| Loại | Giảm giá | Cam kết | Dùng khi |
|------|---------|---------|---------|
| **On-Demand** | 0% | Không | Dev/test, unpredictable workloads |
| **Reserved 1 năm** | ~40% | 1 năm | Steady-state production |
| **Reserved 3 năm** | ~60% | 3 năm | Long-term, very stable |
| **Savings Plans** | ~66% | 1-3 năm | Linh hoạt hơn Reserved |
| **Spot** | ~90% | Không | Batch, fault-tolerant (có thể bị interrupt!) |
| **Dedicated Host** | - | On-Demand/Reserved | BYOL, compliance, licensing |
| **Dedicated Instance** | - | - | Hardware isolation needed |

> 🎯 **Tip thi:**
> - Tiết kiệm nhất cho liên tục = Reserved
> - Tiết kiệm tuyệt đối = Spot (nhưng có thể bị AWS lấy lại!)
> - BYOL (Bring Your Own License) = Dedicated Host

### Câu Hỏi EC2

1. t2.micro thuộc instance family nào? Thuộc Free Tier không?
2. EC2 User Data chạy khi nào và bao nhiêu lần?
3. Security Group chặn traffic bằng cách nào?
4. Cần xử lý batch jobs có thể interrupt, chọn purchasing option nào?
5. Muốn dùng Oracle với existing license, chọn loại instance nào?

<details><summary>📌 Đáp án</summary>

1. **General Purpose (T family)**, có trong Free Tier (750h/tháng)
2. Chỉ **1 lần duy nhất** khi boot lần đầu
3. **Không có rule** = block; traffic chỉ được phép khi có ALLOW rule match
4. **Spot Instances** — ~90% discount
5. **Dedicated Host** — BYOL (Bring Your Own License)

</details>

---


## 3. EC2 Advanced

### Public vs Private vs Elastic IP

| | Public IP | Private IP | Elastic IP |
|--|-----------|-----------|-----------|
| Thay đổi khi Stop/Start | ✅ Có | ❌ Không | ❌ Không |
| Accessible từ internet | ✅ | ❌ | ✅ |
| Cost | Free | Free | Mất tiền nếu không dùng |

> 💡 Elastic IP: tốt để giữ IP cố định, nhưng khuyến khích dùng DNS hoặc Load Balancer thay thế.

### EC2 Placement Groups

| Strategy | Đặt như thế nào | Use Case |
|----------|----------------|---------|
| **Cluster** | Cùng rack, cùng AZ — low latency | HPC, Big Data, cần 10Gbps+ networking |
| **Spread** | Mỗi instance 1 rack riêng (max 7/AZ) | Critical apps, tối đa fault isolation |
| **Partition** | Nhóm partitions (racks), nhiều instances/partition | Hadoop, Kafka, Cassandra — scale lớn |

### ENI — Elastic Network Interface

- **Card mạng ảo** trong VPC
- Mỗi ENI có: Primary Private IP, Secondary Private IPs, Elastic IP, Security Groups, MAC
- **Bound to AZ** — không di chuyển qua AZ khác
- Có thể di chuyển giữa EC2 instances trong cùng AZ (dùng cho failover)

### EC2 Hibernate

| State | RAM | EBS | Boot time |
|-------|-----|-----|-----------|
| Running | ✅ Active | ✅ | - |
| Stop | ❌ Mất | ✅ Giữ | Bình thường (chậm) |
| Terminate | ❌ Mất | ❌ Xóa (mặc định) | N/A |
| **Hibernate** | **✅ Dump vào EBS** | ✅ Giữ | **Rất nhanh!** |

**Yêu cầu Hibernate:**
- Root EBS phải **ENCRYPTED**
- RAM < 150 GB
- Không dùng được Bare Metal
- Max thời gian hibernate: **60 ngày**

---


## 4. EC2 Storage

### So Sánh Storage Options

| | EBS | Instance Store | EFS | S3 |
|--|-----|---------------|-----|-----|
| Loại | Block | Block (local hardware) | File (NFS) | Object |
| Persistent | ✅ | ❌ Ephemeral! | ✅ | ✅ |
| Gắn vào | 1 EC2 (default) | Built-in (không detach) | Nhiều EC2 | API only |
| IOPS | Tốt (up to 64K) | Cực cao (3M+) | Trung bình | Thấp |
| Cross-AZ | Cần snapshot | ❌ | ✅ | ✅ |
| Giá | Trung bình | Built-in (free) | Cao nhất | Rẻ nhất |

### EBS Volume Types

| Type | IOPS | Throughput | Boot? | Dùng khi |
|------|------|-----------|-------|---------|
| **gp3** (SSD) | 3K-16K | 125-1000 MB/s | ✅ | General (khuyến nghị dùng gp3) |
| **gp2** (SSD) | Up to 16K | - | ✅ | General (cũ hơn) |
| **io1/io2** (SSD) | Up to 64K | - | ✅ | High perf databases |
| **st1** (HDD) | - | 500 MB/s | ❌ | Big data, throughput workloads |
| **sc1** (HDD) | - | 250 MB/s | ❌ | Cold data, archive (rẻ nhất) |

> ⚠️ **HDD (st1, sc1) KHÔNG thể làm boot volume!** Chỉ SSD (gp, io) mới boot được.

### EBS Snapshot Features

| Feature | Mô tả | Lưu ý |
|---------|-------|-------|
| Standard | Backup EBS tại thời điểm | Không cần detach |
| **Archive Tier** | Rẻ hơn 75% | Restore mất 24-72 giờ |
| **Recycle Bin** | Giữ snapshot đã xóa | 1 ngày - 1 năm |
| **Fast Snapshot Restore** | Instant restore | Đắt tiền |

### AMI — Amazon Machine Image

```
AMI = Template để tạo EC2 instances
Gồm: OS + Software + Configs + EBS snapshots

3 loại:
- Public AMI:      AWS cung cấp (Amazon Linux 2, Ubuntu...)
- Custom AMI:      Bạn tạo từ existing EC2
- Marketplace AMI: Third party bán (WordPress, MongoDB...)

Lợi ích Custom AMI:
✅ Boot nhanh hơn (pre-installed)
✅ Consistent environments
✅ Copy được sang Regions khác
```

### EBS Delete on Termination

```
Root EBS:     Delete on termination = TRUE (mặc định xóa)
Other EBS:    Delete on termination = FALSE (mặc định giữ)
Có thể thay đổi qua Console hoặc CLI
```

---


## 5. High Availability — ELB & ASG

### Scalability vs High Availability

```
Vertical Scaling:   Nâng cấp 1 máy (t2.micro → r5.xlarge)
                    ↑ Đơn giản | ✗ Có giới hạn hardware | ✗ Downtime

Horizontal Scaling: Thêm nhiều máy (1 EC2 → 10 EC2)
                    ↑ Unlimited scale | ↑ No downtime | ✓ Modern apps

High Availability:  Chạy ở ≥2 AZs, survive được 1 data center sập
                    Passive HA: RDS Multi-AZ (standby chờ sẵn)
                    Active HA:  Multiple instances + ELB (đều serve traffic)
```

### 4 Loại Load Balancer AWS

| Loại | Layer | Protocols | Dùng khi |
|------|-------|-----------|---------|
| **ALB** | 7 (HTTP) | HTTP, HTTPS, WebSocket | Microservices, containers, URL/host routing |
| **NLB** | 4 (TCP) | TCP, UDP, TLS | Real-time gaming, static IP, ultra-low latency |
| **GWLB** | 3 (IP) | IP | Firewall, IDS/IPS, 3rd party network appliances |
| ~~CLB~~ | 3-7 | - | ⚠️ **Deprecated** — không dùng nữa |

### ALB — Application Load Balancer

```
ALB Routing:
  Path:    example.com/users    → Target Group A
           example.com/products → Target Group B
  Host:    api.example.com      → TG API
           www.example.com      → TG Web
  Query:   ?platform=mobile     → TG Mobile
  Header:  Custom headers

Target Group có thể là:
- EC2 Instances (managed by ASG)
- ECS Tasks
- Lambda Functions
- IP Addresses (private)

Important:
- Fixed hostname: xxx.region.elb.amazonaws.com
- Client real IP ở header: X-Forwarded-For
- Port trong header: X-Forwarded-Port
- Protocol trong header: X-Forwarded-Proto
```

### NLB — Network Load Balancer

```
✅ Xử lý hàng triệu requests/giây
✅ Latency thấp ~100ms (ALB ~400ms)
✅ 1 Static IP per AZ (hỗ trợ Elastic IP)
✅ TCP/UDP/TLS: gaming, IoT, VoIP
❌ Không route dựa trên HTTP headers/URL
```

### ASG — Auto Scaling Group

```
Min Size:     Không bao giờ xuống dưới (VD: 2)
Desired:      Số instances hiện tại (VD: 4)
Max Size:     Không bao giờ vượt qua (VD: 10)

Scaling Policies:
🥇 Target Tracking:  Giữ CPU = 40% (đơn giản, recommend)
   Step Scaling:     CPU>70% → +2,  CPU>90% → +4
   Scheduled:        8am scale up, 11pm scale down
   Predictive:       ML học pattern, scale trước traffic

Cooldown: 300s (default) — chờ sau mỗi scaling action
  → Sử dụng AMI sẵn để giảm launch time → giảm cooldown cần thiết
```

### Security Group Pattern (Quan trọng!)

```
ELB Security Group:
  Inbound:  0.0.0.0/0 → port 80 (HTTP) và 443 (HTTPS)
  Outbound: Allow to EC2 SG

EC2 Security Group:
  Inbound:  ONLY từ ELB Security Group ← ĐỪNG mở 0.0.0.0/0 cho EC2!
  Outbound: Allow all
```

---


## 6. RDS, Aurora & ElastiCache

### RDS — Managed Relational Database

**Engines:** PostgreSQL | MySQL | MariaDB | Oracle | SQL Server | IBM Db2 | **Aurora**

### Read Replicas vs Multi-AZ (Cực quan trọng!)

| | Read Replicas | Multi-AZ |
|--|--------------|---------|
| **Mục đích** | Scale đọc (Performance) | Disaster Recovery (HA) |
| **Replication** | **ASYNC** (có thể stale) | **SYNC** (immediate) |
| **Serve traffic?** | ✅ Có (reads only) | ❌ Standby không serve |
| **Failover** | Manual promote | Tự động ~60-120s |
| **Cross-region** | ✅ Có (có network cost) | ❌ Same region only |
| **Dùng khi** | Read-heavy workloads | Production HA |

### Aurora vs RDS

| | Aurora | RDS |
|--|--------|-----|
| Performance | **5x MySQL, 3x PostgreSQL** | Standard |
| Storage | Auto-grow 10GB → 128TB | Phải provision |
| Read Replicas | Tới **15** | Tới 5 |
| Failover | **Instantaneous** | ~60-120 giây |
| Storage Copies | **6 copies across 3 AZs** | 1 copy (+ standby nếu Multi-AZ) |
| Cost | ~20% đắt hơn | Base cost |

### Aurora Architecture

```
Aurora Cluster:
Writer Endpoint  → Primary instance (write + read)
Reader Endpoint  → All read replicas (load balanced reads)

Storage: 6 copies across 3 AZs (2 per AZ)
  - Write: cần 4/6 OK → Highly durable
  - Read:  cần 3/6 OK → Highly available
  - Self-healing: peer-to-peer replication

Aurora Serverless v2:
  - Scale 0.5 → 128 ACUs
  - Tính tiền per second
  - Best for irregular/unpredictable workloads
```

### Aurora Special Features

| Feature | Mô tả |
|---------|-------|
| **Aurora Global** | 1 primary Region + 5 secondary Regions, replication < 1s, RTO < 1 min |
| **Aurora Serverless** | Auto-scale, scale to 0, per-second billing |
| **Aurora Multi-Master** | Multiple writers, instant failover (active-active) |
| **RDS Proxy** | Connection pooling, reduce DB stress, supports Lambda |

### ElastiCache: Redis vs Memcached

| | **Redis** | Memcached |
|--|-----------|-----------|
| Persistence | ✅ AOF, RDB | ❌ |
| Multi-AZ/Replication | ✅ | ❌ |
| Data structures | String, Hash, List, Set, Sorted Set | String only |
| Pub/Sub | ✅ | ❌ |
| Transactions | ✅ | ❌ |
| **Use case** | Sessions, Leaderboards, Pub/Sub | Simple caching only |

> 🎯 **Hầu hết câu hỏi thi = Redis.** Chọn Memcached chỉ khi: simple cache + multi-threaded.

### Cache Patterns

```
Lazy Loading (Cache-Aside):
  Request → Cache hit? → Return data (fast!)
           → Cache miss → DB query → Cache → Return
  ✅ Cache only filled data — tránh lãng phí
  ❌ Cache miss = 3 network calls (penalty)
  ❌ Data có thể stale (TTL cần thiết)

Write Through:
  Write data → DB + Cache cùng lúc
  ✅ Cache luôn fresh
  ❌ Write penalty (2 calls)
  ❌ Cache có data chưa bao giờ được đọc

Hybrid: Write Through cho critical data + Lazy Loading cho rest
```

---


## 7. Route 53

> DNS + Health Checks + Domain Management = Route 53

### DNS Record Types

| Type | Trỏ đến | Root domain? | Cost | Ghi chú |
|------|---------|-------------|------|---------|
| **A** | IPv4 | ✅ | - | example.com → 12.34.56.78 |
| **AAAA** | IPv6 | ✅ | - | example.com → 2001::1 |
| **CNAME** | Hostname | ❌ | - | www → example.com |
| **Alias** | AWS Resource | ✅ | **FREE** | example.com → ALB DNS |
| **NS** | Name Servers | ✅ | - | Delegation |

> 🎯 **Alias luôn tốt hơn CNAME** khi trỏ về AWS resources: FREE, dùng được apex/root domain!
>
> **Alias targets:** ELB, CloudFront, API Gateway, Elastic Beanstalk, S3 Website, VPC Interface Endpoint, Global Accelerator
> ❌ **Không** trỏ Alias vào EC2 DNS name!

### Routing Policies (QUAN TRỌNG NHẤT!)

| Policy | Use Case | Health Check? |
|--------|---------|--------------|
| **Simple** | 1 resource, random if multiple IPs | ❌ |
| **Weighted** | A/B testing, gradual deploys (70%/20%/10%) | Optional |
| **Latency** | Multi-region, route to lowest latency | Optional |
| **Failover** | DR: Primary → Failover khi fail | **Bắt buộc** |
| **Geolocation** | Route theo country/continent (localization) | Optional |
| **Geoproximity** | Route theo proximity + bias shifting | - |
| **Multi-Value** | Như Simple nhưng có HC, trả healthy IPs (max 8) | **Required** |
| **IP-Based** | Route theo client CIDR | - |

```
Geolocation vs Latency:
  Geolocation = Route theo địa lý CỐ ĐỊNH (user ở France → French server)
  Latency     = Route theo LATENCY THỰC TẾ thấp nhất (bất kể địa lý)
```

### Health Checks

```
3 loại:
1. Endpoint (HTTP/HTTPS/TCP) — endpoint phải PUBLIC accessible!
2. Calculated: AND/OR logic từ nhiều HCs
3. CloudWatch Alarm: cho private resources

Threshold: 18%+ health checkers OK → HEALTHY
Interval: 30s (standard) hoặc 10s (fast, đắt hơn)
```

### TTL Strategy

```
Trước khi change DNS:
  1. Giảm TTL xuống thấp (60s)
  2. Chờ old TTL cache expire
  3. Perform DNS change
  4. Verify mọi thứ OK
  5. Tăng TTL lên lại (3600s+)
```

---


## 8. Amazon S3

> Object Storage — lưu trữ mọi loại file, unlimited scale, 99.999999999% durability (11 9s)

### S3 Basics

```
Bucket: globally unique name, region-specific
Key:    full path (folder/subfolder/file.txt)
Value:  file content (max 5TB per object)
Note:   Upload > 5GB PHẢI dùng Multi-Part Upload
```

### S3 Storage Classes

| Class | Durability | Availability | Retrieval | Min Storage | Cost |
|-------|-----------|-------------|-----------|-------------|------|
| **Standard** | 11 9s | 99.99% | Instant | - | $$ |
| **Standard-IA** | 11 9s | 99.9% | Instant | 30 days | $ |
| **One Zone-IA** | 11 9s | 99.5% | Instant | 30 days | $ (1 AZ!) |
| **Glacier Instant** | 11 9s | 99.9% | Instant | 90 days | $$ |
| **Glacier Flexible** | 11 9s | 99.99% | Minutes/Hours | 90 days | $ |
| **Glacier Deep Archive** | 11 9s | 99.99% | 12h/48h | 180 days | ¢ (rẻ nhất!) |
| **Intelligent-Tiering** | 11 9s | 99.9% | Instant | - | $$+fee |

> 🎯 **Tip thi:**
> - "quarterly access, immediate retrieval" → **Glacier Instant Retrieval**
> - "archive, can wait 12 hours" → **Glacier Deep Archive**
> - "don't know access pattern" → **Intelligent-Tiering**

### S3 Lifecycle Rules

```
Ví dụ tối ưu chi phí:
Day 0:   Upload → Standard
Day 30:  → Standard-IA (ít truy cập)
Day 90:  → Glacier Flexible (archive)
Day 180: → Glacier Deep Archive (long-term)
Day 365: → Delete

Cấu hình qua: S3 Console → Management → Lifecycle rules
```

### S3 Replication

| | Same-Region (SRR) | Cross-Region (CRR) |
|--|------------------|-------------------|
| Dùng khi | Log aggregation, same-region backup | DR, compliance, cross-region access |
| Pricing | Thấp | Cao (data transfer) |

```
Yêu cầu: Bật versioning trên CẢ HAI buckets
Chỉ replicate objects MỚI sau khi bật! Objects cũ → dùng S3 Batch Replication
Không replicate: SSE-C encrypted, lifecycle-deleted, existing objects (phải dùng Batch)
```

### S3 Versioning

```
Bật versioning → Mỗi upload TẠO version mới, không overwrite
Delete an object → Tạo "delete marker" (không xóa thật!)
Restore → Xóa delete marker
Permanently delete → Xóa specific version ID

Suspend versioning → Không tạo version mới, versions cũ được giữ
```

### S3 Security

**Bucket Policy (Resource-based):**
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

**Access Control Priority:**
```
1. Explicit DENY (IAM or Bucket Policy) → DENIED
2. IAM Allow → ALLOWED
3. Bucket Policy Allow → ALLOWED
4. No policy → DENIED (default)

Block Public Access (Account level): override tất cả bucket policies → public access
```

**Encryption Options:**

| Method | Key managed by | Use when |
|--------|---------------|---------|
| SSE-S3 | AWS (transparent) | Default (mọi bucket mới bật mặc định) |
| SSE-KMS | KMS | Audit trail, key rotation control |
| SSE-C | Bạn tự quản lý | Compliance, tự kiểm soát keys |
| Client-Side | Bạn tự encrypt | Encrypt trước khi upload |
| DSSE-KMS | Dual-layer KMS | Extra compliance |

### S3 Performance

```
Baseline: 3,500 PUT/s và 5,500 GET/s PER PREFIX
Multi-part upload:  > 100MB (recommend), > 5GB (bắt buộc)
Transfer Acceleration: Upload nhanh hơn qua CloudFront edge
Byte-Range Fetches: Parallel downloads (tăng throughput)
S3 Select: Filter data với SQL (giảm data transfer)
```

### S3 Presigned URLs

```
Tạo URL tạm thời để share private object:
- Console: max 12 giờ
- CLI/SDK: max 168 giờ (7 ngày)
- Người nhận URL có CÙNG permissions với người tạo
- Dùng cho: download files tạm, upload cho users
```

### S3 Event Notifications

```
S3 Events → SNS Topic
          → SQS Queue
          → Lambda Function
          → Amazon EventBridge (filter nâng cao, nhiều destinations)

Ví dụ: Mỗi khi upload ảnh → Lambda → resize → lưu lại
```

### S3 Access Points

```
Thay vì 1 bucket policy phức tạp:
/finance/* → Finance Access Point (policy riêng cho Finance team)
/sales/*   → Sales Access Point (policy riêng cho Sales team)
/logs/*    → DevOps Access Point

Mỗi Access Point có DNS riêng, policy riêng
VPC Access Points: chỉ accessible từ VPC cụ thể
```

### S3 Object Lock & Glacier Vault Lock

```
Object Lock: WORM model (Write Once Read Many)
- Governance mode: chỉ user có quyền đặc biệt mới xóa được
- Compliance mode: KHÔNG AI xóa được kể cả root!
- Dùng cho: compliance, regulatory requirements

Retention period: chỉ định thời gian không thể xóa
Legal Hold: block deletion vô thời hạn (không cần retention period)
```

---


## 9. CloudFront & Global Accelerator

### CloudFront — CDN (Content Delivery Network)

```
User (Paris) → CloudFront Edge (Paris) → Cache hit? Return data!
                                        → Cache miss → Origin → Cache → Return

Origins:
- S3 Bucket (phổ biến nhất, dùng OAC để bảo mật)
- Custom Origin: ALB, EC2, S3 Website, HTTP endpoint

216+ Edge Locations worldwide
TTL: default 24h (có thể customize per behavior)
```

### CloudFront vs S3 CRR

| | CloudFront | S3 Cross-Region Replication |
|--|------------|---------------------------|
| Cache | ✅ TTL-based | ❌ Real-time sync |
| Coverage | 216+ locations global | Chọn specific regions |
| Content | Static + some dynamic | Cần full file availability |
| Use case | Global static web | Specific region low latency |

### CloudFront Cache Behaviors

```
/images/* → Origin: S3, Cache TTL: 86400s (static)
/api/*    → Origin: ALB, Cache TTL: 0 (dynamic, no cache)
/*        → Default behavior

Cache based on:
- Headers, Query strings, Cookies
- Chỉ forward gì cần thiết để tăng cache hit ratio
```

### CloudFront Security

```
Origin Access Control (OAC) — Bảo vệ S3:
  S3 Bucket Policy: Allow chỉ từ CloudFront service
  → Users không thể access S3 URL trực tiếp!

Geo Restriction:
  Allowlist: Chỉ cho phép countries cụ thể
  Blocklist: Chặn countries cụ thể

Signed URLs & Cookies:
  Signed URL: 1 URL = 1 file (premium video streaming)
  Signed Cookies: 1 cookie = nhiều files (premium content access)
  Include: expiry, allowed IPs, HTTPS requirement
```

### CloudFront Price Classes

| Class | Locations | Cost |
|-------|---------|------|
| All | All 216+ edge locations | Đắt nhất |
| 200 | Most regions (trừ đắt nhất) | Trung bình |
| 100 | Only US, EU, Canada | Rẻ nhất |

### AWS Global Accelerator

```
CloudFront:          Cache ở edge, HTTP/HTTPS only
Global Accelerator:  NO cache, anycast routing qua AWS private network

User → Nearest Edge Location → AWS Network → ALB/NLB/EC2

2 Static Anycast IPs (whitelist-friendly!)
Performance: faster routing via AWS backbone (bypass public internet)
Failover: < 1 minute

Use case:
- Gaming (TCP/UDP)
- IoT
- VoIP
- Non-HTTP protocols
- Need static IPs
- Fast failover
```

| | CloudFront | Global Accelerator |
|--|------------|-------------------|
| Protocol | HTTP/HTTPS | TCP/UDP (any) |
| Static IPs | ❌ (hostname) | ✅ 2 Anycast IPs |
| Caching | ✅ At edge | ❌ No cache |
| DDoS protection | ✅ Shield | ✅ Shield |
| Use case | Web content CDN | Non-HTTP, static IP |

---


## 10. Storage Extras

### Snow Family — Physical Data Migration

> Khi network quá chậm: nếu transfer qua internet > 1 tuần → dùng Snow!

| Device | Capacity | Use Case |
|--------|---------|---------|
| **Snowcone HDD** | 8TB HDD | Edge computing/storage, small migration |
| **Snowcone SSD** | 14TB SSD | Edge computing/storage |
| **Snowball Edge Storage** | 80TB usable | Large data migration |
| **Snowball Edge Compute** | 42TB + Compute | Edge computing + storage |
| **Snowmobile** | 100PB (truck!) | Exabyte-scale migration |

```
Quy trình Snow:
1. Order thiết bị từ AWS Console
2. AWS ship thiết bị đến bạn
3. Copy data vào thiết bị (offline)
4. Ship thiết bị về AWS
5. AWS upload data lên S3
6. Thiết bị được wipe (NIST 800-88 standard)
```

### FSx — Managed File Systems

| Service | Protocol | Dùng khi |
|---------|---------|---------|
| **FSx for Windows** | SMB, NFS | Windows apps, Active Directory integration |
| **FSx for Lustre** | Lustre (POSIX) | HPC, ML, Media processing, S3 integration |
| **FSx for NetApp ONTAP** | NFS, SMB, iSCSI | Enterprise storage, multi-protocol |
| **FSx for OpenZFS** | ZFS over NFS | Linux apps, high IOPS, ZFS features |

```
FSx for Lustre + S3:
  Input:  Read from S3 (lazy loading/eager loading)
  Output: Write back to S3

Use case: ML training cần read từ S3 nhanh (Lustre caches S3 data)
```

### Storage Gateway

> Hybrid cloud: kết nối on-premises với AWS Storage

| Gateway Type | Protocol | On-Prem | AWS |
|-------------|---------|---------|-----|
| **S3 File Gateway** | NFS, SMB | Local cache | S3 (Standard/IA/Glacier) |
| **FSx File Gateway** | SMB | Local cache | FSx for Windows |
| **Volume Gateway (Cached)** | iSCSI | Only cache | S3 (primary) + EBS snapshots |
| **Volume Gateway (Stored)** | iSCSI | Full dataset | S3 snapshots (async) |
| **Tape Gateway** | iSCSI VTL | Virtual tape | S3 Glacier |

```
Khi nào dùng gì:
- File share hybrid (Windows AD) → FSx File Gateway
- File share hybrid (Linux/general) → S3 File Gateway
- Block storage hybrid → Volume Gateway
- Backup on-prem to cloud → Tape Gateway
```

### AWS DataSync

```
Automated data transfer (online):
On-premises ──Agent──→ AWS (S3, EFS, FSx)
AWS Service  ─────────→ AWS Service

Speed: up to 10 Gbps per agent
Features: Encryption, compression, integrity check, scheduling
Schedule: Hourly/Daily/Weekly

vs Storage Gateway:
  DataSync:        Migration/Replication tasks
  Storage Gateway: Ongoing hybrid cloud access
```

---


## 11. SQS, SNS & Kinesis — Messaging

### SQS — Simple Queue Service

> Message Queue — Decouple producers và consumers

```
Standard Queue:
  - Unlimited throughput
  - At-least-once delivery (có thể duplicate!)
  - Best-effort ordering (không đảm bảo thứ tự)

FIFO Queue:
  - 300 msg/s (3,000 w/ batching)
  - Exactly-once delivery
  - Strict order
  - Tên PHẢI kết thúc bằng .fifo
```

**SQS Key Settings:**

| Setting | Default | Max | Ý nghĩa |
|---------|---------|-----|---------|
| Message Size | - | 256KB | Lớn hơn → gửi kèm S3 reference |
| Retention | 4 days | 14 days | Bao lâu message tồn tại |
| Visibility Timeout | 30s | 12h | Consumer phải xử lý trong thời gian này |
| Delivery Delay | 0 | 15 min | Delay trước khi message visible |
| Long Polling | 0s | 20s | Chờ message (giảm API calls/cost) |

**Dead Letter Queue (DLQ):**
```
Sau N lần receive nhưng không delete (xử lý lỗi):
→ Message chuyển vào DLQ
maxReceiveCount = số lần thử trước khi vào DLQ
Dùng để: debug, replay failed messages
```

### SNS — Simple Notification Service

> Pub/Sub — 1 publish → nhiều subscribers

```
Publisher → SNS Topic → SQS Queue 1
                     → SQS Queue 2
                     → Lambda Function
                     → HTTP/HTTPS Endpoint
                     → Email/SMS

SNS Fan-out Pattern:
  Application → SNS → SQS (Team A)
                    → SQS (Team B)
                    → Lambda (processing)
→ Đây là cách để S3 event đến nhiều consumers!
```

**SNS FIFO:**
- Strict ordering + deduplication
- **Chỉ** subscribers là SQS FIFO queues
- Kết hợp SNS FIFO + SQS FIFO = ordered fan-out

**SNS Message Filtering:**
```json
{"type": [{"prefix": "order"}]}
→ Subscriber này chỉ nhận messages có type bắt đầu = "order"
```

### Kinesis — Real-time Data Streaming

| Service | Mô tả | Use Case |
|---------|-------|---------|
| **Data Streams** | Stream, custom consumers, replay | Real-time analytics, ML |
| **Firehose** | Load to S3/Redshift/ES/Splunk | Near real-time ETL |
| **Data Analytics** | SQL on streams | Anomaly detection |
| **Video Streams** | Video ingestion | Monitoring, ML |

**Kinesis Data Streams:**
```
Producers → [Shard 1] → Consumers
            [Shard 2]
            [Shard N]

Per Shard:
  Input:  1 MB/s OR 1,000 records/s
  Output: 2 MB/s per consumer

Retention: 1 day (default) → 365 days
Immutable: Không thể xóa sau khi insert
Scale:     Add/remove shards (resharding)
```

### SQS vs SNS vs Kinesis (Hay nhầm!)

| | SQS | SNS | Kinesis |
|--|-----|-----|--------|
| Pattern | Queue (pull) | Pub/Sub (push) | Stream |
| Replay | ❌ | ❌ | ✅ |
| Ordering | FIFO only | FIFO only | Per shard |
| Real-time | Near | Near | ✅ |
| Consumers | 1 per message | Many | Many |
| Use case | Task queue, decouple | Notifications, fan-out | Analytics, ML pipeline |

---


## 12. Containers — ECS, Fargate, ECR, EKS

### ECS — Elastic Container Service

```
EC2 Launch Type:
  ✅ Bạn kiểm soát EC2 (cost saving, GPU, etc.)
  ❌ Phải provision và maintain EC2 instances
  ECS Agent chạy trên mỗi EC2

Fargate Launch Type:
  ✅ Serverless! Không cần quản lý EC2
  ✅ Chỉ define CPU/RAM per task
  ✅ AWS lo infrastructure
  ❌ Ít kiểm soát hơn EC2 launch type
```

**ECS Task Definition:** Blueprint cho container
```json
{
  "containerDefinitions": [{
    "name": "web-app",
    "image": "my-ecr-image:latest",
    "cpu": 256,
    "memory": 512,
    "portMappings": [{"containerPort": 80}],
    "environment": [{"name": "ENV", "value": "prod"}]
  }],
  "taskRoleArn": "arn:...",
  "executionRoleArn": "arn:..."
}
```

```
Task Role:          IAM role cho container gọi AWS services (S3, DDB...)
Execution Role:     IAM role cho ECS agent (pull image, write logs)
```

**ECS + ALB:** Dynamic Port Mapping
```
ALB → port random → ECS Task container port 80
→ Nhiều tasks cùng port trên cùng EC2 instance → ALB tự map
```

### Fargate

```
Just provide:
- Task Definition (CPU, RAM, image, env)
- VPC, Subnet, Security Group

AWS provides:
- EC2 instances (hidden)
- Networking
- Auto scaling
- Load balancing

Pay: per vCPU + Memory per second
```

### ECR — Elastic Container Registry

```
Private Docker registry:
→ Tích hợp với ECS, EKS, Lambda
→ Image scanning (CVE detection)
→ Versioning, lifecycle policies

Public: gallery.ecr.aws
Private: xxx.dkr.ecr.region.amazonaws.com
```

### EKS — Elastic Kubernetes Service

```
Managed Kubernetes control plane:
→ AWS manages etcd, API server, scheduler
→ Worker nodes: EC2 (managed node groups) hoặc Fargate

ECS vs EKS:
  ECS: AWS-proprietary, đơn giản, ít config
  EKS: Kubernetes-standard, portable, phức tạp hơn

Use EKS khi: Đang dùng Kubernetes on-prem, muốn migrate; cần Kubernetes ecosystem
```

### App Runner (Mới nhất, đơn giản nhất)

```
Deploy từ source code hoặc container image:
→ AWS tự build, deploy, scale, HTTPS
→ Không config gì (thực sự serverless)
→ Great for: Microservices, APIs, web apps nhanh
```

---


## 13. Serverless — Lambda, DynamoDB, API Gateway

### Lambda — Functions as a Service

```
Giới hạn Lambda:
- Execution time:    max 15 phút
- Memory:            128MB → 10GB
- /tmp:              512MB → 10GB
- Env vars:          4KB
- Deployment package: 50MB (zip), 250MB (unzipped)
- Concurrency:       1000 default (có thể tăng)

Pricing:
- $0.20 per 1M requests
- $0.0000166667 per GB-second
- Free tier: 1M requests, 400,000 GB-second/month
```

**Lambda Invocation Types:**

| Type | Who calls | Error handling |
|------|----------|--------------|
| Synchronous | API GW, ALB, Lambda@Edge | Caller handles |
| Asynchronous | S3, SNS, EventBridge | Lambda retries 2x + DLQ |
| Polling | SQS, DynamoDB Streams, Kinesis | Lambda processes batch |

**Lambda + VPC:**
```
Default: Lambda chạy trong AWS VPC → access internet, AWS services
         KHÔNG access private VPC resources (private RDS, ElastiCache)

Lambda trong VPC của bạn:
→ Specify: VPC, Subnets, Security Group
→ Internet access: qua NAT Gateway (KHÔNG phải Internet Gateway!)
→ AWS services: qua VPC Endpoints hoặc NAT GW → internet → AWS
```

**Lambda Destinations:**
```
Async invocation results:
Success → SQS/SNS/EventBridge/Lambda
Failure → SQS/SNS/EventBridge/Lambda

Better than DLQ (thêm context về function invocation)
```

### DynamoDB — Serverless NoSQL

```
Tables → Items (max 400KB) → Attributes
Primary Key:
  Option 1: Partition Key alone (unique)
  Option 2: Partition Key + Sort Key (combination unique)

Partition Key design: high cardinality (user_id, product_id)
→ Even distribution across partitions
```

**Capacity Modes:**

| | Provisioned | On-Demand |
|--|------------|----------|
| Billing | RCU + WCU/hour | Per request |
| Cost | Rẻ nếu predictable | Đắt ~2.5x khi so sánh |
| Auto-scale | Có (cần config) | Automatic |
| Dùng khi | Traffic biết trước | Unknown/spiky/new app |

```
RCU = Read Capacity Unit:  1 strongly consistent 4KB/s
                           2 eventually consistent 4KB/s
WCU = Write Capacity Unit: 1 x 1KB/s write
```

**DynamoDB Accelerator (DAX):**
```
DDB → DAX Cache → Application
Microsecond latency (vs single-digit ms của DDB)
No code change required
Cache: item cache (GetItem) + query cache
Không phải dùng ElastiCache Redis cho DDB workloads!
```

**DynamoDB Streams + Lambda:**
```
DDB change → Stream → Lambda →
  - Analytics
  - Cross-region replication
  - Notifications
  - ElasticSearch sync

Retention: 24 hours
```

**DynamoDB Global Tables:**
```
Multi-region, active-active:
  Region A (write + read)
  Region B (write + read)
  → SYNC via DDB Streams

Read/write in any region → low latency everywhere
Prerequisite: Enable DDB Streams
```

**DynamoDB TTL:**
```
Set TTL attribute (Unix timestamp)
DDB deletes expired items automatically (no extra cost)
Deleted within 48 hours of expiry
Use case: Sessions, cache, temp data
```

### API Gateway

> Fully managed REST, HTTP, WebSocket APIs

**Features:**
```
- Auth: IAM, Cognito User Pools, Lambda Authorizer
- Rate limiting & throttling
- API versioning (/v1, /v2)
- Usage plans + API keys
- SDK generation
- OpenAPI spec import/export
- Request/Response transformation
- Caching (up to 237GB)
```

**Endpoint Types:**

| Type | Use Case |
|------|---------|
| Regional | Default, low latency trong 1 Region |
| Edge-Optimized | CloudFront → low latency for global clients |
| Private | VPC only, không access từ internet |

### Cognito — User Authentication

```
Cognito User Pools (CUP):
  → Login/Signup cho app users
  → Returns JWT tokens
  → Federated: Google, Facebook, SAML
  → Integrate: API Gateway, ALB

Cognito Identity Pools (Federated Identity):
  → Exchange any token → AWS credentials (STS)
  → Access AWS services directly (S3, DDB)
  → Support unauthenticated guest access

Flow:
User → CUP (authenticate) → JWT token
JWT → API Gateway (validate JWT with CUP)
    → ALB (native integration)
JWT → Identity Pool → AssumeRoleWithWebIdentity → AWS temp creds → S3/DDB
```

---


## 14. VPC & Networking

> VPC = Virtual Private Cloud = Mạng riêng ảo trong AWS

### VPC Architecture

```
VPC (10.0.0.0/16) — max /16 CIDR
├── Public Subnet  (10.0.1.0/24) ── có route đến Internet Gateway
│   └── Resources: Web servers, NAT Gateway, Bastion host
├── Private Subnet (10.0.2.0/24) ── không có route IGW
│   └── Resources: App servers, databases
└── Isolated Subnet — không route gì cả (maximum security)

Multi-AZ: Tạo subnets ở mỗi AZ cho HA
Default VPC: Tự tạo khi tạo account, public subnets, đã có IGW
```

### Internet Gateway vs NAT Gateway

```
Internet Gateway (IGW):
  ✅ Cho phép VPC connect Internet (2-way)
  → Attach vào VPC
  → Add route: 0.0.0.0/0 → IGW trong subnet Route Table

NAT Gateway (AWS-managed):
  ✅ Private subnet → Internet (1-way, outbound only)
  → Phải đặt trong PUBLIC subnet
  → Cần Elastic IP
  → HA trong 1 AZ; tạo 1 NAT GW per AZ cho full HA
  → Không thể làm Bastion host

NAT Instance (EC2-based, old):
  ❌ Phải tự manage
  ✓ Có thể làm Bastion
  ✓ Rẻ hơn NAT GW
  → Phải disable Source/Destination Check
```

### Security Groups vs NACLs

| | Security Groups | NACLs |
|--|----------------|------|
| Level | **Instance** level | **Subnet** level |
| Rules | Allow only | Allow + **Deny** |
| State | **Stateful** | **Stateless** |
| Default inbound | Block all | Allow all |
| Rule evaluation | All rules | By number (lowest first), stop at match |

```
Stateful  = Return traffic tự động allowed (SG)
Stateless = Phải define BOTH inbound AND outbound (NACL)

Ví dụ NACL Stateless:
Allow inbound port 80 → cũng phải allow outbound ephemeral ports (1024-65535)
```

### VPC Peering

```
VPC A (10.0.0.0/16) ←──peering──→ VPC B (172.16.0.0/16)

Rules:
❌ CIDRs KHÔNG được overlap!
❌ Non-transitive: A↔B, B↔C ≠ A↔C (phải tạo riêng A↔C)
✅ Cross-account OK
✅ Cross-region OK (không qua internet)
→ Update Route Tables để route traffic
```

### VPC Endpoints — Private AWS Service Access

```
Không muốn traffic EC2 → S3/DDB qua internet?
→ VPC Endpoints!

Gateway Endpoint (FREE!):
  → S3 và DynamoDB ONLY
  → Modify Route Table để route đến endpoint
  → No extra charge

Interface Endpoint (PrivateLink):
  → Hầu hết AWS services (SSM, KMS, API GW, SQS, SNS...)
  → Creates ENI trong subnet
  → ~$0.01/hour + data processing fee
```

### Site-to-Site VPN

```
On-premises ──encrypted tunnel over internet──→ AWS VPC

Components:
  Customer Gateway (CGW): on-premises side (device/software)
  Virtual Private Gateway (VGW): AWS side (attached to VPC)
  VPN Connection: encrypted connection between CGW and VGW

Bandwidth: Limited by internet connection
Setup: Minutes
Use when: Quick setup, cost-sensitive, internet OK
```

### AWS Direct Connect (DX)

```
On-premises ──dedicated physical line──→ AWS Direct Connect Location → VPC

Bandwidth: 1Gbps, 10Gbps, 100Gbps
Setup: Weeks to months (physical infrastructure)
Benefits:
  ✅ Consistent, high bandwidth
  ✅ Lower latency
  ✅ Private connection
  ✅ Cost reduction for large data transfer

DX + VPN = Encrypted Direct Connect (IPSec over DX)

DX Gateway: Connect DX to multiple VPCs across Regions
```

### Transit Gateway

```
Problem: Many VPCs need to connect to each other
         N VPCs = N*(N-1)/2 peering connections!

Solution: Transit Gateway (hub and spoke)
  All VPCs → Transit Gateway ← On-premises (VPN/DX)

  N VPCs = N connections!
  Regional service (cross-account OK)
  Route tables per attachment
  ECMP routing for bandwidth scaling
  Cross-region peering
```

### VPC Flow Logs

```
Capture IP traffic information:
  VPC, Subnet, or ENI level

Destination: S3, CloudWatch Logs, Kinesis Firehose

Fields: srcaddr, dstaddr, srcport, dstport, protocol, action (ACCEPT/REJECT)
Use case: Troubleshoot connectivity, security audit, traffic analysis
```

### Bastion Host

```
Access private EC2 via internet:
Internet → Bastion (public subnet, SSH 22) → Private EC2 (SSH 22)

Security:
  Bastion SG: Allow SSH from YOUR IP only (not 0.0.0.0/0!)
  Private EC2 SG: Allow SSH from Bastion's Security Group
```

---


## 15. Security — KMS, SSM, Shield, WAF

### KMS — Key Management Service

```
Customer Master Keys (CMK):
  AWS Managed:     Free, auto-rotate, controlled by AWS services
  Customer Managed: $1/month, you control, manual/auto rotation
  CloudHSM Keys:   Generated in your dedicated HSM hardware

Key Types:
  Symmetric (AES-256): Same key for encrypt/decrypt
  Asymmetric (RSA/ECC): Public key encrypt, Private key decrypt

API:
  Encrypt: max 4KB directly
  GenerateDataKey: For data > 4KB (Envelope Encryption!)
```

**Envelope Encryption (cực quan trọng!):**
```
Problem: KMS can only encrypt up to 4KB
Solution: Envelope Encryption

1. GenerateDataKey(CMK) → Data Key (plaintext + encrypted)
2. Encrypt large file with plaintext Data Key (locally)
3. Store encrypted file + encrypted Data Key together
4. Discard plaintext Data Key

Decrypt:
1. Decrypt Data Key using KMS (send encrypted data key)
2. KMS returns plaintext Data Key
3. Decrypt file locally with plaintext Data Key
```

**KMS Key Policies:**
```json
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::123456:root"},
  "Action": "kms:*",
  "Resource": "*"
}
```

### SSM Parameter Store

```
Store config + secrets:
/myapp/dev/db-url         → String (plain text)
/myapp/prod/db-password   → SecureString (KMS encrypted!)
/myapp/config/list        → StringList

Tiers:
  Standard: Free, 10K params, max 4KB, no auto rotation
  Advanced: $0.05/param, 100K params, 8KB, auto rotation via Lambda
```

### Secrets Manager

```
vs SSM Parameter Store:
  ✅ Auto-rotation built-in (via Lambda)
  ✅ Native RDS/Aurora/Redshift/DocumentDB integration
  ✅ Cross-account access
  ❌ $0.40/secret/month (SSM Standard = free)

Use Secrets Manager when:
  - Database credentials (auto-rotate!)
  - API keys needing rotation
  - Cross-account

Use SSM Parameter Store when:
  - General config (not needing rotation)
  - Cost-sensitive
```

### ACM — Certificate Manager

```
Free SSL/TLS certificates for:
  ✅ ALB, CloudFront, API Gateway, Elastic Beanstalk

Cannot use directly on:
  ❌ EC2 (use ACM Private CA or manual install)

Auto-renewal: Yes!
```

### WAF — Web Application Firewall

```
Deploy on: ALB, API Gateway, CloudFront, AppSync GraphQL

Web ACL (Access Control List) Rules:
  ✅ IP Sets: block specific IPs/CIDRs
  ✅ SQL Injection protection
  ✅ XSS (Cross-Site Scripting) protection
  ✅ Rate-based rules (Layer 7 DDoS protection)
  ✅ Geographic match (block/allow countries)
  ✅ Managed Rules (OWASP Top 10, Bot Control, etc.)
  ✅ String/regex matching

Pricing: $5/Web ACL/month + $1/rule + $0.60/M requests
```

### Shield — DDoS Protection

| | Standard | Advanced |
|--|---------|---------|
| Cost | **FREE** for all AWS customers | $3,000/month |
| Protection | L3/L4 (network/transport) | L3/L4/L7 (application too) |
| Response | Auto | 24/7 DRT (DDoS Response Team) |
| Resources | All | ALB, EC2, CloudFront, Route53, Global Accelerator |
| Cost protection | ❌ | ✅ No spike charges during attack |

### GuardDuty — Intelligent Threat Detection

```
Input sources (no agent needed!):
  - CloudTrail Events Logs
  - VPC Flow Logs
  - DNS Logs
  - EKS/K8s Audit Logs
  - RDS Login Activity
  - S3 Data Events

Output: Findings → CloudWatch Events → Lambda/SNS

Detects:
  - Cryptocurrency mining
  - Unusual API calls
  - Compromised EC2/credentials
  - Malicious IPs

30-day free trial | No infrastructure to manage
```

### Inspector — Automated Security Assessments

```
What it scans:
  EC2:    OS vulnerabilities (CVEs), network reachability (needs SSM agent)
  ECR:    Container image vulnerabilities
  Lambda: Function code vulnerabilities

Reports to: Security Hub, EventBridge
Risk score = severity
```

### Macie — Data Security for S3

```
ML-powered sensitive data discovery in S3:
  - PII (Names, SSNs, credit cards)
  - Financial data
  - Healthcare data

Alerts via EventBridge
Use case: Compliance (GDPR, HIPAA), data governance
```

---


## 16. Monitoring — CloudWatch, CloudTrail, Config

### CloudWatch — Metrics & Monitoring

```
Metrics: CPU, NetworkIn/Out, DiskRead/Write... (every 5 min default)
Detailed Monitoring: Every 1 minute (EC2 thêm $)
Custom Metrics: Push via API (RAM, disk space, app metrics)

Namespaces:
  AWS/EC2:        EC2 metrics
  AWS/RDS:        RDS metrics
  CWAgent:        Agent-based metrics (RAM, disk)
  Custom:         bạn tự push

Resolution:
  Standard: 1 minute
  High-Res: 1/5/10/30 seconds (custom metrics only)
```

**CloudWatch Alarms:**
```
States: OK | ALARM | INSUFFICIENT_DATA

Actions when ALARM:
  EC2 Action:    Stop, Terminate, Reboot, Recover instance
  Auto Scaling:  Scale in/out
  SNS:           Notify (email, SMS, Lambda, SQS...)

Composite Alarms: AND/OR logic trên nhiều alarms
```

**CloudWatch Logs:**
```
Log Groups: /aws/lambda/function-name
Log Streams: per instance, per container

Sources:
  - SDK / CloudWatch Logs Agent / Unified Agent
  - Lambda (automatic)
  - ECS containers
  - RDS enhanced monitoring
  - API Gateway
  - CloudTrail
  - Route 53

Export: S3 (up to 12h delay) | Kinesis (real-time) | Firehose
```

**CloudWatch Insights:**
```
Container Insights: ECS, EKS, K8s, Fargate → metrics + logs
Lambda Insights:    Cold starts, duration, errors
Contributor Insights: Top-N analysis (top IPs, URLs...)
Application Insights: .NET, SQL Server apps monitoring
```

### CloudTrail — API Auditing

> Record mọi API call trong AWS account

```
Event Types:
  Management Events: tạo/xóa resources (default ON)
  Data Events:       S3 object ops, Lambda invocations (OFF, có phí)
  Insights Events:   Detect unusual activity patterns

Retention: 90 days (CloudTrail console)
Long-term: Gửi vào S3 (unlimited)

Dùng để:
  ✅ "Ai đã xóa EC2 lúc 3am?" → CloudTrail
  ✅ Compliance audit
  ✅ Security investigation
  ✅ Change management
```

**CloudTrail Lake:**
```
Aggregated storage + SQL queries trực tiếp trên CloudTrail events
Retention: 7 years
Use case: Long-term audit, compliance queries
```

### AWS Config — Configuration Compliance

> Track resource configurations + compliance over time

```
Config Rules:
  AWS Managed:  predefined rules (100+)
  Custom:       Lambda-based custom rules

Examples:
  restricted-ssh:               Không SG nào mở port 22 với 0.0.0.0/0
  s3-bucket-public-read:        S3 not public
  encrypted-volumes:            EBS phải encrypted
  rds-instance-public-access:   RDS không public

Remediation:
  Auto-remediation via SSM Automation
  Manual remediation

Per region → Aggregate với Config Aggregator (cross-account/region)
```

### So Sánh 3 Services

| | CloudWatch | CloudTrail | Config |
|--|-----------|-----------|--------|
| **Hỏi gì** | Performance metrics | API call audit | Config compliance |
| **Câu hỏi** | CPU cao không? Memory? | Ai đã làm gì? Khi nào? | Config có đúng policy không? |
| **Real-time** | ✅ Near | ✅ Near | ✅ Near |
| **History** | Limited | 90 days | ✅ Full history |
| **Alerts** | Alarms, Events | EventBridge | EventBridge |

---


## 17. IAM Advanced & Organizations

### AWS Organizations

```
Management Account (Root)
├── OU (Organizational Unit): Dev
│   ├── Account A (Dev frontend)
│   └── Account B (Dev backend)
├── OU: Production
│   ├── Account C (Prod web)
│   └── Account D (Prod database)
└── OU: Sandbox
    └── Account E

Benefits:
✅ Consolidated billing (1 hóa đơn)
✅ Volume discounts (aggregated usage)
✅ Share Reserved Instances across accounts
✅ Central security/logging/governance
✅ Easy account creation via API
```

### SCPs — Service Control Policies

```
Permission guardrails cho OU/Account:
  → Áp dụng hierarchically (OU → account)
  → Affect ALL users + roles kể cả ROOT của member accounts!
  → KHÔNG affect Management Account

SCP Deny > IAM Allow (khi có conflict)
SCP không GRANT permissions — chỉ SET BOUNDARIES

Use case:
  - Chặn EC2 ở expensive regions (chỉ cho phép us-east-1)
  - Chặn delete CloudTrail
  - Chặn create IAM users trong sandbox
```

### IAM Roles Advanced

**Cross-Account Access:**
```
Account A User → sts:AssumeRole → Account B Role → B's resources

Account B Role Trust Policy:
{
  "Principal": {"AWS": "arn:aws:iam::ACCOUNT-A-ID:root"},
  "Action": "sts:AssumeRole"
}
```

**Common Service Roles:**
```
EC2 Instance Profile → EC2 → call S3, DDB, SSM...
Lambda Execution Role → Lambda → CloudWatch, S3, DDB...
ECS Task Role → Container → S3, DDB...
CodeDeploy Service Role → Deploy to EC2/ECS...
```

### Permission Boundaries

```
Effective permissions = IAM Policy ∩ Permission Boundary

Example:
  Permission Boundary: Allow S3, EC2
  IAM Policy:          Allow S3, EC2, RDS
  Effective:           Allow S3, EC2 (intersection)

Use case:
  Allow developers to create IAM roles/users
  But prevent them from granting more permissions than they have
  → Delegate IAM management safely
```

### IAM Identity Center (SSO)

```
Single Sign-On for:
  Multiple AWS Accounts (Organizations)
  SAML 2.0 Business Apps (Salesforce, Slack, Office365)

Identity Sources:
  Built-in Identity Store
  Active Directory (AD Connector or AWS Managed AD)
  External IdP (Okta, Azure AD via SAML/OIDC)

Benefits:
  One login → access many AWS accounts + applications
  Centralized permissions management
  Fine-grained permissions via Permission Sets
```

### AWS Directory Service

| Service | Mô tả | Dùng khi |
|---------|-------|---------|
| AWS Managed AD | Full Microsoft AD in AWS | Large enterprise, trusts with on-prem AD |
| AD Connector | Proxy to on-prem AD | Keep directory on-prem |
| Simple AD | Standalone Samba-based | Small simple directory |

---


## 18. Disaster Recovery & Migrations

### RTO vs RPO

```
RPO (Recovery Point Objective): Bao nhiêu data loss có thể chấp nhận?
  RPO = 1 hour → backup mỗi giờ → mất tối đa 1h data
  RPO gần 0    → sync replication required

RTO (Recovery Time Objective): Bao lâu system phải up lại?
  RTO = 4 hours → downtime không quá 4 giờ
  RTO gần 0     → active-active setup
```

### DR Strategies (Cost vs RTO/RPO)

```
1. BACKUP & RESTORE (cheapest, slowest)
   → Backup to S3/Glacier regularly
   → Restore to AWS when disaster
   RPO: Hours | RTO: Hours | Cost: $$

2. PILOT LIGHT
   → Core services running minimally in AWS (DB sync)
   → Disaster: scale up, start remaining services
   RPO: Minutes | RTO: 10s of minutes | Cost: $$$

3. WARM STANDBY
   → Full system at AWS, but scaled down (min capacity)
   → Disaster: scale up to production size
   RPO: Minutes | RTO: Minutes | Cost: $$$$

4. MULTI-SITE / HOT STANDBY (most expensive)
   → 100% scale in AWS and On-prem simultaneously
   → DNS failover
   RPO: Real-time | RTO: Real-time | Cost: $$$$$
```

> ⬇️ Cost increases | ⬆️ RTO/RPO improves
> Backup & Restore → Pilot Light → Warm Standby → Multi-Site

### DMS — Database Migration Service

```
Migrate databases to AWS:
  Homogeneous:  MySQL → RDS MySQL (easy)
  Heterogeneous: Oracle → Aurora (need Schema Conversion Tool!)

Features:
  Continuous Data Replication (CDC): live sync during migration
  Minimal downtime
  Supports encryption

Sources: Oracle, SQL Server, MySQL, PostgreSQL, MongoDB, S3, SAP...
Targets: RDS, Aurora, Redshift, DynamoDB, S3, ElasticSearch, Kafka...

Server: DMS runs on EC2 instance you provision
```

### MGN — Application Migration Service

```
Lift-and-shift physical/virtual/cloud servers:

1. Install MGN replication agent on source
2. Continuous block-level replication → AWS staging
3. Test in AWS (cutover test)
4. Final cutover: redirect traffic to AWS
5. Minimal downtime (hours)

Replaces: CloudEndure Migration (old)
```

### AWS Backup

```
Centralized backup service:
  - EC2, EBS, RDS, Aurora, DynamoDB, EFS, FSx, S3, Storage Gateway

Features:
  Backup Plans: schedule, retention, lifecycle
  Cross-account cross-region backup
  Incremental backups (deduplicated)
  AWS Backup Vault Lock: WORM (cannot delete backups)
```

### Disaster Recovery Tips

```
On-prem to AWS:
  Site-to-Site VPN: Quick setup, encrypted (but uses internet)
  Direct Connect: Dedicated, fast (weeks to setup)

Backups:
  On-prem → S3/Glacier via DataSync or Storage Gateway
  EBS Snapshots → cross-region copy
  RDS Snapshots → cross-region copy

DNS Failover:
  Route 53 Failover routing + Health Checks
  → Automatic from Primary to Secondary
```

---


## 19. Other Services

### Elastic Beanstalk — PaaS

```
Upload code → Beanstalk manages:
  EC2, Auto Scaling, ELB, CloudWatch, HTTPS...

Languages: Node.js, Python, Java, .NET, PHP, Ruby, Go, Docker
Free service (trả tiền cho underlying resources)

Deployment Modes:
  All at once:        Nhanh nhất, downtime
  Rolling:            Không downtime, giảm capacity tạm thời
  Rolling w/batch:    Không downtime, không giảm capacity
  Immutable:          Tạo instances mới, zero downtime
  Blue/Green:         Separate environment, swap DNS (Route 53)
  Traffic Splitting:  Canary testing

Environments: Web Server vs Worker
```

### CloudFormation — Infrastructure as Code

```
Template (YAML/JSON) → CloudFormation → AWS Resources (Stack)

Benefits:
✅ Infrastructure as Code (version control)
✅ Reproducible environments
✅ No manual config (avoid mistakes)
✅ Cost estimation via template
✅ Dependency management

Concepts:
  Stack:         Group of resources from 1 template
  StackSet:      Deploy stack to multiple accounts/regions
  Change Set:    Preview changes before apply
  Drift:         Actual vs Template mismatch
  Nested Stacks: Reusable templates
```

### CDK — Cloud Development Kit

```
Define infrastructure in code:
  TypeScript, Python, Java, C#, Go

CDK → CloudFormation template → Deploy

Benefits vs CloudFormation:
✅ Use real programming language
✅ Reusable components
✅ IDE support, type checking
```

### SES — Simple Email Service

```
Send/Receive emails at scale:
  Transactional: order confirmations, password resets
  Marketing: newsletters
  Automated: alerts, reports

Integrate: S3, SNS, Lambda
```

### AWS Batch

```
Fully managed batch jobs:
  NO time limit (vs Lambda 15min)
  Any runtime (Docker container)
  Dynamic: provision EC2/Spot

Schedule: EventBridge → Batch

vs Lambda:
  Lambda: Serverless, event-driven, 15min max
  Batch:  Long-running, container-based, unlimited time
```

### Amazon Athena

```
Serverless SQL on S3:
  Query CSV, JSON, Parquet, ORC, Avro in S3
  Pricing: $5 per TB scanned
  Save money: Use columnar (Parquet/ORC), compression, partition

Use case:
  - Ad-hoc data analysis
  - VPC/CloudTrail/ALB logs analysis
  - BI with QuickSight

Federated Query: Query other sources (DDB, RDS, on-prem) via Lambda
```

### Amazon Redshift

```
OLAP Data Warehouse (columnar):
  Scale: petabytes
  Based on PostgreSQL (familiar SQL)
  Massively Parallel Processing (MPP)

vs RDS:
  RDS = OLTP (transactions, many rows, row-based)
  Redshift = OLAP (analytics, aggregations, column-based)

Redshift Features:
  Cluster: Leader node + Compute nodes
  Spectrum: Query S3 directly (no loading)
  Enhanced VPC Routing: data through VPC
  Multi-AZ: RA3 node type supports
```

### Amazon EMR — Elastic MapReduce

```
Managed Hadoop ecosystem:
  Apache Spark, Hive, HBase, Presto, Flink

Instance Types:
  Master node:  Manages cluster
  Core nodes:   Run tasks + store data (HDFS)
  Task nodes:   Run tasks only (no storage) → use Spot!

Use case: ML, log analysis, ETL, genomics
```

### Amazon QuickSight

```
Serverless BI tool:
  Dashboards, visualizations
  Connect to: RDS, Aurora, Redshift, Athena, S3, on-prem

SPICE: Super-fast in-memory calculation engine (cached data)
Row-level security: Fine-grained access control
ML Insights: Anomaly detection, forecasting, Q&A

vs Athena:
  Athena: Query engine (SQL)
  QuickSight: Visualization (dashboards)
```

### AWS Glue

```
Serverless ETL:
  Extract → Transform → Load to Redshift/S3/RDS

Components:
  Glue Crawlers:       Discover schema, create Data Catalog
  Glue Data Catalog:   Central metadata repository
  Glue ETL Jobs:       Spark-based transformations
  Glue Studio:         Visual ETL builder

Glue Bookmarks: Track processed data (avoid reprocessing)
Glue Elastic Views: Materialized views combining data sources
```

### Step Functions — Workflow Orchestration

```
Visual workflows using state machines (JSON-based):
  Task:       Invoke Lambda, ECS, DDB, SNS...
  Choice:     Conditional branching
  Parallel:   Run branches in parallel
  Wait:       Time delays
  Map:        Process array items

Features:
  Error handling + retries
  Human approval steps
  Timeout per step
  Execution history

vs SQS: SQS = async messaging; Step Functions = orchestration
```

### Amazon MQ — Managed Message Broker

```
For existing apps using open protocols:
  ActiveMQ, RabbitMQ

Protocols: MQTT, AMQP, STOMP, NMS, OpenWire, WSS

vs SQS/SNS:
  SQS/SNS: AWS-native, cloud-first, unlimited scale
  Amazon MQ: For migration from on-prem ActiveMQ/RabbitMQ

Use when: "We use RabbitMQ on-prem and want to migrate"
```

### AWS AppSync

```
Managed GraphQL service:
  Real-time data with WebSocket
  Offline support (mobile apps)

Data Sources: DDB, Lambda, HTTP, RDS, ES

vs API Gateway:
  API GW = REST/HTTP
  AppSync = GraphQL
```

### Key ML Services (Thi có thể hỏi)

| Service | Does what |
|---------|----------|
| **Rekognition** | Image/video analysis (faces, objects, moderation) |
| **Comprehend** | NLP: sentiment, entities, language, PII detection |
| **Transcribe** | Speech to text |
| **Polly** | Text to speech |
| **Translate** | Language translation |
| **Lex** | Chatbots (powers Alexa) |
| **Forecast** | Time-series forecasting |
| **Kendra** | ML-powered search |
| **SageMaker** | Full ML platform (build, train, deploy) |
| **Textract** | Extract text from documents (+forms, tables) |

---


## 20. Câu Hỏi Tổng Hợp 50 Câu

### IAM & Security

**Q1.** Công ty cần đảm bảo không account nào trong AWS Organizations có thể create EC2 in ap-southeast-1, kể cả root users. Giải pháp?
- A. IAM Policies
- B. **SCP (Service Control Policies)** ✅
- C. Permission Boundaries
- D. Resource Policies

**Q2.** EC2 cần đọc từ S3. Best practice security?
- A. Hardcode access key
- B. **IAM Instance Profile/Role** ✅
- C. Store credentials in /tmp
- D. Use root credentials

**Q3.** Database password cần auto-rotate mỗi 90 ngày. Service nào?
- A. SSM Parameter Store
- B. **Secrets Manager** ✅
- C. KMS
- D. IAM

**Q4.** Protect web app từ SQL injection và XSS?
- A. Security Groups
- B. NACLs
- C. **AWS WAF** ✅
- D. GuardDuty

**Q5.** Encrypt data lớn hơn 4KB với KMS. Kỹ thuật nào?
- A. Direct encryption với CMK
- B. **Envelope Encryption** ✅
- C. Client-side encryption
- D. SSE-S3

### EC2 & Compute

**Q6.** Chạy batch processing job 4 giờ, có thể bị interrupt. Rẻ nhất?
- A. On-Demand
- B. Reserved
- C. **Spot Instances** ✅
- D. Dedicated

**Q7.** 10 EC2 cần ultra-low latency networking, 10Gbps. Placement Group?
- A. Spread
- B. Partition
- C. **Cluster** ✅
- D. Không cần

**Q8.** App server cần giữ nguyên RAM state khi tắt máy ban đêm, boot nhanh?
- A. Stop
- B. Terminate
- C. **Hibernate** ✅
- D. Reboot

**Q9.** Container app không muốn manage EC2 servers. Service?
- A. ECS EC2
- B. **ECS Fargate** ✅
- C. EKS
- D. Lambda

**Q10.** EC2 cần I/O cực cao (millions IOPS), data có thể mất khi restart?
- A. EBS gp3
- B. EBS io2
- C. **EC2 Instance Store** ✅
- D. EFS

### Storage

**Q11.** S3 data accessed once per quarter, cần immediate retrieval?
- A. S3 Standard
- B. S3 Standard-IA
- C. **S3 Glacier Instant Retrieval** ✅
- D. S3 Glacier Deep Archive

**Q12.** Shared file system for multiple Linux EC2 instances, NFS protocol?
- A. EBS
- B. **EFS** ✅
- C. Instance Store
- D. S3

**Q13.** On-prem server backup to AWS via SMB file share?
- A. DataSync
- B. **S3 File Gateway** ✅
- C. Direct Connect
- D. Snowball

**Q14.** 50TB data migration, internet 100Mbps (takes 46 days)?
- A. DataSync
- B. Direct Connect
- C. **Snowball Edge** ✅
- D. S3 Transfer Acceleration

**Q15.** Prevent S3 objects from being deleted for 7 years (compliance)?
- A. S3 Versioning
- B. S3 Lifecycle
- C. **S3 Object Lock (Compliance mode)** ✅
- D. S3 Replication

### Databases

**Q16.** RDS cần scale READ traffic. Giải pháp?
- A. RDS Multi-AZ
- B. **RDS Read Replicas** ✅
- C. RDS Proxy
- D. Aurora Serverless

**Q17.** Aurora cần survive regional failure with sub-second RPO?
- A. Aurora Multi-AZ
- B. **Aurora Global Database** ✅
- C. Aurora Multi-Master
- D. Aurora Serverless

**Q18.** DynamoDB queries slow, cần microsecond latency?
- A. Increase RCU
- B. Global Tables
- C. **DAX** ✅
- D. DDB Streams

**Q19.** Cache cho web session, cần persist if cache fails?
- A. Memcached
- B. **Redis** ✅
- C. DynamoDB
- D. EFS

**Q20.** Migrate Oracle database to Aurora PostgreSQL với minimum downtime?
- A. Dump and restore
- B. **DMS + Schema Conversion Tool** ✅
- C. MGN
- D. Snowball

### Networking

**Q21.** Private EC2 cần download từ internet?
- A. Internet Gateway
- B. **NAT Gateway** ✅
- C. VPC Endpoint
- D. VPN

**Q22.** EC2 access S3, không muốn traffic qua internet, free?
- A. NAT Gateway
- B. Interface Endpoint
- C. **Gateway VPC Endpoint** ✅ (free for S3/DDB)
- D. Direct Connect

**Q23.** Connect 20 VPCs với nhau efficiently?
- A. VPC Peering (190 connections)
- B. **Transit Gateway** ✅ (20 connections)
- C. Direct Connect
- D. PrivateLink

**Q24.** DNS: Route 10% traffic to new app version?
- A. Latency routing
- B. **Weighted routing** ✅
- C. Failover routing
- D. Geolocation

**Q25.** App ở us-east-1 và eu-west-1, route users to lowest latency?
- A. Weighted
- B. **Latency-based** ✅
- C. Geolocation
- D. Failover

### HA & Scalability

**Q26.** Web app: route /api/* to API servers and /web/* to web servers?
- A. NLB
- B. **ALB path-based routing** ✅
- C. Route 53
- D. CloudFront

**Q27.** Scale down EC2 automatically when traffic decreases?
- A. Manual scaling
- B. **ASG with Target Tracking** ✅
- C. Reserved Instances
- D. Spot fleet

**Q28.** Load balancer for gaming (TCP/UDP) with static IP for firewall whitelist?
- A. ALB
- B. **NLB** ✅
- C. CLB
- D. GWLB

### Serverless & Messaging

**Q29.** Process orders in exact sequence, no duplicates?
- A. SQS Standard
- B. **SQS FIFO** ✅
- C. SNS
- D. EventBridge

**Q30.** S3 upload event → 3 different processing teams simultaneously?
- A. S3 → 3 SQS
- B. **S3 → SNS → 3 SQS** ✅ (fan-out)
- C. S3 → Lambda → 3 SQS
- D. S3 → EventBridge → 3 SQS (also valid alternative)

**Q31.** 50K IoT messages/second, real-time analysis, replay needed?
- A. SQS
- B. SNS
- C. **Kinesis Data Streams** ✅
- D. Amazon MQ

**Q32.** Orchestrate multiple Lambda functions with conditions, retries, error handling?
- A. Lambda Destinations
- B. SQS
- C. **Step Functions** ✅
- D. EventBridge

### Monitoring & Compliance

**Q33.** Find out who deleted an S3 bucket at 2am yesterday?
- A. CloudWatch
- B. **CloudTrail** ✅
- C. AWS Config
- D. GuardDuty

**Q34.** Ensure all EBS volumes are encrypted, alert if not?
- A. CloudTrail
- B. CloudWatch
- C. **AWS Config** ✅
- D. Inspector

**Q35.** Monitor EC2 RAM utilisation (not in default metrics)?
- A. CloudWatch default metrics
- B. **CloudWatch custom metrics + CW Agent** ✅
- C. CloudTrail
- D. Inspector

### Architecture

**Q36.** Low-cost DR: RPO 4h, RTO 4h?
- A. Multi-Site
- B. Warm Standby
- C. Pilot Light
- D. **Backup & Restore** ✅

**Q37.** Serve static website to global users, low latency?
- A. EC2 in multiple regions
- B. Route 53 latency routing
- C. **S3 + CloudFront** ✅
- D. Global Accelerator

**Q38.** On-premises needs consistent 10Gbps private link to AWS?
- A. Site-to-Site VPN
- B. **Direct Connect** ✅
- C. VPN + Transit Gateway
- D. PrivateLink

**Q39.** AD users need SSO to multiple AWS accounts?
- A. IAM Users
- B. Cognito
- C. **IAM Identity Center + AD Connector** ✅
- D. Root Account

**Q40.** Deploy Node.js app fast, no infra management?
- A. EC2
- B. ECS EC2
- C. **Elastic Beanstalk** ✅
- D. Lambda (phù hợp nhưng app framework phức tạp hơn)

### Data & Analytics

**Q41.** Run SQL on VPC Flow Logs stored in S3?
- A. Redshift
- B. EMR
- C. **Athena** ✅
- D. QuickSight

**Q42.** BI dashboard for business users from Redshift data?
- A. Athena
- B. **QuickSight** ✅
- C. CloudWatch
- D. Kibana

**Q43.** Daily ETL: S3 → transform → Redshift?
- A. Lambda
- B. **Glue** ✅
- C. Step Functions
- D. Batch

**Q44.** Moderate uploaded images for inappropriate content?
- A. Comprehend
- B. **Rekognition** ✅
- C. SageMaker
- D. Textract

**Q45.** Analyze customer feedback sentiment?
- A. Rekognition
- B. **Comprehend** ✅
- C. Lex
- D. Transcribe

### Mixed Architecture

**Q46.** S3: Finance team can only access /finance/*, Sales only /sales/*?
- A. 2 separate buckets
- B. IAM user policies
- C. **S3 Access Points** ✅
- D. Complex bucket policy

**Q47.** App uses RabbitMQ on-prem, migrate to AWS with minimal code change?
- A. SQS
- B. SNS
- C. **Amazon MQ** ✅
- D. Kinesis

**Q48.** Lambda needs to access private RDS in VPC. What's needed?
- A. Internet Gateway
- B. VPN
- C. **Configure Lambda to run in VPC** ✅ (same subnets as RDS, NAT GW for internet)
- D. VPC Peering

**Q49.** App needs to call multiple AWS APIs across accounts. Best auth?
- A. IAM Users
- B. Root Account
- C. **IAM Roles + AssumeRole** ✅
- D. Access Keys

**Q50.** Infrastructure for dev/staging/prod should be identical and repeatable?
- A. Manual AWS Console
- B. Ansible
- C. **CloudFormation** ✅ (or CDK/Terraform)
- D. AWS Config

---

## 📊 Cheat Sheet Tổng Hợp

### Chọn Đúng Service — Quick Guide

```
"decouple / async"          → SQS
"fan-out"                   → SNS → multiple SQS
"real-time stream + replay" → Kinesis Data Streams
"ordering + exactly-once"   → SQS FIFO
"workflow orchestration"    → Step Functions
"serverless compute"        → Lambda
"container serverless"      → ECS Fargate
"container + kubernetes"    → EKS
"ML training/deployment"    → SageMaker
"image/video AI"            → Rekognition
"NLP/sentiment"             → Comprehend
"speech-to-text"            → Transcribe
"chatbot"                   → Lex
"SQL on S3"                 → Athena
"BI dashboard"              → QuickSight
"ETL"                       → Glue
"data warehouse"            → Redshift
"CDN"                       → CloudFront
"DNS"                       → Route 53
"HTTP load balancer"        → ALB
"TCP/UDP load balancer"     → NLB
"firewall/IPS appliance"    → GWLB
"global static IPs"         → Global Accelerator
"DDoS L7 protection"        → WAF + Shield Advanced
"secrets rotation"          → Secrets Manager
"config compliance"         → AWS Config
"API audit"                 → CloudTrail
"threat detection"          → GuardDuty
"open protocol (AMQP/MQTT)" → Amazon MQ
"offline mobile sync"       → AppSync
"managed AD"                → AWS Managed AD
"SSO multi-account"         → IAM Identity Center
"IaC"                       → CloudFormation / CDK
"PaaS"                      → Elastic Beanstalk
"physical data migration"   → Snowball
"hybrid storage"            → Storage Gateway
"managed HPC filesystem"    → FSx for Lustre
```

### Số/Con Số Cần Nhớ

```
IAM:        MFA bắt buộc cho Root | Least Privilege
EC2:        t2.micro = Free Tier | Spot ~90% off | Dedicated Host = BYOL
EBS:        gp3 max 16K IOPS | io2 max 64K | st1/sc1 = no boot
S3:         11 9s durability | Multipart > 5GB required | Presigned max 7 days
Aurora:     6 copies/3 AZs | 15 read replicas | 5x MySQL perf
Lambda:     Max 15 min | 128MB-10GB RAM | 1000 concurrent default
DynamoDB:   Max 400KB item | DAX = microseconds
VPC:        Max /16 CIDR | NAT GW in public subnet | SG stateful / NACL stateless
ASG:        Cooldown 300s default | Target Tracking = best
Route 53:   Alias = free | Geolocation ≠ Latency | Failover = HC required
```

---

> 💪 **Chúc bạn pass AWS SAA-C03!**
> 📌 Tổng hợp từ Notion cá nhân + Udemy Stephane Maarek
> 🗓️ 22/05/2026
