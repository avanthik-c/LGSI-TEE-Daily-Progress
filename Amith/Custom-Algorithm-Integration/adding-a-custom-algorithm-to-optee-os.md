# Adding a Custom Crypto Algorithm to OP-TEE OS

A complete, step-by-step guide to making a new algorithm callable through
`TEE_AllocateOperation()` / `TEE_CipherInit()` / `TEE_CipherUpdate()`, exactly
like AES, DES, or SM4 — using a genuine `TEE_ALG_*` constant, not a hand-rolled
function inside a Trusted Application.

This guide is written from a real, verified implementation (a teaching
"shift cipher" added to OP-TEE OS), so every file path, line of code, and
gotcha below comes from something that was actually hit and fixed, not
theory.

---

## 0. Who this is for, and what "adding an algorithm" actually means

There are two very different things people mean when they say "add my own
crypto algorithm to OP-TEE":

1. **Write a function that does cipher math, and call it from inside your
   TA.** This does *not* touch `optee_os` at all. It's just C code in your
   Trusted Application, using `TEE_Malloc`, `TEE_MemMove`, etc. as plumbing.
   No `TEE_ALG_*` constant is involved. This is simple, but your algorithm
   never becomes a "real" `TEE_ALG_*` operation — you can't
   `TEE_AllocateOperation()` it.

2. **Make the OP-TEE OS core recognize a brand-new `TEE_ALG_*` constant**,
   so any TA can call `TEE_AllocateOperation(&op, TEE_ALG_YOUR_CIPHER, ...)`
   the same way it would call `TEE_ALG_AES_CBC_NOPAD`. **This guide is about
   option 2.**

Option 2 requires editing `optee_os` itself (not just your TA) and
rebuilding the entire TEE core, because algorithm validation is checked at
**seven independent layers**, each of which has its own hardcoded
allow-list of known algorithms. Miss any one layer, and you'll get
`TEE_ERROR_NOT_SUPPORTED` (`0xFFFF000A`) or a panic with
`TEE_ERROR_BAD_PARAMETERS` (`0xFFFF0006`) at that exact layer, with every
earlier layer working fine.

---

## 1. Hard constraints — read this before you start

- **This only works for algorithms that fit an existing GP-spec operation
  *class***: `CIPHER`, `MAC`, `AE` (authenticated encryption), `DIGEST`,
  `ASYMMETRIC_CIPHER`, `ASYMMETRIC_SIGNATURE`, or `KEY_DERIVATION`. You are
  not inventing a new *class* of operation (that would require touching the
  syscall dispatch's class switch, `TEE_Panic()` paths in multiple more
  places, and likely GP-spec header changes far beyond this guide). You are
  registering a new *algorithm within an existing class*. This guide covers
  the **CIPHER** class specifically, since that's what was implemented and
  verified — MAC/DIGEST/AE/asymmetric classes follow an analogous pattern
  but touch a different (though structurally similar) set of dispatch
  functions (`crypto_mac_alloc_ctx`, `crypto_hash_alloc_ctx`,
  `crypto_authenc_alloc_ctx`, `crypto_acipher_*`, respectively — not
  documented step-by-step here).
- **You must pick a genuinely unused "main algorithm" byte value.** OP-TEE
  packs each `TEE_ALG_*` constant as a 32-bit value: class in the top
  nibble, chain mode in bits [11:8], and a "main algorithm" ID in the low
  byte. If you reuse an existing main-algorithm byte, you will silently
  corrupt behavior for the real algorithm that already owns it. Always
  grep the full codebase for your chosen byte before committing to it (Step
  2 below shows exactly how).
- **This requires a full `optee_os` rebuild**, not just a TA rebuild, since
  the changes span libutee (compiled into every TA) and OP-TEE's kernel/core
  (the secure-world OS image itself). Expect a `make run` cycle of several
  minutes each time you test a change.
- **You need to already have a working OP-TEE build environment** (QEMU or
  real hardware) where you can build and boot `optee_os`, and where you can
  build and deploy example TAs (e.g. `optee_examples`). This guide assumes
  that environment already exists and works.

---

## 2. Step 1 — Choose your algorithm ID and verify it's free

### 2.1 Find the bit-packing macros

The packing logic lives in `lib/libutee/include/utee_defines.h`. Look for:

```c
static inline uint32_t __tee_alg_get_class(uint32_t algo)
{
    ...
    return (algo >> 28) & 0xF;   /* Bits [31:28] = class */
}

static inline uint32_t __tee_alg_get_main_alg(uint32_t algo)
{
    ...
    return algo & 0xff;          /* Bits [7:0] = main algorithm */
}

#define TEE_ALG_GET_CHAIN_MODE(algo) (((algo) >> 8) & 0xF)  /* Bits [11:8] */
```

So a `TEE_ALG_*` constant is built as:

```
0x [class:1 nibble] [unused] [chain_mode:1 nibble] [main_algo:1 byte]
```

Example: `TEE_ALG_AES_CBC_NOPAD = 0x10000110`
- class = `0x1` = `TEE_OPERATION_CIPHER`
- chain_mode = `0x1` (CBC)
- main_algo = `0x10` (AES)

### 2.2 List every existing `TEE_MAIN_ALGO_*` value

```bash
cd <optee_os>
grep -n "TEE_MAIN_ALGO_" lib/libutee/include/utee_defines.h
```

Pick a byte value **not** in that list. In the reference implementation,
`0x20` was free (used ranges at the time were roughly `0x01-0x0B` for
hashes, `0x10-0x14` for symmetric ciphers, `0x30-0x32` for RSA/DSA/DH,
`0x41-0x49` for ECC, `0xC0-0xC4` for KDFs).

### 2.3 Double-check for collisions with a raw grep

Never trust the list alone — grep the whole tree for your chosen final
32-bit value once assembled, since duplicate/leftover definitions can exist
from earlier experiments:

```bash
grep -rn "0x10000020" --include="*.h" --include="*.c" .
```

(Substitute your actual computed constant.) It should return nothing, or
only files you're about to create/edit yourself.

### 2.4 Pick your class and chain mode

For a stream-style cipher with no meaningful chaining, use class
`TEE_OPERATION_CIPHER` (`0x1`) and chain mode `0x0`. Your final constant is:

```
TEE_ALG_YOUR_CIPHER = 0x1 0000 0 <your_main_algo_byte>
```

---

## 3. Step 2 — Define the constant (2 small files)

### File: `lib/libutee/include/tee_api_defines.h`

Add, near the other cipher algorithm defines:

```c
#define TEE_ALG_YOUR_CIPHER    0x10000020 /* OP-TEE extension: <describe it> */
```

### File: `lib/libutee/include/utee_defines.h`

Add, near the other `TEE_MAIN_ALGO_*` defines:

```c
#define TEE_MAIN_ALGO_YOUR_CIPHER 0x20 /* OP-TEE extension */
```

### Verify before moving on

Write a tiny standalone test program that includes only these two headers
(plus their dependencies) and checks the macros compute what you expect —
**do this before touching anything else in the tree**:

```bash
cat > /tmp/test_alg.c << 'EOF'
#include <stdio.h>
#include <stdint.h>
#include "tee_api_defines.h"
#include "utee_defines.h"

int main(void) {
    uint32_t algo = TEE_ALG_YOUR_CIPHER;
    printf("class      = %u (expect 1 = TEE_OPERATION_CIPHER)\n",
           TEE_ALG_GET_CLASS(algo));
    printf("main_alg   = 0x%02x\n", TEE_ALG_GET_MAIN_ALG(algo));
    printf("chain_mode = 0x%x (expect 0)\n", TEE_ALG_GET_CHAIN_MODE(algo));
    return 0;
}
EOF

gcc -I lib/libutee/include \
    -I core/include \
    -I lib/libutils/ext/include \
    /tmp/test_alg.c -o /tmp/test_alg
/tmp/test_alg
```

Expect `class = 1`, `main_alg = 0x20`, `chain_mode = 0x0`. If the include
paths above don't resolve on your tree, use `find . -name "compiler.h"` (or
whatever header errors out) to locate the missing directory and add it.

---

## 4. Step 3 — Implement the actual cipher (`core/crypto/`)

### 4.1 Study a reference implementation

Pick the simplest existing cipher implementation as your template — AES-ECB
is a good choice (no IV, no padding complexity). Find it:

```bash
grep -rln "TEE_ALG_AES_ECB_NOPAD" core/crypto/ lib/ | grep -v "\.o$"
```

Open that file. You'll see it implements a `struct crypto_cipher_ops` with
five function pointers: `init`, `update`, `final`, `free_ctx`,
`copy_state`.

### 4.2 Check the exact ops struct shape for your tree

```bash
grep -n -A20 "struct crypto_cipher_ops {" core/include/crypto/crypto_impl.h
```

Confirm the exact signatures — they can shift slightly between OP-TEE
versions.

### File: `core/crypto/your_cipher.c` (new file)

```c
// SPDX-License-Identifier: BSD-2-Clause
/*
 * <your description here>
 */

#include <assert.h>
#include <compiler.h>
#include <crypto/crypto.h>
#include <crypto/crypto_impl.h>
#include <stdlib.h>
#include <string.h>
#include <tee_api_types.h>
#include <utee_defines.h>
#include <util.h>

struct your_cipher_ctx {
	struct crypto_cipher_ctx ctx;
	/* your algorithm's per-operation state goes here, e.g.: */
	uint8_t key;
	TEE_OperationMode mode;
};

static const struct crypto_cipher_ops your_cipher_ops;

static struct your_cipher_ctx *to_your_ctx(struct crypto_cipher_ctx *ctx)
{
	assert(ctx && ctx->ops == &your_cipher_ops);
	return container_of(ctx, struct your_cipher_ctx, ctx);
}

static TEE_Result your_cipher_init(struct crypto_cipher_ctx *ctx,
				   TEE_OperationMode mode,
				   const uint8_t *key1, size_t key1_len,
				   const uint8_t *key2 __unused,
				   size_t key2_len __unused,
				   const uint8_t *iv __unused,
				   size_t iv_len __unused)
{
	struct your_cipher_ctx *c = to_your_ctx(ctx);

	/* validate key1_len for your algorithm here */

	c->mode = mode;
	/* copy/derive whatever state you need from key1 */

	return TEE_SUCCESS;
}

static TEE_Result your_cipher_update(struct crypto_cipher_ctx *ctx,
				     bool last_block __unused,
				     const uint8_t *data, size_t len,
				     uint8_t *dst)
{
	struct your_cipher_ctx *c = to_your_ctx(ctx);

	/* your actual encrypt/decrypt logic goes here, writing exactly
	 * `len` bytes from `data` into `dst` */

	return TEE_SUCCESS;
}

static void your_cipher_final(struct crypto_cipher_ctx *ctx __unused)
{
	/* release any resources allocated in init, if any */
}

static void your_cipher_free_ctx(struct crypto_cipher_ctx *ctx)
{
	free(to_your_ctx(ctx));
}

static void your_cipher_copy_state(struct crypto_cipher_ctx *dst_ctx,
				   struct crypto_cipher_ctx *src_ctx)
{
	struct your_cipher_ctx *src = to_your_ctx(src_ctx);
	struct your_cipher_ctx *dst = to_your_ctx(dst_ctx);

	*dst = *src; /* or copy fields individually */
}

static const struct crypto_cipher_ops your_cipher_ops = {
	.init = your_cipher_init,
	.update = your_cipher_update,
	.final = your_cipher_final,
	.free_ctx = your_cipher_free_ctx,
	.copy_state = your_cipher_copy_state,
};

TEE_Result crypto_your_cipher_alloc_ctx(struct crypto_cipher_ctx **ctx_ret)
{
	struct your_cipher_ctx *c = calloc(1, sizeof(*c));

	if (!c)
		return TEE_ERROR_OUT_OF_MEMORY;

	c->ctx.ops = &your_cipher_ops;
	*ctx_ret = &c->ctx;

	return TEE_SUCCESS;
}
```

**This file cannot be compile-tested standalone** — it needs the full
`optee_os` header tree. You'll verify it compiles as part of Step 8's full
build, not in isolation.

---

## 5. Step 4 — Register the implementation in the dispatch table

### File: `core/include/crypto/crypto_impl.h`

Add the prototype near the other `crypto_*_alloc_ctx` declarations (mimic
the exact placement style used for the algorithm you copied from):

```c
TEE_Result crypto_your_cipher_alloc_ctx(struct crypto_cipher_ctx **ctx);
```

### File: `core/crypto/crypto.c`

Find `crypto_cipher_alloc_ctx()`'s `switch (algo)` block and add a case:

```c
case TEE_ALG_YOUR_CIPHER:
	res = crypto_your_cipher_alloc_ctx(&c);
	break;
```

### File: `core/crypto/sub.mk`

Add your new source file to the build:

```makefile
srcs-y += your_cipher.c
```

Add it right after the existing `srcs-y += crypto.c` line (unconditional —
no `CFG_CRYPTO_*` feature flag needed unless you want one).

### Verify with git diff, not memory

```bash
git diff core/crypto/crypto.c core/crypto/sub.mk core/include/crypto/crypto_impl.h
```

Confirm each diff shows **only an addition**, nothing else touched.

---

## 6. Step 5 — Client-side API validation

### File: `lib/libutee/tee_api_operations.c`

Find `TEE_AllocateOperation()`. It has **two** switch statements over
`algorithm` you need to check:

**6.1 — "Check algorithm max key size" switch** (near the top of the
function). Most cipher algorithms hit the `default: break;` case here
harmlessly — you likely don't need to add anything, *unless* your algorithm
has a fixed/constrained key size you want enforced early. Skip this unless
you have a specific requirement.

**6.2 — "Check algorithm mode" switch** (the important one). This is a
large switch that groups algorithms by shared behavior and sets
`req_key_usage`. Find the group that matches your algorithm's needs. For an
unconstrained stream cipher (no block-size/padding constraints), the
`TEE_ALG_AES_CTR` / `TEE_ALG_AES_GCM` group is the right one to join, since
it leaves `block_size` at its default of `1`:

```c
case TEE_ALG_YOUR_CIPHER:
case TEE_ALG_AES_CTR:
case TEE_ALG_AES_GCM:
	if (mode == TEE_MODE_ENCRYPT)
		req_key_usage = TEE_USAGE_ENCRYPT;
	else if (mode == TEE_MODE_DECRYPT)
		req_key_usage = TEE_USAGE_DECRYPT;
	else
		return TEE_ERROR_NOT_SUPPORTED;
	break;
```

**Do not** join the `TEE_ALG_AES_ECB_NOPAD` / `TEE_ALG_DES_ECB_NOPAD` group
unless your algorithm genuinely has AES/DES-style block alignment — that
group falls through into block-size-setting logic that will impose the
wrong constraints on you.

### Verify

```bash
grep -n -B2 -A6 "TEE_ALG_YOUR_CIPHER" lib/libutee/tee_api_operations.c
```

Confirm it's inserted in exactly one place, in the correct group, with
correct indentation matching its neighbors.

---

## 7. Step 6 — Key type derivation

### File: `lib/libutee/include/utee_defines.h`

`TEE_AllocateOperation()` auto-derives an expected key *object type* from
your algorithm via `__tee_alg_get_key_type()`. The default formula is
`0xA0000000 | main_algo`, which only works if that exact computed value has
a matching entry in the kernel-side object-type table (Step 7 below). The
simpler, more portable approach is to special-case your algorithm to reuse
an existing, already-registered object type — `TEE_TYPE_GENERIC_SECRET` is
the natural choice for any simple symmetric-key algorithm:

```c
static inline uint32_t __tee_alg_get_key_type(uint32_t algo, bool with_priv)
{
	uint32_t key_type;

	if (algo == TEE_ALG_YOUR_CIPHER)
		return TEE_TYPE_GENERIC_SECRET;

	key_type = 0xA0000000 | TEE_ALG_GET_MAIN_ALG(algo);

	if (with_priv)
		key_type |= 0x01000000;

	return key_type;
}
```

> **C syntax note:** declare `key_type` at the top of the function (not
> inline with its assignment) if you're adding an early-return `if` above
> it — some compiler configurations warn (`-Wdeclaration-after-statement`)
> or, on stricter/older toolchains, outright reject a `return` statement
> appearing before a variable declaration in the same block.

---

## 8. Step 7 — Kernel-side syscall validation

Two more independent gates live in the kernel/core, both triggered the
first time your algorithm is actually used with a real key object.

### 8.1 File: `core/tee/tee_svc_cryp.c` — required key type

Find `tee_svc_cryp_check_key_type()`. It has a `switch
(TEE_ALG_GET_MAIN_ALG(algo))` mapping each main algorithm to its required
`TEE_TYPE_*`. Add your case, matching whatever you returned in Step 6:

```c
case TEE_MAIN_ALGO_YOUR_CIPHER:
	req_key_type = TEE_TYPE_GENERIC_SECRET;
	break;
```

Place it anywhere inside the switch (order doesn't matter for
correctness — match the style of inserting it near a related case, e.g.
next to the other symmetric-cipher entries like `TEE_MAIN_ALGO_SM4`).

**⚠️ This function's `switch` statement is long and has multiple
occurrences of very similar-looking `case` groups (symmetric ciphers,
DES/DES3 variants, etc.) elsewhere in the file.** When using `sed` or
scripted edits to insert your case, always **grep for exact line numbers
first** and confirm with `git diff` afterward — pattern-based insertion
(e.g. "insert before the line containing X") can silently match the wrong
occurrence if that pattern string appears more than once in the file. This
exact mistake happened during the reference implementation and deleted an
unrelated, pre-existing case (`TEE_ALG_DES3_CMAC`) instead of removing what
was intended. **Always verify with `git diff` before rebuilding, every
single time you script-edit a file with repeated patterns.**

### 8.2 File: `core/tee/tee_cryp_utl.c` — block size

Find `tee_cipher_get_block_size()`. This function has **no generic
fallback** — every algorithm that reaches `TEE_CipherUpdate()` must have an
explicit case here, even a stream cipher with no real block constraint.
Use `*size = 1` for "no meaningful block size" (this makes every
`len % block_size` check trivially pass):

```c
case TEE_ALG_YOUR_CIPHER:
	*size = 1;
	break;
```

Insert this as its own case near the top of the switch, **not** merged
into the AES group (`*size = 16`) or the DES group (`*size = 8`) unless
your algorithm genuinely has that block size.

### Verify both, precisely

```bash
git diff core/tee/tee_svc_cryp.c core/tee/tee_cryp_utl.c
```

Read the **entire** diff output line by line. Confirm:
- Only additions, no accidental deletions
- Correct indentation (matches surrounding lines — a mismatched indent is
  a strong visual signal something landed in the wrong spot)
- Your case appears in exactly the function you intended, not a
  similarly-shaped function elsewhere in the same file (this file has more
  than one `switch (algo)` block with overlapping case names)

---

## 9. Step 8 — Full rebuild and end-to-end verification

At this point you've touched (typically) 8 files: 2 constant definitions,
1 new implementation file + 3 registration edits, 1 client-side validation
edit, 1 key-type-derivation edit, 2 kernel-side validation edits. Before
rebuilding, do a full-tree sanity check:

```bash
cd <optee_os>
git status
git diff --stat
git diff        # read the whole thing, every hunk, every file
```

If anything looks unfamiliar or unexpected, stop and investigate before
building — it's much cheaper to catch a mistake here than after a 5+ minute
rebuild cycle.

### Rebuild everything

```bash
cd <your build repo, e.g. ~/optee/build>
make run
```

Watch the build log for your new file compiling
(`CC out/arm/core/tee/your_cipher.o` or similar) and for the absence of any
compiler warnings/errors mentioning the files you touched.

### Write a minimal verification TA

Don't test through a complex TA first. Write the smallest possible TA that
exercises exactly the call sequence you care about, with `DMSG`/`EMSG`
logging at every step, e.g. a self-test that runs in `TA_CreateEntryPoint`:

```c
TEE_Result TA_CreateEntryPoint(void)
{
	TEE_Result res;
	TEE_ObjectHandle key_handle = TEE_HANDLE_NULL;
	TEE_OperationHandle op = TEE_HANDLE_NULL;
	TEE_Attribute attr;
	uint8_t key_byte = 7;
	const char *plaintext = "HELLO";
	char ciphertext[16] = { 0 };
	char decrypted[16] = { 0 };
	size_t pt_len = 5, out_len;

	res = TEE_AllocateTransientObject(TEE_TYPE_GENERIC_SECRET, 8,
					  &key_handle);
	if (res != TEE_SUCCESS) { EMSG("AllocTransientObject: 0x%x", res); goto out; }

	TEE_InitRefAttribute(&attr, TEE_ATTR_SECRET_VALUE, &key_byte, 1);
	res = TEE_PopulateTransientObject(key_handle, &attr, 1);
	if (res != TEE_SUCCESS) { EMSG("Populate: 0x%x", res); goto out; }

	res = TEE_AllocateOperation(&op, TEE_ALG_YOUR_CIPHER, TEE_MODE_ENCRYPT, 8);
	if (res != TEE_SUCCESS) { EMSG("AllocOp: 0x%x", res); goto out; }

	res = TEE_SetOperationKey(op, key_handle);
	if (res != TEE_SUCCESS) { EMSG("SetKey: 0x%x", res); goto out; }

	TEE_CipherInit(op, NULL, 0);
	out_len = sizeof(ciphertext);
	res = TEE_CipherUpdate(op, plaintext, pt_len, ciphertext, &out_len);
	if (res != TEE_SUCCESS) { EMSG("Encrypt: 0x%x", res); goto out; }
	DMSG("Encrypt OK, out_len=%zu", out_len);

	/* ...repeat with TEE_MODE_DECRYPT on a fresh operation, then
	 * memcmp(plaintext, decrypted, pt_len) and log PASS/FAIL... */

out:
	if (op != TEE_HANDLE_NULL) TEE_FreeOperation(op);
	if (key_handle != TEE_HANDLE_NULL) TEE_FreeTransientObject(key_handle);
	return TEE_SUCCESS; /* always let the session open; DMSG/EMSG is the real signal */
}
```

Deploy it as a minimal `optee_examples`-style TA/CA pair (UUID + `sub.mk` +
`Makefile` + `CMakeLists.txt`, following the pattern of any existing
example in your `optee_examples` tree), rebuild, boot QEMU, run the CA, and
read the **secure-world console** output (not the normal-world CA output —
the DMSG/EMSG lines land there).

---

## 10. Debugging strategy if something fails

If you hit `TEE_ERROR_NOT_SUPPORTED` (`0xFFFF000A`) or a panic with
`TEE_ERROR_BAD_PARAMETERS` (`0xFFFF0006`), **do not guess which layer is at
fault** — trace it precisely:

### 10.1 Add a temporary debug print at your suspected case

```c
case TEE_ALG_YOUR_CIPHER:
	EMSG("DEBUG: your-cipher case reached here");
	/* fall through / existing logic */
```

Rebuild, rerun, and check whether the DEBUG line appears before the
failure. If it does, the fault is *after* this point; if not, it's
*before* — this immediately halves your search space per gate.

### 10.2 For a panic, get an exact source line via addr2line

A panic prints a call stack of raw addresses and the TA's load base
(visible in the boot log, e.g. `ldelf:207 ELF (...) at 0x40031000`). Find
your TA's `.elf` file (produced alongside the `.ta` and `.dmp` files in its
build output directory), then:

```bash
ADDR2LINE=<your toolchain>/bin/aarch64-linux-gnu-addr2line
ELF=<path to your TA's .elf>
LOAD_BASE=0x40031000   # substitute the address from your boot log

for a in 0xADDR1 0xADDR2 0xADDR3; do
  off=$(printf "0x%x" $((a - LOAD_BASE)))
  echo "runtime=$a  file_offset=$off"
  $ADDR2LINE -e "$ELF" -f -C "$off"
done
```

This maps each call-stack address directly to `function_name` +
`file:line` — a definitive answer instead of a guess. Repeat this at every
new panic; each fix typically reveals the *next* gate further down the call
chain, since GP-spec conformance is layered and each layer only runs once
the previous one lets you through.

### 10.3 Common failure points, in the order they tend to appear

1. Algorithm ID rejected at `TEE_AllocateOperation()`'s mode-check switch
   (Step 6) → `TEE_ERROR_NOT_SUPPORTED` before any DEBUG line you add there
   fires.
2. Key type mismatch at `tee_svc_cryp_check_key_type()` (Step 8.1) →
   `TEE_ERROR_BAD_PARAMETERS`, but **only once you pass a real key object**
   (i.e., after `TEE_AllocateTransientObject`/`TEE_SetOperationKey`
   succeed) — this can look like it "used to work" earlier in the same
   function call chain and then suddenly fail deeper in, which is
   expected: the kernel-side key-type check only runs once `key1 != 0`.
3. Missing block-size case at `tee_cipher_get_block_size()` (Step 8.2) →
   `TEE_ERROR_NOT_SUPPORTED`, surfacing specifically at `TEE_CipherUpdate()`
   rather than at allocation time.
4. Auto-derived key type from Step 6/7 not matching any entry in
   `tee_cryp_obj_props[]` (the object-type size/attribute table in
   `tee_svc_cryp.c`) → this is why special-casing to
   `TEE_TYPE_GENERIC_SECRET` (which already has a valid table entry) is
   simpler than trying to register a brand-new object type from scratch.

### 10.4 Rebuild caching gotchas (Buildroot-based environments)

If you rebuild `optee_os` but a dependent package (e.g. your examples
package) doesn't seem to pick up the change:

- Confirm the *exported* headers actually changed:
  `grep "YOUR_CONSTANT" <optee_os>/out/.../export-ta_arm64/include/*.h`
- Check whether your build system's dependency graph actually links the
  examples package to `optee_os` — it may not, meaning a plain rebuild
  silently reuses cached artifacts. Find the real build-stamp directory
  (e.g. `out-br/build/<package>-<version>/`) and `rm -rf` it to force a
  genuinely clean rebuild before concluding a fix didn't work.
- After any fresh full rebuild, verify with `git diff` (not memory) that
  your source edits are still exactly what you intend — accidental
  double-application of a `sed` command, or a stray leftover edit from an
  earlier attempt, is a common self-inflicted source of "it stopped
  working" confusion.

---

## 11. Final verification checklist

Before considering the implementation complete:

- [ ] `git status` / `git diff --stat` shows only the files you intended to
      touch, plus your one new implementation file (untracked)
- [ ] `git diff` read start to finish shows **only additions** in every
      hunk except any you deliberately intended to modify — no accidental
      deletions of pre-existing cases
- [ ] A full rebuild completes with **no new compiler warnings or errors**
      in the files you touched
- [ ] A minimal self-test TA shows every step succeeding via DMSG, ending
      in a confirmed encrypt → decrypt → roundtrip-match result
- [ ] You've removed any temporary `EMSG("DEBUG: ...")` lines added during
      troubleshooting
- [ ] You've fixed any compiler warnings introduced by your edits (e.g.
      declaration-after-statement issues from adding early-return checks)

---

## 12. Quick-reference: every file touched, by purpose

| Purpose | File | What changes |
|---|---|---|
| Algorithm ID constant | `lib/libutee/include/tee_api_defines.h` | `#define TEE_ALG_YOUR_CIPHER ...` |
| Main-algorithm byte | `lib/libutee/include/utee_defines.h` | `#define TEE_MAIN_ALGO_YOUR_CIPHER ...` |
| Key type derivation | `lib/libutee/include/utee_defines.h` | special-case in `__tee_alg_get_key_type()` |
| Cipher implementation | `core/crypto/your_cipher.c` (new) | `crypto_cipher_ops` struct + alloc function |
| Implementation prototype | `core/include/crypto/crypto_impl.h` | function declaration |
| Implementation dispatch | `core/crypto/crypto.c` | `case` in `crypto_cipher_alloc_ctx()` |
| Build registration | `core/crypto/sub.mk` | `srcs-y += your_cipher.c` |
| Client-side mode validation | `lib/libutee/tee_api_operations.c` | `case` in `TEE_AllocateOperation()` |
| Kernel-side key type check | `core/tee/tee_svc_cryp.c` | `case` in `tee_svc_cryp_check_key_type()` |
| Block size | `core/tee/tee_cryp_utl.c` | `case` in `tee_cipher_get_block_size()` |

Eight files touched, one new file created — for **any** new symmetric
cipher-class algorithm following this exact pattern.
