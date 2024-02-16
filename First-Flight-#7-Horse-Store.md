# First Flight #7: Horse Store - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. In the Horse Store Huff contract the total supply not incremented causes paused horse NFT minting](#H-01)
    - ### [H-02. `IS_HAPPY_HORSE` macro in huff returns false even though the horse is happy](#H-03)
    - ### [H-03. `HorseStore.huff::FEED_HORSE` macro doesn't allow users to feed horse if the timestamp is divisible by 17](#H-04)

- ## Low Risk Findings
    - ### [L-01. `HorseStore.sol::HorseStore::feedHorse` and `HorseStore.sol::HorseStore::isHappyHorse` lags necessary revert conditions causing updations even though horse id doesn't exist](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #7

### Dates: Jan 11th, 2024 - Jan 18th, 2024

[See more contest details here](https://www.codehawks.com/contests/clr6s75ut00013qg9z8bpkalo)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 0
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. In the Horse Store Huff contract the total supply not incremented causes paused horse NFT minting            



## Summary
The total supply is not incremented in horse store huff contract which will pause the minting of NFT and not allows user to mint more NFTs.

## Impact
User can't mint horse.

## PoC
```
forge test --mt test_HorseIsMintedButTotalSupplyNotIncremented
```

```
function test_HorseIsMintedButTotalSupplyNotIncremented() public {
    uint256 totalSupply = horseStore.totalSupply();

    vm.prank(user);
    horseStore.mintHorse();

    uint256 tokenSupplyAfterMintingHorse = horseStore.totalSupply();

    assertEq(tokenSupplyAfterMintingHorse, totalSupply);

    address alice = makeAddr("alice");
    vm.prank(alice);
    vm.expectRevert();
    horseStore.mintHorse();
}
```

## Tools Used
Manual Review

## Recommendations
Increment the total supply after each nft mint.     
## <a id='H-02'></a>H-03. `IS_HAPPY_HORSE` macro in huff returns false even though the horse is happy            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L100

## Summary
The horse must be happy if fed within the past 24 hours, but inside the huff contract `isHappyHorse` returns false even though horse is actually happy due to wrong implementation inside `IS_HAPPY_HORSE` macro.

## Vulnerability Details
The vulnerability lies inside the `IS_HAPPY_HORSE` macro due to wrong checks at line 100.
It says that if `HORSE_HAPPY_IF_FED_WITHIN < timestamp - horseFedTimestamp` then the horse is happy but instead it should be 
`HORSE_HAPPY_IF_FED_WITHIN > timestamp - horseFedTimestamp`

As `timestamp - horseFedTimestamp` is the seconds ago the horse was last fed, so true should be return if it is less than `HORSE_HAPPY_IF_FED_WITHIN`.

## Impact
Returns false even if horse is happy

## PoC
Add the test in the file - `test/HorseStoreHuff.t.sol`

Run the test:
```cpp
forge test --mt test_isHappyHorse_ReturnsFalse_EvenIfHorseIsHappy -vv
```

```cpp
function test_isHappyHorse_ReturnsFalse_EvenIfHorseIsHappy() public {
    uint256 horseId = horseStore.totalSupply();

    vm.prank(user);
    horseStore.mintHorse();
    
    vm.warp(horseStore.HORSE_HAPPY_IF_FED_WITHIN());

    horseStore.feedHorse(horseId);

    vm.warp(block.timestamp + 1);

    console.log(horseStore.isHappyHorse(horseId));
}
```


## Tools Used
Manual Review

## Recommendations
Return true if `timestamp - horseFedTimestamp < HORSE_HAPPY_IF_FED_WITHIN`
```diff
#define macro IS_HAPPY_HORSE() = takes (0) returns (0) {
    0x04 calldataload                   // [horseId]
    LOAD_ELEMENT(0x00)                  // [horseFedTimestamp]
    timestamp                           // [timestamp, horseFedTimestamp]
    dup2 dup2                           // [timestamp, horseFedTimestamp, timestamp, horseFedTimestamp]
    sub                                 // [timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    [HORSE_HAPPY_IF_FED_WITHIN_CONST]   // [HORSE_HAPPY_IF_FED_WITHIN, timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
-   lt                                  // [HORSE_HAPPY_IF_FED_WITHIN < timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
+   gt                                  // [HORSE_HAPPY_IF_FED_WITHIN > timestamp - horseFedTimestamp, timestamp, horseFedTimestamp]
    start_return_true jumpi             // [timestamp, horseFedTimestamp]
    eq                                  // [timestamp == horseFedTimestamp]
    start_return 
    jump
    
    start_return_true:
    0x01

    start_return:
    // Store value in memory.
    0x00 mstore

    // Return value
    0x20 0x00 return
}
```
## <a id='H-03'></a>H-04. `HorseStore.huff::FEED_HORSE` macro doesn't allow users to feed horse if the timestamp is divisible by 17            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.huff#L86-L90

## Summary
The protocol mentions that horses must be able to feed all the time but in the `FEED_HORSE` macro in huff contract doesn't allow the user to feed horse if the timestamp is divisible by 17.

## Vulnerability Details
The vulnerability lies inside the `FEED_HORSE` macro inside the HorseStore.huff contract where it reverts if the timestamp is divisible by 17 and doesn't allow the user to feed the horse at all times.
```cpp
@> 0x11 timestamp mod      
endFeed jumpi                
revert 
endFeed:
stop
```
here it takes the modulo of timestamp with `0x11` which equals 17 in decimal.

## Impact
The user will not be able to feed the horse if timestamp is divisible by 17.

## PoC
Add the test in the file - `test/HorseStoreHuff.t.sol`

Run the test
```cpp
forge test --mt test_feedHorse
```

```cpp
function test_feedHorse(uint64 _timestamp) public {
    uint256 horseId = horseStore.totalSupply();

    vm.prank(user);
    horseStore.mintHorse();
    
    vm.warp(uint256(17) * _timestamp);

    vm.expectRevert();
    horseStore.feedHorse(horseId);
}
```

## Tools Used
Manual Review

## Recommendations
Neigghhhh at that line which cause revert
```diff
#define macro FEED_HORSE() = takes (0) returns (0) {
    timestamp               // [timestamp]
    0x04 calldataload       // [horseId, timestamp]
    STORE_ELEMENT(0x00)     // []

    // End execution 
-   0x11 timestamp mod      
-   endFeed jumpi                
-   revert 
-   endFeed:
    stop
}
```
		


# Low Risk Findings

## <a id='L-01'></a>L-01. `HorseStore.sol::HorseStore::feedHorse` and `HorseStore.sol::HorseStore::isHappyHorse` lags necessary revert conditions causing updations even though horse id doesn't exist            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.sol#L32

https://github.com/Cyfrin/2024-01-horse-store/blob/main/src/HorseStore.sol#L41

## Summary
The function updates the state for the horse id even though the horse id doesn't exist.

## Vulnerability Details
The vulnerability arises due to the missing revert conditions when the horse id doesn't exist.
Even though if the horse id doesn't exist it updates the state.

## Impact
Unnecessary updations even though horse id doesn't exist

## Tools Used
Manual Review

## Recommendations
Add a revert condition if token id doesn't exist in both the functions.
```sol
error HorseStore__HorseNotFound();

if (ownerOf(horseId) == address(0)) {
    revert HorseStore__HorseNotFound();            
}
```


