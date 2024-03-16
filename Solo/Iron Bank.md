# An attacker can apply grieving attack by preventing users from interacting with `repay` function

## Summary
In `IronBank.sol` we have [`_repay()`](https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/pool/IronBank.sol#L975-L1010)function. This function repays an amount of assets to Iron Bank.
```soldiity
function _repay(DataTypes.Market storage m, address from, address to, address market, uint256 amount)
        internal
        returns (uint256)
    {
        uint256 borrowBalance = _getBorrowBalance(m, to);
        if (amount == type(uint256).max) {
            amount = borrowBalance;
        }

        require(amount <= borrowBalance, "repay too much"); 

        uint256 newUserBorrowBalance;
        uint256 newTotalBorrow;
        unchecked {
            newUserBorrowBalance = borrowBalance - amount;
            // Underflow not possible: amount <= userBorrow <= totalBorrow
            newTotalBorrow = m.totalBorrow - amount;
        }

        // Update storage.
        m.userBorrows[to].borrowBalance = newUserBorrowBalance;
        m.userBorrows[to].borrowIndex = m.borrowIndex;
        m.totalCash += amount;
        m.totalBorrow = newTotalBorrow;

        // Check if need to exit the market.
        if (m.userSupplies[to] == 0 && newUserBorrowBalance == 0) {
            _exitMarket(market, to);
        }

        IERC20(market).safeTransferFrom(from, address(this), amount);

        emit Repay(market, from, to, amount, newUserBorrowBalance, newTotalBorrow);

        return amount;
    }
```
The function is internal and is called in [repay()](https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/pool/IronBank.sol#L460C5-L470) and [liquidate()](https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/pool/IronBank.sol#L480-L510).

## Vulnerability Detail
The problem here is whenever a user is going to use `_repay()` function, malicious user (for example Bob) can apply a grieving attack by preventing users from interacting with it. 
Bob could prevent the user from paying her debt fully by just repaying a very small amount of the user's debt in advance and as a result honest user transaction not pass.
```solidity
984: require(amount <= borrowBalance, "repay too much");
```
For example:
- Alice wants to use repay function with full `borrowBalance` 100 tokens and pass `amount=100`.
- Bob observes mempols and do font-run grieving attack with 1 wei.
- Alice the transaction could not go through because of require check:
- `require(amount <= borrowBalance, "repay too much")`
- 
Bob can apply this attack for all other users who are going to repay their debt fully.

## Impact

A malicious user can prevent an honest user from using a key function.

## Code Snippet
https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/pool/IronBank.sol#L975-L1010
https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/pool/IronBank.sol#L460C5-L470
https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/pool/IronBank.sol#L480-L510

## Tool used

Manual Review

## Recommendation

Instead of this check:
```solidity
        if (amount == type(uint256).max) {
            amount = borrowBalance;
        }
```

Change it like this:
```solidity
        if (amount > borrowBalance) {
            amount = borrowBalance;
        }
```
***

# `liquidationBonus` with a minimum value of `MIN_LIQUIDATION_BONUS` will always revert

## Summary

`liquidationBonus` with a minimum value of 100% will always revert

## Vulnerability Detail

In `MarketConfigurator.sol` we have `configureMarketAsCollateral` and `adjustMarketLiquidationBonus()` functions.  The first function allows the owner to configure a market as collateral and the second is to adjust the liquidation bonus of a market.
The problem here is that both functions will revert when the minimum allowed value is set. 
The minimum allowed value is in `Constants.sol`:

```solidity
12: uint16 internal constant MIN_LIQUIDATION_BONUS = 10000; // 100%
```
Both Functions have the following check:

```solidity
require(
            liquidationBonus > MIN_LIQUIDATION_BONUS && liquidationBonus <= MAX_LIQUIDATION_BONUS,  
            "invalid liquidation bonus"
        );
```
Оmitted is to add the sign greater than or equal to.

As you can see the function will not allow to set the `liquidationBonus` value to 100% (that is 10000).

## Impact

It is impossible to settle a liquidation bonus of 100%

## Code Snippet
https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/pool/MarketConfigurator.sol#L141
https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/pool/MarketConfigurator.sol#L238
https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/pool/Constants.sol#L12

## Tool used

Manual Review

## Recommendation

Change:
```solidity
liquidationBonus > MIN_LIQUIDATION_BONUS && liquidationBonus <= MAX_LIQUIDATION_BONUS
```

To:
```solidity
liquidationBonus >= MIN_LIQUIDATION_BONUS && liquidationBonus <= MAX_LIQUIDATION_BONUS
```
***

# `redeemNativeToken()` missing to call `accrueInterest` before set `redeemAmount`

## Summary
In `TxBuilderExtension.sol` we have [`redeemNativeToken()`](https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/extensions/TxBuilderExtension.sol#L275-L283) function:
```solidity
    function redeemNativeToken(address user, uint256 redeemAmount) internal nonReentrant {
        if (redeemAmount == type(uint256).max) {
            redeemAmount = ironBank.getSupplyBalance(user, weth);
        }
        ironBank.redeem(user, address(this), weth, redeemAmount);
        WethInterface(weth).withdraw(redeemAmount);
        (bool sent,) = user.call{value: redeemAmount}("");
        require(sent, "failed to send native token");
    }
```
This function redeems the wrapped native token and unwraps it to the user.

## Vulnerability Detail

The problem here if `redeemAmount == type(uint256).max` the function should call 
```solidity
ironBank.accrueInterest(weth)
```

in order to calculate `redeemAmount` correctly.

And then call:
```solidity
277: redeemAmount = ironBank.getSupplyBalance(user, weth);
```

By conditionally calling `accrueInterest` when `redeemAmount` is maximum, the function guarantees that the redemption amount accurately reflects the current supply balance, including any accrued interest.

You can see how you did it in `redeemStEth` and `redeemPToken`:
```solidity
        if (amount == type(uint256).max) {
            ironBank.accrueInterest(pToken);
            amount = ironBank.getSupplyBalance(user, pToken);
        }
```
## Impact

Possible loss of funds due to the wrong `redeemAmount`

## Code Snippet
https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/extensions/TxBuilderExtension.sol#L275-L283

## Tool used

Manual Review

## Recommendation

To ensure the redemption amount reflects the most up-to-date accrued interest, it is generally advisable to include an `accrueInterest` step before retrieving the supply balance. This allows for an accurate calculation of the user's current supply balance, considering any interest that has accumulated.

```solidity
        if (redeemAmount == type(uint256).max) {
	        ironBank.accrueInterest(weth);
            redeemAmount = ironBank.getSupplyBalance(user, weth);
        }
```
***

# Missing check for active L2 Sequencer in `PriceOracle.sol`

## Summary

In Q&A we see:

>On what chains are the smart contracts going to be deployed?
>
>  - mainnet, Arbitrum, Optimism

You should always check for sequencer availability when using Chainlink's Arbitrum or Optimism price feeds or another layer 2 chains. 

## Vulnerability Detail

Optimistic rollup protocols move all execution off the layer 1 (L1) Ethereum chain, complete execution on a layer 2 (L2) chain, and return the results of the L2 execution back to the L1. These protocols have a sequencer that executes and rolls up the L2 transactions by batching multiple transactions into a single transaction.

If a sequencer becomes unavailable, it is impossible to access read/write APIs that consumers are using and applications on the L2 network will be down for most users without interacting directly through the L1 optimistic rollup contracts. The L2 has not stopped, but it would be unfair to continue providing service on your applications when only a few users can use them.

## Impact

If the Arbitrum Sequencer goes down, oracle data will not be kept up to date, and thus could become stale.

## Code Snippet

https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/oracle/PriceOracle.sol#L12

## Tool used

Manual Review

## Recommendation

Check this example -> https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code
***

# Price Oracle could get a stale price

## Summary

No check for round completeness could lead to stale prices and wrong price return value, or outdated price. The functions that rely on accurate price feed might not work as expected, which sometimes can lead to fund loss.

## Vulnerability Detail

The oracle wrapper [`getPriceFromChainlink()`](https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/oracle/PriceOracle.sol#L66-L72) call out to an oracle with `latestRoundData()` to get the price of some token. Although the returned timestamp is checked, there is no check for round completeness.

According to Chainlink's [documentation](https://docs.chain.link/data-feeds/price-feeds/historical-data), this function does not error if no answer has been reached but returns 0 or outdated round data. The external Chainlink oracle, which provides index price information to the system, introduces risk inherent to any dependency on third-party data sources. For example, the oracle could fall behind or otherwise fail to be maintained, resulting in outdated data being fed to the index price calculations. Oracle reliance has historically resulted in crippled on-chain systems, and complications that lead to these outcomes can arise from things as simple as network congestion.

The same problem is in `_setAggregators()` function in `PriceOracle.sol`

## Impact

If there is a problem with chainlink starting a new round and finding consensus on the new value for the oracle (e.g. chainlink nodes abandon the oracle, chain congestion, vulnerability/attacks on the chainlink system) consumers of this contract may continue using outdated stale data (if oracles are unable to submit no new round is started).

This could lead to stale prices and wrong price return value, or outdated price.

As a result, the functions that rely on accurate price feed might not work as expected, which sometimes can lead to fund loss. The impacts vary and depend on the specific situation like the following:

- incorrect liquidation
- some users could be liquidated when they should not
- no liquidation is performed when there should be
- wrong price feed
 

## Code Snippet

https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/oracle/PriceOracle.sol#L66-L72

## Tool used

Manual Review

## Recommendation

Validate data feed for round completeness:

```solidity
     (uint80 roundID, int256 price, , uint256 timestamp, uint80 answeredInRound) = registry.latestRoundData(base, quote);
		
	require(answeredInRound >= roundID, "round not complete");
```
***

# Oracle will return the wrong price for asset if underlying aggregator hits `minAnswer`

## Summary
`getPriceFromChainlinkwill()` and `_setAggregators()` will returns the wrong price for asset if underlying aggregator hits `minAnswer `

## Vulnerability Detail

Chainlink aggregators have a built in circuit breaker if the price of an asset goes outside of a predetermined price band. The result is that if an asset experiences a huge drop in value (i.e. LUNA crash) the price of the oracle will continue to return the minPrice instead of the actual price of the asset. This would allow user to continue borrowing with the asset but at the wrong price. This is exactly what happened to [Venus on BSC when LUNA imploded](https://rekt.news/venus-blizz-rekt/).

When `latestRoundData()` is called it request data from the aggregator. The aggregator has a minPrice and a maxPrice. If the price falls below the minPrice instead of reverting it will just return the min price.

## Impact

In the event that an asset crashes the protocol can be manipulated to give out loans at an inflated price

## Code Snippet
https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/oracle/PriceOracle.sol#L66-L72
https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/oracle/PriceOracle.sol#L107

## Tool used

Manual Review

## Recommendation

`getPriceFromChainlink()` and `_setAggregators()` should check the returned answer against the minPrice/maxPrice and revert if the answer is outside of the bounds:

```
if (price >= maxPrice or price <= minPrice) revert();
```
***

# It is possible for a user to completely redeem an amount of asset from Iron Bank, but not leave the market

## Summary
It is possible for a user to completely Redeem an amount of asset from Iron Bank, but not leave the market. This situation is happening in [redeem()](https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/pool/IronBank.sol#L406-L451) function:

```solidity
function redeem(address from, address to, address market, uint256 amount)
        external
        nonReentrant
        isAuthorized(from)
    {
        DataTypes.Market storage m = markets[market];
        require(m.config.isListed, "not listed");

        _accrueInterest(market, m);

        uint256 userSupply = m.userSupplies[from];
        uint256 totalCash = m.totalCash;
  
        uint256 ibTokenAmount;
        bool isRedeemFull;
        if (amount == type(uint256).max) {  
            ibTokenAmount = userSupply;
            amount = (ibTokenAmount * _getExchangeRate(m)) / 1e18;
            isRedeemFull = true;
        } else {
            ibTokenAmount = (amount * 1e18) / _getExchangeRate(m);
        }
  
        require(userSupply >= ibTokenAmount, "insufficient balance");
        require(totalCash >= amount, "insufficient cash");
  
        // Update storage.
        unchecked {
            m.userSupplies[from] = userSupply - ibTokenAmount;
            m.totalCash = totalCash - amount;
            // Underflow not possible: ibTokenAmount <= userSupply <= totalSupply.
            m.totalSupply -= ibTokenAmount;
        }
  
        // Check if need to exit the market.
        if (isRedeemFull && _getBorrowBalance(m, from) == 0) {
            _exitMarket(market, from);
        }  

        IBTokenInterface(m.config.ibTokenAddress).burn(from, ibTokenAmount); // Only emits Transfer event.
        IERC20(market).safeTransfer(to, amount);

        _checkAccountLiquidity(from);

        emit Redeem(market, from, to, amount, ibTokenAmount);
    }
```

## Vulnerability Detail

Consider the following situation:
- Alice wants to redeem a full amount of an asset from the Iron Bank and exit the market. 
- To redeem the full amount she needs to put 100 amount of tokens.
- After Alice does this we go to the following code:
```solidity
420:        bool isRedeemFull;
421:        if (amount == type(uint256).max) {  
422:            ibTokenAmount = userSupply;
423:            amount = (ibTokenAmount * _getExchangeRate(m)) / 1e18;
424:            isRedeemFull = true;
425:        } else {
426:            ibTokenAmount = (amount * 1e18) / _getExchangeRate(m);
427:        }
```
- Alice actually buys all her tokens (100) and wants to get out of the market, but doesn't actually enter the `if` statement.
- The condition `amount == type(uint256).max` checks whether the `amount` parameter is equal to `type(uint256).max`, which is the maximum value representable by a `uint256` variable, and Alice enters the `if` statement.
- Because she does not enter into the `if` statement, `isRedeemFull` is never set to `true` and stays `false`.
- Then we go to the following code in which Alice should exit the market but actually does not pass the check because isRedeemFull remains `false`
```solidity
440:        // Check if need to exit the market.
441:        if (isRedeemFull && _getBorrowBalance(m, from) == 0) {
442:            _exitMarket(market, from);
443:        }
```
## Impact

If the user not entering the `_exitMarket()` function means that the user's account will still be considered as having an open position in the market. The outstanding borrow balance will remain, and the user will continue to accrue interest on their borrowings.

## Code Snippet
https://github.com/sherlock-audit/2023-05-ironbank-cvetanovv/blob/main/ib-v2/src/protocol/pool/IronBank.sol#L406-L451

## Tool used

Manual Review

## Recommendation

In the `if` statement, check how many tokens a user can have, not the maximum number. like this `if` statement is executed and when a user wants to redeem everything `isRedeemFull` will be set to true.
