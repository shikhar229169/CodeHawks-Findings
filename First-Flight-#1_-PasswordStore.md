# First Flight #1: PasswordStore - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Password can be viewed by anyone and is not really private.](#H-01)
    - ### [H-02. The password set by PasswordStore::setPassword() can be viewed by anyone.](#H-02)
    - ### [H-03. PasswordStore::setPassword() lags only owner check.](#H-03)

- ## Low Risk Findings
    - ### [L-01. Wrong NatSpec for the PasswordStore::getPassword()](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #1

### Dates: Oct 18th, 2023 - Oct 25th, 2023

[See more contest details here](https://www.codehawks.com/contests/clnuo221v0001l50aomgo4nyn)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 0
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Password can be viewed by anyone and is not really private.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/main/src/PasswordStore.sol#L14

## Summary
Even if the variable s_password is private, but it can be viewed by anyone. Everyone can read data from the contract's storage slot.

## Vulnerability Details
If anyone gets the storage slot at which s_password data is stored, then they can easily get the password.

- If the password fits in the 32 bytes slot, then it will be present at the slot 1 (in reference with the PasswordStore contract).
So, we can easily get it by: 
```solidity
function test_GetPassword() public view {        
    uint256 zero = 0;
    bytes32 slot = bytes32(zero + 1);                                        // Storage slot 1
    bytes32 data = vm.load(address(passwordStore), slot);         // Getting data at storage slot 1
    string memory passwd = string(abi.encodePacked(data));                     // Converting to string
    console.log(passwd);
}
```

- But if the password can't fit in the 32 bytes slot, then the storage slot changes where password is stored, and slot 1 will only contain the length of string.

```solidity
function test_GetPasswordTwo() public view {
    uint256 zero = 0;

    bytes32 firstSlot = keccak256(abi.encode(zero + 1));               // Now the password is stored at keccak256 of slot 1
    bytes32 bytesPassword = vm.load(address(passwordStore), firstSlot);  // Get the data from that slot
    string memory passwd = string(abi.encodePacked(bytesPassword));
    console.log(passwd);

    // If the password is big and it can't fit in firstSlot, then it is continued in previous slot (i.e. firstSlot) + 1
    bytes32 secondSlot = bytes32(uint256(firstSlot) + 1);          
    bytes32 bytesPassword2 = vm.load(address(passwordStore), secondSlot);
    string memory passwd2 = string(abi.encodePacked(bytesPassword2));
    console.log(passwd2);
}
```

And if the password still can't be stored in secondSlot, then the next slot will be previous slot (i.e. secondSlot) + 1, and so on.



## Impact
Bad impact on user's accounts, as anyone can use the password to authenticate to user's accounts.

## Tools Used
Manual Review

## Recommendations
To encrypt the password with user's public key, so that only the user can decrypt it with their private key.
## <a id='H-02'></a>H-02. The password set by PasswordStore::setPassword() can be viewed by anyone.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/main/src/PasswordStore.sol#L26

## Summary
When a txn. is made on blockchain, it can be viewed by anyone, so anyone can see the password from the argument of the function call.

## Vulnerability Details
Anyone can see the user's set password

## Impact
Other people can exploit the user by getting access to user's accounts.

## Tools Used
Manual Review

## Recommendations
To encrypt the password with the user's public key, so that only the user can decrypt it with their private key.
## <a id='H-03'></a>H-03. PasswordStore::setPassword() lags only owner check.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/main/src/PasswordStore.sol#L23

## Summary
The function setPassword's NatSpec mentions that only owner should be allowed to change it, but it can be changed by anyone as it lags the only owner check.

## Vulnerability Details
If there is no only owner check, then anyone can call the setPassword function and can change the password anytime.

## Impact
Owner will get wrong password.

## Tools Used
Manual Review

## Recommendations
To add a only owner check.
```diff
function setPassword(string memory newPassword) external {
+   if (msg.sender != s_owner) {
+       revert PasswordStore__NotOwner();
+   }
    s_password = newPassword;
    emit SetNetPassword();
}
```
		


# Low Risk Findings

## <a id='L-01'></a>L-01. Wrong NatSpec for the PasswordStore::getPassword()            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-PasswordStore/blob/main/src/PasswordStore.sol#L33

## Summary
There is a wrong NatSpec for the PasswordStore::getPassword(), as it allows only the user to get the password, but NatSpec mentions a param newPassword, which is not actually required.

## Impact
Low

## Tools Used
Manual

## Recommendations
Remove the wrong NatSpec.


