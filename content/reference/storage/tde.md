# CUBRID TDE (Transparent Data Encryption) Module — Comprehensive Analysis

**Generated from:** `src/storage/tde.c` + `src/storage/tde.h`
**Date:** 2026-03-27
**Analyst:** oh-my-claudecode executor (claude-sonnet-4-6)

---

## 1. File Overview

| Attribute        | Value                                         |
|------------------|-----------------------------------------------|
| **Source file**  | `src/storage/tde.c`                           |
| **Header file**  | `src/storage/tde.h`                           |
| **Line count**   | 1,720 lines (tde.c) + 214 lines (tde.h)       |
| **Language**     | C (compiled as C++17 via `c_to_cpp.sh`)       |
| **Module**       | `storage/` — Storage engine layer             |
| **Purpose**      | Transparent Data Encryption — encrypts/decrypts data pages and log pages at rest without requiring changes to the upper layers of the database |

### Purpose in Detail

TDE provides at-rest encryption for CUBRID databases. The design goal is complete transparency to the query engine: pages are decrypted on read from disk into the buffer pool and re-encrypted on flush to disk. The upper layers (query executor, optimizer, transaction manager) work entirely on plaintext data in memory.

The module manages a two-level key hierarchy:
- **Master Key (MK):** A 256-bit key stored in a dedicated flat file (`<dbname>_keys`), never in the database itself. It is managed externally by a DBA.
- **Data Keys (DK):** Three 256-bit keys (permanent, temporary, log), each encrypted under the master key and stored in a heap record (`TDE_KEYINFO`) inside the database. On server restart, the master key is used to decrypt the data keys into an in-memory `TDE_CIPHER` structure.

### Build Mode Applicability

| Build mode     | TDE support                                                              |
|----------------|--------------------------------------------------------------------------|
| `SERVER_MODE`  | Full: all encryption, decryption, key management, key rotation           |
| `SA_MODE`      | Full: same as SERVER_MODE (both client and server in one process)        |
| `CS_MODE`      | Excluded (`#if !defined(CS_MODE)`). Key-file utility functions (create, add, find, delete, dump master key) are available in CS_MODE. Log page encryption/decryption is excluded pending `UNSTABLE_TDE_FOR_REPLICATION_LOG`. |

The file uses `#if !defined(CS_MODE)` to gate the cipher, data-page, log-page, and heap-backed keyinfo functions. Only the master-key file management functions (`tde_create_mk`, `tde_add_mk`, `tde_find_mk`, etc.) and the algorithm-name utility are compiled in all modes.

---

## 2. Includes & Dependencies

### Internal / Project Headers (within `src/`)

| Header | Provides |
|--------|----------|
| `heap_file.h` | `heap_insert_logical`, `heap_update_logical`, `heap_first`, scan cache APIs (`!CS_MODE` only) |
| `btree.h` | Pulled in for `RVBT_*` recovery index constants used in `LOG_MAY_CONTAIN_USER_DATA` macro |
| `system_parameter.h` | `prm_get_string_value(PRM_ID_TDE_KEYS_FILE_PATH)`, `prm_get_bool_value(PRM_ID_ER_LOG_TDE)` |
| `boot_sr.h` | `boot_db_full_name()`, `boot_Db_parm->tde_keyinfo_hfid` |
| `file_io.h` | `fileio_mount`, `fileio_dismount`, `fileio_open`, `fileio_close`, `fileio_is_volume_exist`, `fileio_make_keys_name`, `fileio_unformat_and_rename` |
| `error_manager.h` | `er_set`, `er_errid`, `ASSERT_ERROR`, `ASSERT_ERROR_AND_SET` |
| `error_code.h` | All `ER_TDE_*` error constants |
| `log_storage.hpp` | `LOG_PAGE`, `LOG_HDRPAGE`, `LOG_PAGESIZE`, `LOG_IS_PAGE_TDE_ENCRYPTED` |
| `log_volids.hpp` | `LOG_DBTDE_KEYS_VOLID`, `LOG_DBCOPY_VOLID` volume ID constants |
| `tde.h` | Own header — types, macros, extern declarations |
| `memory_wrapper.hpp` | CUBRID memory override layer — **MUST BE LAST** |

### Cross-Module (system) Headers

| Header | Provides |
|--------|----------|
| `<openssl/conf.h>` | OpenSSL configuration |
| `<openssl/evp.h>` | `EVP_CIPHER_CTX`, `EVP_aes_256_ctr`, `EVP_aria_256_ctr`, `EVP_EncryptInit_ex`, `EVP_EncryptUpdate`, `EVP_EncryptFinal_ex`, `EVP_DecryptInit_ex`, `EVP_DecryptUpdate`, `EVP_DecryptFinal_ex`, `EVP_CIPHER_CTX_free` |
| `<openssl/err.h>` | OpenSSL error handling |
| `<openssl/sha.h>` | `SHA256_CTX`, `SHA256_Init`, `SHA256_Update`, `SHA256_Final` |
| `<openssl/rand.h>` | `RAND_bytes` (CSPRNG for key generation) |
| `<signal.h>` | `sigset_t`, `sigfillset`, `sigdelset`, `sigprocmask` (signal masking during file I/O) |
| `<sys/types.h>`, `<sys/stat.h>`, `<fcntl.h>` | POSIX file I/O primitives |
| `<stdlib.h>`, `<assert.h>` | Standard C utilities |
| `<io.h>` | Windows-only file I/O |

### Reverse Dependencies (who includes/uses tde.h or tde.c symbols)

| File | Usage |
|------|-------|
| `src/storage/page_buffer.c` | `tde_encrypt_data_page`, `tde_decrypt_data_page`, `pgbuf_set_tde_algorithm`, `pgbuf_get_tde_algorithm` |
| `src/transaction/boot_sr.c` | `tde_initialize`, `tde_cipher_initialize`, `tde_is_loaded` |
| `src/transaction/log_writer.c` | `tde_encrypt_log_page`, `tde_Cipher.data_keys.log_key`, `logwr_set_tde_algorithm` |
| `src/transaction/log_applier.c` | `tde_decrypt_log_page`, `LOG_IS_PAGE_TDE_ENCRYPTED` |
| `src/communication/network_interface_sr.cpp` | `stde_get_data_keys`, `stde_is_loaded`, `stde_get_mk_file_path`, `stde_get_mk_info`, `stde_change_mk_on_server`, `xtde_get_mk_info`, `xtde_change_mk_without_flock` |
| `src/base/xserver_interface.h` | Forward declarations of `xtde_*` server-side RPC handlers |

---

## 3. Preprocessor & Compilation

### Conditional Compilation Guards

```c
#if !defined(CS_MODE)
  // The following are excluded from client-library builds:
  // - TDE_CIPHER struct definition and tde_Cipher global
  // - tde_Keyinfo_oid, tde_Keyinfo_hfid static globals
  // - tde_initialize(), tde_cipher_initialize(), tde_is_loaded()
  // - tde_generate_keyinfo(), tde_get_keyinfo(), tde_update_keyinfo()
  // - tde_change_mk(), tde_load_mk(), tde_load_dks()
  // - tde_validate_mk(), tde_make_mk_hash()
  // - tde_create_dk(), tde_encrypt_dk(), tde_decrypt_dk(), tde_dk_nonce()
  // - tde_encrypt_internal(), tde_decrypt_internal()
  // - tde_encrypt_data_page(), tde_decrypt_data_page()
  // - tde_encrypt_log_page(), tde_decrypt_log_page()
  // - xtde_get_mk_info(), xtde_change_mk_without_flock()
  // - LOG_MAY_CONTAIN_USER_DATA() macro
  // - tde_er_log() macro
#endif

#ifdef UNSTABLE_TDE_FOR_REPLICATION_LOG
  // Future: CS_MODE will define TDE_HA_SOCK_NAME for HA log replication
#endif
```

### Key Macros (from tde.h)

| Macro | Value | Meaning |
|-------|-------|---------|
| `TDE_DK_ALGORITHM` | `TDE_ALGORITHM_AES` | Algorithm used to encrypt data keys under master key |
| `TDE_DATA_PAGE_ENC_OFFSET` | `sizeof(FILEIO_PAGE_RESERVED)` | Skip page header, encrypt from here |
| `TDE_DATA_PAGE_ENC_LENGTH` | `DB_PAGESIZE` | Length of data encrypted per data page |
| `TDE_LOG_PAGE_ENC_OFFSET` | `sizeof(LOG_HDRPAGE)` | Skip log page header |
| `TDE_LOG_PAGE_ENC_LENGTH` | `LOG_PAGESIZE - TDE_LOG_PAGE_ENC_OFFSET` | Rest of log page |
| `TDE_DATA_PAGE_NONCE_LENGTH` | `16` (128 bits) | Nonce size for data pages |
| `TDE_LOG_PAGE_NONCE_LENGTH` | `16` (128 bits) | Nonce size for log pages |
| `TDE_DK_NONCE_LENGTH` | `16` (128 bits) | Nonce size for data key encryption |
| `TDE_MASTER_KEY_LENGTH` | `32` (256 bits) | Master key size |
| `TDE_DATA_KEY_LENGTH` | `32` (256 bits) | Data key size |
| `TDE_MK_FILE_CONTENTS_START` | `CUBRID_MAGIC_MAX_LENGTH` | Key file items start after magic bytes |
| `TDE_MK_FILE_ITEM_SIZE` | `sizeof(TDE_MK_FILE_ITEM)` | Size of one key file slot |
| `TDE_MK_FILE_ITEM_OFFSET(index)` | `CONTENTS_START + ITEM_SIZE * index` | Byte offset for key at index |
| `TDE_MK_FILE_ITEM_INDEX(offset)` | `(offset - CONTENTS_START) / ITEM_SIZE` | Index from byte offset |
| `TDE_MK_FILE_ITEM_COUNT_MAX` | `128` | Maximum master keys per file |

### Signal Masking Macros

```c
#define off_signals(new_mask, old_mask)  // sigfillset, keep SIGINT/QUIT/TERM/HUP/ABRT, sigprocmask SIG_SETMASK
#define restore_signals(old_mask)        // sigprocmask SIG_SETMASK to restore
```

Used in all file I/O operations to prevent signal interruption during atomic key file writes. Windows builds skip this (`#if !defined(WINDOWS)`).

---

## 4. Data Structures & Types

### `TDE_ALGORITHM` (enum, tde.h:71)

```c
typedef enum {
  TDE_ALGORITHM_NONE = 0,   // No encryption
  TDE_ALGORITHM_AES  = 1,   // AES-256-CTR
  TDE_ALGORITHM_ARIA = 2,   // ARIA-256-CTR
} TDE_ALGORITHM;
```

Values double as indices into a name string array. Each also maps to OpenSSL EVP cipher types: `EVP_aes_256_ctr()` and `EVP_aria_256_ctr()` respectively. `NONE` is the default (unencrypted) state.

Stored in page flags: `FILEIO_PAGE_FLAG_ENCRYPTED_AES = 0x1`, `FILEIO_PAGE_FLAG_ENCRYPTED_ARIA = 0x2`, both in the `pflag` byte of `FILEIO_PAGE_RESERVED`.

### `TDE_DATA_KEY_TYPE` (enum, tde.h:78)

```c
typedef enum tde_data_key_type {
  TDE_DATA_KEY_TYPE_PERM,   // Permanent (heap, btree) pages
  TDE_DATA_KEY_TYPE_TEMP,   // Temporary file pages
  TDE_DATA_KEY_TYPE_LOG,    // WAL log pages
} TDE_DATA_KEY_TYPE;
```

Used internally to select which data key encrypts a given page type and to derive the deterministic nonce used during data-key encryption.

### `TDE_DATA_KEY_SET` (struct, tde.h:85)

```c
typedef struct tde_data_key_set {
  unsigned char perm_key[32];   // Key for permanent (heap/btree) pages
  unsigned char temp_key[32];   // Key for temporary file pages
  unsigned char log_key[32];    // Key for WAL log pages
} TDE_DATA_KEY_SET;
```

Three independent 256-bit AES/ARIA keys; all live only in process memory (in `tde_Cipher`). Each is generated once via `RAND_bytes` during `tde_initialize`, then stored encrypted in the heap record.

### `TDE_MK_FILE_ITEM` (struct, tde.h:92)

```c
typedef struct tde_mk_file_item {
  time_t        created_time;          // -1 means slot is deleted/available
  unsigned char master_key[32];        // The master key bytes, stored in plaintext in _keys file
} TDE_MK_FILE_ITEM;
```

Represents one slot in the flat-file key store. Deletion is a soft-delete: `created_time` is set to `-1`. Up to 128 items fit in the file (`TDE_MK_FILE_ITEM_COUNT_MAX`). The file begins with `CUBRID_MAGIC_KEYS` magic bytes, then items at sequential offsets calculated by `TDE_MK_FILE_ITEM_OFFSET(index)`.

**Security note:** Master keys are stored in plaintext in the `_keys` file. Protection is by OS file permissions (created with mode `0600`) and DBA responsibility.

### `TDE_CIPHER` (struct, tde.h:148, `!CS_MODE` only)

```c
typedef struct tde_cipher {
  bool             is_loaded;           // Whether cipher was initialized successfully
  TDE_DATA_KEY_SET data_keys;           // Three decrypted data keys (perm, temp, log)
  int64_t          temp_write_counter;  // Atomic counter used as nonce for temp pages
} TDE_CIPHER;
```

The core in-memory cipher state. One global instance: `TDE_CIPHER tde_Cipher`. All encryption/decryption operations read from this. The `temp_write_counter` is the only field modified at runtime during normal operation; it is incremented atomically via `ATOMIC_INC_64` on each temp page write to ensure nonce uniqueness.

### `TDE_KEYINFO` (struct, tde.h:160, `!CS_MODE` only)

```c
typedef struct tde_keyinfo {
  int           mk_index;             // Index of master key in _keys file
  time_t        created_time;         // When the master key was created
  time_t        set_time;             // When this keyinfo was last written
  unsigned char mk_hash[32];          // SHA-256 hash of master key (for validation)
  unsigned char dk_perm[32];          // Encrypted permanent data key
  unsigned char dk_temp[32];          // Encrypted temporary data key
  unsigned char dk_log[32];           // Encrypted log data key
} TDE_KEYINFO;
```

Persisted in a special heap file (HFID stored in `boot_Db_parm->tde_keyinfo_hfid`). On server restart, this record is read and used to decrypt the data keys into `tde_Cipher`. The master key hash enables validation without exposing the plaintext master key in the database.

**Storage layout trick:** The heap record prepends a 4-byte `repid_and_flag_bits = 0` integer before `TDE_KEYINFO` to prevent vacuum from misinterpreting the record type during undo operations. See the "HACK" comments in `tde_initialize` and `tde_update_keyinfo`.

---

## 5. Global & Static Variables

### Global (extern)

| Symbol | Type | Location | Purpose |
|--------|------|----------|---------|
| `tde_Cipher` | `TDE_CIPHER` | `tde.c:71` | Global cipher state; holds loaded data keys and temp write counter |

Declared `extern` in `tde.h` and accessed directly by `network_interface_sr.cpp` (to pack data keys for HA replication via `stde_get_data_keys`).

### Static (file-local)

| Symbol | Type | Location | Purpose |
|--------|------|----------|---------|
| `tde_Keyinfo_oid` | `OID` | `tde.c:73` | OID of the keyinfo heap record (set in `tde_initialize`, loaded in `tde_cipher_initialize`) |
| `tde_Keyinfo_hfid` | `HFID` | `tde.c:74` | HFID of the keyinfo heap file (copied from `boot_Db_parm`) |

Both are initialized to null/zero values via `OID_INITIALIZER` and `HFID_INITIALIZER`. They are populated at server startup and remain constant thereafter.

---

## 6. Function Catalog

### Public Functions (non-static)

---

#### `tde_initialize`
```c
int tde_initialize(THREAD_ENTRY *thread_p, HFID *keyinfo_hfid);
```
**File:** `tde.c:108` | **Mode:** `!CS_MODE` | **Visibility:** `extern`

**Description:** First-time TDE bootstrap during `boot_db_create`. Creates the `_keys` file if it does not exist, generates a default master key and three data keys, encrypts the data keys under the master key, stores the `TDE_KEYINFO` record in a new heap record, and saves `tde_Keyinfo_oid` and `tde_Keyinfo_hfid`.

**Algorithm:**
1. Construct key file path via `tde_make_keys_file_fullname`.
2. Attempt `tde_create_keys_file`. If `ER_BO_VOLUME_EXISTS`, mount the existing file instead; if neither, fail.
3. If newly created: generate default MK via `tde_create_mk`, add to file via `tde_add_mk`.
4. If existing: find first valid MK via `tde_find_first_mk`.
5. Generate three random data keys via `tde_create_dk` (perm, temp, log).
6. Generate `TDE_KEYINFO` via `tde_generate_keyinfo` (hashes MK, encrypts DKs under MK).
7. Prepend 4-byte zero `repid_and_flag_bits` to avoid vacuum adjustment on undo.
8. Insert heap record via `heap_insert_logical` into `keyinfo_hfid`.
9. Save `tde_Keyinfo_hfid` and `tde_Keyinfo_oid`.
10. Always `fileio_dismount` in `exit` label.

**Error handling:** Uses `goto exit` pattern; all errors propagate upward. `fileio_dismount` is always called.

**Callers:** `boot_sr.c:5104` during `xboot_initialize_server` / `boot_db_create`.

---

#### `tde_cipher_initialize`
```c
int tde_cipher_initialize(THREAD_ENTRY *thread_p, const HFID *keyinfo_hfid, const char *mk_path_given);
```
**File:** `tde.c:234` | **Mode:** `!CS_MODE` | **Visibility:** `extern`

**Description:** Loads TDE at server restart. Mounts the key file, validates it, reads `TDE_KEYINFO` from the heap, loads and validates the master key, decrypts the three data keys into `tde_Cipher`, and sets `tde_Cipher.is_loaded = true`.

**Algorithm:**
1. If `mk_path_given != NULL`, try to mount it (backup path during `restoredb`).
2. If mount fails or `mk_path_given == NULL`, use the default path from system parameter.
3. Validate the key file magic via `tde_validate_keys_file`.
4. Copy `keyinfo_hfid` into static `tde_Keyinfo_hfid`.
5. Read `TDE_KEYINFO` from heap via `tde_get_keyinfo`.
6. Load and validate MK via `tde_load_mk` (reads from file, checks hash and timestamp).
7. Decrypt data keys via `tde_load_dks`.
8. Reset `temp_write_counter = 0`, set `is_loaded = true`.
9. Always dismount in `exit`.

**Error handling:** Returns early with `er_errid()` if mount fails. `goto exit` for all subsequent errors.

**Callers:** `boot_sr.c:2324` during `boot_restart_server`; also `boot_sr.c:5233` during standby restart.

---

#### `tde_is_loaded`
```c
bool tde_is_loaded(void);
```
**File:** `tde.c:310` | **Mode:** `!CS_MODE` | **Visibility:** `extern`

**Description:** Returns `tde_Cipher.is_loaded`. Used as a guard in all encryption/decryption entry points and in `pgbuf_set_tde_algorithm`.

**Callers:** `tde_encrypt_data_page`, `tde_decrypt_data_page`, `tde_encrypt_log_page`, `tde_decrypt_log_page`, `tde_change_mk`, `xtde_get_mk_info`, `page_buffer.c:4883`, `network_interface_sr.cpp:2936`, `boot_sr.c:2869`.

---

#### `tde_make_keys_file_fullname`
```c
void tde_make_keys_file_fullname(char *keys_vol_fullname, const char *db_full_name, bool ignore_parm);
```
**File:** `tde.c:505` | **Mode:** All | **Visibility:** `extern`

**Description:** Computes the full path of the `_keys` file. If `ignore_parm` is false and the system parameter `TDE_KEYS_FILE_PATH` is set, uses that path with the database basename. Otherwise uses the default `fileio_make_keys_name` naming convention (`<dbname>_keys`).

**Algorithm:**
1. Read `PRM_ID_TDE_KEYS_FILE_PATH`.
2. If parameter is empty or `ignore_parm` is true: `fileio_make_keys_name(keys_vol_fullname, db_full_name)`.
3. Otherwise: extract basename from `db_full_name`, call `fileio_make_keys_name_given_path`.

**Callers:** `tde_initialize`, `tde_cipher_initialize`, `xtde_change_mk_without_flock`, `network_interface_sr.cpp:3020`, `boot_sr.c` (backup/restore paths).

---

#### `tde_validate_keys_file`
```c
bool tde_validate_keys_file(int vdes);
```
**File:** `tde.c:371` | **Mode:** All | **Visibility:** `extern`

**Description:** Validates a key file by checking the `CUBRID_MAGIC_KEYS` magic bytes at offset 0.

**Algorithm:**
1. Mask signals.
2. `lseek` to position 0.
3. `read` `CUBRID_MAGIC_MAX_LENGTH` bytes.
4. `memcmp` against `CUBRID_MAGIC_KEYS`.
5. Restore signals.

Returns `true` if magic matches, `false` otherwise.

---

#### `tde_copy_keys_file`
```c
int tde_copy_keys_file(THREAD_ENTRY *thread_p, const char *dest_fullname, const char *src_fullname,
                       bool keep_dest_mount, bool keep_src_mount);
```
**File:** `tde.c:411` | **Mode:** All | **Visibility:** `extern`

**Description:** Copies the source `_keys` file to the destination. Used during database backup to copy the key file alongside the data volumes. The `keep_*_mount` flags allow the caller to retain the file descriptors after the call (e.g., for subsequent operations in the same backup session).

**Algorithm:**
1. Mount `src_fullname`, validate magic.
2. If dest exists, remove it via `fileio_unformat_and_rename`.
3. Create new dest via `tde_create_keys_file`.
4. Mount dest.
5. `lseek` source to 0, read/write loop in 4096-byte chunks, handling `EINTR` by retrying partial writes.
6. Dismount src and/or dest based on `keep_*` flags.

**Error handling:** Any write error sets `ER_IO_WRITE` via `er_set_with_oserror` and returns. Dest file is cleaned up via `fileio_unformat_and_rename` on failure.

---

#### `tde_load_mk`
```c
int tde_load_mk(int vdes, const TDE_KEYINFO *keyinfo, unsigned char *master_key);
```
**File:** `tde.c:706` | **Mode:** `!CS_MODE` | **Visibility:** `extern`

**Description:** Finds the master key in the `_keys` file at `keyinfo->mk_index`, validates it matches the hash and creation timestamp stored in `keyinfo`, and returns it in `master_key`.

**Algorithm:**
1. `tde_find_mk(vdes, keyinfo->mk_index, mk, &created_time)`.
2. Validate: `tde_validate_mk(mk, keyinfo->mk_hash) && created_time == keyinfo->created_time`.
3. If valid: `memcpy(master_key, mk, TDE_MASTER_KEY_LENGTH)`.
4. If invalid: `ER_TDE_INVALID_MASTER_KEY`.

**Callers:** `tde_cipher_initialize`.

---

#### `tde_change_mk`
```c
int tde_change_mk(THREAD_ENTRY *thread_p, const int mk_index, const unsigned char *master_key,
                  const time_t created_time);
```
**File:** `tde.c:662` | **Mode:** `!CS_MODE` | **Visibility:** `extern`

**Description:** Re-encrypts the current data keys under a new master key and persists the updated `TDE_KEYINFO` to heap. Does NOT replace the data keys themselves — existing encrypted data remains readable because the same data keys are re-wrapped. Calls `heap_flush` after update to ensure durability before the old MK can be deleted.

**Algorithm:**
1. Check `tde_is_loaded()`.
2. `tde_generate_keyinfo` with new MK, using `tde_Cipher.data_keys` (existing DKs).
3. `tde_update_keyinfo` to write to heap.
4. `heap_flush(thread_p, &tde_Keyinfo_oid)` — forces heap page to disk.

**Callers:** `xtde_change_mk_without_flock`.

---

#### `tde_get_keyinfo`
```c
int tde_get_keyinfo(THREAD_ENTRY *thread_p, TDE_KEYINFO *keyinfo);
```
**File:** `tde.c:570` | **Mode:** `!CS_MODE` | **Visibility:** `extern`

**Description:** Reads the `TDE_KEYINFO` record from the heap file via `heap_first` and `heap_scancache_quick_start_with_class_hfid`. Strips the leading 4-byte `repid_and_flag_bits` padding. Asserts that `tde_Keyinfo_hfid` is not null.

**Callers:** `tde_cipher_initialize`, `xtde_get_mk_info`, `xtde_change_mk_without_flock`.

---

#### `tde_encrypt_data_page`
```c
int tde_encrypt_data_page(const FILEIO_PAGE *iopage_plain, TDE_ALGORITHM tde_algo,
                          bool is_temp, FILEIO_PAGE *iopage_cipher);
```
**File:** `tde.c:909` | **Mode:** `!CS_MODE` | **Visibility:** `extern`

**Description:** Encrypts the content of a data page in-place (or into a separate output buffer). The page header (`FILEIO_PAGE_RESERVED`) and watermark (`FILEIO_PAGE_WATERMARK`) are copied unencrypted. Only the `DB_PAGESIZE` body is encrypted.

**Nonce selection:**
- **Permanent pages:** The page LSA (`iopage_plain->prv.lsa`) serves as the nonce — unique per write because CUBRID never overwrites a page with an earlier LSA.
- **Temporary pages:** An atomic counter `ATOMIC_INC_64(&tde_Cipher.temp_write_counter, 1)` provides a monotonically increasing nonce, stored as an `int64_t` in the low 8 bytes of the 16-byte nonce field.

The chosen nonce is stored in `iopage_cipher->prv.tde_nonce` so decryption can recover it without additional lookups.

**Algorithm:**
1. Check `tde_is_loaded()`.
2. Select key and nonce based on `is_temp`.
3. `memcpy` the header and watermark unchanged into `iopage_cipher`.
4. Store nonce in `iopage_cipher->prv.tde_nonce`.
5. Call `tde_encrypt_internal` on `[ENC_OFFSET .. ENC_OFFSET + ENC_LENGTH)`.

**Caller:** `page_buffer.c:10532` inside `pgbuf_write_to_disk_or_dwb` — called just before writing a BCB's page to disk (or DWB).

---

#### `tde_decrypt_data_page`
```c
int tde_decrypt_data_page(const FILEIO_PAGE *iopage_cipher, TDE_ALGORITHM tde_algo,
                          bool is_temp, FILEIO_PAGE *iopage_plain);
```
**File:** `tde.c:962` | **Mode:** `!CS_MODE` | **Visibility:** `extern`

**Description:** Symmetric inverse of `tde_encrypt_data_page`. Recovers the nonce from `iopage_cipher->prv.tde_nonce`, selects the appropriate key, decrypts the page body.

**Algorithm:**
1. Check `tde_is_loaded()`.
2. Select key based on `is_temp`.
3. `memcpy` header and watermark unchanged into `iopage_plain`.
4. Recover nonce from `iopage_cipher->prv.tde_nonce`.
5. Call `tde_decrypt_internal` on the encrypted region.

**Caller:** `page_buffer.c:8277` inside `pgbuf_fix` — called just after a page is read from disk into a BCB, before it becomes visible to upper layers.

---

#### `tde_encrypt_log_page`
```c
int tde_encrypt_log_page(const LOG_PAGE *logpage_plain, TDE_ALGORITHM tde_algo, LOG_PAGE *logpage_cipher);
```
**File:** `tde.c:1010` | **Mode:** `!CS_MODE` | **Visibility:** `extern`

**Description:** Encrypts the body of a log page. The `LOG_HDRPAGE` header (containing `logical_pageid`) is copied unencrypted; the remainder is encrypted.

**Nonce:** The `logpage_plain->hdr.logical_pageid` (a monotonically increasing log page sequence number) is used directly as the nonce. This is deterministic — the same logical page ID always produces the same nonce — which is safe because each physical log page write corresponds to exactly one logical page ID under CUBRID's WAL model.

**Callers:** `log_writer.c:917`, `log_writer.c:982` — when shipping log pages to a standby.

---

#### `tde_decrypt_log_page`
```c
int tde_decrypt_log_page(const LOG_PAGE *logpage_cipher, TDE_ALGORITHM tde_algo, LOG_PAGE *logpage_plain);
```
**File:** `tde.c:1040` | **Mode:** `!CS_MODE` | **Visibility:** `extern`

**Description:** Symmetric inverse of `tde_encrypt_log_page`. Recovers `logical_pageid` from the header (which is not encrypted), reconstructs the nonce, and decrypts.

**Callers:** `log_applier.c:1050`, `log_applier.c:1119`, `log_applier.c:7396`.

---

#### `xtde_get_mk_info`
```c
int xtde_get_mk_info(THREAD_ENTRY *thread_p, int *mk_index, time_t *created_time, time_t *set_time);
```
**File:** `tde.c:1227` | **Mode:** `!CS_MODE` | **Visibility:** `extern` (xserver RPC handler)

**Description:** Server-side RPC handler. Returns the master key index, its creation time, and when it was last set on the database. Used by the `tde` admin utility.

**Callers:** `network_interface_sr.cpp` via `stde_get_mk_info`.

---

#### `xtde_change_mk_without_flock`
```c
int xtde_change_mk_without_flock(THREAD_ENTRY *thread_p, const int mk_index);
```
**File:** `tde.c:1259` | **Mode:** `!CS_MODE` | **Visibility:** `extern` (xserver RPC handler)

**Description:** Server-side handler for master key rotation. Opens the `_keys` file without a lock (the client-side `tde()` utility holds the file lock). Reads the specified master key, validates the previous key still exists, then calls `tde_change_mk` to re-wrap the data keys.

**Algorithm:**
1. Open `_keys` file with `O_RDONLY` (no mount/lock).
2. `tde_find_mk` for `mk_index`.
3. `tde_get_keyinfo` — read current keyinfo from heap.
4. If same key already set (index and hash match), exit early.
5. Verify previous key (`keyinfo.mk_index`) still exists in the file.
6. `tde_change_mk(thread_p, mk_index, master_key, created_time)`.
7. Close file.

**Callers:** `network_interface_sr.cpp` via `stde_change_mk_on_server`.

---

#### `tde_create_mk`
```c
int tde_create_mk(unsigned char *master_key, time_t *created_time);
```
**File:** `tde.c:1324` | **Mode:** All | **Visibility:** `extern`

**Description:** Generates a new 256-bit master key using `RAND_bytes` (OpenSSL CSPRNG) and records the current time as `created_time`.

**Error handling:** Returns `ER_TDE_KEY_CREATION_FAIL` if `RAND_bytes` fails.

**Callers:** `tde_initialize` (first-time bootstrap), `tde` admin utility (via CS_MODE client).

---

#### `tde_print_mk`
```c
void tde_print_mk(const unsigned char *master_key);
```
**File:** `tde.c:1344` | **Mode:** All | **Visibility:** `extern`

**Description:** Prints all 32 bytes of the master key as lowercase hex to `stdout`. Used by `tde_dump_mks` when `print_value` is true and by the `tde` utility for key display.

---

#### `tde_add_mk`
```c
int tde_add_mk(int vdes, const unsigned char *master_key, time_t created_time, int *mk_index);
```
**File:** `tde.c:1364` | **Mode:** All | **Visibility:** `extern`

**Description:** Appends (or recycles) a master key entry in the `_keys` file. Scans from `TDE_MK_FILE_CONTENTS_START` for the first slot with `created_time == -1` (soft-deleted); if none found, appends at EOF. Enforces the 128-slot maximum. Writes then `fsync`s.

**Algorithm:**
1. Mask signals.
2. Seek to `TDE_MK_FILE_CONTENTS_START`.
3. Read items sequentially; stop at first invalidated slot or EOF.
4. Check `TDE_MK_FILE_ITEM_INDEX(current_position) < 128`.
5. Set `*mk_index = current_index`.
6. Write item, `fsync`.
7. Restore signals.

**Error handling:** `ER_TDE_MAX_KEY_FILE` if 128 slots are full; `ER_IO_READ` on read failure; `ER_TDE_INVALID_KEYS_FILE` if partial read.

**Callers:** `tde_initialize`, `tde` admin utility.

---

#### `tde_find_mk`
```c
int tde_find_mk(int vdes, int mk_index, unsigned char *master_key, time_t *created_time);
```
**File:** `tde.c:1450` | **Mode:** All | **Visibility:** `extern`

**Description:** Reads the key file item at `TDE_MK_FILE_ITEM_OFFSET(mk_index)`, checks it is not soft-deleted (`created_time != -1`), and copies key and time to output parameters. Either output parameter may be `NULL` (used for existence-only checks, e.g., in `xtde_change_mk_without_flock`).

**Error handling:** `ER_TDE_MASTER_KEY_NOT_FOUND` on seek failure, short read, or soft-deleted item.

---

#### `tde_find_first_mk`
```c
int tde_find_first_mk(int vdes, int *mk_index, unsigned char *master_key, time_t *created_time);
```
**File:** `tde.c:1517` | **Mode:** All | **Visibility:** `extern`

**Description:** Scans the key file from index 0 and returns the first non-deleted entry. Used during `tde_initialize` when restoring from an existing `_keys` file.

**Error handling:** On partial read (unexpected EOF mid-item), returns `ER_TDE_INVALID_KEYS_FILE`. Returns `ER_FAILED` if the seek to the start fails.

---

#### `tde_delete_mk`
```c
int tde_delete_mk(int vdes, int mk_index);
```
**File:** `tde.c:1582` | **Mode:** All | **Visibility:** `extern`

**Description:** Soft-deletes a master key entry by reading the item, setting `created_time = -1`, seeking back, and rewriting + fsyncing. Used after a successful key rotation to invalidate the old master key.

**Error handling:** `ER_TDE_MASTER_KEY_NOT_FOUND` if item not found or already deleted.

---

#### `tde_dump_mks`
```c
int tde_dump_mks(int vdes, bool print_value);
```
**File:** `tde.c:1641` | **Mode:** All | **Visibility:** `extern`

**Description:** Iterates all items in the `_keys` file, prints index and creation time for valid entries (and optionally the hex key value), and prints a count summary. Used by the `tde` admin utility for diagnostics.

---

#### `tde_get_algorithm_name`
```c
const char *tde_get_algorithm_name(TDE_ALGORITHM tde_algo);
```
**File:** `tde.c:1706` | **Mode:** All | **Visibility:** `extern`

**Description:** Maps `TDE_ALGORITHM` enum to a human-readable string (`"NONE"`, `"AES"`, `"ARIA"`). Returns `NULL` for unknown values. Used in log messages and diagnostic output.

---

### Static (Internal) Functions

---

#### `tde_create_keys_file`
```c
static int tde_create_keys_file(const char *keyfile_fullname);
```
**File:** `tde.c:322` | **Mode:** `!CS_MODE`

**Description:** Creates a new `_keys` file with mode `0600` and writes the `CUBRID_MAGIC_KEYS` magic bytes. Returns `ER_BO_VOLUME_EXISTS` if the file already exists (used by `tde_initialize` to distinguish first-time vs. restart).

**Algorithm:**
1. Check `fileio_is_volume_exist` — if exists, return `ER_BO_VOLUME_EXISTS`.
2. `fileio_open` with `O_CREAT | O_RDWR`, mode `0600`.
3. Write `CUBRID_MAGIC_MAX_LENGTH` bytes of magic.
4. `fsync`.
5. `fileio_close` on exit.
6. Signal masking around file I/O.

---

#### `tde_generate_keyinfo`
```c
static int tde_generate_keyinfo(TDE_KEYINFO *keyinfo, int mk_index, const unsigned char *master_key,
                                const time_t created_time, const TDE_DATA_KEY_SET *dks);
```
**File:** `tde.c:533`

**Description:** Builds a `TDE_KEYINFO` struct from the given master key and data key set. Hashes the master key, encrypts each data key with `tde_encrypt_dk`, records timestamps.

**Steps:**
1. `keyinfo->mk_index = mk_index`.
2. `tde_make_mk_hash(master_key, keyinfo->mk_hash)`.
3. `tde_encrypt_dk(dks->perm_key, PERM, master_key, keyinfo->dk_perm)`.
4. `tde_encrypt_dk(dks->temp_key, TEMP, master_key, keyinfo->dk_temp)`.
5. `tde_encrypt_dk(dks->log_key, LOG,  master_key, keyinfo->dk_log)`.
6. `keyinfo->created_time = created_time; keyinfo->set_time = time(NULL)`.

---

#### `tde_update_keyinfo`
```c
static int tde_update_keyinfo(THREAD_ENTRY *thread_p, const TDE_KEYINFO *keyinfo);
```
**File:** `tde.c:608`

**Description:** Writes the updated `TDE_KEYINFO` to the heap record at `tde_Keyinfo_oid`. Uses `heap_scancache_start_modify` with `SINGLE_ROW_UPDATE`. Hacks `scan_cache.node.class_oid = *oid_Root_class_oid` to bypass the `heap_scancache_check_with_hfid` check (the keyinfo OID has no real class). Uses `UPDATE_INPLACE_CURRENT_MVCCID` to avoid creating MVCC versions.

---

#### `tde_validate_mk`
```c
static bool tde_validate_mk(const unsigned char *master_key, const unsigned char *mk_hash);
```
**File:** `tde.c:775`

**Description:** Hashes `master_key` with SHA-256 and compares the result to `mk_hash` using `memcmp`. Returns `true` if matching.

---

#### `tde_make_mk_hash`
```c
static void tde_make_mk_hash(const unsigned char *master_key, unsigned char *mk_hash);
```
**File:** `tde.c:795`

**Description:** Computes SHA-256 of `master_key` (32 bytes) into `mk_hash` (32 bytes) using `SHA256_CTX`. Asserts `SHA256_DIGEST_LENGTH == TDE_MASTER_KEY_LENGTH`.

---

#### `tde_load_dks`
```c
static int tde_load_dks(const unsigned char *master_key, const TDE_KEYINFO *keyinfo);
```
**File:** `tde.c:743`

**Description:** Decrypts all three data keys from `keyinfo` using `master_key` and stores them directly into `tde_Cipher.data_keys`. Called once during `tde_cipher_initialize`.

---

#### `tde_create_dk`
```c
static int tde_create_dk(unsigned char *data_key);
```
**File:** `tde.c:815`

**Description:** Generates a 256-bit data key via `RAND_bytes`. Returns `ER_TDE_KEY_CREATION_FAIL` on failure.

---

#### `tde_encrypt_dk`
```c
static int tde_encrypt_dk(const unsigned char *dk_plain, TDE_DATA_KEY_TYPE dk_type,
                           const unsigned char *master_key, unsigned char *dk_cipher);
```
**File:** `tde.c:839`

**Description:** Wraps `tde_encrypt_internal` for data-key encryption. Derives a deterministic nonce via `tde_dk_nonce(dk_type)` — all-zeros for PERM, all-ones for TEMP, all-twos for LOG. Uses `TDE_DK_ALGORITHM` (AES-256-CTR). The deterministic nonce is safe here because the master key is changed atomically and each (key, nonce) pair is used exactly once.

---

#### `tde_decrypt_dk`
```c
static int tde_decrypt_dk(const unsigned char *dk_cipher, TDE_DATA_KEY_TYPE dk_type,
                           const unsigned char *master_key, unsigned char *dk_plain);
```
**File:** `tde.c:859`

**Description:** Symmetric inverse of `tde_encrypt_dk`. Derives the same deterministic nonce and calls `tde_decrypt_internal`.

---

#### `tde_dk_nonce` (inline)
```c
static inline void tde_dk_nonce(TDE_DATA_KEY_TYPE dk_type, unsigned char *dk_nonce);
```
**File:** `tde.c:876`

**Description:** Sets `dk_nonce` (16 bytes) to `0x00` (PERM), `0x01` (TEMP), or `0x02` (LOG) via `memset`. The differing nonces ensure that even if two data keys happen to be identical bytes (extremely unlikely), their encrypted forms in the keyinfo will differ.

---

#### `tde_encrypt_internal`
```c
static int tde_encrypt_internal(const unsigned char *plain_buffer, int length, TDE_ALGORITHM tde_algo,
                                 const unsigned char *key, const unsigned char *nonce,
                                 unsigned char *cipher_buffer);
```
**File:** `tde.c:1075`

**Description:** Core encryption function using OpenSSL EVP. All encryption paths in the module funnel through here.

**Algorithm:**
1. `EVP_CIPHER_CTX_new()`.
2. Select cipher: `EVP_aes_256_ctr()` or `EVP_aria_256_ctr()`.
3. `EVP_EncryptInit_ex(ctx, cipher_type, NULL, key, nonce)`.
4. `EVP_EncryptUpdate(ctx, cipher_buffer, &len, plain_buffer, length)`.
5. `EVP_EncryptFinal_ex(ctx, cipher_buffer + len, &len)` — for CTR mode, no padding, so this is a no-op but required.
6. Assert `cipher_len == length` (stream cipher: output == input size).
7. `EVP_CIPHER_CTX_free(ctx)`.

**Error handling:** Initializes `err = ER_TDE_ENCRYPTION_ERROR`. Any OpenSSL failure falls through `cleanup` to `exit` where `er_set` is called. Returns `NO_ERROR` only if all steps succeed. CTX is always freed in `cleanup`.

---

#### `tde_decrypt_internal`
```c
static int tde_decrypt_internal(const unsigned char *cipher_buffer, int length, TDE_ALGORITHM tde_algo,
                                 const unsigned char *key, const unsigned char *nonce,
                                 unsigned char *plain_buffer);
```
**File:** `tde.c:1154`

**Description:** Symmetric inverse using `EVP_DecryptInit_ex`, `EVP_DecryptUpdate`, `EVP_DecryptFinal_ex`. Identical structure to `tde_encrypt_internal` (CTR mode: encryption and decryption are the same operation). Initializes `err = ER_TDE_DECRYPTION_ERROR`. Returns `ER_TDE_DECRYPTION_ERROR` on failure.

---

## 7. Key Algorithms & Logic Flows

### 7.1 Key Hierarchy

```
  ┌─────────────────────────────────────────────────┐
  │         _keys File (on OS filesystem)            │
  │  [MAGIC][MK item 0][MK item 1]...[MK item 127]  │
  │   Each item: time_t created_time + 32-byte key   │
  └─────────────────────┬───────────────────────────┘
                        │  tde_load_mk()
                        ▼
              Master Key (32 bytes, in memory only during init)
                        │
         ┌──────────────┼──────────────┐
         │ tde_decrypt_dk(PERM)        │ tde_decrypt_dk(TEMP)   tde_decrypt_dk(LOG)
         ▼              ▼              ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ perm_key │  │ temp_key │  │ log_key  │   ← tde_Cipher.data_keys (in memory)
  └──────────┘  └──────────┘  └──────────┘
       │               │             │
  heap/btree       temp files      WAL log
  page encrypt    page encrypt    page encrypt

  ┌─────────────────────────────────────────────┐
  │        TDE_KEYINFO heap record               │
  │  mk_index, mk_hash, created_time, set_time   │
  │  dk_perm (encrypted), dk_temp, dk_log        │
  └─────────────────────────────────────────────┘
```

The master key is only in memory transiently (during `tde_cipher_initialize`). After loading, only the decrypted data keys persist in `tde_Cipher`. The master key is wiped from the stack when `tde_cipher_initialize` returns (stack variable).

### 7.2 Server Startup Flow (Restart)

```
boot_restart_server()
  └─ tde_cipher_initialize(thread_p, &boot_Db_parm->tde_keyinfo_hfid, mk_path_given)
       ├─ fileio_mount(_keys file)
       ├─ tde_validate_keys_file()
       ├─ tde_get_keyinfo()          ← reads TDE_KEYINFO from heap
       ├─ tde_load_mk()              ← reads MK from _keys file, validates hash+time
       ├─ tde_load_dks()             ← decrypts perm/temp/log keys into tde_Cipher
       ├─ tde_Cipher.temp_write_counter = 0
       ├─ tde_Cipher.is_loaded = true
       └─ fileio_dismount()
```

### 7.3 First-Time Database Creation Flow

```
xboot_initialize_server() → boot_db_create()
  ├─ xheap_create() → boot_Db_parm->tde_keyinfo_hfid
  └─ tde_initialize(thread_p, &boot_Db_parm->tde_keyinfo_hfid)
       ├─ tde_create_keys_file()     ← creates <db>_keys with magic
       ├─ fileio_mount()
       ├─ tde_create_mk()            ← RAND_bytes → default MK
       ├─ tde_add_mk()               ← writes MK to slot 0, fsync
       ├─ tde_create_dk() × 3       ← RAND_bytes → perm, temp, log DKs
       ├─ tde_generate_keyinfo()     ← hash MK, encrypt DKs under MK
       ├─ heap_insert_logical()      ← store TDE_KEYINFO record
       ├─ save tde_Keyinfo_oid/hfid
       └─ fileio_dismount()
```

### 7.4 Data Page Encryption (Write Path)

```
pgbuf_write_to_disk_or_dwb()
  ├─ pgbuf_get_tde_algorithm(pgptr)   ← reads pflag from FILEIO_PAGE_RESERVED
  ├─ if tde_algo != NONE:
  │    tde_encrypt_data_page(iopage, tde_algo, is_temp, out_iopage)
  │      ├─ if is_temp: nonce = ATOMIC_INC_64(temp_write_counter)
  │      │              key   = tde_Cipher.data_keys.temp_key
  │      └─ else:       nonce = iopage->prv.lsa
  │                     key   = tde_Cipher.data_keys.perm_key
  │      ├─ memcpy header (FILEIO_PAGE_RESERVED) → cipher page
  │      ├─ memcpy watermark (FILEIO_PAGE_WATERMARK) → cipher page
  │      ├─ store nonce in cipher->prv.tde_nonce
  │      └─ tde_encrypt_internal(body, DB_PAGESIZE, algo, key, nonce, out)
  └─ else: memcpy plain iopage
```

### 7.5 Data Page Decryption (Read Path)

```
pgbuf_fix() → pgbuf_load_page()
  ├─ fileio_read() → BCB iopage_buffer
  ├─ pgbuf_get_tde_algorithm(pgptr)
  ├─ if tde_algo != NONE:
  │    tde_decrypt_data_page(iopage, tde_algo, is_temp, iopage)  ← in-place
  │      ├─ key   = temp_key or perm_key (based on is_temp)
  │      ├─ nonce = iopage_cipher->prv.tde_nonce
  │      ├─ memcpy header and watermark unchanged
  │      └─ tde_decrypt_internal(body, DB_PAGESIZE, algo, key, nonce, out)
  └─ page is now plaintext in BCB
```

Note: the decryption modifies `iopage` in-place (source and destination are the same buffer), which is valid for CTR mode since it is a stream cipher and output does not depend on previously written output bytes.

### 7.6 Log Page Encryption

```
log_writer.c: logwr_write_log_pages()
  ├─ for each log page to ship:
  │    if LOG_IS_PAGE_TDE_ENCRYPTED(log_pgptr):
  │      tde_encrypt_log_page(log_pgptr, tde_algo, buf_pgptr)
  │        ├─ nonce = log_pgptr->hdr.logical_pageid (8 bytes)
  │        ├─ key   = tde_Cipher.data_keys.log_key
  │        ├─ memcpy LOG_HDRPAGE header unchanged
  │        └─ tde_encrypt_internal(body, LOG_PAGESIZE - hdr, algo, key, nonce, out)
  └─ ship encrypted page to standby

log_applier.c: apply log records
  ├─ if LOG_IS_PAGE_TDE_ENCRYPTED(logpage):
  │    tde_decrypt_log_page(logpage, tde_algo, logpage)  ← in-place
  │      └─ (symmetric: recover nonce from hdr.logical_pageid)
  └─ process plaintext log record
```

### 7.7 Key Rotation Flow

```
DBA runs: tde -changeMasterKey -index N
  ├─ client: locks _keys file
  ├─ client: sends RPC → stde_change_mk_on_server
  │    └─ server: xtde_change_mk_without_flock(thread_p, mk_index)
  │         ├─ open _keys with O_RDONLY (no lock — client holds it)
  │         ├─ tde_find_mk(mk_index)           ← read new MK from file
  │         ├─ tde_get_keyinfo()               ← read current keyinfo
  │         ├─ check: is this MK already set? → skip if yes
  │         ├─ tde_find_mk(keyinfo.mk_index)   ← verify old MK still in file
  │         ├─ tde_change_mk()
  │         │    ├─ tde_generate_keyinfo(new MK, existing DKs)  ← re-wrap DKs
  │         │    ├─ tde_update_keyinfo()        ← write to heap
  │         │    └─ heap_flush()               ← force to disk
  │         └─ close file
  └─ client: can now delete old MK from file with tde -deleteMasterKey
```

**Critical safety property:** The data keys (perm, temp, log) are never changed during key rotation. Only the key-wrapping layer changes. All encrypted data pages and log pages remain readable after rotation because the same data keys are used, just wrapped under a new master key.

---

## 8. Concurrency & Thread Safety

### `tde_Cipher` Thread Safety

The global `tde_Cipher` structure is subject to concurrent access from multiple worker threads:

- **`is_loaded`** and **`data_keys`**: Written once during server startup (`tde_cipher_initialize`) and only read thereafter. No synchronization needed at runtime.
- **`temp_write_counter`**: Incremented atomically via `ATOMIC_INC_64` in `tde_encrypt_data_page` for temporary pages. This is the only runtime-modified field. The 64-bit atomic ensures each temp page gets a unique nonce even under high concurrency.

### Permanent Page Nonce Safety

For permanent pages, the nonce is the page LSA. CUBRID's WAL protocol guarantees that a dirty page's LSA advances monotonically; a page is never written to disk with the same LSA twice (once written, the next modification creates a new LSA). Therefore the (key, nonce) pair for permanent pages is unique across writes.

### Key File Access

All key file operations (`tde_add_mk`, `tde_find_mk`, `tde_delete_mk`, etc.) mask signals via `off_signals`/`restore_signals` to prevent interrupted writes. There is no mutex protecting the file itself — the expectation is that concurrent access is serialized by the caller. The `xtde_change_mk_without_flock` comment confirms that the client-side `tde` utility holds a file lock.

### Heap Record Access

`tde_update_keyinfo` uses `heap_scancache_start_modify` with `SINGLE_ROW_UPDATE`, which acquires appropriate heap-level locks. The `UPDATE_INPLACE_CURRENT_MVCCID` flag means the update does not create a new MVCC version, reducing overhead since the keyinfo record is logically a single-version configuration record.

---

## 9. Memory Management

All memory usage in this module follows CUBRID's C-style patterns:

- **Stack allocation:** All key material (master key, data key, nonces) is allocated on the stack as fixed-size `unsigned char[N]` arrays. No heap allocation for key bytes.
- **OpenSSL context:** `EVP_CIPHER_CTX *ctx` is allocated via `EVP_CIPHER_CTX_new()` and freed via `EVP_CIPHER_CTX_free(ctx)` in the `cleanup` label. There is no memory leak even on error paths.
- **Heap records:** `recdes_buffer` for keyinfo read/write is stack-allocated as `char[sizeof(int) + sizeof(TDE_KEYINFO)]`.
- **File I/O buffer:** `tde_copy_keys_file` uses a stack-allocated 4096-byte buffer.
- No `db_private_alloc`, `malloc`, or `free` calls anywhere in this module.
- No `free_and_init` calls needed since no heap allocation is performed.

**Key material zeroing:** The master key variable in `tde_cipher_initialize` is a stack `unsigned char[32]` that is reused but not explicitly zeroed after use. This is a potential minor security concern — the master key bytes remain in stack memory until overwritten by the next function call. In practice, the stack is overwritten quickly, but a hardened implementation would use `OPENSSL_cleanse` or `memset_explicit`.

---

## 10. Error Handling

### Error Code Catalog

| Error Code | Value | Trigger |
|-----------|-------|---------|
| `ER_TDE_INVALID_KEYS_FILE` | -1248 | Magic bytes mismatch, partial read in key file |
| `ER_TDE_MASTER_KEY_NOT_FOUND` | -1249 | Seek/read failure or soft-deleted slot |
| `ER_TDE_INVALID_MASTER_KEY` | -1250 | Hash mismatch or timestamp mismatch |
| `ER_TDE_ENCRYPTION_ERROR` | -1251 | OpenSSL EVP encrypt failure |
| `ER_TDE_DECRYPTION_ERROR` | -1252 | OpenSSL EVP decrypt failure |
| `ER_TDE_CIPHER_IS_NOT_LOADED` | -1253 | `tde_is_loaded()` returns false |
| `ER_TDE_KEY_CREATION_FAIL` | -1254 | `RAND_bytes` returns != 1 |
| `ER_TDE_CIPHER_LOAD_FAIL` | -1255 | Defined but set externally (boot_sr.c) |
| `ER_TDE_COPY_KEYS_FILE_FAIL` | -1256 | Defined externally |
| `ER_TDE_BACKUP_KEYS_FILE_FAIL` | -1257 | Defined externally |
| `ER_TDE_RESTORE_KEY_FOUND_ONLY_FROM_BACKUP` | -1258 | Restore path |
| `ER_TDE_RESTORE_MAKE_KEYS_FILE_OLD` | -1259 | Restore path |
| `ER_TDE_RESTORE_COPY_KEYS_FILE` | -1260 | Restore path |
| `ER_TDE_RESTORE_CHANGE_MASTER_KEY` | -1261 | Restore path |
| `ER_TDE_MAX_KEY_FILE` | -1262 | 128-slot limit reached |
| `ER_TDE_ENCRYPTION_LOGPAGE_ERORR_AND_OFF_TDE` | -1263 | Log page encryption failure |

### Error Propagation Patterns

1. **`goto exit` pattern:** Used in functions with cleanup requirements (file descriptor dismount). The exit label always runs cleanup regardless of error.
2. **Early return pattern:** Used in simpler functions without cleanup resources.
3. **`ASSERT_ERROR()`:** Used when an error is expected to already be set in the error manager (e.g., after `fileio_mount` returns `NULL_VOLDES`).
4. **`er_set_with_oserror`:** Used for file I/O errors to include the OS `errno` in the message.

### Diagnostic Logging

```c
#define tde_er_log(...) \
  if (prm_get_bool_value(PRM_ID_ER_LOG_TDE)) \
    _er_log_debug(ARG_FILE_LINE, "TDE: " __VA_ARGS__)
```

Controlled by `PRM_ID_ER_LOG_TDE` system parameter. Used in `page_buffer.c` (`pgbuf_set_tde_algorithm`) for per-page algorithm change events.

---

## 11. Integration Points

### 11.1 `page_buffer.c` Integration

The page buffer manager integrates TDE at two points:

**Read path** (`pgbuf_fix` / `pgbuf_load_page`, line ~8277):
```c
tde_algo = pgbuf_get_tde_algorithm(pgptr);
if (tde_algo != TDE_ALGORITHM_NONE) {
    tde_decrypt_data_page(&bufptr->iopage_buffer->iopage, tde_algo,
                          pgbuf_is_temporary_volume(vpid->volid),
                          &bufptr->iopage_buffer->iopage);  // in-place decrypt
}
```
Pages arrive from disk encrypted; they are decrypted in-place into the BCB before being returned to callers. All subsequent access through `pgbuf_fix` sees plaintext.

**Write path** (`pgbuf_write_to_disk_or_dwb`, line ~10532):
```c
tde_algo = pgbuf_get_tde_algorithm(pgptr);
if (tde_algo != TDE_ALGORITHM_NONE) {
    error = tde_encrypt_data_page(&bufptr->iopage_buffer->iopage, tde_algo, is_temp, iopage);
} else {
    memcpy(iopage, &bufptr->iopage_buffer->iopage, IO_PAGESIZE);
}
```
Pages are encrypted into a temporary `iopage` buffer before being passed to `fileio_write` or DWB. The in-memory BCB page remains unencrypted.

**Algorithm storage** (page flags, `file_io.h:63-66`):
```c
#define FILEIO_PAGE_FLAG_ENCRYPTED_AES  0x1
#define FILEIO_PAGE_FLAG_ENCRYPTED_ARIA 0x2
#define FILEIO_PAGE_FLAG_ENCRYPTED_MASK 0x3
// in FILEIO_PAGE_RESERVED.pflag (1-byte bitfield)
```

`pgbuf_get_tde_algorithm` reads `pflag` and returns the `TDE_ALGORITHM` enum. `pgbuf_set_tde_algorithm` writes `pflag` and logs the change as a redo-only log record (`RVPGBUF_SET_TDE_ALGORITHM`) for recovery.

The nonce is stored in `FILEIO_PAGE_RESERVED.tde_nonce` (INT64, offset `~175` in `storage_common.h`). For permanent pages this duplicates the LSA; for temp pages it holds the atomic counter value.

### 11.2 `file_io.c` Integration

`tde.c` uses `file_io.c` functions for:
- `fileio_mount` / `fileio_dismount` — volume management for the `_keys` file
- `fileio_open` / `fileio_close` — direct POSIX-level file access without volume table
- `fileio_is_volume_exist` — existence check before creation
- `fileio_unformat_and_rename` — safe deletion during `tde_copy_keys_file`
- `fileio_make_keys_name` / `fileio_make_keys_name_given_path` — path construction
- `fileio_get_base_file_name` — basename extraction
- `fileio_get_volume_label_by_fd` — for error messages in `tde_add_mk`

The `_keys` file is treated as a special volume with `LOG_DBTDE_KEYS_VOLID`, keeping it in the volume tracking table. This allows it to benefit from volume-level error handling and abstraction.

### 11.3 Log Manager Integration

**Log writer** (`log_writer.c`):
- `logwr_set_tde_algorithm` sets the TDE algorithm flag in a log page header.
- `LOG_IS_PAGE_TDE_ENCRYPTED(log_pgptr)` checks the header flag before encrypting.
- `tde_encrypt_log_page` is called when shipping pages to a standby replica.
- `tde_Cipher.data_keys.log_key` is directly accessed to share with HA replica (via `stde_get_data_keys` network RPC).

**Log applier** (`log_applier.c`):
- Checks `LOG_IS_PAGE_TDE_ENCRYPTED` before calling `tde_decrypt_log_page`.
- Supports in-place decryption.
- The HA socket path `TDE_HA_SOCK_NAME = ".ha_sock"` is used for out-of-band data key distribution to `copylogdb`.

**Note:** `UNSTABLE_TDE_FOR_REPLICATION_LOG` is not defined in the current codebase, so TDE for replication log is present but marked unstable. The `LOG_MAY_CONTAIN_USER_DATA(rcvindex)` macro (tde.h:107) identifies which redo log record types contain user data that must be encrypted — used externally to determine whether a log record should be encrypted before shipping.

### 11.4 Boot Manager Integration (`boot_sr.c`)

| Line | Call | Context |
|------|------|---------|
| 4976 | `xheap_create(&boot_Db_parm->tde_keyinfo_hfid, ...)` | Create keyinfo heap at DB creation |
| 5104 | `tde_initialize(thread_p, &boot_Db_parm->tde_keyinfo_hfid)` | First-time init |
| 2324 | `tde_cipher_initialize(thread_p, &..., mk_path)` | Restart: load cipher |
| 5233 | `tde_cipher_initialize(thread_p, &..., NULL)` | Standby restart |
| 2869 | `if (tde_is_loaded()) { ... }` | Conditional logic during restart |
| 2913 | `assert(tde_is_loaded())` | Assertion after cipher init |

The `boot_Db_parm->tde_keyinfo_hfid` (defined in `boot_sr.h:135`) is the persistent link between the boot parameters structure (stored in the database) and the TDE keyinfo heap file. It is loaded from disk on every restart.

### 11.5 Network Interface Integration (`network_interface_sr.cpp`)

Five server-side RPC handlers wrap TDE operations for the `tde` admin utility:

| Handler | Function | Description |
|---------|----------|-------------|
| `stde_get_data_keys` | Direct `tde_Cipher` access | Pack perm/temp/log keys for HA |
| `stde_is_loaded` | `tde_is_loaded()` | Check cipher state |
| `stde_get_mk_file_path` | `tde_make_keys_file_fullname` | Return key file path |
| `stde_get_mk_info` | `xtde_get_mk_info` | Return current MK metadata |
| `stde_change_mk_on_server` | `xtde_change_mk_without_flock` | Execute key rotation |
| `sfile_apply_tde_to_class_files` | `xfile_apply_tde_to_class_files` | Apply TDE to existing class files |

---

## 12. Complexity & Metrics

### File Statistics

| Metric | Value |
|--------|-------|
| Total lines | 1,720 (tde.c) + 214 (tde.h) |
| Blank lines | ~200 |
| Comment lines | ~300 |
| Code lines | ~1,220 |
| Number of functions | 27 (tde.c: 20 public/static + 2 xtde = 22; tde.h declares 14 extern) |
| Static functions | 9 |
| Public (extern) functions | 18 |
| External dependencies | OpenSSL EVP (AES/ARIA CTR), OpenSSL SHA256, OpenSSL RAND |

### Cyclomatic Complexity Estimates

| Function | Branches | Complexity (approx.) |
|----------|----------|---------------------|
| `tde_initialize` | 6 | Medium |
| `tde_cipher_initialize` | 5 | Medium |
| `tde_add_mk` | 7 (loop + conditionals) | Medium-High |
| `tde_find_first_mk` | 5 (loop) | Medium |
| `tde_dump_mks` | 4 (loop) | Low-Medium |
| `xtde_change_mk_without_flock` | 5 | Medium |
| `tde_encrypt_internal` | 4 | Low-Medium |
| `tde_copy_keys_file` | 6 | Medium |
| All others | 1-3 | Low |

### Lines per Function (approximate)

| Function | Lines |
|----------|-------|
| `tde_initialize` | 115 |
| `tde_cipher_initialize` | 70 |
| `tde_add_mk` | 76 |
| `tde_find_first_mk` | 57 |
| `tde_copy_keys_file` | 85 |
| `tde_dump_mks` | 57 |
| `tde_encrypt_internal` | 65 |
| `tde_decrypt_internal` | 63 |
| `tde_update_keyinfo` | 43 |
| `xtde_change_mk_without_flock` | 54 |
| All others | < 40 |

---

## 13. Notable Patterns & Idioms

### Signal Masking for Atomic File Writes

All key file modifications mask all signals except SIGINT, SIGQUIT, SIGTERM, SIGHUP, SIGABRT:
```c
#define off_signals(new_mask, old_mask) do { sigfillset(&(new_mask)); sigdelset(&(new_mask), SIGINT); ... } while(0)
```
This ensures that a SIGPIPE or other signal cannot interrupt a partial `write` + `fsync` sequence that writes key material to disk, preventing a corrupted `_keys` file.

### Two-Label Error Cleanup (`cleanup` + `exit`)

`tde_encrypt_internal` and `tde_decrypt_internal` use two goto labels:
```c
cleanup:
  EVP_CIPHER_CTX_free(ctx);
exit:
  if (err != NO_ERROR) { er_set(...); }
  return err;
```
`cleanup` is jumped to on OpenSSL failure (ctx must be freed); `exit` is jumped to if ctx allocation fails (ctx is NULL, so `EVP_CIPHER_CTX_free` cannot be called). This two-stage pattern correctly handles both failure modes.

### Repid Hack for Vacuum Safety

The 4-byte zero prefix on the keyinfo heap record:
```c
int repid_and_flag_bits = 0;
memcpy(recdes_buffer, &repid_and_flag_bits, sizeof(int));
memcpy(recdes_buffer + sizeof(int), keyinfo, sizeof(TDE_KEYINFO));
```
Prevents `vacuum_rv_check_at_undo()` from misinterpreting the record's representation ID during undo, since the keyinfo record has no associated class schema.

### Class OID Hack for Heap Scan Cache

```c
scan_cache.node.class_oid = *oid_Root_class_oid;
```
The keyinfo OID has no real class (it predates normal schema objects). By faking the class OID as root class, `heap_scancache_check_with_hfid` is bypassed while still allowing `heap_update_logical` to look up the correct heap file type from its cache.

### CTR Mode: Encrypt == Decrypt

AES-256-CTR and ARIA-256-CTR are stream ciphers where encryption and decryption are the same operation (XOR with keystream). The codebase exploits this implicitly in the in-place decrypt calls:
```c
tde_decrypt_data_page(&bufptr->iopage_buffer->iopage, ..., &bufptr->iopage_buffer->iopage);  // src == dst
```
Since CTR mode is a stream cipher, in-place operation is safe.

### Nonce Strategy Summary

| Context | Nonce source | Uniqueness guarantee |
|---------|-------------|---------------------|
| Permanent data pages | `iopage->prv.lsa` (8 bytes, zero-padded to 16) | LSA advances monotonically per WAL |
| Temporary data pages | `ATOMIC_INC_64(temp_write_counter)` (8 bytes) | Reset to 0 at restart; counter is server-lifetime unique |
| Log pages | `logpage->hdr.logical_pageid` (8 bytes) | Log pages have globally unique sequential IDs |
| Data key encryption | `memset(nonce, 0/1/2, 16)` per key type | Different nonces per key type; each (MK, nonce) pair used once per key rotation |

### Soft Deletion in Key File

Rather than compacting the key file, deleted keys are marked by setting `created_time = -1`. This preserves slot indices (which are stored in `TDE_KEYINFO.mk_index`) and avoids the complexity of renumbering. The maximum 128-slot limit means the file can grow to at most `CUBRID_MAGIC_MAX_LENGTH + 128 * sizeof(TDE_MK_FILE_ITEM)` bytes.

### `TDE_DK_ALGORITHM` Constant

Data keys are always encrypted under the master key using AES-256-CTR regardless of the database's configured `TDE_ALGORITHM`. This means ARIA-encrypted databases still use AES to wrap the data keys. The rationale is likely that key wrapping requires a fixed, well-tested algorithm independent of the per-database choice.

### Deterministic Nonces for Data Key Wrapping

Data keys are encrypted with deterministic nonces (`0x00`, `0x01`, `0x02` for PERM, TEMP, LOG). This is cryptographically safe because:
1. The master key changes with each call to `tde_change_mk`.
2. Each (master_key, nonce) pair is used at most once in a system lifetime.
3. The three different nonces prevent related-key attacks even if two data keys are identical bytes.

### No IV/Tag: CTR Mode Without Authentication

The design uses CTR mode without authentication (no GCM/CCM/Poly1305). This provides confidentiality but not integrity/authenticity. An attacker with write access to the data files could flip bits in encrypted pages without detection (bit-flipping attack). This is a known trade-off; the system relies on OS-level file permissions and physical security. CTR mode is also faster than authenticated encryption modes, which matters for database I/O throughput.

---

*End of report. Generated from full source read of `tde.c` (1,720 lines) and `tde.h` (214 lines), with cross-reference analysis of `page_buffer.c`, `boot_sr.c`, `log_writer.c`, `log_applier.c`, `network_interface_sr.cpp`, `file_io.h`, `storage_common.h`, `log_storage.hpp`, and `error_code.h`.*
