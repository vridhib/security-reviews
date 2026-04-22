## Medium

### [M-1] Duplicate Treasure Hash in Circuit Allows Only 9 Unique Treasures to Be Claimed, Permanently Blocking Normal Withdraw Functionality and Requiring Manual Intervention for Treasure Hunt Conclusions

**Description:**

The Noir circuit `main.nr` defines a global array `ALLOWED_TREASURE_HASHES` containing 10 entries. However, the last two entries (indices 8 and 9) are identical. As a result, two different treasure secrets produce the same on‑chain identifier. Once one is claimed, the `claimed` mapping prevents the second from ever being redeemed, permanently locking its associated reward.

**Impact:**

There are four distinct impacts stemming from this duplicate entry issue. First, the `TreasureHunt::claim` function marks a treasure hash as claimed after a successful proof, but since the two hashes are identical, the second treasure will produce the same `treasureHash` public input. After the first is redeemed, the claimed mapping prevents the second from ever being claimed. Second, `TreasureHunt::withdraw` is permanently blocked, since it requires `claims >= MAX_TREASURES` to be true. Since the duplicated entry allows at most nine treasures to be claimed, the aforementioned condition can never be satisfied--which leads to the third impact. Third, the owner is forced to use the `TreasureHunt::emergencyWithdraw` function as it provides the only recovery path. The owner can still recover the locked funds in a two-step process: call `TreasureHunt::pause` to halt new claims, and then invoke `emergencyWithdraw` to manually extract the remaining funds. However, this remediation bypasses the intended hunt conclusion mechanism and requires manual intervention, violating the protocol's expected autonomous operation. Finally, this affects the deployment artifacts by requiring a redeployment. The `Verifier.sol` contract is generated from the circuit's verification key, and the duplicate hash is embedded in the verifier's logic. Fixing this issue requires updating `main.nr` with a unique 10th hash, regenerating the verifier contract via the build script, and then redeploying a new verifier to update the existing `TreasureHunt` contract's verifier reference via `TreasureHunt::updateVerifier`.

**Proof of Concept:**

The vulnerability is visible directly in the Noir circuit source (`circuits/src/main.nr`). The array `ALLOWED_TREASURE_HASHES` contains a duplicate value at indices 8 and 9:

```nr
global ALLOWED_TREASURE_HASHES: [Field; 10] = [
    // ... first 8 entries
    -961435057317293580094826482786572873533235701183329831124091847635547871092, // index 8
    -961435057317293580094826482786572873533235701183329831124091847635547871092  // index 9 (DUPLICATE)
];
```

**Recommended Mitigation:**

Replace the duplicate hash with the intended unique hash for the tenth treasure. Verify the corrected set contains exactly 10 distinct values.


### [M-2] Missing Access Control in `TreasureHunt::withdraw` Enables Unauthorized Users to Trigger Owner Fund Withdrawal, Enabling Griefing Attacks

**Description:**

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

**Impact:**

While the funds themselves are not stolen, this violates the principle of least privilege and opens the door to operational griefing. The lack of access control in the `withdraw` function enables a griefing attack with a four-fold impact. One, any user can prematurely force the owner to receive the contract's balance. This may occur at an inconvenient time, disrupting the operational cash flow. Two, if the owner intends to keep the funds in the contract (e.g., for a second treasure hunt), the attacker forces the owner to spend gas to first receive the funds and then re-deposit them via `TreasureHunt::fund`. An attacker can repeat the process each time the contract is re-funded and a hunt ends, resulting in cumulative gas waste. Three, an attacker monitoring the mempool can front-run the owner's own `withdraw` transaction. In doing so, the attacker's call succeeds, and the owner's transaction reverts with `NO_FUNDS_TO_WITHDRAW`, wasting the owner's gas and causing confusion. Four, although these funds ultimately return to the owner, the attack degrades the owner or team's control over their own protocol--and thus eroding trust in the protocol.

To summarize, this issue is classified as medium severity because, while not resulting in permanent loss of funds (the owner can recover them), it enables a reliable griefing attack vector that disrupts intended protocol operations as well as imposes unnecessary gas costs to the owner.


**Proof of Concept:**

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

    // 3. Verify hunt is over
    assertEq(treasureHuntWithMock.claimsCount(), recipientsNum);
    assertEq(treasureHuntWithMock.claimsCount(), treasureHuntWithMock.MAX_TREASURES());

    // 4. Attacker does a griefing attack (withdrawing funds)
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

**Recommended Mitigation:**

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



## Low

### [L-1] Inconsistent Access Control Between `TreasureHunt::fund()` and `TreasureHunt::receive()` Allows Unrestricted ETH Donations

**Description:**

The `fund()` function is restricted to the `owner` and in the NatSpec documentation is described as: "Allow the owner to add more funds if needed." This expected behavior is also tested for in `TreasureHunt.t.sol` in the existing `testNonOwnerCannotFund` test. However, the contract also implements a `receive()` fallback function that allows any external account or smart contract to send ETH directly to the contract, emitting the same `Funded` event. This creates an inconsistency between the documented protocol intent and the actual implemented behavior. 

```js
    receive() external payable {
        emit Funded(msg.value, address(this).balance);  
    }
```

**Impact:**

The discrepancy between the actual implementation and the documented implementation results in minor impacts. For one, the protocol's intended funding mechanism (owner-only) is able to be bypassed, which can lead to operational confusion. Another such attack, albeit negligible and unlikely, is that an attacker can repeatedly send dust amounts to bloat event logs and slightly increase gas costs for the owner during withdrawals.

**Proof of Concept:**

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

**Recommended Mitigation:**

As per the protocol's documentation, it explicitly states that only the owner should be able to fund the contract. Therefore, it is recommended to remove the `receive` function entirely. This ensures that only the owner can intentionally add funds via `fund`. Users who accidentally send ETH to the contract address will have their transactions revert, preventing unintended donations.

```diff
-   receive() external payable {
-       emit Funded(msg.value, address(this).balance);
-   }
```

*Note: This does not prevent forced ETH transfers via SELFDESTRUCT, but that is an unavoidable EVM behavior and does not represent an intentional funding path.*




