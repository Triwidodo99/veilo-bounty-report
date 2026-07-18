# MED-001: `reduce_to_field` Off-By-One — FR_MODULUS Equality Bypass

## Severity
**MEDIUM**

## Affected Instructions
All instructions using `ExtData` / `ext_data_hash` as ZK public inputs:
- `transact` (lib.rs)
- `transact_swap` (swap.rs)
- `jperp_reissue_notes`, `jperp_recover_native` (perps.rs)
- `phoenix_reissue_notes` (phoenix.rs)
- `prediction_reissue` (predictions.rs)

## Root Cause

The `reduce_to_field` function in `lib.rs` has an off-by-one error in its comparison loop:

```rust
fn reduce_to_field(bytes: [u8; 32]) -> [u8; 32] {
    const FR_MODULUS: [u8; 32] = [
        0x30, 0x64, 0x4e, 0x72, 0xe1, 0x31, 0xa0, 0x29,
        0xb8, 0x50, 0x45, 0xb6, 0x81, 0x81, 0x58, 0x5d,
        0x28, 0x33, 0xe8, 0x48, 0x79, 0xb9, 0x70, 0x91,
        0x43, 0xe1, 0xf5, 0x93, 0xf0, 0x00, 0x00, 0x01,
    ];

    let mut needs_reduction = false;
    for i in 0..32 {
        if bytes[i] < FR_MODULUS[i] {
            break;                                     // case 1: bytes < modulus
        }
        if bytes[i] > FR_MODULUS[i] {
            needs_reduction = true;
            break;                                     // case 2: bytes > modulus
        }
        // case 3: bytes[i] == FR_MODULUS[i] for ALL i
        // → loop completes without triggering either branch
        // → needs_reduction stays FALSE
    }

    if !needs_reduction {
        return bytes;  // ← When bytes == FR_MODULUS, returns UNCHANGED
    }

    // ... reduction logic ...
}
```

**The bug:** When `bytes == FR_MODULUS` exactly, the loop iterates through all 32 bytes with equality at each position. Neither `break` condition is hit. `needs_reduction` remains `false`. The function returns `FR_MODULUS` unchanged — an **invalid BN254 field element**.

The BN254 Fr field requires all elements to be strictly less than the modulus. `FR_MODULUS` itself is not a valid field element.

## Attack Scenario

1. Attacker constructs `ExtData` such that `ext_data.hash()` returns exactly `FR_MODULUS`
   - This is achievable if the Poseidon hash of the bytes equals the modulus value (any value `v` where `hash(v) ≡ FR_MODULUS (mod FR_MODULUS)`)
   - Since `hash(bytes) mod FR_MODULUS = bytes` when `bytes == FR_MODULUS`, the attacker needs `hash(bytes) = FR_MODULUS` or any value congruent to it
2. `ext_data_hash = FR_MODULUS` (invalid field element) is passed to the ZK proof as `ext_data_hash`
3. If the circuit does not explicitly constrain `ext_data_hash ≠ 0` or `ext_data_hash < FR_MODULUS`:
   - The circuit may receive a degenerate input
   - For private transfers (`public_amount = 0`): circuit equation `sumIns + 0 = sumOuts` becomes trivially satisfiable with zero-value commitments
   - Attacker forges proof with fake commitments that balance to zero publicly
   - Attacker withdraws without proper value transfer through the pool

## Impact
- ZK proof verification may accept invalid public inputs
- Potential withdrawal without proper value transfer (depends on circuit constraints)
- Confidentiality breach of the privacy pool

## Remediation

Add an explicit check after the comparison loop to catch the `== FR_MODULUS` case:

```rust
let mut needs_reduction = false;
let mut all_equal = true;
for i in 0..32 {
    if bytes[i] < FR_MODULUS[i] {
        break;
    }
    if bytes[i] > FR_MODULUS[i] {
        needs_reduction = true;
        break;
    }
    // bytes[i] == FR_MODULUS[i], continue
}

if !needs_reduction {
    // Additional check: if bytes == FR_MODULUS exactly, reduce to 0
    return [0u8; 32];  // canonical reduction
}
```

Or add a check at the start of the function:

```rust
if bytes == FR_MODULUS {
    return [0u8; 32];  // canonical zero
}
```

## References
- Related finding: [Akash1070/Audit-VeiloLayer - MED-001](https://github.com/Akash1070/Audit-VeiloLayer/tree/main/findings/MED-001-reduce-to-field-off-by-one.md)
- Program source: [VeiloSolana/privacy-program/lib.rs](https://github.com/VeiloSolana/privacy-program/blob/main/programs/privacy-pool/src/lib.rs) (reduce_to_field function)