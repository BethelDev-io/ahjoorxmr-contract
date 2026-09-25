# On-Chain Audit Trail in Ahjoor ROSCA

## Overview

The `ahjoor-rosca` contract maintains a comprehensive, tamper-proof on-chain audit trail of completed rounds (cycles). Every time a round completes—either automatically when all members contribute or administratively when a round is finalized past its deadline—an immutable cycle record is generated and stored on-chain.

This audit trail provides full visibility into community savings activity, enabling members, pool administrators, auditors, and indexers to inspect historical contributions, payout distributions, defaults, skips, penalty collections, protocol fees, and insurance claims.

The implementation resides in `contracts/ahjoor-rosca/src/audit_trail.rs` with supporting data structures in `contracts/ahjoor-rosca/src/types.rs`, events in `contracts/ahjoor-rosca/src/events.rs`, and comprehensive unit tests in `contracts/ahjoor-rosca/src/test_audit_trail.rs`.

---

## When Audit Records Are Captured

Cycle audit records are captured **atomically at round completion** inside `internals::complete_round_payout`. This occurs under two execution paths:

1. **Automatic Round Completion (`contribute`)**: When the last active member deposits their required contribution for the active round, the round immediately closes, executes the payout to the scheduled recipient, and records the audit record.
2. **Administrative Finalization (`finalize_round`)**: If a round deadline expires with missing contributions, the contract administrator calls `finalize_round`. Defaulters are recorded, insurance draws or penalties are applied, available pot funds are disbursed, and the audit trail entry is finalized.

In addition, the start timestamp of each cycle is recorded at cycle initialization (`audit_trail::record_cycle_start`) so that round durations can be calculated accurately.

---

## Captured Data Structures

### `CycleRecord`

Each completed cycle is captured as a `CycleRecord` struct containing:

```rust
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct CycleRecord {
    pub cycle_number: u32,
    pub total_pool_amount: i128,
    pub payout_recipient: Address,
    pub payout_amount: i128,
    pub contributions: Vec<ContributionEntry>,
    pub defaulters: Vec<Address>,
    pub skippers: Vec<Address>,
    pub penalties_collected: i128,
    pub fee_collected: i128,
    pub insurance_drawn: i128,
    pub cycle_start_timestamp: u64,
    pub cycle_end_timestamp: u64,
}
```

#### Field Descriptions

| Field | Type | Description |
|---|---|---|
| `cycle_number` | `u32` | 0-indexed sequence number of the completed cycle. |
| `total_pool_amount` | `i128` | Total pot collected and distributed for the cycle. |
| `payout_recipient` | `Address` | Stellar address of the member who received this round's payout. |
| `payout_amount` | `i128` | Amount paid out to the recipient (matching `total_pool_amount`). |
| `contributions` | `Vec<ContributionEntry>` | List of all individual member contribution entries for this round. |
| `defaulters` | `Vec<Address>` | Addresses of members who failed to contribute before the deadline. |
| `skippers` | `Vec<Address>` | Addresses of members who successfully exercised a round skip. |
| `penalties_collected` | `i128` | Cumulative penalty fees assessed against defaulters for this cycle. |
| `fee_collected` | `i128` | Protocol management fee deducted from the pot and sent to fee recipient. |
| `insurance_drawn` | `i128` | Funds drawn from the group insurance reserve to cover contribution shortfalls. |
| `cycle_start_timestamp` | `u64` | Starting timestamp (or ledger sequence) of the cycle. |
| `cycle_end_timestamp` | `u64` | Ending timestamp (or ledger sequence) at round closure. |

### `ContributionEntry`

Individual deposits are tracked in the `contributions` list as `ContributionEntry` structs:

```rust
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct ContributionEntry {
    pub member: Address,
    pub amount: i128,
    pub timestamp: u64,
}
```

#### Timestamp Handling

- When `use_timestamp_schedule` is enabled in `RoscaConfig`, timestamps reflect Unix epoch seconds from `env.ledger().timestamp()`.
- In ledger-sequence mode (`use_timestamp_schedule = false`), timestamps reflect the ledger sequence number (`env.ledger().sequence() as u64`) to ensure non-zero, monotonically increasing markers.
- Cycle 0's start timestamp defaults to the cycle end timestamp if no previous round completion hook was executed.

---

## Storage Architecture and Archival Lifecycle

### Per-Cycle Storage Keys (O(1) Access)

Earlier architectures stored cycle history in a single unbounded map. In the current design:
- Each cycle record is stored under its own dedicated persistent storage key:
  `DataKey5::CycleRecordEntry(cycle_number)`.
- This ensures reads and writes cost $O(1)$ CPU and memory, regardless of how many cycles the ROSCA group has run.

### TTL Extension

Cycle records stored in persistent storage are kept active using Soroban's time-to-live (TTL) management:
- `PERSISTENT_LIFETIME_THRESHOLD`: 100,000 ledgers (~5.7 days on mainnet).
- `PERSISTENT_BUMP_AMOUNT`: 120,000 ledgers (~6.9 days on mainnet).

Whenever a record or retention window is accessed or written, its TTL is automatically bumped to ensure data persistence.

### Retention Window and Automatic Archival

To balance on-chain state footprint with historical transparency, ROSCA supports a configurable retention window:

- **Default Retention Window**: 100 cycles (`DEFAULT_RETENTION_WINDOW = 100`).
- **Archive Threshold**: Calculated as `archive_threshold = current_cycle - retention_window`.
- **Automatic Migration**: When a new round closes and `current_cycle > retention_window`, any cycle records older than the retention threshold are migrated from `DataKey5::CycleRecordEntry(cycle_num)` to `DataKey5::ArchivedCycleRecordEntry(cycle_num)`.
- **Persistent Archival**: Archived records continue to reside in persistent storage with proactive TTL bumping, preventing accidental expiration while freeing the primary active key namespace.
- **Bounded Amortized $O(1)$ Execution**: The contract maintains `DataKey5::OldestPersistentCycle` to walk forward incrementally rather than scanning all cycles, ensuring negligible gas overhead during round closure.

---

## Audit Trail Events

The contract emits structured events for real-time monitoring and off-chain indexing:

### 1. `CycleRecordCreated`

Emitted immediately when a cycle audit entry is created upon round completion.

```rust
#[contractevent]
pub struct CycleRecordCreated {
    pub cycle_number: u32,
    pub total_pool_amount: i128,
    pub payout_recipient: Address,
}
```

### 2. `CycleRecordArchived`

Emitted when a cycle record exceeding the retention window is moved to archived storage.

```rust
#[contractevent]
pub struct CycleRecordArchived {
    pub cycle_number: u32,
}
```

### 3. `RetentionWindowUpdated`

Emitted when the administrator updates the group's retention window.

```rust
#[contractevent]
pub struct RetentionWindowUpdated {
    pub old_window: u32,
    pub new_window: u32,
}
```

---

## Public Query API

The smart contract exposes four public functions in `AhjoorContract` to inspect the audit log and configure retention:

```rust
pub fn get_cycle_record(env: Env, cycle_number: u32) -> Option<CycleRecord>;
pub fn get_member_contribution_history(
    env: Env,
    member: Address,
    from_cycle: Option<u32>,
    to_cycle: Option<u32>,
) -> Vec<ContributionEntry>;
pub fn get_cycle_retention_window(env: Env) -> u32;
pub fn set_cycle_retention_window(env: Env, new_window: u32);
```

### 1. `get_cycle_record(cycle_number: u32) -> Option<CycleRecord>`

Fetches the complete audit record for a given cycle.
- First searches active persistent storage (`DataKey5::CycleRecordEntry`).
- If not present, seamlessly checks archived persistent storage (`DataKey5::ArchivedCycleRecordEntry`).
- Returns `None` if the cycle has not yet completed or does not exist.

### 2. `get_member_contribution_history(member: Address, from_cycle: Option<u32>, to_cycle: Option<u32>) -> Vec<ContributionEntry>`

Queries historical contributions made by a specific member across a bounded cycle window.

#### Bounded Range & Pagination Rules

To prevent transaction timeouts and excessive CPU consumption:
- **Maximum Query Span (`MAX_CONTRIBUTION_HISTORY_RANGE`)**: 100 cycles per call. Panics with `"cycle range exceeds maximum allowed"` if `to_cycle - from_cycle >= 100`.
- **Default Query Window (`DEFAULT_CONTRIBUTION_HISTORY_WINDOW`)**: 25 cycles.
  - If both `from_cycle` and `to_cycle` are omitted (`None, None`), returns the most recent 25 cycles up to the latest completed cycle.
  - If only `from_cycle` is specified (`Some(from), None`), returns `from..=from + 24`.
  - If only `to_cycle` is specified (`None, Some(to)`), returns `to - 24..=to`.
  - If both are provided (`Some(from), Some(to)`), validates `from_cycle <= to_cycle` (panics with `"from_cycle must not exceed to_cycle"` otherwise).

Callers can paginate through complete historical records by advancing `from_cycle` and `to_cycle` in 100-cycle increments.

### 3. `get_cycle_retention_window() -> u32`

Returns the currently configured retention window (in number of cycles). Defaults to `100`.

### 4. `set_cycle_retention_window(new_window: u32)`

Updates the retention window. Requires contract administrator authorization (`admin.require_auth()`).

---

## How to Read the Audit Log

### Using Soroban CLI

#### 1. Query a Specific Cycle Record

```bash
soroban contract invoke \
  --id <CONTRACT_ID> \
  --source <CALLER_IDENTITY> \
  --network <NETWORK> \
  -- \
  get_cycle_record \
  --cycle_number 0
```

**Example JSON Response:**
```json
{
  "cycle_number": 0,
  "total_pool_amount": "3000000000",
  "payout_recipient": "GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5",
  "payout_amount": "3000000000",
  "contributions": [
    {
      "member": "GBBD47IF6LWK7P7MDEVSCWR7DPUWV3NY3DTQEVFL4NAT4AQH3ZLLFLA5",
      "amount": "1000000000",
      "timestamp": 1700000100
    },
    {
      "member": "GCSW2JNZKVZZOGJD5GQL74VWWVZZOGJD5GQL74VWWVZZOGJD5GQL74VW",
      "amount": "1000000000",
      "timestamp": 1700000120
    },
    {
      "member": "GA7QYNF7SOWQ3GLR2BGMZEHXAVIRZA4KVWLTJJFC7MGXUA74P7UJVREW",
      "amount": "1000000000",
      "timestamp": 1700000150
    }
  ],
  "defaulters": [],
  "skippers": [],
  "penalties_collected": "0",
  "fee_collected": "30000000",
  "insurance_drawn": "0",
  "cycle_start_timestamp": 1700000000,
  "cycle_end_timestamp": 1700000150
}
```

#### 2. Query Member Contribution History (Recent Window)

```bash
soroban contract invoke \
  --id <CONTRACT_ID> \
  --source <CALLER_IDENTITY> \
  --network <NETWORK> \
  -- \
  get_member_contribution_history \
  --member <MEMBER_ADDRESS>
```

#### 3. Query Member Contribution History (Explicit Paged Range)

```bash
soroban contract invoke \
  --id <CONTRACT_ID> \
  --source <CALLER_IDENTITY> \
  --network <NETWORK> \
  -- \
  get_member_contribution_history \
  --member <MEMBER_ADDRESS> \
  --from_cycle 0 \
  --to_cycle 50
```

#### 4. Query and Update Retention Window (Admin Only)

```bash
# Query retention window
soroban contract invoke \
  --id <CONTRACT_ID> \
  --source <CALLER_IDENTITY> \
  --network <NETWORK> \
  -- \
  get_cycle_retention_window

# Set retention window to 50 cycles
soroban contract invoke \
  --id <CONTRACT_ID> \
  --source <ADMIN_IDENTITY> \
  --network <NETWORK> \
  -- \
  set_cycle_retention_window \
  --new_window 50
```

---

### Using Rust Soroban Client SDK

```rust
use soroban_sdk::{Address, Env};
use ahjoor_rosca::{AhjoorContractClient, CycleRecord, ContributionEntry};

fn inspect_audit_trail(env: &Env, contract_id: &Address, member: &Address) {
    let client = AhjoorContractClient::new(env, contract_id);

    // 1. Fetch record for cycle 0
    if let Some(record) = client.get_cycle_record(&0u32) {
        println!("Cycle {} payout to {}", record.cycle_number, record.payout_recipient);
        println!("Total distributed: {}", record.payout_amount);
        println!("Defaulters count: {}", record.defaulters.len());
        println!("Skippers count: {}", record.skippers.len());
    }

    // 2. Fetch the most recent contributions for a member (default 25-cycle window)
    let recent_contributions = client.get_member_contribution_history(member, &None, &None);
    println!("Recent contributions recorded: {}", recent_contributions.len());

    // 3. Paginate through historical cycles 0 to 49
    let paged_contributions = client.get_member_contribution_history(
        member,
        &Some(0u32),
        &Some(49u32),
    );
    for entry in paged_contributions.iter() {
        println!("Cycle contribution amount: {}, timestamp: {}", entry.amount, entry.timestamp);
    }
}
```

---

## Test Coverage

Unit and integration tests for the audit trail are located in `contracts/ahjoor-rosca/src/test_audit_trail.rs`:

| Test Name | Verifies |
|---|---|
| `test_cycle_record_created_on_round_completion` | Validates that closing a round automatically generates a `CycleRecord` for the cycle. |
| `test_cycle_record_contains_all_contributions` | Ensures all member contributions with correct amounts and addresses are stored. |
| `test_cycle_record_tracks_defaulters` | Verifies delinquent non-contributing members are logged in `defaulters` upon `finalize_round`. |
| `test_cycle_record_tracks_skippers` | Confirms members requesting skips are recorded under `skippers`. |
| `test_member_contribution_history` | Tests retrieving member contribution history across multiple completed rounds. |
| `test_member_contribution_history_range_returns_only_requested_slice` | Checks that explicit `from_cycle..=to_cycle` bounds return exact slices. |
| `test_member_contribution_history_defaults_to_recent_bounded_window` | Verifies omitting bounds returns the default 25-cycle window. |
| `test_member_contribution_history_rejects_oversized_range` | Confirms panics when the queried range meets or exceeds 100 cycles. |
| `test_member_contribution_history_rejects_inverted_range` | Validates rejection when `from_cycle > to_cycle`. |
| `test_member_contribution_history_cost_bounded_by_range_not_total_history` | Verifies query gas cost scales with the requested slice, not total contract age. |
| `test_retention_window_configuration` | Confirms reading and updating the retention window by the admin. |
| `test_cycle_record_includes_fee_collected` | Asserts protocol fee deductions are logged accurately in `fee_collected`. |
| `test_cycle_record_includes_insurance_drawn` | Validates insurance pool draw amounts are tracked when covering shortfalls. |
| `test_cycle_timestamps_recorded` | Ensures start and end timestamps are recorded and ordered properly. |
| `test_multiple_cycles_recorded` | Verifies sequential multi-round cycle creation across consecutive rounds. |
| `test_archived_records_accessible` | Validates that records older than the retention window remain queryable after archival. |
| `test_cycle_record_total_pool_amount` | Ensures `total_pool_amount` equals sum of member contributions. |
| `test_cycle_record_timestamp_nonzero_ledger_mode` | Confirms timestamps fall back to ledger sequence numbers when timestamp mode is disabled. |
