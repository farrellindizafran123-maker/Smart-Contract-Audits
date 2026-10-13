# Smart-Contract-Audits
Portfolio of Web3 Smart Contract Security Audits

[False Positive] Slither Detector: arbitrary-send-eth on _safeTransfer
<img width="526" height="119" alt="image" src="https://github.com/user-attachments/assets/53b1ce76-8619-4535-8f5c-58f0153083eb" />

**Output Terminal:**
```text
INFO:Detectors:
Detector: arbitrary-send-eth
AntePool._safeTransfer(address,uint256) (src/AntePool/contracts/AntePool.sol#610-612) sends eth to arbitrary user
    Dangerous calls:
    - to.transfer(_min(amount,address(this).balance)) (src/AntePool/contracts/AntePool.sol#611)
Reference: [https://github.com/crytic/slither/wiki/Detector-Documentation#functions-that-send-ether-to-arbitrary-destinations](https://github.com/crytic/slither/wiki/Detector-Documentation#functions-that-send-ether-to-arbitrary-destinations)
```
### Manual Analysis & Reverse Tracing
Upon conducting manual reverse tracing, it is confirmed that the `_safeTransfer` function operates with `internal` visibility, preventing direct external execution. All `public` or `external` parent functions invoking this method strictly hardcode the destination address to `msg.sender`. Consequently, there is no user-controlled input parameter that allows manipulation of the recipient address.


### Conclusion
Classified as a **False Positive**. There is no viable attack vector for malicious actors to siphon funds to external addresses. The contract's underlying logic is secure against arbitrary-send vulnerabilities.

---

## [False Positive] Slither Detector: incorrect-exp on mulDiv

<img width="502" height="94" alt="image" src="https://github.com/user-attachments/assets/514e5c7f-02af-4638-a9ab-7e709cc5bc83" />


**Output Terminal:**
```text
INFO:Detectors:
Detector: incorrect-exp
FullMath.mulDiv(uint256,uint256,uint256) (src/AntePool/contracts/libraries/FullMath.sol#20-112) has bitwise-xor operator ^ instead of the exponentiation operator **:
    - inv = (3 * denominator) ^ 2 (src/AntePool/contracts/libraries/FullMath.sol#93)
Reference: [https://github.com/crytic/slither/wiki/Detector-Documentation#incorrect-exponentiation](https://github.com/crytic/slither/wiki/Detector-Documentation#incorrect-exponentiation)
```
### Affected Code Snippet
```solidity
// Initial guess for the Newton-Raphson method
inv = (3 * denominator) ^ 2;
```
### Manual Analysis

The FullMath library implements 512-bit precision math, adopting the standard algorithm from Uniswap V3. The expression (3 * denominator) ^ 2 provides the mathematically correct initial seed for finding the modular inverse modulo $2^{256}$ using the Newton-Raphson iteration. Slither incorrectly flags the bitwise XOR operator (^) as a typo for exponentiation (**).

### Conclusion

Classified as a False Positive. The bitwise XOR operator is deliberate and strictly required for the algorithm's accuracy. Altering it to exponentiation would completely break the modular inverse calculation.
