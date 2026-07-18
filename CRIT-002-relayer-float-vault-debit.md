# CRIT-002: Relayer-Float Path Skips Vault Debit

## Severity
**CRITICAL**

## Affected Instruction
- `transact_swap` (swap.rs, lines ~390-466)

## Root Cause

The `transact_swap` instruction has two funding modes:

1. **Pre-funded path** (`is_prefunded == 1`): `fund_native_source` debits the vault before swap
2. **Relayer-float path** (`is_prefunded == 0`): Relayer provides their own SOL to fund the executor ATA, then recoup via swap output

In the relayer-float path (swap.rs:399-428), the code transfers SOL from the **relayer** to the executor:

```rust
anchor_lang::system_program::transfer(
    CpiContext::new(system_program, Transfer {
        from: ctx.accounts.relayer.to_account_info(),   // dari relayer
        to: executor.to_account_info(),                  // ke executor
        // vault TIDAK di-debit pada path ini!
    }),
    swap_amount
)?;
```

The vault is **never debited** in the relayer-float path. The `config.total_tvl` counter is also not updated, creating a mismatch between on-chain TVL state and actual vault lamports.

## Attack Scenario

1. Pool has 1000 SOL actual vault balance, TVL reports 1000 SOL
2. Attacker (as whitelisted relayer) initiates a large swap via relayer-float path with `swap_amount = 800 SOL`
3. Relayer's 800 SOL moves to executor, swap executes correctly, output returned to pool
4. But the vault itself was never debited — the actual vault may now hold 1000 SOL + output tokens, while the vault should have been debited 800 SOL at step 2
5. TVL is now **inflated** by 800 SOL (phantom capacity)
6. Subsequent withdrawals/operations based on TVL calculations are vulnerable

Alternatively, if the swap output > swap input:
- Attacker extracts more than vault's actual net contribution
- Difference is paid from pool's existing TVL

## Impact
- TVL accounting inflation → phantom pool capacity
- Vault fund theft via incorrect accounting
- Incorrect fee calculations based on wrong TVL

## Remediation

Add vault debit in the relayer-float path, mirroring the pre-funded path:

```rust
// In relayer-float path, add vault debit:
anchor_lang::system_program::transfer(
    CpiContext::new_with_signer(
        system_program,
        Transfer { from: vault.to_account_info(), to: executor.to_account_info() },
        &[vault_seeds...]
    ),
    swap_amount
)?;
```

Also update `config.total_tvl` consistently across both paths.

## References
- [Akash1070/Audit-VeiloLayer - CRIT-002](https://github.com/Akash1070/Audit-VeiloLayer/tree/main/findings/CRIT-002-relayer-float-vault-debit.md)
- Program source: [VeiloSolana/privacy-program/swap.rs](https://github.com/VeiloSolana/privacy-program/blob/main/programs/privacy-pool/src/swap.rs)