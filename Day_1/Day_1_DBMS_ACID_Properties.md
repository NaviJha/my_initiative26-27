# Day 1: DBMS — ACID Properties & Transaction Processing

## 📌 Overview & Core Concepts

In a Database Management System (DBMS), a **transaction** is a sequence of operations executed as a single logical unit of work. A transaction begins with a set of reading or writing operations and finishes with either a **Commit** (permanently saving changes) or a **Rollback** (undoing all changes).

To maintain data integrity, accuracy, and reliability—especially during hardware failures, system crashes, power outages, or concurrent multi-user execution—databases enforce the **ACID** properties.

```text
              +-----------------------------------+
              |      DATABASE TRANSACTION         |
              +-----------------------------------+
                                |
      +-------------------------+-------------------------+
      |                         |                         |
      v                         v                         v
[ Begin Trans ]          [ Read/Write Ops ]         [ Commit / Abort ]
      |                         |                         |
      +-------------------------+-------------------------+
                                |
                                v
                 +-------------------------------+
                 |  ACID GUARANTEES ENFORCED     |
                 |                               |
                 |  Atomicity   |  Consistency   |
                 |  Isolation   |  Durability    |
                 +-------------------------------+
```

## 🔄 Step-by-Step Transaction Walkthrough

Consider a banking transaction where Account A (initial balance: $2,000) transfers $1,000 to Account B (initial balance: $3,000).

### Execution Flow

```text
 ACCOUNT A ($2,000)                         ACCOUNT B ($3,000)
+--------------------+                     +--------------------+
| 1. Read(A)         |                     | 4. Read(B)         |
| 2. A = A - 1000    |                     | 5. B = B + 1000    |
| 3. Write(A)        |                     | 6. Write(B)        |
+---------+----------+                     +---------+----------+
          |                                          |
          v                                          v
  [ Balance = $1,000 ]                       [ Balance = $4,000 ]
          |                                          |
          +----------------------+-------------------+
                                 |
                                 v
                         [ COMMIT APPLIED ]
```

### Detailed Execution Steps

| Step | Operation | Action | Account A | Account B |
|---:|---|---|---:|---:|
| 0 | Initial state | Database at rest | $2,000 | $3,000 |
| 1 | `Read(A)` | Fetch A's balance into memory | $2,000 | $3,000 |
| 2 | `A = A - 1000` | Compute deduction in memory | $1,000 (RAM) | $3,000 |
| 3 | `Write(A)` | Update A's value in a memory buffer | $1,000 (buffer) | $3,000 |
| 4 | `Read(B)` | Fetch B's balance into memory | $1,000 | $3,000 |
| 5 | `B = B + 1000` | Compute addition in memory | $1,000 | $4,000 (RAM) |
| 6 | `Write(B)` | Update B's value in a memory buffer | $1,000 | $4,000 (buffer) |
| 7 | `Commit` | Commit the transaction; make its effects recoverable and durable according to the DBMS | $1,000 | $4,000 |

> **Important:** A commit does not necessarily mean every modified table page is immediately written to its final disk location. With write-ahead logging, the commit can be durable once the required log records have been safely flushed.

## 🏛️ The 4 Pillars of ACID

### 1. Atomicity — “All or Nothing”

**Definition:** A transaction must either execute completely or have no effect. If a failure occurs before commit, the DBMS uses recovery mechanisms to undo incomplete work.

**Key idea:** A transaction is not allowed to leave only some of its intended changes in the database.

```text
SUCCESS:
[Step 1] ---> [Step 2] ---> [Step 3] ---> [COMMIT]

FAILURE:
[Step 1] ---> [Step 2] ---> [CRASH]
                  |
                  v
              [ROLLBACK]
                  |
                  v
        [Restore prior logical state]
```

**Real-world examples**

- **ATM cash withdrawal:** If an account is debited but the ATM fails to dispense cash, the bank's recovery and reconciliation process must ensure the customer is not left with an incorrect final outcome. This is not always an instant, automatic rollback: ATM hardware actions and database updates may require coordinated protocols and later reconciliation.
- **Bulk payroll:** If a single transaction updates salary records for 100 employees and fails on record 85, the database can roll back the transaction's earlier updates as well.
- **Online order:** Creating an order and reserving inventory may be grouped into a transaction so a failed database operation does not leave only part of the database update applied.

### 2. Consistency — “Valid State to Valid State”

**Definition:** A transaction must take the database from one valid state to another, preserving defined constraints and business rules.

**Key idea:** The database rules must hold before and after a successful transaction.

```text
BEFORE TRANSACTION                 AFTER TRANSACTION
+-----------------------+          +-----------------------+
| Account A  =  $2,000  |          | Account A  =  $1,000  |
| Account B  =  $3,000  |          | Account B  =  $4,000  |
+-----------------------+          +-----------------------+
| TOTAL SUM  =  $5,000  |          | TOTAL SUM  =  $5,000  |
+-----------------------+          +-----------------------+
```

**Real-world examples**

- **Financial invariant:** A transfer of $1,000 between these two accounts should preserve the combined balance of $5,000.
- **Integrity constraints:** A table may enforce `CHECK (age >= 18)`.
- **Foreign keys:** An order's `user_id` must refer to an existing user when the schema requires that relationship.
- **Inventory:** A business rule may prevent stock from becoming negative, for example with a constraint or a guarded update.

Consistency is not magic error detection: the DBMS can enforce declared constraints, while application-level rules must be correctly designed and implemented.

### 3. Isolation — “Concurrent Transactions Should Not Interfere Incorrectly”

**Definition:** Isolation controls how concurrent transactions interact. It prevents certain anomalies and, at stronger levels, makes concurrent execution behave like a valid serial order.

**Key idea:** Other transactions should not observe invalid intermediate states. The exact guarantees depend on the chosen isolation level and database implementation.

```text
TRANSACTION 1 (Updating)              TRANSACTION 2 (Reading)
+---------------------------+         +---------------------------+
| Read(A) -> $2,000         |         | Read(A)                   |
| A = A - 1,000             |         |                           |
| Write(A) -> $1,000        | ------> | May wait or see a         |
| (uncommitted)             |         | committed version         |
| ... processing ...        |         |                           |
| Commit                    |         |                           |
+---------------------------+         +---------------------------+
```

**Real-world examples**

- **Last-seat reservation:** Two users attempt to book the final seat. Concurrency control helps ensure both transactions cannot successfully reserve the same seat.
- **Dirty read:** Transaction 1 changes a balance but has not committed. If Transaction 2 reads that uncommitted value and Transaction 1 later rolls back, Transaction 2 has read data that never became committed.
- **Lost update:** Two transactions read the same value and then write updates based on that old value. Without suitable concurrency control, one update may overwrite the other.

**Common isolation anomalies**

| Anomaly | Meaning |
|---|---|
| Dirty read | Reading another transaction's uncommitted change |
| Non-repeatable read | Re-reading a row and getting a different committed value |
| Phantom read | Repeating a predicate query and seeing a changed set of rows |
| Lost update | One concurrent write overwrites another update |
| Write skew | Concurrent transactions make individually valid changes that together violate a business rule |

**Common isolation levels (SQL terminology)**

| Level | General guarantee |
|---|---|
| Read Uncommitted | May allow dirty reads; exact behavior is DBMS-dependent |
| Read Committed | Prevents dirty reads |
| Repeatable Read | Prevents dirty and non-repeatable reads; phantom behavior varies by implementation |
| Serializable | Strongest standard level; execution is equivalent to some serial order |

### 4. Durability — “Committed Means Recoverable”

**Definition:** Once a transaction commits successfully, its effects must survive subsequent failures covered by the DBMS's durability guarantees.

**Key idea:** A successful commit must not disappear merely because the server restarts.

```text
[Transaction Commits]
          |
          v
[Required log records safely persisted]
          |
          v
[Power loss / crash]
          |
          v
[System restarts]
          |
          v
[Recovery uses logs to restore committed state]
```

**Real-world examples**

- **Airline booking:** After a booking is confirmed, the committed reservation should remain recorded after a database crash.
- **Payment record:** A successfully committed payment entry should be recoverable after a restart.
- **Write-ahead logging (WAL):** The DBMS records relevant changes in a log before the corresponding data pages are written. During recovery, it can redo committed changes when needed.

Durability is bounded by the system's configuration and failure model. For example, local disk durability alone does not automatically guarantee survival of destruction of the entire data center.

## ⚙️ Internal DBMS Mechanisms for ACID

| ACID property | Commonly involved subsystem | Mechanisms |
|---|---|---|
| Atomicity | Transaction and recovery manager | Undo information, rollback, recovery |
| Consistency | DBMS constraint engine and application logic | Primary keys, foreign keys, `CHECK`, triggers, business-rule validation |
| Isolation | Concurrency-control manager | Locks, two-phase locking (2PL), MVCC, timestamp ordering, serializable scheduling |
| Durability | Recovery and storage subsystems | WAL, redo records, stable storage, checkpoints, replication where configured |

### Undo, Redo, and WAL

- **Undo information** helps reverse changes from transactions that did not commit.
- **Redo information** helps reapply changes that belong to committed transactions but may not yet have reached the database's data files.
- **Write-ahead logging (WAL)** requires relevant log records to be persisted before modified data pages are written. A commit is acknowledged only after the DBMS has satisfied its configured durability requirements.

> Logging designs differ across database systems. Some use combined log records for undo and redo; others use different recovery strategies. Avoid assuming every DBMS uses identical files or algorithms.

## 💻 SQL Example: A Bank Transfer

The following is illustrative SQL. Exact syntax and behavior can vary by database.

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 1000
WHERE account_id = 'A'
  AND balance >= 1000;

UPDATE accounts
SET balance = balance + 1000
WHERE account_id = 'B';

COMMIT;
```

In a real application, check that the first update affected exactly one row and that the second update succeeded. If any required step fails, issue `ROLLBACK` rather than committing a partial transfer. Also consider locking, concurrent updates, account existence, currency, and transaction isolation.

### Transaction control commands

| Command | Purpose |
|---|---|
| `BEGIN` / `START TRANSACTION` | Start a transaction |
| `COMMIT` | Make the transaction's changes final |
| `ROLLBACK` | Undo uncommitted changes |
| `SAVEPOINT name` | Create a point to which part of a transaction can be rolled back |
| `ROLLBACK TO name` | Roll back to a savepoint, where supported |

## 🎯 ACID at a Glance

| Property | Main question | Failure or anomaly it addresses |
|---|---|---|
| Atomicity | Did all intended operations happen, or none? | Partial updates |
| Consistency | Are database rules still satisfied? | Invalid database state |
| Isolation | Did concurrent transactions interact safely? | Dirty reads, lost updates, other concurrency anomalies |
| Durability | Will committed changes survive a covered failure? | Lost committed data after a crash |

**Memory trick:**  
- **A** — All or nothing  
- **C** — Constraints remain satisfied  
- **I** — Independent concurrent execution  
- **D** — Data persists after commit  

## ❓ Technical Interview Questions & Answers

### Q1. Why do we need ACID properties in a DBMS?

**Answer:** ACID helps databases remain reliable when transactions fail or run concurrently.

- Without atomicity, a transfer could debit one account without crediting the other.
- Without consistency, declared rules and invariants could be violated.
- Without isolation, concurrent transactions could interfere and produce anomalies.
- Without durability, committed changes could be lost after a covered system failure.

### Q2. What is the difference between Atomicity and Durability?

**Answer:**

- **Atomicity** concerns whether a transaction's operations take effect as a complete unit.
- **Durability** concerns whether the effects of a committed transaction survive subsequent failures.

### Q3. Can a failed transaction be resumed from the exact point of failure?

**Answer:** Usually, a database transaction that aborts is rolled back rather than continued from the failure point. The application may retry the whole transaction. Some systems and applications can use savepoints or retry selected operations, but that is different from automatically resuming an aborted transaction as if nothing happened.

### Q4. What is Write-Ahead Logging (WAL), and how does it relate to ACID?

**Answer:** WAL is a recovery strategy in which required log records are persisted before the corresponding modified data pages are written. It supports durability by allowing committed work to be recovered, and atomicity by helping undo incomplete work, depending on the DBMS's logging and recovery design.

### Q5. What is the difference between `COMMIT` and `ROLLBACK`?

**Answer:** `COMMIT` successfully finishes a transaction and makes its changes final under the database's guarantees. `ROLLBACK` cancels uncommitted work and restores the transaction's prior logical state.

### Q6. What is a dirty read?

**Answer:** A dirty read occurs when one transaction reads a change made by another transaction before that change has committed. If the writer later rolls back, the reader has used a value that was never committed.

### Q7. What is serializability?

**Answer:** Serializability is a correctness criterion for concurrent execution. A schedule is serializable if its outcome is equivalent to some serial execution of the same transactions.

### Q8. What is the difference between a lock and MVCC?

**Answer:** Locks coordinate access by restricting conflicting operations. Multi-Version Concurrency Control (MVCC) keeps multiple versions of data so readers can often access a consistent snapshot while writers make changes. Systems may combine MVCC with locks.

### Q9. Does consistency mean every transaction preserves the total sum of all account balances?

**Answer:** Only if that invariant is part of the application's rules and the transaction is designed to preserve it. Consistency means preserving all relevant declared and correctly implemented rules—not one universal rule about sums.

### Q10. Is a transaction automatically atomic across two different databases?

**Answer:** No. A local database transaction generally covers work managed by that database. Coordinating multiple databases may require distributed transaction protocols, such as two-phase commit, or application-level patterns such as sagas.

### Q11. What is a checkpoint?

**Answer:** A checkpoint is a recovery mechanism that records a known point in the database's recovery history and can reduce how much log processing is needed after a crash. Its exact implementation varies.

### Q12. Can a database guarantee durability if the storage device lies about successful writes?

**Answer:** No system can provide stronger guarantees than its storage and configuration actually support. Reliable durability depends on correct flush behavior, hardware, operating-system behavior, and any configured replication or backup strategy.

## 📝 Practice Multiple Choice Questions (MCQs)

### MCQ 1

**Question:** A transaction updates 4 out of 5 database records and then encounters a power failure. Upon reboot, the database reverts the 4 updated records to their original values. Which ACID property is being demonstrated?

A) Consistency  
B) Isolation  
C) Atomicity  
D) Durability

**Answer: C — Atomicity.** It enforces the all-or-nothing rule, so incomplete work is undone.

### MCQ 2

**Question:** Which type of log information is commonly used by the recovery manager to reverse partial changes made by an aborted transaction?

A) Undo information  
B) Redo information  
C) Audit log  
D) Error log

**Answer: A — Undo information.** It records what is needed to reverse changes from incomplete transactions.

### MCQ 3

**Question:** A dirty read occurs when a transaction reads uncommitted changes written by a concurrent transaction. Which ACID property is involved?

A) Atomicity  
B) Consistency  
C) Isolation  
D) Durability

**Answer: C — Isolation.** Isolation governs interactions among concurrent transactions.

### MCQ 4

**Question:** Which two ACID properties are commonly supported by WAL-based crash recovery?

A) Consistency and Isolation  
B) Atomicity and Durability  
C) Isolation and Durability  
D) Atomicity and Consistency

**Answer: B — Atomicity and Durability.** Undo and redo recovery can respectively help reverse incomplete work and restore committed work.

### MCQ 5

**Question:** A transaction changes a product's price to a negative value, violating a declared `CHECK (price >= 0)` constraint. Which ACID property is most directly protected by rejecting the update?

A) Atomicity  
B) Consistency  
C) Isolation  
D) Durability

**Answer: B — Consistency.** The constraint helps keep the database in a valid state.

### MCQ 6

**Question:** Two transactions read the same stock quantity. Both sell the same final item, and both write back a reduced quantity. Which concurrency problem may occur?

A) Dirty read  
B) Lost update or overselling race  
C) Durability failure  
D) Checkpointing

**Answer: B — Lost update or overselling race.** The application needs suitable concurrency control and a guarded update.

### MCQ 7

**Question:** Which isolation level is intended to make concurrent execution equivalent to some serial order?

A) Read Uncommitted  
B) Read Committed  
C) Repeatable Read  
D) Serializable

**Answer: D — Serializable.**

### MCQ 8

**Question:** A transaction has committed. Immediately afterward, the server crashes. Which property requires the committed result to be recoverable after restart?

A) Atomicity  
B) Consistency  
C) Isolation  
D) Durability

**Answer: D — Durability.**

### MCQ 9

**Question:** Which command cancels the uncommitted changes of the current transaction?

A) `SAVEPOINT`  
B) `COMMIT`  
C) `ROLLBACK`  
D) `SELECT`

**Answer: C — `ROLLBACK`.**

### MCQ 10

**Question:** Transaction T1 updates a row but has not committed. T2 reads the old committed version using an MVCC snapshot. Which statement is most accurate?

A) T2 necessarily performs a dirty read  
B) MVCC can allow T2 to read a committed version without waiting for T1  
C) Durability has failed  
D) T1 has committed automatically

**Answer: B.** MVCC can provide readers with a consistent committed snapshot.

### MCQ 11

**Question:** What is the primary purpose of a database checkpoint?

A) Encrypt all records  
B) Reduce recovery work after a crash  
C) Prevent every deadlock  
D) Replace transaction logs permanently

**Answer: B.** Checkpoints can shorten recovery by establishing a useful recovery point.

### MCQ 12

**Question:** A transfer transaction debits account A, but the credit to account B fails. What should the application normally do?

A) Commit anyway  
B) Ignore the error  
C) Roll back the transaction  
D) Delete account A

**Answer: C — Roll back.** This prevents the transfer from being left partially applied.

### MCQ 13

**Question:** Which statement about consistency is correct?

A) It means all transactions run one at a time  
B) It means every query returns the same result forever  
C) It means successful transactions preserve the database's defined rules  
D) It means committed data is stored only in RAM

**Answer: C.**

### MCQ 14

**Question:** Which mechanism is commonly associated with concurrency control?

A) Two-phase locking  
B) Image compression  
C) Data visualization  
D) File naming

**Answer: A — Two-phase locking.**

### MCQ 15

**Question:** An application needs to update two independent databases as one coordinated unit. Which concept may be relevant?

A) Two-phase commit  
B) A `CHECK` constraint alone  
C) A local savepoint only  
D) A database index

**Answer: A — Two-phase commit.** It is a distributed transaction coordination protocol.

## 🧠 Quick Revision

- A **transaction** is a logical unit of database work.
- **Commit** completes a transaction; **rollback** cancels its uncommitted changes.
- **Atomicity:** all-or-nothing execution.
- **Consistency:** preserve defined database rules.
- **Isolation:** control interference between concurrent transactions.
- **Durability:** committed changes survive failures covered by the system's guarantees.
- **Undo** helps reverse incomplete work; **redo** helps recover committed work.
- **WAL** persists required log records before modified data pages are written.
- **Locks** and **MVCC** are common concurrency-control techniques.
- **Serializable** is the SQL isolation level intended to provide serial-equivalent execution.

## 📚 Suggested Practice

1. Write a SQL transaction that transfers money between two accounts and handles errors.
2. Create a table with a `CHECK` constraint and test an invalid insert.
3. Explore dirty reads and isolation levels using two database sessions.
4. Compare lock-based concurrency control with MVCC.
5. Explain how a database can recover after a crash using undo and redo information.

---
**Day 1 Complete: DBMS — ACID Properties & Transaction Processing**
