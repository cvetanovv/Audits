# Do not hardcode `swapRouter()`  address

## Summary

In `Fees.sol` we use `swapRouter` address which is hardcoded:

```solidity
ISwapRouter public constant swapRouter =
        ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);
```

The problem is that this address may not be valid for all chains that will be deployed.
## Vulnerability Details

From the discord server, we have information that the protocol will be debugged at different times.

Hardcoding the address of  `ISwapRouter` can lead to issues, especially when deploying the contract across different chains. Different chains may have different addresses and hardcoding an address may limit the contract's portability.

Is better to provide the address as a constructor parameter. This way, when the contract is deployed, the address of the dependency can be specified, allowing the contract to be used across different chains.
## Impact

`swapRouter` maybe not work on some chains
## Tools Used

Visual Studio Code
## Recommendations

Set `swapRouter` address in the constructor on every chain.
***

# The protocol doesn't have support for fee on transfer type of ERC20 tokens

## Summary

The protocol doesn't have support for fee on transfer type of ERC20 tokens
## Vulnerability Details

In the following places in `Lender.sol` we see this problem:

```solidity
152:             IERC20(p.loanToken).transferFrom(

159:             IERC20(p.loanToken).transfer(

187:         IERC20(pools[poolId].loanToken).transferFrom(

203:         IERC20(pools[poolId].loanToken).transfer(msg.sender, amount);

267:             IERC20(loan.loanToken).transfer(feeReceiver, fees);

269:             IERC20(loan.loanToken).transfer(msg.sender, debt - fees);

271:             IERC20(loan.collateralToken).transferFrom(

317:             IERC20(loan.loanToken).transferFrom(

323:             IERC20(loan.loanToken).transferFrom(

329:             IERC20(loan.collateralToken).transfer(

403:             IERC20(loan.loanToken).transfer(feeReceiver, protocolInterest);

505:         IERC20(loan.loanToken).transfer(feeReceiver, protocolInterest);

563:             IERC20(loan.collateralToken).transfer(feeReceiver, govFee);

565:             IERC20(loan.collateralToken).transfer(

642:                 IERC20(loan.loanToken).transferFrom(

651:                 IERC20(loan.loanToken).transfer(feeReceiver, fee);

653:                 IERC20(loan.loanToken).transfer(msg.sender, debt - debtToPay - fee);

656:             IERC20(loan.loanToken).transfer(feeReceiver, protocolInterest);

663:                 IERC20(loan.collateralToken).transferFrom(

670:                 IERC20(loan.collateralToken).transfer(
```
## Impact

Some ERC20 token implementations have a fee that is charged on each token transfer. This means that the transferred amount isn't exactly what the receiver will get.
## Tools Used

Visual Studio Code
## Recommendations

Improve support for a fee on transfer type of ERC20. When pulling funds from the user using `transferFrom()` and `transfer()` the usual approach is to compare balances pre/post transfer.
***

# `ExactInputSingleParams()` don't have slippage protection

## Summary

In `Fees.sol` we have `sellProfits()` function. This function swap loan tokens for collateral tokens from liquidations:
```solidity
function sellProfits(address _profits) public {
        require(_profits != WETH, "not allowed");
        uint256 amount = IERC20(_profits).balanceOf(address(this));
  
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0,  //@audit - no slippage
                sqrtPriceLimitX96: 0
            });

        amount = swapRouter.exactInputSingle(params);
        IERC20(WETH).transfer(staking, IERC20(WETH).balanceOf(address(this)));
    }
```

This function calls `ExactInputSingleParams()` from ISwapRouter without any slippage protection.
## Vulnerability Details

The functions `ExactInputSingleParams()`, don't have slippage protection:
```solidity
ExactInputSingleParams({
                tokenIn: _profits,
                tokenOut: WETH,
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: amount,
                amountOutMinimum: 0,  //@audit - no slippage
                sqrtPriceLimitX96: 0
            });
```

We can see  `amountOutMinimum` is hardcoded to 0. `amountOutMinimum` is used to prevent high slippage. By setting them to a value greater than zero, you would ensure that the transaction reverts if the amount of tokens that will be added to the liquidity pool is less than these minimums. 

In a volatile market, or when dealing with large orders, the price can shift while the transaction is being mined, and the actual amount of tokens added can be less than the desired amount.
## Impact

Without slippage, If the price of the tokens changes significantly during the swap, it could result in a large slippage, causing users to lose a significant amount of funds.
An attacker can watch the mempool and then (using flash bots) execute a sandwich attack to manipulate the price before and after the swap.

## Tools Used

Visual Studio Code

## Recommendations

Do not hardcode `amountOutMin` to 0, but let the user choose the value.
***

# A Malicious Lender can increase the interest rate to a maximum and screw the borrower

## Summary

A Malicious Lender can increase the interest rate to a maximum and screw the borrower. This can be done through the function `updateInterestRate()`.

## Vulnerability Details

In `Lender.sol` we have `updateInterestRate()`:

```solidity
function updateInterestRate(bytes32 poolId, uint256 interestRate) external {
        if (pools[poolId].lender != msg.sender) revert Unauthorized();
        if (interestRate > MAX_INTEREST_RATE) revert PoolConfig();
        pools[poolId].interestRate = interestRate;    
        emit PoolInterestRateUpdated(poolId, interestRate);
    }
```

This function updates the interest rate for a pool and can only be called by the pool lender. 

Imagine the following situation:
- A borrower may find a given interest rate advantageous and decide to take out a loan. 
- During this time, the malicious Lender sees the borrower's transaction in the mempool and immediately makes a front-run attack and increases the interest rate.
- The borrower's transaction is then minted but at a higher and undesirable interest rate.
## Impact

The borrower takes an unwanted loan at the maximum interest rate. 

## Tools Used

## Recommendations

I think you should add a parameter with which a borrower is willing to accept a given interest rate. For example `maxInterestRate`. 