# Dein Finance Audit Findings - High and Medium Severity Issues Report

## Summary of Findings

| ID | Title | Severity |
|----|-------|----------|
| H-01 | Price feed updates malfunction if price drop or increase by 50% and then stabilize there | High |
| M-01 | In dein staking applyVoterPenalty does not update the locked amount for the vesting users | Medium |
| M-02 | DOS of BMI Swapping functionality if the position NFT is transferred/traded | Medium |
| M-03 | DEIN tokens not swapped in exchange for BMI tokens get stuck in the SwapEvent.sol forever | Medium |
| M-04 | Hardcoded number of blocks_per_day can cause issues in case of future upgrades | Medium |
| M-05 | addLiquidityETH and removeLiquidityETH calls in AbstractLiquidityMiningStaking.sol will fail due to hardcoded slippage settings | Medium |
| M-06 | The use of the permit() function is susceptible to frontrunning attacks, resulting in a denial-of-service (DOS) for users | Medium |
| M-07 | Restake Reward can be used to game the reward system | Medium |

## High Severity

### [H-01] Price feed updates malfunction if price drop or increase by 50% and then stabilize there

**Description:**
The `_updateTokenPrice` function in the smart contract is designed to update the price of a given token. However, there's a critical issue where price updates can be blocked if the token's price suddenly drops or increases by 50% or more and then stabilizes at that level. This creates a deadlock scenario where the price cannot be updated in the smart contract, effectively causing a denial of service (DOS) for price updates.

**Relevant Code:**
```solidity
function _updateTokenPrice(Token _token, uint256 _tokenPrice) internal {
    if (_tokenPriceList[_token].length() == MAX_PRICE_LIST) {
        uint256 _oldPrice = _tokenPriceList[_token].at(0);
        _tokenPriceList[_token].remove(_oldPrice);
    }
    uint256 _tokenPriceAverage = _getAveragePriceInUSDT(_token);
    uint256 priceChangeRatio = _tokenPrice.mul(PERCENTAGE_100).tryDiv(_tokenPriceAverage);
    uint256 priceChangePercentage = PERCENTAGE_100.trySub(priceChangeRatio);
    if (priceChangePercentage == 0)
        priceChangePercentage = priceChangeRatio.sub(PERCENTAGE_100);
    if (priceChangePercentage != 0 && priceChangePercentage < PRICE_CHANGE_THRESHOLD) {
        _tokenPriceList[_token].add(_tokenPrice);
        if (_token == Token.DEIN) emit DEINPriceUpdated(block.timestamp, _tokenPrice);
        else emit ETHPriceUpdated(block.timestamp, _tokenPrice);
    }
}
```

**Impact:**
If the price change is not zero and is greater than 50%, the new price is not recorded. The system continues to compare with the old pricing, causing price updates to fail consistently. This results in the system operating with incorrect pricing, which can have severe implications across the entire codebase.

**Recommendation:**
Implement a Time-Weighted Average Price (TWAP) Oracle for price updates. This approach can significantly reduce the impact of price manipulation by averaging out the price over a period, making it more challenging for bad actors to influence the price with a single large transaction.

## Medium Severity

### [M-01] In dein staking applyVoterPenalty does not update the locked amount for the vesting users

**Description:**
When applying a penalty in the `deinstaking.sol` contract using the `applyVoterPenalty()` function, it only updates the `stakers[_tokenId]` mapping. There's no check for vesting users, and the locked amount is not updated. This can lead to users claiming more than they should be entitled to.

**Relevant Code:**
```solidity
function claim(uint256 _tokenId, uint256 _amount)
    public
    override
    updateReward(_tokenId)
    forceUpdateTokensPrice
{
    // ... [existing code] ...
    VestingInfo storage vesting = vestings[_tokenId];
    uint256 _dueAmount = deinStakingView.getDueVestedAmount(
        stakers[_tokenId].lockingPeriod,
        vesting.locked,
        vesting.claimed
    );
    // ... [rest of the function] ...
}
```

**Recommendation:**
In the `applyVoterPenalty` function, add an additional check for vesting users and update the locked amount as well.

### [M-02] DOS of BMI Swapping functionality if the position NFT is transferred/traded

**Description:**
If a user transfers their NFT position to another user, it completely blocks the original user from subsequent swaps. This is due to the `tokenToVesting` mapping not being updated in the `_beforeTokenTransfer` function.

**Recommendation:**
Update the `tokenToVesting` mapping in the `_beforeTokenTransfer()` function to reflect NFT transfers.

### [M-03] DEIN tokens not swapped in exchange for BMI tokens get stuck in the SwapEvent.sol forever

**Description:**
If not all BMI holders participate in the token swap, the unclaimed DEIN tokens become permanently trapped within the contract. There's no function defined to claim or handle these remaining tokens after the event ends.

**Recommendation:**
Implement a function to transfer unclaimed tokens to another address (e.g., DEINTreasury) at the conclusion of the event. Consider incorporating a `burn()` function for DEIN tokens.

### [M-04] Hardcoded number of blocks_per_day can cause issues in case of future upgrades

**Description:**
The `blocks_per_day` constant is hardcoded, which can lead to significant discrepancies in calculated blocks per month and year if there are changes to the network's block time (e.g., due to hardforks, upgrades, or deployment to L2 networks).

**Relevant Code:**
```solidity
uint256 public constant blocks_per_day = 7200;
```

**Recommendation:**
Implement a setter function, only callable by the owner, to update the value of `blocks_per_day` in case of network changes or discrepancies.

### [M-05] addLiquidityETH and removeLiquidityETH calls in AbstractLiquidityMiningStaking.sol will fail due to hardcoded slippage settings

**Description:**
The `addLiquidityETH` and `removeLiquidityETH` functions have issues with hardcoded deadline and slippage parameters. The deadline is set to the maximum possible value, and the slippage is hardcoded to 1%, which lacks flexibility and may lead to complications in varying liquidity scenarios.

**Relevant Code:**
```solidity
router.addLiquidityETH{value: msg.value}(
    address(deinToken),
    amountDEIN,
    amountDEIN.mul(99).div(100),
    msg.value.mul(99).div(100),
    address(this),
    type(uint256).max
);
```

**Recommendation:**
Allow users to pass the slippage and deadline parameters themselves.

### [M-06] The use of the permit() function is susceptible to frontrunning attacks, resulting in a denial-of-service (DOS) for users

**Description:**
The `permit()` function is vulnerable to frontrunning attacks due to its design and the visibility of transactions in the mempool. This can lead to a denial-of-service for users, as it disrupts the expected functionality following the `permit()` call.

**Recommendation:**
Wrap any calls that use `permit()` in a try/catch block. This way, even if an attacker front-runs the transaction, the function will continue to execute following the call to `permit()`.

### [M-07] Restake Reward can be used to game the reward system

**Description:**
Users can potentially game the reward system by restaking rewards into existing positions with higher staking multipliers, even when the original locking period is almost over.

**Recommendation:**
Assess the restaking mechanism. One approach could be to only allow restaking rewards into new positions, not existing ones.