# Issues 1–35: `registration` Contract

---

## Issue 1 — `initialize` should reject a zero/default Address as admin

**Labels:** `bug`, `registration`, `security`

**Description:**
`initialize` currently accepts any `Address` value including a zeroed-out address. Passing a default address would lock admin control to an unowned key.

**Task:**
Add an explicit check that `admin` is not the zero/default address before writing it to instance storage.

**Additional Requirements:**
- Return `ScoutChainError::InvalidInput` (error code 13) on failure.
- The check must happen before `require_auth()` so the error is deterministic.

**Acceptance Criteria:**
- [ ] Calling `initialize` with a zero address returns `InvalidInput`.
- [ ] Calling `initialize` with a valid address succeeds as before.
- [ ] Unit test covers both branches.

---

## Issue 2 — `initialize` emits no event; add an `contract_initialized` event

**Labels:** `enhancement`, `registration`, `events`

**Description:**
All other state-changing functions emit events for off-chain indexing, but `initialize` is silent. The backend indexer cannot distinguish a freshly deployed contract from one that was never set up.

**Task:**
Emit a `contract_initialized` event from `events.rs` at the end of a successful `initialize` call, publishing the admin address as the event data.

**Additional Requirements:**
- Follow the existing `Symbol::new(env, "…")` pattern in `events.rs`.
- Do not emit the event if `initialize` returns an error.

**Acceptance Criteria:**
- [ ] `contract_initialized` event is published with the admin address.
- [ ] No event is emitted when `initialize` fails (already-initialized path).
- [ ] Unit test asserts the event is present in `env.events()`.

---

## Issue 3 — `register_player` does not validate `age` field in `PlayerVitals`

**Labels:** `bug`, `registration`, `validation`

**Description:**
`PlayerVitals.age` is a `u32` so it accepts values like `0` or `300`. There is no on-chain guard preventing nonsensical ages from being stored permanently.

**Task:**
Add a validation step in `register_player` that rejects ages outside the range `[10, 60]` with `ScoutChainError::InvalidInput`.

**Additional Requirements:**
- The range `[10, 60]` should be defined as named constants (`MIN_PLAYER_AGE`, `MAX_PLAYER_AGE`) at the top of `lib.rs`.
- `update_profile` does not change vitals, so no change needed there.

**Acceptance Criteria:**
- [ ] `register_player` with `age = 0` returns `InvalidInput`.
- [ ] `register_player` with `age = 300` returns `InvalidInput`.
- [ ] `register_player` with `age = 18` succeeds.
- [ ] Unit tests cover boundary values (9, 10, 60, 61).

---

## Issue 4 — `register_player` does not validate that `ipfs_hashes` is non-empty

**Labels:** `bug`, `registration`, `validation`

**Description:**
A player can register with an empty `ipfs_hashes` vector, producing a profile with no highlight reel or photo. Scouts browsing the talent pool would see profiles with no media.

**Task:**
Reject `register_player` calls where `ipfs_hashes.len() == 0` with `ScoutChainError::InvalidInput`.

**Additional Requirements:**
- Also enforce a maximum of 10 hashes per registration to prevent storage bloat.
- Define `MAX_IPFS_HASHES: u32 = 10` as a constant.

**Acceptance Criteria:**
- [ ] Empty `ipfs_hashes` returns `InvalidInput`.
- [ ] More than 10 hashes returns `InvalidInput`.
- [ ] Exactly 1 and exactly 10 hashes succeed.
- [ ] Unit tests cover all four cases.

---

## Issue 5 — `register_player` does not validate `position` string length

**Labels:** `bug`, `registration`, `validation`

**Description:**
`PlayerVitals.position` is an unbounded `String`. An attacker could submit a multi-kilobyte string, inflating persistent storage costs for the contract.

**Task:**
Enforce a maximum length of 64 bytes on `position`, `region`, and `nationality` strings inside `register_player`.

**Additional Requirements:**
- Return `ScoutChainError::InvalidInput` on violation.
- Define `MAX_STRING_LEN: u32 = 64` as a constant.
- Reuse the same constant for all three fields.

**Acceptance Criteria:**
- [ ] A 65-byte `position` string returns `InvalidInput`.
- [ ] A 64-byte `position` string succeeds.
- [ ] Same validation applies to `region` and `nationality`.
- [ ] Unit tests cover each field independently.

---

## Issue 6 — `update_profile` should validate new `ipfs_hashes` the same way as `register_player`

**Labels:** `bug`, `registration`, `validation`

**Description:**
`register_player` (after Issue 4 is fixed) will validate `ipfs_hashes`, but `update_profile` currently has no such check. A player could clear all hashes or add more than 10 via an update.

**Task:**
Apply the same non-empty and max-10 validation to `update_profile`.

**Additional Requirements:**
- Reuse the `MAX_IPFS_HASHES` constant introduced in Issue 4.
- Return `ScoutChainError::InvalidInput` on violation.

**Acceptance Criteria:**
- [ ] `update_profile` with empty hashes returns `InvalidInput`.
- [ ] `update_profile` with 11 hashes returns `InvalidInput`.
- [ ] Valid update succeeds and persists new hashes.
- [ ] Unit test verifies the stored hashes match the updated list.

---

## Issue 7 — `update_profile` emits a generic `prof_upd` symbol; rename to full `profile_updated`

**Labels:** `enhancement`, `registration`, `events`

**Description:**
`events::profile_updated` uses `symbol_short!("prof_upd")` which is truncated and inconsistent with the other event names that use full `Symbol::new(env, "…")` strings.

**Task:**
Replace `symbol_short!("prof_upd")` with `Symbol::new(env, "profile_updated")` in `events.rs`.

**Additional Requirements:**
- Update any downstream indexer documentation or `CONTRACT_REFERENCE.md` to reflect the new event name.
- Ensure the event also publishes `player_id` as data (currently it does — keep that).

**Acceptance Criteria:**
- [ ] Event topic is `"profile_updated"` not `"prof_upd"`.
- [ ] `CONTRACT_REFERENCE.md` lists the corrected event name.
- [ ] Existing `update_profile` test still passes.

---

## Issue 8 — `register_scout` does not validate `region` string length

**Labels:** `bug`, `registration`, `validation`

**Description:**
`ScoutProfile.region` is an unbounded `String`. Consistent with player vitals validation, scout region should be capped.

**Task:**
Enforce a maximum of 128 bytes on the `region` parameter in `register_scout`, returning `ScoutChainError::InvalidInput` on violation.

**Additional Requirements:**
- Define `MAX_REGION_LEN: u32 = 128` as a constant (separate from `MAX_STRING_LEN` which is 64).

**Acceptance Criteria:**
- [ ] A 129-byte region string returns `InvalidInput`.
- [ ] A 128-byte region string succeeds.
- [ ] Unit test covers both boundary values.

---

## Issue 9 — Add `get_scout_by_wallet` query function to `registration` contract

**Labels:** `enhancement`, `registration`, `query`

**Description:**
`get_player_by_wallet` exists for players but there is no equivalent for scouts. Frontend and backend need to look up a scout profile by wallet address without knowing the `scout_id`.

**Task:**
Implement `get_scout_by_wallet(env: Env, wallet: Address) -> Result<ScoutProfile, ScoutChainError>` using the existing `DataKey::ScoutByWallet` index.

**Additional Requirements:**
- Return `ScoutChainError::ScoutNotFound` (error code 12) if the wallet is not registered.
- Add the function to `CONTRACT_REFERENCE.md`.

**Acceptance Criteria:**
- [ ] Returns the correct `ScoutProfile` for a registered wallet.
- [ ] Returns `ScoutNotFound` for an unregistered wallet.
- [ ] Unit test covers both cases.

---

## Issue 10 — `PlayerCounter` and `ScoutCounter` are stored in instance storage but never TTL-bumped

**Labels:** `bug`, `registration`, `storage`

**Description:**
Soroban instance storage entries expire after a fixed number of ledgers unless their TTL is extended. `PlayerCounter` and `ScoutCounter` are in instance storage and will expire, causing `next_player_id` to reset to 1 and create duplicate IDs.

**Task:**
Call `env.storage().instance().extend_ttl(min_ledgers, max_ledgers)` at the start of every public function that reads or writes instance storage.

**Additional Requirements:**
- Use constants `INSTANCE_TTL_MIN: u32 = 100` and `INSTANCE_TTL_MAX: u32 = 500` defined at the top of `lib.rs`.
- Document the TTL strategy in a code comment.

**Acceptance Criteria:**
- [ ] Every public function calls `extend_ttl` on instance storage.
- [ ] Constants are defined and used (no magic numbers).
- [ ] Unit test registers 3 players and verifies IDs are 1, 2, 3 (no reset).

---

## Issue 11 — Persistent player and scout profiles are never TTL-bumped on read

**Labels:** `bug`, `registration`, `storage`

**Description:**
`get_player` and `get_scout` read from persistent storage without extending TTL. Long-lived profiles will expire and become inaccessible.

**Task:**
After each successful `get` from persistent storage in `load_player`, `get_scout`, and `get_player_by_wallet`, call `extend_ttl` on the relevant key.

**Additional Requirements:**
- Use constants `PERSISTENT_TTL_MIN: u32 = 500` and `PERSISTENT_TTL_MAX: u32 = 2000`.
- Apply the same bump in `update_profile` after loading the profile.

**Acceptance Criteria:**
- [ ] `get_player` extends TTL on the `Player(id)` key.
- [ ] `get_scout` extends TTL on the `Scout(id)` key.
- [ ] `get_player_by_wallet` extends TTL on both the index key and the profile key.
- [ ] Unit tests verify no panic after simulated ledger advancement past default TTL.

---

## Issue 12 — `pause_contract` / `unpause_contract` emit no events

**Labels:** `enhancement`, `registration`, `events`

**Description:**
When the admin pauses or unpauses the contract, there is no on-chain event. The backend indexer and monitoring tools cannot detect the circuit-breaker state change.

**Task:**
Add `contract_paused` and `contract_unpaused` event emitters in `events.rs` and call them from `pause_contract` and `unpause_contract`.

**Additional Requirements:**
- Publish the admin address as event data so the indexer knows who triggered it.
- Follow the existing event pattern.

**Acceptance Criteria:**
- [ ] `pause_contract` emits `contract_paused` with admin address.
- [ ] `unpause_contract` emits `contract_unpaused` with admin address.
- [ ] Unit tests assert both events appear in `env.events()`.

---

## Issue 13 — `require_not_paused` is not called in `update_profile`

**Labels:** `bug`, `registration`, `security`

**Description:**
`register_player` and `register_scout` both call `require_not_paused`, but `update_profile` does not. A player can update their IPFS hashes even when the contract is paused.

**Task:**
Add `Self::require_not_paused(&env)?;` as the first line of `update_profile`.

**Additional Requirements:**
- Verify all other state-changing functions also call `require_not_paused` and add it where missing.

**Acceptance Criteria:**
- [ ] `update_profile` returns `ContractPaused` when the contract is paused.
- [ ] Unit test: pause → attempt update → expect `ContractPaused` error.

---

## Issue 14 — Admin transfer function is missing from `registration` contract

**Labels:** `enhancement`, `registration`, `admin`

**Description:**
There is no way to transfer admin ownership. If the admin key is compromised or lost, the contract is permanently locked to that key with no recovery path.

**Task:**
Implement `transfer_admin(env: Env, new_admin: Address) -> Result<(), ScoutChainError>` that requires the current admin's auth and updates `DataKey::Admin`.

**Additional Requirements:**
- Emit a `admin_transferred` event with both old and new admin addresses.
- The new admin must not be the zero address (reuse the check from Issue 1).
- Require auth from the current admin, not the new one.

**Acceptance Criteria:**
- [ ] Only the current admin can call `transfer_admin`.
- [ ] After transfer, the old admin cannot call admin-only functions.
- [ ] `admin_transferred` event is emitted.
- [ ] Unit tests cover success, unauthorized caller, and zero-address cases.

---

## Issue 15 — `get_player` is callable on a paused contract; add a read-only exemption note

**Labels:** `documentation`, `registration`

**Description:**
Query functions (`get_player`, `get_scout`, `health`) intentionally skip the paused check because reads should always be available. This design decision is undocumented, leading to contributor confusion.

**Task:**
Add a `// NOTE: read-only — intentionally skips paused check` comment above each query function in `lib.rs` and document the policy in `CONTRACT_REFERENCE.md`.

**Additional Requirements:**
- No code changes beyond comments and docs.

**Acceptance Criteria:**
- [ ] Each query function has the comment.
- [ ] `CONTRACT_REFERENCE.md` has a "Paused Behaviour" section explaining reads remain available.

---

## Issue 16 — `PlayerVitals` has no `height_cm` or `weight_kg` fields; add optional physical stats

**Labels:** `enhancement`, `registration`, `types`

**Description:**
Scouts filter by physical attributes. The current `PlayerVitals` struct only stores `age`, `position`, `region`, and `nationality`. Height and weight are standard scouting data points.

**Task:**
Add `height_cm: Option<u32>` and `weight_kg: Option<u32>` to `PlayerVitals` in `types.rs`.

**Additional Requirements:**
- Both fields are optional (`Option`) so existing registrations remain valid.
- Add validation: if provided, `height_cm` must be in `[100, 250]` and `weight_kg` in `[30, 200]`.
- Update `dummy_vitals` in tests to include `None` for both new fields.

**Acceptance Criteria:**
- [ ] `PlayerVitals` compiles with the two new optional fields.
- [ ] Out-of-range values return `InvalidInput`.
- [ ] All existing tests still pass.
- [ ] `CONTRACT_REFERENCE.md` documents the new fields.

---

## Issue 17 — `ScoutProfile` has no `name` or `organization` field

**Labels:** `enhancement`, `registration`, `types`

**Description:**
`ScoutProfile` only stores `scout_id`, `wallet`, `region`, and `registered_at`. Players and admins have no way to identify which club or agency a scout represents.

**Task:**
Add `organization: String` (max 128 bytes) to `ScoutProfile` and update `register_scout` to accept it as a parameter.

**Additional Requirements:**
- Validate length ≤ 128 bytes; return `InvalidInput` otherwise.
- Emit the organization name in the `scout_registered` event data.
- Update `CONTRACT_REFERENCE.md`.

**Acceptance Criteria:**
- [ ] `register_scout` accepts and stores `organization`.
- [ ] Length validation enforced.
- [ ] `scout_registered` event includes organization.
- [ ] Unit tests updated.

---

## Issue 18 — Write a test for `get_player_by_wallet` returning `PlayerNotFound`

**Labels:** `test`, `registration`

**Description:**
The existing test suite covers the happy path for `get_player_by_wallet` implicitly, but there is no explicit test asserting `PlayerNotFound` is returned for an unregistered wallet.

**Task:**
Add a `#[test] #[should_panic] fn test_get_player_by_wallet_not_found()` test in `lib.rs`.

**Additional Requirements:**
- Use `Address::generate(&env)` for a wallet that was never registered.
- The test must use `#[should_panic]` or explicitly match the error variant.

**Acceptance Criteria:**
- [ ] Test exists and passes.
- [ ] Test name follows the `test_<function>_<scenario>` convention.

---

## Issue 19 — Write a test for `get_scout` returning `ScoutNotFound`

**Labels:** `test`, `registration`

**Description:**
No test currently asserts that `get_scout` returns `ScoutNotFound` for an unknown `scout_id`.

**Task:**
Add `fn test_get_scout_not_found()` that calls `get_scout(999u64)` on a freshly initialized contract and expects a panic/error.

**Acceptance Criteria:**
- [ ] Test exists and passes.
- [ ] Uses `#[should_panic]` annotation.

---

## Issue 20 — Write a test for `pause_contract` blocking `register_player`

**Labels:** `test`, `registration`

**Description:**
The circuit-breaker logic is implemented but not tested end-to-end for `register_player`.

**Task:**
Add `fn test_register_player_blocked_when_paused()`: initialize → pause → attempt `register_player` → expect `ContractPaused`.

**Acceptance Criteria:**
- [ ] Test exists and passes with `#[should_panic]`.
- [ ] Unpause → register succeeds in the same test or a companion test.

---

## Issue 21 — Write a test for `pause_contract` blocking `register_scout`

**Labels:** `test`, `registration`

**Description:**
Companion to Issue 20 — `register_scout` also needs a paused-state test.

**Task:**
Add `fn test_register_scout_blocked_when_paused()`.

**Acceptance Criteria:**
- [ ] Test exists and passes.

---

## Issue 22 — Write a test for `update_profile` blocked when paused (after Issue 13 fix)

**Labels:** `test`, `registration`

**Description:**
Once Issue 13 is resolved, a test must verify the fix.

**Task:**
Add `fn test_update_profile_blocked_when_paused()`: register player → pause → attempt update → expect `ContractPaused`.

**Acceptance Criteria:**
- [ ] Test exists and passes.
- [ ] Depends on Issue 13 being merged first (note in PR description).

---

## Issue 23 — `next_player_id` uses `expect("overflow")` instead of returning an error

**Labels:** `bug`, `registration`, `error-handling`

**Description:**
`next_player_id` calls `.checked_add(1).expect("overflow")` which panics with a non-descriptive message. Soroban contracts should return typed errors rather than panic.

**Task:**
Change `next_player_id` and `next_scout_id` to return `Result<u64, ScoutChainError>` and propagate `ScoutChainError::Overflow` (error code 11) instead of panicking.

**Additional Requirements:**
- Update callers (`register_player`, `register_scout`) to use `?` propagation.

**Acceptance Criteria:**
- [ ] No `.expect("overflow")` in `lib.rs`.
- [ ] `ScoutChainError::Overflow` is returned on counter overflow.
- [ ] Unit test simulates overflow by manually setting the counter to `u64::MAX` and calling `register_player`.

---

## Issue 24 — Add `deregister_player` admin function for GDPR/right-to-erasure compliance

**Labels:** `enhancement`, `registration`, `admin`

**Description:**
Players in some jurisdictions have a legal right to erasure. There is currently no way to remove a player profile from on-chain storage.

**Task:**
Implement `deregister_player(env: Env, player_id: u64) -> Result<(), ScoutChainError>` (admin only) that removes `DataKey::Player(player_id)` and `DataKey::PlayerByWallet(wallet)` from persistent storage.

**Additional Requirements:**
- Emit a `player_deregistered` event with `player_id`.
- Return `PlayerNotFound` if the ID does not exist.
- Document the irreversibility in a code comment.

**Acceptance Criteria:**
- [ ] After `deregister_player`, `get_player` returns `PlayerNotFound`.
- [ ] `PlayerByWallet` index is also removed.
- [ ] Only admin can call the function.
- [ ] Unit tests cover success and not-found cases.

---

## Issue 25 — Add `get_player_count` and `get_scout_count` query functions

**Labels:** `enhancement`, `registration`, `query`

**Description:**
The frontend dashboard needs to display total registered players and scouts. Currently there is no public function to read the counters.

**Task:**
Add `get_player_count(env: Env) -> u64` and `get_scout_count(env: Env) -> u64` that return the current counter values from instance storage.

**Additional Requirements:**
- Return `0` if the contract is not yet initialized (graceful default).
- Add both functions to `CONTRACT_REFERENCE.md`.

**Acceptance Criteria:**
- [ ] Returns correct count after registering N players/scouts.
- [ ] Returns `0` before `initialize` is called.
- [ ] Unit tests register 3 players and assert count is 3.

---

## Issue 26 — `PlayerProfile.updated_at` is not updated when `level` changes externally

**Labels:** `bug`, `registration`, `storage`

**Description:**
`updated_at` is only set in `update_profile`. When the progress contract advances a player's level, the registration contract's `updated_at` field stays stale. Scouts see an outdated timestamp.

**Task:**
Add a `set_player_level(env: Env, player_id: u64, level: ProgressLevel) -> Result<(), ScoutChainError>` function that can be called by the progress contract to sync the level and refresh `updated_at`.

**Additional Requirements:**
- Restrict callers: only the registered progress contract address (stored in a new `DataKey::ProgressContract`) may call this.
- Add `DataKey::ProgressContract` and a `set_progress_contract(env, addr)` admin function.
- Emit a `player_level_synced` event.

**Acceptance Criteria:**
- [ ] `set_player_level` updates both `level` and `updated_at` in the stored profile.
- [ ] Unauthorized callers are rejected.
- [ ] Unit test mocks the progress contract address and verifies the update.

---

## Issue 27 — `register_player` does not check `require_initialized` before `require_not_paused`

**Labels:** `bug`, `registration`, `error-handling`

**Description:**
`register_player` calls `require_not_paused` before `require_initialized`. On an uninitialized contract, `Paused` defaults to `false`, so the paused check passes and the function proceeds to fail later with a less informative error.

**Task:**
Swap the order: call `require_initialized` first, then `require_not_paused` in `register_player` and `register_scout`.

**Acceptance Criteria:**
- [ ] Calling `register_player` on an uninitialized contract returns `NotInitialized`.
- [ ] Unit test verifies the error code is `NotInitialized` (not `ContractPaused`).

---

## Issue 28 — Add `#[cfg(test)]` guard to the test `setup` helper to avoid dead-code warnings

**Labels:** `chore`, `registration`, `code-quality`

**Description:**
The `setup()` and `dummy_vitals()` helpers in the test module are only used in tests but may trigger dead-code lints in some Rust toolchain versions.

**Task:**
Ensure all test helper functions are inside `#[cfg(test)]` blocks and annotate with `#[allow(dead_code)]` only if strictly necessary.

**Additional Requirements:**
- Run `cargo clippy --workspace -- -D warnings` and confirm zero warnings after the change.

**Acceptance Criteria:**
- [ ] `cargo clippy --workspace -- -D warnings` exits with code 0.
- [ ] No `#[allow(dead_code)]` annotations added unless unavoidable.

---

## Issue 29 — Document all `DataKey` variants with inline comments in `types.rs`

**Labels:** `documentation`, `registration`

**Description:**
`DataKey` variants like `PlayerByWallet(Address)` and `ScoutByWallet(Address)` have no comments explaining their purpose or the data they index.

**Task:**
Add a `///` doc comment to every `DataKey` variant in `contracts/registration/src/types.rs`.

**Additional Requirements:**
- Comments must explain the key's purpose and the type of value stored.
- Follow the style already used for `ipfs_hashes` in `PlayerProfile`.

**Acceptance Criteria:**
- [ ] Every `DataKey` variant has a doc comment.
- [ ] `cargo doc --no-deps` generates documentation without warnings.

---

## Issue 30 — Add `filter_players` query to `registration` contract

**Labels:** `enhancement`, `registration`, `query`

**Description:**
The README lists `filter_players(region, position, min_level)` as a query function, but it is not implemented. Scouts cannot filter the talent pool on-chain.

**Task:**
Implement `filter_players(env: Env, region: Option<String>, position: Option<String>, min_level: ProgressLevel) -> Vec<PlayerProfile>` that iterates stored profiles and returns matches.

**Additional Requirements:**
- Use a stored `Vec<u64>` player ID list (`DataKey::PlayerIndex`) maintained by `register_player` and `deregister_player`.
- Return at most 50 results to bound gas usage; document this limit.
- All filter parameters are optional except `min_level`.

**Acceptance Criteria:**
- [ ] Returns only players matching all provided filters.
- [ ] Returns at most 50 results.
- [ ] Unit test registers 3 players with different regions and asserts correct filtering.

---

## Issue 31 — `ScoutProfile` is missing a `verified` boolean flag

**Labels:** `enhancement`, `registration`, `types`

**Description:**
The platform needs to distinguish between self-registered scouts and admin-verified scouts (e.g., scouts from licensed agencies). There is currently no such flag.

**Task:**
Add `verified: bool` to `ScoutProfile` (default `false` on registration). Add `verify_scout(env: Env, scout_id: u64) -> Result<(), ScoutChainError>` (admin only) that sets it to `true`.

**Additional Requirements:**
- Emit a `scout_verified` event with `scout_id`.
- `CONTRACT_REFERENCE.md` updated.

**Acceptance Criteria:**
- [ ] Newly registered scouts have `verified = false`.
- [ ] Admin can set `verified = true`.
- [ ] Non-admin call returns `Unauthorized`.
- [ ] Unit tests cover all three cases.

---

## Issue 32 — Write integration test: full player registration → profile update flow

**Labels:** `test`, `registration`

**Description:**
No single test exercises the complete player lifecycle within the registration contract: initialize → register → update profile → read back updated hashes.

**Task:**
Add `fn test_full_player_registration_and_update_flow()` that covers all four steps in sequence and asserts the final stored state.

**Acceptance Criteria:**
- [ ] Test passes end-to-end.
- [ ] Asserts `updated_at` is greater than `registered_at` after the update.
- [ ] Asserts `ipfs_hashes` matches the updated list.

---

## Issue 33 — Write integration test: register player and scout with same wallet should fail

**Labels:** `test`, `registration`

**Description:**
A wallet can currently register as both a player and a scout. The `AlreadyRegistered` check is per-role (separate `PlayerByWallet` and `ScoutByWallet` keys), so this is allowed. Confirm this is intentional and document it, or add a cross-role uniqueness check.

**Task:**
1. Write a test that registers the same wallet as both player and scout and documents the expected outcome.
2. If cross-role uniqueness is desired, add a `DataKey::WalletUsed(Address)` check and return `AlreadyRegistered`.
3. Update `CONTRACT_REFERENCE.md` with the chosen policy.

**Acceptance Criteria:**
- [ ] The policy (allow or deny dual-role) is explicitly documented.
- [ ] A test enforces the documented policy.

---

## Issue 34 — `health()` should also return the paused state

**Labels:** `enhancement`, `registration`, `query`

**Description:**
`health()` returns a plain `bool` (initialized or not). Operators monitoring the contract cannot tell if it is paused without a separate call.

**Task:**
Change `health` to return a struct `ContractHealth { initialized: bool, paused: bool }` defined in `types.rs`.

**Additional Requirements:**
- Update `CONTRACT_REFERENCE.md`.
- Update the existing `test_initialize_and_health` test.

**Acceptance Criteria:**
- [ ] `health()` returns `ContractHealth { initialized: true, paused: false }` after normal init.
- [ ] `health()` returns `ContractHealth { initialized: true, paused: true }` after pause.
- [ ] Unit tests cover both states.

---

## Issue 35 — Add `cargo test` output to CI for the `registration` contract specifically

**Labels:** `ci`, `registration`, `devops`

**Description:**
The CI workflow runs `cargo test --workspace` which tests all contracts together. A failure in any contract blocks the entire pipeline. Per-contract test steps would give faster, more targeted feedback.

**Task:**
Update `.github/workflows/contract-ci.yml` to add a dedicated step that runs `cargo test -p scoutchain-registration -- --nocapture` after the workspace-level test step.

**Additional Requirements:**
- The per-contract step should be named `Test registration contract`.
- Keep the workspace-level step as the primary gate.

**Acceptance Criteria:**
- [ ] CI workflow has a `Test registration contract` step.
- [ ] The step runs only the registration contract tests.
- [ ] CI passes on the current `main` branch.
