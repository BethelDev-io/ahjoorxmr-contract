# Co-Payer Contribution Splitting in `ahjoor-rosca`

## Overview

Co-payer contribution splitting lets a ROSCA member share a single round's contribution obligation across one or more external addresses ("co-payers"). The member registers a fixed set of splits before contributing, and thereafter settles the round through `contribute_split`, which pulls each co-payer's agreed amount directly into the group contract and credits the member as having paid.

This is useful when a member's slot is funded by several backers — for example a savings group where family members, a sponsor, or a sub-team each cover part of a contribution. The member keeps a single membership and voting identity; only the source of the tokens is split.

This feature lives in `contracts/ahjoor-rosca/src/lib.rs` and uses:

- `CoPayerSplit` — the split record (`contracts/ahjoor-rosca/src/types.rs`).
- `DataKey3::CoPayerSplits(Address)` — persistent storage of a member's registered splits.
- `events::CoPayerSplitRegistered` / `CoPayerContributed` / `CoPayerSplitRevoked` (`contracts/ahjoor-rosca/src/events.rs`).
- `ExtError2::NoCopayersRegistered`, `ExtError2::CopayerAmountsMismatch`, `ExtError2::CopayerSplitsAlreadySet` (`contracts/ahjoor-rosca/src/errors.rs`).

---

## How `register_co_payer_splits` Divides a Contribution

A member registers their split with:

```rust
pub fn register_co_payer_splits(env: Env, member: Address, splits: Vec<CoPayerSplit>)
```

where each entry is:

```rust
pub struct CoPayerSplit {
    pub co_payer: Address, // address that will fund this share
    pub amount:    i128,   // exact token amount this co-payer pays
}
```

### Required contribution

The contract first computes the member's own required contribution for the round, accounting for their membership tier. The base amount is read from `DataKey::ContributionAmt` and the member's tier weight from `DataKey2::MemberTiers`:

```text
required = base_amount × tier_bps / 10_000
```

`tier_bps` defaults to `10_000` (100 %, i.e. the full base amount) when the member has no explicit tier.

### Validation rules

`register_co_payer_splits` enforces the following, in order:

1. **Contract not paused or frozen** — `check_not_paused` / `check_not_frozen`.
2. **Member authorisation** — `member.require_auth()`. Only the member can register splits for their own slot.
3. **Membership** — `member` must be listed in `DataKey::Members`, otherwise `Error::NotAMember`.
4. **Not exited** — `member` must not be in `DataKey::ExitedMembers`, otherwise `Error::MemberHasExited`.
5. **No existing registration** — if `DataKey3::CoPayerSplits(member)` already exists, the call reverts with `ExtError2::CopayerSplitsAlreadySet` (117). The existing registration must be revoked first.
6. **Each share positive** — every `split.amount` must be `> 0`, otherwise `Error::AmountMustBePositive`.
7. **Exact sum** — the sum of all `split.amount` values must equal `required`, otherwise `ExtError2::CopayerAmountsMismatch` (115). Over- and under-funding are both rejected; partial or surplus splits are not accepted.

On success the vector is written verbatim to persistent storage under `DataKey3::CoPayerSplits(member)`, the persistent TTL is extended, and the event `CoPayerSplitRegistered { member, co_payer_count, total_split_amount }` is published.

> The contract stores the supplied vector as-is. There is no cap on the number of co-payers in a split, and duplicate co-payer addresses are not de-duplicated — the registered amounts are enforced exactly as submitted.

### Example

For a member whose tier-adjusted contribution is `1_000` tokens, a 60/40 split is registered as:

```rust
client.register_co_payer_splits(&member, &vec![
    CoPayerSplit { co_payer: alice.clone(), amount: 600 },
    CoPayerSplit { co_payer: bob.clone(),   amount: 400 },
]);
```

---

## How Each Co-Payer's Share Is Collected

Once a split is registered, the member settles the round with:

```rust
pub fn contribute_split(env: Env, member: Address, token: Address)
```

The member authorises the call (`member.require_auth()`), and the contract then iterates over the registered splits. For every entry it calls the SEP-41 token client:

```rust
token_client.transfer(&split.co_payer, &env.current_contract_address(), &split.amount);
```

so the tokens move **directly from each co-payer's address into the group contract** — they never pass through the member's wallet. Each co-payer must therefore have authorised the contract to debit their share for this transaction; the amount is the exact value registered for them.

### Validation rules

1. **Contract not paused or frozen** — `check_not_paused` / `check_not_frozen`.
2. **Member authorisation** — `member.require_auth()`.
3. **Group active** — the ledger timestamp must be at or after `DataKey4::StartAt`, otherwise `ExtError::GroupNotYetActive`.
4. **Membership** — `member` must be in `DataKey::Members` and not in `DataKey::ExitedMembers`.
5. **Not already paid** — `member` must not already be in `DataKey::PaidMembers`, otherwise `Error::AlreadyContributed`.
6. **Token approved** — `token` must be in `DataKey::ApprovedTokens` and pass the token-whitelist check, otherwise `Error::TokenNotApproved`.
7. **Registration exists** — `DataKey3::CoPayerSplits(member)` must be present, otherwise `ExtError2::NoCopayersRegistered` (114).

### State updates

For each split the contract transfers the share, emits `CoPayerContributed { member, co_payer, amount, round }`, and accumulates the total. After the loop it:

- appends `member` to `DataKey::PaidMembers`,
- records `DataKey::MemberContributions[member] = total_transferred`,
- adds `total_transferred` to `DataKey::TotalCollected`,
- emits the standard `ContributionReceived` event via `events::emit_contrib(member, current_round, token, total_transferred)`.

The round is credited to `member`, not to the individual co-payers: `PaidMembers`, contribution tallies, and any event-driven round accounting treat the member as the contributor.

### If a co-payer cannot pay

Settlement is **atomic**. All of the per-split transfers happen in one transaction, so if any co-payer has not authorised the debit, holds an insufficient balance, or is otherwise blocked, the whole `contribute_split` call reverts. No co-payer is charged, `member` is not added to `PaidMembers`, and no split is partially settled.

To recover, the member can either:

1. resolve the failing co-payer's authorisation/balance and retry `contribute_split`, or
2. call `revoke_co_payer_splits` and settle the round through the standard single-payer `contribute` path instead.

> `contribute_split` transfers the pre-registered amounts as-is. Unlike the standard `contribute` path it does not apply exchange-rate conversion, per-token limits, insurance auto-deduction, deadline/grace-period checks, or reinstatement-fee collection. Registration is validated against the tier-adjusted required amount at registration time only.

---

## What `revoke_co_payer_splits` Reverts To

A member removes their split with:

```rust
pub fn revoke_co_payer_splits(env: Env, member: Address)
```

The member authorises the call and must be a current member; the contract then requires an existing registration (`ExtError2::NoCopayersRegistered` if absent) and deletes `DataKey3::CoPayerSplits(member)` from persistent storage. The event `CoPayerSplitRevoked { member }` is published.

After revocation the member's obligation reverts to the **standard single-payer contribution flow**:

| Aspect | With splits registered | After `revoke_co_payer_splits` |
|---|---|---|
| Settlement entry point | `contribute_split(member, token)` | `contribute(contributor, token, amount)` |
| Who funds the round | Each registered co-payer, per their `amount` | The member alone |
| Required amount | Fixed, pre-validated against the tier-adjusted requirement | Supplied to `contribute` and validated by that path |
| Registered state | Stored under `DataKey3::CoPayerSplits(member)` | Removed; `get_co_payer_splits` returns an empty vector |
| Calling `contribute_split` | Succeeds (if not yet paid) | Reverts with `ExtError2::NoCopayersRegistered` (114) |

Revocation is a cancellation of the **registration**, not a reversal of money already collected:

- Rounds already settled through `contribute_split` are unaffected — the tokens are already in the contract and the member was already recorded in `PaidMembers` for those rounds. Revoking does not refund co-payers or un-mark the paid round.
- A member may register a fresh split after revoking, because the `CopayerSplitsAlreadySet` guard only blocks registration while a stored registration exists.

---

## Events

| Event | Fields | Emitted when |
|---|---|---|
| `CoPayerSplitRegistered` | `member`, `co_payer_count`, `total_split_amount` | `register_co_payer_splits` succeeds |
| `CoPayerContributed` | `member`, `co_payer`, `amount`, `round` | Once per split during `contribute_split` |
| `CoPayerSplitRevoked` | `member` | `revoke_co_payer_splits` succeeds |

---

## Error Reference

| Error | Code | Raised by | Meaning |
|---|---|---|---|
| `ExtError2::NoCopayersRegistered` | 114 | `contribute_split`, `revoke_co_payer_splits` | No splits are registered for the member |
| `ExtError2::CopayerAmountsMismatch` | 115 | `register_co_payer_splits` | Split amounts do not sum to the required contribution |
| `ExtError2::CopayerSplitsAlreadySet` | 117 | `register_co_payer_splits` | A registration already exists; revoke first |
| `Error::AmountMustBePositive` | — | `register_co_payer_splits` | A split amount is zero or negative |
| `Error::NotAMember` | — | both registration/settlement | Address is not a member of the group |
| `Error::MemberHasExited` | — | registration/settlement | Member has exited the group |
| `Error::AlreadyContributed` | — | `contribute_split` | Member already paid for the current round |
| `Error::TokenNotApproved` | — | `contribute_split` | Token is not approved or not whitelist-allowed |
| `ExtError::GroupNotYetActive` | — | `contribute_split` | Ledger time is before the group start time |

---

## API Reference

```rust
// Register a member's co-payer splits. Sum of amounts must equal the
// tier-adjusted required contribution. Member must authorise.
fn register_co_payer_splits(env: Env, member: Address, splits: Vec<CoPayerSplit>);

// Settle the current round by pulling each co-payer's share into the contract.
fn contribute_split(env: Env, member: Address, token: Address);

// Remove a member's registration, reverting them to the single-payer path.
fn revoke_co_payer_splits(env: Env, member: Address);

// Read the registered splits for a member (empty vector if none).
fn get_co_payer_splits(env: Env, member: Address) -> Vec<CoPayerSplit>;
```

---

## Testing

A dedicated integration test module is referenced from `contracts/ahjoor-rosca/src/lib.rs` but its source (`test_co_payer_split.rs`) is not present in the repository at the time of writing, so this guide is based on a static reading of the implementation rather than an executed test suite.
