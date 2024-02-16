# First Flight #8: Math Master - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect constant number used inside square root function for comparison.](#H-01)
    - ### [H-02. Irrelevant statement incrementing x inside `mulWadUp` leads to incorrect answer](#H-02)
    - ### [H-03. Incorrect overflow check inside `mulWadUp` evaluates revert condition to false even if there is a overflow.](#H-03)

- ## Low Risk Findings
    - ### [L-01. mulWad and mulWadUp overrides solidity free memory pointer and takes data for revert from incorrect memory position.](#L-01)
    - ### [L-02. Solidity version 0.8.3 doesn't support custom errors defined by `error` keyword.](#L-02)
    - ### [L-03. Incorrect error selector used while reverting in mulWad and mulWadUp](#L-03)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #8

### Dates: Jan 25th, 2024 - Feb 2nd, 2024

[See more contest details here](https://www.codehawks.com/contests/clrp8xvh70001dq1os4gaqbv5)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 0
   - Low: 3


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect constant number used inside square root function for comparison.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L77

## Summary
`sqrt` function of MathMasters library uses incorrect constant value for comparison, which leads to wrong answer for certain cases.

## Vulnerability Details
The vulnerability occurs due to the usage of wrong value for comparison.
At line 77, constant value `16777002` is used instead of `16777215`.

## Impact
Value of `r` calculated at line 77 will be incorrect for some test cases and will lead to wrong answer.

## Tools Used
Manual Review

## Recommendations
Change `16777002` to `16777215` at line 77.
```diff
-r := or(r, shl(4, lt(16777002, shr(r, x))))
+r := or(r, shl(4, lt(16777215, shr(r, x))))
```

## <a id='H-02'></a>H-02. Irrelevant statement incrementing x inside `mulWadUp` leads to incorrect answer            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L56

## Summary
Irrelevant statement incrementing x inside mulWadUp leads to incorrect answer.

## Vulnerability Details
- The vulnerability occurs due to the statement inside mulWadUp at line 56 performing irrelevant increment of `x`.
```cpp
    function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, or(div(not(0), y), x))) {
                mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
                revert(0x1c, 0x04)
            }
@>          if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
            z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
        }
    }
```

## Impact
Returns incorrect answer.

## PoC
- Add the test in the file: `test/MathMasters.t.sol`
- Set the `runs` to  `100000000` under `[fuzz]` inside `foundry.toml` and comment out all the other parameters.

Run the test: 
```cpp
forge test --mt test_MulWadUp_ReturnsWrongAnswer -vv
```

```cpp
function test_MulWadUp_ReturnsWrongAnswer(uint256 x, uint256 y) public view {
    vm.assume(y == 0 || x <= type(uint256).max / y);
    uint256 ans1 = MathMasters.mulWadUp(x, y);
    uint256 ans2 = x * y == 0 ? 0 : (x * y - 1) / 1e18 + 1;

    console2.log('Actual Answer:', ans1);
    console2.log('Expected Answer:', ans2);

    assert(ans1 == ans2);
}
```

Failed CounterExample:
```
Running 1 test for test/MathMasters.t.sol:MathMastersTest
[FAIL. Reason: panic: assertion failed (0x01); counterexample: calldata=0xf494bb07000000000000000000000000000000000000000000061e597cef96fd49e369c0000000000000000000000000000000000000000000030f2cbe77cb7ea4f1b4e1 args=[7396876674976619302185408 [7.396e24], 3698438337488309651092705 [3.698e24]]] test_MulWadUp_ReturnsWrongAnswer(uint256,uint256) (runs: 512, μ: 7693, ~: 7744)
Logs:
  Actual Answer: 27356892272406583674190305384389
  Expected Answer: 27356892272406583674190301685951

Traces:
  [7785] MathMastersTest::test_MulWadUp_ReturnsWrongAnswer(7396876674976619302185408 [7.396e24], 3698438337488309651092705 [3.698e24])
    ├─ [0] VM::assume(true) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log("Actual Answer:", 27356892272406583674190305384389 [2.735e31]) [staticcall]
    │   └─ ← ()
    ├─ [0] console::log("Expected Answer:", 27356892272406583674190301685951 [2.735e31]) [staticcall]
    │   └─ ← ()
    └─ ← panic: assertion failed (0x01)

Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 39.77ms
```

## Tools Used
Manual Review

## Recommendations
Remove the irrelevant statement
```diff
    function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
            if mul(y, gt(x, or(div(not(0), y), x))) {
                mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
                revert(0x1c, 0x04)
            }
-          if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
            z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
        }
    }
```
## <a id='H-03'></a>H-03. Incorrect overflow check inside `mulWadUp` evaluates revert condition to false even if there is a overflow.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L52

## Summary
mulWadUp function should revert if there is a overflow but due to incorrect overflow checks it will not revert and returns a wrong answer because of overflow.

## Vulnerability Details
The vulnerability occurs due to the wrong overflow condition check inside mulWadUp function inside MathMasters.sol

It is assumed that the overflow check at line 52 is equivalent to `require(y == 0 || x <= type(uint256).max / y)` but it is actually not.
```cpp
    function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
@>          if mul(y, gt(x, or(div(not(0), y), x))) {
                mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
                revert(0x1c, 0x04)
            }
            if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
            z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
        }
    }
```
The incorrect overflow check at line 52 is `mul(y, gt(x, or(div(not(0), y), x)))`

Here first it gets the maximum safe value of x by `div(not(0), y)` and then performs `or` operation with x. 

We know that `or` operation performed on `x` and `div(not(0), y)` will yield at least `x` as the answer and the condition `gt(x, or(div(not(0), y), x))` will always be false, because `or(div(not(0), y), x)` will always be `>=x`.

Thus, even if there is a overflow it will not revert and still returns incorrect answer.

## Impact
In case of overflow, the function `mulWadUp` will not revert and still returns incorrect answer.

## PoC
Add the test in the file: `test/MathMasters.t.sol`

Run the test:
```cpp
forge test --mt test_MulWadUp_NotReverts_Even_IfThereIsAOverflow -vv
```
```cpp
function test_MulWadUp_NotReverts_Even_IfThereIsAOverflow() public view {
    // Both the args are maximum uint256 values
    uint256 ans = MathMasters.mulWadUp(type(uint256).max, type(uint256).max);

    // the function doesn't revert on overflow and still gives the incorrect answer
    console2.log(ans);
}
```

## Tools Used
Manual Review

## Recommendations
Eliminate the unnecessary `or` operation.
```diff
    function mulWadUp(uint256 x, uint256 y) internal pure returns (uint256 z) {
        /// @solidity memory-safe-assembly
        assembly {
            // Equivalent to `require(y == 0 || x <= type(uint256).max / y)`.
-          if mul(y, gt(x, or(div(not(0), y), x))) {
+          if mul(y, gt(x, div(not(0), y))) {
                mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
                revert(0x1c, 0x04)
            }
            if iszero(sub(div(add(z, x), y), 1)) { x := add(x, 1) }
            z := add(iszero(iszero(mod(mul(x, y), WAD))), div(mul(x, y), WAD))
        }
    }
```
		


# Low Risk Findings

## <a id='L-01'></a>L-01. mulWad and mulWadUp overrides solidity free memory pointer and takes data for revert from incorrect memory position.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L40-L41

https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L53-L54

## Summary
- The mulWad and mulWadUp function of MathMaster library overrides the solidity free memory pointer, i.e. 0x40. 
0x40 stores the address of memory which is free but instead of using the value at 0x40, both these functions directly use the 0x40 memory position to store the error selector, thus overriding the free memory pointer.
- Along with that while reverting with custom errors the memory position of the stored error selector is to be used but both these functions uses incorrect memory position (0x1c) to get the revert selector.

## Vulnerability Details
The vulnerability occurs due to the usage of 0x40 memory location to store the revert selector instead of using the value at 0x40.
Thus losing the free memory address which is stored at 0x40 in solidity.
Also, in the revert statement the error selector is fetched from 0x1c location which is irrelevant as it is not used to store the selector.

## Impact
- Free memory address which is stored at 0x40 will be lost.
- Reverts with incorrect error.

## Tools Used
Manual Review

## Recommendations
- Use the value present as 0x40 to store the error selector and fetch the error selector from the correct position, i.e. the position at which the error selector is stored.
```diff
-mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
-revert(0x1c, 0x04)
+let pos = mload(0x40)
+mstore(pos, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
+revert(pos, 0x04)
```
## <a id='L-02'></a>L-02. Solidity version 0.8.3 doesn't support custom errors defined by `error` keyword.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L14-L17

https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L3

## Summary
The library `MathMasters` uses pragma version `^0.8.3`, which means that it should be usable by all versions `0.8.3` and above but custom errors are not supported in version `0.8.3` and it will not be usable by contracts of version `0.8.3`. 

## Vulnerability Details
The vulnerability occurs due to the use of custom errors with `^0.8.3` because if contract with version `0.8.3` are used then custom errors cannot be used as they are not supported by version `0.8.3`.

## Impact
Any contract of version `0.8.3` will not be able to use the library as custom errors are not supported.

## Tools Used
Manual Review

## Recommendations
Custom errors are supported from `0.8.4`, thus update the pragma version to `^0.8.4`
```diff
-pragma solidity ^0.8.3;
+pragma solidity ^0.8.4;
```
## <a id='L-03'></a>L-03. Incorrect error selector used while reverting in mulWad and mulWadUp            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L40

https://github.com/Cyfrin/2024-01-math-master/blob/main/src/MathMasters.sol#L53

## Summary
`MathMasters::mulWad` and `MathMasters::mulWadUp` uses incorrect error selector while reverting, leading to incorrect error display to user.

## Vulnerability Details
The vulnerability occurs due to the usage of incorrect error selectors in mulWad and mulWadUp function.
Both of them use `0xbac65e5b` as the error selector but it is not the error selector of mentioned error function: `MathMasters__MulWadFailed()`.

## Impact
Reverts with incorrect error.

## Tools Used
Manual Review

## Recommendations
Change the error selector with the correct one as below:
```diff
+bytes4 sel = MathMasters__MulWadFailed.selector;
-mstore(0x40, 0xbac65e5b) // `MathMasters__MulWadFailed()`.
+mstore(0x40, sel)
```


