0x00ffDa

medium

# Token pair removal does not settle existing user funds

## Summary
When TokenManager removes a token pair via `removeTokensAndPairs()`, any existing user funds are not automatically settled to USD1 nor an underlying asset. Since swaps of that pair are no longer unsupported by Unitas, user funds could be locked in an abandoned token contract, or only be recoverable via 3rd party market at a reduced value due to stablecoin losing its peg.

## Vulnerability Detail
If circumstances ever require removing support for any Emerging Market Currency (EMC) OR upgrading / replacing any deployed Unitas EMC stablecoin (USDEMC) `ERC20Token` contract, the associated USD1-USDEMC token pair would be removed to end trading of that USDEMC stablecoin and as a [necessary step for possible token removal](https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/TokenManager.sol#L266).

Unitas has no logic to force settlement USDEMC => USD1 (and possibly USD1 => USDT) during the process of [removing a token pair](https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/TokenPairs.sol#LL128C1-L146C1).
In the event a USD1-USDEMC pair is removed, swaps of that pair become impossible within Unitas. This means that the arbitragers that maintain the USDEMC stablecoin's price peg to the underlying EMC cannot operate and the USDEMC token would lose its price peg.

Note that upgrading / replacing the USD1 token contract via `setUSD1()` would have the same issue. It would require first removing all existing token pairs and all funds in the USD1 contract would be jeopardized in the same way described above.

## Impact

In the event of USD1-USDEMC token pair removal, user funds in the affected USDEMC stablecoin are jeopardized because they can no longer be swapped via Unitas back to UDS1 nor the underlying asset. Removing the market that provided (via arbitrage) the stablecoin's peg to its EMC will result in lower valuation in any 3rd party DEX/CEX/OTC markets. If 3rd party market avenues no longer exist, the user funds will be effectively lost. 

While this scenario would lead to loss of user funds, it may not be a high severity because it is contingent upon an unforeseen (but possible) governance action:  either upgrading / replacing any Unitas `ERC20Token` deployment, or removing support for any currency.

## Code Snippet
The logic for removing a token pair is in [TokenPair._removePairByTokens()](https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/TokenPairs.sol#LL128C1-L146C1), removing the pair from storage without handling existing funds:
```javascript
    /**
     * @notice Removes the pair from the pool, reverts if the pair doesn't exist.
     * @param tokenX The smaller token address
     * @param tokenY The larger token address
     */
    function _removePairByTokens(address tokenX, address tokenY) internal virtual returns (bytes32) {
        bytes32 pairHash = _getPairHash(tokenX, tokenY);

        _require(_pairHashes.remove(pairHash), Errors.PAIR_NOT_EXISTS);
        _require(
            _pairTokens[tokenX].remove(tokenY) && _pairTokens[tokenY].remove(tokenX),
            Errors.PAIR_NOT_EXISTS
        );

        emit PairRemoved(pairHash, tokenX, tokenY);

        return pairHash;
    }
```


## Tool used

Manual Review

## Recommendation
Add automatic settling of all funds in any token being removed or any token for which Unitas swaps will become impossible when a pair is removed. This would be similar to (but a limited application of) the planned process described in the whitepaper for emergency universal settlement upon asset reserve level dropping to 100%.