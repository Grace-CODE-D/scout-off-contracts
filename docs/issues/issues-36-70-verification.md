# Issues 36–70: `verification` Contract

---

## Issue 36 — `initialize` in `verification` emits no event

**Labels:** `enhancement`, `verification`, `events`

**Description:**
Consistent with Issue 2 for the registration contract, `verification.initialize` is silent. The backend indexer cannot confirm the contract is live.

**Task:**
Emit a `contract_initialized` event at the end of a successful `initialize` call, publishing the admin address.

**Additional Requirements:**
- Follow the `Symbol::new(env, "contract_initialized")` pattern.
- No event on the already-initialized error path.

**Acceptance Criteria:**
- [ ] Event is emitted on first successful `initialize`.
- [ ] No event on duplicate `initialize` call.
- [ ] Unit test asserts event presence.

---

## Issue 37 — `register_validator` does not validate `credentials` string length

**Labels:** `bug`, `verification`, `validation`

**Description:**
`Validator.credentials` is an unbounded `String`. A malicious admin could store a multi-kilobyte credential string, bloating persistent storage.

**Task:**
Enforce a maximum of 256 bytes on `credentials` in `register_validator`, returning `VerificationError::InvalidInput` (error code 9) on violation.

**Additional Requirements:**
- Define `MAX_CREDENTIALS_LEN: u32 = 256` as a constant in `lib.rs`.

**Acceptance Criteria:**
- [ ] A 257-byte credentials string returns `InvalidInput`.
- [ ] A 256-byte credentials string succeeds.
- [ ] Unit test covers both boundary values.

---

## Issue 38 — `register_validator` does not emit a `validator_registered` event with credentials

**Labels:** `enhancement`, `verification`, `events`

**Description:**
`events::validator_registered` currently only publishes the wallet address. The backend indexer also needs the credential label to populate the `validators` table without a separate `get_validator` call.

**Task:**
Update `validator_registered` in `events.rs` to include `credentials: &String` as event data alongside the wallet.

**Additional Requirements:**
- Change the event data from `wallet.clone()` to a tuple `(wallet.clone(), credentials.clone())`.
- Update the call site in `register_validator`.

**Acceptance Criteria:**
- [ ] Event data is `(wallet, credentials)`.
- [ ] Unit test asserts both values are present in the emitted event.

---

## Issue 39 — `revoke_validator` should emit the reason for revocation

**Labels:** `enhancement`, `verification`, `events`

**Description:**
`validator_revoked` only emits the wallet. Auditors and the backend indexer have no way to record why a validator was revoked.

**Task:**
Add an optional `reason: Option<String>` parameter to `revoke_validator` and include it in the `validator_revoked` event data.

**Additional Requirements:**
- If `reason` is `Some`, validate length ≤ 128 bytes.
- Existing callers that pass `None` must still compile.

**Acceptance Criteria:**
- [ ] `revoke_validator` accepts an optional reason.
- [ ] Event data includes the reason (or empty string if `None`).
- [ ] Unit test covers both `Some` and `None` cases.

---

## Issue 40 — `approve_milestone` does not validate `description` string length

**Labels:** `bug`, `verification`, `validation`

**Description:**
`Milestone.description` is unbounded. A validator could submit a very long description, inflating persistent storage costs.

**Task:**
Enforce a maximum of 256 bytes on `description` in `approve_milestone`, returning `VerificationError::InvalidInput`.

**Additional Requirements:**
- Define `MAX_DESCRIPTION_LEN: u32 = 256` as a constant.

**Acceptance Criteria:**
- [ ] A 257-byte description returns `InvalidInput`.
- [ ] A 256-byte description succeeds.
- [ ] Unit test covers both boundary values.

---

## Issue 41 — `approve_milestone` does not validate `evidence_hash` format

**Labels:** `bug`, `verification`, `validation`

**Description:**
`evidence_hash` should be a valid IPFS CID (starts with `"Qm"` for CIDv0 or `"bafy"` for CIDv1). Any arbitrary string is currently accepted.

**Task:**
Add a prefix check: `evidence_hash` must start with `"Qm"` or `"bafy"`. Return `VerificationError::InvalidInput` otherwise.

**Additional Requirements:**
- Define the valid prefixes as constants.
- Also enforce a maximum length of 128 bytes on `evidence_hash`.

**Acceptance Criteria:**
- [ ] `"QmTest123"` passes validation.
- [ ] `"bafyTest"` passes validation.
- [ ] `"http://example.com"` returns `InvalidInput`.
- [ ] Unit tests cover all three cases.

---

## Issue 42 — `approve_milestone` cross-contract call silently ignores all errors

**Labels:** `bug`, `verification`, `cross-contract`

**Description:**
The `try_advance_level` call result is discarded with `let _ = …`. If the progress contract returns an unexpected error (e.g., `NotInitialized`, `ContractPaused`), the milestone is still recorded but the level is not advanced — with no indication to the caller.

**Task:**
Distinguish between `AlreadyAtMaxLevel` (acceptable — ignore) and all other errors (propagate as `VerificationError::InvalidInput` or a new `ProgressCallFailed` error variant).

**Additional Requirements:**
- Add `ProgressCallFailed = 10` to `VerificationError`.
- Only ignore `ProgressError::AlreadyAtMaxLevel`; propagate everything else.

**Acceptance Criteria:**
- [ ] `AlreadyAtMaxLevel` from progress contract is silently ignored.
- [ ] Any other progress contract error causes `approve_milestone` to return `ProgressCallFailed`.
- [ ] Unit test mocks a failing progress contract and asserts the error.

---

## Issue 43 — `set_progress_contract` has no guard against being called twice

**Labels:** `bug`, `verification`, `admin`

**Description:**
`set_progress_contract` can be called multiple times, silently overwriting the progress contract address. A misconfigured re-call could break the cross-contract link.

**Task:**
Add a `DataKey::ProgressContractSet` boolean flag. If already set, return a new `VerificationError::AlreadyConfigured = 11` error. Add a separate `update_progress_contract` function (admin only) for intentional updates.

**Additional Requirements:**
- `update_progress_contract` emits a `progress_contract_updated` event with old and new addresses.

**Acceptance Criteria:**
- [ ] Second call to `set_progress_contract` returns `AlreadyConfigured`.
- [ ] `update_progress_contract` succeeds and emits the event.
- [ ] Unit tests cover both paths.

---

## Issue 44 — `get_milestone` returns `InvalidInput` for a missing milestone; use a more specific error

**Labels:** `bug`, `verification`, `error-handling`

**Description:**
`get_milestone` returns `VerificationError::InvalidInput` when the milestone does not exist. This is ambiguous — `InvalidInput` implies bad parameters, not a missing record.

**Task:**
Add `MilestoneNotFound = 12` to `VerificationError` and return it from `get_milestone` when the key is absent.

**Additional Requirements:**
- Update `CONTRACT_REFERENCE.md` error table.

**Acceptance Criteria:**
- [ ] `get_milestone` with a non-existent index returns `MilestoneNotFound`.
- [ ] Unit test asserts the specific error variant.

---

## Issue 45 — `is_active_validator` should be renamed to `get_validator_status` and return a richer type

**Labels:** `enhancement`, `verification`, `query`

**Description:**
`is_active_validator` returns a plain `bool`. Callers cannot distinguish "not registered" from "registered but revoked" without a separate `get_validator` call.

**Task:**
Add a `ValidatorStatus` enum (`NotRegistered`, `Active`, `Revoked`) to `types.rs` and implement `get_validator_status(env: Env, wallet: Address) -> ValidatorStatus`.

**Additional Requirements:**
- Keep `is_active_validator` as a deprecated wrapper that calls `get_validator_status` internally.
- Add `#[deprecated]` attribute to `is_active_validator`.

**Acceptance Criteria:**
- [ ] `get_validator_status` returns `NotRegistered` for unknown wallets.
- [ ] Returns `Active` for registered, active validators.
- [ ] Returns `Revoked` for revoked validators.
- [ ] Unit tests cover all three states.

---

## Issue 46 — `pause_contract` / `unpause_contract` in `verification` emit no events

**Labels:** `enhancement`, `verification`, `events`

**Description:**
Same gap as Issue 12 for the registration contract. The backend cannot detect circuit-breaker state changes.

**Task:**
Add `contract_paused` and `contract_unpaused` events to `verification/src/events.rs` and call them from the respective functions.

**Acceptance Criteria:**
- [ ] Both events are emitted with the admin address.
- [ ] Unit tests assert event presence.

---

## Issue 47 — `verification` contract has no `transfer_admin` function

**Labels:** `enhancement`, `verification`, `admin`

**Description:**
Same gap as Issue 14 for the registration contract. Admin key rotation is impossible.

**Task:**
Implement `transfer_admin(env: Env, new_admin: Address) -> Result<(), VerificationError>` with the same requirements as Issue 14.

**Acceptance Criteria:**
- [ ] Only current admin can call it.
- [ ] `admin_transferred` event emitted.
- [ ] Old admin loses access after transfer.
- [ ] Unit tests cover success and unauthorized cases.

---

## Issue 48 — Validator persistent storage entries are never TTL-bumped

**Labels:** `bug`, `verification`, `storage`

**Description:**
`Validator` records in persistent storage will expire after the default TTL. An expired validator record would cause `approve_milestone` to return `ValidatorNotFound` for a previously valid validator.

**Task:**
Call `extend_ttl` on `DataKey::Validator(wallet)` in `register_validator`, `revoke_validator`, `approve_milestone`, and `get_validator`.

**Additional Requirements:**
- Use `PERSISTENT_TTL_MIN: u32 = 500` and `PERSISTENT_TTL_MAX: u32 = 2000` constants.

**Acceptance Criteria:**
- [ ] All four functions extend TTL on the validator key.
- [ ] Unit test simulates ledger advancement and verifies validator is still accessible.

---

## Issue 49 — Milestone persistent storage entries are never TTL-bumped

**Labels:** `bug`, `verification`, `storage`

**Description:**
`Milestone` records in persistent storage will expire. Expired milestones would cause `get_milestone` to return `MilestoneNotFound` for valid historical records.

**Task:**
Call `extend_ttl` on `DataKey::Milestone(player_id, index)` in `approve_milestone` and `get_milestone`.

**Additional Requirements:**
- Also bump `DataKey::MilestoneCounter(player_id)` in `approve_milestone`.

**Acceptance Criteria:**
- [ ] `approve_milestone` extends TTL on both the milestone and counter keys.
- [ ] `get_milestone` extends TTL on the milestone key.
- [ ] Unit test verifies milestone is accessible after simulated ledger advancement.

---

## Issue 50 — Add `get_all_milestones_for_player` query that returns a `Vec<Milestone>`

**Labels:** `enhancement`, `verification`, `query`

**Description:**
Callers must currently loop `get_milestone(player_id, 1..N)` to retrieve all milestones. A batch query would reduce round-trips for the frontend.

**Task:**
Implement `get_all_milestones_for_player(env: Env, player_id: u64) -> Vec<Milestone>` that collects all milestones up to `get_milestone_count`.

**Additional Requirements:**
- Cap at 50 milestones to bound gas; document the limit.
- Return an empty `Vec` if the player has no milestones.

**Acceptance Criteria:**
- [ ] Returns all milestones for a player with ≤ 50 records.
- [ ] Returns empty vec for a player with no milestones.
- [ ] Unit test registers 3 milestones and asserts all 3 are returned.

---

## Issue 51 — `approve_milestone` does not check if the player exists in the registration contract

**Labels:** `enhancement`, `verification`, `cross-contract`

**Description:**
A validator can approve a milestone for `player_id = 9999` even if that player was never registered. The milestone is stored but references a non-existent player.

**Task:**
Add an optional cross-contract check: if `DataKey::RegistrationContract` is set, call `registration.get_player(player_id)` and return `VerificationError::PlayerNotFound` if it fails.

**Additional Requirements:**
- Add `set_registration_contract(env, addr)` admin function (similar to `set_progress_contract`).
- The check is skipped if the registration contract address is not set (backward-compatible).

**Acceptance Criteria:**
- [ ] With registration contract set, approving a milestone for a non-existent player returns `PlayerNotFound`.
- [ ] Without registration contract set, the check is skipped.
- [ ] Unit tests cover both configurations.

---

## Issue 52 — Write a test for `register_validator` duplicate prevention

**Labels:** `test`, `verification`

**Description:**
`register_validator` returns `ValidatorAlreadyRegistered` on duplicate, but no test covers this path.

**Task:**
Add `fn test_register_validator_duplicate_fails()` that registers the same wallet twice and expects a panic.

**Acceptance Criteria:**
- [ ] Test exists and passes with `#[should_panic]`.

---

## Issue 53 — Write a test for `approve_milestone` when contract is paused

**Labels:** `test`, `verification`

**Description:**
No test verifies that `approve_milestone` is blocked when the contract is paused.

**Task:**
Add `fn test_approve_milestone_blocked_when_paused()`: initialize → register validator → pause → attempt `approve_milestone` → expect `ContractPaused`.

**Acceptance Criteria:**
- [ ] Test exists and passes.

---

## Issue 54 — Write a test for `get_milestone` with a valid index

**Labels:** `test`, `verification`

**Description:**
`get_milestone` is not directly tested — only `get_milestone_count` is asserted in existing tests.

**Task:**
Add `fn test_get_milestone_returns_correct_data()` that approves a milestone and then calls `get_milestone(player_id, 1)` and asserts all fields match.

**Acceptance Criteria:**
- [ ] Test asserts `description`, `evidence_hash`, `validator`, and `player_id` fields.

---

## Issue 55 — Write a test for `get_validator` returning correct `Validator` struct

**Labels:** `test`, `verification`

**Description:**
`get_validator` is not directly tested.

**Task:**
Add `fn test_get_validator_returns_correct_data()` that registers a validator and asserts all fields of the returned `Validator` struct.

**Acceptance Criteria:**
- [ ] Test asserts `wallet`, `credentials`, `active`, and `registered_at` fields.

---

## Issue 56 — Write a test for `get_validator` returning `ValidatorNotFound`

**Labels:** `test`, `verification`

**Description:**
No test covers the not-found path of `get_validator`.

**Task:**
Add `fn test_get_validator_not_found()` that calls `get_validator` with an unregistered wallet and expects a panic.

**Acceptance Criteria:**
- [ ] Test exists and passes with `#[should_panic]`.

---

## Issue 57 — Write a test for `health()` returning `false` before `initialize`

**Labels:** `test`, `verification`

**Description:**
`health()` should return `false` on an uninitialized contract, but this is not tested.

**Task:**
Add `fn test_health_false_before_initialize()` that deploys the contract without calling `initialize` and asserts `health() == false`.

**Acceptance Criteria:**
- [ ] Test exists and passes.

---

## Issue 58 — `MilestoneCounter` overflow uses `expect` instead of returning an error

**Labels:** `bug`, `verification`, `error-handling`

**Description:**
`index.checked_add(1).expect("overflow")` in `approve_milestone` panics on overflow instead of returning a typed error.

**Task:**
Return `VerificationError::InvalidInput` (or a new `Overflow` variant) instead of panicking.

**Additional Requirements:**
- Add `Overflow = 13` to `VerificationError`.

**Acceptance Criteria:**
- [ ] No `.expect("overflow")` in `verification/src/lib.rs`.
- [ ] Unit test simulates overflow by setting the counter to `u32::MAX`.

---

## Issue 59 — Add `get_validators_list` query returning all active validators

**Labels:** `enhancement`, `verification`, `query`

**Description:**
The README lists `get_validators()` as a query function but it is not implemented. The admin panel needs to display all active validators.

**Task:**
Maintain a `DataKey::ValidatorIndex` (`Vec<Address>`) updated by `register_validator` and `revoke_validator`. Implement `get_validators(env: Env) -> Vec<Validator>` that returns all validators (active and revoked).

**Additional Requirements:**
- Cap at 100 validators to bound gas.
- Add to `CONTRACT_REFERENCE.md`.

**Acceptance Criteria:**
- [ ] Returns all registered validators.
- [ ] Includes revoked validators (callers can filter by `active` field).
- [ ] Unit test registers 3 validators, revokes 1, and asserts all 3 are returned.

---

## Issue 60 — `approve_milestone` event does not include `description` or `evidence_hash`

**Labels:** `enhancement`, `verification`, `events`

**Description:**
`milestone_approved` only emits `player_id` and `validator`. The backend indexer must make a separate `get_milestone` call to populate the `milestones` table.

**Task:**
Update `milestone_approved` in `events.rs` to include `milestone_index: u32`, `description: &String`, and `evidence_hash: &String` in the event data tuple.

**Additional Requirements:**
- Update the call site in `approve_milestone`.
- Update `CONTRACT_REFERENCE.md` events table.

**Acceptance Criteria:**
- [ ] Event data includes all four fields.
- [ ] Unit test asserts the full event payload.

---

## Issue 61 — `verification` contract instance storage is never TTL-bumped

**Labels:** `bug`, `verification`, `storage`

**Description:**
Same gap as Issue 10 for the registration contract. `Admin`, `Initialized`, `Paused`, and `ProgressContract` keys in instance storage will expire.

**Task:**
Call `env.storage().instance().extend_ttl(INSTANCE_TTL_MIN, INSTANCE_TTL_MAX)` at the start of every public function.

**Additional Requirements:**
- Use the same constants as the registration contract (`INSTANCE_TTL_MIN = 100`, `INSTANCE_TTL_MAX = 500`).

**Acceptance Criteria:**
- [ ] Every public function extends instance TTL.
- [ ] Unit test verifies contract is still functional after simulated ledger advancement.

---

## Issue 62 — Add `update_validator_credentials` function for credential updates

**Labels:** `enhancement`, `verification`, `admin`

**Description:**
Once a validator is registered, their credentials cannot be updated. A coach who upgrades from a UEFA C to UEFA B license has no way to reflect this on-chain.

**Task:**
Implement `update_validator_credentials(env: Env, wallet: Address, new_credentials: String) -> Result<(), VerificationError>` (admin only).

**Additional Requirements:**
- Validate `new_credentials` length ≤ 256 bytes.
- Emit a `validator_credentials_updated` event with wallet and new credentials.
- Return `ValidatorNotFound` if the wallet is not registered.

**Acceptance Criteria:**
- [ ] Admin can update credentials.
- [ ] Non-admin call returns `Unauthorized`.
- [ ] Event is emitted.
- [ ] Unit tests cover success, unauthorized, and not-found cases.

---

## Issue 63 — Write a test for `set_progress_contract` and verify cross-contract address is stored

**Labels:** `test`, `verification`

**Description:**
`set_progress_contract` is called in production but has no dedicated unit test verifying the address is stored correctly.

**Task:**
Add `fn test_set_progress_contract_stores_address()` that calls `set_progress_contract` and then reads `DataKey::ProgressContract` from instance storage to assert the address matches.

**Acceptance Criteria:**
- [ ] Test exists and passes.
- [ ] Asserts the stored address equals the one passed in.

---

## Issue 64 — Write a test for `revoke_validator` on a non-existent wallet

**Labels:** `test`, `verification`

**Description:**
`revoke_validator` returns `ValidatorNotFound` for unknown wallets, but this path is not tested.

**Task:**
Add `fn test_revoke_nonexistent_validator_fails()` with `#[should_panic]`.

**Acceptance Criteria:**
- [ ] Test exists and passes.

---

## Issue 65 — Document the cross-contract wiring requirement in `verification/src/lib.rs`

**Labels:** `documentation`, `verification`

**Description:**
The comment above the `progress_contract` module import explains the wiring requirement, but it is easy to miss. New contributors frequently skip `set_progress_contract` and wonder why levels don't advance.

**Task:**
Add a prominent `// IMPORTANT:` block comment at the top of `lib.rs` (below the `use` statements) explaining that `set_progress_contract` must be called after deployment, with a reference to `scripts/initialize.sh`.

**Additional Requirements:**
- Add the same warning to `docs/DEPLOYMENT.md` under a "Common Mistakes" section.

**Acceptance Criteria:**
- [ ] Comment is present in `lib.rs`.
- [ ] `DEPLOYMENT.md` has a "Common Mistakes" section mentioning the wiring step.

---

## Issue 66 — Add `milestone_count_per_validator` query for analytics

**Labels:** `enhancement`, `verification`, `query`

**Description:**
The admin panel needs to show how many milestones each validator has approved. There is no such query.

**Task:**
Maintain a `DataKey::ValidatorMilestoneCount(Address)` counter incremented in `approve_milestone`. Implement `get_validator_milestone_count(env: Env, wallet: Address) -> u32`.

**Additional Requirements:**
- Return `0` for unregistered validators (graceful default).

**Acceptance Criteria:**
- [ ] Counter increments correctly with each `approve_milestone` call.
- [ ] Returns `0` for unknown wallets.
- [ ] Unit test approves 3 milestones from the same validator and asserts count is 3.

---

## Issue 67 — `approve_milestone` should record the ledger sequence number alongside the timestamp

**Labels:** `enhancement`, `verification`, `types`

**Description:**
`Milestone.approved_at` stores a Unix timestamp. For tamper-proof auditability, the ledger sequence number at the time of approval should also be stored, as it is immutable and verifiable on-chain.

**Task:**
Add `ledger_sequence: u32` to the `Milestone` struct and populate it with `env.ledger().sequence()` in `approve_milestone`.

**Additional Requirements:**
- Update `CONTRACT_REFERENCE.md` to document the new field.

**Acceptance Criteria:**
- [ ] `Milestone` struct has `ledger_sequence: u32`.
- [ ] Field is populated correctly.
- [ ] Unit test asserts `ledger_sequence > 0`.

---

## Issue 68 — Write a test for multiple validators approving milestones for the same player

**Labels:** `test`, `verification`

**Description:**
No test covers the scenario where two different validators each approve a milestone for the same player.

**Task:**
Add `fn test_two_validators_approve_milestones_for_same_player()` that registers two validators, each approves one milestone for `player_id = 1`, and asserts `get_milestone_count(1) == 2`.

**Acceptance Criteria:**
- [ ] Test exists and passes.
- [ ] Asserts each milestone has the correct `validator` address.

---

## Issue 69 — Add `#[cfg(test)]` per-contract CI step for `verification` (mirrors Issue 35)

**Labels:** `ci`, `verification`, `devops`

**Description:**
Consistent with Issue 35, the CI pipeline should have a dedicated step for the verification contract.

**Task:**
Add a `Test verification contract` step to `.github/workflows/contract-ci.yml` that runs `cargo test -p scoutchain-verification -- --nocapture`.

**Acceptance Criteria:**
- [ ] CI step exists and is named `Test verification contract`.
- [ ] CI passes on `main`.

---

## Issue 70 — Document all `VerificationError` variants with descriptions in `errors.rs`

**Labels:** `documentation`, `verification`

**Description:**
`VerificationError` variants have no doc comments. Contributors cannot tell when each error is returned without reading the full `lib.rs`.

**Task:**
Add a `///` doc comment to every variant in `contracts/verification/src/errors.rs` explaining when it is returned.

**Additional Requirements:**
- `cargo doc --no-deps` must generate documentation without warnings.

**Acceptance Criteria:**
- [ ] Every error variant has a doc comment.
- [ ] `cargo doc` succeeds without warnings.
