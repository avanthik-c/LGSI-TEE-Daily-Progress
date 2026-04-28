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

### Terms and Macros
TEEC_SUCCESS vs TEE_SUCCESS
TEEC_SUCCESS -> used int normal world, simply the error code 0x00000000
TEE_SUCCESS -> used in secure world, signals OPTEE kernel of TA execution completion
## Function call Timeline

TEEC_InitializeContext()
TEEC_OpenSession() -> TA_CreateEntryPoint()
TEEC_InvokeCommand() -> TA_InvokeCommandEntryPoint()
TEEC_CloseSession() -> TA_CloseSessionEntryPoint()
TEEC_FinalizeContext()


### Phase 1: Establishing the Pipeline
Normal World: TEEC_InitializeContext()
Action: Opens the Linux file descriptor (/dev/tee0).
State: Host App is RUNNING.
Secure World: Asleep/Unaware.
-------------------------------------------
### Phase 2: Booting the Vault & Connecting
Normal World: TEEC_OpenSession()
Action: Packages the UUID, fires the smc instruction.
State: Host App PAUSES (Execution is physically suspended by the CPU).

[ BOUNDARY CROSSING to Secure World ]

Secure World: TA_CreateEntryPoint()
Action: The TA is loaded into Secure RAM. This function runs only once to do global setup (like allocating memory).
Secure World: TA_OpenSessionEntryPoint()
Action: Allocates tracking variables for this specific Host connection. Returns TEE_SUCCESS.

[ BOUNDARY CROSSING to Normal World ]

Normal World: TEEC_OpenSession finishes.
State: Host App WAKES UP and resumes execution.
--------------------------
### Phase 3: The Execution
Normal World: Host code runs printf and scanf to get the user_input.
Normal World: TEEC_InvokeCommand()
Action: Packages the data into Shared Memory, fires the smc instruction.
State: Host App PAUSES.

[ BOUNDARY CROSSING to Secure World ]

Secure World: TA_InvokeCommandEntryPoint()
Action: The Switchboard receives the Command ID, routes it to your specific math function, modifies the Shared Memory, and returns TEE_SUCCESS.

[ BOUNDARY CROSSING to Normal World ]

Normal World: TEEC_InvokeCommand finishes.
State: Host App WAKES UP and resumes execution. It pulls the result from Shared Memory and prints it.
-----------------------------------
### Phase 4: Tearing Down the Vault
Normal World: TEEC_CloseSession()
Action: Tells OP-TEE to drop the secure tracker, fires the smc instruction.
State: Host App PAUSES.

[ BOUNDARY CROSSING to Secure World ]

Secure World: TA_CloseSessionEntryPoint()
Action: Cleans up variables specific to that Host connection.
Secure World: TA_DestroyEntryPoint()
Action: Because this was the only active session, OP-TEE is preparing to kick the TA out of Secure RAM. This function runs only once to free global memory.

[ BOUNDARY CROSSING to Normal World ]

Normal World: TEEC_CloseSession finishes.
State: Host App WAKES UP and resumes execution.
--------------------------
### Phase 5: Final Cleanup
Normal World: TEEC_FinalizeContext()
Action: Closes the Linux file descriptor.
State: Host App finishes and cleanly exits.
Secure World: Unaffected (The TA was already wiped from Secure RAM).

## Host Program
```c
#include <err.h>
#include <stdio.h>
#include <string.h>
#include <tee_client_api.h>  //optee_client
#include <hello_world_ta.h>

int main(void){
    //structures
    TEEC_Result res;
	TEEC_Context ctx;
	TEEC_Session sess;
	TEEC_Operation op; //region of host memory that is to be copied to shared memory region
	TEEC_UUID uuid = TA_HELLO_WORLD_UUID; //need to replace macro accordingly

	uint32_t err_origin;

    res = TEEC_InitializeContext(NULL, &ctx);
	if (res != TEEC_SUCCESS) errx(1, "TEEC_InitializeContext failed with code 0x%x", res);
    // changeable
    // NULL- default tee (here optee)
    //     can put tee1,tee2 etc depending on the tee needed
    
	res = TEEC_OpenSession(&ctx,&sess,&uuid,TEEC_LOGIN_PUBLIC,NULL(connection data),NULL(operation),&err_origin);
    //    TEEC_OpenSession(TEEC_Context *context,TEEC_Session *session,const TEEC_UUID *destination,
    // uint32_t connectionMethod,const void *connectionData,TEEC_Operation *operation,uint32_t *returnOrigin);
	
    // changeable
    // &uuid - changes according to the TA we are opening
    // TEEC_LOGIN_PUBLIC (The Bouncer) - access control mechanism
    //     TEEC_LOGIN_PUBLIC means any program on Linux can knock on the TAs door. 
    //     TEEC_LOGIN_USER Only a specific Linux user (like the root user) is allowed to open the session.
    //     TEEC_LOGIN_GROUP Only programs belonging to a specific Linux user group can access the TA.
    // operation - pass payload at this stage
    // connection data - pass data regarding connection data
    if (res != TEEC_SUCCESS) errx(1, "TEEC_Opensession failed with code 0x%x origin 0x%x",res, err_origin);
	memset(&op, 0, sizeof(op)); // clean the region,setting it all 0s
    
    int user_input;
 	printf("Enter a number to add to the secret key: ");
 	scanf("%d", &user_input);
    // changeable 
    // change this accordingly for user interaction

    op.paramTypes = TEEC_PARAM_TYPES(TEEC_VALUE_INOUT, TEEC_NONE, TEEC_NONE, TEEC_NONE);
    // changeable
    // we have 4 slots to assign 2 types of payload - value or memory address
    // we can have them as input(just send value to ta) ,output(just return value from ta) ,inout(modify the input value and return)
    // following macros
    // value:
    //  TEEC_VALUE_INPUT
    //  TEEC_VALUE_OUTPUT
    //  TEEC_VALUE_INOUT
    //     op.params[0].value.a 
    //     op.params[0].value.b ([0] is the slot index)
    //  one value slot can store two 32-bit integer (.a and .b)
    // memory address:    
    //  TEEC_MEMREF_TEMP_INPUT
    //  TEEC_MEMREF_TEMP_OUTPUT
    //  TEEC_MEMREF_TEMP_INOUT
    //     op.params[0].memref.buffer ( starting address)
    //     op.params[0].memref.size  (size of block)
    //  one memory address slot should be assigned starting address and size of block
    // TEEC_NONE - no value is stored

    op.params[0].value.a = user_input; // .b not used here, so to prevent garbage value from getting sent, we did memset earlier
    res = TEEC_InvokeCommand(&sess, COMMAND_ID_MACRO, &op,&err_origin);// !!!!!!!!!!!!need to modify!!!!!!!!!!
    // changeable
    // &sess - give the session variable of which session we need
    // TA_HELLO_WORLD_CMD_ADD_SK - command id macro,can define macro in header file/give direct number
    // &op - the payload to be copied
    // &err_origin - Error Origin (where the error happened)
    //     possible macros:
    //      TEEC_ORIGIN_API (The Host Library)
    //      TEEC_ORIGIN_COMMS (The Linux Kernel Driver)
    //      TEEC_ORIGIN_TEE (The Secure OS)
    //      TEEC_ORIGIN_TRUSTED_APP (Your Secure Code)
    TEEC_CloseSession(&sess);
    TEEC_FinalizeContext(&ctx);
    return 0;
}
```
TEEC_InitializeContext()
TEEC_OpenSession() -> TA_CreateEntryPoint() (once for each program running)
                   -> TA_OpenSessionEntryPoint() (others run rach time these function are called)
TEEC_InvokeCommand() -> TA_InvokeCommandEntryPoint()
TEEC_CloseSession() -> TA_CloseSessionEntryPoint()
TEEC_FinalizeContext()

## TA Program
```c
#include <tee_internal_api.h>
#include <tee_internal_api_extensions.h>

#include <hello_world_ta.h>
#define SECRET_KEY 42

TEE_Result TA_CreateEntryPoint(void)
{
	DMSG("has been called");
	return TEE_SUCCESS;
}

TEE_Result TA_OpenSessionEntryPoint(uint32_t param_types,TEE_Param __unused params[4],void __unused **sess_ctx)
{
	uint32_t exp_param_types=TEE_PARAM_TYPES(TEE_PARAM_TYPE_NONE,TEE_PARAM_TYPE_NONE,
											TEE_PARAM_TYPE_NONE,TEE_PARAM_TYPE_NONE);//expecting no payload
	DMSG("has been called");
	if (param_types != exp_param_types)
		return TEE_ERROR_BAD_PARAMETERS;
	IMSG("hello world called in secureworld terminal\n");
	return TEE_SUCCESS;
}

static TEE_Result add_sk(uint32_t param_types, TEE_Param params[4])
{
	uint32_t exp_param_types = TEE_PARAM_TYPES(TEE_PARAM_TYPE_VALUE_INOUT,TEE_PARAM_TYPE_NONE,
						   						TEE_PARAM_TYPE_NONE,TEE_PARAM_TYPE_NONE);//expecting inout payload

	if (param_types != exp_param_types)
		return TEE_ERROR_BAD_PARAMETERS;
	IMSG("Got value: %u from NW", params[0].value.a);
	uint32_t user_input = params[0].value.a;
	params[0].value.a = user_input + SECRET_KEY;
	IMSG("change value to: %u", params[0].value.a);


	return TEE_SUCCESS;
}

static TEE_Result dec_value(uint32_t param_types, TEE_Param params[4])
{
	uint32_t exp_param_types = TEE_PARAM_TYPES(TEE_PARAM_TYPE_VALUE_INOUT,TEE_PARAM_TYPE_NONE,
											   TEE_PARAM_TYPE_NONE,TEE_PARAM_TYPE_NONE);

	DMSG("has been called");
	if (param_types != exp_param_types)
		return TEE_ERROR_BAD_PARAMETERS;
	IMSG("Got value: %u from NW", params[0].value.a);
	params[0].value.a--;
	IMSG("Decrease value to: %u", params[0].value.a);
	return TEE_SUCCESS;
}

TEE_Result TA_InvokeCommandEntryPoint(void __unused *sess_ctx,uint32_t cmd_id, uint32_t param_types,TEE_Param params[4])
{
	switch (cmd_id) {
	case TA_HELLO_WORLD_CMD_ADD_SK:
		return add_sk(param_types, params);
	case TA_HELLO_WORLD_CMD_DEC_VALUE:
		return dec_value(param_types, params);
	default:
		return TEE_ERROR_BAD_PARAMETERS;
	}
}

void TA_CloseSessionEntryPoint(void __unused *sess_ctx)
{
	IMSG("Goodbye!\n");
}

void TA_DestroyEntryPoint(void)
{
	DMSG("has been called");
}


```

export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
make run