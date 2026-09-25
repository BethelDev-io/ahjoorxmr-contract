# Merchant Reserve Fund in ahjoor-refund

## Overview

`ahjoor-refund` lets the admin require merchants to hold a **refund reserve**: an on-contract balance the contract draws on to pay customers when a refund is approved. The reserve requirement is expressed as a ratio (in basis points) of the merchant's recorded payment volume, so a merchant that processes more volume is expected to hold proportionally more reserve.

The feature is implemented in `contracts/ahjoor-refund/src/lib.rs` (the `#274` reserve functions) and covered by `contracts/ahjoor-refund/src/test_reserve_fund.rs`.

Two reserve subsystems currently live side by side in the contract:

* the **legacy `#274` subsystem** documented below (`DataKey::MerchantReserve*`), which is what the `set_reserve_ratio_bps` / `deposit_reserve` / `withdraw_reserve` / `check_reserve_compliance` / `get_merchant_reserve` entry points operate on; and
* a **canonical `#334` subsystem** (`DataKey2::MerchantReserveBalance`, `*_merchant_reserve` entry points), which uses a minimum-reserve-in-bps-of-monthly-volume model with a waiver mechanism.

`get_reserve_balance_summary` and `migrate_merchant_reserve` reconcile the two; see [Parallel canonical reserve subsystem](#parallel-canonical-reserve-subsystem-334) below.

## How the reserve ratio is set and enforced

The admin sets one global ratio with `set_reserve_ratio_bps(admin, ratio_bps)`:

* `ratio_bps` is in basis points, so `200` means **2%**.
* Values above `10_000` (100%) panic with `"reserve_ratio_bps cannot exceed 10000"`.
* The value is stored in instance storage under `DataKey::ReserveRatioBps` and defaults to `0` (no requirement) when unset.

The required reserve for a merchant is derived from that ratio and the merchant's tracked volume:

```text
required_reserve = (merchant_volume * reserve_ratio_bps) / 10_000
```

Two consequences follow from `required_reserve_internal`:

* If `reserve_ratio_bps == 0`, the required reserve is always `0`, so every merchant is compliant regardless of volume.
* The division is integer division, so the requirement is floored to the smallest whole token unit.

Payment volume is recorded by `record_payment_volume(caller, merchant, amount)`, which is meant to be called by the configured payments contract when a payment is created:

* `caller` must authorize and must equal the address stored in `DataKey::PaymentContractAddress`, otherwise the call panics with `"Unauthorized: caller is not the configured payment contract"`.
* Non-positive `amount` values are ignored (the call returns without recording anything).
* Otherwise `amount` is added to the merchant's `DataKey::MerchantVolume` entry.
* If the merchant is already flagged as non-compliant, the call panics with `"ReserveBelowMinimum: merchant is non-compliant; deposit reserve before creating payments"`, blocking new volume from being recorded while the merchant is under-reserved.

Compliance is evaluated — and the on-chain flag updated — by `check_reserve_compliance(admin, merchant)`, which is admin-only:

1. Reads the merchant's legacy reserve (`DataKey::MerchantReserve`, default `0`).
2. Computes `required_reserve` from the ratio and the merchant's recorded volume.
3. Marks the merchant compliant when `current >= required`.
4. If **not** compliant, sets `DataKey::MerchantFlagged(merchant)` and emits `MerchantFlaggedLowReserve { merchant, current_reserve, required_reserve }`.
5. If compliant, removes the flag if it was previously set.
6. Returns the boolean compliance result.

Both `is_merchant_flagged(merchant)` and `get_merchant_volume(merchant)` are exposed as read-only getters; both default to `false` / `0` for merchants with no recorded state.

## How deposits and withdrawals affect a merchant's available reserve

### Deposits

`deposit_reserve(merchant, token, amount)`:

* Requires the contract to be unpaused and the merchant to authorize the call.
* Rejects non-positive amounts with `"amount must be positive"`.
* Requires `token` to be whitelisted via `require_token_allowed` — only allowed tokens can be deposited.
* Transfers `amount` from the merchant to the contract and credits `DataKey::MerchantReserve(merchant)`.
* If the merchant was flagged and the new balance now meets the requirement, the flag is cleared automatically.
* Emits `ReserveDeposited { merchant, amount }`.

Deposits accumulate and can be topped up at any time, including after a refund has drawn the reserve down.

### Withdrawals

`withdraw_reserve(merchant, token, amount)`:

* Requires the merchant to authorize the call and rejects non-positive amounts.
* Reads the current reserve and the requirement derived from the merchant's volume.
* Panics with `"WithdrawalWouldBreachMinimum: reserve would fall below required minimum"` if `current - amount < required`. Withdrawing exactly down to the required minimum is allowed.
* Otherwise debits the reserve, transfers `amount` from the contract back to the merchant, and emits `ReserveWithdrawn { merchant, amount }`.

A merchant with no recorded volume has a required reserve of `0`, so it can withdraw its entire balance. As volume grows (raising the requirement), the merchant's freely-withdrawable amount shrinks until it tops the reserve back up.

### Reserve draw on an approved refund

When a refund is approved, the contract draws from the merchant's legacy reserve first:

* `draw = min(reserve_balance, refund.amount)`.
* The drawn amount is debited from `DataKey::MerchantReserve(merchant)` and transferred from the contract to the customer.
* Emits `ReserveUsedForRefund { merchant, refund_id, amount }`.

A draw does not itself flag the merchant; a subsequent `check_reserve_compliance` call is what records non-compliance if the reserve has fallen below the requirement.

## Public API

### Admin functions

* `set_reserve_ratio_bps(admin: Address, ratio_bps: u32)` — sets the global reserve ratio (basis points); panics above `10_000`.
* `check_reserve_compliance(admin: Address, merchant: Address) -> bool` — evaluates compliance, updates the merchant's flag, emits a low-reserve event when non-compliant, and returns the result.

### Merchant functions

* `deposit_reserve(merchant: Address, token: Address, amount: i128)` — deposits `amount` of an allowed token into the merchant's reserve.
* `withdraw_reserve(merchant: Address, token: Address, amount: i128)` — withdraws excess reserve; blocked if it would drop below the required minimum.

### Internal / payments-contract functions

* `record_payment_volume(caller: Address, merchant: Address, amount: i128)` — adds to the merchant's tracked volume; callable only by the configured payments contract.

### Read-only getters

* `get_merchant_reserve(merchant: Address) -> i128` — legacy reserve balance (`0` for unknown merchants).
* `is_merchant_flagged(merchant: Address) -> bool` — whether the merchant is flagged non-compliant.
* `get_merchant_volume(merchant: Address) -> i128` — recorded payment volume (`0` when unset).

## Storage keys

All keys below are on `DataKey` (the legacy `#274` subsystem):

* `ReserveRatioBps` — instance-level global reserve ratio in basis points.
* `MerchantReserve(Address)` — persistent per-merchant reserve balance.
* `MerchantVolume(Address)` — persistent trailing payment volume per merchant, used to derive the requirement.
* `MerchantFlagged(Address)` — persistent flag set when a merchant is below the required reserve.

## Events

* `ReserveDeposited { merchant, amount }` — emitted after a successful deposit.
* `ReserveWithdrawn { merchant, amount }` — emitted after a successful withdrawal.
* `ReserveUsedForRefund { merchant, refund_id, amount }` — emitted when an approved refund is paid from the merchant's reserve.
* `MerchantFlaggedLowReserve { merchant, current_reserve, required_reserve }` — emitted when `check_reserve_compliance` finds the merchant below the requirement.

## Parallel canonical reserve subsystem (#334)

The contract also exposes a second reserve subsystem built around a **minimum reserve in basis points of monthly volume**:

* `set_reserve_config(admin, token, min_bps, alert_bps)` — stores `DataKey2::ReserveToken`, `DataKey2::MinReserveBpsOfMonthlyVolume`, and `DataKey2::ReserveAlertThresholdBps`. The default `min_bps` is `500` (5%) and the default `alert_bps` is `10_000`.
* `get_reserve_config() -> (Option<Address>, u32, u32)` — returns `(reserve_token, min_reserve_bps, alert_threshold_bps)`.
* `deposit_merchant_reserve(merchant, amount)` — pulls `amount` from the merchant into the contract via `transfer_from` and credits `DataKey2::MerchantReserveBalance`; emits `MerchantReserveDeposited { merchant, amount, new_balance }`.
* `check_merchant_reserve(merchant) -> bool` — returns `true` while a merchant waiver is active; otherwise compares the canonical balance against `volume * min_bps / 10_000` and emits `MerchantReserveLow { merchant, balance, required_minimum }` when the balance is below `alert_bps` of the requirement.
* `withdraw_merchant_reserve(merchant, amount)` — blocked when the withdrawal would take the canonical balance below the minimum requirement.
* `waive_reserve_requirement(admin, merchant, waiver_expiry_ledger)` / `get_reserve_waiver_expiry(merchant) -> Option<u32>` — administrative waiver that exempts a merchant until the given ledger; emits `ReserveRequirementWaived { merchant, expiry_ledger }`.

The two subsystems are reconciled by:

* `get_reserve_balance_summary(merchant) -> ReserveSummary` — reports `legacy_reserve_balance`, `merchant_reserve_balance`, and their `total_reserve_balance` side by side.
* `migrate_merchant_reserve(admin, merchant) -> i128` — one-time admin move of the legacy balance into the canonical entry, zeroing the legacy key and returning the amount migrated (`0` when there is nothing to migrate). Total reserve value is preserved; emits `MerchantReserveMigrated { merchant, migrated_amount, new_canonical_balance }`.

> Note: the `#334` `set_reserve_config` / `waive_reserve_requirement` setters call `admin.require_auth()` twice (once directly, once inside `require_admin`), which trips `soroban-sdk`'s duplicate-authorization check under `mock_all_auths`. The reserve tests seed the related instance storage directly to work around this pre-existing issue.

## Test coverage

Tests live in `contracts/ahjoor-refund/src/test_reserve_fund.rs`:

* Deposit and withdrawal with no recorded volume (required reserve of `0`), asserting the balance updates.
* Withdrawal rejected with `WithdrawalWouldBreachMinimum` once recorded volume raises the requirement above the balance.
* `check_reserve_compliance` flagging a merchant with volume but no reserve, and passing when the requirement is `0`.
* Migration of a legacy balance into the canonical subsystem, preservation of total value, and the no-op case when there is no legacy balance.
* `get_reserve_balance_summary` for zero activity, activity in both subsystems, and activity in only the legacy subsystem.
* Flag getter behavior (never-flagged vs. flagged), volume accumulation via repeated `record_payment_volume`, and waiver-expiry resolution for active, expired, and absent waivers.

## Notes

* The reserve is denominated in whatever token the merchant deposits / the refund pays out in; deposits must use a whitelisted token.
* The requirement is recalculated on each `check_reserve_compliance` / `withdraw_reserve` call, using the merchant's cumulative recorded volume, so it grows with volume and does not automatically decay.
* The reserve is held by the contract, not by the merchant, and is drawn down automatically when refunds are approved.
