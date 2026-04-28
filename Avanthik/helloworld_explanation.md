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