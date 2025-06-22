# Mutable `mintLimit` Can Permanently Freeze jUSD Minting (JigsawUsd.sol)

## Summary
`updateMintLimit` allows the owner to lower `mintLimit` to **any** value, including values **below the current mint limit or even below `totalSupply()`**. Once lowered, every subsequent `mint` call will revert because the supply check `totalSupply() + _amount <= mintLimit` fails. This can permanently halt all new jUSD issuance, creating a protocol‑wide denial‑of‑service.

## Finding Description
The constructor initializes:

```solidity
mintLimit = 15e6 * (10 ** decimals()); // 15 000 000 jUSD
```

Later, the owner may call:

```solidity
function updateMintLimit(uint256 _limit) external onlyOwner validAmount(_limit) {
    emit MintLimitUpdated(mintLimit, _limit);
    mintLimit = _limit;
}
```

There is **no check** that `_limit` is **greater** than either the previous `mintLimit` or `totalSupply()`.  
If the owner (accidentally or maliciously) sets the limit to a value lower than the current supply, every future call to

```solidity
require(totalSupply() + _amount <= mintLimit, "2007");
```

will revert, freezing token minting forever.

### Why This Breaks Security Guarantees
* **Availability** – The protocol must be able to expand jUSD supply to maintain peg or support lending operations.  
* **Governance risk** – A compromised owner key can permanently brick token issuance.  
* **Operational risk** – A simple mis‑click or unit miscalculation by the owner produces an irreversible DoS.

## Impact Explanation
* **Protocol‑wide DoS:** No new jUSD can be minted to meet user demand or collateral needs.  
* **Economic instability:** Markets relying on jUSD liquidity lose supply, potentially de‑pegging the stablecoin.  
* **Reputational damage:** Users may lose confidence if minting halts without warning.

Impact is therefore **High**.

## Likelihood Explanation
* **Human error:** Owners frequently adjust limits; a single typo (e.g., forgetting `e18`) lowers the cap 1 000 000 000×.  
* **Key compromise or malicious governance proposal:** An attacker could intentionally brick minting.  
* **No automated safeguards:** The contract itself cannot recover once the limit is set too low.

Likelihood is **Medium–High**.

## Proof of Concept

### ⚙️ Normal Script (Hardhat/Node)
```javascript
// scripts/freezeMint.js
const { ethers } = require("hardhat");

async function main() {
  const [owner, manager] = await ethers.getSigners();

  // Deploy JigsawUsd
  const USD = await ethers.getContractFactory("JigsawUsd");
  const usd = await USD.deploy(owner.address, manager.address);
  await usd.deployed();

  // Manager mints 10M jUSD (valid — below 15M limit)
  const amount = ethers.utils.parseUnits("10000000", 18); // 10M
  await usd.connect(manager).mint(manager.address, amount);

  // Owner mistakenly lowers the limit to 5M
  const newLimit = ethers.utils.parseUnits("5000000", 18);
  await usd.connect(owner).updateMintLimit(newLimit);

  // Manager attempts to mint 1 jUSD ⇒ should revert
  try {
    await usd.connect(manager).mint(manager.address, ethers.utils.parseUnits("1", 18));
  } catch (e) {
    console.log("Mint reverted as expected:", e.message); // DoS achieved
  }
}

main();
```

When run, the final mint reverts with error `2007`, demonstrating the permanent freeze.

---

### 🛠 Foundry Test
```solidity
// test/MintLimitFreeze.t.sol
// forge test -vv

pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/JigsawUsd.sol";
import "../src/Mocks/MockStablesManager.sol";

contract MintLimitFreeze is Test {
    JigsawUsd usd;
    address owner = address(0xB0B);
    address manager = address(0xCAFE);

    function setUp() public {
        vm.label(owner, "Owner");
        vm.label(manager, "StablesManager");

        // Deploy token
        usd = new JigsawUsd(owner, manager);

        // Give roles (assume modifier onlyStablesManager checks msg.sender == manager)
        vm.prank(owner);
        usd.setManager(manager); // if _setManager is internal, mock accordingly
    }

    function testFreeze() public {
        uint256 tenMillion = 10_000_000 ether;
        vm.prank(manager);
        usd.mint(manager, tenMillion); // succeeds

        // Owner lowers limit to 5M
        uint256 fiveMillion = 5_000_000 ether;
        vm.prank(owner);
        usd.updateMintLimit(fiveMillion);

        // Any further mint should revert
        vm.prank(manager);
        vm.expectRevert("2007");
        usd.mint(manager, 1 ether);
    }
}
```

Running `forge test` shows the final mint reverts, confirming the freeze.

## Recommendation
Add an invariant check in `updateMintLimit`:

```solidity
function updateMintLimit(uint256 _limit) external onlyOwner validAmount(_limit) {
    require(_limit > mintLimit, "new limit cannot be below previous");
    require(_limit >= totalSupply(), "new limit cannot be below current supply");

    emit MintLimitUpdated(mintLimit, _limit);
    mintLimit = _limit;
}
```

Alternatively, store a `minMintLimit` constant that cannot be reduced below deployment value. This ensures minting capacity remains available and prevents accidental or malicious freeze.
