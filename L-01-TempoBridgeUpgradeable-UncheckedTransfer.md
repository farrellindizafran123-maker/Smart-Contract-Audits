## [L-01] Unchecked return value of ERC20 `transferFrom` in `_debit`

**Severity:** Low

### Summary
The `_debit` function in `TempoBridgeUpgradeable.sol` calls `underlyingToken.transferFrom()` without validating its boolean return value. Certain ERC20 tokens do not revert on failure and instead return `false`. Ignoring this return value can lead to protocol state updates (such as bridging tokens) occurring without actual transfer of underlying collateral.

### Vulnerability Details
Based on static analysis using Slither (detector: `unchecked-transfer`), the issue occurs within `contracts/token/TempoBridgeUpgradeable.sol` inside the `_debit` function (lines 64-76):

```solidity
// Line 73: Return value of transferFrom is ignored
underlyingToken.transferFrom(_from, address(this), amountSentLD);
```

If `underlyingToken` is a non-standard ERC20 token (such as USDT or tokens that return `false` on transfer failure rather than reverting), the contract assumes the transfer succeeded and continues execution.

### Impact
An attacker or user could trigger the debit/bridge mechanism without transferring actual tokens into the contract, resulting in insolvency or uncollateralized minting on the target side.

### Recommendation
Use OpenZeppelin's `SafeERC20` library to safely handle non-standard ERC20 return values and ensure execution reverts if a transfer fails.

**Recommended Code Changes:**
```solidity
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract TempoBridgeUpgradeable {
    using SafeERC20 for IERC20;

    function _debit(address _from, uint256 amountSentLD, ...) internal {
        // Use safeTransferFrom to enforce check on return value
        IERC20(address(underlyingToken)).safeTransferFrom(_from, address(this), amountSentLD);
    }
}
```
