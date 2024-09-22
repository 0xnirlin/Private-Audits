# Chad Finance Audit Findings - March 2024

## Summary Table

| Severity | Number of Findings |
|----------|---------------------|
| Medium   | 1                   |
| Low      | 3                   |
| Info/Gas Optimizations | 3     |
| Total    | 7                   |

| # | Title | Severity | Context |
|---|-------|----------|---------|
| 1.1 | No minimum borrow amount and incentive to liquidate small positions can create bad debt | Medium | L2Vault.sol#L84-L97 |
| 2.1 | Liquidator's payout should round down instead of up | Low | L2Vault.sol#L311-L319 |
| 2.2 | Oracle cannot be deployed with latest version of OpenZeppelin's Ownable | Low | Oracle.sol |
| 2.3 | No deadline when borrowing can execute function at a later time when the collateral is de-valued | Low | L2Vault.sol#L84-L97 |
| 3.1 | Event emission is wrong when repaying debt | Info | N/A |
| 3.2 | Hardcoded values in conjurer.sol | Info | N/A |
| 3.3 | _getTick function is redundant | Info | N/A |

## 1. Medium Risk

### 1.1 No minimum borrow amount and incentive to liquidate small positions can create bad debt

**Severity:** Medium

**Context:** L2Vault.sol#L84-L97

**Description:**
There is currently no incentive in the protocol to liquidate small borrows that don't ever get repaid. Furthermore, there is no minimum amount to borrow either, as the available borrow amount corresponds to the liquidity available in the UniswapV3 NFT used as collateral:

```solidity
function borrow(uint256 pos, uint256 amount) external {
    _checkOwnerAndtransferNFT(pos);
    _accrueInterest(pos);
    _addDebt(pos, amount);
    conjurer.conjure(msg.sender, amount);
    if(!checkPositionHealth(pos)){
        revert Vault__positionUnderwater();
    }
    emit Borrow(msg.sender, pos, amount);
}
```

Gas on mainnet is becoming evermore expensive so this is a real problem and due to all of this, the protocol can accumulate bad debt from small positions over time which users don't want to liquidate.

**Recommendation:** Set a minimum borrow amount which would counteract this.

## 2. Low Risk

### 2.1 Liquidator's payout should round down instead of up

**Severity:** Low

**Context:** L2Vault.sol#L311-L319

**Description:**
The liquidator's payout is rounded up. Currently in the contract it does not pose a risk, but it is always recommended to round up only for protocol side, while rounding down for clients. In case of future changes or integrations, this type of rounding might create issues.

### 2.2 Oracle cannot be deployed with latest version of OpenZeppelin's Ownable

**Severity:** Low

**Context:** Oracle.sol

**Description:**
The oracle contract is missing a constructor which is necessary when using the latest version of OZ's Ownable contract. The issue is that an address needs to be specified in the constructor's Ownable(address), but since the oracle contract is missing one, compilation will revert and contract cannot be deployed. Since foundry did not install the dependencies, we could not assess which version is used. In that case we installed the latest versions where it is an issue.

### 2.3 No deadline when borrowing can execute function at a later time when the collateral is de-valued

**Severity:** Low

**Context:** L2Vault.sol#L84-L97

**Description:**
When users borrow from the protocol, they collaterize their Uniswap V3 NFT, whose value determines the amount they can borrow. The maxDebt() function is the one which values the NFT and outputs the available borrow amount:

```solidity
function maxDebt(uint256 tokenId) public view returns (uint256){
    (
        address token0,
        address token1,
        uint256 amount0,
        uint256 amount1,
        TokenInfo memory token0Info,
        TokenInfo memory token1Info
    ) = _getTokenAndAmounts(tokenId);
    uint256 coll = Math.min(token0Info.collateralFactor, token1Info.collateralFactor);
    if(coll == 0){
        return 0;
    }
    return Math.mulDiv(
        coll,
        _calculatePrice(
            amount0,
            amount1,
            uint256(token0Info.decimals),
            uint256(token1Info.decimals),
            token0,
            token1
        ),
        BASIS_POINT);
}
```

But when users are borrowing, since there is no deadline, their transaction can get executed at a later time when the value of the collateral is less. This allows for a scenario where a user's borrow amount could be much closer to the liquidation threshold than intended due to price fluctuations between transaction submission and execution.

**Recommendation:** Allow deadline to be passed for borrows since the collateral of the NFT can drop.

## 3. Info/Gas Optimizations

### 3.1 Event emission is wrong when repaying debt

**Severity:** Info

**Description:**
When repaying debt and amount <= interest, at the end of the if statement, amount is zero'd out. Since the variable is later used in an event emission, it will always emit 0 amount repaid in this case.

### 3.2 Hardcoded values in conjurer.sol

**Severity:** Info

**Description:**
Hardcoded values in conjurer are not recommended in case of deploying on multiple chains. It is recommended to add them to the constructor instead.

### 3.3 _getTick function is redundant

**Severity:** Info

**Description:**
The _getTick function is currently not used anywhere in the protocol and redundant.