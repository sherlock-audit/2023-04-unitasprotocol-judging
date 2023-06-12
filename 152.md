kutugu

medium

# ReserveRatio calculation has precision error

## Summary

ReserveRatio calculation has precision error of division before multiplication

## Vulnerability Detail

```solidity
    uint256 valueBase = 10 ** tokenManager.usd1().decimals();
    reserveRatio = ScalingUtils.scaleByBases(
        allReserves * valueBase / liabilities,
        valueBase,
        tokenManager.RESERVE_RATIO_BASE()
    );

    function scaleByBases(uint256 sourceValue, uint256 sourceBase, uint256 targetBase)
        internal
        pure
        returns (uint256)
    {
        if (targetBase >= sourceBase) {
            return sourceValue * (targetBase / sourceBase);
        } else {
            return sourceValue / (sourceBase / targetBase);
        }
    }
```

When `targetBase  >= sourceBase`, `allReserves * valueBase / liabilities` will multiply `targetBase / sourceBase`.   
Assume: 
```shell
valueBase  = 1e6
reserveRatioThreshold = 1300000 * 1e12
allReserves  = 12000000000
liabilities = 9230769230
origin reserveRatio =  1300000 * 1e12, equal with threshold, check fail 
real reserveRatio  = 1300000000108333300, greater than threshold, check success
```
In other words, when the price of emerging market currencies rises and the ratio approaches the threshold, the wrong calculation resulting in revert. But in fact, the reserve is sufficient to support the swap of asset.  

## Impact

The user cannot continue to use the protocol

## Code Snippet

- https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/d5328421bea80e3b0fd4595e4eb6b732a40e421e/Unitas-Protocol/src/Unitas.sol#L486-L493
- https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/d5328421bea80e3b0fd4595e4eb6b732a40e421e/Unitas-Protocol/src/utils/ScalingUtils.sol#L54-L64

## Tool used

Manual Review

## Recommendation

Divide by liabilities at the end
```solidity
    reserveRatio = ScalingUtils.scaleByBases(
        allReserves * valueBase,
        valueBase,
        tokenManager.RESERVE_RATIO_BASE()
    ) / liabilities;
```