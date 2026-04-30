# ckpool-digibyte

Fork of [ctubio/ckpool](https://github.com/ctubio/ckpool) with patches for DigiByte (DGB) SHA-256d mining pool support.

## Changes from upstream

### 1. OpenSSL 1.1+ compatibility (`configure.ac`, `src/ckdb_crypt.c`)
- Replaced deprecated `BN_init` with `BN_new` in OpenSSL library check
- Changed `HMAC_CTX` from stack allocation to `HMAC_CTX_new()`/`HMAC_CTX_free()` for OpenSSL 1.1+

### 2. ckdb global symbol fix (`src/ckdb.c`)
- Added `ckpool_t *global_ckp;` definition to `ckdb.c` since ckdb is a separate binary

### 3. ckpool.h multiple definition fix (`src/ckpool.h`)
- Changed `ckpool_t *global_ckp;` to `extern ckpool_t *global_ckp;` to fix linker error with GCC 11+

### 4. DigiByte GBT compatibility (`src/bitcoin.c`)
- Added `taproot` and `csv` to understood rules (DGB returns `!segwit`, `taproot`, `csv` in GBT)
- Fixed `flags` null handling: DGB's `coinbaseaux.flags` is empty string, not null

### 5. mining.configure support (`src/stratifier.c`)
- Added handling for `mining.configure` before `mining.subscribe` for version-rolling ASICs (e.g. Avalon Nano 3S)
- Returns version-rolling enabled response with mask `1fffe000`

### 6. mining.submit fix (`src/stratifier.c`)
- Added `mining.submit` handler in `parse_method()` which was missing in ctubio fork
- Without this fix, shares from miners are silently dropped

### 7. DGB multi-algo block filtering (`src/bitcoin.c`, `src/generator.c`, `src/stratifier.c`, `src/stratifier.h`)
- DGB GBT responses include `pow_algo_id` field; only SHA-256d blocks (`pow_algo_id == 0`) should be mined
- `gen_gbtbase()` now returns `false` with `*skipped = true` when `pow_algo_id != 0`, so non-SHA-256d blocks are silently skipped without marking the connection as failed
- Startup connectivity test in `server_alive()` switched from `gen_gbtbase()` to `get_blockcount()` so that the server is correctly marked alive even when the current block is non-SHA-256d
- `gen_loop()` and `generator_getbase()` distinguish "skipped" from "failed": on skip the server stays alive and no reconnect is triggered
- `block_update()` in stratifier treats a skipped workbase as success (no retry loop, no warning)

### 8. Allow mining.subscribe without a current workbase (`src/stratifier.c`)
- `parse_subscribe()` previously returned "Pool Initialising" and disconnected clients when `current_workbase` was NULL, which happens on every non-SHA-256d block on DGB
- Changed the guard so only a missing `sdata` is fatal; a missing `current_workbase` is allowed
- `new_enonce1()` now assigns enonce1 using `ckp->nonce1length` / `ckp->nonce2length` (pool config defaults) when no workbase is present, instead of returning false
- Session reconnect path's `__fill_enonce1data()` call is guarded against a NULL workbase
- Result: ASICs can subscribe and authorize during non-SHA-256d blocks; `stratum_broadcast_update()` automatically delivers a job when the next SHA-256d block arrives

### 9. Fix blockupdate spam for non-SHA256d blocks (`src/bitcoin.c`, `src/stratifier.c`)
- When a non-SHA256d block arrived on DGB, `lastswaphash` (SHA256d-only, managed by `add_base()`) was never updated, causing the `blockupdate` thread to re-fire `GEN_PRIORITY` on every poll loop iteration for the same block
- **Wrong fix attempted**: updating `lastswaphash` with the non-SHA256d block hash polluted the SHA256d change-detection variable, risking missed updates when the next SHA256d block arrived
- **Correct fix**: added `lastseenhash[68]` to `sdata_t` to track the last seen block hash for any algo; `blockupdate()` now compares/updates `lastseenhash` (not `lastswaphash`) before calling `update_base()`, so only one `GEN_PRIORITY` fires per block regardless of algo; `lastswaphash` remains SHA256d-only (managed exclusively by `add_base()`)
- `gen_gbtbase()` sets `gbt->prevhash` before the `pow_algo_id` skip check (available for future use by callers)
- `block_update()` leaves `sdata->update_time` unchanged when skipping a non-SHA256d block, so GEN_NORMAL polling fires again after the normal interval as a fallback to pick up the next SHA256d block
- SHA256d block arrival path: `lastseenhash` changes → one `GEN_PRIORITY` → `gen_gbtbase()` succeeds → `add_base()` updates `lastswaphash`/`current_workbase` → `stratum_broadcast_update()` sends jobs ✓

### 10. Fix stratum_loop GEN_NORMAL polling during non-SHA256d blocks (`src/stratifier.c`)
- The do-while loop in `stratum_loop()` used `sdata->update_time` (last successful SHA256d workbase time) to decide when to call `update_base(GEN_NORMAL)`
- During non-SHA256d blocks `update_time` is intentionally never updated, so the condition was permanently true, causing `update_base(GEN_NORMAL)` to be attempted on every loop iteration (rate-limited only by the semaphore) rather than once per `update_interval`
- Fixed by introducing `last_base_check` (local to `stratum_loop`) that records the time of each GEN_NORMAL attempt regardless of whether the block was skipped; the do-while condition now compares against `last_base_check`, giving exactly one GEN_NORMAL attempt per `update_interval` during non-SHA256d periods

### 11. Fix getblocktemplate algo parameter for DGB SHA-256d (`src/bitcoin.c`)
- DGB's `getblocktemplate` RPC takes an optional second argument `algo` (default: `"scrypt"`)
- `gbt_req` did not specify the algo, so DGB always returned Scrypt templates (`pow_algo_id=1`) and SHA-256d work was never produced
- Fixed by adding `"sha256d"` as the second element of the `params` array in `gbt_req`
- This was the root cause of SHA-256d blocks never being processed: `block_update()` always skipped the template as non-SHA256d, so `current_workbase` was never set and miners never received work

### 12. Fix btcdnotify allocation size (`src/ckpool.c`)
- `sizeof(bool *)` (pointer size, 8 bytes on 64-bit) was used instead of `sizeof(bool)` (1 byte), over-allocating the array and masking the real element size

### 11. Remove startup wait loop in `parse_instance_msg()` (`src/stratifier.c`)
- `parse_instance_msg()` had a loop that blocked all incoming messages until `current_workbase` was set
- On DGB, `current_workbase` is NULL during every non-SHA-256d block (~75s intervals), making the wait indefinite
- The loop has been removed entirely; each handler (`parse_subscribe`, `parse_authorise`, etc.) already guards against a NULL workbase and returns an appropriate response immediately

## Tested environment
- Ubuntu 22.04 LTS
- DigiByte Core 8.26.2
- OpenSSL 3.0
- GCC 11
- Avalon Nano 3S (SHA-256d, ~6.5 TH/s)

## Build

```bash
sudo apt install -y build-essential yasm \
  libpq-dev libgsl-dev libssl-dev \
  autoconf automake libtool \
  libzmq3-dev postgresql postgresql-contrib

./autogen.sh
./configure --with-ckdb
make -j$(nproc)
sudo make install
```

## PostgreSQL初期設定

ckdbを初めて使う場合、`idcontrol`テーブルの初期データが必要です：

```sql
INSERT INTO idcontrol (idname, lastid, createdate, createby, createcode, createinet, modifyby, modifycode, modifyinet)
VALUES 
('userid', 1, NOW(), 'admin', 'init', '127.0.0.1', '', '', ''),
('workerid', 1, NOW(), 'admin', 'init', '127.0.0.1', '', '', ''),
('authid', 1, NOW(), 'admin', 'init', '127.0.0.1', '', '', ''),
('poolstatid', 1, NOW(), 'admin', 'init', '127.0.0.1', '', '', ''),
('shareid', 1, NOW(), 'admin', 'init', '127.0.0.1', '', '', ''),
('workinfoid', 1, NOW(), 'admin', 'init', '127.0.0.1', '', '', ''),
('payoutid', 1, NOW(), 'admin', 'init', '127.0.0.1', '', '', ''),
('paymentid', 1, NOW(), 'admin', 'init', '127.0.0.1', '', '', '');
```

## Notes
- SHA-256d only (Skein not supported by ckpool)
- Tested with DigiByte (DGB) mainnet
- PPLNS support via ckdb + PostgreSQL

## Original
- https://github.com/ctubio/ckpool
- https://bitbucket.org/ckolivas/ckpool
