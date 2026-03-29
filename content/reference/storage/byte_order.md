# Analysis Report: `src/storage/byte_order.c` and `src/storage/byte_order.h`

**Generated:** 2026-03-27
**Codebase:** CUBRID Database Engine — Apache 2.0

---

## 1. File Overview

| Attribute        | `.c` file                                         | `.h` file                                         |
|------------------|---------------------------------------------------|---------------------------------------------------|
| **Path**         | `src/storage/byte_order.c`                        | `src/storage/byte_order.h`                        |
| **Lines**        | 199                                               | 187                                               |
| **Language**     | C (compiled as C++17 via `c_to_cpp.sh`)           | C/C++ header                                      |
| **Purpose**      | Fallback implementations of byte-order conversion routines for little-endian targets lacking OS-provided equivalents | Endianness detection macros, type definitions, macro/function declarations for byte-order conversion |
| **Module**       | `src/storage/` — disk storage utilities layer     |                                                   |
| **Build modes**  | All three: `SERVER_MODE`, `SA_MODE`, `CS_MODE`    | Same — shared header, no mode guards              |
| **Guard**        | N/A (`.c`)                                        | `#ifndef _BYTE_ORDER_H_` / `#define _BYTE_ORDER_H_` |

### Purpose Summary

`byte_order` provides a portable, self-contained layer for converting numeric types (short, int, 64-bit integer, float, double) between host byte order and network byte order (big-endian). It mirrors the POSIX `htons`/`ntohs`/`htonl`/`ntohl` API and extends it with CUBRID-specific 64-bit integer and floating-point variants (`ntohi64`, `ntohd`, `ntohf`, `htoni64`, `htond`, `htonf`). The header also defines the `MOVING_VAN` union for safe double-precision copy on IA-64, and the `swap64` macro for 64-bit integer byte reversal used by serialization macros.

---

## 2. Includes & Dependencies

### `byte_order.h` — Direct Includes

| Include                   | Platform Guard               | Provides                               |
|---------------------------|------------------------------|----------------------------------------|
| `"system.h"`              | None (always)                | `UINT32`, `UINT64`, `INT64`, etc. via `<stdint.h>`/`<inttypes.h>`; portable type definitions |
| `<arpa/inet.h>`           | `defined(LINUX)`             | `htons`, `ntohs`, `htonl`, `ntohl` (OS implementations) |
| `<sys/types.h>`           | `defined(sun)`               | BSD/Solaris type definitions           |
| `<netinet/in.h>`          | `defined(HPUX)`              | HP-UX network byte order functions     |
| `<net/nh.h>`              | `defined(_AIX)`              | AIX network byte order functions       |
| `<winsock2.h>`            | `defined(WINDOWS)`           | Windows socket API including `htons`/`htonl` |

### `byte_order.c` — Direct Includes

| Include                   | Purpose                                            |
|---------------------------|----------------------------------------------------|
| `"byte_order.h"`          | Own header — pulls in all types and declarations   |
| `"memory_wrapper.hpp"`    | MUST be last (CUBRID project rule); custom allocator tracking wrapper |

### Reverse Dependencies (Files That Include `byte_order.h`)

| File                                      | How used                                                |
|-------------------------------------------|---------------------------------------------------------|
| `src/base/object_representation.h`        | Primary consumer — `OR_PUT_*` / `OR_GET_*` macros for all packed disk/network values call `htonf`, `htond`, `ntohf`, `ntohd`, `swap64`, `OR_MOVE_DOUBLE` |
| `src/sp/pl_struct_compile.cpp`            | Stored procedure struct compilation — transitively via `object_representation.h` |
| `src/query/numeric_opfunc.c`              | Numeric operations — transitively via `object_representation.h`; `OR_BYTE_ORDER` comparison |
| `src/base/tz_compile.c`                   | Timezone compilation — transitively includes `byte_order.h` |
| `src/broker/cas_net_buf.c`               | Direct use: calls `ntohi64` for BIGINT deserialization in CAS network buffer |

### Cross-Module Consumers (Transitive)

Any file that includes `object_representation.h` indirectly depends on `byte_order.h`. Key examples:

- `src/query/query_executor.c` — entire query execution pipeline
- `src/storage/btree.c` — B-tree index scans
- `src/transaction/log_*.c` — WAL logging
- `src/object/object_primitive.c` — DB_VALUE packing/unpacking via `OR_MOVE_DOUBLE`
- `src/compat/db_macro.c` — `OR_MOVE_DOUBLE` for `DB_MONETARY`
- `src/compat/db_value_printer.cpp` — `OR_MOVE_DOUBLE` for monetary display
- `src/storage/extendible_hash.c` — `OR_MOVE_DOUBLE` for double key comparison in hash index

---

## 3. Preprocessor & Compilation

### Endianness Detection Macros

```c
#define OR_LITTLE_ENDIAN 1234
#define OR_BIG_ENDIAN    4321
```

These are the canonical endianness sentinel values used throughout the CUBRID codebase (mirroring the `<endian.h>` convention of `__LITTLE_ENDIAN = 1234`, `__BIG_ENDIAN = 4321`).

```c
#if defined(HPUX) || defined(_AIX) || defined(sparc)
#define OR_BYTE_ORDER OR_BIG_ENDIAN
#else
#define OR_BYTE_ORDER OR_LITTLE_ENDIAN  /* WINDOWS, LINUX, x86_SOLARIS */
#endif
```

Platform classification is **static at compile time** — no runtime detection. The `sparc` identifier covers legacy SPARC/Solaris targets. All x86/x86-64 Linux and Windows targets take the little-endian path.

### OS-Provided Function Availability Macros

```c
#define OR_HAVE_NTOHS   /* defined when OS provides ntohs() */
#define OR_HAVE_NTOHL   /* defined when OS provides ntohl() */
#define OR_HAVE_HTONS   /* defined when OS provides htons() */
#define OR_HAVE_HTONL   /* defined when OS provides htonl() */
#define OR_HAVE_NTOHF   /* defined on MSVC >= 1700 (VS 2012+) */
#define OR_HAVE_NTOHD   /* defined on MSVC >= 1700 */
#define OR_HAVE_HTONF   /* defined on MSVC >= 1700 */
#define OR_HAVE_HTOND   /* defined on MSVC >= 1700 */
```

These are defined for: HPUX, AIX, Windows, Linux (`OR_HAVE_NTOHS/L/HTONS/L`). Windows MSVC >= 1700 additionally defines the float/double variants. Each guard prevents duplicate definitions of the fallback functions in `byte_order.c`.

### Compilation Path for Little-Endian (e.g., Linux x86-64)

- `OR_BYTE_ORDER == OR_LITTLE_ENDIAN` is true.
- `OR_HAVE_NTOHS`, `OR_HAVE_NTOHL`, `OR_HAVE_HTONS`, `OR_HAVE_HTONL` are defined — those 4 functions are provided by `<arpa/inet.h>`.
- `OR_HAVE_NTOHF`, `OR_HAVE_NTOHD`, `OR_HAVE_HTONF`, `OR_HAVE_HTOND` are **not** defined on Linux — so `ntohf`, `ntohd`, `htonf`, `htond` are compiled from `byte_order.c`.
- `ntohi64` and `htoni64` have no OS-provided variants anywhere; they are always compiled from `byte_order.c` (on little-endian).

### Compilation Path for Big-Endian (e.g., HPUX, AIX, SPARC)

- All conversion functions become identity macros (no-ops) since network order == host order:
  ```c
  #define ntohs(x)      (x)
  #define ntohl(x)      (x)
  #define ntohi64(x)    (x)
  #define htons(x)      (x)
  #define htonl(x)      (x)
  #define htoni64(x)    (x)
  ```
- Float/double conversions on big-endian use pointer semantics:
  ```c
  #define ntohf(ptr, value)   (*(value) = *(ptr))
  #define ntohd(ptr, value)   OR_MOVE_DOUBLE(ptr, value)
  #define htonf(ptr, value)   (*(ptr) = *(value))
  #define htond(ptr, value)   OR_MOVE_DOUBLE(value, ptr)
  ```

### IA-64 Alignment Guard

```c
#if !defined(IA64)
#define OR_MOVE_DOUBLE(src, dst) \
  (((MOVING_VAN *)(dst))->bits = ((MOVING_VAN *)(src))->bits)
#else
#define OR_MOVE_DOUBLE(src, dst) \
  memcpy(((MOVING_VAN*)(dst))->bits.buf, ((MOVING_VAN *)(src))->bits.buf, sizeof(MOVING_VAN))
#endif
```

On IA-64 (Itanium), strict alignment rules require `memcpy` instead of a direct struct-copy assignment. Non-IA-64 uses the faster assignment form.

---

## 4. Data Structures & Types

### `MOVING_VAN` union (`byte_order.h`, lines 76–84)

```c
typedef union moving_van MOVING_VAN;
union moving_van
{
  struct
  {
    unsigned int buf[2];   /* Two 32-bit words covering 8 bytes */
  } bits;
  double dbl;              /* Reinterprets the same 8 bytes as a double */
};
```

**Purpose:** Provides a portable, standards-safe way to copy or move `double` values as raw bits without invoking undefined behavior from type-punning via direct pointer casts. On most architectures `double` is 8 bytes, exactly matching `unsigned int buf[2]`.

| Field       | Type                | Size   | Purpose                                       |
|-------------|---------------------|--------|-----------------------------------------------|
| `bits`      | `struct { unsigned int buf[2]; }` | 8 bytes | Raw bit representation as two 32-bit integers |
| `bits.buf`  | `unsigned int[2]`   | 8 bytes | Array form for `memcpy` on IA-64              |
| `dbl`       | `double`            | 8 bytes | Overlay — interprets same bits as IEEE 754 double |

**Usage:** `OR_MOVE_DOUBLE(src, dst)` casts both pointers to `MOVING_VAN*` and assigns the `bits` sub-struct, avoiding aliasing issues.

### Integer Types

`UINT32` and `UINT64` come from `system.h` → `<stdint.h>`:

| Alias    | Underlying type        | Size    | Role in this module                  |
|----------|------------------------|---------|--------------------------------------|
| `UINT32` | `uint32_t`             | 4 bytes | Bit-pattern carrier for `float` conversions (`htonf`/`ntohf`) |
| `UINT64` | `uint64_t`             | 8 bytes | Bit-pattern carrier for `double` and 64-bit integer conversions |
| `INT64`  | `int64_t`              | 8 bytes | Used in broker (`ntohi64` caller uses `DB_BIGINT`) |

---

## 5. Global & Static Variables

**None.** The module is entirely stateless. All functions are pure transformations with no global or static variables.

---

## 6. Function Catalog

This section documents every function defined in `byte_order.c` and every macro/function declared in `byte_order.h`.

---

### 6.1 `ntohs` — Network-to-Host Short

**Guard:** `#if OR_BYTE_ORDER == OR_LITTLE_ENDIAN && !defined(OR_HAVE_NTOHS)`

```c
unsigned short ntohs(unsigned short from);
```

**Defined at:** `byte_order.c:42–54`

**Description:** Converts a 16-bit unsigned integer from network byte order (big-endian) to host byte order (little-endian). Reverses bytes at positions 0 and 1.

**Algorithm:**
```
to[0] = from[1]   /* high byte of network → low byte of host */
to[1] = from[0]   /* low byte of network  → high byte of host */
```
Uses `char*` pointer aliasing to access individual bytes. On Linux this function is provided by `<arpa/inet.h>` and this definition is compiled out (`OR_HAVE_NTOHS` is set).

**Callers:** `OR_GET_SHORT` macro in `object_representation.h`; OS-provided on Linux/Windows.

**Callees:** None.

---

### 6.2 `ntohl` — Network-to-Host Long

**Guard:** `#if OR_BYTE_ORDER == OR_LITTLE_ENDIAN && !defined(OR_HAVE_NTOHL)`

```c
unsigned int ntohl(unsigned int from);
```

**Defined at:** `byte_order.c:57–73`

**Description:** Converts a 32-bit unsigned integer from network byte order to host byte order. Full 4-byte reversal.

**Algorithm:**
```
to[0] = from[3]
to[1] = from[2]
to[2] = from[1]
to[3] = from[0]
```

**Callers:** `OR_GET_INT`, `OR_GET_TIME`, `OR_GET_DATE`, `OR_GET_UTIME` macros in `object_representation.h`; `cas_net_buf.c` (`ntohl` called directly for LOB type); OS-provided on Linux/Windows.

**Callees:** None.

---

### 6.3 `ntohf` — Network-to-Host Float

**Guard:** `#if OR_BYTE_ORDER == OR_LITTLE_ENDIAN && !defined(OR_HAVE_NTOHF)`

```c
float ntohf(UINT32 from);
```

**Defined at:** `byte_order.c:75–91`

**Description:** Converts a 32-bit raw bit pattern (in network/big-endian byte order) to a host-order `float`. The input is the raw IEEE 754 bits read from disk or wire as a `UINT32`. Reverses all 4 bytes and reinterprets as `float`.

**Algorithm:**
```
to[0] = from[3]
to[1] = from[2]
to[2] = from[1]
to[3] = from[0]
```
The `to` variable is declared as `float` — the byte reversal writes directly into its memory representation.

**Note:** Signature differs from big-endian macro form (`ntohf(ptr, value)`) — on little-endian it is a true function returning a `float`; on big-endian it is a macro that dereferences a pointer pair.

**Callers:**
- `OR_GET_FLOAT(ptr, value)` macro in `object_representation.h:148`:
  ```c
  #define OR_GET_FLOAT(ptr, value) \
    (*(value) = ntohf(*(UINT32 *)(ptr)))
  ```
  Used throughout the serialization layer when deserializing `DB_TYPE_FLOAT` values from disk pages or XASL streams.

**Callees:** None.

---

### 6.4 `ntohd` — Network-to-Host Double

**Guard:** `#if OR_BYTE_ORDER == OR_LITTLE_ENDIAN && !defined(OR_HAVE_NTOHD)`

```c
double ntohd(UINT64 from);
```

**Defined at:** `byte_order.c:93–113`

**Description:** Converts a 64-bit raw bit pattern (network byte order) to a host-order `double`. Full 8-byte reversal into a `double` target.

**Algorithm:**
```
to[0] = from[7]
to[1] = from[6]
to[2] = from[5]
to[3] = from[4]
to[4] = from[3]
to[5] = from[2]
to[6] = from[1]
to[7] = from[0]
```

**Callers:**
- `OR_GET_DOUBLE(ptr, value)` macro in `object_representation.h:157`:
  ```c
  #define OR_GET_DOUBLE(ptr, value) \
    (*(value) = ntohd(*(UINT64 *)(ptr)))
  ```
  Used for `DB_TYPE_DOUBLE` and `DB_TYPE_MONETARY` deserialization.

**Callees:** None.

---

### 6.5 `ntohi64` — Network-to-Host 64-bit Integer

**Guard:** Always compiled on little-endian (no `OR_HAVE_*` check — always custom).

```c
UINT64 ntohi64(UINT64 from);
```

**Defined at:** `byte_order.c:115–133`

**Description:** Converts a 64-bit unsigned integer from network byte order to host byte order. Identical byte-reversal algorithm to `ntohd` but for an integer carrier type. This function has **no OS-provided equivalent** and is always present in the little-endian build.

**Algorithm:** Full 8-byte reversal (same pattern as `ntohd`).

**Callers:**
- `cas_net_buf.c:458` — deserializing `DB_BIGINT` from CAS network protocol:
  ```c
  memcpy(&tmp_i, cur_p, NET_SIZE_BIGINT);
  *value = ntohi64(tmp_i);
  ```
- `cas_net_buf.c:723` — deserializing LOB size field:
  ```c
  lob->lob_size = ntohi64(tmp_i64);
  ```
- `byte_order.c:154` — `htoni64` delegates to this (since host-to-network == network-to-host for byte reversal).

**Callees:** None.

---

### 6.6 `htons` — Host-to-Network Short

**Guard:** `#if OR_BYTE_ORDER == OR_LITTLE_ENDIAN && !defined(OR_HAVE_HTONS)`

```c
unsigned short htons(unsigned short from);
```

**Defined at:** `byte_order.c:135–141`

**Description:** Converts a 16-bit unsigned integer from host byte order to network byte order. On little-endian the operation is identical to `ntohs` (byte reversal is its own inverse).

**Algorithm:** Delegates to `ntohs(from)`.

**Callers:** `OR_PUT_SHORT` macro in `object_representation.h`; OS-provided on Linux/Windows.

**Callees:** `ntohs`.

---

### 6.7 `htonl` — Host-to-Network Long

**Guard:** `#if OR_BYTE_ORDER == OR_LITTLE_ENDIAN && !defined(OR_HAVE_HTONL)`

```c
unsigned int htonl(unsigned int from);
```

**Defined at:** `byte_order.c:143–148`

**Description:** Converts a 32-bit unsigned integer from host byte order to network byte order. Delegates to `ntohl` since reversal is self-inverse.

**Algorithm:** Delegates to `ntohl(from)`.

**Callers:** `OR_PUT_INT`, `OR_PUT_SHORT`, `OR_PUT_TIME`, `OR_PUT_DATE` macros; OS-provided on Linux/Windows.

**Callees:** `ntohl`.

---

### 6.8 `htoni64` — Host-to-Network 64-bit Integer

**Guard:** Always compiled on little-endian.

```c
UINT64 htoni64(UINT64 from);
```

**Defined at:** `byte_order.c:151–155`

**Description:** Converts a 64-bit unsigned integer from host byte order to network byte order. Delegates to `ntohi64`. No OS-provided equivalent; always a CUBRID custom function.

**Algorithm:** Delegates to `ntohi64(from)`.

**Callers:** (On little-endian) `cas_net_buf.c` uses its own `net_htoni64` for the send path; `htoni64` is declared in `byte_order.h` for use in serialization contexts.

**Callees:** `ntohi64`.

---

### 6.9 `htonf` — Host-to-Network Float

**Guard:** `#if OR_BYTE_ORDER == OR_LITTLE_ENDIAN && !defined(OR_HAVE_HTONF)`

```c
UINT32 htonf(float from);
```

**Defined at:** `byte_order.c:157–174`

**Description:** Converts a host-order `float` into a 32-bit big-endian bit pattern (`UINT32`) suitable for writing to disk or the network. The inverse of `ntohf`. Returns an unsigned integer carrying the IEEE 754 bit pattern of the float in network byte order.

**Algorithm:**
```
q[0] = p[3]   /* host byte 3 (MSB) → network byte 0 */
q[1] = p[2]
q[2] = p[1]
q[3] = p[0]   /* host byte 0 (LSB) → network byte 3 */
```

**Callers:**
- `OR_PUT_FLOAT` inline function in `object_representation.h:141–145`:
  ```c
  STATIC_INLINE void OR_PUT_FLOAT(char *ptr, float val) {
    UINT32 ui = htonf(val);
    memcpy(ptr, &ui, sizeof(ui));
  }
  ```
  Used for all `DB_TYPE_FLOAT` serialization to disk pages and XASL streams.

**Callees:** None.

---

### 6.10 `htond` — Host-to-Network Double

**Guard:** `#if OR_BYTE_ORDER == OR_LITTLE_ENDIAN && !defined(OR_HAVE_HTOND)`

```c
UINT64 htond(double from);
```

**Defined at:** `byte_order.c:176–197`

**Description:** Converts a host-order `double` into a 64-bit big-endian bit pattern (`UINT64`) for writing to disk or network. The inverse of `ntohd`. Returns an unsigned 64-bit integer carrying the IEEE 754 double bits in network byte order.

**Algorithm:**
```
q[0] = p[7]   /* host byte 7 (MSB) → network byte 0 */
q[1] = p[6]
q[2] = p[5]
q[3] = p[4]
q[4] = p[3]
q[5] = p[2]
q[6] = p[1]
q[7] = p[0]   /* host byte 0 (LSB) → network byte 7 */
```

**Callers:**
- `OR_PUT_DOUBLE` inline function in `object_representation.h:150–155`:
  ```c
  STATIC_INLINE void OR_PUT_DOUBLE(char *ptr, double val) {
    UINT64 ui = htond(val);
    memcpy(ptr, &ui, sizeof(ui));
  }
  ```
  Used for all `DB_TYPE_DOUBLE` and `DB_TYPE_MONETARY` serialization.

**Callees:** None.

---

### 6.11 `swap64` — 64-bit Byte Swap Macro (Header Only)

**Guard:** Only expands to the swap expression on little-endian; identity on big-endian.

```c
#define swap64(x)  \
  ((((unsigned long long)(x) & 0x00000000000000FFULL) << 56) \
 | (((unsigned long long)(x) & 0xFF00000000000000ULL) >> 56) \
 | (((unsigned long long)(x) & 0x000000000000FF00ULL) << 40) \
 | (((unsigned long long)(x) & 0x00FF000000000000ULL) >> 40) \
 | (((unsigned long long)(x) & 0x0000000000FF0000ULL) << 24) \
 | (((unsigned long long)(x) & 0x0000FF0000000000ULL) >> 24) \
 | (((unsigned long long)(x) & 0x00000000FF000000ULL) << 8)  \
 | (((unsigned long long)(x) & 0x000000FF00000000ULL) >> 8))
```

**Defined at:** `byte_order.h:96–107`

**Description:** Pure expression macro for byte-reversing a 64-bit integer value. Takes a value (not a pointer). All 8 bytes are individually masked and shifted into reversed positions in a single expression.

**Algorithm:** Eight mask+shift pairs, one per byte:
- Byte 0 (bits 0–7): shifts left 56 → becomes byte 7
- Byte 7 (bits 56–63): shifts right 56 → becomes byte 0
- Byte 1 (bits 8–15): shifts left 40 → becomes byte 6
- Byte 6 (bits 48–55): shifts right 40 → becomes byte 1
- Byte 2 (bits 16–23): shifts left 24 → becomes byte 5
- Byte 5 (bits 40–47): shifts right 24 → becomes byte 2
- Byte 3 (bits 24–31): shifts left 8  → becomes byte 4
- Byte 4 (bits 32–39): shifts right 8 → becomes byte 3

**Callers:**
- `OR_PUT_INT64` macro (`object_representation.h:114–119`): serializes `INT64` to disk
- `OR_GET_INT64` macro (`object_representation.h:121–126`): deserializes `INT64` from disk
- `OR_PUT_PTR` / `OR_GET_PTR` (`object_representation.h:164–165`): pointer-size serialization on 64-bit platforms

---

### 6.12 `OR_MOVE_DOUBLE` — Safe Double Copy Macro (Header Only)

```c
/* Non-IA64: */
#define OR_MOVE_DOUBLE(src, dst) \
  (((MOVING_VAN *)(dst))->bits = ((MOVING_VAN *)(src))->bits)

/* IA-64: */
#define OR_MOVE_DOUBLE(src, dst) \
  memcpy(((MOVING_VAN*)(dst))->bits.buf, \
         ((MOVING_VAN *)(src))->bits.buf, sizeof(MOVING_VAN))
```

**Defined at:** `byte_order.h:87–93`

**Description:** Copies 8 bytes from `src` to `dst` by treating both as `MOVING_VAN*`, safely transferring a `double` value by its raw bits without strict aliasing violations.

**Note:** On big-endian, `ntohd` and `htond` are macros using `OR_MOVE_DOUBLE` directly (no byte reversal needed, just a safe copy). On little-endian, the function-based versions (`ntohd`/`htond`) use direct `char*` aliasing in their implementations.

**Callers:**
- `src/object/object_primitive.c` — `double` memory field read/write (3 sites)
- `src/compat/db_macro.c` — `DB_MONETARY` amount extraction
- `src/compat/db_value_printer.cpp` — monetary display
- `src/storage/extendible_hash.c` — double key comparison in hash index
- Big-endian `ntohd`/`htond` macro definitions

---

## 7. Key Algorithms & Logic Flows

### 7.1 Byte Swap Algorithm (char-pointer method)

All multi-byte conversion functions in `byte_order.c` use the same technique:

```c
unsigned int ntohl(unsigned int from) {
  unsigned int to;
  char *ptr  = (char *) &from;
  char *vptr = (char *) &to;
  vptr[0] = ptr[3];   /* network MSB → host LSB position (big to little) */
  vptr[1] = ptr[2];
  vptr[2] = ptr[1];
  vptr[3] = ptr[0];   /* network LSB → host MSB position */
  return to;
}
```

**Why `char*`:** In C and C++, `char*` (and `unsigned char*`) are exempted from strict aliasing rules — accessing any object through a `char*` is always defined behavior. This is the standards-safe way to access raw bytes of any type.

**Memory layout (little-endian host):** The host stores the least significant byte at the lowest address. Network order (big-endian) stores the most significant byte first. So reading a 4-byte network integer where `ptr[0]` is MSB into a little-endian `int` where `vptr[0]` is LSB requires the full reversal.

### 7.2 Float/Double as Integer Carriers

`htonf`/`htond` return `UINT32`/`UINT64` (not `float`/`double`), and `ntohf`/`ntohd` take `UINT32`/`UINT64`. This type-switching design means:

1. **Caller reads** raw bytes from disk/wire into an integer-typed variable.
2. **Passes integer** to `ntohf`/`ntohd`.
3. **Function reverses** bytes and reinterprets as float/double.
4. **Returns** the correctly-typed host value.

This avoids the need for callers to manage intermediate pointers or unions.

### 7.3 `swap64` Expression vs. Function

`swap64` is a pure value-level macro (no pointer dereference), usable in constant expressions or as an rvalue. This contrasts with `ntohi64` which operates on a passed value. The two serve different callsites:

- `swap64(x)`: used in `OR_PUT_INT64`/`OR_GET_INT64` macros — operates on local variables
- `ntohi64(x)`: used in CAS network buffer — operates on `memcpy`-initialized variables

### 7.4 Identity on Big-Endian

On big-endian targets, all conversion macros are identity operations — `ntohs(x) = x`, etc. This means the same `OR_PUT_*`/`OR_GET_*` macros and the same calling code work on both endiannesses with zero overhead on big-endian platforms.

### 7.5 Self-Inverse Property

Byte reversal is its own inverse: `ntoh*(hton*(x)) == x`. This is why:
- `htons` delegates to `ntohs`
- `htonl` delegates to `ntohl`
- `htoni64` delegates to `ntohi64`

Only `htonf`/`htond` have distinct implementations because their input/output types differ (`float`↔`UINT32`, `double`↔`UINT64`), even though the byte operations are the same.

---

## 8. Concurrency & Thread Safety

**Fully thread-safe.** All functions are pure, stateless transformations:
- No global variables
- No static variables
- No shared mutable state
- All computations use local stack variables only

Multiple threads may call any function concurrently without synchronization.

---

## 9. Memory Management

**None.** No heap allocation occurs in any function. All operations are on:
- Function parameters (passed by value)
- Local stack variables (`to`, `p`, `q`, `ptr`, `vptr`)

`memory_wrapper.hpp` is included as the last header per CUBRID project rules, but its allocator tracking hooks are never triggered by this module.

---

## 10. Error Handling

**None.** The module has no error paths:
- All inputs are valid values of their declared types
- No precondition checks
- No `er_set` calls
- No return codes for errors

Callers are responsible for ensuring inputs are within the value range of their types. In practice, all inputs come from disk page buffers or network packets that have been validated upstream.

---

## 11. Integration Points

### 11.1 Disk Serialization (`object_representation.h`)

`byte_order.h` is the foundational dependency of `object_representation.h`, which defines the entire `OR_PUT_*` / `OR_GET_*` macro family. Every value written to or read from a CUBRID disk page goes through these macros:

| Macro family         | Byte-order function used     | Data type              |
|----------------------|------------------------------|------------------------|
| `OR_PUT_SHORT` / `OR_GET_SHORT` | `htons` / `ntohs`  | `short` (16-bit)      |
| `OR_PUT_INT` / `OR_GET_INT`     | `htonl` / `ntohl`  | `int` (32-bit)        |
| `OR_PUT_INT64` / `OR_GET_INT64` | `swap64`           | `INT64` (64-bit)      |
| `OR_PUT_BIGINT` / `OR_GET_BIGINT` | via `OR_PUT_INT64` | `DB_BIGINT`          |
| `OR_PUT_FLOAT` / `OR_GET_FLOAT` | `htonf` / `ntohf`  | `float` (IEEE 754)    |
| `OR_PUT_DOUBLE` / `OR_GET_DOUBLE` | `htond` / `ntohd` | `double` (IEEE 754)  |
| `OR_PUT_PTR` / `OR_GET_PTR`     | `swap64`           | Pointer (64-bit arch) |

CUBRID stores all multi-byte fields on disk in **network byte order** (big-endian). This ensures disk pages written on little-endian x86 hardware can be read on big-endian SPARC/AIX without format conversion.

### 11.2 XASL Stream Serialization

The XASL serialization layer (`src/query/xasl_to_stream.c`, `stream_to_xasl.c`) uses `OR_PUT_*`/`OR_GET_*` macros extensively to pack query plans for client→server transmission. All numeric fields in XASL nodes pass through `byte_order` conversions.

### 11.3 CAS Network Protocol (`cas_net_buf.c`)

The broker's CAS (Client Access Server) process uses `ntohi64` directly for BIGINT and LOB fields in the CCI network protocol. The broker has its own parallel implementations (`net_htoni64`, `net_htonf`, `net_htond`) for the **send path** — these duplicate the logic of `htoni64`/`htonf`/`htond` but are kept separate in the broker layer. The **receive path** uses `ntohi64` from `byte_order.h`.

### 11.4 Object Primitive Layer (`object_primitive.c`)

`OR_MOVE_DOUBLE` from `byte_order.h` is used in the in-memory attribute representation layer when reading and writing `double` values from heap attribute memory (`tp_Double_domain` operations).

### 11.5 Extendible Hash (`extendible_hash.c`)

`OR_MOVE_DOUBLE` is used for `double` and `DB_MONETARY` key handling in the extendible hash index — both for storing keys and comparing them.

### 11.6 Stored Procedures (`pl_struct_compile.cpp`)

Transitively includes `byte_order.h` via `object_representation.h` for struct compilation in the Java PL engine bridge.

---

## 12. Complexity & Metrics

### Lines of Code

| File            | Total lines | Blank/comment | Effective code |
|-----------------|-------------|---------------|----------------|
| `byte_order.c`  | 199         | ~40           | ~60 (mostly guarded functions, all simple) |
| `byte_order.h`  | 187         | ~40           | ~100 (macro/function declarations) |

### Cyclomatic Complexity

Every function has cyclomatic complexity **1** — pure sequential byte assignments with no branches, loops, or conditionals.

### Function Sizes

| Function     | Lines (body) | Complexity |
|--------------|-------------|------------|
| `ntohs`      | 8           | 1          |
| `ntohl`      | 9           | 1          |
| `ntohf`      | 8           | 1          |
| `ntohd`      | 12          | 1          |
| `ntohi64`    | 12          | 1          |
| `htons`      | 1 (delegate)| 1          |
| `htonl`      | 1 (delegate)| 1          |
| `htoni64`    | 1 (delegate)| 1          |
| `htonf`      | 9           | 1          |
| `htond`      | 12          | 1          |

### Preprocessor Complexity

The header has significant compile-time branching complexity:
- 2 main paths: `OR_BYTE_ORDER == OR_LITTLE_ENDIAN` vs. big-endian
- 8 `OR_HAVE_*` guards within the little-endian path
- 1 `IA64` guard for `OR_MOVE_DOUBLE`
- 5 platform guards for system headers (LINUX, sun, HPUX, AIX, WINDOWS)
- 1 MSVC version guard for Windows float/double variants

Total distinct compile-time configurations: theoretical maximum ~96, practical: ~5 (Linux x86-64, Windows MSVC, Windows legacy, HPUX/AIX, SPARC).

---

## 13. Notable Patterns & Idioms

### Pattern 1: `char*` Aliasing for Byte Access

```c
ptr = (char *) &from;
vptr = (char *) &to;
vptr[0] = ptr[3];
```

The `char*` cast is the C-standard-blessed way to access raw bytes of any object. This is the canonical idiom for byte swapping in pre-C++20 C/C++ code without `std::bit_cast`.

### Pattern 2: Type Carrier Pattern for Float/Double

```c
UINT32 htonf(float from);     /* float in → raw bits out */
float  ntohf(UINT32 from);    /* raw bits in → float out */
```

Returning an integer from a float conversion (and vice versa) avoids the need for callers to manage union or pointer types. The caller can write:
```c
UINT32 ui = htonf(val);
memcpy(ptr, &ui, sizeof(ui));
```
This is cleaner than requiring the caller to hold an intermediate union.

### Pattern 3: Self-Inverse Delegation

```c
unsigned short htons(unsigned short from) { return ntohs(from); }
```

Since byte-reversal is self-inverse on any symmetric type, the `hton*` functions for integer types simply delegate to their `ntoh*` counterparts. This eliminates code duplication while correctly expressing the symmetry.

### Pattern 4: `OR_HAVE_*` Capability Macros

Rather than testing platform macros directly in the `.c` file, the header centralizes platform detection into `OR_HAVE_NTOHS` etc. This one-level indirection means the `.c` file only tests `OR_HAVE_*` — making it easy to add a new platform by only editing the header.

### Pattern 5: `MOVING_VAN` Union for Safe Double Copy

Using a named union with a `bits` sub-struct for safe double-as-bits access is a well-known C idiom predating `memcpy` widespread use for this purpose. The name "moving van" humorously evokes moving data from one container to another.

### Pattern 6: Endianness as Compile-Time Constant

`OR_BYTE_ORDER` is set at compile time from platform detection macros. CUBRID does **not** use runtime endianness detection (e.g., via a union probe). This is correct for virtually all modern architectures where endianness is fixed at build time, and avoids any branch overhead in production code.

### Pattern 7: No `bswap` Intrinsics

The code does not use compiler intrinsics like GCC's `__builtin_bswap32` or x86 `bswap` instruction. The byte-by-byte char-copy approach is portable but may be slightly slower than a single `bswap` instruction on x86. Modern compilers (GCC, Clang) typically recognize the byte-swap pattern and emit `bswap` automatically — so the portability cost is essentially zero on optimized builds.

### Pattern 8: Broker Duplication (`net_htoni64` etc.)

`cas_net_buf.c` defines its own `net_htoni64`, `net_htonf`, `net_htond` with identical byte-swap logic. This is a historical artifact — the broker process is a largely self-contained binary that predates the unified `byte_order.h`. The receive path in the broker does call back into `ntohi64` from `byte_order.h`, but the send path uses its own copies. This minor duplication is a known CUBRID code pattern and is not a bug.

### Pattern 9: `memory_wrapper.hpp` Last-Include Rule

```c
#include "byte_order.h"
// XXX: SHOULD BE THE LAST INCLUDE HEADER
#include "memory_wrapper.hpp"
```

This is a project-wide invariant enforced by CI. `memory_wrapper.hpp` overrides `malloc`/`free`/`new`/`delete` for leak tracking; placing it last ensures it wraps all memory operations from all previously included headers.

---

## Appendix: Cross-Reference Table

| Symbol           | Defined in           | Big-endian form        | Little-endian form      | Used in                              |
|------------------|----------------------|------------------------|-------------------------|--------------------------------------|
| `ntohs`          | OS or `byte_order.c` | `#define ntohs(x) (x)` | function / OS           | `OR_GET_SHORT`                       |
| `ntohl`          | OS or `byte_order.c` | `#define ntohl(x) (x)` | function / OS           | `OR_GET_INT`, `OR_GET_TIME`, etc.    |
| `ntohi64`        | `byte_order.c`       | `#define ntohi64(x) (x)` | function              | `cas_net_buf.c`, `htoni64`           |
| `ntohf`          | `byte_order.c`       | `#define ntohf(p,v) ...` | function              | `OR_GET_FLOAT`                       |
| `ntohd`          | `byte_order.c`       | `#define ntohd(p,v) ...` | function              | `OR_GET_DOUBLE`                      |
| `htons`          | OS or `byte_order.c` | `#define htons(x) (x)` | function / OS           | `OR_PUT_SHORT`                       |
| `htonl`          | OS or `byte_order.c` | `#define htonl(x) (x)` | function / OS           | `OR_PUT_INT`, `OR_PUT_TIME`, etc.    |
| `htoni64`        | `byte_order.c`       | `#define htoni64(x) (x)` | function              | (serialization contexts)             |
| `htonf`          | `byte_order.c`       | `#define htonf(p,v) ...` | function              | `OR_PUT_FLOAT`                       |
| `htond`          | `byte_order.c`       | `#define htond(p,v) ...` | function              | `OR_PUT_DOUBLE`                      |
| `swap64`         | `byte_order.h`       | `#define swap64(x) (x)` | expression macro       | `OR_PUT_INT64`, `OR_GET_INT64`, `OR_PUT_PTR`, `OR_GET_PTR` |
| `OR_MOVE_DOUBLE` | `byte_order.h`       | macro (assign or memcpy) | macro (assign or memcpy) | `object_primitive.c`, `db_macro.c`, `db_value_printer.cpp`, `extendible_hash.c` |
| `MOVING_VAN`     | `byte_order.h`       | union type             | union type              | `OR_MOVE_DOUBLE` implementation      |
| `OR_LITTLE_ENDIAN` | `byte_order.h`     | 1234 constant          | 1234 constant           | `OR_BYTE_ORDER` comparison           |
| `OR_BIG_ENDIAN`  | `byte_order.h`       | 4321 constant          | 4321 constant           | `OR_BYTE_ORDER` comparison           |
| `OR_BYTE_ORDER`  | `byte_order.h`       | `OR_BIG_ENDIAN`        | `OR_LITTLE_ENDIAN`      | compile-time dispatch throughout     |
