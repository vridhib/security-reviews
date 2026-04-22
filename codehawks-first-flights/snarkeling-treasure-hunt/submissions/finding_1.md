# Duplicate Treasure Hash in Circuit Allows Only 9 Unique Treasures to Be Claimed, Permanently Blocking Normal Withdraw Functionality and Requiring Manual Intervention for Treasure Hunt Conclusions

## Description

The Noir circuit `main.nr` defines a global array `ALLOWED_TREASURE_HASHES` containing ten entries. This circuit is expected to contain exactly ten distinct allowed treasure hashes, enabling up to ten unique treasure claims before the hunt concludes normally. However, the last two entries (indices 8 and 9) are identical. As a result, two different treasure secrets map to the same on‑chain identifier. Once one treasure is claimed, the `claimed` mapping prevents the second treasure from ever being redeemed, permanently locking its associated reward.

```nr
global ALLOWED_TREASURE_HASHES: [Field; 10] = [
    // ...
@>  -961435057317293580094826482786572873533235701183329831124091847635547871092, // index 8
@>  -961435057317293580094826482786572873533235701183329831124091847635547871092  // index 9 (DUPLICATE)
];
```

## Risk

**Likelihood:**

The duplicate hash is coded into the deployed circuit and verifier. The condition is guaranteed to manifest when the tenth treasure is attempted to be claimed. No external factors or attacker actions are required as the bug is a static data corruption in the source code.

**Impact:**

There are four distinct impacts stemming from this duplicate entry issue. First, the `TreasureHunt::claim` function marks a treasure hash as claimed after a successful proof, but since the two hashes are identical, the second treasure will produce the same `treasureHash` public input. After the first treasure is redeemed, the claimed mapping prevents the second treasure from ever being claimed. Second, `TreasureHunt::withdraw` is permanently blocked due to it requiring `claimsCount >= MAX_TREASURES` to be true. Since the duplicated entry allows at most nine treasures to be claimed, the condition can never be satisfied—which leads to the third impact. Third, the owner is forced to use the `TreasureHunt::emergencyWithdraw` function as it provides the only recovery path. The owner can still recover the locked funds in a two-step process: call `TreasureHunt::pause` to halt new claims, and then invoke `emergencyWithdraw` to manually extract the remaining funds. However, this remediation bypasses the intended hunt conclusion mechanism and requires manual intervention, violating the protocol's expected autonomous operation. Finally, this affects the deployment artifacts by requiring a redeployment. The `Verifier.sol` contract is generated from the circuit's verification key, and the duplicate hash is embedded in the verifier's logic. Fixing this issue requires updating `main.nr` with a unique tenth hash, regenerating the verifier contract via the build script, and then redeploying a new verifier to update the existing `TreasureHunt` contract's verifier reference via `TreasureHunt::updateVerifier`.



## Proof of Concept

The vulnerability is visible directly in the Noir circuit source (`circuits/src/main.nr`). The array `ALLOWED_TREASURE_HASHES` contains a duplicate value at indices 8 and 9:

```nr
global ALLOWED_TREASURE_HASHES: [Field; 10] = [
    // ... first 8 entries
    -961435057317293580094826482786572873533235701183329831124091847635547871092, // index 8
    -961435057317293580094826482786572873533235701183329831124091847635547871092  // index 9 (DUPLICATE)
];
```

## Recommended Mitigation

It is recommended to replace the duplicate hash with the intended unique hash for the tenth treasure and verify that the corrected set contains exactly ten distinct values.

