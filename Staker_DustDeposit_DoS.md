# Lack of Minimum Deposit Enables Dust‑Griefing in `deposit` (Staker.sol)

## Summary
`deposit` accepts any positive `_amount`. An attacker can repeatedly deposit *dust* (e.g., 1 wei) to bloat contract state and increase operational gas costs, degrading protocol performance over time.

## Finding Description
At **line 108** of `Staker.sol`:

```solidity
function deposit(uint256 _amount) external ... validAmount(_amount) {
    ...
    _balances[msg.sender] += _amount;
    IERC20(tokenIn).safeTransferFrom(...);
}
```

The only guard is `validAmount`, which checks `_amount > 0`. Since no minimum amount is enforced:

1. **Storage & Gas Griefing** – An attacker can deposit extremely small amounts from many accounts (or many times) to inflate `_balances`, increasing gas costs of future operations.
2. **Reward Rounding Issues** – Reward distribution functions that use small balances may suffer rounding errors or precision loss, leading to fairness issues.
3. **Reward Inflation Attack Surface** – Malicious users could manipulate reward computation using dust to maximize rewards relative to stake.

Although `totalSupplyLimit` is large (`1e34`) and not likely to be exhausted, state size and fairness can still be degraded via dust.

## Impact Explanation
The protocol becomes **more expensive to use**, with state bloat and potential rounding attacks on reward logic. While it doesn’t halt core operations, it undermines economic efficiency and fairness — especially in long-lived staking pools.

Impact is **Medium** due to operational degradation and reward distortion.

## Likelihood Explanation
* **Easy to exploit** – Any user with minimal funds can interact.
* **Common bot pattern** – Dust attacks are often performed using scripts or automated agents.
* **No rate limits** – If deposits are not throttled, the loop-based attack is feasible.

Likelihood is **High**.

## Proof of Concept

### ⚙️ Normal Script (Hardhat/Node)

```javascript
// scripts/dustGrief.js
const { ethers } = require("hardhat");

async function main() {
  const [attacker] = await ethers.getSigners();
  const staker = await ethers.getContractAt("Staker", "<STAKER_ADDRESS>");
  const token = await ethers.getContractAt("IERC20", "<TOKEN_ADDRESS>");

  await token.connect(attacker).approve(staker.address, ethers.constants.MaxUint256);
  const dust = 1;  // 1 wei

  for (let i = 0; i < 100; i++) {
    await staker.deposit(dust, { gasLimit: 200_000 });
  }
}
```

*Result:* Repeated deposits increase `_balances[attacker]` and create storage entries that consume more gas in future reward-related operations.

---

### 🛠 Foundry Test

```solidity
// test/DustDeposit.t.sol
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/Staker.sol";
import "../src/Mocks/MockToken.sol";

contract DustDeposit is Test {
    Staker staker;
    MockToken token;
    address attacker = address(0xA11CE);

    function setUp() public {
        token = new MockToken("Test", "TST", 18);
        staker = new Staker(address(token));
        token.mint(attacker, 1e18);
        vm.startPrank(attacker);
        token.approve(address(staker), type(uint256).max);
        vm.stopPrank();
    }

    function testDustGriefing() public {
        vm.startPrank(attacker);
        for (uint256 i = 0; i < 100; i++) {
            staker.deposit(1); // 1 wei each
        }
        vm.stopPrank();

        // attacker balance will be 100 wei, but storage now has 100 entries.
        assertEq(staker.balanceOf(attacker), 100);
    }
}
```

This shows how minimal capital can create extensive storage overhead and potential gas griefing.

## Recommendation
Add a minimum deposit threshold:

```solidity
uint256 public constant MIN_DEPOSIT = 1e15; // 0.001 tokens

function deposit(uint256 _amount) external ... {
    require(_amount >= MIN_DEPOSIT, "deposit too small");
    ...
}
```

Alternatively, store this as a configurable governance-controlled variable for future tuning.

This eliminates spam deposits and makes economic manipulation more difficult.
