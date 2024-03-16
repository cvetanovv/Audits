# A malicious user can burn another user's  voting power 

## Summary

A malicious user can burn another user's  voting power

## Vulnerability Detail

This `claim()` function allows a beneficiary (or any external caller) to claim tokens on behalf of a beneficiary. A malicious user can exploit this by calling a function with the victim's address. The function call `_executeClaim()`:

```solidity
function _executeClaim(
    address beneficiary,
    uint256 totalAmount
  ) internal virtual override returns (uint256 _claimed) {
    _claimed = super._executeClaim(beneficiary, totalAmount);

    // reduce voting power through ERC20Votes extension
    _burn(beneficiary, tokensToVotes(_claimed));
  }
```

Note that the function not only claims the tokens but then burns voting power. 
Thus an honest user can fall victim to a malicious user and burns voting power when the user does not want this to happen. 
Note also that `_executeClaim()` is also called by other functions, so special attention should be paid to other places to see if a similar scenario is possible.

## Impact

Malicious user can burn another user's  voting power

## Code Snippet

https://github.com/sherlock-audit/2023-06-tokensoft-cvetanovv/blob/main/contracts/contracts/claim/BasicDistributor.sol#L46-L51
https://github.com/sherlock-audit/2023-06-tokensoft-cvetanovv/blob/main/contracts/contracts/claim/abstract/AdvancedDistributor.sol#L87-L95

## Tool used

Manual Review

## Recommendation

Add access control to the function to be called only by the admin or beneficiary.
***

# `_settleClaim()` cannot be executed because `_relayerFee` is missing

## Summary

`_settleClaim()` cannot be executed because `_relayerFee` is missing

## Vulnerability Detail

`_settleClaim()` is used for claimed tokens to any valid Connext domain and is called by the functions `xReceive()`, `claimByMerkleProof()` and `claimBySignature`.

```solidity
function _settleClaim( 
    address _beneficiary,
    address _recipient,
    uint32 _recipientDomain,
    uint256 _amount
  ) internal virtual {
    bytes32 id;
    if (_recipientDomain == 0 || _recipientDomain == domain) {
      token.safeTransfer(_recipient, _amount);
    } else {
      id = connext.xcall(
        _recipientDomain, // destination domain
        _recipient, // to
        address(token), // asset
        _recipient, // delegate, only required for self-execution + slippage
        _amount, // amount
        0, // slippage -- assumes no pools on connext
        bytes('') // calldata
      );
    }
    emit CrosschainClaim(id, _beneficiary, _recipient, _recipientDomain, _amount);
  }
```

The problem here is that `_relayerFee` is missing to execute the function. 
According to the `connext` documentation: 

> `_relayerFee` is a parameter in an overloaded `xcall` method. If the call doesn't specify `relayerFee`, it will assume that the relayer fee will be paid in native asset (e.g. `xcall{value: relayerFeeInNative}`

[Reference](https://docs.connext.network/developers/reference/contracts/calls#xcall)

The relayer fee can be paid in either the native asset or the transacting asset. 
As we can see from the code snippet above the function neither passes `_relayerFee` parameter nor pays in native asset.

## Impact

`_settleClaim()` cannot be executed because `_relayerFee` is missing

## Code Snippet

https://github.com/sherlock-audit/2023-06-tokensoft-cvetanovv/blob/main/contracts/contracts/claim/abstract/CrosschainDistributor.sol#L61-L90
https://docs.connext.network/developers/reference/contracts/calls#xcall
https://docs.connext.network/developers/guides/estimating-fees

## Tool used

Manual Review

## Recommendation

Add `_relayerFee` parameter or pay in native asset
***

# Due to the lack of functionality to increase Relayer Fees it is possible that the transaction will get stuck

## Summary

Due to the lack of functionality to increase **Relayer Fees** it is possible that the transaction will get stuck

## Vulnerability Detail

The `initiateClaim()` function call `xcall()`:

```solidity
87:    // Send claim to distributor via crosschain call
88:    bytes32 transferId = connext.xcall{ value: msg.value }(  
89:      _distributorDomain, // destination domain              
90:      address(distributor), // to
91:      address(0), // asset    //@audit - missing delegate
92:      address(0), // delegate, only required for self-execution + slippage
93:      0, // total
94:      0, // slippage
95:      abi.encodePacked(msg.sender, _domain, total, proof) 
96:    );
```

To execute a cross-chain call you need to pay **Relayer Fee**

**Relayer Fee** is a fee charged by relayers on top of normal gas costs in exchange for providing a meta-transaction service. In code snippet on above is paid on line 88.

According to connext [docs](https://docs.connext.network/developers/guides/estimating-fees#bumping-relayer-fees):

> Since gas conditions are impossible to predict, transactions can potentially stay pending on destination if fees aren't high enough. Connext allows the user (or anyone if they are feeling charitable) to increase the original fee until sufficient for relayers.
	Anyone can call the Connext contract function `bumpTransfer()` to increase the original relayer fee for an `xcall`.

But such an implementation is missing and therefore it is possible that the transaction remains stuck.

## Impact

The transaction will remain stuck and nobody can react because the protocol is not implemented `bumpTransfer()` function.

## Code Snippet

https://docs.connext.network/developers/guides/estimating-fees#bumping-relayer-fees
https://github.com/sherlock-audit/2023-06-tokensoft-cvetanovv/blob/main/contracts/contracts/claim/Satellite.sol#L88-L96
https://github.com/sherlock-audit/2023-06-tokensoft-cvetanovv/blob/main/contracts/contracts/claim/abstract/CrosschainDistributor.sol#L78-L87

## Tool used

Manual Review

## Recommendation

Implemented `bumpTransfer()` function:

```solidity
function bumpTransfer(bytes32 _transferId) external payable;
```
***

# The user has no control over the crosschain transaction because `delegate` is `address(0)`

## Summary

The user has no control over the crosschain transaction because `delegate` is `address(0)`

## Vulnerability Detail

In `Satellite.sol` we have `initiateClaim()` function. This fucntion initiates crosschain claim by msg.sender. `initiateClaim()` call `connext.xcall`:

```solidity
87:    // Send claim to distributor via crosschain call
88:    bytes32 transferId = connext.xcall{ value: msg.value }(  
89:      _distributorDomain, // destination domain              
90:      address(distributor), // to
91:      address(0), // asset    //@audit - missing delegate
92:      address(0), // delegate, only required for self-execution + slippage
93:      0, // total
94:      0, // slippage
95:      abi.encodePacked(msg.sender, _domain, total, proof) 
96:    );
```

On line 92 we notice that in place of `delegate` we have `address(0)` assuming you won't have to because there's no slippage. But `delegate` is important not only because of slippage.

According to connext docs `delegate`  is:

> An address on destination domain that has rights to update slippage tolerance, retry transactions, or revert back to origin in the event that a transaction fails at the destination.

[Reference](https://docs.connext.network/developers/reference/contracts/calls#functions)

Left at `address(0)`, absolutely no one has control of the transaction and according to the documentation it is entirely possible that the transaction remains stuck or some event happens and the transaction fails at the destination. 

## Impact

When a transaction remains stuck or some event happens and the transaction fails at the destination we have no way to respond.

## Code Snippet

https://docs.connext.network/developers/reference/contracts/calls#functions
https://github.com/sherlock-audit/2023-06-tokensoft-cvetanovv/blob/main/contracts/contracts/claim/Satellite.sol#L88-L96

## Tool used

Manual Review

## Recommendation

Replace `address(0)` with `msg.sender` or `address(distributor)` to have some control over the transaction in case it gets stuck or you need to repeat it. 
***

# Signature replay attack because the missing nonce

## Summary

Signature replay attack because the missing nonce

## Vulnerability Detail

In `CrosschainMerkleDistributor.sol` we have `claimBySignature()`:
```solidity
function claimBySignature(
    address _recipient,
    uint32 _recipientDomain,
    address _beneficiary,
    uint32 _beneficiaryDomain,
    uint256 _total,
    bytes calldata _signature,
    bytes32[] calldata _proof
  ) external {
    // Recover the signature by beneficiary
    bytes32 _signed = keccak256( 
      abi.encodePacked(_recipient, _recipientDomain, _beneficiary, _beneficiaryDomain, _total)
    );
    address recovered = _recoverSignature(_signed, _signature);
    require(recovered == _beneficiary, '!recovered');

    // Validate the claim
    _verifyMembership(_getLeaf(_beneficiary, _total, _beneficiaryDomain), _proof);
    uint256 claimedAmount = _executeClaim(_beneficiary, _total);
  
    _settleClaim(_beneficiary, _recipient, _recipientDomain, claimedAmount);
  }
```
This function claims tokens for a beneficiary using a merkle proof and beneficiary signature. 
The problem here is the lack of `nonce`. Because no `nonce` in the signed message, the user can call the function multiple times. 

To prevent signature replay attacks you need to add `nonce` in the signed message. And after that, you need to validate the signature using the current `nonce`.

You can see how OpenZeppelin [pemit()](https://github.com/fractional-company/contracts/blob/master/src/OpenZeppelin/drafts/ERC20Permit.sol#L57-L60) has implemented it.
## Impact

Signature replay attack because the missing `nonce`

## Code Snippet

https://github.com/sherlock-audit/2023-06-tokensoft-cvetanovv/blob/main/contracts/contracts/claim/abstract/CrosschainMerkleDistributor.sol#L119-L140

## Tool used

Manual Review

## Recommendation

Use `nonce` to protect `claimBySignature()` from signatures reuse. 