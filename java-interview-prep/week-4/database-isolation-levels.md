# Week 4 — Bonus: Database Isolation Levels & Transactions

> 📖 **Estimated reading time:** 35 minutes  
> 🎯 **Focus:** Isolation levels, concurrency problems, PostgreSQL specifics, Spring integration  
> ⚠️ **Why it matters:** Healthcare = critical financial and clinical data. Wrong isolation = silent data corruption. This is a "senior signal" question.

---

## 1. Concurrency Problems in Database Transactions

Before isolation levels, understand what they protect against:

```
Timeline showing 3 classic problems:

DIRTY READ — T2 reads T1's uncommitted change
─────────────────────────────────────────────
T1: UPDATE patients SET status = 'CRITICAL'  ──┐
T2:          SELECT status FROM patients  ──────┤── reads 'CRITICAL' (T1 not committed yet!)
T1: ROLLBACK ─────────────────────────────────┘
T2: acting on data that never officially existed → DANGER in healthcare

NON-REPEATABLE READ — same row, different values in same TX
──────────────────────────────────────────────────────────
T1: SELECT medication_dose FROM prescriptions WHERE id=1  → returns 50mg
T2: UPDATE prescriptions SET medication_dose = 100mg WHERE id=1; COMMIT
T1: SELECT medication_dose FROM prescriptions WHERE id=1  → returns 100mg ← different!
Same query, same transaction, different result = non-repeatable read

PHANTOM READ — same query, different rows in same TX
────────────────────────────────────────────────────
T1: SELECT COUNT(*) FROM active_patients WHERE ward='ICU'  → 10
T2: INSERT INTO active_patients (ward) VALUES ('ICU'); COMMIT
T1: SELECT COUNT(*) FROM active_patients WHERE ward='ICU'  → 11 ← phantom!
T1 sees a "phantom" row it didn't see before
```

---

## 2. Isolation Levels — The Four Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-------|-----------|---------------------|--------------|-------------|
| **READ_UNCOMMITTED** | ✅ possible | ✅ possible | ✅ possible | Fastest |
| **READ_COMMITTED** | ❌ prevented | ✅ possible | ✅ possible | Fast |
| **REPEATABLE_READ** | ❌ prevented | ❌ prevented | ✅ possible | Medium |
| **SERIALIZABLE** | ❌ prevented | ❌ prevented | ❌ prevented | Slowest |

**PostgreSQL defaults:** `READ_COMMITTED` for regular transactions, `REPEATABLE_READ` for explicit transactions.

PostgreSQL actually implements `REPEATABLE_READ` using MVCC (Multi-Version Concurrency Control), so it also prevents phantoms in practice — but SERIALIZABLE is still the only 100% guarantee.

---

## 3. Spring @Transactional Isolation

```java
@Service
public class ClinicalDataService {

    // READ_COMMITTED — default, good for most use cases
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public PatientSummary getPatientSummary(Long patientId) {
        return patientRepository.findSummaryById(patientId);
    }

    // REPEATABLE_READ — use when you read the same data twice and need consistency
    // Example: read medication dose, apply formula, write back
    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public void adjustMedicationDose(Long prescriptionId, double factor) {
        Prescription p = prescriptionRepository.findById(prescriptionId).orElseThrow();
        double currentDose = p.getDoseMilligrams(); // first read
        
        // Calculate new dose
        double newDose = currentDose * factor;
        
        // Between these two lines, another transaction CANNOT change the dose
        // (with REPEATABLE_READ — the row is locked for this transaction)
        p.setDoseMilligrams(newDose);
        prescriptionRepository.save(p); // second use of same data
    }

    // SERIALIZABLE — financial/audit operations, critical sections
    // All operations in this TX appear to have run serially
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void processInsuranceClaim(Long claimId) {
        // Complex financial logic where any concurrent modification is unacceptable
        InsuranceClaim claim = claimRepository.findByIdForUpdate(claimId).orElseThrow();
        // Process...
    }

    // READ_UNCOMMITTED — almost never use in production
    // Healthcare context: NEVER — dirty reads of patient data = harmful decision-making
    @Transactional(isolation = Isolation.READ_UNCOMMITTED)
    public long getDirtyCount() {
        return patientRepository.count(); // slightly faster approximate count
    }
}
```

---

## 4. PostgreSQL MVCC — Why It Performs Well

```
MVCC (Multi-Version Concurrency Control):
- Instead of locking rows for reads, PostgreSQL creates multiple versions
- Writers create new row versions; readers see old versions
- No read/write blocking: readers never block writers, writers never block readers
- Each transaction sees a snapshot of the DB at its start time

Practical effect for Spring Boot:
- @Transactional(readOnly = true) → extremely fast, no lock overhead
- Long-running reads don't block short writes
- PostgreSQL handles the "which version did this transaction start?" logic via xmin/xmax
```

---

## 5. Detecting Isolation Issues in Practice

```java
// Problem: lost update — two transactions read-then-write the same row
@Transactional
public void addMedication(Long patientId, String medication) {
    Patient patient = patientRepository.findById(patientId).orElseThrow();
    List<String> meds = patient.getMedications(); // read
    meds.add(medication);
    patient.setMedications(meds);
    patientRepository.save(patient); // write
}

// If T1 and T2 both read {aspirin}, then T1 adds {ibuprofen}, T2 adds {lisinopril}:
// T1 saves: {aspirin, ibuprofen}
// T2 saves: {aspirin, lisinopril}  ← T1's update LOST!

// ✅ Solution 1: Optimistic locking with @Version
@Entity
public class Patient {
    @Version
    private Long version;
    // T2 will get OptimisticLockException — catch and retry
}

// ✅ Solution 2: Pessimistic lock (for high-contention critical data)
@Transactional(isolation = Isolation.SERIALIZABLE)
public void addMedication(Long patientId, String medication) {
    Patient patient = patientRepository.findByIdForUpdate(patientId).orElseThrow();
    patient.getMedications().add(medication);
    patientRepository.save(patient);
}

// ✅ Solution 3: Use PostgreSQL array/JSONB operations directly
@Query(value = "UPDATE patients SET medications = medications || :med WHERE id = :id",
       nativeQuery = true)
@Modifying
void appendMedication(@Param("id") Long id, @Param("med") String medication);
// PostgreSQL atomic array append — no lost update possible
```

---

## 6. Transaction Timeout — Healthcare Critical

```java
// Healthcare system: never allow long-running transactions to hold locks indefinitely
@Transactional(timeout = 5)  // 5 seconds — throw TransactionTimedOutException after
public DicomProcessingResult processDicomStudy(Long studyId) {
    // If DICOM processing takes > 5s, transaction is rolled back
    // Prevents DB lockup from slow external calls inside transactions
}

// CORRECT pattern: keep transactions SHORT, do long work OUTSIDE
@Transactional(readOnly = true)
public DicomStudy fetchStudy(Long studyId) {
    return studyRepository.findById(studyId).orElseThrow(); // fast DB read
}

public DicomProcessingResult processDicomStudy(Long studyId) {
    DicomStudy study = fetchStudy(studyId);           // TX 1: short read
    DicomProcessingResult result = dicomProcessor.process(study); // slow — NO transaction
    saveResult(studyId, result);                      // TX 2: short write
    return result;
}
```

---

## 7. Deadlocks — Detection & Prevention

```java
// Deadlock scenario in healthcare app:
// Drug A interaction check locks patient record then medication record
// Drug B interaction check locks medication record then patient record
// → circular wait → deadlock

// ❌ Problem
@Transactional
public void checkDrugAInteraction(Long patientId, Long medicationId) {
    Patient patient = patientRepository.findByIdForUpdate(patientId);  // lock 1
    Medication med = medicationRepository.findByIdForUpdate(medicationId); // lock 2
}

@Transactional
public void checkDrugBInteraction(Long medicationId, Long patientId) {
    Medication med = medicationRepository.findByIdForUpdate(medicationId); // lock 1 (reversed!)
    Patient patient = patientRepository.findByIdForUpdate(patientId);  // lock 2
}

// ✅ Solution: always acquire locks in the SAME order
@Transactional
public void checkInteraction(Long patientId, Long medicationId) {
    // Always lock in same order: patient first, then medication
    Patient patient = patientRepository.findByIdForUpdate(patientId);
    Medication med = medicationRepository.findByIdForUpdate(medicationId);
    // Consistent ordering prevents circular wait
}

// PostgreSQL deadlock detection:
// pg_stat_activity view shows waiting queries
// ERROR: deadlock detected — PostgreSQL aborts one transaction, you must retry
// Spring: catch DeadlockLoserDataAccessException → retry logic
```

---

## 8. PostgreSQL-Specific Features

### Optimistic Locking with PostgreSQL

```sql
-- Explicit version check without @Version annotation
-- Useful for native queries on critical tables
UPDATE prescriptions
SET dose_mg = ?,
    version = version + 1
WHERE id = ? AND version = ?  -- will update 0 rows if version changed
RETURNING id;
-- In Java: check returned count — 0 rows = lost update → throw exception
```

### SELECT FOR UPDATE SKIP LOCKED — Queue Pattern

```java
// Healthcare task queue: claim tasks without waiting for locked rows
@Query(value = """
    SELECT * FROM pending_lab_results
    WHERE status = 'UNPROCESSED'
    ORDER BY received_at
    LIMIT 10
    FOR UPDATE SKIP LOCKED
    """, nativeQuery = true)
@Lock(LockModeType.PESSIMISTIC_WRITE)
List<LabResult> claimLabResultsForProcessing();

// Each worker thread claims a batch without blocking others
// Perfect for parallel lab result processing pipelines
```

### Advisory Locks — Application-Level Locking

```java
// PostgreSQL advisory locks: named locks that don't block actual rows
// Use for: preventing duplicate scheduled job runs, rate limiting

@Transactional
public void runScheduledSync() {
    // Try to acquire lock — returns false if another instance holds it
    Boolean acquired = jdbcTemplate.queryForObject(
        "SELECT pg_try_advisory_xact_lock(?)", Boolean.class, SYNC_JOB_LOCK_KEY
    );
    
    if (!acquired) {
        log.info("Sync already running on another instance, skipping");
        return;
    }
    
    // Only one instance runs this — lock released when TX ends
    performSync();
}
```

---

## 9. Quick Reference — When to Use What

| Scenario | Solution | Why |
|----------|----------|-----|
| Standard CRUD operations | `READ_COMMITTED` (default) | Good balance, no dirty reads |
| Read then recalculate and write same row | `REPEATABLE_READ` + `@Version` | Prevent non-repeatable reads, optimistic retry |
| Financial transaction / critical update | `SERIALIZABLE` or `PESSIMISTIC_WRITE` | Absolute consistency |
| Parallel task processing without contention | `SELECT FOR UPDATE SKIP LOCKED` | High throughput, no blocking |
| Report/dashboard queries | `readOnly = true` | PostgreSQL optimizes, no lock overhead |
| Audit log (must commit even if parent TX rolls back) | `REQUIRES_NEW` | Independent transaction |
| Distributed multi-service operation | Saga pattern | No distributed ACID, use compensating TXs |

---

## Interview Q&A

**Q: "What are database isolation levels and why do they matter?"**

📢 **Headline:** "Isolation levels define what concurrent transactions can see. The four levels trade consistency for performance: READ_UNCOMMITTED allows dirty reads (never use), READ_COMMITTED prevents dirty reads (PostgreSQL default), REPEATABLE_READ prevents non-repeatable reads, and SERIALIZABLE prevents phantom reads. In healthcare, I default to READ_COMMITTED but upgrade to SERIALIZABLE or use pessimistic locking for medication dosing calculations — a phantom read could mean a dangerous drug interaction is missed."

**Q: "What is a deadlock and how do you handle it in Spring Boot?"**

📢 **Headline:** "A deadlock is when two transactions each hold a lock the other needs, creating a circular wait. PostgreSQL detects this and aborts one transaction, throwing a `DeadlockLoserDataAccessException`. Prevention: always acquire locks in the same consistent order across all code paths. Handling: catch `DeadlockLoserDataAccessException` and retry with exponential backoff, or use `@Retryable`."

**Q: "What is MVCC and why does PostgreSQL not need read locks?"**

📢 **Headline:** "MVCC (Multi-Version Concurrency Control) means PostgreSQL keeps multiple row versions. Each transaction sees a snapshot of the database as it was at transaction start. Readers see old versions; writers create new ones. This means reads never block writes and writes never block reads — no read locks needed. The trade-off: old versions accumulate until `VACUUM` cleans them up."
