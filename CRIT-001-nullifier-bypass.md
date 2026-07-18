# CRIT-001: `init_if_needed` Nullifier Bypass on Reissue Paths

## Severity
**CRITICAL**

## Affected Instructions
- `jperp_reissue_notes` (lib.rs:2830-2847)
- `jperp_recover_native` (lib.rs:2946-2963)
- `phoenix_reissue_notes` (lib.rs:2386-2404)
- `prediction_reissue` (lib.rs:3155-3172)

## Root Cause

The standard `transact` instruction uses Anchor's `init` constraint for nullifier marker accounts:

```rust
// lib.rs:1832-1848 (Transact) — CORRECT
#[account(
    init,                  // fails if account already exists, prevents replay
    payer = relayer,
    seeds = [b"nullifier_v3", mint_address.as_ref(), input_nullifier_0.as_ref()],
    ...
)]
pub nullifier_marker_0: Box<Account<'info, NullifierMarker>>,
```

However, all **reissue-path** instructions use `init_if_needed` instead:

```rust
// lib.rs:2831-2838 (JperpReissueNotes) — VULNERABLE
#[account(
    init_if_needed,        // does NOT fail if account already exists
    payer = relayer,
    seeds = [b"nullifier_v3", mint_address.as_ref(), input_nullifier_0.as_ref()],
    bump,
    space = NullifierMarker::LEN
)]
pub nullifier_marker_0: Box<Account<'info, NullifierMarker>>,
```

The `init_if_needed` constraint does not fail if the PDA account already exists — it opens the existing account silently. The handler's `require!(!marker.is_spent)` check is the only remaining defense, but the `init_if_needed` on **escrow/slot accounts** in the same reissue paths creates additional replay surface.

## Attack Scenario

1. Attacker deposits 100 USDC, calls `jperp_open_position`
   - Nullifiers N1, N2 spent. Executor holds 100 USDC.
2. Attacker calls `jperp_reissue_notes` (call #1):
   - `input_nullifier_0` = fresh dummy hash D1 (never seen before)
   - `input_nullifier_1` = fresh dummy hash D2 (never seen before)
   - `reissue_amount = 100 USDC`, valid ZK proof provided
   - `init_if_needed` creates D1, D2 markers (`is_spent = false` → passes check)
   - 100 USDC transferred: executor → vault. Commitments inserted.
3. Attacker **still has executor with 100 USDC** (step 2 did not drain executor!) OR calls `jperp_reissue_notes` again with new dummy nullifiers D3, D4.
   - Result: **double-minted notes**, attacker extracted 200 USDC on 100 deposited.

The stated rationale for `init_if_needed` is "deposit circuit reuses dummy zero-value witnesses." However, fresh Poseidon hashes of random witnesses produce unique nullifiers per call — `init` is always safe and correct.

## Impact
- Proof replay → double-minted private notes
- Vault depletion via accumulated reissues beyond deposited amount
- Fund theft from the privacy pool

## Remediation

Change `init_if_needed` to `init` for ALL reissue nullifier markers and escrow accounts:

```rust
#[account(
    init,           // NOT init_if_needed
    payer = relayer,
    seeds = [b"nullifier_v3", mint_address.as_ref(), input_nullifier_0.as_ref()],
    bump,
    space = NullifierMarker::LEN
)]
pub nullifier_marker_0: Box<Account<'info, NullifierMarker>>,
```

## References
- [Akash1070/Audit-VeiloLayer - CRIT-001](https://github.com/Akash1070/Audit-VeiloLayer/tree/main/findings/CRIT-001-nullifier-bypass.md)
- Program source: [VeiloSolana/privacy-program/lib.rs](https://github.com/VeiloSolana/privacy-program/blob/main/programs/privacy-pool/src/lib.rs)