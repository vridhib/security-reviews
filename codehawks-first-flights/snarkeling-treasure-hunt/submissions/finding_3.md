# Inconsistent Access Control Between `TreasureHunt::fund()` and `TreasureHunt::receive()` Allows Unrestricted ETH Donations


## Description

The `fund()` function is restricted to the `owner` and in the NatSpec documentation is described as: "Allow the owner to add more funds if needed." This expected behavior is also tested for in `TreasureHunt.t.sol` in the existing `testNonOwnerCannotFund` test. However, the contract also implements a `receive()` fallback function that allows any external account or smart contract to send ETH directly to the contract, emitting the same `Funded` event. This creates an inconsistency between the documented protocol intent and the actual implemented behavior. 

```js
receive() external payable {
    emit Funded(msg.value, address(this).balance);  
}
```

## Risk

**Likelihood**:

This has a medium likelihood as anyone can send ETH directly to the contract address. This would not be a targeted attack as it could happen accidentally or via a curious user.

**Impact**:

The discrepancy between the actual implementation and the documented implementation results in minor impacts. For one, the protocol's intended funding mechanism (owner-only) is able to be bypassed, which can lead to operational confusion. Another such attack, albeit negligible and unlikely, is that an attacker can repeatedly send dust amounts to bloat event logs and slightly increase gas costs for the owner during withdrawals.

## Proof of Concept

Any user can send ETH directly to the contract address via `transfer`, `send`, or `call`, triggering the `receive` function and emitting a `Funded` event, despite not being the owner.

The provided Proof of Code demonstrates that any non-owner address can successfully send any amount of ETH to the `TreasureHunt` contract.

<details> 
<summary>Proof of Code</summary>

Insert the following test into `TreasureHunt.t.sol`.
```js
function testNonOwnerCanFundContract() public {
        uint256 contractBalanceBefore = address(hunt).balance;
        vm.prank(participant);
        uint256 amountToSend = 1 wei;
        (bool success,) = address(hunt).call{value: amountToSend}("");
        require(success);
        assertEq(address(hunt).balance, contractBalanceBefore + amountToSend);
}
```

</details>

## Recommended Mitigation

As per the protocol's documentation, it explicitly states that only the owner should be able to fund the contract. Therefore, it is recommended to remove the `receive` function entirely. This ensures that only the owner can intentionally add funds via `fund`. Users who accidentally send ETH to the contract address will have their transactions revert, preventing unintended donations.

```diff
-   receive() external payable {
-       emit Funded(msg.value, address(this).balance);
-   }
```

*Note: This does not prevent forced ETH transfers via SELFDESTRUCT, but that is an unavoidable EVM behavior and does not represent an intentional funding path.*

