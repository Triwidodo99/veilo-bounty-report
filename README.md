# VeiloLayer Privacy Pool - Security Audit Report

## Program Information
- **Program ID:** `GYy4kM6GHhpgLCUscuABbzkD2ZbJ2fneYryaZ6Ch7fFU`
- **Repository:** [VeiloSolana/privacy-program](https://github.com/VeiloSolana/privacy-program)
- **Target Scope:** privacy_pool Solana smart contracts (Anchor framework)
- **Chain:** Solana
- **Bounty Program:** [Superteam Earn - veilo-bounty](https://superteam.fun/earn/listing/veilo-bounty)
- **Reward:** $2,000 USDC (fixed, CRITICAL severity)
- **Auditor:** Tri Widodo S.Pd
- **Audit Date:** July 18, 2026

---

## Executive Summary

A low-level security review was conducted on the VeiloLayer `privacy_pool` Anchor program. The scope focused on vulnerabilities causing direct fund loss, double-spending, unauthorized pool depletion, and ZK-SNARK verification bypasses.

**2 CRITICAL** and **1 MEDIUM** severity vulnerabilities were identified and documented below.

---

## Findings Summary

| ID | Severity | Title | Impact |
|----|----------|-------|--------|
| CRIT-001 | CRITICAL | `init_if_needed` Nullifier Bypass on Reissue Paths | Proof replay leading to double-minted private notes and vault depletion |
| CRIT-002 | CRITICAL | Relayer-Float Path Skips Vault Debit | TVL accounting inflation, phantom pool capacity, vault fund theft |
| MED-001 | MEDIUM | `reduce_to_field` Off-By-One - FR_MODULUS Equality Bypass | Degenerate zero element in ZK public inputs, circuit equation bypass |

---

## Detailed Findings

- [CRIT-001: init_if_needed Nullifier Bypass on Reissue Paths](./CRIT-001-nullifier-bypass.md)
- [CRIT-002: Relayer-Float Path Skips Vault Debit](./CRIT-002-relayer-float-vault-debit.md)
- [MED-001: reduce_to_field Off-By-One on FR_MODULUS Equality](./MED-001-reduce-to-field-off-by-one.md)

---

## Scope

### In-Scope Contracts
- `programs/privacy-pool/src/lib.rs` - Main entry, transact, config, admin functions
- `programs/privacy-pool/src/swap.rs` - Cross-pool swap logic
- `programs/privacy-pool/src/perps.rs` - Jupiter Perps integration
- `programs/privacy-pool/src/phoenix.rs` - Phoenix exchange integration
- `programs/privacy-pool/src/positions.rs` - Position pool management
- `programs/privacy-pool/src/zk.rs` - Groth16 proof verification
- `programs/privacy-pool/src/merkle_tree.rs` - Merkle tree operations

### Out-of-Scope
- Frontend/web applications
- Third-party Jupiter/Raydium/Phoenix protocols
- Client-side note generation
- ZK circuit logic (assumed correct)
