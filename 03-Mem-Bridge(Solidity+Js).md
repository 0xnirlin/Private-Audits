# MEM-tech Audit Findings

## Summary Table

| Severity | Number of Findings |
|----------|---------------------|
| High     | 2                   |
| Medium   | 2                   |
| Low      | 4                   |
| JS Issues | 2                   |
| Info/Gas Optimizations | 7      |
| Total    | 17                  |

| # | Title | Severity | Context |
|---|-------|----------|---------|
| 1.1 | Permissionless validateUnlock() allows anyone to spam empty requests and use up all designated oracleFee funds | High | bridge.sol#L90-L130 |
| 1.2 | A malicious user can grief contract out of oracle balance if the total lock/unlock fee per tx is less than oracle fee per tx | High | bridge.sol#L122 |
| 2.1 | Calling executeUnlock() before fulfill() will render all funds stuck | Medium | bridge.sol#L154-L169 |
| 2.2 | Bridging funds is incompatible with smart contract wallets | Medium | bridge.sol#L175-L203 |
| 3.1 | Implement treasury setter function in case of emergency | Low | N/A |
| 3.2 | Restrict fee setting to MIN/MAX range | Low | N/A |
| 3.3 | Subtract amount instead of net_amount | Low | N/A |
| 3.4 | Hardcoded MEM URL is not recommended | Low | N/A |
| 4.1 | Current address check is not compatible with all EVM compatible chains | JS Low | bridge.js#L93-L98 |
| 4.2 | Max value a js integer can hold is 9007199254740991 which can lead to loss of funds | JS Low | N/A |
| 5.1 | Redundant code can be removed | Info | N/A |
| 5.2 | No address(0) check on setter functions | Info | N/A |
| 5.3 | Use locked instead of unlocked pragma | Info | N/A |
| 5.4 | There is no way to query MemID to RequestID | Info | N/A |
| 5.5 | Implement errors instead of revert string for gas savings | Gas | N/A |
| 5.6 | Use solady string library for more gas optimized hex conversion | Gas | N/A |
| 5.7 | Use solady safeTransferLib library to save gas | Gas | N/A |

## 1. High Risk

### 1.1 Permissionless validateUnlock() allows anyone to spam empty requests and use up all designated oracleFee funds

**Severity:** High

**Context:** bridge.sol#L90-L130

**Description:**
The function to validate an unlock is permissionless and the only check made is that the _memid is not yet redeemed.

```solidity
function validateUnlock(
string calldata _memid
) public returns (bytes32 requestId) {
// memid can be redeemed once
assert(!midIsRedeemed[_memid]);
```

A Chainlink Client request is then created and filled with data, after which the request is pushed to the Chainlink operator.

```solidity
// Sends the request
requestId = sendOperatorRequest(req, oracleFee);
// map requestId to caller
reqToCaller[requestId] = msg.sender;
// map the chainlink requestId to memid
reqToMemId[requestId] = _memid;
// map the memid redeeming status to false
midIsRedeemed[_memid] = false;
return requestId;
```

This allows anyone to spam the function with random _memIds and use up all available $LINK token balance designated for paying oracle fees in the contract. When the funds run out, the system will cease to function properly and will be temporarily DOS'd until new funds are added by the protocol team.

**Recommendation:**
At the very least, implement access control to validate that only users that have locked funds can call the function. Additional input validation like a mapping from addr => memId is required. Otherwise, if a user has locked 1 wei, they can still pass the access control check and spam different _memIds which will achieve the same result.

### 1.2 A malicious user can grief contract out of oracle balance if the total lock/unlock fee per tx is less than oracle fee per tx

**Severity:** High

**Context:** bridge.sol#L122

**Description:**
If the oracle fee that is paid is not >= bridgeFee, the function can be griefed at virtually no cost by anyone using the protocol as intended. For example, a user can split up a 100USDC lock/unlock into many multiple different transactions, and since on L2 the gas is negligible, their total bridge fee will still be 0.5% of amount + gas. But if the oracle fee does not cover that, a malicious user can grief all of the remaining oracle fee balance in the contract while retaining functionality and losing nothing.

```solidity
requestId = sendOperatorRequest(req, oracleFee);
```

**Recommendation:**
Require for every tx that bridgeFee >= oracleFee.

## 2. Medium Risk

### 2.1 Calling executeUnlock() before fulfill() will render all funds stuck

**Severity:** Medium

**Context:** bridge.sol#L154-L169

**Description:**
If a user calls validateUnlock() for a memId with intended amount to transfer 100 WETH, the oracle does the request side validation on chain and pass in the amount as result into the requests mapping.

```solidity
// Sends the request
requestId = sendOperatorRequest(req, oracleFee);
// map requestId to caller
reqToCaller[requestId] = msg.sender;
// map the chainlink requestId to memid
reqToMemId[requestId] = _memid;
// map the memid redeeming status to false
midIsRedeemed[_memid] = false;
return requestId;
```

After this a user can call the executeUnlock() for that memId and unlock their tokens. However, in the case that a user calls it before validating first, the transaction will still pass but the default value for the request will be 0 since chainlink has not assigned it an amount, it will be marked redeemed. It will revert even if the oracle tries to fulfill it after this call, hence locking in all user funds.

**Recommendation:**
In executeUnlock() check that the amount returned from requests mapping is non zero.

### 2.2 Bridging funds is incompatible with smart contract wallets

**Severity:** Medium

**Context:** bridge.sol#L175-L203

**Description:**
When funds are to be bridged from EVM <> MEM, users call lock() which calculates the lock fee, transfers the tokens in and adjusts user & treasury balances accordingly:

```solidity
function lock(uint256 _amount) external {
    uint256 net_amount = computeNetAmount(_amount);
    uint256 generateFees = _amount - net_amount;
    // ERC20 token transfer
    token.safeTransferFrom(msg.sender, address(this), _amount);
    // update balances map
    balanceOf[msg.sender] += net_amount;
    // update treasury balance from fee cut
    balanceOf[treasury] += generateFees;
    // update totalLocked amount
    totalLocked += net_amount;
    //update treasury cumultive fee
    cumulativeFees += generateFees;
    // emit event
    emit Lock(msg.sender, net_amount);
}
```

The issue is that if a user is using a smart contract wallet instead of an EOA, the address will be different on different networks. When the locked in amount is incremented as balanceOf[msg.sender], but the user's address on the receiving side is different, this will lead to the funds being attributed to another address and potentially lost.

**Recommendation:**
Let the user pass an argument for receiving address _to.

## 3. Low Risk

### 3.1 Implement treasury setter function in case of emergency

**Severity:** Low

**Description:**
Currently the contract is missing functionality to change the treasury address once set in the constructor. In case of emergency or compromised treasury EOA, the protocol will lose all accumulated and future fees and will likely need to be re-deployed. Consider adding such functionality to the contract.

### 3.2 Restrict fee setting to MIN/MAX range

**Severity:** Low

**Description:**
There are currently no MIN/MAX ranges when setting fees in the protocol. Consider adding a >MIN_FEE && <MAX_FEE constants range when setting fees in the constructor and in setFeeInJuels().

### 3.3 Subtract amount instead of net_amount

**Severity:** Low

**Description:**
Currently the wrong value is subtracted from the totalLocked variable when unlocking. Make sure to subtract amount instead of net_amount.

### 3.4 Hardcoded MEM URL is not recommended

**Severity:** Low

**Description:**
The URL is currently hardcoded as following:

```solidity
string memory arg1 = string.concat("https://0xmem.net/vu/", _memid);
```

There is no way to set a new URL, so in case the domain is lost or changed, there is no way to adjust it to the new one. Consider adding a function to set a new one.

## 4. JS Issues

### 4.1 Current address check is not compatible with all EVM compatible chains

**Severity:** JS Low

**Context:** bridge.js#L93-L98

**Description:**
Bridge.js validates the account address by checking its format, but the problem is not all the EVM compatible chains have the same address format that we see on ethereum. For example, on Tron, which is EVM compatible, an EOA address looks like this:

```
TTUCJ3dKKikv3AhScBQ9aRsGeJ2EuBXQhd
```

So for such chains this check will fail:

```javascript
function _validateEoaSyntax(address) {
    ContractAssert(
        /^(0x)?[0-9a-fA-F]{40}$/.test(address),
        "ERROR_INVALID_EOA_ADDR",
    );
}
```

### 4.2 Max value a js integer can hold is 9007199254740991 which can lead to loss of funds

**Severity:** JS Low

**Description:**
In solidity largest type of integer is uint256 which can hold max value of uint256.max(115792089237316195423570985008687907853269984665640564039457584007913129639935) but in JS the max value for integer is:

```
2^53-1, or
+/- 9,007,199,254,740,991
```

And adding any number to it won't overflow it to zero like old solidity versions but instead it remains the same.

Consider using bigInt.

## 5. Info/Gas Optimizations

### 5.1 Redundant code can be removed

**Severity:** Info

**Description:**
Redundant code can be removed on L#164.

### 5.2 No address(0) check on setter functions

**Severity:** Info

**Description:**
There are currently no address(0) checks when setting addresses in the constructor or when updating the oracle's address in setOracleAddress(). Consider implementing these checks.

### 5.3 Use locked instead of unlocked pragma

**Severity:** Info

**Description:**
Currently, the pragma version is unlocked, it is recommended to lock the pragma to the latest compatible version since there are usually gas optimizations and bug fixes.

### 5.4 There is no way to query MemID to RequestID

**Severity:** Info

**Description:**
In the contract there is a mapping for reqToMemId to get the memId for a request, consider adding a mapping for memIdToRequestId.

### 5.5 Implement errors instead of revert string for gas savings

**Severity:** Gas

**Description:**
Reverting with a custom error instead of a string in require statements saves gas, it is recommended to change all instances of strings to errors.

All instances are on lines L141, L144, L183, L185, L216, L228.

### 5.6 Use solady string library for more gas optimized hex conversion

**Severity:** Gas

**Description:**
Solady provides a more gas-optimized string library than OpenZeppelin's whilst having the same toHexString function. Saves a lot of gas.

### 5.7 Use solady safeTransferLib library to save gas

**Severity:** Gas

**Description:**
Solady also has a gas-optimised safeTransferLib library written in the assembly that can save a lot of gas for the user. Use that in your codebase for better UX.