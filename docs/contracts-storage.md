# Contracts Storage Layout & TTL Policy

This document describes the on-chain storage keys, value shapes, and the
time-to-live / bump strategy used by the Soroban smart contract in the
`Talenttrust-Contracts` repository (`contracts/predictify-hybrid/`).

Scope: **on-chain Soroban contract storage only**.  The backend SQLite
persistence layer for contracts (the `ContractRepository` in
`repositories/contractRepository.ts`) is documented separately in
[contracts-flow.md](contracts-flow.md) under "Persistence Layer".

---

## Overview

| Aspect | Value |
|--------|-------|
| Contract package | `predictify-hybrid` |
| Storage kind | Soroban **Instance** storage (`env.storage().instance()`) |
| Defined in | [storage.rs](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/storage.rs) |
| Used by | `place_bets` in [bets.rs](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/bets.rs#L54-L98) |
| Public re-exports | `DataKey`, `IDEM_KEY_TTL_LEDGERS` in [lib.rs](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/lib.rs#L22) |

All storage operations go through the Soroban host's instance storage API
because the idempotency sentinels are contract-instance-scoped state that
must survive across invocations but does not need the persistence guarantees
of **Persistent** storage.

---

## Storage Key — `DataKey` Enum

The contract defines a single `#[contracttype]` enum that enumerates every
storage key the contract may ever write.  Adding a new storage field means
adding a new variant here; do **not** ad-hoc string keys elsewhere.

Defined at [storage.rs#L18-L23](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/storage.rs#L18-L23):

```rust
#[contracttype]
#[derive(Clone)]
pub enum DataKey {
    PlaceBetsIdem(Address, BytesN<32>),
}
```

### Variant details

| Variant | Components | Semantic key |
|---------|-----------|--------------|
| `PlaceBetsIdem` | `(caller: Address, token: BytesN<32>)` | Idempotency sentinel for a `place_bets` batch submitted by `caller` using the caller-chosen 32-byte `token`. |

**Composite-key design rationale** (documented in
[storage.rs#L15-L17](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/storage.rs#L15-L17)):
the idempotency token is bound to the submitting `Address` so that two
different callers may independently reuse the same 32-byte token without
colliding.  A caller cannot "poison" another user's idempotency space by
picking a token that the victim has already scheduled for use.

---

## Value Shapes

Stored values for each key variant:

| Key variant | Value type | Literal value | Source |
|-------------|-----------|---------------|--------|
| `PlaceBetsIdem(_, _)` | `bool` | Always `true` | [bets.rs#L82](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/bets.rs#L82): `env.storage().instance().set(&idem_key, &true);` |

A boolean sentinel is sufficient because the key's existence **is** the
relevant fact — the value only needs to be a small, cheap-to-serialize
inhabitant.  If richer per-batch metadata is ever needed (e.g., digest of
the bets vector, ledger height of consumption), this value can be upgraded
to a struct without changing the key shape.

### `Bet` payload shape (not stored, but context for the batch)

Each element of the `bets: Vec<Bet>` argument processed by `place_bets` has
the shape in [bets.rs#L11-L18](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/bets.rs#L11-L18):

```rust
#[soroban_sdk::contracttype]
#[derive(Clone)]
pub struct Bet {
    pub market_id: u64,
    pub amount: i128,   // in stroops
}
```

**Note:** As of this revision, the bet payload itself is **not** written to
contract storage.  `place_bets` emits a diagnostic event (`Symbol::new("place_bets")`)
and returns — market-state mutations are deferred to a future market-storage
module.  See the `TODO` at [bets.rs#L91-L93](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/bets.rs#L91-L93).

---

## TTL Policy

### Idempotency key TTL constant

All idempotency sentinels share a single TTL expressed in **ledgers**
(Soroban's native unit of time, ~5 s each on the public Stellar network).

Constant defined at [storage.rs#L10](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/storage.rs#L10):

```rust
pub const IDEM_KEY_TTL_LEDGERS: u32 = 17_280; // ~24 h at 5 s/ledger
```

| Property | Value |
|----------|-------|
| Ledgers | 17 280 |
| Wall-clock @ 5 s/ledger | ≈ 24 hours |
| Purpose | Replay-protection window — a re-submission with the same `(caller, token)` pair within this window is rejected. |

### Bump / extend strategy

The TTL is set **exactly once**, at the moment the key is first written.
There is **no read-triggered renewal**; the key is eligible for eviction
after the window elapses regardless of how many times the contract reads
it in the interim.

The exact call at [bets.rs#L83-L85](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/bets.rs#L83-L85):

```rust
env.storage()
    .instance()
    .extend_ttl(IDEM_KEY_TTL_LEDGERS, IDEM_KEY_TTL_LEDGERS);
```

Soroban's `extend_ttl` signature is `extend_ttl(threshold: u32, extend_to: u32)`,
meaning: "if the current remaining TTL is ≤ `threshold`, bump it so that it
lives for at least `extend_to` more ledgers from the current ledger".

| Argument | Value | Meaning |
|----------|-------|---------|
| `threshold` | `IDEM_KEY_TTL_LEDGERS` (17 280) | Only bump if TTL is already ≤ 24 h.  Since this call immediately follows `set`, the condition is always true on first write. |
| `extend_to` | `IDEM_KEY_TTL_LEDGERS` (17 280) | Set the remaining lifetime to exactly 24 h from now. |

### Semantics of expiry

- **Within the TTL window** (0 to 17 280 ledgers after write):
  `env.storage().instance().has(&idem_key)` returns `true` and any
  re-submission with the same `(caller, token)` pair returns
  `Error::IdempotentBatchAlreadyApplied`.  See [bets.rs#L76-L78](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/bets.rs#L76-L78).

- **After the TTL window** (> 17 280 ledgers after write):
  The entry is eligible for host eviction.  A re-submission with the same
  token is treated as a **fresh batch** — the `has()` check returns
  `false`, the key is re-written with a fresh 24-h TTL, and the bets are
  re-applied.  This behavior is enforced by the integration test
  `same_key_accepted_after_ttl_expiry` in
  [batch_operations_tests.rs#L121-L139](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/batch_operations_tests.rs#L121-L139).

### Bump policy: write-once, no renewal

| Operation | TTL bumped? |
|-----------|-------------|
| First `place_bets` with a fresh `(caller, token)` | ✅ Yes — set + extend_ttl at write time. |
| Repeat read of `has(&idem_key)` (duplicate reject path) | ❌ No — TTL is left as-is, expires on schedule. |
| Any other contract operation | ❌ No — only `place_bets` writes to instance storage today. |

Rationale: the idempotency entry is a replay-protection guard, not a LRU or
hot datum.  A 24-hour window is long enough to cover all reasonable
client-side retry / outbox replay scenarios, and the fixed expiry ensures
the storage set does not grow unboundedly over the lifetime of the
contract instance.

---

## Idempotency Write Ordering

The idempotency sentinel is written **before** the batch is applied
([bets.rs#L80-L85](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/bets.rs#L80-L85)
vs. [bets.rs#L91-L97](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/bets.rs#L91-L97)).
This ordering is security-sensitive:

1. **Check-then-set** of the sentinel makes concurrent invocations (same
   ledger, same `(caller, token)`) fail fast against each other rather
   than both observing a "not present" state and double-applying.
2. Because Soroban applies a single invocation atomically within one
   ledger, a partial-success scenario (sentinel written, batch failed to
   apply) is **not possible** inside one transaction.  If the transaction
   aborts, the sentinel write is rolled back along with everything else.

---

## Deprecated Zero-Key Path

A caller-supplied idempotency token of all zeroes
(`[0u8; 32]`, i.e. `BytesN::from_array(env, &[0u8; 32])`) short-circuits
the entire idempotency check.  The check at
[bets.rs#L72-L73](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/bets.rs#L72-L73)
skips the sentinel read/write entirely:

```rust
let zero_key: BytesN<32> = BytesN::from_array(env, &[0u8; 32]);
if idempotency_key != zero_key {
    // …dedup logic…
}
```

Behavioral consequence: repeated batches submitted with the zero key are
**never** deduplicated and are always applied.  The deprecation notice at
[bets.rs#L48-L53](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/bets.rs#L48-L53)
states this path will be removed in a future version.  Callers should
always generate a cryptographically-random `BytesN<32>` per batch.

Integration test asserting the backward-compat behavior:
`zero_key_disables_idempotency_deprecated` at
[batch_operations_tests.rs#L188-L198](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/batch_operations_tests.rs#L188-L198).

---

## Error Codes Touching Storage

The idempotency path produces one contract error variant, defined in
[errors.rs#L10-L18](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/errors.rs#L10-L18):

| Variant | Discriminant | Trigger |
|---------|-------------|---------|
| `IdempotentBatchAlreadyApplied` | `1` | `env.storage().instance().has(&idem_key)` returned `true` — the `(caller, token)` pair was already consumed within the TTL window. |

Clients can match on the `u32` discriminant rather than parsing the
error string.  **Do not renumber existing variants** — the discriminants
are a stable on-chain interface.

---

## Storage Surface Area Summary

| Area | Count | Notes |
|------|-------|-------|
| `DataKey` variants | 1 | `PlaceBetsIdem(Address, BytesN<32>)` |
| Distinct value shapes | 1 | `bool` (sentinel `true`) |
| TTL buckets | 1 | All sentinels share `IDEM_KEY_TTL_LEDGERS` |
| Bump sites | 1 | Only `place_bets`'s first-write path calls `extend_ttl` |
| Zero-key escape hatch | 1 | Deprecated; skips storage I/O entirely |

---

## References

- Storage definitions: [storage.rs](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/storage.rs)
- Write / check / extend_ttl site: [bets.rs](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/bets.rs#L68-L86)
- Error codes: [errors.rs](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/errors.rs)
- Integration tests for idempotency & TTL expiry: [batch_operations_tests.rs](file:///c:/Users/USER/downloads/drips/Talenttrust-Backend/contracts/predictify-hybrid/src/batch_operations_tests.rs)
- Backend contracts lifecycle (off-chain): [contracts-flow.md](contracts-flow.md)
