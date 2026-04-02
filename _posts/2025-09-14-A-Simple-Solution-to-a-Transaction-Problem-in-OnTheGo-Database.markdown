---
layout: post
title: "Instead of Complexity: A Simple Solution to a Transaction Problem in OnTheGo Database"
date: 2025-09-14 15:04:00 +0200
categories: OnTheGo Database
---

# Instead of Complexity: A Simple Solution to a Transaction Problem in OnTheGo Database

The **OnTheGo Database** project, which I am personally developing ([GitHub link](https://github.com/nieuwmijnleven/OnTheGoDatabase)), is structurally simpler than commercial database systems. For instance, it does not use advanced techniques like **MVCC (Multi-Version Concurrency Control)** for transaction processing. Instead, I chose a straightforward approach based on **undo logs**.

In this blog post, I will share a specific problem that arose during the **reuse of record blocks during undo processing** and how I solved it in a **simple yet effective way**.

---

## Project Background: Simplicity as a Guiding Principle

OnTheGo Database is designed as a learning and experimentation project. Unlike commercial databases, it does not implement complex isolation levels or versioning; instead, it focuses on a minimalist and understandable structure.

- **Transaction Handling**: Based on undo logs.
- **No MVCC**: No snapshot-based concurrency control.
- **Update Processing**: Internally treated as a combination of `insert` + `delete`.

Thanks to this simple structure, it is easier to quickly identify the root cause of errors and implement a solution.

---

## The Problem: Passing Tests, but Failing in Real Scenarios

The issue emerged after refactoring the **serialization** of `BTreeNode`, an internal node in the `BTreeIndex` structure. While the test code initially showed no problems, this change caused an existing transaction test to fail. This revealed an **unexpected bug in the undo process**.

---

## Undo Processing: Core Logic

The undo processing works as follows:

- The actions `insert`, `update`, and `delete` are logged when a transaction is executed.
- During a **rollback**, these actions are **undone in reverse order**.
- Except when the record size remains identical, an `update` is internally treated as a `delete + insert`. Therefore, the undo logic primarily focuses on `insert` and `delete`.

### Example Scenario:

```text
insert D -> delete D -> insert N -> delete N
```

Rollback (Undo) Order:

```text
undo delete N -> undo insert N -> undo delete D -> undo insert D
```

---

## The Core Issue: Record Blocks Not Returning to Their Original Locations

During the undo action for `insert N` (which performs a delete), the system expects to find **record N at its original location**. Problems arise if:

- The original block where record N resided has been **modified or occupied by another transaction** in the meantime.
- Consequently, during the previous undo of a `delete`, record N might have been **placed in a different block**.
- When the subsequent undo of `delete N` (an insert) occurs, the system tries to remove N from the original block where it no longer exists → **Data Integrity Error**.

This error did not surface in simple tests but **fortunately became visible** during more extensive testing following the B-Tree structure refactoring.

---

## The Solution: Precise Undo with Block ID Mapping

Instead of building a complex block-tracking mechanism, I opted for a **simple and clear approach**:

### Solution in Brief:

1. **During Undo Delete (which is an Insert)**:
   - If the record **cannot be placed in its original block**,
   - Register a mapping of `original block ID → new block ID` in a Map.

2. **During Undo Insert (which is a Delete)**:
   - Consult the Map to determine **where the record is actually located**.
   - Delete the record from the correct block.

### Example:

```java
Map<OldBlockId, NewBlockId> recordPosTracker = new HashMap<>();
recordPosTracker.put(oldBlockId, newBlockId);
```

By using this mapping, the undo action can **refer exactly to the correct block**, maintaining data consistency.

---

## The Power of Simplicity: Solving Complex Problems Without Complexity

Initially, I considered building an extensive mechanism to track block status live or manage additional metadata. However, that would have **undermined the simple design philosophy of this project**.

Instead:
- I preserved the existing logic as much as possible.
- I only implemented tracking for cases where the block location changed.
- I solved the problem without making the code needlessly complicated.

The result: **A secure solution without over-engineering.**

---

## Conclusion: Simplicity is Often the Best Strategy

This experience reaffirmed for me that **"simplicity is powerful."** Advanced features and architectures are valuable, but ultimately, success comes down to **focusing on the problem and finding the simplest possible solution**.

OnTheGo Database is still very much under development. Currently, I am focusing on a **stable I/O architecture and reliable node management**. Performance and structural optimizations will come **later**.

In my view, this is the correct order for building a good system:

> **Correctness → Stability → Performance** > Correct now, even more correct later.

I will continue to share practical problem-solving experiences like this in the future.

Thank you for reading!

---

## Related Code

[GitHub Link - StandardTable.java](https://github.com/nieuwmijnleven/OnTheGoDatabase/blob/18dfda3f64445449dd111b921897dc3117973420/onthego.database/app/src/main/java/onthego/database/core/table/StandardTable.java#L93)

---
