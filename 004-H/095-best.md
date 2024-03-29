Juntao

high

# Users may not be able to fully redeem USD1 into USDT even when reserve ratio is above 100%

## Summary

Users may not be able to fully redeem USDT even when reserve ratio is above 100%, because of portfolio being taken into the account for calculation.

## Vulnerability Detail

Reserve ratio shows how many liabilities is covered by reserves, a reserve ratio above 100% guarantees protocol has enough USDT to redeem, the way of calculating reserve ratio is `Reserve Ratio = allReserves / liabilities` and is implemented in [Unitas#_getReserveStatus(...)](https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/Unitas.sol#L476-L495) function:
```solidity
            reserveRatio = ScalingUtils.scaleByBases(
                allReserves * valueBase / liabilities,
                valueBase,
                tokenManager.RESERVE_RATIO_BASE()
            );
```

`allReserves` is the sum of the [balance](https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/PoolBalances.sol#L22) of Unitas and InsurancePool,  calculated in [Unitas#_getTotalReservesAndCollaterals()](https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/Unitas.sol#L500-L535) function:
```solidity
        for (uint256 i; i < tokenCount; i++) {
            address token = tokenManager.tokenByIndex(tokenTypeValue, i);
            uint256 tokenReserve = _getBalance(token);
            uint256 tokenCollateral = IInsurancePool(insurancePool).getCollateral(token);


            if (tokenReserve > 0 || tokenCollateral > 0) {
                uint256 price = oracle.getLatestPrice(token);


                reserves += _convert(
                    token,
                    baseToken,
                    tokenReserve,
                    MathUpgradeable.Rounding.Down,
                    price,
                    priceBase,
                    token
                );


                collaterals += _convert(
                    token,
                    baseToken,
                    tokenCollateral,
                    MathUpgradeable.Rounding.Down,
                    price,
                    priceBase,
                    token
                );
            }
        }
```

`liabilities` is the total value of USD1 and USDEMC tokens, calculated in [Unitas#_getTotalLiabilities()](https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/Unitas.sol#L540-L567) function:
```solidity
        for (uint256 i; i < tokenCount; i++) {
            address token = tokenManager.tokenByIndex(tokenTypeValue, i);
            uint256 tokenSupply = IERC20Token(token).totalSupply();


            if (token == baseToken) {
                // Adds up directly when the token is USD1
                liabilities += tokenSupply;
            } else if (tokenSupply > 0) {
                uint256 price = oracle.getLatestPrice(token);


                liabilities += _convert(
                    token,
                    baseToken,
                    tokenSupply,
                    MathUpgradeable.Rounding.Down,
                    price,
                    priceBase,
                    token
                );
            }
        }
```

Some amount of USDT in both Unitas and InsurancePool is [portfolio](https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/PoolBalances.sol#L26), which represents the current amount of assets used for strategic investments, it is worth noting that after [sending portfolio](https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/PoolBalances.sol#L83-L99), `balance` remains the same, which means `portfolio` is taken into account in the calculation of reserve ratio.

This is problematic because `portfolio` is not available when user redeems, and user may not be able to fully redeem for USDT even when protocols says there is sufficient reserve ratio. 

Let's assume :

> Unitas's balance is 10000 USD and its portfolio is 2000 USD, avaliable balance is 8000 USD
> InsurancePool's balance is 3000 USD and its portfolio is 600 USD, available balance is 2400 USD
> AllReserves value is 13000 USD
> Liabilities (USDEMC) value is 10000 USD
> Reserve Ratio is (10000 + 3000) /  10000 = 130%.

Later on, USDEMC appreciates upto 10% and we can get:
> AllReserves value is still 13000 USD
> Liabilities (USDEMC) value is 11000 USD
> Reserve Ratio is (10000 + 3000) /  11000 = 118%.

The available balance in Unitas is 8000 USD so there is 3000 USD in short, it needs to be obtain from InsurancePool, however,  the available balance in InsurancePool is 2400 USD, transaction will be reverted and users cannot redeem.

There would also be an extreme situation when reserve ratio is above 100% but there is no available balance in protocol because all the `balance` is `portfolio` (this is possible when InsurancePool is drained out), users cannot redeem any USDT in this case.

## Impact

Users may not be able to fully redeem USD1 into USDT even when reserve ratio is above 100%, this defeats the purpose of reserve ratio and breaks the promise of the protocol, users may be mislead and lose funds.

## Code Snippet

https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/Unitas.sol#L500-L535

## Tool used

Manual Review

## Recommendation

Portfolio should not be taken into account for the calculation of reserve ratio.
```diff
    function _getTotalReservesAndCollaterals() internal view returns (uint256 reserves, uint256 collaterals) {
        ...
-           uint256 tokenReserve = _getBalance(token);
+           uint256 tokenReserve = _getBalance(token) - _getPortfolio(token);
-           uint256 tokenCollateral = IInsurancePool(insurancePool).getCollateral(token);
+           uint256 tokenCollateral = IInsurancePool(insurancePool).getCollateral(token) - IInsurancePool(insurancePool).getPortfolio(token);
        ...
    }
```