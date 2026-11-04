# Security Audit Report: Alchemix v3

## [L-02] Uninitialized State Variable `strategyList` in `PerpetualGauge.sol` Leads to Logic Failures

**Severity:** Low / QA
**Target:** Alchemix v3
**Contract:** `PerpetualGauge.sol`
**Variable:** `strategyList` (Line 175)
**Affected Function:** `getCurrentAllocations(uint256)` (Lines 115-136)
**Methodology:** Slither Static Analysis, Manual Review

### Description
In Solidity, state variables that are declared but not properly initialized default to their zero values (e.g., `0` for integers, `false` for booleans, or an empty array for dynamic arrays). Slither analysis identified that the state variable `strategyList` in `src/PerpetualGauge.sol` is declared but never initialized or populated within the contract's source code.

This uninitialized dynamic array is actively utilized by the `getCurrentAllocations` function. Because the array remains permanently empty (`length == 0`), any loop or logic attempting to iterate over `strategyList` will immediately bypass its execution block. As a result, the function will consistently return a default value of `0` (or an inaccurate base allocation), breaking the core calculation mechanism of the gauge.

### Vulnerability Details
The uninitialized state issue is located in `src/PerpetualGauge.sol`:

```solidity
// [VULNERABLE CODE] Variable is declared but never populated or initialized
address[] public strategyList; // Line 175

// ...

function getCurrentAllocations(uint256 _parameter) public view returns (uint256) { // Lines 115-136
    uint256 totalAllocations = 0;
    
    // Because strategyList was never initialized, strategyList.length is 0.
    // This loop is entirely skipped.
    for (uint i = 0; i < strategyList.length; i++) {
        // Core allocation logic calculation
    }
    
    return totalAllocations; // Consistently returns 0
}
```

### Impact
*   **Broken Protocol Logic:** The `getCurrentAllocations` function fails to compute actual values, rendering the gauge's allocation tracking completely non-functional.
*   **Downstream Failures:** Any upstream administrative functions or external protocols querying this contract for allocation data will receive invalid data (zeroes), which can disrupt yield distribution or reward mechanisms.

### Recommendation
It is strongly recommended to initialize the `strategyList` either during contract deployment via the constructor or by introducing an access-controlled setter function.

**Option 1: Access-Controlled Setter Function (Industry Standard for Updatable Lists)**
Introduce a restricted function allowing the protocol administrator to add valid strategy addresses.

```solidity
address[] public strategyList;

// Add this function to allow population of the array
function addStrategy(address _strategy) external onlyOwner {
    require(_strategy != address(0), "PerpetualGauge: Invalid strategy address");
    strategyList.push(_strategy);
}
```

**Option 2: Initialize via Constructor (For Static Strategy Lists)**
If the strategies are predetermined and immutable, assign them during deployment.

```solidity
address[] public strategyList;

constructor(address[] memory _initialStrategies) {
    require(_initialStrategies.length > 0, "PerpetualGauge: Strategy list cannot be empty");
    strategyList = _initialStrategies;
}
```
