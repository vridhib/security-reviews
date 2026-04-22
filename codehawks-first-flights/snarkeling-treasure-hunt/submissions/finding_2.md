# Missing Access Control in `TreasureHunt::withdraw` Enables Unauthorized Users to Trigger Owner Fund Withdrawal, Enabling Griefing Attacks

## Description

The missing access control in `TreasureHunt::withdraw` allows unauthorized (non-owner) users to trigger the fund withdrawal to the owner address after a treasure hunt has finished. The `withdraw` function is intended to be callable only by the contract `owner` after the conclusion of the current treasure hunt. The treasure hunt concludes upon the satisfaction of this condition: `claimsCount >= MAX_TREASURES`. However, the function suffers from lack of adequate access control, such as the `onlyOwner` modifier or an explicit `require(msg.sender == owner)`. This omission reveals a discrepancy between the documented intent (the NatSpec comment states "Allow the owner to withdraw...") and the actual implementation, which imposes no caller restrictions. As a result, after the hunt ends, any external account can invoke the `withdraw` function and force the contract's entire ETH balance to be sent to the `owner` address.

```js
    /// @notice Allow the owner to withdraw unclaimed funds after the hunt is over.
    function withdraw() external {
        require(claimsCount >= MAX_TREASURES, "HUNT_NOT_OVER");     

        uint256 balance = address(this).balance;
        require(balance > 0, "NO_FUNDS_TO_WITHDRAW");
        (bool sent, ) = owner.call{value: balance}("");
        require(sent, "ETH_TRANSFER_FAILED");

        emit Withdrawn(balance, address(this).balance);
    }   
```

## Risk

**Likelihood**:

This issue has a high likelihood of occurring. The `withdraw` function becomes callable by any address immediately after the final treasure is claimed (`claimsCount >= MAX_TREASURES`). Invoking this function requires no special permissions, capital, or technical expertise, so any external account can submit the transaction. Additionally, an attacker monitoring the mempool can front‑run the owner's intended withdrawal with a higher gas price, reliably executing the griefing attack. Even without front‑running, any curious or malicious user can call the function at any time after hunt has concluded.

**Impact**:

While the funds themselves are not stolen, this violates the principle of least privilege and opens the door to operational griefing. The lack of access control in the `withdraw` function enables a griefing attack with a four-fold impact. One, any user can prematurely force the owner to receive the contract's balance. This may occur at an inconvenient time, disrupting the operational cash flow. Two, if the owner intends to keep the funds in the contract (e.g., for a second treasure hunt), the attacker forces the owner to spend gas to first receive the funds and then re-deposit them via `TreasureHunt::fund`. An attacker can repeat the process each time the contract is re-funded and a hunt ends, resulting in cumulative gas waste. Three, an attacker monitoring the mempool can front-run the owner's own `withdraw` transaction. In doing so, the attacker's call succeeds, and the owner's transaction reverts with `NO_FUNDS_TO_WITHDRAW`, wasting the owner's gas and causing confusion. Four, although these funds ultimately return to the owner, the attack degrades the owner or team's control over their own protocol—and thus eroding trust in the protocol. To summarize, this issue is classified as medium severity because, while not resulting in permanent loss of funds (the owner can recover them), it enables a reliable griefing attack vector that disrupts intended protocol operations as well as imposes unnecessary gas costs to the owner.

## Proof of Concept

The following scenario outlines a realistic griefing attack that does not require any special privileges or ZK proof knowledge:

1. The attacker runs a simple script (or uses a block explorer) to watch the `Claimed` events emitted by the `TreasureHunt` contract. They maintain a counter of how many treasures have been claimed.
2. When the `claimsCount` reaches `MAX_TREASURES - 1`, the attacker prepares a transaction calling `withdraw()` with a competitive gas price.
3. As soon as the final `claim` transaction appears in the mempool, the attacker submits their `withdraw()` transaction. If it is mined before the owner's intended withdrawal, the attacker successfully empties the contract.
4. Even without front‑running, the attacker can simply call `withdraw()` at any moment after the final claim. The owner has no way to prevent this.
5. The owner receives the ETH (so funds are not lost), however:
   - If the owner planned to use the remaining balance for another purpose (e.g., a second treasure hunt), they must now spend gas to re‑deposit funds via `fund()`.
   - If the owner had submitted their own `withdraw()` transaction, it reverts, wasting that gas.
   - The owner loses control over the exact timing of fund withdrawal.

The provided Proof of Code simulates this exact scenario using a mock verifier to bypass ZK proof validation, demonstrating that any non‑owner address can successfully call `withdraw()` after 10 claims.

<details>
<summary>Proof of Code</summary>

Insert the following test in `TreasureHunt.t.sol`.

Before doing so insert the following import statement in the test file.

```js
import {IVerifier} from "../src/Verifier.sol";
```

```js
function testNonOwnerCanWithdrawAfterHuntEnds() public {
    // 1. Setup contracts
    MockVerifier mockVerifier = new MockVerifier();
    // Deal the owner two hunts worth of rewards
    uint256 ownerAmount = 200e18;
    vm.deal(owner, ownerAmount); // fund with 200 ETH
    vm.startPrank(owner);
    TreasureHunt treasureHuntWithMock = new TreasureHunt{value: ownerAmount}(address(mockVerifier));
    vm.stopPrank();

    // 2. Simulate 10 treasure claims
    uint256 recipientsNum = 10;
    address payable[] memory recipients = new address payable[](recipientsNum);
    bytes32[] memory treasureHashes = new bytes32[](recipientsNum);

    for (uint256 i = 0; i < recipientsNum; i++) {
        recipients[i] = payable(address(uint160(uint256(keccak256(abi.encodePacked(i, "recipient"))))));
        treasureHashes[i] = keccak256(abi.encodePacked("i", "treasure"));
        vm.prank(participant);
        treasureHuntWithMock.claim(new bytes(0), treasureHashes[i], recipients[i]);
    }

    // 3. Verify the hunt is over
    assertEq(treasureHuntWithMock.claimsCount(), recipientsNum);
    assertEq(treasureHuntWithMock.claimsCount(), treasureHuntWithMock.MAX_TREASURES());

    // 4. Attacker does a griefing attack (withdrawing funds to owner)
    uint256 contractBalanceBefore = address(treasureHuntWithMock).balance;
    uint256 ownerBalanceBefore = owner.balance;
    vm.prank(attacker);
    treasureHuntWithMock.withdraw();
    uint256 contractBalanceAfter = address(treasureHuntWithMock).balance;
    uint256 ownerBalanceAfter = owner.balance;

    // 5. Assert funds were successfully transferred to the owner (without permission)
    // Before hunt: 200 ETH
    // Rewards distributed were 10 * 10 ETH = 100 ETH
    // After hunt: 100 ETH (enough for next hunt)
    // After attacker withdraws: 0 ETH
    assertEq(contractBalanceAfter, 0);
    assertEq(ownerBalanceAfter, ownerBalanceBefore + contractBalanceBefore);
}
```

Place this mock verifier contract into `TreasureHunt.t.sol` as well.

```js
contract MockVerifier is IVerifier {
    function verify(bytes calldata _proof, bytes32[] calldata _publicInputs) external view returns (bool) {
        return true;
    }
}
```

</details>

## Recommended Mitigation

It is recommended to impose proper access controls on the `withdraw` function. For code clarity and readability, use the existing `onlyOwner` modifier as such:

```diff
-   function withdraw() external {
+   function withdraw() external onlyOwner {
        require(claimsCount >= MAX_TREASURES, "HUNT_NOT_OVER");     

        uint256 balance = address(this).balance;
        require(balance > 0, "NO_FUNDS_TO_WITHDRAW");
        (bool sent, ) = owner.call{value: balance}("");
        require(sent, "ETH_TRANSFER_FAILED");

        emit Withdrawn(balance, address(this).balance);
    }   
```

