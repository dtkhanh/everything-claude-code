# AdvaHealth + AWS SAA (Udemy) Integrated Study Plan

> Date: May 22, 2026
> Input sources:
> - `java-interview-prep/ADVAHEALTH-INTERVIEW-STRATEGY.md`
> - `java-interview-prep/GAP-COVERAGE-ASSESSMENT.md`
> - Your AWS SAA Udemy learning track (Notion notes)

## Goal

Use one plan for two outcomes:
1. Pass AdvaHealth Backend interview (Java/Spring/PostgreSQL/Security/System Design)
2. Build enough AWS depth from your Udemy SAA track to answer cloud questions confidently

## Time Budget (Most Effective)

- Weekdays: 70 minutes/day
- Weekend: 90 minutes/day (1 day), 30 minutes/day (1 day light review)
- Total: 7.5-8.5 hours/week

Daily split (weekday):
- 40 min: AdvaHealth core topic (Java/Spring/DB/Security)
- 25 min: AWS SAA Udemy module + notes
- 5 min: recall drill (no notes)

If too busy, fallback mode:
- 45 min/day minimum
- 30 min core + 10 min AWS + 5 min recall

## 6-Week Integrated Roadmap

### Week 1 - Java 21 + AWS Foundations
- Backend focus:
  - Records, Sealed classes, Pattern matching, Virtual threads
  - Source: `week-2/06-java9-21-features.md`
- AWS focus (Udemy):
  - IAM basics, shared responsibility model, regions/AZs
- Output by end of week:
  - Explain Virtual Threads vs platform threads in under 90s
  - Explain IAM user/group/role/policy in under 90s

### Week 2 - Spring Boot Internals + Networking Core
- Backend focus:
  - Auto-config, bean lifecycle, AOP, Security filter chain, JWT
  - Source: `week-3/07-spring-boot-core.md`, `week-3/08-spring-security.md`
- AWS focus (Udemy):
  - VPC, subnet, route table, security group vs NACL
- Output by end of week:
  - Trace one authenticated request through Spring Security chain
  - Draw private/public subnet architecture and explain traffic flow

### Week 3 - PostgreSQL Optimization + Compute/Storage
- Backend focus:
  - N+1 fixes, index strategy, EXPLAIN ANALYZE, window functions
  - Source: `week-3/09-spring-data-jpa.md`, `week-4/13-postgresql-query-optimization.md`
- AWS focus (Udemy):
  - EC2, EBS, ASG/ELB basics, S3 classes/lifecycle
- Output by end of week:
  - Diagnose slow query with EXPLAIN ANALYZE and propose index
  - Explain when to use EC2 vs ASG + ELB and key S3 classes

### Week 4 - Secure Coding + Data Layer on AWS
- Backend focus:
  - OWASP Top 10, input validation, PHI-safe logging, rate limiting
  - Source: `week-4/security-secure-coding.md`
- AWS focus (Udemy):
  - RDS/Aurora fundamentals, backup, Multi-AZ, read replicas, parameter groups
- Output by end of week:
  - Explain secure API baseline for healthcare data (JWT + validation + logging policy)
  - Explain Multi-AZ vs Read Replica clearly with a scenario

### Week 5 - System Design + Observability/Integration
- Backend focus:
  - API gateway, caching, resilience, idempotency
  - Source: `week-4/11-system-design-microservices.md`
- AWS focus (Udemy):
  - CloudWatch, SQS/SNS/EventBridge basics, ECS overview
- Output by end of week:
  - Present design for patient ingestion API at 10k req/min
  - Choose correct messaging service for 2-3 practical scenarios

### Week 6 - Mock Interview + Consolidation
- Backend focus:
  - Mock interview drills from `week-4/12-interview-qna-master.md`
- AWS focus (Udemy):
  - Review wrong answers and weak services from previous weeks
- Output by end of week:
  - 2 full mock rounds (backend + cloud mix)
  - One 1-page cheatsheet for final revision

## Notion Sync Template (Paste into your Notion)

## Day Plan
- Date:
- Backend topic (40m):
- AWS topic (25m):
- 3 recall questions:
  1.
  2.
  3.
- Today weak point:
- Fix action tomorrow:

## Weekly Review
- Strongest topic:
- Weakest topic:
- 3 interview answers to improve:
- One architecture sketch completed: Yes/No
- Next week priority:

## Interview-Oriented Rule (Very Important)

- Learn to explain every topic in this order:
  1. What it is
  2. Why it matters in production
  3. Trade-off
  4. A concrete bug/performance/security case

This format is exactly what interviewers reward.

## Priority Mapping (Backend JD -> AWS SAA)

- Performance optimization -> EC2 sizing, connection pooling, RDS sizing, caching
- Security -> IAM least privilege, security groups, encrypted storage, secrets handling
- Scalability -> ASG, load balancing, stateless service design, queue decoupling
- Reliability -> Multi-AZ, backups, health checks, retries/circuit breaker
- Observability -> CloudWatch logs/metrics/alarms + app-level tracing mindset

## If You Only Have 45 Minutes/Day

- Mon/Wed/Fri: 35m backend + 10m AWS
- Tue/Thu/Sat: 25m backend + 20m AWS
- Sun: 20m mock verbal answers + 20m weak-point patch

Consistency beats intensity. Do not skip recall practice.

## Next Action (Today)

1. Open `java-interview-prep/ADVAHEALTH-INTERVIEW-STRATEGY.md`
2. Start Week 1 day plan (40m backend + 25m AWS)
3. Fill Notion Day Plan with 3 recall questions before ending session

