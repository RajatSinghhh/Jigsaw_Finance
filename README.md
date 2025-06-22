# Jigsaw Finance Audit – Findings Summary

This document summarizes critical and informational findings uncovered during the security assessment of the protocol’s smart contract architecture. Each vulnerability has been detailed in individual reports stored in this directory.

---

## 📑 Table of Contents

| ID | Finding Title                                                                             | Severity<sup>†</sup> | Link                                                        |
| -- | ----------------------------------------------------------------------------------------- | -------------------- | ----------------------------------------------------------- |
| 1  | Lack of Minimum Deposit Threshold Enables Dust-Griefing in `HoldingManager.deposit`       | Medium               | [Report 1](./finding-1-dust-griefing-holdingmanager.md)     |
| 2  | Missing Success Verification in `emergencyGenericCall` (Holding.sol)                      | Medium               | [Report 2](./finding-2-missing-success-verification.md)     |
| 3  | Unbounded Loop in `borrowMultiple` Enables DoS via Oversized `_data` (HoldingManager.sol) | High                 | [Report 3](./finding-3-unbounded-loop-dos.md)               |
| 4  | Mutable `mintLimit` Can Permanently Freeze jUSD Minting (JigsawUsd.sol)                   | High                 | [Report 4](./finding-4-mutable-mintlimit-freeze.md)         |
| 5  | Naming Convention Violation in `getFeeAbsolute` Function (OperationsLib.sol)              | Informational        | [Report 5](./finding-5-naming-convention-getfeeabsolute.md) |
| 6  | Lack of Minimum Deposit Enables Dust-Griefing in `deposit` (Staker.sol)                   | Medium               | [Report 6](./finding-6-dust-griefing-staker.md)             |
| 7  | Uninitialized `manager` Pointer Breaks Access-Control & Fee Logic                         | High                 | [Report 7](./finding-7-uninitialized-manager-pointer.md)    |
| 8  | Uninitialized Token Addresses Cause Permanent DoS in Multi-Strategy Contracts             | High                 | [Report 8](./finding-8-uninitialized-token-dos.md)          |

> <sup>†</sup> *Severity levels are provisional. Refer to each report for detailed context.*

---

## 1 Lack of Minimum Deposit Threshold Enables Dust-Griefing in `HoldingManager.deposit`

* **Summary:** Arbitrarily small deposits can be abused to bloat storage and gas usage.
* 👉 [Full report](./finding-1-dust-griefing-holdingmanager.md)

## 2 Missing Success Verification in `emergencyGenericCall` (Holding.sol)

* **Summary:** Without checking for call success, execution failures may silently occur.
* 👉 [Full report](./finding-2-missing-success-verification.md)

## 3 Unbounded Loop in `borrowMultiple` Enables DoS via Oversized `_data`

* **Summary:** Attacker can supply large input to trigger gas exhaustion in loop logic.
* 👉 [Full report](./finding-3-unbounded-loop-dos.md)

## 4 Mutable `mintLimit` Can Permanently Freeze jUSD Minting

* **Summary:** Setting `mintLimit` to 0 locks minting indefinitely.
* 👉 [Full report](./finding-4-mutable-mintlimit-freeze.md)

## 5 Naming Convention Violation in `getFeeAbsolute` Function

* **Summary:** Lacks `_` prefix, reducing readability and consistency.
* 👉 [Full report](./finding-5-naming-convention-getfeeabsolute.md)

## 6 Lack of Minimum Deposit Enables Dust-Griefing in `deposit` (Staker.sol)

* **Summary:** Small-value deposits can be spammed to increase gas usage and attack UX.
* 👉 [Full report](./finding-6-dust-griefing-staker.md)

## 7 Uninitialized `manager` Pointer Breaks Access-Control & Fee Logic

* **Summary:** Undefined `manager` state leads to loss of governance & execution rights.
* 👉 [Full report](./finding-7-uninitialized-manager-pointer.md)

## 8 Uninitialized Token Addresses Cause Permanent Denial of Service

* **Summary:** Token-related functions revert permanently without proper initialization.
* 👉 [Full report](./finding-8-uninitialized-token-dos.md)

---

### 📂 Directory Structure

```
.
├── finding-1-dust-griefing-holdingmanager.md
├── finding-2-missing-success-verification.md
├── finding-3-unbounded-loop-dos.md
├── finding-4-mutable-mintlimit-freeze.md
├── finding-5-naming-convention-getfeeabsolute.md
├── finding-6-dust-griefing-staker.md
├── finding-7-uninitialized-manager-pointer.md
├── finding-8-uninitialized-token-dos.md
└── README.md   ← (you are here)
```

### ✍️ Contributing

Security engineers and contributors are welcome to submit issues or pull requests to refine this audit.

---

© <?= date('Y') ?> <?= $ownerName ?? "AuditDrop" ?> – All rights reserved.
