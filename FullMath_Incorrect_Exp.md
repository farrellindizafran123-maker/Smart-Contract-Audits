## [False Positive] Slither Detector: incorrect-exp on mulDiv

### 1. Preliminary Findings (Automated Scanner Report)
* **Contract Target:** `FullMath.sol`
* **Detected Function:** `mulDiv(uint256,uint256,uint256)`
* **Code Location:** Line 93
* **Warning Type:** `incorrect-exp`

**Output Terminal:**
```text
INFO:Detectors:
Detector: incorrect-exp
FullMath.mulDiv(uint256,uint256,uint256) (src/AntePool/contracts/libraries/FullMath.sol#20-112) has bitwise-xor operator ^ instead of the exponentiation operator **:
    - inv = (3 * denominator) ^ 2 (src/AntePool/contracts/libraries/FullMath.sol#93)
```

### 2. Affected Code Snippet
```solidity
// Initial guess for the Newton-Raphson method
inv = (3 * denominator) ^ 2;
```

### 3. Manual Analysis & Logical Argument
The `FullMath` library implements 512-bit precision math, adopting the standard algorithm from Uniswap V3. Slither's static analysis flags the bitwise XOR operator (`^`) as a typographical error, incorrectly assuming the developer intended to use the exponentiation operator (`**`).

Upon manual review, the expression `(3 * denominator) ^ 2` provides the mathematically correct initial seed required for finding the modular inverse modulo 2^256 using the Newton-Raphson iteration method. The automated scanner lacks the contextual mathematical awareness of this specific cryptographic algorithm and falsely flags a strictly valid bitwise operation.

### 4. Audit Conclusion
Classified as a **False Positive**. The bitwise XOR operator (`^`) is deliberate and strictly required for the algorithm's accuracy. Altering it to an exponentiation operator (`**`) as suggested by the scanner would completely break the modular inverse calculation and corrupt the contract's arithmetic precision.
