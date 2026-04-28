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

### TEEC_OpenSession
```c
TEEC_Result TEEC_OpenSession(
    TEEC_Context *context,
    TEEC_Session *session,
    const TEEC_UUID *destination,
    uint32_t connectionMethod,
    const void *connectionData,
    TEEC_Operation *operation,
    uint32_t *returnOrigin
);
```
1. `context` - check the file descriptor we made earlier

2.  `session` - an empty struct we pass
    ```c
    typedef struct {
        /* A pointer back to the pipeline this session belongs to */
        TEEC_Context *ctx; 
        
        /* The actual unique tracker ID returned by the Secure World */
        uint32_t session_id; 
    } TEEC_Session;
    ```
    after function execution,filled with the context and the unique session id

    
1. Driver requests the Normal Kernel to Assign contiguous block of RAM
    to be used as shared region accessible by both the Non-Secure and Secure worlds

2. ioctl() (Input/Output Control) syscall and Pass UUID (Pass the UUID using the pipeline) 
    into the shared region

### TEEC_InvokeCommand Function call
- Trigger Software Interrupt(EL0->EL1) ,runs
- Puts the op.params[0] array to the common region of the RAM
- eg: res = TEEC_InvokeCommand(&sess, TA_HELLO_WORLD_CMD_INC_VALUE, &op &err_origin);
 
### smc Instruction
### System Call Working
when we call a system call,
1.The task we need is stored in a register (suppose register X8)
2.The datas required for the task (operands,memory addresses,etc...) are stored in another 
  register (suppose register B)
3.The assembly instruction (svc or smc) is executed, 
  switches the mode bit to 0 (Kernel Takes over)
4.The kernel saves the state to its pcb
5.The kernel reads the instruction from the register(Register X8)
6.The kernel checks the instruction number with the secure System Call Table and executes the code 
  at that  memory address, there may be input parameters required for some codes, which will be taken 
  from Register B
7.Cases such as i/o read/write and such may need the memory address of the place to read/write from/to


```plain
System Call Table
| Instruction | Memory Address of the instruction |
---------------------------------------------------
| 1           |      0x11322                      |
---------------------------------------------------

OPTEE driver is one such code in the System call table
OPTEE driver structure:
1) Validity Check:
    checks if your Host app is allowed to make this request, if the Session ID is valid, 
    and if the parameters (the rules you set with TEEC_PARAM_TYPES) make sense. 
    If your app tries to do something illegal, the driver rejects the request right here, 
    and the Secure World is never bothered.
2) Memory Translation:
    The Register B contents are moved to the Shared region between secure and normal RAM
    in a standardized form
3) Store in Registers:
    The Driver stores the Function ID in one register and shared memory address in another register

    "Function ID: Single TA itself will have many functions in its program,we decide which function to run
    using Function ID"

4) smc Assembly Instruction executed :
    ns-bit flipped to 0
    When ns-bit is 0 immediately secure kernel takes control(similar to how kernel space taked control on mode bit =0)
    now the OPTEE kernel takes over
```
