# Missing important check if `repay` and `redeem` functions in `CompoundV2MigrationBundler.sol` are successful

## Description

In `CompoundV2MigrationBundler.sol` we have `compoundV2Repay()` and `compoundV2Redeem()`:
```solidity
    function compoundV2Repay(address cToken, uint256 amount) external payable protected {
        if (cToken == C_ETH) {
            amount = Math.min(amount, address(this).balance);

            require(amount != 0, ErrorsLib.ZERO_AMOUNT);

            ICEth(C_ETH).repayBorrowBehalf{value: amount}(initiator());
        } else {
            address underlying = ICToken(cToken).underlying();

            if (amount != type(uint256).max) amount = Math.min(amount, ERC20(underlying).balanceOf(address(this)));

            require(amount != 0, ErrorsLib.ZERO_AMOUNT);

            _approveMaxTo(underlying, cToken);

            ICToken(cToken).repayBorrowBehalf(initiator(), amount);
        }
    }
```

```solidity
    function compoundV2Redeem(address cToken, uint256 amount) external payable protected {
        amount = Math.min(amount, ERC20(cToken).balanceOf(address(this)));

        require(amount != 0, ErrorsLib.ZERO_AMOUNT);

        ICToken(cToken).redeem(amount); //@audit - not check return value
    }
```

- **`compoundV2Repay()`**: Repays a borrow position on Compound V2.
- **`compoundV2Redeem()`**: Redeems assets from a cToken position on Compound V2. It calculates the redeemable amount based on the bundler's cToken balance and then calls the `redeem` function of the cToken contract.

The problem here is that `compoundV2Repay()` calls `repayBorrowBehalf` and `compoundV2Redeem()` calls `redeem()`. These functions on failure will not rollback the transaction but return 0 or `Error Code`. 
You can see the docs for [repayBorrowBehalf](https://docs.compound.finance/v2/ctokens/#repay-borrow-behalf) and [redeem](https://docs.compound.finance/v2/ctokens/#redeem).

One small disclaimer is that with `repayBorrowBehalf()` at `C_ETH` the function will revert if the transaction fails:

> - `RETURN`: No return, reverts on error.

## Recommended Mitigation Steps

You should always make sure that these functions will not fail without reverting.
Therefore it is mandatory to check the returned value and revert the transaction if unsuccessful.