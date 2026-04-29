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
