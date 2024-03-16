# In `L1ERC20Bridge.sol` it is possible that the `totalDepositedAmountPerUser` mapping is not correct and therefore a user cannot use the `deposit()` function

## Impact

In `L1ERC20Bridge.sol` it is possible that the `totalDepositedAmountPerUser` mapping is not correct and therefore a user cannot use the `deposit()` function for a specific token.  

This is because the `deposit()` function checks the deposit limit by calling `_verifyDepositLimit()`. And because of an incorrect `totalDepositedAmountPerUser` statement, a user will not be able to use the `deposit()` function when they should be able to make that transaction.
## Proof of Concept

In `L1ERC20Bridge.sol` we have `deposit()` function:
```solidity
function deposit(
        address _l2Receiver,
        address _l1Token,
        uint256 _amount,
        uint256 _l2TxGasLimit,
        uint256 _l2TxGasPerPubdataByte,
        address _refundRecipient
    ) public payable nonReentrant senderCanCallFunction(allowList) returns (bytes32 l2TxHash) {
        require(_amount != 0, "2T"); // empty deposit amount
        uint256 amount = _depositFunds(msg.sender, IERC20(_l1Token), _amount);
        require(amount == _amount, "1T"); // The token has non-standard transfer logic
        // verify the deposit amount is allowed
        _verifyDepositLimit(_l1Token, msg.sender, _amount, false);


        bytes memory l2TxCalldata = _getDepositL2Calldata(msg.sender, _l2Receiver, _l1Token, amount);
        // If the refund recipient is not specified, the refund will be sent to the sender of the transaction.
        // Otherwise, the refund will be sent to the specified address.
        // If the recipient is a contract on L1, the address alias will be applied.
        address refundRecipient = _refundRecipient;
        if (_refundRecipient == address(0)) {
            refundRecipient = msg.sender != tx.origin ? AddressAliasHelper.applyL1ToL2Alias(msg.sender) : msg.sender;
        }
        l2TxHash = zkSync.requestL2Transaction{value: msg.value}(
            l2Bridge,
            0, // L2 msg.value
            l2TxCalldata,
            _l2TxGasLimit,
            _l2TxGasPerPubdataByte,
            new bytes[](0),
            refundRecipient
        );

        // Save the deposited amount to claim funds on L1 if the deposit failed on L2
        depositAmount[msg.sender][_l1Token][l2TxHash] = amount;

        emit DepositInitiated(l2TxHash, msg.sender, _l2Receiver, _l1Token, amount);
    }
```
This function initiates a deposit by locking funds on the contract and sending the request of processing an L2 transaction where tokens would be minted. 

If the transaction fails we can call `claimFailedDeposit()`:
```
function claimFailedDeposit(
        address _depositSender,
        address _l1Token,
        bytes32 _l2TxHash,
        uint256 _l2BatchNumber,
        uint256 _l2MessageIndex,
        uint16 _l2TxNumberInBatch,
        bytes32[] calldata _merkleProof
    ) external nonReentrant senderCanCallFunction(allowList) {
        bool proofValid = zkSync.proveL1ToL2TransactionStatus(
            _l2TxHash,
            _l2BatchNumber,
            _l2MessageIndex,
            _l2TxNumberInBatch,
            _merkleProof,
            TxStatus.Failure
        );
        require(proofValid, "yn");

        uint256 amount = depositAmount[_depositSender][_l1Token][_l2TxHash];
        require(amount > 0, "y1");

        // Change the total deposited amount by the user
        _verifyDepositLimit(_l1Token, _depositSender, amount, true);

        delete depositAmount[_depositSender][_l1Token][_l2TxHash];
        // Withdraw funds
        IERC20(_l1Token).safeTransfer(_depositSender, amount);
  
        emit ClaimedFailedDeposit(_depositSender, _l1Token, amount);
    }
```

`claimFailedDeposit()` withdraw funds from the initiated deposit, that failed when finalizing on L2.

Both functions call `_verifyDepositLimit()`:
```solidity
function _verifyDepositLimit(address _l1Token, address _depositor, uint256 _amount, bool _claiming) internal {
        IAllowList.Deposit memory limitData = IAllowList(allowList).getTokenDepositLimitData(_l1Token);
        if (!limitData.depositLimitation) return; // no deposit limitation is placed for this token

        if (_claiming) {
            totalDepositedAmountPerUser[_l1Token][_depositor] -= _amount;
        } else {
            require(totalDepositedAmountPerUser[_l1Token][_depositor] + _amount <= limitData.depositCap, "d1");
            totalDepositedAmountPerUser[_l1Token][_depositor] += _amount;
        }
    }
```
The `deposit()` function uses this functor to check the deposit limit is reached to its cap or not. This is done by the following mapping:
```solidity
totalDepositedAmountPerUser[_l1Token][_depositor]
```
`claimFailedDeposit()` calls the `_verifyDepositLimit()` function with the last parameter `bool _claiming = true` to be able to reduce the `totalDepositedAmountPerUser` mapping because the transaction did not actually occur:
```solidity
        if (_claiming) {
            totalDepositedAmountPerUser[_l1Token][_depositor] -= _amount;
```

Now imagine the following situation:
*I use sample numbers and a random token for easy understanding*
- At the beginning, there is a deposit limit with a `depositCap` of 100 000 tokens
- A user decides to deposit 100,000 tokens and `totalDepositedAmountPerUser[_l1Token][_depositor]` becomes the maximum allowed.
- During this time the admin removes the deposit limitation.
- However, the transaction fails and the user calls `claimFailedDeposit()` which calls `_verifyDepositLimit()` and everything goes right. 
- However, we should note that `totalDepositedAmountPerUser[_l1Token][_depositor]` is not reset because there is no longer a deposit limit and the function cannot get to `totalDepositedAmountPerUser[_l1Token][_depositor] -= _amount;`
- Thus `totalDepositedAmountPerUser` will remain forever at 100 000 tokens, with no way to reduce it.
- Some time passes and the admin puts the deposit limitation again, for example, 90 000 tokens. 
- And now `totalDepositedAmountPerUser[_l1Token][_depositor]` is 100 000 tokens but actually, it should be 0.
- The user cannot use the `deposit()` function at all because he will always be stopped at the following code:
```solidity
require(totalDepositedAmountPerUser[_l1Token][_depositor] + _amount <= limitData.depositCap, "d1");
```
## Tools Used

Visual Studio Code

## Recommended Mitigation Steps

When the `_verifyDepositLimit()` function is called with the `_claiming = true` parameter, the mapping reduction must be before the deposit limitation check:

```solidity
function _verifyDepositLimit(address _l1Token, address _depositor, uint256 _amount, bool _claiming) internal {
        IAllowList.Deposit memory limitData = IAllowList(allowList).getTokenDepositLimitData(_l1Token);
        if (_claiming) {
            totalDepositedAmountPerUser[_l1Token][_depositor] -= _amount;
        }
        if (!limitData.depositLimitation) return; // no deposit limitation is placed for this token

        if (!_claiming) {
            require(totalDepositedAmountPerUser[_l1Token][_depositor] + _amount <= limitData.depositCap, "d1");
            totalDepositedAmountPerUser[_l1Token][_depositor] += _amount;
        };
    }
```