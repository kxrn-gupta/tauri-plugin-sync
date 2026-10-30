# Conflict Resolution Specification & Edge Cases

`tauri-plugin-sync` employs a deterministic **Last-Write-Wins (LWW)** model augmented with **soft tombstones**, **lexicographical tiebreakers**, and **session-level trigger suppression**.

This document outlines the formal behavior for all conflict and synchronization edge cases.

---

## 1. Core Resolution Rules

### Rule 1: Timestamp Dominance (Last-Write-Wins)
When a remote mutation arrives for an existing row:
- If `remote.updated_at > local.updated_at`: The remote mutation is applied locally.
- If `remote.updated_at < local.updated_at`: The remote mutation is rejected as stale.

### Rule 2: Microsecond Tiebreaker (Deterministic Local Precedence)
If two independent offline devices write to the same record at the exact same microsecond (`remote.updated_at == local.updated_at`):
- Local pending modifications take deterministic precedence over remote writes (`local_timestamp >= remote.timestamp`), preventing oscillation.
- Both nodes deterministically converge without distributed deadlock.

---

## 2. Edge Case Specifications

### Edge Case A: Deletion Tombstones vs Concurrent Updates
*Scenario: Device A deletes a record while offline. Device B updates the same record while offline.*
- **Mechanism**: Deletions are recorded in `_sync_tombstones` with `(table_name, record_id, deleted_at)`.
- **Precedence Rule**:
  ```sql
  IF EXISTS (
      SELECT 1 FROM _sync_tombstones
      WHERE table_name = ? AND record_id = ? AND deleted_at >= remote.updated_at
  ) THEN
      DISCARD remote_mutation; -- Tombstone wins; row remains deleted
  ```
- **Guaranteed Outcome**: Deleted records are **never resurrected** by stale remote update packets arriving after the deletion.

### Edge Case B: Legitimate Tombstone Pruning & Row Recreation
*Scenario: Device A deletes a record at $t=1000$. Later, Device B legitimately recreates the item with an explicit newer insert at $t=2000$.*
- When the incoming mutation has `remote.updated_at > tombstone.deleted_at`, the engine clears the tombstone and recreates the row cleanly.

### Edge Case C: Out-of-Order Packet Delivery
*Scenario: Device sends Update #1 ($t=100$) and Update #2 ($t=200$). Due to network latency, Update #2 arrives first, followed by Update #1.*
- When Update #2 arrives ($t=200$), it is applied, and the local row timestamp becomes $t=200$.
- When Update #1 arrives ($t=100$), the engine checks:
  $$100 \le 200 \implies \text{REJECT}$$
- The stale update is dropped without overwriting the newer state.

### Edge Case D: Infinite Echo Loop Prevention (Recursion Suppression)
*Scenario: Device pulls an incoming change from the server and executes an `INSERT` on the local SQLite table. Does this trigger local CDC and re-push the change back to the server?*
- **Mechanism**: A dedicated session table `_sync_suppress (suppressed INTEGER)` is checked in every trigger's `WHEN` clause:
  ```sql
  CREATE TRIGGER trg_todos_insert
  AFTER INSERT ON todos
  WHEN (SELECT suppressed FROM _sync_suppress) = 0
  BEGIN
      INSERT INTO _sync_changes ...
  END;
  ```
- During a `pull` pass, the engine sets `suppressed = 1`, applies incoming remote mutations atomically, and resets `suppressed = 0`.
- **Guaranteed Outcome**: **0 circular loops**. Inbound changes are never re-journaled as outbound changes.

### Edge Case E: Foreign Key Constraint Deferral
*Scenario: A batch contains both a parent record (`categories`) and a child record (`todos`), but network serialization ordered the child before the parent.*
- **Mechanism**: During the pull transaction, the engine executes:
  ```sql
  PRAGMA defer_foreign_keys = ON;
  ```
- Foreign key validity is enforced at the end of the transaction rather than after individual statements.
- **Guaranteed Outcome**: Batch operations never abort due to transient intra-batch foreign key ordering discrepancies.

### Edge Case F: Forward Schema Evolution (Unknown Server Columns)
*Scenario: Server or a newer client version adds a new column (`priority`) to `todos`. An older desktop client without this column pulls the changeset.*
- **Mechanism**: Before applying an incoming JSON payload, the engine introspects local table columns via `PRAGMA table_info()` and filters out any key not present in the local table schema.
- **Guaranteed Outcome**: Older clients continue to function reliably without `SQLITE_ERROR: no such column` exceptions.

---

## 3. Verified Automated Test Matrix (15 Tests)

Every edge case above is covered by an automated test in `crates/tauri-plugin-sync/tests/sync_tests.rs`:

| Test Name | Edge Case Covered | Status |
| :--- | :--- | :---: |
| `test_cdc_triggers_record_insert_update_delete` | Basic CDC capture | Passed (100%) |
| `test_trigger_suppression_prevents_recursive_cdc` | Infinite loop immunity | Passed (100%) |
| `test_apply_remote_record_lww` | Last-Write-Wins precedence | Passed (100%) |
| `test_exact_timestamp_collision_tiebreaker_local_precedence` | Microsecond collision tiebreaker | Passed (100%) |
| `test_deletion_tombstone_wins_over_stale_update` | Tombstone resurrection prevention | Passed (100%) |
| `test_tombstone_resurrection_when_remote_strictly_newer` | Legitimate newer recreation | Passed (100%) |
| `test_foreign_key_deferral_reverse_batch_order` | Deferred foreign keys | Passed (100%) |
| `test_schema_evolution_drops_unknown_server_columns` | Forward schema evolution | Passed (100%) |
| `test_sql_injection_and_reserved_keywords_sanitization` | Identifier quoting & safety | Passed (100%) |
| `test_concurrent_writes_1000_operations_stress` | Multi-threaded race conditions | Passed (100%) |
| `test_full_push_pull_with_mock_server` | End-to-end sync loop | Passed (100%) |
| `test_pause_resume_and_frontend_config` | Dynamic lifecycle controls | Passed (100%) |
| `test_unauthorized_401_triggers_error_and_event` | Token refresh and auth lifecycle | Passed (100%) |
| `test_mark_and_prune_changes` | Watermark pruning & memory management | Passed (100%) |
| `test_builder_pattern_construction` | Safe Rust API construction | Passed (100%) |
