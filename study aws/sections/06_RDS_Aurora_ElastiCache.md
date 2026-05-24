# 📚 AWS SAA — Phần 6: RDS, Aurora & ElastiCache

> Databases as a Service (DBaaS) trên AWS

---

## 🗄️ RDS — Relational Database Service

> RDS = Database SQL được AWS quản lý hoàn toàn

### DB Engines Được Hỗ Trợ

| Engine | Ghi chú |
|--------|---------|
| PostgreSQL | Open source, phổ biến |
| MySQL | Open source, phổ biến nhất |
| MariaDB | Fork của MySQL |
| Oracle | Enterprise, license riêng |
| Microsoft SQL Server | Windows ecosystem |
| IBM Db2 | Enterprise |
| **Aurora** | AWS-native, hiệu năng cao nhất |

### Ưu Điểm RDS vs EC2 tự cài DB

| RDS (Managed) | EC2 + DB (Self-managed) |
|--------------|------------------------|
| ✅ Auto backup | ❌ Tự backup |
| ✅ Point-in-time restore | ❌ Phức tạp |
| ✅ Multi-AZ tự động | ❌ Tự setup |
| ✅ Read Replicas | ❌ Tự cấu hình |
| ✅ Monitoring tích hợp | ❌ Tự monitor |
| ✅ Storage auto-scaling | ❌ Tự scale |
| ❌ Không SSH vào DB | ✅ Có thể SSH |

### 🔄 RDS Storage Auto Scaling

- Tự động tăng storage khi gần đầy
- Cần set **Maximum Storage Threshold**
- Scales khi: free storage < 10%, kéo dài > 5 phút, qua 6 giờ kể từ lần scale trước
- Hỗ trợ tất cả DB engines

### 📖 Read Replicas

```
Primary RDS ──replication──→ Read Replica 1 (us-east-1a)
                          └──replication──→ Read Replica 2 (us-east-1b)
                          └──replication──→ Read Replica 3 (us-west-2) ← Cross Region

App viết → Primary
App đọc  → Read Replicas (phân tải read traffic)
```

**Đặc điểm Read Replicas:**
- Tối đa **15 Read Replicas** (Aurora), **5** cho RDS khác
- Trong AZ, Cross-AZ, hoặc Cross-Region
- Replication **ASYNC** → reads eventually consistent
- Replicas có thể promoted thành standalone DB
- **Network cost:** Trong cùng Region = **FREE**; Cross-Region = **tốn tiền**

### 🏢 Multi-AZ (Disaster Recovery)

```
Primary DB (us-east-1a)
      │
      │ SYNC replication
      ↓
Standby DB (us-east-1b) ← Không serve traffic!
                          Chỉ standby để failover

Failover tự động:
- Primary fail → Standby tự động promoted
- DNS flips → Applications reconnect
- Downtime: ~60-120 giây
```

**Multi-AZ vs Read Replicas:**

| | Multi-AZ | Read Replica |
|--|---------|-------------|
| Mục đích | **Availability** (HA) | **Performance** (scale reads) |
| Replication | SYNC | ASYNC |
| Serve traffic | ❌ Standby không serve | ✅ Serve read traffic |
| Failover | ✅ Tự động | ❌ Manual promote |

> 💡 **Tip:** Có thể enable Read Replica với Multi-AZ để vừa scale vừa HA

### 🔄 RDS Từ Single-AZ → Multi-AZ (Zero Downtime!)

1. Click "Modify" → Enable Multi-AZ
2. RDS tạo snapshot của Primary
3. Restore snapshot vào Standby ở AZ mới
4. Sync được thiết lập giữa Primary và Standby
5. **Không có downtime** trong quá trình này

### 🔵 RDS Custom

- Dành cho **Oracle và Microsoft SQL Server**
- Cho phép access vào OS và DB (SSH, SSM Session Manager)
- Customize cấu hình, install patches, enable native features
- **Deactivate Automation Mode** trước khi customize

---

## 🌟 Amazon Aurora

> Aurora = AWS-native database engine, tương thích PostgreSQL và MySQL

### Aurora vs RDS

| | Aurora | RDS |
|--|--------|-----|
| Performance | **5x MySQL, 3x PostgreSQL** | Standard |
| Storage | Tự động grow (10GB → 128TB) | Phải provision |
| Replicas | Tới **15** read replicas | Tới 5 |
| Failover | **Instantaneous** | ~60-120s |
| Cost | ~20% đắt hơn RDS | Base |
| HA | Built-in, mặc định | Cần cấu hình |

### Aurora Storage Architecture (Đặc Biệt!)

```
Aurora Cluster
├── Writer Endpoint (Primary) ── write traffic
└── Reader Endpoint (Load balanced) ── read traffic

Storage: 6 copies across 3 AZs
├── AZ-1: 2 copies
├── AZ-2: 2 copies  
└── AZ-3: 2 copies

Write: cần 4/6 copies available
Read:  cần 3/6 copies available
→ Highly fault tolerant!
```

### Aurora Features

| Feature | Mô tả |
|---------|-------|
| **Auto Scaling** | Tự động thêm/bớt Read Replicas |
| **Custom Endpoints** | Gửi traffic đến subset replicas |
| **Aurora Serverless** | Auto-scale capacity, trả tiền theo giây |
| **Aurora Global** | 1 primary Region, tới 5 secondary Regions |
| **Aurora ML** | Tích hợp SageMaker và Comprehend |
| **Aurora Multi-Master** | Multiple write nodes (active-active) |

### Aurora Serverless

```
Thay vì chạy DB 24/7:
- Auto scale lên khi có queries
- Scale xuống 0 khi không có traffic
- Trả tiền theo ACU (Aurora Capacity Units) per second
- Phù hợp: dev/test, irregular workloads
```

### Aurora Global Database

```
Primary Region (us-east-1): Writer
    │
    │ Replication < 1 second
    ↓
Secondary Region (eu-west-1): 16 Read Replicas
Secondary Region (ap-southeast-1): 16 Read Replicas

Recovery Time Objective (RTO) < 1 minute
→ Promote Secondary Region nếu Primary fail
```

---

## ⚡ ElastiCache — In-Memory Database

> ElastiCache = Cache layer tốc độ cao, giúp giảm tải Database

### Redis vs Memcached

| | Redis | Memcached |
|--|-------|-----------|
| **Data structures** | String, Hash, List, Set, Sorted Set, Stream | Chỉ String |
| **Persistence** | ✅ Yes (AOF, RDB) | ❌ No |
| **Multi-AZ** | ✅ Yes | ❌ No |
| **Read Replicas** | ✅ Yes | ❌ No |
| **Pub/Sub** | ✅ Yes | ❌ No |
| **Backup/Restore** | ✅ Yes | ❌ No |
| **Use Case** | Session store, Leaderboards, Pub/Sub | Simple caching, Multi-threaded |

> 🎯 **Tip thi:** Hầu hết câu hỏi đều hỏi **Redis** vì tính năng phong phú hơn. Chọn Memcached chỉ khi cần simple caching + multi-threaded.

### Cache Patterns

**Lazy Loading (Cache-Aside):**
```
App → Cache hit? → Yes → Return data
                → No  → Query DB → Store in Cache → Return data

✅ Cache chỉ chứa data được request
❌ Cache miss = 3 network calls (penalty)
❌ Data có thể stale
```

**Write Through:**
```
App → WRITE → DB + Cache cùng lúc

✅ Cache luôn fresh
❌ Write penalty (2 calls)
❌ Cache có data chưa bao giờ được đọc
```

### ElastiCache Sessions

```
User login → App server → Store session in ElastiCache
User next request (different server) → App server → Get session from ElastiCache
→ Stateless: any app server can handle any request
```

### Redis Sorted Sets (thi hay hỏi)

```
Use case: Leaderboard / Gaming rankings

ZADD leaderboard 100 "player1"
ZADD leaderboard 150 "player2"
ZADD leaderboard 80 "player3"

ZRANGE leaderboard 0 -1 WITHSCORES → ranked list
→ Guaranteed uniqueness + ordering
```

---

## ❓ Câu Hỏi Ôn Tập Databases

1. RDS Read Replica dùng cho mục đích gì? Multi-AZ dùng cho mục đích gì?
2. Aurora tự động lưu bao nhiêu copies? Ở bao nhiêu AZ?
3. Khi nào dùng ElastiCache Redis vs Memcached?
4. RDS Read Replica replication là SYNC hay ASYNC?
5. Aurora Serverless phù hợp với workload như thế nào?
6. Network cost cho Read Replica Cross-Region có free không?

<details>
<summary>📌 Đáp án</summary>

1. Read Replica = scale đọc (performance); Multi-AZ = disaster recovery (availability)
2. **6 copies** across **3 AZs**
3. Redis khi cần persistence, data structures, HA; Memcached khi chỉ cần simple cache + multi-threaded
4. **ASYNC** — eventually consistent
5. Variable/unpredictable workloads, dev/test, sporadic usage
6. **Không free** — Cross-Region replication tốn network cost

</details>

