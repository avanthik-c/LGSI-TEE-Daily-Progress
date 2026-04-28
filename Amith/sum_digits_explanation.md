# Sum of Digits — OP-TEE TA/CA

A simple OP-TEE program that computes the sum of digits of an integer across the TrustZone boundary.

---

## Directory Structure

```
sum_digits/
├── Makefile
├── host/
│   ├── Makefile
│   └── main.c
└── ta/
    ├── Makefile
    ├── sub.mk
    ├── include/
    │   └── sum_digits_ta.h
    ├── sum_digits_ta.c
    └── user_ta_header_defines.h
```

---

## Full Code

### `ta/include/sum_digits_ta.h`

```c
#ifndef SUM_DIGITS_TA_H
#define SUM_DIGITS_TA_H

#define TA_SUM_DIGITS_UUID \
    { 0x12345678, 0x8765, 0x4321, \
        { 0xab, 0xcd, 0x11, 0x22, 0x33, 0x44, 0x55, 0x66} }

#define TA_SUM_DIGITS_CMD_CALCULATE 0

#endif /* SUM_DIGITS_TA_H */
```

---

### `ta/sum_digits_ta.c`

```c
#include <tee_internal_api.h>
#include <tee_internal_api_extensions.h>
#include <sum_digits_ta.h>

TEE_Result TA_CreateEntryPoint(void) {
    return TEE_SUCCESS;
}

void TA_DestroyEntryPoint(void) {
}

TEE_Result TA_OpenSessionEntryPoint(uint32_t param_types,
                                    TEE_Param params[4],
                                    void **sess_ctx) {
    uint32_t exp_param_types = TEE_PARAM_TYPES(TEE_PARAM_TYPE_NONE,
                                               TEE_PARAM_TYPE_NONE,
                                               TEE_PARAM_TYPE_NONE,
                                               TEE_PARAM_TYPE_NONE);
    if (param_types != exp_param_types) return TEE_ERROR_BAD_PARAMETERS;
    (void)&params;
    (void)&sess_ctx;
    return TEE_SUCCESS;
}

void TA_CloseSessionEntryPoint(void *sess_ctx) {
    (void)&sess_ctx;
}

static TEE_Result sum_digits(uint32_t param_types, TEE_Param params[4]) {
    uint32_t exp_param_types = TEE_PARAM_TYPES(TEE_PARAM_TYPE_VALUE_INOUT,
                                               TEE_PARAM_TYPE_NONE,
                                               TEE_PARAM_TYPE_NONE,
                                               TEE_PARAM_TYPE_NONE);

    if (param_types != exp_param_types)
        return TEE_ERROR_BAD_PARAMETERS;

    uint32_t number = params[0].value.a;
    uint32_t sum = 0;

    while (number > 0) {
        sum += number % 10;
        number /= 10;
    }

    params[0].value.b = sum;
    return TEE_SUCCESS;
}

TEE_Result TA_InvokeCommandEntryPoint(void *sess_ctx,
                                      uint32_t cmd_id,
                                      uint32_t param_types, TEE_Param params[4]) {
    (void)&sess_ctx;
    switch (cmd_id) {
        case TA_SUM_DIGITS_CMD_CALCULATE:
            return sum_digits(param_types, params);
        default:
            return TEE_ERROR_BAD_PARAMETERS;
    }
}
```

---

### `host/main.c`

```c
#include <err.h>
#include <stdio.h>
#include <string.h>
#include <tee_client_api.h>
#include <sum_digits_ta.h>

int main(void) {
    TEEC_Result res;
    TEEC_Context ctx;
    TEEC_Session sess;
    TEEC_Operation op;
    TEEC_UUID uuid = TA_SUM_DIGITS_UUID;
    uint32_t err_origin;
    uint32_t user_input;

    res = TEEC_InitializeContext(NULL, &ctx);
    if (res != TEEC_SUCCESS)
        errx(1, "TEEC_InitializeContext failed with code 0x%x", res);

    res = TEEC_OpenSession(&ctx, &sess, &uuid,
                           TEEC_LOGIN_PUBLIC, NULL, NULL, &err_origin);
    if (res != TEEC_SUCCESS)
        errx(1, "TEEC_OpenSession failed with code 0x%x origin 0x%x", res, err_origin);

    printf("Enter an integer to calculate the sum of its digits: ");
    if (scanf("%u", &user_input) != 1) {
        printf("Invalid input.\n");
        goto cleanup;
    }

    memset(&op, 0, sizeof(op));
    op.paramTypes = TEEC_PARAM_TYPES(TEEC_PARAM_TYPE_VALUE_INOUT,
                                     TEEC_PARAM_TYPE_NONE,
                                     TEEC_PARAM_TYPE_NONE,
                                     TEEC_PARAM_TYPE_NONE);
    op.params[0].value.a = user_input;

    printf("Invoking TA to calculate sum...\n");
    res = TEEC_InvokeCommand(&sess, TA_SUM_DIGITS_CMD_CALCULATE, &op, &err_origin);
    if (res != TEEC_SUCCESS)
        errx(1, "TEEC_InvokeCommand failed with code 0x%x origin 0x%x", res, err_origin);

    printf("The sum of the digits is: %u\n", op.params[0].value.b);

cleanup:
    TEEC_CloseSession(&sess);
    TEEC_FinalizeContext(&ctx);

    return 0;
}
```

---

### `Makefile` (root)

```makefile
export V ?= 0

.PHONY: all
all:
	$(MAKE) -C host
	$(MAKE) -C ta

.PHONY: clean
clean:
	$(MAKE) -C host clean
	$(MAKE) -C ta clean
```

---

### `host/Makefile`

```makefile
CC      ?= $(CROSS_COMPILE)gcc
LD      ?= $(CROSS_COMPILE)ld
AR      ?= $(CROSS_COMPILE)ar
NM      ?= $(CROSS_COMPILE)nm
OBJCOPY ?= $(CROSS_COMPILE)objcopy
OBJDUMP ?= $(CROSS_COMPILE)objdump
READELF ?= $(CROSS_COMPILE)readelf

OBJS = main.o

CFLAGS += -Wall -I../ta/include -I$(TEEC_EXPORT)/include
LDADD  += -L$(TEEC_EXPORT)/lib -lteec

.PHONY: all
all: sum_digits

sum_digits: $(OBJS)
	$(CC) $(LDFLAGS) -o $@ $< $(LDADD)

.PHONY: clean
clean:
	rm -f $(OBJS) sum_digits
```

---

### `ta/Makefile`

```makefile
CFG_TEE_TA_LOG_LEVEL ?= 4
CPPFLAGS += -DCFG_TEE_TA_LOG_LEVEL=$(CFG_TEE_TA_LOG_LEVEL)

BINARY = 12345678-8765-4321-abcd-112233445566

-include $(TA_DEV_KIT_DIR)/mk/ta_dev_kit.mk

ifeq ($(wildcard $(TA_DEV_KIT_DIR)/mk/ta_dev_kit.mk), )
clean:
	@echo 'Note: $$(TA_DEV_KIT_DIR)/mk/ta_dev_kit.mk not found, cannot clean TA'
	@echo 'Note: TA_DEV_KIT_DIR=$(TA_DEV_KIT_DIR)'
endif
```

---

### `ta/sub.mk`

```makefile
global-incdirs-y += include
srcs-y += sum_digits_ta.c
```

---

### `ta/user_ta_header_defines.h`

```c
#ifndef USER_TA_HEADER_DEFINES_H
#define USER_TA_HEADER_DEFINES_H

#include <sum_digits_ta.h>

#define TA_UUID       TA_SUM_DIGITS_UUID

#define TA_FLAGS      (TA_FLAG_EXEC_DDR | TA_FLAG_SINGLE_INSTANCE | \
                       TA_FLAG_MULTI_SESSION)
#define TA_STACK_SIZE (2 * 1024)
#define TA_DATA_SIZE  (32 * 1024)

#endif /* USER_TA_HEADER_DEFINES_H */
```

---

## Explanation

### Shared Header (`sum_digits_ta.h`)
This file is included by both the CA and TA. It defines the UUID that uniquely identifies the TA, and the command ID (`0`) that the CA uses to trigger the calculation. Both sides need to agree on these values, so keeping them in one place avoids mismatches.

### Trusted Application (`sum_digits_ta.c`)
This runs in Secure World. The four entry points (`Create`, `Destroy`, `OpenSession`, `CloseSession`) are mandatory boilerplate for any OP-TEE TA — none of them do anything special here. The actual work happens in `sum_digits()`, which pulls the input number from `params[0].value.a`, loops through each digit using `% 10` and `/ 10`, and writes the result to `params[0].value.b`. The `TA_InvokeCommandEntryPoint` function just routes the incoming command ID to `sum_digits()`.

### Client Application (`main.c`)
This runs in Normal World (Linux userspace). It initializes a TEE context, opens a session with the TA by UUID, packs the user's integer into an operation struct, and calls `TEEC_InvokeCommand`. After the TA processes it, the result is sitting in `op.params[0].value.b`. Cleanup always happens at the end regardless of errors.

### Makefiles
The root Makefile just delegates to `host/` and `ta/`. The host Makefile links against `libteec` (the OP-TEE client library). The TA Makefile pulls in `ta_dev_kit.mk` which handles all the Secure World compilation. One important detail: `BINARY` in `ta/Makefile` must be the UUID string exactly — OP-TEE uses the filename to find the TA when a session is opened.

### `user_ta_header_defines.h`
Sets the TA's UUID (must match the header), flags, and memory limits. `SINGLE_INSTANCE` means only one copy of the TA runs at a time. `MULTI_SESSION` allows multiple CAs to connect to it simultaneously.

---
