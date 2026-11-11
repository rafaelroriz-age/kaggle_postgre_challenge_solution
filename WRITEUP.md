# Durable Expansion Checkpoints and Transaction-Scoped Tuning

**Repository:** https://github.com/rafaelroriz-age/kaggle_postgre_challenge_solution
**Submission commit:** `<COMMIT_HASH>`

## 1. Baseline architecture and bottlenecks

The Expand operation (`PostgresService.execute_cross_table_expansion`) already pushes all heavy work to the server. It sends one SQL script through SSH to `psql`, and that script runs inside a single `REPEATABLE READ` transaction:

1. lock the destinations;
2. scan `raw_data` and build the `raw_schema`/`raw_hash` dependency manifest (`GROUP BY`);
3. run six `INSERT ... SELECT` recipes, one per destination, each inside a `DO` block that updates the row counter;
4. `COMMIT`. Version registration and the dump then run as separate round-trips (`register_cross_table_expansion_versions`).

Main bottlenecks:

- **Repeated full scans of `raw_data` with JSONB detoasting.** The manifest and each of the six recipes read and parse the `raw` column again. Most recipes use `SELECT DISTINCT` / `GROUP BY` over expressions extracted from JSONB.
- **Hash aggregates spilling to disk.** With the default `work_mem` (4 MB), the `DISTINCT`/`GROUP BY` steps of the recipes and the manifest spill to temporary files at multi-million row scale. This costs time and also temporary disk, which is one of the judged resources.
- **No durable checkpoint between the data COMMIT and version registration.** If the SSH session, the client or the network failed after `COMMIT` but before `register_cross_table_expansion_versions` finished, the destinations held committed rows with no version record. The UI could only warn "Do not repeat the expansion before reconciling version history". The current code had no way to finish that work.

## 2. Changes

### 2.1 Transaction-scoped server tuning (performance and temp disk)

`EXPANSION_SESSION_TUNING_SQL` is injected right after `BEGIN` of the expansion transaction:

```sql
SET LOCAL work_mem = '256MB';
SET LOCAL maintenance_work_mem = '512MB';
SET LOCAL max_parallel_workers_per_gather = 4;
```

- `SET LOCAL` means the settings exist only for this transaction and revert automatically on COMMIT or ROLLBACK. No server configuration changes.
- Larger `work_mem` keeps the hash aggregates of the `DISTINCT`/`GROUP BY` recipes and of the manifest in memory. They run faster and write far less temporary data to disk.
- Parallel workers apply to the read-only manifest scan (`CREATE TEMP TABLE ... AS SELECT ... GROUP BY`). PostgreSQL never runs the `INSERT ... SELECT` statements in parallel, so their plans are unaffected.
- Trade-off: peak backend RAM is bounded by `work_mem` × the number of concurrent hash nodes. The values are moderate and are constants that are easy to change.

### 2.2 Durable expansion checkpoint and automatic resume (failure recovery)

New table in each data database, created idempotently with the existing row-counter metadata (`ensure_data_database_metadata`):

```sql
public.pgdm_expansion_journal(
  expansion_token text PRIMARY KEY,
  status text CHECK (status IN ('committed','registered')),
  request_payload jsonb,       -- source, raw_schemas, source_versions, prepared destinations/recipes
  dependency_manifest jsonb,   -- the raw_hash manifest computed inside the transaction
  committed_at timestamptz, registered_at timestamptz)
```

- The manifest's final `SELECT` now first materialises into an `ON COMMIT DROP` temp table, `pgdm_cross_table_manifest_json`. The same JSON still goes back to the client, and the journal row copies it.
- The journal row is inserted **inside the same transaction, just before `COMMIT`**. This step has its own timing: *"Write durable expansion checkpoint"*. The data and the checkpoint therefore commit atomically. If committed rows exist, a checkpoint exists too.
- After `register_cross_table_expansion_versions` succeeds, the journal row is set to `registered` (`mark_expansion_journal_registered`).
- **Automatic resume.** At the start of every expansion, `list_pending_expansion_journals` looks for `committed` rows. For each one, `build_expansion_result_from_journal` rebuilds the exact `execute_cross_table_expansion` result. Version registration and the dump then run from it, and the row is closed. Committed data is **never re-expanded or duplicated**, and no expansion is left without its version record.

Failure matrix:

| Interruption point | Result |
|---|---|
| Before or during the data transaction (SSH drop, psql killed, cancel) | PostgreSQL rolls back. No rows, no checkpoint. Temp tables (`ON COMMIT DROP`, session temp) vanish with the session. Safe to rerun. |
| After COMMIT, before or during version registration | A `committed` checkpoint survives. The next expansion finishes registration automatically. |
| After registration | The checkpoint is `registered`. Nothing pending. |

### 2.3 Guarantees preserved

Source validation, destination validation, SQL function safety checks, managed-dependency checks, destination locks, the `REPEATABLE READ` snapshot, row-counter updates, version registration, raw dumps and the timing report are all unchanged. Nothing was removed. The existing unit tests pass (`pytest tests`: 6 passed).

## 3. Resource usage

- **RAM:** bounded by the transaction-local `work_mem`. The client-side Python footprint is unchanged.
- **Temporary disk:** lower than the baseline, because hash aggregates no longer spill. The journal stores one small row per expansion (recipes plus the manifest, which has one entry per `raw_schema`/`raw_hash`), not row data.
- **Storage overhead:** one tiny table per data database.

## 4. Measured results

Local benchmark numbers are **not** reported. No Linux/PostgreSQL environment was available while this was prepared, so the changes were checked only with the repository's unit tests and static compilation. The official benchmark on the organisers' infrastructure is authoritative.

## 5. Limitations and next steps

- Resume works at the level of the whole expansion, which is a single atomic transaction. It does not checkpoint inside the data transaction. Splitting the work into per-destination or per-`raw_hash` batch transactions, each with its own checkpoint, is the natural next step for 300M-row datasets. It requires care to keep the cross-table snapshot consistent.
- If version registration itself fails halfway, after some versions are already written, the resumed registration may need reconciliation. Making `register_cross_table_expansion_versions` idempotent per `expansion_token` is the next step.
- Further speed-ups that were identified but not implemented: a single pass that extracts the JSONB once into an `UNLOGGED` staging table shared by all six recipes (a trade of temporary disk for CPU), and an `ANALYZE` of the destinations after large loads.
