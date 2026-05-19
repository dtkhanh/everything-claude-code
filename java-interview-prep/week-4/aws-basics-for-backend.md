# AWS Basics for Java Backend Developers

> 📖 **Estimated reading time:** 30 minutes  
> 🎯 **Focus:** Services cần biết để phỏng vấn vị trí Backend tại SaaS company  
> ⚡ **Level:** Conceptual understanding + common interview angles — không cần AWS cert

---

## Core Services Map (AdvaHealth context)

```
┌─────────────────────────────────────────────────────────┐
│                    AdvaHealth AWS Architecture          │
│                                                         │
│  Internet → Route53 → CloudFront → ALB                  │
│                                    ↓                    │
│                              ECS (Spring Boot)          │
│                                    ↓                    │
│                    ┌───────────────┼───────────────┐    │
│                   RDS           ElastiCache       S3   │
│               (PostgreSQL)       (Redis)       (DICOM) │
│                                                         │
│  Observability: CloudWatch Logs + Metrics + X-Ray      │
│  Secrets: AWS Secrets Manager / Parameter Store        │
└─────────────────────────────────────────────────────────┘
```

---

## 1. EC2 — Elastic Compute Cloud

```
What it is: Virtual servers in the cloud
When used: Long-running services, custom environments

Key interview concepts:
- Instance types: t3.micro (dev/test), t3.large (production API)
- Auto Scaling Group: automatically add/remove instances based on CPU/memory
- Launch Template: defines instance configuration (AMI, instance type, security groups)

Spring Boot relevance:
- Your JAR runs on EC2 inside a Docker container (typically)
- Health checks via /actuator/health → ALB routes traffic away from unhealthy instances
```

**Interview Q:** *How would you ensure your Spring Boot API has zero downtime during deployment?*  
**📢 Headline:** "I use rolling deployment via AWS ECS or EC2 Auto Scaling with ALB health checks — new instances are registered and receive traffic only after passing `/actuator/health`, then old instances drain connections gracefully."

---

## 2. RDS — Relational Database Service (PostgreSQL)

```
What it is: Managed PostgreSQL (handles backups, patching, failover)
Why it matters: AdvaHealth explicitly lists PostgreSQL as required

Key configurations:
- Multi-AZ: synchronous standby replica for failover (~1 min RTO)
- Read Replicas: async copies for read-heavy workloads
- Parameter Groups: tune postgresql.conf (max_connections, work_mem, etc.)
- Automated Backups: point-in-time recovery up to 35 days

Spring Boot connection:
spring.datasource.url=jdbc:postgresql://${RDS_ENDPOINT}:5432/advahealth
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
```

**HikariCP + RDS best practice:**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10          # Start conservative
      minimum-idle: 2
      connection-timeout: 30000      # 30s
      idle-timeout: 600000           # 10 min
      max-lifetime: 1800000          # 30 min (less than RDS wait_timeout)
      connection-test-query: SELECT 1
```

**Gotcha:** RDS has a `max_connections` limit based on instance size. `db.t3.micro` = ~87 connections. If you have 5 ECS tasks × 10 HikariCP connections = 50 connections — fine. 5 tasks × 50 connections = 250 — you'll hit limits.

---

## 3. S3 — Simple Storage Service

```
What it is: Object storage (files, images, DICOM files, backups)
Healthcare relevance: Store DICOM medical imaging files

Key concepts:
- Buckets: containers for objects
- Presigned URLs: time-limited direct download links (avoid proxying large files)
- Server-side encryption: SSE-S3 or SSE-KMS (required for PHI under HIPAA)
- Bucket policies: IAM-based access control
- Versioning: keep history of files (required for medical records)
```

**Spring Boot S3 integration:**
```java
// AWS SDK v2 with Spring
@Service
public class DicomStorageService {

    private final S3Client s3Client;
    private final String bucketName;

    // Upload DICOM file
    public String uploadDicom(String patientId, byte[] dicomData) {
        String key = "dicom/" + patientId + "/" + UUID.randomUUID() + ".dcm";
        
        s3Client.putObject(
            PutObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .serverSideEncryption(ServerSideEncryption.AWS_KMS) // ✅ encrypt PHI
                .build(),
            RequestBody.fromBytes(dicomData)
        );
        return key;
    }

    // Generate presigned URL (don't proxy large DICOM files through your API)
    public String getPresignedUrl(String key) {
        S3Presigner presigner = S3Presigner.create();
        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
            .signatureDuration(Duration.ofMinutes(15)) // short-lived
            .getObjectRequest(r -> r.bucket(bucketName).key(key))
            .build();
        return presigner.presignGetObject(presignRequest).url().toString();
    }
}
```

---

## 4. ECS — Elastic Container Service

```
What it is: Docker container orchestration (simpler than Kubernetes)
Why relevant: Most startups deploy Spring Boot as Docker containers on ECS

Components:
- Task Definition: Docker image + CPU/memory + env vars + port mappings
- Service: maintains N running tasks, handles rolling deployments
- Cluster: group of compute resources (EC2 or Fargate)
- Fargate: serverless containers — no EC2 management needed

Spring Boot Dockerfile:
FROM openjdk:21-jre-slim
COPY target/app.jar app.jar
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Environment variables in ECS Task Definition:**
```json
"environment": [
    {"name": "SPRING_PROFILES_ACTIVE", "value": "prod"}
],
"secrets": [
    {"name": "DB_PASSWORD", "valueFrom": "arn:aws:ssm:us-east-1:123:parameter/prod/db-password"},
    {"name": "JWT_SECRET", "valueFrom": "arn:aws:ssm:us-east-1:123:parameter/prod/jwt-secret"}
]
```
✅ Secrets pulled from SSM Parameter Store — never hardcoded in task definition

---

## 5. CloudWatch — Observability

```
What it is: AWS-native logging, metrics, and alerting

For Spring Boot:
- Logs: stdout/stderr → CloudWatch Logs (via awslogs driver in ECS)
- Metrics: custom metrics via CloudWatch SDK or Micrometer + CloudWatch reporter
- Alarms: alert on error rate, p99 latency, CPU > 80%
```

**Spring Boot → CloudWatch Logs (application.properties):**
```yaml
# With Micrometer (included in Spring Boot Actuator)
management:
  metrics:
    export:
      cloudwatch:
        namespace: AdvaHealth/Production
        step: 1m
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
```

**Key metrics to track for Healthcare API:**
- `http.server.requests` — request latency by endpoint
- `hikaricp.connections.active` — DB connection pool pressure
- `jvm.memory.used` — heap usage
- Custom: `patient.records.accessed` — audit metric

---

## 6. IAM — Identity and Access Management

```
What it is: AWS permissions system
Key concept: Principle of Least Privilege

For Spring Boot on ECS:
- IAM Task Role: the permissions your application code has
  (read S3, write CloudWatch, access RDS)
- IAM Execution Role: permissions ECS needs to start your container
  (pull ECR image, read SSM secrets)

Never do: hardcode AWS credentials in code
Always do: use IAM roles → SDK auto-discovers credentials
```

```java
// ✅ SDK auto-discovers credentials from IAM role — no keys needed
S3Client s3 = S3Client.builder()
    .region(Region.US_EAST_1)
    .credentialsProvider(DefaultCredentialsProvider.create()) // uses IAM role
    .build();

// ❌ Never do this
S3Client s3 = S3Client.builder()
    .credentialsProvider(StaticCredentialsProvider.create(
        AwsBasicCredentials.create("AKIAIOSFODNN7EXAMPLE", "secret")))
    .build();
```

---

## 7. Quick Architecture Q&A

**Q: "How would you architect a patient data ingestion API that handles 10k req/min?"**

```
Answer structure:
1. Load Balancer (ALB) → multiple ECS tasks (horizontal scaling)
2. Auto Scaling: scale out when CPU > 70% or req/target > 1000
3. Async processing: don't process inline — publish to SQS/Kafka queue
4. Database: RDS PostgreSQL with Read Replica for reporting queries
5. Caching: ElastiCache Redis for frequently-accessed reference data
6. Rate limiting: Bucket4j at application level + AWS WAF at edge
7. Monitoring: CloudWatch alarms on latency p99 > 500ms → PagerDuty alert

Key trade-offs to mention:
- Async = better throughput but adds eventual consistency complexity
- Read replicas = better read performance but replication lag
- Caching = faster but need invalidation strategy
```

**Q: "How do you handle database migrations in a zero-downtime deployment?"**

```
Answer:
1. Use Flyway or Liquibase for version-controlled migrations
2. Backwards-compatible schema changes ONLY:
   - Add columns as nullable first (not NOT NULL with no default)
   - Never rename columns — add new column, migrate data, drop old
   - Never drop columns while old code still references them
3. Migration order: DB migration runs before new code deploys
4. Rollback plan: every migration has a corresponding undo script
```

---

## Interview Cheat Sheet — AWS in 60 Seconds

| Service | 1-line description | Why you care |
|---------|--------------------|--------------|
| EC2 | Virtual servers | Run your Spring Boot JAR |
| ECS/Fargate | Managed containers | Deploy Docker images |
| RDS | Managed PostgreSQL | Your production database |
| ElastiCache | Managed Redis/Memcached | API response caching |
| S3 | Object storage | DICOM files, assets |
| CloudWatch | Logs + Metrics + Alarms | Observability |
| IAM | Permissions | Secure access to AWS services |
| ALB | HTTP/HTTPS load balancer | Route traffic, health checks |
| SQS | Message queue | Async job processing |
| Secrets Manager | Secret storage | DB passwords, API keys |

---

## Gotchas

- ❌ `t3.micro` RDS in production — max ~87 connections, will break under load
- ❌ Exposing RDS to public internet — always use VPC private subnet
- ❌ `s3:*` IAM permission — scope to specific bucket and operations
- ❌ Not setting S3 encryption for PHI — HIPAA violation
- ❌ CloudWatch Logs without log retention policy — costs grow indefinitely
- ✅ ECS health check grace period — give Spring Boot time to start before health checks begin (60–120s)
