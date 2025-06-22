# Missing Success Verification in `emergencyGenericCall` (Holding.sol)

## Summary
The `emergencyGenericCall` function does not verify that the low‑level call it performs actually succeeded. If the invoked contract reverts or returns `false`, execution continues silently, potentially masking failed emergency actions.

## Finding Description
`emergencyGenericCall` performs a raw `call`:

```solidity
(success, result) = _contract.call{ value: msg.value }(_call);
```

Because the function returns `(bool success, bytes memory result)` without any follow‑up check, callers **must** remember to inspect the returned `success` flag themselves. In practice, many higher‑level routines assume an emergency action just “works” and will not inspect the flag, leaving failed calls undetected.

This breaks the **availability** and **operational integrity** guarantees that an emergency module is expected to provide. A malicious or faulty callee can revert (or return `false`) while all other protocol components proceed under the illusion that the emergency action succeeded.

## Impact Explanation
If an emergency operation silently fails:

* **Protocol recovery procedures may never execute.**  
* Time‑critical mitigations (e.g., pausing transfers, withdrawing funds) can be skipped.  
* Incident responders may assume everything is safe, prolonging or worsening the exploit window.

Given that emergency calls are typically invoked during crisis scenarios, missing their failure is **high‑impact**.

## Likelihood Explanation
Although the emergency role is protected by `onlyEmergencyInvoker`, failures are **reasonably likely**:

1. **Accidental misuse:** An operator could supply malformed calldata or target the wrong contract.
2. **External dependency failure:** The target contract could revert due to internal logic or insufficient gas.
3. **Malicious callee:** An attacker controlling the target contract (e.g., via upgradability) can craft behavior that always reverts.

Because these scenarios can arise without deep effort from an attacker, the likelihood is **medium**.

## Proof of Concept
1. Deploy a dummy contract with:

```solidity
contract AlwaysRevert {
    function doom() external pure {
        revert("nope");
    }
}
```

2. As `EMERGENCY_INVOKER`, call:

```solidity
holding.emergencyGenericCall(
    address(alwaysRevert),
    abi.encodeWithSelector(AlwaysRevert.doom.selector)
);
```

3. Observe that the transaction succeeds at the Holding level, returning `success = false`, yet no revert bubbles up and no on‑chain evidence signals failure.

## Recommendation
Require the `success` flag to be `true`:

```solidity
function emergencyGenericCall(
    address _contract,
    bytes calldata _call
)
    external
    payable
    onlyEmergencyInvoker
    nonReentrant
    returns (bytes memory result)
{
    (bool success, bytes memory data) = _contract.call{ value: msg.value }(_call);
    require(success, "Holding: emergency call failed");
    return data;
}
```

This ensures any revert or failed call bubbles up immediately, alerting operators and halting subsequent logic.
