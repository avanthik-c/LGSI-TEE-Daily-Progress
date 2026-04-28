# Program Explained
## File Structure for the program (hello_world)
```bash
hello_world
├── Android.mk
├── CMakeLists.txt
├── Makefile
├── host
│   ├── Makefile
│   └── main.c
└── ta
    ├── Android.mk
    ├── Makefile
    ├── hello_world_ta.c
    ├── include
    │   └── hello_world_ta.h
    ├── sub.mk
    └── user_ta_header_defines.h
```
## Terms
- op.params[0] array : used to pass values b/w host program and secure program .
- TEEC_InvokeCommand switch from host to secure mode(flip ns-bit)
    eg: res = TEEC_InvokeCommand(&sess, TA_HELLO_WORLD_CMD_INC_VALUE, &op,&err_origin);
    what  is res?
- return TEE_SUCCESS  : Secure program terminated, ns-bit flips
- setup uuid: create the secure program (`ta/hello_world_ta.c`), find its uuid(`uuidgen`) 
    then save it as a macro in `ta/include/hello_world_ta.h`

- why a seperate file fora macro you ask?
    -`ta/include/hello_world_ta.h` is in the common memory buffer in the RAM so that both
    normal and secure os can load/copy the contents
```bash
hello_world/
├── host/
│   └── main.c                 <-- Compiles into the Normal World app
├── ta/
│   ├── hello_world_ta.c       <-- Compiles into the Secure World app
│   └── include/
│       └── hello_world_ta.h   <-- The Shared Contract(i.e, common buffer)
```
## Privileges
- Exception Levels (EL): ARM processors use levels to define hardware privileges
- EL0: Unprivileged (User apps like your Host and TA) //Similar to user space
- EL1: Privileged (Operating Systems like Linux and OP-TEE) //similar to kernel space
- EL3: The highest privilege level, reserved exclusively for the Secure Monitor
    (the ultimate hardware gatekeeper)
- The NS Bit (Non-Secure Bit): This is a single, physical transistor inside a core hardware register (SCR_EL3)
 When this bit is 1, the CPU is physically restricted (Normal World)
 When it is 0, the CPU has access to everything (Secure World)

## Function calls

### TEEC_InitializeContext

```c
TEEC_Result TEEC_InitializeContext(
    const char *name,
    TEEC_Context *context
);
```
1. `name` - Name of the TEE we want (pass `NULL` for default)
    passing `NULL` means to open `/dev/tee0`
    
2. `context` - an empty struct we pass , 
    after function operation file decriptor is filled
    the location to connect is decided based on the `name` parameter
    if we pass tee1 in `name` then `/dev/tee1` will be connected
    `open()` syscall on Linux OP-TEE driver file (create a pipeline to driver)
