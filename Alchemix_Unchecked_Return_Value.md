# Security Audit Report: Alchemix v3

## [L-01] Unchecked Return Value of ERC20 Transfer in `MYTStrategy.sol` Leads to Silent Failures

**Severity:** Low / QA
**Target:** Alchemix v3
**Contract:** `MYTStrategy.sol`
**Function:** `rescueTokens(address, address, uint256)`
**Methodology:** Slither Static Analysis, Manual Review

### Description
The `rescueTokens` function in `MYTStrategy.sol` executes a standard ERC20 `transfer` to move tokens to a specified recipient. However, the function does not verify the boolean return value of the `transfer` call. 

While the standard ERC20 specification dictates that a `transfer` function should return `true` upon success and `false` upon failure, many deployed tokens in the DeFi ecosystem deviate from this standard. Some tokens return `false` instead of reverting the transaction when a transfer fails (e.g., due to insufficient balance or contract pausing). 

Because the return value is ignored in this contract, if a token transfer fails and returns `false`, the `rescueTokens` function will continue execution and complete successfully. This creates a "silent failure" where the protocol assumes the tokens were safely transferred, but the on-chain state remains unchanged.

### Vulnerability Details
The vulnerable code implementation is located in `src/MYTStrategy.sol`:

```solidity
function rescueTokens(address token, address to, uint256 amount) external onlyOwner {
    require(to != address(0), "Invalid recipient");
    require(!_isProtectedToken(token), "Protected token");
    uint256 balance = IERC20(token).balanceOf(address(this));
    // ...
    
    // [VULNERABLE CODE] Return value is ignored
    IERC20(token).transfer(to, amount); 
}
```

Although this function is protected by the `onlyOwner` modifier—restricting access to authorized administrators and preventing malicious external exploitation (which downgrades the severity to Low/QA)—the lack of return value validation degrades the operational reliability of the protocol. If an admin attempts to rescue a non-standard ERC20 token and the transfer fails, no error is thrown, leading to administrative confusion.

### Impact
*   **Operational Risk:** Administrators may falsely believe a token rescue operation was successful.
*   **Off-chain Inconsistency:** Off-chain tracking systems or indexing scripts monitoring transaction success will record a successful execution, whereas the actual token balances on the blockchain remain unchanged.

### Recommendation
It is a widely accepted industry best practice to either explicitly check the return value using a `require` statement or utilize OpenZeppelin's `SafeERC20` library to handle non-standard ERC20 behaviors safely.

**Option 1: Explicit Require (Minimal Code Change)**
Bind the transfer execution to a boolean variable and revert the transaction if it returns false.

```solidity
bool success = IERC20(token).transfer(to, amount);
require(success, "MYTStrategy: Token transfer failed");
```

**Option 2: SafeERC20 (Industry Standard)**
Import and apply OpenZeppelin's `SafeERC20` library, then replace the standard `.transfer` with `.safeTransfer`.

```solidity
using SafeERC20 for IERC20;

// Inside the rescueTokens function:
IERC20(token).safeTransfer(to, amount);
```
