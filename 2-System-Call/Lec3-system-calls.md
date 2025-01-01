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
Process has two mode of operation: user mode and kernel mode
- user mode: upriviledged instruction (add, branch, sub, )
- kernel mode: priviledged instruction (setting up page table, disable clock interrupts)

Virtual memory (memory isolation)
- page table: virtual addr -> physical
- process has own page table

- what should run in kernel mode?
    - monothlic kernel design
        - run all lines in kernel mode
    - micro-kernel design
        - run few line possible in kernel mode
            - FS (File System), Echo, VM System [USER]
            - IPC, multiplexing, virtual memory [KERNEL]
            - implementing using msg (jump in-out of kernel => performance issue)

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


## how the kernel starts and runs the first process.

**1. Power-On & Boot**  
- The system powers on, and a ROM-based boot loader loads the xv6 kernel into memory at `0x80000000`.  
- Execution begins at `_entry` with machine-mode paging disabled.

**2. Kernel Entry & Stack Setup**  
- `_entry` (in `entry.S`) sets up an initial stack (`stack0`) and jumps to `start()` in C.

**3. Machine Mode to Supervisor Mode**  
- `start()` configures key RISC-V registers (timer interrupts, privilege levels, etc.), then uses `mret` to switch to supervisor mode and jump to `main()`.

**4. `main()` & First Process**  
- `main()` (in `main.c`) initializes devices and subsystems, then calls `userinit()`.  
- `userinit()` creates the first user process, which runs `initcode.S`.

**5. First System Call & `/init`**  
- In `initcode.S`, the process loads the `SYS_EXEC` call number into `a7` and invokes `ecall`.  
- The kernel handles the system call (`sys_exec()`), loading `/init` to replace the first process.  
- `/init` then opens the console (as file descriptors 0, 1, 2) and starts the shell. The system is ready to run user commands.
    
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
