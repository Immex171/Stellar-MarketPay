# Escrow Core v2: modular lifecycle, milestone templates, streaming, and safe migration

## Summary

This PR restructures the MarketPay escrow contract into explicit domain modules while preserving every existing public entrypoint through `lib.rs` as a compatibility facade. It replaces scattered lifecycle checks with one audited transition table, introduces reusable named milestone templates and continuous streaming settlement, and adds lazy v1-to-v2 migration with a tested rollback path.

It also fixes the backend migration journal so migrations that share a numeric version are tracked by `(version, name)` and each named migration runs exactly once.

## What was fixed

### Contract structure and lifecycle safety

- Split escrow responsibilities into `escrow`, `state_machine`, `milestones`, `streaming`, `migration`, `multisig`, `arbitration`, `referrals`, `ratings`, and `certificates` modules.
- Kept `lib.rs` as the ABI-compatible entrypoint facade so existing integrations continue to work during migration.
- Replaced repeated status checks with a central transition table covering `Locked`, `Active`, `Paused`, `Disputed`, `Released`, `Refunded`, and `Cancelled` states.
- Added exhaustive tests for all 7 states and 11 lifecycle actions so unsupported transitions are rejected in one place.

### Milestone templates and amendments

- Replaced the fixed, unnamed milestone shape for v2 flows with reusable templates containing:
  - milestone names;
  - acceptance-criteria hashes;
  - amounts; and
  - per-milestone deadlines.
- Added immutable escrow snapshots so later template reuse does not mutate active agreements.
- Added mid-engagement amendment proposals that activate only after independent client and freelancer authorization.
- Preserved completed and already-paid value when validating amendments.
- Enforced and benchmarked a maximum of 20 milestones per template.

### Streaming settlement

- Added per-ledger linear streaming from client to freelancer over a defined ledger window.
- Added withdrawal of accrued funds without ending the stream.
- Added pause, resume, cancel, and dispute handling with accrual checkpointed at each action.
- Made disputes stop accrual atomically.
- Made cancellation pay accrued value to the freelancer and return the exact unvested remainder to the client.
- Used cumulative entitlement rather than a rounded per-ledger rate:

  ```text
  vested(t) = floor(total * active_elapsed_ledgers(t) / duration_ledgers)
  withdrawable(t) = vested(t) - withdrawn
  ```

- Added a no-dust test that streams 101 units over 100 ledgers and withdraws on every ledger; the final accounting conserves every unit.

### Storage migration and rollback

- Added additive v2 storage keys rather than overwriting v1 records.
- Added lazy, idempotent forward migration on first v2 access.
- Stored the exact decoded v1 record as a rollback backup before writing the v2 representation.
- Defined deterministic mappings for v1 escrows in `Locked`, `InProgress`, `Disputed`, `Released`, and `Refunded` states.
- Kept migrated v1 escrows in discrete settlement mode; v2-only fields are introduced only through explicit v2 entrypoints.
- Added an upgrade test that writes a v1-shaped escrow, migrates it lazily, starts work, and settles it correctly under v2.
- Added `rollback_escrow_v2` plus `scripts/rollback-escrow-v2.sh` to restore pristine, still-v1-representable records before reinstalling the previous v1 WASM.
- Added an on-chain guard that prevents lossy rollback after streaming, amendments, or other v2-only behavior has been used.

### Backend migration reliability

- Changed `schema_migrations` from a version-only primary key to `(version, name)`.
- Fixed the case where two different migration files shared a version number and the second file was silently skipped.
- Updated migration detection, duplicate handling, and rollback selection to operate on the exact migration name.

## Compatibility and operational impact

- Existing contract entrypoint names, parameters, and return types remain available.
- New v2 entrypoints and storage keys are additive.
- Legacy escrows migrate only when accessed through the v2 path; the upgrade transaction does not perform an unbounded storage scan.
- Deployment should initially keep direct v2 creation disabled during the rollback observation window.
- Pristine lazy migrations can be rolled back to the exact v1 backup. Records that have used v2-only features must remain on a known-good v2 WASM and use a forward fix.
- The migration journal schema upgrade is performed in place and supports repositories with multiple same-version migration files.

## Resource measurements

All v2 entrypoints were measured against the closest v1 operation. The maximum supported 20-item template was used for template and amendment measurements.

- Most measured v2 calls cost between `0.29x` and `0.99x` of their nearest v1 comparator.
- Maximum-size amendment approval costs `1.04x` the closest v1 comparator because it validates both approvals, preserves completed value, checks conservation, and writes both legacy and v2 views atomically.
- Full CPU and memory results are documented in `docs/ESCROW_V2_RESOURCE_BASELINE.md`.

## Verification

- [x] Existing public entrypoints remain available through the compatibility facade.
- [x] Illegal transitions are centrally rejected by the transition table.
- [x] Streaming withdrawal, pause, resume, cancellation, and dispute behavior are covered end to end.
- [x] No-dust accounting holds across many partial withdrawals.
- [x] A pre-upgrade v1-shaped escrow lazily migrates and settles under v2.
- [x] Balance conservation and at-most-once settlement hold across 10,000 randomized state-machine traces with 64 actions per trace.
- [x] Negative authorization coverage exists for every v2 state-changing entrypoint.
- [x] Every v2 entrypoint and the 20-milestone upper bound have resource measurements.
- [x] Rollback behavior and the executable rollback script are tested.

### Commands and results

```text
cargo test --features std
  59 unit + 11 differential + 1 fuzz + 10 regression
  + 8 v2 integration + 2 v2 property tests passed

cargo clippy --all-targets --features std -- -D warnings
  passed

cargo check --target wasm32-unknown-unknown
  passed

cargo build --target wasm32-unknown-unknown --release
  passed

cargo audit --ignore RUSTSEC-2020-0071
  0 vulnerabilities; 2 explicitly allowed unmaintained warnings

Frontend Jest
  9 suites, 76 tests, and 36 snapshots passed

Backend Jest with PostgreSQL and Redis
  52 suites and 508 tests passed
```

The pre-push CI gate also passed the frontend suite, backend unit suite, contract unit/integration suite, invariant fuzz test, regression tests, and v2 property tests.

## Review guide

Recommended review order:

1. `docs/ESCROW_V2_DESIGN_COMMENT.md` — architecture, accounting, migration semantics, and rollout sequence.
2. `contracts/marketpay-contract/src/state_machine.rs` — central lifecycle rules.
3. `contracts/marketpay-contract/src/escrow.rs` and `src/migration.rs` — v1/v2 records and lazy migration.
4. `contracts/marketpay-contract/src/milestones.rs` and `src/streaming.rs` — new settlement modes.
5. `contracts/marketpay-contract/tests/v2_escrow.rs` and `tests/v2_properties.rs` — integration and invariant coverage.
6. `docs/ESCROW_V2_RESOURCE_BASELINE.md` — resource comparison.
7. `scripts/rollback-escrow-v2.sh` — operational rollback procedure.

## Type of change

- [x] Bug fix
- [x] New feature
- [x] Documentation
- [x] Refactor or maintenance
- [x] Smart contract change

## Scope and impact

- [ ] Frontend
- [x] Backend or API
- [x] Soroban contract
- [x] Database or migration
- [x] Documentation

## Screenshots

Not applicable; this PR has no user-interface changes.

## Supporting artifacts

- Design: `docs/ESCROW_V2_DESIGN_COMMENT.md`
- Resource baseline: `docs/ESCROW_V2_RESOURCE_BASELINE.md`
- Verification record: `artifacts/escrow-v2-verification.txt`
- Original v1 hash: `artifacts/escrow-v1-original.sha256`
- Reproducible patch: `artifacts/escrow-v2.patch.gz`
- Rollback runbook: `scripts/rollback-escrow-v2.sh`
