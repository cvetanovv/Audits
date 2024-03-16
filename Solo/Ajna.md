# Users may lose rewards

## Impact

In `RewardsManager.sol` we have [claimRewards()](https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/RewardsManager.sol#L106-L125) function:

```solidity
function claimRewards(
        uint256 tokenId_,
        uint256 epochToClaim_
    ) external override {
        StakeInfo storage stakeInfo = stakes[tokenId_];

        if (msg.sender != stakeInfo.owner) revert NotOwnerOfDeposit();

        if (isEpochClaimed[tokenId_][epochToClaim_]) revert AlreadyClaimed();

        _claimRewards(stakeInfo, tokenId_, epochToClaim_, true, stakeInfo.ajnaPool);
    }
```
This function allows the owner of a stake, to claim the rewards for a specific epoch.
First check if the `msg.sender` is the owner of the stake, after this checks if the rewards for the specified epoch have already been claimed by the stake owner and call [_claimRewards()](https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/RewardsManager.sol#L553-L598) function:
```solidity
function _claimRewards(
        StakeInfo storage stakeInfo_,
        uint256 tokenId_,
        uint256 epochToClaim_,
        bool validateEpoch_,
        address ajnaPool_
    ) internal {
  
        // revert if higher epoch to claim than current burn epoch
        if (validateEpoch_ && epochToClaim_ > IPool(ajnaPool_).currentBurnEpoch()) revert EpochNotAvailable();

        // update bucket exchange rates and claim associated rewards
        uint256 rewardsEarned = _updateBucketExchangeRates(
            ajnaPool_,
            positionManager.getPositionIndexes(tokenId_)
        );

        rewardsEarned += _calculateAndClaimRewards(tokenId_, epochToClaim_);

        uint256[] memory burnEpochsClaimed = _getBurnEpochsClaimed(
            stakeInfo_.lastClaimedEpoch,
            epochToClaim_
        );  

        emit ClaimRewards(
            msg.sender,
            ajnaPool_,
            tokenId_,
            burnEpochsClaimed,
            rewardsEarned
        );

        // update last interaction burn event
        stakeInfo_.lastClaimedEpoch = uint96(epochToClaim_);

        // transfer rewards to sender
        _transferAjnaRewards(rewardsEarned);
    }
```

And `_claimRewards()` call [_transferAjnaRewards()](https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/RewardsManager.sol#L806-L821):

```solidity
function _transferAjnaRewards(uint256 rewardsEarned_) internal {
        // check that rewards earned isn't greater than remaining balance
        // if remaining balance is greater, set to remaining balance
        uint256 ajnaBalance = IERC20(ajnaToken).balanceOf(address(this));
        if (rewardsEarned_ > ajnaBalance) rewardsEarned_ = ajnaBalance;

        if (rewardsEarned_ != 0) {
            // transfer rewards to sender
            IERC20(ajnaToken).safeTransfer(msg.sender, rewardsEarned_);
        }
    }
```
`_transferAjnaRewards()` finally transfers rewards tokens.

## Proof of Concept
The problem here is that it is possible for a user to claim their tokens and receive only a portion of the rewards.
Imagine the following situation:

- Alice decide to call `claimRewards()` and take her rewards. For example 2000 tokens.
- She passes the first two checks and function call `_claimRewards`.
- Then `_claimRewards` calculates the rewards in the variable `rewardsEarned`
- Rewards are calculated by calling the [_calculateAndClaimRewards](https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/RewardsManager.sol#L377-L414) function 
- At the end `_calculateAndClaimRewards()` updated the map that the prizes are taken: `isEpochClaimed[tokenId_][epoch] = true;`
- After this `_claimRewards` call `_transferAjnaRewards()`.
- `_transferAjnaRewards()` check that rewards earned isn't greater than remaining balance and if remaining balance is greater, set to remaining balance: 
- `if (rewardsEarned_ > ajnaBalance) rewardsEarned_ = ajnaBalance;`
- For example, `ajnaBalance` has only 500 tokens and sets `rewardsEarned_` to 500 and Alice lost 1500 tokens.
- Alice then sees that she has not received the required tokens and calls again the first function `claimRewards()` but is stopped at the second check : 
```solidity 
if (isEpochClaimed[tokenId_][epochToClaim_]) revert AlreadyClaimed();
```
- Alice lost 1500 tokens

## Tools Used

Manual Review

## Recommended Mitigation Steps

This doesn't seem very fair to me, and I think rewards should be used for each user in a mapping and tracked if there are rewards left to take and allow a user to take them instead of losing them.