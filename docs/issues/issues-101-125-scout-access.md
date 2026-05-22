# Issues 101–125: `scout_access` Contract

---

## Issue 101 — `initialize` in `scout_access` emits no event

**Labels:** `enhancement`, `scout_access`, `events`

**Description:**
Consistent with Issues 2, 36, and 71, `scout_access.initialize` is silent. The backend indexer cannot confirm the contract is live.

**Task:**
Emit a `contract_initialized` event at the end of a successful `initialize` call, publishing the admin address.

**Acceptance Criteria:**
- [ ] Event is emitted on first successful `initialize`.
- [ ] No event on duplicate `initialize` call.
- [ ] Unit test asserts event presence.

---

## Issue 102 — `subscribe` does not prevent re-subscribing with a lower tier before expiry

**Labels:** `bug`, `scout_access`, `subscriptions`

**Description:**
A scout with an active Elite subscription can call `subscribe(Basic)` and overwrite their subscription record with a cheaper, shorter-duration one. This is a downgrade attack that could be used to reduce fees retroactively.

**Task:**
In `subscribe`, if an active subscription already exists, only allow upgrades (Basic → Pro → Elite). Downgrades or same-tier re-subscriptions before expiry should return a new `ScoutAccessError::SubscriptionDowngradeNotAllowed = 12`.

**Additional Requirements:**
- Allow re-subscription at any tier after the current subscription has expired.
- Define tier ordering: `Basic < Pro < Elite`.

**Acceptance Criteria:**
- [ ] Downgrading from Elite to Pro before expiry returns `SubscriptionDowngradeNotAllowed`.
- [ ] Upgrading from Basic to Elite before expiry succeeds and extends the expiry.
- [ ] Re-subscribing at any tier after expiry succeeds.
- [ ] Unit tests cover all three cases.

---

## Issue 103 — `subscribe` does not validate that the XLM fee transfer succeeded

**Labels:** `bug`, `scout_access`, `payments`

**Description:**
`token::Client::new(&env, &xlm).transfer(...)` panics on failure (insufficient balance, unauthorized). The contract relies on the SDK panic rather than returning a typed error.

**Task:**
Wrap the `transfer` call in a `try_transfer` (if available in the SDK) or document that the panic is the intended behavior and add a code comment explaining it. If `try_transfer` is available, return `ScoutAccessError::InsufficientFee` on failure.

**Additional Requirements:**
- Check the Soroban SDK version (`25.3.1`) for `try_transfer` availability.
- If not available, add a `// NOTE:` comment explaining the panic behavior.

**Acceptance Criteria:**
- [ ] Either `try_transfer` is used and returns `InsufficientFee` on failure, or a clear comment explains the panic behavior.
- [ ] Unit test attempts to subscribe with zero balance and asserts the expected outcome.

---

## Issue 104 — `subscribe` fee accumulation can overflow `AccumulatedFees`

**Labels:** `bug`, `scout_access`, `payments`

**Description:**
`accumulate_fee` adds `amount` to `AccumulatedFees` with plain `+` (no overflow check). With enough subscriptions, this could overflow `i128`.

**Task:**
Replace `current + amount` in `accumulate_fee` with `current.checked_add(amount).ok_or(ScoutAccessError::Overflow)?` and propagate the error.

**Additional Requirements:**
- Update `accumulate_fee` to return `Result<(), ScoutAccessError>`.
- Update all callers to propagate with `?`.

**Acceptance Criteria:**
- [ ] No unchecked arithmetic in `accumulate_fee`.
- [ ] `ScoutAccessError::Overflow` is returned on overflow.
- [ ] Unit test simulates overflow by setting `AccumulatedFees` to `i128::MAX - 1` and calling `subscribe`.

---

## Issue 105 — `withdraw_fees` does not emit the amount in the event

**Labels:** `enhancement`, `scout_access`, `events`

**Description:**
`fees_withdrawn` already includes `amount` in the event data — this is correct. However, the event does not include the ledger timestamp, making it harder to correlate with off-chain records.

**Task:**
Update `fees_withdrawn` in `events.rs` to include `timestamp: u64` (from `env.ledger().timestamp()`) in the event data tuple.

**Additional Requirements:**
- Update the call site in `withdraw_fees`.
- Update `CONTRACT_REFERENCE.md`.

**Acceptance Criteria:**
- [ ] Event data is `(recipient, amount, timestamp)`.
- [ ] Unit test asserts all three values.

---

## Issue 106 — `withdraw_fees` should require a minimum withdrawal amount

**Labels:** `enhancement`, `scout_access`, `admin`

**Description:**
`withdraw_fees` allows withdrawing 0 fees (it returns `Ok(0)` silently). This wastes a transaction and could be used to probe the contract state.

**Task:**
Return `ScoutAccessError::InsufficientFee` if `fees == 0` instead of returning `Ok(0)`.

**Additional Requirements:**
- Update the existing test that calls `withdraw_fees` to ensure it only calls after fees have accumulated.

**Acceptance Criteria:**
- [ ] `withdraw_fees` with zero accumulated fees returns `InsufficientFee`.
- [ ] `withdraw_fees` with positive fees succeeds and resets `AccumulatedFees` to 0.
- [ ] Unit test covers both cases.

---

## Issue 107 — `pay_to_contact` does not check that the player exists

**Labels:** `bug`, `scout_access`, `validation`

**Description:**
A scout can pay to contact `player_id = 9999` even if that player was never registered. The fee is collected but the contact record references a non-existent player.

**Task:**
Add an optional cross-contract check: if `DataKey::RegistrationContract` is set, call `registration.get_player(player_id)` and return a new `ScoutAccessError::PlayerNotFound = 13` if it fails.

**Additional Requirements:**
- Add `set_registration_contract(env, addr)` admin function.
- The check is skipped if the registration contract address is not set.

**Acceptance Criteria:**
- [ ] With registration contract set, contacting a non-existent player returns `PlayerNotFound`.
- [ ] Without registration contract set, the check is skipped.
- [ ] Unit tests cover both configurations.

---

## Issue 108 — `log_trial_offer` does not advance the player to Level 3 atomically

**Labels:** `bug`, `scout_access`, `cross-contract`

**Description:**
The README and `CONTRACT_REFERENCE.md` state that `log_trial_offer` advances the player to Level 3. The current implementation only records the offer — the comment says "The backend should call progress.advance_level after this succeeds." This is not atomic and creates a race condition.

**Task:**
Add a cross-contract call to `progress.advance_level` inside `log_trial_offer`, similar to how `verification.approve_milestone` calls it. Store the progress contract address in `DataKey::ProgressContract`.

**Additional Requirements:**
- Add `set_progress_contract(env, addr)` admin function.
- Ignore `AlreadyAtMaxLevel` (player already at Level 3).
- Propagate other progress contract errors as `ScoutAccessError::ProgressCallFailed = 14`.

**Acceptance Criteria:**
- [ ] `log_trial_offer` atomically advances the player to Level 3 when the progress contract is set.
- [ ] `AlreadyAtMaxLevel` is silently ignored.
- [ ] Other errors return `ProgressCallFailed`.
- [ ] Unit tests cover all three cases.

---

## Issue 109 — `log_trial_offer` `TrialCounter` overflow uses `expect` instead of returning an error

**Labels:** `bug`, `scout_access`, `error-handling`

**Description:**
`index.checked_add(1).expect("overflow")` panics instead of returning a typed error.

**Task:**
Return `ScoutAccessError::Overflow` instead of panicking.

**Acceptance Criteria:**
- [ ] No `.expect("overflow")` in `scout_access/src/lib.rs`.
- [ ] Unit test simulates overflow by setting `TrialCounter` to `u32::MAX`.

---

## Issue 110 — `get_trial_offer` returns `TrialOfferNotFound` but the error is not documented

**Labels:** `documentation`, `scout_access`, `errors`

**Description:**
`ScoutAccessError::TrialOfferNotFound` exists but has no doc comment and is not listed in the README error table.

**Task:**
Add a `///` doc comment to `TrialOfferNotFound` in `errors.rs` and add it to the README error codes table.

**Additional Requirements:**
- Also add all other `ScoutAccessError` variants that are missing from the README table.

**Acceptance Criteria:**
- [ ] All `ScoutAccessError` variants have doc comments.
- [ ] README error table is complete.

---

## Issue 111 — `scout_access` contract instance storage is never TTL-bumped

**Labels:** `bug`, `scout_access`, `storage`

**Description:**
Same gap as Issues 10, 61, and 78. `Admin`, `Initialized`, `Paused`, `FeeConfig`, `AccumulatedFees`, and `XlmToken` keys in instance storage will expire.

**Task:**
Call `env.storage().instance().extend_ttl(INSTANCE_TTL_MIN, INSTANCE_TTL_MAX)` at the start of every public function.

**Acceptance Criteria:**
- [ ] Every public function extends instance TTL.
- [ ] Constants defined (`INSTANCE_TTL_MIN = 100`, `INSTANCE_TTL_MAX = 500`).

---

## Issue 112 — `Subscription` and `ContactRecord` persistent storage entries are never TTL-bumped

**Labels:** `bug`, `scout_access`, `storage`

**Description:**
`DataKey::Subscription(scout)` and `DataKey::ContactRecord(player_id, scout)` will expire from persistent storage.

**Task:**
Extend TTL on `Subscription` in `subscribe`, `pay_to_contact`, `log_trial_offer`, and `get_subscription`. Extend TTL on `ContactRecord` in `pay_to_contact` and `has_contacted`.

**Additional Requirements:**
- Use `PERSISTENT_TTL_MIN = 500` and `PERSISTENT_TTL_MAX = 2000`.

**Acceptance Criteria:**
- [ ] All relevant functions extend TTL on the appropriate keys.
- [ ] Unit test verifies subscription is accessible after simulated ledger advancement.

---

## Issue 113 — `TrialOffer` persistent storage entries are never TTL-bumped

**Labels:** `bug`, `scout_access`, `storage`

**Description:**
`DataKey::TrialOffer(player_id, index)` and `DataKey::TrialCounter(player_id)` will expire from persistent storage.

**Task:**
Extend TTL on `TrialOffer` in `log_trial_offer` and `get_trial_offer`. Extend TTL on `TrialCounter` in `log_trial_offer` and `get_trial_count`.

**Acceptance Criteria:**
- [ ] All four functions extend TTL on the relevant keys.
- [ ] Unit test verifies trial offer is accessible after simulated ledger advancement.

---

## Issue 114 — `scout_access` has no `transfer_admin` function

**Labels:** `enhancement`, `scout_access`, `admin`

**Description:**
Same gap as Issues 14, 47, and 80. Admin key rotation is impossible.

**Task:**
Implement `transfer_admin(env: Env, new_admin: Address) -> Result<(), ScoutAccessError>` with the same requirements as Issue 14.

**Acceptance Criteria:**
- [ ] Only current admin can call it.
- [ ] `admin_transferred` event emitted.
- [ ] Unit tests cover success and unauthorized cases.

---

## Issue 115 — `pause_contract` / `unpause_contract` in `scout_access` emit no events

**Labels:** `enhancement`, `scout_access`, `events`

**Description:**
Same gap as Issues 12, 46, and 81.

**Task:**
Add `contract_paused` and `contract_unpaused` events to `scout_access/src/events.rs` and call them from the respective functions.

**Acceptance Criteria:**
- [ ] Both events emitted with admin address.
- [ ] Unit tests assert event presence.

---

## Issue 116 — `update_fee_config` emits no event

**Labels:** `enhancement`, `scout_access`, `events`

**Description:**
`update_fee_config` silently updates the fee configuration. The backend indexer and scouts have no way to know fees changed.

**Task:**
Add a `fee_config_updated` event in `events.rs` that publishes the new `FeeConfig` and call it from `update_fee_config`.

**Additional Requirements:**
- Also emit the old `FeeConfig` for comparison.

**Acceptance Criteria:**
- [ ] `fee_config_updated` event is emitted with old and new configs.
- [ ] Unit test asserts both configs in the event data.

---

## Issue 117 — `FeeConfig` has no validation; zero fees are accepted

**Labels:** `bug`, `scout_access`, `validation`

**Description:**
`initialize` and `update_fee_config` accept a `FeeConfig` with all fees set to `0`. A zero contact fee would allow scouts to contact players for free, draining the platform's revenue model.

**Task:**
Add a `validate_fee_config` helper that returns `ScoutAccessError::InvalidInput` (add `InvalidInput = 15`) if any fee field is `0` or negative.

**Additional Requirements:**
- Call `validate_fee_config` in both `initialize` and `update_fee_config`.
- Also validate `sub_duration_secs > 0`.

**Acceptance Criteria:**
- [ ] `initialize` with a zero contact fee returns `InvalidInput`.
- [ ] `update_fee_config` with a zero subscription fee returns `InvalidInput`.
- [ ] Valid fee config succeeds.
- [ ] Unit tests cover all validation cases.

---

## Issue 118 — `subscribe` event does not include the fee amount paid

**Labels:** `enhancement`, `scout_access`, `events`

**Description:**
`scout_subscribed` event only includes the scout address and tier. The backend indexer needs the fee amount to populate the `scout_subscriptions` table without a separate `get_fee_config` call.

**Task:**
Update `scout_subscribed` in `events.rs` to include `fee_paid: i128` in the event data.

**Additional Requirements:**
- Update the call site in `subscribe`.
- Update `CONTRACT_REFERENCE.md`.

**Acceptance Criteria:**
- [ ] Event data includes `fee_paid`.
- [ ] Unit test asserts the correct fee amount in the event.

---

## Issue 119 — `player_contacted` event does not include the fee amount paid

**Labels:** `enhancement`, `scout_access`, `events`

**Description:**
`player_contacted` event only includes `player_id` and `scout`. The backend indexer needs the fee amount for the `contact_records` audit log.

**Task:**
Update `player_contacted` in `events.rs` to include `fee_paid: i128` in the event data.

**Additional Requirements:**
- Update the call site in `pay_to_contact`.

**Acceptance Criteria:**
- [ ] Event data includes `fee_paid`.
- [ ] Unit test asserts the correct fee amount.

---

## Issue 120 — Write a test for `subscribe` with each of the three tiers

**Labels:** `test`, `scout_access`

**Description:**
The existing test only covers `Basic` and `Elite` subscriptions. `Pro` is not tested.

**Task:**
Add `fn test_subscribe_pro_tier()` that subscribes with `SubscriptionTier::Pro`, asserts the tier is stored correctly, and asserts the correct fee was accumulated.

**Acceptance Criteria:**
- [ ] Test exists and passes.
- [ ] Asserts `sub.tier == SubscriptionTier::Pro`.
- [ ] Asserts `get_accumulated_fees() == 3_000_000`.

---

## Issue 121 — Write a test for `withdraw_fees` transferring the correct amount

**Labels:** `test`, `scout_access`

**Description:**
No test verifies that `withdraw_fees` actually transfers the accumulated fees to the recipient's token balance.

**Task:**
Add `fn test_withdraw_fees_transfers_correct_amount()` that:
1. Subscribes a scout (accumulates fees).
2. Calls `withdraw_fees(recipient)`.
3. Asserts the recipient's token balance increased by the accumulated amount.
4. Asserts `get_accumulated_fees() == 0` after withdrawal.

**Acceptance Criteria:**
- [ ] Test exists and passes.
- [ ] Asserts both the recipient balance and the reset accumulated fees.

---

## Issue 122 — Write a test for `log_trial_offer` with a non-Elite subscription

**Labels:** `test`, `scout_access`

**Description:**
The existing test covers the `Pro` tier rejection. A dedicated test for `Basic` tier should also exist.

**Task:**
Add `fn test_trial_offer_rejected_for_basic_tier()` that subscribes with `Basic` and attempts `log_trial_offer`, expecting a panic.

**Acceptance Criteria:**
- [ ] Test exists and passes with `#[should_panic]`.

---

## Issue 123 — Write a test for `get_trial_offer` returning correct `TrialOffer` data

**Labels:** `test`, `scout_access`

**Description:**
`get_trial_offer` is not directly tested for field correctness — only the count and index are asserted.

**Task:**
Add `fn test_get_trial_offer_correct_data()` that logs a trial offer and asserts all fields of the returned `TrialOffer` (`player_id`, `scout`, `details_hash`, `logged_at`).

**Acceptance Criteria:**
- [ ] Test asserts every field of the `TrialOffer` struct.

---

## Issue 124 — Add `#[cfg(test)]` per-contract CI step for `scout_access` (mirrors Issue 35)

**Labels:** `ci`, `scout_access`, `devops`

**Description:**
Consistent with Issues 35, 69, and 95, the CI pipeline should have a dedicated step for the scout_access contract.

**Task:**
Add a `Test scout_access contract` step to `.github/workflows/contract-ci.yml` that runs `cargo test -p scoutchain-scout-access -- --nocapture`.

**Acceptance Criteria:**
- [ ] CI step exists and is named `Test scout_access contract`.
- [ ] CI passes on `main`.

---

## Issue 125 — Write end-to-end integration test: subscribe → pay_to_contact → log_trial_offer flow

**Labels:** `test`, `scout_access`

**Description:**
No single test exercises the complete scout workflow: subscribe with Elite → pay to contact a player → log a trial offer → verify all state changes.

**Task:**
Add `fn test_full_scout_workflow()` that:
1. Initializes the contract with default fees.
2. Mints XLM to a scout.
3. Subscribes the scout with `Elite` tier.
4. Calls `pay_to_contact(scout, player_id = 1)`.
5. Calls `log_trial_offer(scout, player_id = 1, "QmTrialDetails")`.
6. Asserts `has_contacted(scout, 1) == true`.
7. Asserts `get_trial_count(1) == 1`.
8. Asserts `get_accumulated_fees() == elite_sub_fee + contact_fee`.

**Acceptance Criteria:**
- [ ] Test passes end-to-end.
- [ ] All seven assertions pass.
- [ ] Test is self-contained (no external dependencies).
