# TempleGold - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Users staking in TempleGoldStaking will have more than expected claimable rewards and create a DoS as for a user the rewardPerToken also contains the rewardPerToken from starting due to unupdated `userRewardPerTokenPaid` on staking](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Tokens cannot be recovered for `SpiceAuction` even when the auction is in cooldown](#M-01)
- ## Low Risk Findings
    - ### [L-01. `SpiceAuction::removeAuctionConfig` cannot be called for starting auction as it reverts due to startTime check](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: TempleDAO

### Dates: Jul 4th, 2024 - Jul 11th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-templegold)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 1
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Users staking in TempleGoldStaking will have more than expected claimable rewards and create a DoS as for a user the rewardPerToken also contains the rewardPerToken from starting due to unupdated `userRewardPerTokenPaid` on staking            



## Relevant Github Links

<https://github.com/Cyfrin/2024-07-templegold/blob/main/protocol/contracts/templegold/TempleGoldStaking.sol#L495>
<https://github.com/Cyfrin/2024-07-templegold/blob/main/protocol/contracts/templegold/TempleGoldStaking.sol#L463>

## Summary

The Staking keeps a track of global `rewardPerTokenStored` and for a particular user the `userRewardPerTokenPaid`, which ensures that the user will get their reward subtracting their previously claimed reward.

When a user opens a stake in future within the current reward distribution period they will have their `userRewardPerTokenPaid` for that stake as 0 which was expected to be updated to `rewardPerTokenStored` so that they will be excluded from all the previous reward accumulated and have a start with current reward distribution, but due to this missing implementation the user will be included in the rewards distribution from the starting and have their claimable rewards more than expected and will make (Total Claimable Rewards of all Users > Total TGLD avaiable for claims for current distribution period) and thus creates a DoS for withdrawal and other stuffs.

## Vulnerability Details

The vulnerability is present in the `TempleGoldStaking::_applyStake` function where it lags the implementation to update the `userRewardPerTokenPaid` which makes the users participating to have claimable rewards from the starting value of `rewardPerTokenStored` which will thus increase their claimable rewards by a large amount and creates a Denial of Service for claiming of rewards, as the total claimable rewards are more than TGLD claim avaiable for a particular distribution period, thus some users will not be able to claim.

When a user stakes, the `rewardPerTokenStored` is updated to calculate it for the current timestamp by including the previous supply of stakes before the user has staked so the user will be expected to receive their rewards from when they have deposited for an ongoing reward distribution period, but due to the missing updation of `userRewardPerTokenPaid` when a user stakes, it makes it calcuate their rewards by considering the previous rewards.

## Impact

* As the claimable rewards for users will be much larger than they actually should get, will make the total claimable rewards of user much more than the actual TGLD amount available for claim, thus some or all users will not be able to perform their claim.
* If minted TGLD is supplied to `TempleGoldStaking`, so after the reward for a period is consumed then this TGLD which was for next distribution will be consumed and thus creating DoS for the whole Staking system.

## PoC

Add the below coded PoC in the `TempleGoldStakingTest` contract present in file: `test/forge/templegold/TempleGoldStaking.t.sol`

Run the test:

```Solidity
forge test --mt test_ClaimableRewardsAreAllocated_MoreThanWhatWasExpected
```

```solidity
function test_ClaimableRewardsAreAllocated_MoreThanWhatWasExpected() public {
    // send temple gold to Temple Gold Staking
    // to keep things simple, lets say the reward duration is of 2 weeks and vesting period is of 1 week
    // let the tgld amount be such that reward rate is 1 tgld per second
    vm.startPrank(executor);
    staking.setVestingPeriod(1 weeks);
    staking.setRewardDuration(2 weeks);
    staking.setRewardDistributionCoolDown(0);
    vm.stopPrank();
    uint256 tgldTotalRewardAmount = 2 weeks * 1 ether;
    deal(address(templeGold), address(staking), tgldTotalRewardAmount);

    vm.prank(address(templeGold));
    staking.notifyDistribution(tgldTotalRewardAmount);


    deal(address(templeToken), address(this), 1000000 ether);
    templeToken.approve(address(staking), type(uint256).max);

    uint256 stakeAmt = 50 ether;
    // stake for alice
    staking.stakeFor(alice, stakeAmt);

    // start the distribution reward
    staking.distributeRewards();

    // now after 1 week bob stakes
    vm.warp(block.timestamp + 1 weeks);
    staking.stakeFor(bob, stakeAmt);

    // alice claims current reward
    uint256 aliceLatestStakeIdx = staking.getAccountLastStakeIndex(alice);
    staking.getReward(alice, aliceLatestStakeIdx);

    // now after 1 week, bob vesting period will be over
    vm.warp(block.timestamp + 1 weeks);

    // now alice and bob again claims their reward
    uint256 bobLatestStakeIdx = staking.getAccountLastStakeIndex(bob);
    
    uint256 aliceCurrentClaimable = staking.earned(alice, aliceLatestStakeIdx);
    uint256 bobCurrentEarnedClaimable = staking.earned(bob, bobLatestStakeIdx);
    uint256 totalCurrentClaimable = aliceCurrentClaimable + bobCurrentEarnedClaimable;

    // As the reward for bob was calculated considering the previous previous reward per token also
    // therefore the totalCurrentClaimable will be more than current avaiable rewards
    // and one of them will not be able to claim their reward
    // Because bob was allocated more rewards than they were actually expected to get

    assert(totalCurrentClaimable > templeGold.balanceOf(address(staking)));

    // alice claims
    // even if Alice doesn't make a claim, the bob claim will always revert because
    // the claimable amt calculated is higher than the total amount available for claim
    staking.getReward(alice, aliceLatestStakeIdx);

    // bob claim reverts
    vm.expectRevert();
    staking.getReward(bob, bobLatestStakeIdx);
}
```

## Tools Used

Manual Review, Unit Test in Foundry

## Recommendations

When a user peforms a stake, then it should be ensured that they are not included in the previous rewards, therefore in the `_applyStake` function update the `userRewardPerTokenPaid` for that user's stake to current `rewardPerTokenStored` so that while calculating their reward they will only be included in their current period and not in previous reward period from the starting.

```diff
function _applyStake(address _for, uint256 _amount, uint256 _index) internal updateReward(_for, _index) {
    totalSupply += _amount;
    _balances[_for] += _amount;
    _stakeInfos[_for][_index] = StakeInfo(uint64(block.timestamp), uint64(block.timestamp + vestingPeriod), _amount);
+   userRewardPerTokenPaid[_for][_index] = uint256(rewardData.rewardPerTokenStored);
    emit Staked(_for, _amount);
}
```

Now after applying the recommendation, the test fails due to a failing assert which shows that now the rewards are allocated correctly.

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Tokens cannot be recovered for `SpiceAuction` even when the auction is in cooldown            



## Relevant Github Links

<https://github.com/Cyfrin/2024-07-templegold/blob/main/protocol/contracts/templegold/SpiceAuction.sol#L250-L252>

<https://github.com/Cyfrin/2024-07-templegold/blob/main/protocol/contracts/templegold/SpiceAuction.sol#L119-L126>

## Summary

The tokens in in SpiceAuction is expected to be recovered by the `SpiceAuction::recoverToken` function for the last but not started auction. For the case when the `startAuction` is called for an auction and it is currently in cooldown, the function reverts with a message to call `RemoveAuctionConfig` function.

But `removeAuctionConfig` just removes the config of the auction and doesn't perform any recovery of the token, as a result of which calling `removeAuctionConfig` will remove config and tokens that were expected to be recovered will be stuck in the contract and as `_totalAuctionTokenAllocation` has the value of the token that was expected to be recovered, and the contract evaluate tokens in there as tokens that are allocated to the auction and now there is no possible way to perform recovery after calling `removeAuctionConfig`, thus resulting in stuck funds.

## Vulnerability Details

The vulnerability is present in the `recoverToken` function of the `SpiceAuction` contract, where it reverts with a message to call the `removeAuctionConfig` for the case when an auction is in cooldown and yet to be started.

It was expected for the tokens allocated for an auction currently in cooldown to be recovered via `recoverToken`, but due to incorrect implementation there is no way to perform recover tokens operation due to above discussed issue, as remove auction config just removes the config and does nothing else.

As `removeAuctionConfig` performs a reset operation on the epochs and auctionConfigs mapping but as `startAuction` function was already called so the funds were already allocated in \_totalAuctionTokenAllocation mapping as a result of which there is no way for those tokens to be recovered and `removeAuctionConfig` doesn't perform any updations related to `_totalAuctionTokenAllocation`.

## Impact

Tokens cannot be recovered for the case when the auction is in cooldown and is not started yet.

## PoC

Add the below coded PoC in the `SpiceAuctionTest` contract in the file: `test/forge/templegold/SpiceAuction.t.sol`

Run the test:

```Solidity
forge test --mt test_AuctionTokenCannotBeRecovered_EvenIfCoolDownIsActive
```

```solidity
function test_AuctionTokenCannotBeRecovered_EvenIfCoolDownIsActive() public {
    uint160 auctionTokenReward = 100e18;
    address recipient = makeAddr("recipient");
    
    // setting up auction config
    ISpiceAuction.SpiceAuctionConfig memory _config = ISpiceAuction.SpiceAuctionConfig({
        duration: 1 weeks,
        waitPeriod: 1,
        minimumDistributedAuctionToken: auctionTokenReward,
        starter: address(0),
        startCooldown: 1 hours,
        isTempleGoldAuctionToken: false,
        activationMode: ISpiceAuction.ActivationMode.AUCTION_TOKEN_BALANCE,
        recipient: recipient
    });

    vm.prank(daoExecutor);
    spice.setAuctionConfig(_config);

    vm.warp(block.timestamp + _config.waitPeriod);
    deal(daiToken, address(spice), auctionTokenReward);
    spice.startAuction();

    // now the cooldown period is active for the current epoch
    uint256 currentEpoch = spice.currentEpoch();
    IAuctionBase.EpochInfo memory _epochInfo = spice.getEpochInfo(currentEpoch);

    // epoch info set, and auction in cooldown
    assert(_epochInfo.startTime > block.timestamp);

    // Now there is a need to recover the reward token
    // the recover token function have a revert statement with the message to call remove auction config 
    // when startAuction is called, but on cooldown for start

    uint256 initRecipientBalance = IERC20(daiToken).balanceOf(recipient);

    vm.startPrank(daoExecutor);
    vm.expectRevert(ISpiceAuction.RemoveAuctionConfig.selector);
    spice.recoverToken(daiToken, recipient, auctionTokenReward);

    // now call the remove auction config as guided to recover the tokens
    spice.removeAuctionConfig();
    vm.stopPrank();

    // but as we know remove auction config is just used to reset the config parameters.
    // thus it is observed that the recoverToken function has implemented things in opposite manner
    assertEq(IERC20(daiToken).balanceOf(recipient) - initRecipientBalance, 0);
}
```

## Tools Used

Manual Review, Unit Test in Foundry

## Recommendations

* Updation 1
  Update the `recoverToken` function to perform the recovery of the tokens for the auction that is in cooldown and yet to be started. Instead of performing a revert for this case recover the tokens to the recipient.
  Perform the following recover operation in the `recoverToken` function for the case when auction is in cooldown:

```diff
+ uint256 amountToRecover = info.totalAuctionTokenAmount;
+ _totalAuctionTokenAllocation[token] -= amountToRecover;
+ IERC20(token).safeTransfer(to, amountToRecover);
+ delete auctionConfigs[id];
+ delete epochs[id];
```

* Updation 2
  In `removeAuctionConfig` update the `_totalAuctionTokenAllocation` mapping to remove the tokens allocated for the auction that is being removed.

```diff
if (!configSetButAuctionStartNotCalled) {
    /// @dev unlikely because this is a DAO execution, but avoid deleting old ended auctions
    if (info.hasEnded()) { revert AuctionEnded(); }
    /// auction was started but cooldown has not passed yet
+   (, address auctionToken) = _getBidAndAuctionTokens(auctionConfigs[id]);
+   _totalAuctionTokenAllocation[auctionToken] -= info.totalAuctionTokenAmount;
    delete auctionConfigs[id];
    delete epochs[id];
    _currentEpochId = id - 1;
    emit AuctionConfigRemoved(id, id);
}
```


# Low Risk Findings

## <a id='L-01'></a>L-01. `SpiceAuction::removeAuctionConfig` cannot be called for starting auction as it reverts due to startTime check            



## Relevant Github Links
<https://github.com/Cyfrin/2024-07-templegold/blob/main/protocol/contracts/templegold/SpiceAuction.sol#L113>

## Summary

The `removeAuctionConfig` function is expected to reset an auction config either for an auction that is in cooldown phase or for an auction for which config is set but `startAuction` is not called.


## Vulnerability Details
The vulnerability is present in `removeAuctionConfig` function at line 113 which reverts when startTime for current auction epoch info is 0, but it missed the scenario where for the very first auction, as when config is set then `startTime` will be 0, as epoch id is not incremented as a result of which config cannot be reset.

For the very first auction config set via `setAuctionConfig`, the config is updated for the next epoch id, i.e `1`, the currentEpochId still stores 0, as it will be updated when `startAuction` is called.

Now, for the requirement to remove auction config for the very first auction will fail, as it checks for the startTime for the current epoch id to be `0` and as for the current epoch id which is not used, it will always be 0 and as a result of which for the very first auction for which only config is set, the `removeAuctionConfig` cannot be called.

## Impact
`removeAuctionConfig` will revert for starting auction, and auction config cannot be reset.

## Tools Used
Manual Review

## Recommendations
Remove the below check from `removeAuctionConfig`:
```diff
- if (info.startTime == 0) { revert InvalidConfigOperation(); }
```



