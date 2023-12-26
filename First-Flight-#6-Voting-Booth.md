# First Flight #6: Voting Booth - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Malicious Test inside VotingBoothTest contract creates a new file inside the current directory and ffi is enable true](#H-01)
    - ### [H-02. Reward per voter calculated by `VotingBooth::_distributeRewards()` also considers the `against` voters leading to stuck eth and reduced payout to `for` voters](#H-02)
- ## Medium Risk Findings
    - ### [M-01. Unsupported Opcode PUSH0 in solidity 0.8.23 for deployment on Arbitrum Chain](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #6

### Dates: Dec 15th, 2023 - Dec 22nd, 2023

[See more contest details here](https://www.codehawks.com/contests/clq5cx9x60001kd8vrc01dirq)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 1
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Malicious Test inside VotingBoothTest contract creates a new file inside the current directory and ffi is enable true            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-Voting-Booth/blob/main/test/VotingBoothTest.t.sol#L88-L93

https://github.com/Cyfrin/2023-12-Voting-Booth/blob/main/foundry.toml#L9

## Summary
The Test contract contains the malicious function `testPwned` which creates a new file in the current directory as ffi is turned on true.

## Vulnerability Details
The vulnerability lies inside the VotingBoothTest contract which contains the function `testPwned` and can run arbitrary commands in user's cli as ffi is turned true in `foundry.toml`.
The function creates a new file named `youve-been-pwned-remember-to-turn-off-ffi!` by interacting with the user's cli. It can also be used to carry out harmful stuffs such as deleting the user's data, installing malicious files, uploading the user's private data to hacker.

## Impact
Considering the current scenario it only creates a new file, but many different harmful commands can be executed on user's machine that can install malicious content, delete the user's data, send the user's sensitive data to the attacker.

## Tools Used
Hawk's eyes 👀

## Recommendations
Before interacting, straightaway go to `foundry.toml` and if there is `ffi = true` then instantly punch it out off the file. (Punch go booooooom)
## <a id='H-02'></a>H-02. Reward per voter calculated by `VotingBooth::_distributeRewards()` also considers the `against` voters leading to stuck eth and reduced payout to `for` voters            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-Voting-Booth/blob/main/src/VotingBooth.sol#L192

https://github.com/Cyfrin/2023-12-Voting-Booth/blob/main/src/VotingBooth.sol#L207

## Summary
The VotingBooth contract pays the eth reward to `for` voters if the proposal was passed, but while calculating `rewardPerVoter` it also considers the `against` voters and as the protocol pays eth reward only to the `for` voters, the remaining amount of eth will get stuck in the VotingBooth contract as the eth reward was reduced by also considering `against` voters. 

As the protocol only pays to the `for` voters on successful proposal, so the intended implementation should be to calculate reward by only considering `for` voters and not `against` voters.

## Vulnerability Details
The vulnerability lies inside the VotingBooth contract inside the `_distributeRewards` function at line 192 and 207 while calculating the rewards that is to be distributed to each `for` voter.

The function is intended to send eth only to the `for` voters (when the proposal is successful), but while calculating the `rewardPerVoter` it also considers the `against` voters as a result of which the eth to be distributed to each `for` voter is reduced by `against` voters and `for` voters will receive less amount of eth.

The total eth reward in the contract is divided by `totalVotes` which is a sum of both `for` voters and `against` voters, thus reducing the eth amount that is to be sent to the `for` voters and in the for loop eth is distributed to `for` voters (which is intended) but as a result of this there will be eth remaining in the contract and it will get permanently stuck in there and there is no way to retrieve it. 

```cpp
    function _distributeRewards() private {
        uint256 totalVotesFor = s_votersFor.length;
        uint256 totalVotesAgainst = s_votersAgainst.length;
        uint256 totalVotes = totalVotesFor + totalVotesAgainst;

        uint256 totalRewards = address(this).balance;

        if (totalVotesAgainst >= totalVotesFor) {
            _sendEth(s_creator, totalRewards);
        }
        else {
@>          uint256 rewardPerVoter = totalRewards / totalVotes;

            for (uint256 i; i < totalVotesFor; ++i) {
                if (i == totalVotesFor - 1) {
@>                  rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
                }
                _sendEth(s_votersFor[i], rewardPerVoter);
            }
        }
    }
```

## Impact
- Due to wrong implemented `_distributeRewards` function, it calculates reward to be distributed to `for` voters also considering the `against` voters, leading to reduced amount of eth sent to `for` voters. Thus it will have a impact on the eth amount to be distributed to `for` voters and it will be less than the expected amount that is to be sent.
- Also, as the reward per voter amount is reduced, there will be eth remaining in the `VotingBooth` contract and it will get permanently stuck in there and there is no way to rescue the remaining eth amount.

## PoC
Create a new file `VotingBoothInvariantTest.t.sol` inside the `Test` folder and add the below test contract inside it.

Run the test:
```cpp
forge test --mt invariant_voting -vvvv
```

```cpp
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.23;

import { VotingBooth } from "../src/VotingBooth.sol";
import { Test, console } from "forge-std/Test.sol";
import { StdInvariant } from "forge-std/StdInvariant.sol";

contract VotingBoothInvariantTest is StdInvariant {
    VotingBooth votingBooth;
    uint256 initBalance = 1 ether;

    function setUp() external {
        address[] memory allowList = new address[](9);
        for (uint160 i = 1; i < 10; i++) {
            allowList[i - 1] = address(i);
            targetSender(allowList[i - 1]);
        }
        votingBooth = new VotingBooth{value: initBalance}(allowList);
        targetContract(address(votingBooth));
    }

    function invariant_voting() public view {
        // if on voting, the quorum is not reached then voting is still active, and whole money should be there
        // but if quorum is reached upon voting then it should be inactive and contract should distribute all the
        // eth to the 'for' voters

        if (votingBooth.isActive()) {
            assert(address(votingBooth).balance == initBalance);
        }
        else {
            console.log("Remaining Contract Balance-", address(votingBooth).balance);
            assert(address(votingBooth).balance == 0);
        }
    }

    // as this (VotingBoothInvariantTest) contract deploys the VotingBooth contract, therefore
    // it is the creator of the VotingBooth contract, and if in case proposal fails so it will send
    // eth to this contract, thus receive function is req to receive eth
    receive() external payable {}
}
```
- Here it says the assertion failed. The remaining contract balance is expected to be 0 after voting ends but it is 0.4 eth
```diff
+[FAIL. Reason: panic: assertion failed (0x01)]
        [Sequence]
                sender=0x0000000000000000000000000000000000000005 addr=[src/VotingBooth.sol:VotingBooth]0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f calldata=vote(bool) args=[true]
                sender=0x0000000000000000000000000000000000000003 addr=[src/VotingBooth.sol:VotingBooth]0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f calldata=vote(bool) args=[true]
                sender=0x0000000000000000000000000000000000000004 addr=[src/VotingBooth.sol:VotingBooth]0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f calldata=vote(bool) args=[false]
                sender=0x0000000000000000000000000000000000000006 addr=[src/VotingBooth.sol:VotingBooth]0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f calldata=vote(bool) args=[true]
                sender=0x0000000000000000000000000000000000000009 addr=[src/VotingBooth.sol:VotingBooth]0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f calldata=vote(bool) args=[false]
 invariant_voting() (runs: 256, calls: 3832, reverts: 2574)
Logs:
+ Remaining Contract Balance- 400000000000000000
```

## Tools Used
Manual Review, Foundry Test, Fuzzy Cat (Meow)

## Recommendations
### 1. Calculate the `rewardPerVoter` correctly by dividing the total rewards by total number of `for` voters.

- For line 192
```diff
-uint256 rewardPerVoter = totalRewards / totalVotes;
+uint256 rewardPerVoter = totalRewards / totalVotesFor;
```
- For line 207
```diff
-rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
+rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotesFor, Math.Rounding.Ceil);
```

### 2. If the protocol wants to reduce the rewards of the `for` voters by considering the `against` voters, then remaining balance should be sent back to the `s_creator`

Therefore consider sending eth to the creator inside the else statement which is at line 191 inside `VotingBooth::_distributeRewards()` function.
Add the below line just after the for loop inside the else statement discussed above.
```cpp
_sendEth(s_creator, address(this).balance);
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Unsupported Opcode PUSH0 in solidity 0.8.23 for deployment on Arbitrum Chain            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-Voting-Booth/blob/main/src/VotingBooth.sol#L2

## Summary
The contract is set to be deployed on Arbitrum chain but the solidity version 0.8.23 has opcode PUSH0 which is not supported on Arbitrum, thus it cannot be deployed.

## Vulnerability Details
The concern is regarding the usage of solc version 0.8.23 in the smart contract. The version exhibits the PUSH0 opcode and is currently not supported across all EVM chains. The contract is required to be deployed on Arbitrum chain, but it doesn't support the PUSH0 opcode.
The vulnerability here is that the contracts compiled with solidity versions above 0.8.19 will not be able to deploy, or even if they are able to deploy then may not function properly and may lead to other consequences.

## Impact
The impact of using the solidity version 0.8.23 is that it comes with the PUSH0 opcode and this opcode is not supported on Arbitrum causing the smart contract to malfunction and the contract may not execute correctly.

## Tools Used
Manual Review

## Recommendations
PUSH0 opcode comes with 0.8.20 and higher versions, therefore switching to 0.8.19 will make the smart contract fully compatible to be deployed on Arbitrum chain.

- VotingBooth.sol
```diff
// SPDX-License-Identifier: MIT
-pragma solidity ^0.8.23;
+pragma solidity 0.8.19;
```




