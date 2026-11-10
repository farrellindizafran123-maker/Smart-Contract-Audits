## [H-01] Reentrancy vulnerability in `EigenServiceManager.slash()` allows manipulation of slashing logic

**Severity:** High

### Summary
The `slash` function in `EigenServiceManager.sol` is vulnerable to a reentrancy attack. The function reads a token balance, performs external calls via the internal `_slash()` function, and subsequently uses the previously calculated balance in a conditional check. The absence of a `nonReentrant` modifier and the violation of the Checks-Effects-Interactions (CEI) pattern allow an attacker to manipulate the state and bypass intended slashing logic using stale balances.

### Vulnerability Details
Based on static analysis using Slither (detector: `reentrancy-balance`), the vulnerability occurs in the following sequence within `contracts/delegation/providers/eigenlayer/EigenServiceManager.sol`:

1. **Balance Read (Line 72):** The contract reads its current balance of slashed collateral:
   ```solidity
   uint256 beforeSlash = _slashedCollateral.balanceOf(address(this));
   ```

2. **External Interaction (Line 78):** The contract executes the internal function `_slash()`, which triggers external calls to `IAllocationManager` and `IStrategyManager`.
   ```solidity
   _slash(_strategy, _operator, _slashShare);
   ```

3. **Stale Balance Usage (Line 82):** After the external calls are completed, the contract evaluates a condition based on `slashedAmount` (derived from the pre-call balance):
   ```solidity
   if (slashedAmount == 0) { ... }
   ```

Because the `slash` function (defined at line 58) does not utilize a reentrancy guard, an attacker can hijack the execution flow during the external calls at step 2. By re-entering the contract or manipulating the token balance before the execution reaches line 82, the calculated `slashedAmount` becomes stale and inaccurate.

### Impact
An attacker can exploit this stale balance condition to bypass the protocol's slashing mechanism. Depending on the collateral token's characteristics (e.g., ERC777 callback hooks) or the behavior of the external managers, this could lead to the attacker avoiding being slashed, locking the protocol's state, or unfairly redistributing collateral.

### Recommendation
To secure the function against reentrancy, it is highly recommended to apply the Checks-Effects-Interactions (CEI) pattern and implement a mutex lock.

1. **Implement `ReentrancyGuard`:** Inherit OpenZeppelin's `ReentrancyGuard` and apply the `nonReentrant` modifier to the `slash` function.
2. **Refactor using CEI Pattern:** Move the external interaction (`_slash()`) to the very end of the function, strictly after all balance calculations and condition checks have been validated.

**Recommended Code Changes:**
```solidity
// 1. Import and inherit ReentrancyGuard at the contract level
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract EigenServiceManager is ReentrancyGuard {
    // ...

    // 2. Add `nonReentrant` modifier to the function signature
    function slash(
        address _strategy, 
        address _operator, 
        uint256 _slashShare, 
        uint48 _time
    ) external nonReentrant {
        
        // --- CHECKS & EFFECTS ---
        // Read balance
        uint256 beforeSlash = _slashedCollateral.balanceOf(address(this));
        
        // (Calculate slashedAmount based on beforeSlash)
        
        // Execute conditional checks based on the calculated amount
        if (slashedAmount == 0) revert InvalidSlashingAmount();

        // --- INTERACTIONS ---
        // 3. Move external calls to the end of the function
        _slash(_strategy, _operator, _slashShare);
    }
}
```
