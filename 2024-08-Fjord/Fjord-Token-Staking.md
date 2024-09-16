# Fjord Token Staking - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Incorrect argument passed while calling `FjordPoints::onUnstaked` inside `FjordStaking::_unstakeVested`](#M-01)
- ## Low Risk Findings
    - ### [L-01. Incorrect event emission in `FjordPoints::distributePoints`](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Fjord

### Dates: Aug 20th, 2024 - Aug 27th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-fjord)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- Medium: 1
- Low: 1



    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Incorrect argument passed while calling `FjordPoints::onUnstaked` inside `FjordStaking::_unstakeVested`            



## Relevant Github Links
<https://github.com/Cyfrin/2024-08-fjord/blob/main/src/FjordStaking.sol#L561>

## Summary
- `FjordStaking::_unstakeVested` is an internal function, which can be called via `unstakeVested` and `onStreamCanceled` function.
-  `unstakeVested` function allows users to unstake their whole NFT, while `onStreamCanceled` is called when the stream creator cancels the stream and the function either unstakes the NFT or removes the staked amount for the NFT.
- But when `onStreamCanceled` function calls `_unstakeVested`, where `_unstakeVested` function will unstake the amount for the NFT and also updates the staked amount on `FjordPoints` contract via `FjordPoints::onUnstaked`. 
- But instead of passing the sablier NFT owner it passes `msg.sender` i.e., Sablier contract to the `onUnstaked` function and as Sablier contract has 0 staked amount on fjord points, it will result in a revert which thus reverts the `onStreamCanceled` function but Sablier contract will ignore this revert.
- Thus the stream was cancelled but staked data remains unupdated on `FjordStaking` contract and the sablier NFT owner will be able to receive rewards on the whole value including the value which was supposed to be unstaked.

## Vulnerability Details
- The vulnerability is present in the `FjordStaking::_unstakeVested` where it passes `msg.sender` as user for which unstaking is done to `onUnstaked` function instead of `streamOwner`.
- Here, `msg.sender` will not be the sablier NFT owner (stream owner) in all cases.
- For the case when `_unstakeVested` is called by `onStreamCanceled` function, then `msg.sender` is going to be the Sablier contract and not the stream owner as a result of which it tries to reduce the staked amount data of Sablier contract on `FjordPoints` contract, thus leading to revert.
- This revert would thus cause the `onStreamCanceled` function to revert and all the updations related to unstaking of amount of user will be undone, but the stream creator cancelling of the stream would be successful as the revert caused by `onStreamCanceled` is ignored by Sablier contract.

## Impact
- When stream is cancelled by creator, the NFT holder's will have no ownership of the unstreamed amount. But as `onStreamCanceled` faced a revert, the unstreamed amount is still in the accounting of the `FjordStaking` and `FjordPoints` contract.
- Thus, user will be able to receive rewards on the amount which they do not own as it is still in their staked amount.

## Tools Used
Manual Review

## Recommendations
In `FjordStaking::_unstakeVested`, instead of passing `msg.sender` to `onUnstaked` function, pass `streamOwner`
```diff
- points.onUnstaked(msg.sender, amount);
+ points.onUnstaked(streamOwner, amount);
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Incorrect event emission in `FjordPoints::distributePoints`            



## Github Link
<https://github.com/Cyfrin/2024-08-fjord/blob/main/src/FjordPoints.sol#L247>

## Summary
- `distributePoints` function emits the event `PointsDistributed`.
- The first parameter of the event is the total amount of points that are distributed.
- But for the case when `weeksPending` has a value `>1`, then total points distributed will be `weeksPending * pointsPerEpoch`.
- But `pointsPerEpoch` is passed to the event for every case even if `weeksPending` has a value `>1`.

## Vulnerability Details
- The vulnerability is present in the `distributePoints` function where it passes incorrect argument while emitting the `PointsDistributed` event.
- The event expects 2 parameters - total points distributed and points per token.
- But for the total points distributed it passes `pointsPerEpoch` which denotes the total points for a single epoch and not total points distributed.
- As for the case when `weeksPending` > 1, the total points distributed will be the product of `pointsPerEpoch` and `weeksPending`, but instead `pointsPerEpoch` is passed for every case.

## Impact
Incorrect event data emitted leads to incorrect off-chain updation.

## Tools Used
Manual Review

## Recommendations
Correct the event emission as below:
```diff
- emit PointsDistributed(pointsPerEpoch, pointsPerToken);
+ emit PointsDistributed(pointsPerEpoch * weeksPending, pointsPerToken);
```



