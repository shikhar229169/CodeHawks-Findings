No check in proxy factory for the data to delegatecall the distribute function on Distributor through the Proxy contract.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/ProxyFactory.sol#L127

https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/ProxyFactory.sol#L152

https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/ProxyFactory.sol#L179

https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/ProxyFactory.sol#L205

https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/ProxyFactory.sol#L249

## Summary
The data is being sent to the ProxyFactory to perform distribution of winnings on the Proxy contract by doing delegate call on Distributor. But there is no check for the data whether the data has the function selector same as the distribute function on the Distributor contract, thus this can lead to calling of any other functions and no distribution of winning takes place.

## Vulnerability Details
When we are calling the function to deploy Proxy and distribute the rewards on the ProxyFactory contract, the data sent by organizer or the owner is not validated whether it is actually for calling the distribute function or not, so it can lead to calling of any other function, resulting in no distribution of winnings.

## PoC
Below is the PoC which was performed in the file ```FuzzTestProxyFactory.t.sol```

```solidity
function createWrongData() public pure returns (bytes memory data) {
    return abi.encodeWithSelector(Distributor.getConstants.selector);
}

function test_DoesNotRevert_If_WrongCalldataSent()
    public
    setUpContestForJasonAndSentJpycv2Token(organizer, jpycv2Address, 10000 ether, block.timestamp + 1 days) 
{
    bytes32 contestId = keccak256(abi.encode("Jason", "001"));
    bytes memory wrongData = createWrongData();   // this data doesn't invoke the distribute function, instead it contains the selector of another function of distributor contract.

    vm.warp(block.timestamp + 1 days);

    vm.startPrank(organizer);

    proxyFactory.deployProxyAndDistribute(contestId, address(distributor), wrongData);

    vm.stopPrank();
}
```

The above test passes, even the data doesn't invokes the distribute function call.

## Tools Used
Foundry Unit Test

## Recommendations
- To add a check that the first 4 bytes of data is equal to the function selector of distribute function.
```solidity
function distribute(address token, address[] memory winners, uint256[] memory percentages, bytes memory data)
```
- So we can compare the first 4 bytes of data with the function selector of the above distribute function and if they are not same we need to revert.
- But if we upgraded our implementation and there was a different function signature for the distribute function, so to solve this issue we can create a mapping which maps implementation(distributor) contract address to its correct distribute function selector.