## how to add (“stub”) a new system call (e.g., `trace`) in the xv64 kernel. 


### 1. Define the System Call Number and Prototype

1. **Assign a System Call Number**  
   In `syscall.h`, define the constant representing the new system call:
   ```c
   #define SYS_trace  22
   ```

2. **Declare the Function Prototype**  
   In `syscall.c` (or the file where the system call prototypes are collected), add the prototype:
   ```c
   extern uint64 sys_trace(void);
   ```

### 2. Implement the Kernel-Side Function

1. **Write the Function Logic**  
   In `sysproc.c` (or another appropriate source file), implement the system call:
   ```c
   uint64
   sys_trace(void)
   {
     int n;
     if(argint(0, &n) < 0)
       return -1;
     myproc()->syscall_mask = n;
     return 0;
   }
   ```

2. **Register the Function in the System Call Table**  
   In `syscall.c` (or equivalent), locate the system call dispatch table, which is an array of function pointers. Add a mapping from `SYS_trace` to `sys_trace`:
   ```c
   static uint64 (*syscalls[])(void) = {
     [SYS_fork]    sys_fork,
     [SYS_exit]    sys_exit,
     [SYS_wait]    sys_wait,
     ...
     [SYS_trace]   sys_trace
   };
   ```

### 3. Provide a User-Space Stub/Wrapper

1. **Add Entry in `usys.S` (or Equivalent)**  
   In many xv6/xv64 variants, user-level system calls are declared in assembly (e.g., `usys.S`).  
   Add a line such as:
   ```assembly
   entry("trace");
   ```
   This creates a user-space function named `trace()` that performs the necessary trap into the kernel.

2. **Declare the User-Space Function in a Header**  
   In `user.h` (or another user-visible header), declare the function prototype:
   ```c
   int trace(int);
   ```
   Now, user programs can call `trace(n)` to invoke the new system call.

