# History Fetch Analysis

## How `wx history` works

### CLI layer (`src/cli/history.rs`)

1. `cmd_history()` parses args: chat name, `--limit` (default 50), `--offset` (default 0), `--since`, `--until`, `--type`
2. Converts time strings → Unix timestamps via `parse_time` / `parse_time_end`
3. Sends `Request::History` over Unix socket to daemon

### Daemon query layer (`src/daemon/query.rs:178`, `q_history()`)

1. **Name resolution**: `resolve_username()` maps chat display name → WeChat `username` using contact DB
2. **Table discovery**: `find_msg_tables()` (line 412):
   - Computes `Msg_<md5(username)>` as the table name
   - Iterates all `msg_db_keys` (= relative paths of `message/message_*.db` that have encryption keys)
   - For each DB, checks `sqlite_master` for table existence, gets `MAX(create_time)` for ordering
   - Returns matching (db_path, table_name) pairs sorted by max timestamp descending
3. **Per-shard query**: `query_messages()` (line 458):
   - Opens a fresh `rusqlite::Connection` to the decrypted DB
   - Builds SQL with optional WHERE filters (since, until, msg_type):
     ```sql
     SELECT local_id, local_type, create_time, real_sender_id,
            message_content, WCDB_CT_message_content
     FROM [Msg_<md5>] [WHERE ...] ORDER BY create_time DESC LIMIT ? OFFSET ?
     ```
   - Limit per DB shard = `offset + limit` (to allow correct global pagination)
   - For each row: decompress zstd content, resolve sender name, format content
4. **Merge & paginate**:
   - Collect results from all shards into `Vec<Value>`
   - Sort by timestamp descending
   - Apply global `skip(offset).take(limit)`
   - Re-sort ascending for output

### Cache layer (`src/daemon/cache.rs`, `DbCache::get()`)

- Called by `find_msg_tables()` for each `msg_db_key` to get the decrypted DB path
- Checks current mtime of BOTH `.db` AND `.db-wal` against cached values
- If changed → re-decrypts: `crypto::full_decrypt()` + `wal::apply_wal()`
- Decrypted output goes to `~/.wx-cli/cache/<md5(rel_key)>.db`
- Mtimes persisted to `~/.wx-cli/cache/_mtimes.json` for reuse across restarts

## Root Cause #1: Recently imported data from phone not showing

**Most likely: daemon serves stale decrypted DB cache.**

Sequence:
1. Daemon starts, decrypts `message/message_0.db` (and applies its `.db-wal`)
2. WeChat imports chat history from phone → writes new encrypted pages to `.db-wal`
3. Daemon's `get()` checks mtimes — correctly detects `.db-wal` change in most cases
4. **BUT**: if the import finished and the WAL was checkpointed (merged into `.db`) rapidly, only `.db` mtime changes. The code handles this, but timing-sensitive edge cases exist.

**Also possible: new DB shard missing from keys.**

If the phone had messages in a shard that doesn't exist on desktop (e.g. `message/message_2.db`), WeChat would create it during import. But `all_keys.json` lacks an entry for this new path → daemon never sees it → `msg_db_keys` doesn't include it → messages invisible.

**Fix**: re-run `wx init` after importing, then restart the daemon (`wx daemon stop`).

## Root Cause #2: Only 4495 messages visible after fresh init

**Actual cause: `all_keys.json` is missing entries for newer message DB shards.**

WeChat creates multiple `message/message_N.db` shards over time. The key scanner (`wx init`) only finds keys for databases that are currently OPEN in WeChat's process memory. If a shard existed but wasn't open during the scan, its entry is missing from `all_keys.json`, and `msg_db_keys` doesn't include it → `find_msg_tables()` never looks there.

**Fix applied** at `src/daemon/mod.rs:48-72`: after loading keys from `all_keys.json`, the daemon now scans the filesystem for `message/message_*.db` files. Any that exist on disk but lack a key entry are added to `all_keys` using the same encryption key as other message shards (they all share the same derived AES-256 key).

```rust
// Scan for message DB shards on disk not present in all_keys
// All message/message_N.db share the same encryption key
let message_dir = cfg.db_dir.join("message");
if message_dir.is_dir() {
    if let Ok(entries) = std::fs::read_dir(&message_dir) {
        for entry in entries.flatten() {
            // ... match message_*.db pattern ...
            if !all_keys.contains_key(&rel_key) {
                if let Some(key) = all_keys.values().next().cloned() {
                    all_keys.insert(rel_key, key);
                }
            }
        }
    }
}
```

Also possible but less likely:
- **Default limit is 50**: use `wx history <chat> -n 10000` to see more
- **Messages span multiple DB shards**: `find_msg_tables()` now sees all of them since `msg_db_keys` includes every shard found on disk


## Recommendations

1. After phone import → re-run `wx init` (or just restart the daemon — it auto-discovers missing shards)
2. Use `wx history <chat> -n 5000 --offset 0` to see a large window
3. Use `--since` / `--until` to narrow by date range
4. Use `--offset` to paginate through older pages
