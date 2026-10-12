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
