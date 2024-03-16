# Slippage protection is ineffective in the `swap()` function

## Summary

Slippage protection is ineffective in the `swap()` function. `minimumAmountOut` can be zero and this means no slippage protection.
## Vulnerability Details

The `Swap()` function in `SmartVaultV3.sol` use `calculateMinimumAmountOut()` function to calculate the `minimumAmountOut` value:
```solidity
    function calculateMinimumAmountOut(bytes32 _inTokenSymbol, bytes32 _outTokenSymbol, uint256 _amount) private view returns (uint256) {
        ISmartVaultManagerV3 _manager = ISmartVaultManagerV3(manager);
        uint256 requiredCollateralValue = minted * _manager.collateralRate() / _manager.HUNDRED_PC();
        uint256 collateralValueMinusSwapValue = euroCollateral() - calculator.tokenToEur(getToken(_inTokenSymbol), _amount);
        return collateralValueMinusSwapValue >= requiredCollateralValue ?
            0 : calculator.eurToToken(getToken(_outTokenSymbol), requiredCollateralValue - collateralValueMinusSwapValue);
    }
```
This function calculate the minimum amount of an output token that needs to be received in a token swap operation. The problem is that this function is designed to safeguard the vault's collateralization ratio, not to protect against market slippage. It can return a zero value and so the `swap()` function remains without slippage protection.
## Impact

Without slippage, If the price of the tokens changes significantly during the swap, it could result in a large slippage, causing users to lose a significant amount of funds.
## Tools Used

Visual Studio Code
## Recommendations

It would be better to pass the `minimumAmountOut` value as a parameter in the `swap()` function
***

# Deadline checking is useless when swapping tokens

## Summary

Deadline checking is useless when swapping tokens because the deadline is set on `block.timestamp`.
## Vulnerability Details

In `SmartVaultV3.sol.swap()` deadline is set on `block.timestamp`:
```solidity
    function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
        uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        address inToken = getSwapAddressFor(_inToken);
        uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
                tokenIn: inToken,
                tokenOut: getSwapAddressFor(_outToken),
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: _amount - swapFee,
                amountOutMinimum: minimumAmountOut,
                sqrtPriceLimitX96: 0
            });
        inToken == ISmartVaultManagerV3(manager).weth() ?
            executeNativeSwapAndFee(params, swapFee) :
            executeERC20SwapAndFee(params, swapFee);
    }
```

As you can see the `deadline` is set to `block.timestamp` which means that whenever the miner decides to include the transaction in a block, it will be valid at that time, since `deadline` will be `block.timestamp`.

The deadline check ensures that the transaction can be executed on time and that the expired transaction reverts.
## Impact

Missing deadline checks allow pending transactions to be maliciously executed in the future. Without deadline parameters, as a consequence, users can have their operations executed at unexpected times, when the market conditions are unfavorable and they can lose funds from this.
## Tools Used

Visual Studio Code
## Recommendations

Pass the `deadline` parameter in `swap()` function.
***

# `swap()` should not use the same poolFee for all token pairs

## Summary

Fixed fee level is used when swap tokens on Uniswap.
## Vulnerability Details

`swap()` function in `SmartVaultV3.sol` using hardcoded pool fee 3000:
```solidity
    function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
        uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        address inToken = getSwapAddressFor(_inToken);
        uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount);
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
                tokenIn: inToken,
                tokenOut: getSwapAddressFor(_outToken),
                fee: 3000,
                recipient: address(this),
                deadline: block.timestamp,
                amountIn: _amount - swapFee,
                amountOutMinimum: minimumAmountOut,
                sqrtPriceLimitX96: 0
            });
        inToken == ISmartVaultManagerV3(manager).weth() ?
            executeNativeSwapAndFee(params, swapFee) :
            executeERC20SwapAndFee(params, swapFee);
    }
```

The problem is not all pools in Uniswap are created with a fee being 3000. When this does not match then the function cannot be executed.
## Impact

The swap will fail when the pool fee is not 3000
## Tools Used

Visual Studio Code
## Recommendations

Allow poolFee to be passed in as a parameter so that the correct pool will be used for the swap.
***

# The protocol does not work properly with the PAXG token because this token takes a transfer fee

## Summary

The protocol does not work properly with the PAXG token because this token takes a transfer fee
## Vulnerability Details

The protocol currently uses these tokens:

> ETH, WBTC, ARB, LINK, & PAXG

The problem is that PAXG tokens take a transfer fee on each transfer. This means that the transferred amount isn't exactly what the receiver will get. Nowhere in the protocol does it track the balance before and after the transfer. This means the protocol does not properly support the PAXG token.

## Impact

The protocol does not properly support the PAXG token. 
## Tools Used

Visual Studio Code
## Recommendations

If you want to use a PAXG token then you need to track the balance before and after the transfer.
***
# The `tokenURI` function does not check if the `_tokenId` has been minted

## Summary

The `tokenURI` function does not check if the `_tokenId` has been minted. Thus breaks the EIP721 spec.
## Vulnerability Details

In `SmartVaultManagerV5.sol` we have `tokenURI()`:
```
    function tokenURI(uint256 _tokenId) public view virtual override returns (string memory) {  
        ISmartVault.Status memory vaultStatus = ISmartVault(smartVaultIndex.getVaultAddress(_tokenId)).status();
        return INFTMetadataGenerator(nftMetadataGenerator).generateNFTMetadata(_tokenId, vaultStatus);
    }
```
This function returns unique URI  for each token (NFT) based on its `tokenId`.
According to the standard, the `tokenURI` method must revert if a non-existent `tokenId` is passed.

[Reference](https://github.com/code-423n4/2023-10-opendollar-findings/issues/243)
## Impact

`tokenURI()` is not compliant with EIP721
## Tools Used

Visual Studio Code
## Recommendations

Add a `_tokenId` existence check in `tokenURI()`.