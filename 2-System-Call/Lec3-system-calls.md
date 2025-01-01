# OS organization and system calls

## 1. Isolation
### layer of isolation
- isolaion between different application
- within os itself

### without isolation (no OS between app and h/w)
- enforce multiplexing: greedy app process have infinite loop (no shceudle class), run on ee encofrced multiplexing
- enforc stron memory isolationno memory protection on physical memeory -> one app can overwrite/flow to memory place of other process

### Unix interface
- abstract the h/w resources:
    - process: abstract CPU resources
    - exec: abstract memory
    - files: abstract disk block

### OS should be defensive
- app can't crash the OS
- app can/t break out isolation
    - H/w suport:
        - user kernel mode
        - virtual memory

## 2. User/Kernel Mode
### Modes of Operation
Process has two mode of operation: user mode and kernel mode
- **user mode**: upriviledged instruction (`add`, `sub`)
- **kernel mode**: priviledged instruction (setting up page table, disable clock interrupts)

### Virtual memory (memory isolation)
- page table: virtual addr -> physical
- process has own page table

### what should run in kernel mode?
**Monolithic Kernel:**
- All services, including file systems and device drivers, operate in kernel mode.

**Microkernel:**
- **Kernel Mode**: Essential services like IPC, multiplexing, and virtual memory management.
- **User Mode**: Non-essential services such as file systems and device drivers, communicating via IPC.

## 3. System Call
### Entering the Kernel
- In xv64, a system call is triggered by: <n/sys_call_number>
- The CPU switches from user mode to kernel mode, and the kernel’s syscall handler uses the system call number to dispatch the correct function.
- **Example**:
`fork -> ecall sys_fork (user mode) -> syscall (kernel) -> fork system call
`
### Kernel as a Trusted Computing Base
- The kernel must be bug-free.
- The kernel must treat all processes as potentially malicious.


### HOW the kernel starts and runs the first process.
**1. Power-On & Boot**
- System powers on and ROM-based boot loader loads the xv6 kernel into memory at `0x80000000` (I/O devices at `0x0:0x80000000`).
- Execution starts at `_entry` (`0x80000000`) with machine-mode paging disabled.

**2. `_entry`**
- Initializes the stack (`stack0`) and sets the stack pointer (`sp`) to `stack0 + 4096` (stack grows downward).
- Jumps to the `start()` function in C.

**3. `start()`**
- **Machine Mode**: Configures key RISC-V registers (timer interrupts, privilege levels, etc.).
- **Supervisor Mode**: Sets up `mret` to switch to supervisor mode and jumps to `main()`.

**4. `main()` & First Process**
- `main()` (in `main.c`) initializes devices and subsystems, then calls `userinit()`.
- `userinit()` creates the first user process, executing `initcode.S`.

**5. First System Call & `/init`**
- In `initcode.S`, the process sets `SYS_EXEC` in `a7` and invokes `ecall`.
- Kernel's `sys_exec()` handles the call, loading `/init` to replace the first process.
- `/init` opens the console (file descriptors 0, 1, 2) and starts the shell, making the system ready for user commands.

    
## Filesystem, User, and Kernel

### Kernel Compilation (RISC-V)
compilation:
- **proc.c**
    - `gcc` → produces `proc.s` (RISC-V assembly)
    - Assembler → produces `proc.o` (binary object)
- **write.c**
    - `gcc` → produces `write.s` (RISC-V assembly)
    - Assembler → produces `write.o` (binary object)

    **...**
    
linkage:
- Linker (`ld`) → combines all objects into the final kernel image  

### QEMU Emulation (RISC-V)
- QEMU can emulate a RISC-V machine.  
- Official documentation: [QEMU About Docs](https://www.qemu.org/docs/master/about/index.html)

### Execution Loop Example
```c
for (;;) {
  // 1. Read instruction
  // 2. Decode instruction (e.g., is it 'add'? 'sub'?)
  // 3. Execute the instruction on registers
  // 4. Run on the host
}
