ast3ros

medium

# Profit of the portfolio are not transferred back to the pool balance.

## Summary

The pool balance is transferred to the portfolio for investment. In case the portfolio makes a profit, the excess amount cannot be transferred back to the pool.

## Vulnerability Detail

When the portfolio returns the assets back to the pool, it calls the function `receivePortfolio`.

        function receivePortfolio(address token, uint256 amount)
            external
            onlyPortfolio(msg.sender)
            nonReentrant
        {
            _receivePortfolio(token, msg.sender, amount);
        }

https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/Unitas.sol#L244-L250
https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/InsurancePool.sol#L116-L122

The function will transfer the asset `amount` to the pool contract. However, if the portfolio gains profit, it cannot return back the profit to the pool because the pool only allows it to return the original amount.

        _require(amount <= portfolio, Errors.AMOUNT_INVALID);

https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/PoolBalances.sol#L71

## Impact

The profit amount of the investment is stuck in the portfolio and cannot be returned to the pool. This leads to an underestimation of the total reserves and reserve ratio, which can affect the operation of the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/Unitas.sol#L244-L250
https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/InsurancePool.sol#L116-L122
https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/PoolBalances.sol#L71

## Tool used

Manual Review

## Recommendation

Account for profit of the portfolio and allow the portfolio role to return the profit.