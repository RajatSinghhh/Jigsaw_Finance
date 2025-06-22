# Jigsaw Finance Audit – Findings Summary

This document summarizes critical and informational findings uncovered during the security assessment of the protocol’s smart contract architecture. Each vulnerability has been detailed in individual reports stored in this directory.

---

## 📑 Table of Contents

| ID | Finding Title                                                                             | Severity<sup>†</sup> | Link                                                        |
| -- | ----------------------------------------------------------------------------------------- | -------------------- | ----------------------------------------------------------- |
| 1  | Lack of Minimum Deposit Threshold Enables Dust-Griefing in `HoldingManager.deposit`       | Medium               | [Report 1](./DustGriefing_HoldingManagerDeposit.md)     |
| 2  | Missing Success Verification in `emergencyGenericCall` (Holding.sol)                      | Medium               | [Report 2](./Holding_EmergencyCallBug.md)     |
| 3  | Unbounded Loop in `borrowMultiple` Enables DoS via Oversized `_data` (HoldingManager.sol) | Medium                | [Report 3](./HoldingManager_BorrowMultiple_DOS.md)               |
| 4  | Mutable `mintLimit` Can Permanently Freeze jUSD Minting (JigsawUsd.sol)                   | Medium                 | [Report 4](./JigsawUSD_MintLimit_Freeze.md)         |
| 5  | Naming Convention Violation in `getFeeAbsolute` Function (OperationsLib.sol)              | Informational        | [Report 5](./OperationsLib_InternalNamingBug.md) |
| 6  | Lack of Minimum Deposit Enables Dust-Griefing in `deposit` (Staker.sol)                   | Medium               | [Report 6](./Staker_DustDeposit_DoS.md)             |
| 7  | Uninitialized `manager` Pointer Breaks Access-Control & Fee Logic                         | High                 | [Report 7](./UninitializedManagerPointer.md)    |
| 8  | Uninitialized Token Addresses Cause Permanent DoS in Multi-Strategy Contracts             | High                 | [Report 8](./UninitializedTokenAddresses.md)          |

> <sup>†</sup> *Severity levels are provisional. Refer to each report for detailed context.*

---

## 1 Lack of Minimum Deposit Threshold Enables Dust-Griefing in `HoldingManager.deposit`

* **Summary:** Arbitrarily small deposits can be abused to bloat storage and gas usage.
* 👉 [Full report](./DustGriefing_HoldingManagerDeposit.md)

## 2 Missing Success Verification in `emergencyGenericCall` (Holding.sol)

* **Summary:** Without checking for call success, execution failures may silently occur.
* 👉 [Full report](./Holding_EmergencyCallBug.md)

## 3 Unbounded Loop in `borrowMultiple` Enables DoS via Oversized `_data`

* **Summary:** Attacker can supply large input to trigger gas exhaustion in loop logic.
* 👉 [Full report](./HoldingManager_BorrowMultiple_DOS.md)

## 4 Mutable `mintLimit` Can Permanently Freeze jUSD Minting

* **Summary:** Setting `mintLimit` to 0 locks minting indefinitely.
* 👉 [Full report](./JigsawUSD_MintLimit_Freeze.md)

## 5 Naming Convention Violation in `getFeeAbsolute` Function

* **Summary:** Lacks `_` prefix, reducing readability and consistency.
* 👉 [Full report](./OperationsLib_InternalNamingBug.md)

## 6 Lack of Minimum Deposit Enables Dust-Griefing in `deposit` (Staker.sol)

* **Summary:** Small-value deposits can be spammed to increase gas usage and attack UX.
* 👉 [Full report](./Staker_DustDeposit_DoS.md)

## 7 Uninitialized `manager` Pointer Breaks Access-Control & Fee Logic

* **Summary:** Undefined `manager` state leads to loss of governance & execution rights.
* 👉 [Full report](./UninitializedManagerPointer.md)

## 8 Uninitialized Token Addresses Cause Permanent Denial of Service

* **Summary:** Token-related functions revert permanently without proper initialization.
* 👉 [Full report](./UninitializedTokenAddresses.md)

---

### 📂 Directory Structure

```
.
├── DustGriefing_HoldingManagerDeposit
├──Holding_EmergencyCallBug
├──HoldingManager_BorrowMultiple_DOS
├──JigsawUSD_MintLimit_Freeze
├──OperationsLib_InternalNamingBug
├──Staker_DustDeposit_DoS
├──UninitializedManagerPointer
├──UninitializedTokenAddresses
└── README.md   ← (you are here)
```

### ✍️ Contributing

Security engineers and contributors are welcome to submit issues or pull requests to refine this audit.

---

© <?= date('Y') ?> <?= $ownerName ?? "AuditDrop" ?> – All rights reserved.
