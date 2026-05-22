# Issues 71–100: `progress` Contract

---

## Issue 71 — `initialize` in `progress` emits no event

**Labels:** `enhancement`, `progress`, `events`

**Description:**
Consistent with Issues 2 and 36, `progress.initialize` is silent. The backend indexer cannot confirm the contract is live.

**Task:**
Emit a `contract_initialized` event at the end of a successful `initialize` call, publishing the admin address.

**Acceptance Criteria:**
- [ ] Event is emitted on first successful `initialize`.
- [ ] No event on duplicate `initialize` call.
- [ ] Unit test asserts event presence.

---

## Issue 72 — `advance_level` does not verify the caller is an authorized validator or scout

**Labels:** `bug`, `progress`, `security`

**Description:**
`advance_level` calls `caller.require_auth()` but does not check whether `caller` is a registered validator or an Elite scout. Any authenticated address can advance a player's level.

**Task:**
Add a `DataKey::AuthorizedCallers` set (or check against the verification contract's validator registry via cross-contract call) to restrict `advance_level` to known validators and the verification contract address.

**Additional Requirements:**
- Add `set_verification_contract(env, addr)` admin function storing `DataKey::VerificationContract`.
- If `DataKey::VerificationContract` is set, only that address may call `advance_level` (since it already validates the caller is an active validator).
- If not set, fall back to the current behavior (for testing without full deployment).

**Acceptance Criteria:**
- [ ] With verification contract set, a random address calling `advance_level` returns `Unauthorized`.
- [ ] The verification contract address can call `advance_level` successfully.
- [ ] Unit tests cover both configurations.

---

## Issue 73 — `advance_level` overflow in `HistoryCounter` uses `expect` instead of returning an error

**Labels:** `bug`, `progress`, `error-handling`

**Description:**
`index.checked_add(1).expect("overflow")` panics instead of returning a typed error.

**Task:**
Add `Overflow = 8` to `ProgressError` and return it instead of panicking.

**Additional Requirements:**
- Update callers to propagate with `?`.

**Acceptance Criteria:**
- [ ] No `.expect("overflow")` in `progress/src/lib.rs`.
- [ ] Unit test simulates overflow by setting `HistoryCounter` to `u32::MAX`.

---

## Issue 74 — `ProgressLevel` enum is duplicated between `registration` and `progress` contracts

**Labels:** `enhancement`, `progress`, `code-quality`

**Description:**
`ProgressLevel` is defined identically in both `registration/src/types.rs` and `progress/src/types.rs`. Any change to the enum must be made in two places, risking drift.

**Task:**
Create a shared workspace crate `contracts/shared/` with a `types` module containing the canonical `ProgressLevel` definition. Both contracts import from it.

**Additional Requirements:**
- Add `scoutchain-shared` to `Cargo.toml` workspace members.
- Both contracts' `types.rs` files re-export `ProgressLevel` from the shared crate.
- All existing tests must still pass.

**Acceptance Criteria:**
- [ ] `ProgressLevel` is defined in exactly one place.
- [ ] Both contracts compile using the shared definition.
- [ ] `cargo test --workspace` passes.

---

## Issue 75 — `get_level` returns `Unverified` for a player that was never registered

**Labels:** `bug`, `progress`, `error-handling`

**Description:**
`get_level` returns `ProgressLevel::Unverified` as a default for any `player_id`, including IDs that were never registered. Callers cannot distinguish "Level 0 player" from "non-existent player".

**Task:**
Change `get_level` to return `Result<ProgressLevel, ProgressError>` and return `ProgressError::PlayerNotFound` when `DataKey::PlayerLevel(player_id)` is absent.

**Additional Requirements:**
- Update `CONTRACT_REFERENCE.md`.
- Update the existing test that calls `get_level`.

**Acceptance Criteria:**
- [ ] `get_level` for an unregistered player returns `PlayerNotFound`.
- [ ] `get_level` for a registered player (after `advance_level`) returns the correct level.
- [ ] Unit tests cover both cases.

---

## Issue 76 — `progress` contract has no `initialize_player` function; levels default to `Unverified` implicitly

**Labels:** `enhancement`, `progress`, `design`

**Description:**
Player levels are implicitly `Unverified` (via `unwrap_or`) until `advance_level` is called. There is no explicit on-chain record that a player exists at Level 0. This makes it impossible to distinguish "player at Level 0" from "player ID never used".

**Task:**
Add `initialize_player(env: Env, caller: Address, player_id: u64) -> Result<(), ProgressError>` that explicitly sets `DataKey::PlayerLevel(player_id)` to `Unverified` and emits a `player_initialized` event.

**Additional Requirements:**
- Only the registration contract (stored in `DataKey::RegistrationContract`) may call this.
- Add `set_registration_contract(env, addr)` admin function.
- Return `AlreadyInitialized` (or a new `PlayerAlreadyInitialized = 9`) if the key already exists.

**Acceptance Criteria:**
- [ ] `initialize_player` sets the level to `Unverified`.
- [ ] Duplicate call returns `PlayerAlreadyInitialized`.
- [ ] Unauthorized caller returns `Unauthorized`.
- [ ] Unit tests cover all three cases.

---

## Issue 77 — `ProgressEntry.milestone_ref` is `0` when `advance_level` is called directly (not via verification)

**Labels:** `bug`, `progress`, `data-integrity`

**Description:**
When `advance_level` is called directly (e.g., in tests or by a scout for Level 3), `milestone_ref` is whatever the caller passes. There is no validation that the referenced milestone index actually exists in the verification contract.

**Task:**
Document this as a known limitation in a code comment. Add a `source: AdvanceLevelSource` enum (`ValidatorMilestone(u32)`, `TrialOffer(u32)`, `Direct`) to `ProgressEntry` to make the call origin explicit.

**Additional Requirements:**
- Update `advance_level` signature to accept `source: AdvanceLevelSource` instead of `milestone_ref: u32`.
- Update `CONTRACT_REFERENCE.md`.

**Acceptance Criteria:**
- [ ] `ProgressEntry` stores the source variant.
- [ ] Existing tests updated to use the new signature.
- [ ] `cargo test --workspace` passes.

---

## Issue 78 — `progress` contract instance storage is never TTL-bumped

**Labels:** `bug`, `progress`, `storage`

**Description:**
Same gap as Issues 10 and 61. `Admin`, `Initialized`, and `Paused` keys in instance storage will expire.

**Task:**
Call `env.storage().instance().extend_ttl(INSTANCE_TTL_MIN, INSTANCE_TTL_MAX)` at the start of every public function.

**Acceptance Criteria:**
- [ ] Every public function extends instance TTL.
- [ ] Constants defined (`INSTANCE_TTL_MIN = 100`, `INSTANCE_TTL_MAX = 500`).

---

## Issue 79 — `PlayerLevel` and `HistoryEntry` persistent storage entries are never TTL-bumped

**Labels:** `bug`, `progress`, `storage`

**Description:**
`DataKey::PlayerLevel(player_id)` and `DataKey::HistoryEntry(player_id, index)` will expire from persistent storage.

**Task:**
Extend TTL on `PlayerLevel` in `advance_level` and `get_level`. Extend TTL on `HistoryEntry` in `advance_level` and `get_history_entry`.

**Additional Requirements:**
- Use `PERSISTENT_TTL_MIN = 500` and `PERSISTENT_TTL_MAX = 2000`.

**Acceptance Criteria:**
- [ ] All four functions extend TTL on the relevant keys.
- [ ] Unit test verifies data is accessible after simulated ledger advancement.

---

## Issue 80 — `progress` contract has no `transfer_admin` function

**Labels:** `enhancement`, `progress`, `admin`

**Description:**
Same gap as Issues 14 and 47. Admin key rotation is impossible.

**Task:**
Implement `transfer_admin(env: Env, new_admin: Address) -> Result<(), ProgressError>` with the same requirements as Issue 14.

**Acceptance Criteria:**
- [ ] Only current admin can call it.
- [ ] `admin_transferred` event emitted.
- [ ] Unit tests cover success and unauthorized cases.

---

## Issue 81 — `pause_contract` / `unpause_contract` in `progress` emit no events

**Labels:** `enhancement`, `progress`, `events`

**Description:**
Same gap as Issues 12 and 46.

**Task:**
Add `contract_paused` and `contract_unpaused` events to `progress/src/events.rs` and call them from the respective functions.

**Acceptance Criteria:**
- [ ] Both events emitted with admin address.
- [ ] Unit tests assert event presence.

---

## Issue 82 — `progress_updated` event does not include `old_level`

**Labels:** `enhancement`, `progress`, `events`

**Description:**
`events::progress_updated` publishes `(player_id, new_level)` but not `old_level`. The backend indexer must infer the previous level from history, adding complexity.

**Task:**
Update `progress_updated` in `events.rs` to include `old_level: &ProgressLevel` in the event data tuple.

**Additional Requirements:**
- Update the call site in `advance_level`.
- Update `CONTRACT_REFERENCE.md`.

**Acceptance Criteria:**
- [ ] Event data is `(player_id, old_level, new_level)`.
- [ ] Unit test asserts all three values in the emitted event.

---

## Issue 83 — `get_history_entry` returns `PlayerNotFound` for a missing entry; use a more specific error

**Labels:** `bug`, `progress`, `error-handling`

**Description:**
`get_history_entry` returns `ProgressError::PlayerNotFound` when the history entry is absent. This is misleading — the player may exist but the index is out of range.

**Task:**
Add `HistoryEntryNotFound = 10` to `ProgressError` and return it from `get_history_entry` when the key is absent.

**Additional Requirements:**
- Update `CONTRACT_REFERENCE.md` error table.

**Acceptance Criteria:**
- [ ] `get_history_entry` with an out-of-range index returns `HistoryEntryNotFound`.
- [ ] Unit test asserts the specific error variant.

---

## Issue 84 — Add `get_full_history` batch query returning all `ProgressEntry` records for a player

**Labels:** `enhancement`, `progress`, `query`

**Description:**
The README lists `get_progress_history(player_id)` as a query function. Currently callers must loop `get_history_entry(player_id, 1..N)`. A batch query reduces round-trips.

**Task:**
Implement `get_progress_history(env: Env, player_id: u64) -> Vec<ProgressEntry>` that collects all history entries.

**Additional Requirements:**
- Cap at 50 entries to bound gas.
- Return empty `Vec` if no history exists.
- Add to `CONTRACT_REFERENCE.md`.

**Acceptance Criteria:**
- [ ] Returns all entries for a player with ≤ 50 records.
- [ ] Returns empty vec for a player with no history.
- [ ] Unit test advances level 3 times and asserts all 3 entries are returned.

---

## Issue 85 — Write a test for `advance_level` when contract is paused

**Labels:** `test`, `progress`

**Description:**
No test verifies that `advance_level` is blocked when the contract is paused.

**Task:**
Add `fn test_advance_level_blocked_when_paused()`: initialize → pause → attempt `advance_level` → expect `ContractPaused`.

**Acceptance Criteria:**
- [ ] Test exists and passes with `#[should_panic]`.

---

## Issue 86 — Write a test for `advance_level` when contract is not initialized

**Labels:** `test`, `progress`

**Description:**
No test verifies that `advance_level` returns `NotInitialized` on an uninitialized contract.

**Task:**
Add `fn test_advance_level_not_initialized()` that deploys the contract without calling `initialize` and attempts `advance_level`.

**Acceptance Criteria:**
- [ ] Test exists and passes with `#[should_panic]`.

---

## Issue 87 — Write a test for `get_history_entry` returning correct `ProgressEntry` data

**Labels:** `test`, `progress`

**Description:**
`get_history_entry` is not directly tested for field correctness — only the count is asserted.

**Task:**
Add `fn test_get_history_entry_correct_data()` that advances level once and asserts all fields of the returned `ProgressEntry` (`old_level`, `new_level`, `updated_by`, `milestone_ref`).

**Acceptance Criteria:**
- [ ] Test asserts every field of the `ProgressEntry` struct.

---

## Issue 88 — Write a test for `get_level` returning `Unverified` for a fresh player

**Labels:** `test`, `progress`

**Description:**
After Issue 75 is resolved, `get_level` will return `PlayerNotFound` for unknown IDs. But for a player explicitly initialized at Level 0, it should return `Unverified`. This needs a dedicated test.

**Task:**
Add `fn test_get_level_unverified_for_new_player()` that calls `initialize_player` (Issue 76) and then asserts `get_level` returns `Unverified`.

**Acceptance Criteria:**
- [ ] Test exists and passes.
- [ ] Depends on Issues 75 and 76 being merged first.

---

## Issue 89 — Write a test for `advance_level` producing correct history count

**Labels:** `test`, `progress`

**Description:**
The existing test asserts `get_history_count == 3` after three advances, but does not test intermediate counts.

**Task:**
Add `fn test_history_count_increments_correctly()` that asserts the count after each individual `advance_level` call (1, 2, 3).

**Acceptance Criteria:**
- [ ] Test asserts count is 1 after first advance, 2 after second, 3 after third.

---

## Issue 90 — Write a test for two different players advancing levels independently

**Labels:** `test`, `progress`

**Description:**
No test verifies that level state is isolated per `player_id` — advancing one player's level should not affect another's.

**Task:**
Add `fn test_two_players_advance_independently()` that advances `player_id = 1` to Level 2 and `player_id = 2` to Level 1, then asserts each has the correct level and history count.

**Acceptance Criteria:**
- [ ] `get_level(1)` returns `PerformanceMilestones`.
- [ ] `get_level(2)` returns `VerifiedIdentity`.
- [ ] `get_history_count(1) == 2`, `get_history_count(2) == 1`.

---

## Issue 91 — `ProgressLevel.next()` is a public method but not documented

**Labels:** `documentation`, `progress`

**Description:**
`ProgressLevel::next()` is a public method with no doc comment. Contributors may not realize it is the canonical source of valid transitions.

**Task:**
Add a `///` doc comment to `next()` explaining it returns the next valid level or `None` at `EliteTier`, and that it is the single source of truth for valid transitions.

**Additional Requirements:**
- Add a `# Examples` section to the doc comment with a code example.

**Acceptance Criteria:**
- [ ] `next()` has a doc comment with an examples section.
- [ ] `cargo doc --no-deps` generates documentation without warnings.

---

## Issue 92 — Add `reset_player_level` admin function for dispute resolution

**Labels:** `enhancement`, `progress`, `admin`

**Description:**
If a milestone is fraudulently approved, there is no way to roll back a player's level. An admin `reset_player_level` function is needed for dispute resolution.

**Task:**
Implement `reset_player_level(env: Env, player_id: u64, target_level: ProgressLevel) -> Result<(), ProgressError>` (admin only) that sets the player's level to `target_level` and records a `ProgressEntry` with `updated_by = admin`.

**Additional Requirements:**
- Emit a `player_level_reset` event with `player_id`, `old_level`, and `target_level`.
- The history entry must be immutable — do not delete existing entries, only add a new one.
- `target_level` can be any valid level including `Unverified`.

**Acceptance Criteria:**
- [ ] Admin can reset a player's level to any tier.
- [ ] A new history entry is added (not deleted).
- [ ] `player_level_reset` event is emitted.
- [ ] Non-admin call returns `Unauthorized`.
- [ ] Unit tests cover success and unauthorized cases.

---

## Issue 93 — Document all `ProgressError` variants with descriptions in `errors.rs`

**Labels:** `documentation`, `progress`

**Description:**
`ProgressError` variants have no doc comments.

**Task:**
Add a `///` doc comment to every variant in `contracts/progress/src/errors.rs`.

**Acceptance Criteria:**
- [ ] Every error variant has a doc comment.
- [ ] `cargo doc --no-deps` succeeds without warnings.

---

## Issue 94 — Document all `DataKey` variants in `progress/src/types.rs`

**Labels:** `documentation`, `progress`

**Description:**
`DataKey` variants in the progress contract have no inline comments.

**Task:**
Add `///` doc comments to every `DataKey` variant in `contracts/progress/src/types.rs`.

**Acceptance Criteria:**
- [ ] Every `DataKey` variant has a doc comment.
- [ ] `cargo doc --no-deps` succeeds without warnings.

---

## Issue 95 — Add `#[cfg(test)]` per-contract CI step for `progress` (mirrors Issue 35)

**Labels:** `ci`, `progress`, `devops`

**Description:**
Consistent with Issues 35 and 69, the CI pipeline should have a dedicated step for the progress contract.

**Task:**
Add a `Test progress contract` step to `.github/workflows/contract-ci.yml` that runs `cargo test -p scoutchain-progress -- --nocapture`.

**Acceptance Criteria:**
- [ ] CI step exists and is named `Test progress contract`.
- [ ] CI passes on `main`.

---

## Issue 96 — `advance_level` does not emit the `milestone_ref` in the event

**Labels:** `enhancement`, `progress`, `events`

**Description:**
`progress_updated` event does not include `milestone_ref`. The backend indexer cannot link a level change back to the specific milestone that triggered it without a separate query.

**Task:**
Update `progress_updated` in `events.rs` to include `milestone_ref: u32` in the event data.

**Additional Requirements:**
- Update the call site in `advance_level`.
- Update `CONTRACT_REFERENCE.md`.

**Acceptance Criteria:**
- [ ] Event data includes `milestone_ref`.
- [ ] Unit test asserts the value in the emitted event.

---

## Issue 97 — Write a test for `health()` returning `false` before `initialize` in `progress`

**Labels:** `test`, `progress`

**Description:**
`health()` should return `false` on an uninitialized progress contract, but this is not tested.

**Task:**
Add `fn test_health_false_before_initialize()` that deploys the contract without calling `initialize` and asserts `health() == false`.

**Acceptance Criteria:**
- [ ] Test exists and passes.

---

## Issue 98 — Write a test for `pause_contract` and `unpause_contract` in `progress`

**Labels:** `test`, `progress`

**Description:**
The circuit-breaker functions exist but are not tested in the progress contract.

**Task:**
Add `fn test_pause_and_unpause()`: initialize → pause → assert `advance_level` fails → unpause → assert `advance_level` succeeds.

**Acceptance Criteria:**
- [ ] Test exists and passes.
- [ ] Uses `#[should_panic]` for the paused call or explicit error matching.

---

## Issue 99 — `ProgressEntry` should store the ledger sequence number for auditability

**Labels:** `enhancement`, `progress`, `types`

**Description:**
Consistent with Issue 67 for milestones, `ProgressEntry` should store the ledger sequence number at the time of the level change for tamper-proof auditability.

**Task:**
Add `ledger_sequence: u32` to `ProgressEntry` and populate it with `env.ledger().sequence()` in `advance_level`.

**Additional Requirements:**
- Update `CONTRACT_REFERENCE.md`.

**Acceptance Criteria:**
- [ ] `ProgressEntry` has `ledger_sequence: u32`.
- [ ] Field is populated correctly.
- [ ] Unit test asserts `ledger_sequence > 0`.

---

## Issue 100 — Write integration test: full level progression from 0 to 3 with history verification

**Labels:** `test`, `progress`

**Description:**
No single test exercises the complete level progression from `Unverified` to `EliteTier` and verifies the full history chain.

**Task:**
Add `fn test_full_level_progression_with_history()` that:
1. Advances player from Level 0 → 1 → 2 → 3.
2. Asserts `get_level` returns `EliteTier`.
3. Asserts `get_history_count` returns 3.
4. Reads each `ProgressEntry` and asserts `old_level` and `new_level` form a valid chain.

**Acceptance Criteria:**
- [ ] Test passes end-to-end.
- [ ] History chain is `(Unverified→VerifiedIdentity)`, `(VerifiedIdentity→PerformanceMilestones)`, `(PerformanceMilestones→EliteTier)`.
- [ ] Attempting a 4th advance returns `AlreadyAtMaxLevel`.
