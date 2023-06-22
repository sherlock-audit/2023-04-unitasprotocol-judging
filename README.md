# Issue M-1: In case the portfolio makes a loss, the total reserves and reserve ratio will be inflated. 

Source: https://github.com/sherlock-audit/2023-04-unitasprotocol-judging/issues/16 

## Found by 
ast3ros
## Summary  

The pool balance is transferred to the portfolio for investment, for example sending USDT to Curve/Aave/Balancer etc. to generate yield. However, there are risks associated with those protocols such as smart contract risks. In case a loss happens, it will not be reflected in the pool balance and the total reserve and reserve ratio will be inflated.

## Vulnerability Detail

The assets in the pool can be sent to the portfolio account to invest and earn yield. The amount of assets in the insurance pool and Unitas pool is tracked by the `_balance` variable. This amount is used to calculate the total reserve and total collateral, which then are used to calculate the reserve ratio.

            uint256 tokenReserve = _getBalance(token);
            uint256 tokenCollateral = IInsurancePool(insurancePool).getCollateral(token);

When there is a loss to the portfolio, there is no way to write down the `_balance` variable. This leads to an overstatement of the total reserve and reserve ratio.

## Impact

Overstatement of the total reserve and reserve ratio can increase the risk for the protocol because of undercollateralization of assets.

## Code Snippet

https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/Unitas.sol#L508-L509
https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/PoolBalances.sol#L111-L113

## Tool used

Manual Review

## Recommendation

Add function to allow admin to write off the `_balance` in case of investment lost. Example:

```javascript

function writeOff(address token, uint256 amount) external onlyGuardian {

    uint256 currentBalance = IERC20(token).balanceOf(address(this));

    // Require that the amount to write off is less than or equal to the current balance
    require(amount <= currentBalance, "Amount exceeds balance");
    _balance[token] -= amount;

    emit WriteOff(token, amount);
}

```



## Discussion

**Adityaxrex**

Aditya: This will need to be manually updated. The likelyhood is small but can be contained by diversification and risk management.

**SunXrex**

in phase 1, reserve ratio will not impact by portfolio impact

**ctf-sec**

> This will need to be manually updated. The likelyhood is small but can be contained by diversification and risk management.

Intend to maintain the medium severity

**0xffff11**

I agree with a medium. Despite being very unlikely, the risk exists. 

# Issue M-2: Unitas swap function is vulnerable to Sandwich Attack at oracle price update 

Source: https://github.com/sherlock-audit/2023-04-unitasprotocol-judging/issues/67 

## Found by 
DevABDee, Dug, n33k, qpzm, sashik\_eth, shogoki, vagrant

## Summary

WHen the oracle price is updates, an attacker can sandwich the update adderss with 2 swap transactions to gain a profit and drain the collateral.

## Vulnerability Detail

Inside the `swap` function of the `Unitas` contract the `getLatestPrice` function of the  `XOracle` contract is used to fetch the current price of the USDx token to be swapped to/from.
The price has to be updated periodically because the currencies fluctuate against the USD price.
A user could use these fluctuations to speculate on prices and gain a profit or loss.

However,as the price for the oracle is updated by a transaction that is publicly visible inside the mempool, a malicious user or attacker can see the new price before it is active. As the oracle price is the only thing, which has influence of the token number to mint/burn on a swap call, an attacker can easily exploit a temporarily appreciation of an USDx token agains the USD1 (US Dollar price). 
This can be achieved by "sandwiching" the price update transaction with a transaction to swap USD1 into the relevant USDx token first, and swap it back after the price update.

Example: 
THe price of USD91 (INR) is to be increased from 0.012 to 0.013
Attacker already holds 10,000 of USD1 tokens.

- The Feeder sends the transaction to update the oracle price, and it gets placed in the mempool.
- Attacker sees these transaction, and sends himself 2 transactions.
1. Swap 10,000 possible mount of USD1 to USD91
2. Swap all USD91 to USD1
- The attacker sets the gas to ensure that the first tx gets included before the price update, and the second one after the price update. 
- The executed Transactions in order will be:
1. Attacker swaps 10,000 USD1 to USD91 (should receive: 10,000 / 0.012 = 833,333.333)
2. FEEDER updates oracle price of USD91 to 0.013
3. Attacker swaps all USD91 tokens to USD1 (should receive: 833,333.333 * 0.013 = 10,833.333)

The attacker just made 833 USD1 profit in these 2 transactions, and can redeem the USD1 tokens for USDT, which will deduct the collateral by this amount. 
 

## Impact

Attacker can gain profit and "steal" collateral

## Code Snippet

https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/Unitas.sol#L439

https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/XOracle.sol#L26-L32

## Tool used

Manual Review

## Recommendation

The 2 way minting/burning mechanism by the orcale price might be dangerous.
However to prevent the specific attack vector, maybe the minting can be paused and unpaused before and after the price update. (in separate transactions) 



## Discussion

**SunXrex**

Sun & Lucas: To update prices, you can consider using an RPC (Remote Procedure Call) that helps prevent MEV (Miner Extractable Value). 

Risk should be **medium**.

Usually, updates to the exchange rate for stablecoins result in less than a 1~1.6% change. Additionally, we apply a fee for minting and burning. This makes arbitrage difficult. To provide a perspective from the auditors' case, an increase from 0.012 to 0.013 represents an approximate growth of 8.33% which is almost impossible.

Aditya: Price update frequency is much lower for Unitas compared to other defi swaps like Uniswaps. Unitas update frequency could be as low as once per day. Also the expected users/minters of the protocol are low frequency but high ticket size. Chances of users mint request being in the mempool and Unitas price update happening at the same time given are fairly low. Also, unlike Uniswap any new trade does not move the price. The price is only updated at certain periods of the day or once a day.

**ctf-sec**

Can be a valid medium

**0xffff11**

I agree with a medium. Due to the current architecture of having a fee and the stability of the assets, the value it is quite limited. 

**hrishibhat**

Considering this a medium issue based on the above comments.

# Issue M-3: No slippage or deadline control for swapping while stability burning 

Source: https://github.com/sherlock-audit/2023-04-unitasprotocol-judging/issues/88 

## Found by 
0x4non, 0xGoodess, Jiamin, Juntao, PawelK, circlelooper, ctf\_sec, moneyversed, okolicodes, radev\_sw, stopthecap, tsvetanovv
## Summary
No slippage or deadline control for swapping while stability burning

## Vulnerability Detail

Even though Unitas claims there will not be slippage because if it is burned from one side, it is minted in the other one, there is an edge-case where it does create a slight slippage depending on the burn amount.  Unitas intends to make stability burns when the reserve ratio is below 130% to try and get it back to normal levels. This burn, is a one sided burn of `usd1`, which reduces the `totalSupply` of `usd1` unilaterally.
https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/Unitas.sol#L208

If any user is trying to swap `usd1` while there is a stability burn being conducted, they will be affected by that slippage.

The other recommendation is the usage of a deadline param. Without a deadline parameter, the transaction may sit in the mempool and be executed at a much later time potentially resulting in a swap after/before a stability burn.

## Impact
User will be affected by unintended and unhandled slippage, potentially affecting the funds they get back from the swap

## Code Snippet
https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/Unitas.sol#L208

## Tool used

Manual Review

## Recommendation
Allow a user to specify to key parameters, a `deadline` and a `minOutAmount`. And make a check for both at start and end of the execution in the swap function.



## Discussion

**SunXrex**

Sun: In this phase we don’t support stability burn.

Aditya: Slippage on a swap DEX like Uniswap is part of the design. This is to ensure that the pool does not get fully empty. So if the user places a large order, they see a slippage in price. Every order on a DEX like Uniswap, moves the price. Large orders move the price in higher degree. As the order grows, the slippage also grows. This is inherent property of xy=k bonding curve on which Uniswap operate.

In case of Unitas, every swap order is essentially mint/burn order. As long as there is enough USDT in reserve and insurance pool, protocol can mint and burn any amount. This leads to 1:1 value transfer without slippage. Unitas does not mint on a bondig curve, instead the protocol simply mints new tokens or burns the existing.
Eg: On uniswap if I buy 1 ETH, I will face low slippage but if I buy 1000 ETH, the slippage will be much higher. However, in case of Unitas whether the user mints 1000 USD91 or 1million USD91, the user will get all the tokens at Oracle price.

In case of Uniswap, the DEX does not have the authority to mint/burn but only facilitate a swap. In case of Unitas, the protocol has the authority to mint and burn. This leads to no slippage for minters/burners.

**Adityaxrex**

This style of slippage is low probability for our design. We may fix it in the future but edge case scenario for now. 

**SunXrex**

It should be considered a low-medium risk because the chances of encountering it are very low.

**ctf-sec**

> This style of slippage is low probability for our design. We may fix it in the future but edge case scenario for now.

medium

**hrishibhat**

Considering this issue as a medium based on the above comments

# Issue M-4: No clear threshold on when the oracle is updated will cause stale prices to be accepted 

Source: https://github.com/sherlock-audit/2023-04-unitasprotocol-judging/issues/150 

## Found by 
0xGoodess, 0xJuda, Juntao, Norah, PRAISE, PawelK, carrotsmuggler, ctf\_sec, kutugu, mau, stopthecap, thekmj, toshii
## Summary
No clear threshold on when the oracle is updated will cause stale prices to be accepted

## Vulnerability Detail

According to the team, the oracle is updated depending on the gas fees. 

"" the oracle price update is limited by cost of gas fee during updating. we want to have a balance between reaching the true price vs saving the cost. Once we move to cheaper chains like arbitrum or matic, we will update the oracle a lot more frequently
just fyi ""

Not having a clear point in time when the oracle will be updated, will cause that in times when the network is very expensive to include transactions, stale prices of assets/stables will be accepted as the current price, causing wrong/stale prices be fetched as if they were the latest.

## Impact
Wrong/stale prices will be used in times when gas fees are very expensive

## Code Snippet8

https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/d5328421bea80e3b0fd4595e4eb6b732a40e421e/Unitas-Protocol/src/XOracle.sol#L34-L46

## Tool used

Manual Review

## Recommendation
Add a clear threshold when the prices should be updated (ex. each 2 minutes), potentially not using the gas fees as an indicator to when updating the oracle.

