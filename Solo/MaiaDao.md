# [H] Protocol rewards will be lost in `BoostAggregator.sol`

## Impact

In `BoostAggregator.sol` we have `withdrawProtocolFees()` function:
```solidity
function withdrawProtocolFees(address to) external onlyOwner {
        uniswapV3Staker.claimReward(to, protocolRewards);
        delete protocolRewards;
    }
```
This function allows the owner of the contract to withdraw protocol fees that have  in the contract. The problem comes when `uniswapV3Staker.claimReward` is called. 

```solidity
function claimReward(address to, uint256 amountRequested) external returns (uint256 reward) {
        reward = rewards[msg.sender];
        if (amountRequested != 0 && amountRequested < reward) {
            reward = amountRequested;
            rewards[msg.sender] -= reward;
        } else {
            rewards[msg.sender] = 0;
        }

        if (reward > 0) hermes.safeTransfer(to, reward);
  
        emit RewardClaimed(to, reward);
    }
```

As we can see the function initially sets the `rewards` to be the value of the rewards mapping at the key `msg.sender`. If `amountRequested < reward`, variable `reward` is set to the rewards from `rewards[msg.sender]` mapping.

This means that if the rewards request is larger than  `reward` variable, `amountRequested` will be set to the `reward` variable  and the remainder will be lost.

## Proof of Concept

- Owner call `withdrawProtocolFees()` to take the rewards from the contract.
- The function call `uniswapV3Staker.claimReward(to, protocolRewards);`
- In `claimReward()` suppose `amountRequested` is bigger than `reward` variable.
- The rewards to be sent are from the `UniswapV3Staker.sol` rewards mapping, and the remainder will be lost.
- After this protocol rewards will be deleted in `withdrawProtocolFees()`

```solidity
BoostAggregator.sol:

161: delete protocolRewards;
```
- Owner loses rewards forever

## Tools Used

Visual Studio Code

## Recommended Mitigation Steps

I think rewards can be sent directly from the `withdrawProtocolFees()` function without calling `claimReward()`. The other option is to refactor `claimReward()`.
Or make it so they can get them later and not lose them.
***

# [H] Users may lose rewards when call `unstakeAndWithdraw()` in `BoostAggregator.sol`

## Impact
In `BoostAggregator.sol` we have `unstakeAndWithdraw()` function:
```solidity
function unstakeAndWithdraw(uint256 tokenId) external {
        address user = tokenIdToUser[tokenId];
        if (user != msg.sender) revert NotTokenIdOwner();

        // unstake NFT from Uniswap V3 Staker
        uniswapV3Staker.unstakeToken(tokenId);

        uint256 pendingRewards = uniswapV3Staker.tokenIdRewards(tokenId) - tokenIdRewards[tokenId];

        if (pendingRewards > DIVISIONER) {
            uint256 newProtocolRewards = (pendingRewards * protocolFee) / DIVISIONER;
            /// @dev protocol rewards stay in stake contract
            protocolRewards += newProtocolRewards;
            pendingRewards -= newProtocolRewards;

            address rewardsDepot = userToRewardsDepot[user];
            if (rewardsDepot != address(0)) {
                // claim rewards to user's rewardsDepot
                uniswapV3Staker.claimReward(rewardsDepot, pendingRewards);
            } else {
                // claim rewards to user
                uniswapV3Staker.claimReward(user, pendingRewards);
            }
        }

        // withdraw rewards from Uniswap V3 Staker
        uniswapV3Staker.withdrawToken(tokenId, user, "");
    }
```

This function removes a staked `tokenId`, from the `UniswapV3Staker` contract, calculates the pending rewards, handles the protocol fees, and then withdraws the pending rewards to the user or their `rewardsDepot`.

## Proof of Concept

The problem comes when the rewards need to be sent to the user and the `claimReward()` function is called.

```solidity
function claimReward(address to, uint256 amountRequested) external returns (uint256 reward) {
        reward = rewards[msg.sender];
        if (amountRequested != 0 && amountRequested < reward) {
            reward = amountRequested;
            rewards[msg.sender] -= reward;
        } else {
            rewards[msg.sender] = 0;
        }

        if (reward > 0) hermes.safeTransfer(to, reward);
  
        emit RewardClaimed(to, reward);
    }
```

If the rewards request is larger than  `reward` variable, `amountRequested` will be set to the `reward` variable  and the remainder will be lost.
The user will think they have received the rewards but may not actually receive them.

**Note:**
I don't know if I should post another report but it will be similar so I decide to post it here and let the judge decide whether to split it.

The `claimReward()` function in `UniswapV3Staker.sol` is itself external and can be called without being bound to `BoostAggregator.sol`.
Аnd again the same problem occurs that I have described above. It is possible for a user to lose the rewards that are obtained from `_unstakeToken()`
## Tools Used

Visual Studio Code

## Recommended Mitigation Steps

You need to refactor `claimReward()` function. If there are any leftover rewards it is not fair for users to lose them.
***

# [M] Possible infinite loop in `_decrementWeightUntilFree()`

## Impact

In `_decrementWeightUntilFree()` there is a great possibility of an Infinite loop. This is because `i++` is an increment inside `if` condition. This can lead to excessive gas consumption, causing the Ethereum transaction to fail due to the gas limit.

## Proof of Concept

In `ERC20Gauges.sol` we have `_decrementWeightUntilFree()`. This function is used for freeing up "weight" before a token burn or transfer. 
`_decrementWeightUntilFree()` is an important function and is called as a greedy algorithm to free up weight.  This function is used in the function `_burn()`, `transfer()`, and `transferFrom()`. 

We should pay more attention to the following code:

```solidity
for (uint256 i = 0; i < size && (userFreeWeight + totalFreed) < weight;) {
            address gauge = gaugeList[i];
            uint112 userGaugeWeight = getUserGaugeWeight[user][gauge];
            if (userGaugeWeight != 0) {
                // If the gauge is live (not deprecated), include its weight in the total to remove
                if (!_deprecatedGauges.contains(gauge)) {
                    totalFreed += userGaugeWeight;
                }
                userFreed += userGaugeWeight;
                _decrementGaugeWeight(user, gauge, userGaugeWeight, currentCycle);
                unchecked {   
                    i++;
                }
            }
        }
```

As we can see `i++` is incremented only when `userGaugeWeight != 0` is true. 
If we don't enter the `if` condition, `i` won't increase and so we get an infinite loop.
This can lead to excessive gas consumption and any of the following functions `_burn()`, `transfer()`, and `transferFrom()` will revert.

## Tools Used

Visual Studio Code

## Recommended Mitigation Steps

To avoid this potential infinite loop, move the unchecked box outside the `if` condition.

***

# [M] No slippage protection in `deposit()`, `_withdrawAll()` and `_compoundFees()`

## Impact

Without slippage, If the price of the tokens changes significantly during the swap, it could result in a large slippage, causing users to lose a significant amount of funds.
An attacker can watch the mempool and then (using flash bots) execute a sandwich attack to manipulate the price before and after the swap.

## Proof of Concept

These functions `deposit()`, `_withdrawAll()`, and `_compoundFees()` don't have slippage protection. We can see they have params `amount0Min`  and `amount1Min` hardcoded to 0.

```solidity
    amount0Min: 0,  
    amount1Min: 0,
```

The `amount0Min` and `amount1Min` parameters in the Uniswap  are used to prevent high slippage. By setting them to a value greater than zero, you would ensure that the transaction reverts if the amount of tokens that will be added to the liquidity pool is less than these minimums. 
In a volatile market, or when dealing with large orders, the price can shift while the transaction is being mined, and the actual amount of tokens added can be less than the desired amount.

## Tools Used

Visual Studio Code

## Recommended Mitigation Steps

Do not automatically set `amount0Min` and `amount1Min` to 0, but let the user choose the value
***

# [M] Missing deadline check when performing a swap

## Impact

Missing deadline checks allow pending transactions to be maliciously executed in the future. Without deadline parameters, as a consequence, users can have their operations executed at unexpected times, when the market conditions are unfavorable.

## Proof of Concept

The problem occurs  in `RootBridgeAgent.sol`.`_gasSwapOut()` function:
```solidity
(int256 amount0, int256 amount1) = IUniswapV3Pool(poolAddress).swap(
            address(this),
            !zeroForOneOnInflow,
            int256(_amount),
            sqrtPriceLimitX96,
            abi.encode(SwapCallbackData({tokenIn: address(wrappedNativeToken)}))
        );
```

We don't have a deadline check in `_gasSwapOut()`. The deadline check ensures that the transaction can be executed on time and the expired transaction revert.

## Tools Used

Visual Studio Code

## Recommended Mitigation Steps

Introduce a `deadline` parameter in `_gasSwapOut()` function.
***

# [M] Time constants in `GovernorBravoDelegateMaia.sol` are wrong

## Impact

In `GovernorBravoDelegateMaia.sol` contract at the beginning we have several constants. We should pay more attention to the following:

```solidity
/// @notice The minimum setable voting period
    uint256 public constant MIN_VOTING_PERIOD = 80640; // About 2 weeks  

    /// @notice The max setable voting period
    uint256 public constant MAX_VOTING_PERIOD = 161280; // About 4 weeks
  
    /// @notice The min setable voting delay
    uint256 public constant MIN_VOTING_DELAY = 40320; // About 1 weeks

    /// @notice The max setable voting delay
    uint256 public constant MAX_VOTING_DELAY = 80640; // About 2 weeks
```
These constants define the minimum and maximum constants for the voting period and voting delay.

As we can see from the placed comments it is assumed that they are for this period of time because the developers calculate that a block in the Ethereum network is minted in 15 seconds. But not only is this not true (the average time is 12 sec in `mainnet`), but different chains are minted at different times. Possibly even for 1 second.

## Proof of Concept

From the documentation we can see:

> - Is it multi-chain?:  true

The average block time in [Ethereum](https://ethereum.org/en/developers/docs/blocks/#block-time) is 12s. This means 1 block every 12 sec is 5 blocks. The duration for 1 week  is `5 * 60 * 24 = 50400. 

But this is not the only problem. MaiaDao is multi-chain. I can give the following examples to see how wrong it is to hardcode time constants:
- On Polygon average block time is around 2 seconds -> https://polygonscan.com/chart/blocktime
- On BNB is 3 seconds -> https://ycharts.com/indicators/binance_smart_chain_average_block_time

These constants are used throughout the contract for checks that will lead to totally wrong results and expectations.
It's important to consider the specific block time of the blockchain where the smart contract is deployed and adjust each value accordingly to achieve the desired frequency.

## Tools Used

Visual Studio Code

## Recommended Mitigation Steps

Instead of hardcoding these constants, the best idea is to set them in the `initialize()` for each chain. 
By setting the duration dynamically in the `initialize()`, you ensure that the contract adapts to the block time of each network, making the duration consistent and appropriate for each specific blockchain environment.