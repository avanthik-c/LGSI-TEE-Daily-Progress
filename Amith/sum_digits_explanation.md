# Sum of Digits — OP-TEE TA/CA

A simple OP-TEE example that computes the sum of digits of an integer. The actual calculation runs inside the Secure World (TA), while the Normal World program (CA) handles user input and displays the result.

---

## Directory Structure

```
sum_digits/
├── Makefile
├── host/
│   ├── Makefile
│   └── main.c                        # Client Application (CA)
└── ta/
    ├── Makefile
    ├── sub.mk
    ├── include/
    │   └── sum_digits_ta.h           # Shared UUID & command IDs
    ├── sum_digits_ta.c               # Trusted Application (TA)
    └── user_ta_header_defines.h
```

---

## Shared Header (`ta/include/sum_digits_ta.h`)

Defines the TA's UUID and the command ID. Both the CA and TA include this file so they stay in sync.

```c
#define TA_SUM_DIGITS_UUID \
    { 0x12345678, 0x8765, 0x4321, \
        { 0xab, 0xcd, 0x11, 0x22, 0x33, 0x44, 0x55, 0x66} }

#define TA_SUM_DIGITS_CMD_CALCULATE 0
```

---

## Trusted Application (`ta/sum_digits_ta.c`)

The core logic lives here. It receives the number via `params[0].value.a`, runs a modulo-10 loop, and writes the result to `params[0].value.b`.

```c
uint32_t number = params[0].value.a;
uint32_t sum = 0;

while (number > 0) {
    sum += number % 10;
    number /= 10;
}

params[0].value.b = sum;
```

The file also implements the four required OP-TEE entry points (`TA_CreateEntryPoint`, `TA_DestroyEntryPoint`, `TA_OpenSessionEntryPoint`, `TA_CloseSessionEntryPoint`) — none of them need any special setup for this example.

---

## Client Application (`host/main.c`)

Standard OP-TEE client flow:

1. `TEEC_InitializeContext` — connect to the TEE
2. `TEEC_OpenSession` — open a session with the TA using its UUID
3. Read integer from user, put it in `op.params[0].value.a`
4. `TEEC_InvokeCommand` — send `CMD_CALCULATE` to the TA
5. Read result from `op.params[0].value.b` and print it
6. `TEEC_CloseSession` + `TEEC_FinalizeContext` — clean up

---

## Makefiles

**Root `Makefile`** — just calls `make` in `host/` and `ta/`.

**`host/Makefile`** — compiles `main.c` and links against `libteec`. Needs `TEEC_EXPORT` pointing to the OP-TEE client lib.

**`ta/Makefile`** — the `BINARY` name must be the UUID string exactly, since that's how OP-TEE locates the TA at runtime.

**`ta/user_ta_header_defines.h`** — sets the UUID, flags, and stack/heap sizes for the TA.


---
