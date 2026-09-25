# ROSCA Emergency Loan

> **Status:** Implemented in `contracts/ahjoor-rosca/src/lib.rs`
> (`set_emergency_payout_config`, `request_emergency_payout`, `vote_emergency_payout`,
> `execute_emergency_payout`, `request_emergency_loan`, `repay_emergency_loan`,
> `get_emergency_loan`, `get_member_active_loan`, `get_emergency_loan_counter`,
> `get_emergency_reserve_balance`) and exercised in
> `contracts/ahjoor-rosca/src/test_emergency_loan.rs`.

---

## Overview

`ahjoor-rosca` provides two complementary safety nets so a member can access group funds
outside of the normal round payout:

1. **Emergency payout** — a member asks the group for their contribution amount early.
   The request is approved by a group vote (request → vote → execute) before any funds move.
2. **Emergency liquidity reserve loan** — a member borrows from the group's emergency
   reserve. A loan is recorded on-chain with a repayment deadline and a running
   `repaid_amount`, and is repaid (partially or in full) through `repay_emergency_loan`.

The two flows are independent: an emergency **payout** is a one-off disbursal from the
contract balance that is not repaid, while an emergency **loan** is drawn from the reserve
and must be repaid.

| Fact | Value |
| --- | --- |
| Contract | `ahjoor-rosca` |
| Emergency payout entry points | `set_emergency_payout_config`, `request_emergency_payout`, `vote_emergency_payout`, `execute_emergency_payout` |
| Reserve loan entry points | `request_emergency_loan`, `repay_emergency_loan` |
| Query functions | `get_emergency_loan`, `get_member_active_loan`, `get_emergency_loan_counter`, `get_emergency_reserve_balance` |
| Payout config storage key | `DataKey2::EmergencyPayoutConfig` |
| Payout request / vote storage keys | `DataKey2::EmergencyPayoutRequests`, `DataKey2::EmergencyPayoutVotes`, `DataKey2::EmergencyPayoutApproved`, `DataKey2::EmergencyPayoutCount` |
| Reserve loan storage keys | `DataKey3::ReserveEnabled`, `DataKey3::EmergencyReserveBalance`, `DataKey3::EmergencyLoanCounter`, `DataKey3::EmergencyLoan(u32)`, `DataKey3::MemberOutstandingLoan(Address)` |

---

## Emergency Payout Lifecycle

```text
        +----------------------------------------------------+
        | member calls request_emergency_payout(reason_hash) |
        | - Membership + group-status checks                 |
        | - One request per (round, member)                  |
        | - Per-cycle cap honoured                           |
        | - deadline = now + vote_window_seconds             |
        +----------------------------------------------------+
                              |
                              v
        +----------------------------------------------------+
        | members call vote_emergency_payout(requester, ...) |
        | - Requester cannot vote for themselves             |
        | - One vote per (round, requester, voter)           |
        | - Vote weight from get_member_voting_weight        |
        | - Voting stops once deadline has passed            |
        +----------------------------------------------------+
                              |
                              v
        +----------------------------------------------------+
        | anyone calls execute_emergency_payout(requester)   |
        | - Quorum check over members/voting weight          |
        | - votes_for must beat votes_against                |
        +----------------------------------------------------+
                 |                            |
     quorum not met / majority no        approved
                 |                            |
                 v                            v
   +----------------------------+   +----------------------------------+
   | EmergencyPayoutRejected    |   | Transfer ContributionAmt to      |
   | (no funds move)            |   | requester; mark as paid for the  |
   |                            |   | round; EmergencyPayoutExecuted   |
   +----------------------------+   +----------------------------------+
```

### 1. Request — `request_emergency_payout`

```rust
pub fn request_emergency_payout(env: Env, member: Address, reason_hash: BytesN<32>)
```

**Authentication:** `member.require_auth()`

**Behaviour:**
1. Rejects if the contract is paused (`check_not_paused`) or if the group is
   `GroupStatus::Dissolved` (`ExtError::GroupAlreadyDissolved`).
2. Rejects non-members (`Error::NotAMember`).
3. Rejects a second request for the same `(current_round, member)`
   (`ExtError::EmergencyPayoutRequested`).
4. Rejects a member who has already had an emergency payout executed in the current
   cycle (`ExtError::EmergencyPayoutAlreadyExecuted`).
5. Rejects the request when the current cycle has already reached
   `max_emergency_per_cycle` (`ExtError::EmergencyPayoutLimitReached`).
6. Stores an `EmergencyPayoutRequest` keyed by `(current_round, member)` with
   `created_at = ledger timestamp`, `deadline = created_at + vote_window_seconds`,
   and zeroed vote counters, then emits `EmergencyPayoutRequested`.

The `reason_hash` is an opaque `BytesN<32>` supplied by the caller (for example a hash of
off-chain documentation); the contract stores it but does not interpret it.

> **Cycles:** a cycle is derived as `cycle_index = current_round / payout_order.len()` and
> is used for both the "already executed" check and the per-cycle emergency limit.

### 2. Vote — `vote_emergency_payout`

```rust
pub fn vote_emergency_payout(env: Env, voter: Address, requester: Address, approve: bool)
```

**Authentication:** `voter.require_auth()`

**Behaviour:**
1. Rejects if paused or if the group is dissolved.
2. Rejects non-members (`Error::OnlyMembersAllowed`).
3. Rejects self-voting: the requester cannot vote on their own request.
4. Requires an existing, not-yet-executed request for `(current_round, requester)`.
5. Rejects votes cast after `request.deadline`
   (`ExtError::EmergencyPayoutVoteExpired`).
6. Rejects a duplicate vote from the same voter
   (`DataKey2::EmergencyPayoutVotes` marks `(round, requester, voter)`).
7. Adds `get_member_voting_weight(voter)` to `votes_for` or `votes_against`:
   `1` per member in `VotingMode::Equal`, or the member's contributed amount in
   `VotingMode::WeightedByContributions`.
8. Emits `EmergencyPayoutVoteCast` with the updated tallies.

### 3. Execute — `execute_emergency_payout`

```rust
pub fn execute_emergency_payout(env: Env, requester: Address)
```

There is no `require_auth` on this entry point — execution is permissionless, and the
outcome is fully determined by the recorded votes.

**Behaviour:**
1. Rejects if paused, if the group is dissolved, if no request exists for
   `(current_round, requester)`, if the request was already executed
   (`ExtError::EmergencyPayoutAlreadyExecuted`), or if the vote window has expired
   (`ExtError::EmergencyPayoutVoteExpired`).
2. Computes the required approvals as
   `required = ceil(total_possible_votes * emergency_quorum_bps / 10_000)`, where
   `total_possible_votes` is the member count (`Equal`) or the sum of all member
   contributions (`WeightedByContributions`). If
   `votes_for + votes_against < required`, it panics with
   `ExtError::EmergencyPayoutQuorumNotMet`.
3. If `votes_for <= votes_against`, emits `EmergencyPayoutRejected` with reason
   `"votes_failed"` and returns without transferring any funds.
4. Otherwise it marks the request `executed`, records the requester as approved for the
   cycle, increments the per-cycle emergency count, transfers the configured
   `ContributionAmt` from the contract to the requester, appends the requester to the
   round's paid members, and emits `EmergencyPayoutExecuted`.

Because the requester is added to `PaidMembers`, an executed emergency payout replaces the
member's normal round payout rather than stacking on top of it.

---

## Configuration — `set_emergency_payout_config`

```rust
pub fn set_emergency_payout_config(
    env: Env,
    admin: Address,
    emergency_quorum_bps: u32,
    vote_window_seconds: u64,
    max_emergency_per_cycle: u32,
)
```

**Authentication:** `admin.require_auth()`, and the caller must equal `DataKey::Admin`
(otherwise the call panics). The function stores an `EmergencyPayoutConfig` under
`DataKey2::EmergencyPayoutConfig` and emits `EmergencyPayoutConfigUpdated`.

| Field | Meaning | Validation |
| --- | --- | --- |
| `emergency_quorum_bps` | Share of possible votes required to pass, in basis points. `1000` = 10%, `10000` = 100%. | Must be between `1000` and `10000` inclusive, else `ExtError::InvalidEmergencyConfig`. |
| `vote_window_seconds` | Length of the voting window; `deadline = created_at + vote_window_seconds`. | Must be non-zero, else `ExtError::InvalidEmergencyConfig`. |
| `max_emergency_per_cycle` | Maximum number of emergency payouts that may be executed per cycle. | Must be non-zero, else `ExtError::InvalidEmergencyConfig`. |

If no configuration has ever been stored, the contract falls back to these defaults at
request/execute time:

| Field | Default |
| --- | --- |
| `emergency_quorum_bps` | `6667` (66.67%) |
| `vote_window_seconds` | `604800` (7 days) |
| `max_emergency_per_cycle` | `1` |

---

## Emergency Reserve Loan

Unlike an emergency payout, a reserve loan is drawn from the group's emergency reserve,
is tracked as an on-chain `EmergencyLoan`, and must be repaid.

### Request — `request_emergency_loan`

```rust
pub fn request_emergency_loan(
    env: Env,
    member: Address,
    amount: i128,
    repayment_window_ledgers: u32,
) -> u32
```

**Authentication:** `member.require_auth()`

**Preconditions:**
- Caller must be a member (`Error::NotAMember`).
- `amount` must be positive.
- The reserve must be enabled (`DataKey3::ReserveEnabled` is `true`).
- The member must not already have an outstanding loan
  (`DataKey3::MemberOutstandingLoan` must be `0`), else
  `ExtError2::OutstandingLoanExists`.
- `amount` must not exceed the current reserve balance.
- `amount` must not exceed `MAX_LOAN_FRACTION_BPS` of the reserve balance, where
  `MAX_LOAN_FRACTION_BPS = 5_000` (50%).

**Effect:**
1. Allocates `loan_id` from `DataKey3::EmergencyLoanCounter` (incremented per loan).
2. Stores an `EmergencyLoan` under `DataKey3::EmergencyLoan(loan_id)` with:
   - `borrower`, `amount`
   - `created_at_ledger = env.ledger().sequence()`
   - `repayment_deadline_ledger = created_at_ledger + repayment_window_ledgers`
   - `repaid_amount = 0`
   - `defaulted = false`
3. Maps the borrower to the active loan via
   `DataKey3::MemberOutstandingLoan(member) = loan_id`.
4. Deducts `amount` from `DataKey3::EmergencyReserveBalance` and transfers the tokens
   from the contract to the member.
5. Emits `EmergencyLoanGranted` and returns the new `loan_id`.

The `repayment_window_ledgers` value is supplied by the caller and recorded as-is; the
contract does not enforce a minimum or maximum window, and `request_emergency_loan`
does not check whether the contract or group is paused or dissolved.

### Repayment — `repay_emergency_loan`

Repayment is tracked per loan through `EmergencyLoan.repaid_amount`:

```rust
pub fn repay_emergency_loan(env: Env, member: Address, loan_id: u32, amount: i128)
```

**Authentication:** `member.require_auth()`

**Behaviour:**
1. Rejects a non-positive `amount`.
2. Loads `EmergencyLoan(loan_id)`; the caller must be the borrower, and a loan whose
   `defaulted` flag is `true` cannot be repaid.
3. Rejects a repayment larger than the remaining balance
   (`amount > loan.amount - loan.repaid_amount`), so partial repayments are allowed but
   the loan can never be over-paid.
4. Adds the payment to `loan.repaid_amount`. When
   `repaid_amount >= loan.amount`, the loan is considered settled and
   `DataKey3::MemberOutstandingLoan(borrower)` is removed, freeing the borrower to take a
   new loan.
5. Transfers the tokens from the member back to the contract, adds the amount to
   `DataKey3::EmergencyReserveBalance`, and emits `EmergencyLoanRepaid` with the amount
   paid and the new remaining balance.

Repayments are interest-free and there is no late fee; the `repayment_deadline_ledger` is
recorded but is not enforced by `repay_emergency_loan`.

### Default Handling

`EmergencyLoan.defaulted` exists and is honoured by `repay_emergency_loan` (a defaulted
loan cannot be repaid). However, in the current `ahjoor-rosca` implementation no entry
point sets `defaulted` to `true`: the `LoanDefaultDeducted` event and its
`emit_loan_default_deducted` helper are defined in `contracts/ahjoor-rosca/src/events.rs`
but are not invoked from `lib.rs`. Default enforcement is therefore not yet wired on-chain.

### Queries

| Function | Returns |
| --- | --- |
| `get_emergency_loan(loan_id)` | The full `EmergencyLoan` record; panics with `"Loan not found"` if it does not exist. |
| `get_member_active_loan(member)` | The member's active `loan_id`, or `0` when they have no outstanding loan. |
| `get_emergency_loan_counter()` | Total number of emergency loans ever issued; `0` when none. |
| `get_emergency_reserve_balance()` | Current `DataKey3::EmergencyReserveBalance`; `0` when unset. |

Remaining principal is not stored separately — compute it as
`EmergencyLoan.amount - EmergencyLoan.repaid_amount`.

---

## Events

| Event | Emitted by | Fields |
| --- | --- | --- |
| `EmergencyPayoutRequested` | `request_emergency_payout` | `requester`, `round`, `reason_hash`, `deadline` |
| `EmergencyPayoutVoteCast` | `vote_emergency_payout` | `requester`, `round`, `voter`, `approve`, `votes_for`, `votes_against` |
| `EmergencyPayoutRejected` | `execute_emergency_payout` | `requester`, `round`, `reason` |
| `EmergencyPayoutExecuted` | `execute_emergency_payout` | `requester`, `round`, `payout_amount` |
| `EmergencyPayoutConfigUpdated` | `set_emergency_payout_config` | `emergency_quorum_bps`, `vote_window_seconds`, `max_emergency_per_cycle` |
| `EmergencyLoanGranted` | `request_emergency_loan` | `group_id`, `member`, `loan_id`, `amount`, `repayment_deadline` |
| `EmergencyLoanRepaid` | `repay_emergency_loan` | `group_id`, `loan_id`, `amount`, `remaining` |

> [!NOTE]
> The `EmergencyPayoutApproved` event struct is defined in `events.rs` but no current entry
> point emits it; successful payouts emit `EmergencyPayoutExecuted`. The loan events pass
> `group_id = 0`, matching this single-group contract.

---

## Error Codes

| Code | Name | Raised by | Description |
| --- | --- | --- | --- |
| 8 | `NotAMember` | `request_emergency_payout`, `request_emergency_loan` | Caller is not a member of the group. |
| 19 | `OnlyMembersAllowed` | `vote_emergency_payout` | Voter is not a member of the group. |
| 57 | `EmergencyPayoutRequested` | `request_emergency_payout` | A request already exists for this member in this round. |
| 58 | `EmergencyPayoutQuorumNotMet` | `execute_emergency_payout` | Total votes cast are below the required quorum. |
| 59 | `EmergencyPayoutVoteExpired` | `vote_emergency_payout`, `execute_emergency_payout` | The vote window has closed. |
| 60 | `EmergencyPayoutAlreadyExecuted` | `request_emergency_payout`, `execute_emergency_payout` | This member already received an emergency payout in this cycle. |
| 61 | `EmergencyPayoutLimitReached` | `request_emergency_payout` | `max_emergency_per_cycle` reached for the cycle. |
| 62 | `GroupAlreadyDissolved` | Request/vote/execute | The group has been dissolved. |
| 67 | `InvalidEmergencyConfig` | `set_emergency_payout_config` | Quorum, vote window, or per-cycle limit out of range. |
| 113 | `OutstandingLoanExists` | `request_emergency_loan` | The member already has an active loan. |

---

## Function Reference

| Function | Caller | Purpose |
| --- | --- | --- |
| `set_emergency_payout_config(admin, quorum_bps, vote_window_seconds, max_per_cycle)` | Admin | Configures quorum, voting window, and per-cycle cap for emergency payouts. |
| `request_emergency_payout(member, reason_hash)` | Member | Opens a voted emergency payout request for the current round. |
| `vote_emergency_payout(voter, requester, approve)` | Member | Casts a weighted vote on a pending request. |
| `execute_emergency_payout(requester)` | Anyone | Tallies votes and, on approval, transfers the contribution amount to the requester. |
| `request_emergency_loan(member, amount, repayment_window_ledgers)` | Member | Draws a reserve loan and records it with a repayment deadline. |
| `repay_emergency_loan(member, loan_id, amount)` | Borrower | Applies a partial or full repayment and updates the reserve. |
| `get_emergency_loan(loan_id)` | Anyone | Returns a loan record. |
| `get_member_active_loan(member)` | Anyone | Returns the member's outstanding loan id (`0` if none). |
| `get_emergency_loan_counter()` | Anyone | Returns the number of loans ever issued. |
| `get_emergency_reserve_balance()` | Anyone | Returns the current reserve balance. |

---

## Storage Reference

| Key | Storage | Data stored |
| --- | --- | --- |
| `DataKey2::EmergencyPayoutConfig` | Instance | `EmergencyPayoutConfig { emergency_quorum_bps, vote_window_seconds, max_emergency_per_cycle }`. |
| `DataKey2::EmergencyPayoutRequests` | Instance | `Map<(round, requester), EmergencyPayoutRequest>`. |
| `DataKey2::EmergencyPayoutVotes` | Instance | `Map<(round, requester, voter), bool>` marking who has voted. |
| `DataKey2::EmergencyPayoutApproved` | Instance | `Map<(cycle_index, requester), bool>` marking executed payouts. |
| `DataKey2::EmergencyPayoutCount` | Instance | `Map<cycle_index, u32>` count of emergency payouts per cycle. |
| `DataKey3::ReserveEnabled` | Instance | `bool` gate for the emergency reserve. |
| `DataKey3::EmergencyReserveBalance` | Persistent | `i128` tokens available in the reserve. |
| `DataKey3::EmergencyLoanCounter` | Persistent | `u32` monotonic loan id counter. |
| `DataKey3::EmergencyLoan(u32)` | Persistent | `EmergencyLoan` record for the given id. |
| `DataKey3::MemberOutstandingLoan(Address)` | Persistent | `u32` id of the member's active loan (`0`/absent when none). |
