Ruhum

high

# USD1 is priced as $1 instead of being pegged to USDT

## Summary
The system treats 1 USD1 = $1 instead of 1 USD1 = 1 USDT which allows arbitrage opportunities. 

## Vulnerability Detail
To swap from one token to another Unitas first get's the price of the quote token and then calculates the swap result. Given that we want to swap 1 USD1 for USDT, we have USDT as the quote token:

```sol
        address priceQuoteToken = _getPriceQuoteToken(tokenIn, tokenOut);
        price = oracle.getLatestPrice(priceQuoteToken);
        _checkPrice(priceQuoteToken, price);

        feeNumerator = isBuy ? pair.buyFee : pair.sellFee;
        feeToken = IERC20Token(priceQuoteToken == tokenIn ? tokenOut : tokenIn);

        SwapRequest memory request;
        request.tokenIn = tokenIn;
        request.tokenOut = tokenOut;
        request.amountType = amountType;
        request.amount = amount;
        request.feeNumerator = feeNumerator;
        request.feeBase = tokenManager.SWAP_FEE_BASE();
        request.feeToken = address(feeToken);
        request.price = price;
        request.priceBase = 10 ** oracle.decimals();
        request.quoteToken = priceQuoteToken;

        (amountIn, amountOut, fee) = _calculateSwapResult(request);
```

Since `amountType == AmountType.In`, it executes `_calculateSwapResultByAmountIn()`:

```sol
            // When tokenOut is feeToken, subtracts the fee after converting the amount
            amountOut = _convert(
                request.tokenIn,
                request.tokenOut,
                amountIn,
                MathUpgradeable.Rounding.Down,
                request.price,
                request.priceBase,
                request.quoteToken
            );
            fee = _getFeeByAmountWithFee(amountOut, request.feeNumerator, request.feeBase);
            amountOut -= fee;
```

Given that the price is 0.99e18, i.e. 1 USDT is worth $0.99, it calculates the amount of USDT we should receive as:

```sol
    function _convertByFromPrice(
        address fromToken,
        address toToken,
        uint256 fromAmount,
        MathUpgradeable.Rounding rounding,
        uint256 price,
        uint256 priceBase
    ) internal view virtual returns (uint256) {
        uint256 fromBase = 10 ** IERC20Metadata(fromToken).decimals();
        uint256 toBase = 10 ** IERC20Metadata(toToken).decimals();

        return fromAmount.mulDiv(price * toBase, priceBase * fromBase, rounding);
    }
```
Given that:
- toBase = 10**6 = 1e6 (USDT has 6 decimals)
- fromBase = 10**18 = 1e18 (USD1 has 18 decimals)
- priceBase = 1e18 
- price = 0.99e18 (1 USDT = $0.99)
- fromAmount = 1e18 (we swap 1 USD1)
we get: $1e18 * 0.99e18 * 1e6 / (1e18 * 1e18) = 0.99e6$

So by redeeming 1 USD1 I only get back 0.99 USDT. The other way around, trading USDT for USD1, would get you 1.01 USD1 for 1 USDT: $1e6 * 1e18 * 1e18 / (0.99e18 * 1e6) = 1.01e18$

The contract values USD1 at exactly $1 while USDT's price is variable. But, in reality, USD1 is not pegged to $1. It's pegged to USDT the only underlying asset.

That allows us to do the following:
- Given that USDT loses its peg slightly, e.g. it goes down to 0.997, which happens quite regularly: https://coinmarketcap.com/currencies/tether/
- we can swap a lot of USDT for USD1, wait for USDT to recover and swap back to USDT
$100,000e6 * 1e18 * 1e18 / (0.997e18 * 1e6) = 1.003009e+23$

With USDT back to $1 we get:
$1.003009e+23 * 1e18 * 1e6 / (1e18 * 1e18) = 100300.9e6$

That's a profit of 300 USDT. The profit is taken from other users of the protocol who deposited USDT to get access to the other stablecoins.

## Impact
An attacker can abuse the price variation of USDT to buy USD1 for cheap.

## Code Snippet
https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/SwapFunctions.sol#L28
https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/SwapFunctions.sol#L63
https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/SwapFunctions.sol#L203-L239
## Tool used

Manual Review

## Recommendation
1 USDT should always be 1 USD1. You treat 1 USD1 as $1 but that's not the case.