# System Calls and Processes

## Code Reading Questions

### `kern/syscall` Existing Process Support
1.  What are the ELF magic numbers?
	>Magic numbers are a sequence of bytes in the beginning of a file, used to identify or verify the content of a file. ELF magic numbers consists of 0x7F followed by **ELF** ( 45 4c 46 ) in ASCII.

2.  What is the difference between `UIO_USERISPACE` and `UIO_USERSPACE`? When should one use `UIO_SYSSPACE` instead?
	>`UIO_USERISPACE` , `UIO_USERSPACE` and `UIO_SYSSPACE` are defined in [kern/include/uio.h](https://github.com/s00y33/os161/blob/master/kern/include/uio.h#L67). They represent user process code segment, user process data segment, and kernel segment respectively.

3.  Why can the `struct uio` that is used to read in a segment be allocated on the stack in `load_segment`? Or, put another way, where does the memory read actually go?
	>The memory read is at the virtual address `VADDR`. The segment being read does not go into the stack of `load_segment` because `struct uio` only stores the pointer to the data block (`uio.iov`).

4.  In `runprogram` why is it important to call `vfs_close` before going to user mode?
	>`struct vnode` is an abstract representation of a file. It has a reference count `vn_refcount` initialized to 1 and guarded by a spinlock. When destroying an abstract vnode, the [`vnode_cleanup`](https://github.com/s00y33/os161/blob/master/kern/vfs/vnode.c#L64) function requires that the reference count is exactly 1, i.e, no process is using it. In `runprogram`, `vfs_close` is called before switching to user mode because the user program's execution does not return except on error. If a symmetric `vfs_close` is not called for the `vfs_open` called in `runprogram`, the file's resources will not be released and its reference count will never be decremented. As a result, the `vnode` accessed by the user program can never be cleaned up.

5.  What function forces the processor to switch into user mode? Is this function machine dependent?
	>[`mips_usermode`](https://github.com/s00y33/os161/blob/master/kern/arch/mips/locore/trap.c#L368) is a machine dependent function called by `enter_new_process` to switch from kernel mode to user mode.

6.  In what files are `copyin`, `copyout`, and `memmove` defined? Why are `copyin` and `copyout` necessary? (as opposed to just using `memmove`)
	>`copyin` and `copyout` are defined in [kern/vm/copyinout.c](https://github.com/s00y33/os161/blob/master/kern/vm/copyinout.c). `memmove`, on the other hand, is defined in [common/libc/string/memmove.c](https://github.com/s00y33/os161/blob/master/common/libc/string/memmove.c). `copyin` and `copyout` are necessary for copying data across the kernel-userland boundary. For safety reasons, they both call the [`copycheck`](https://github.com/s00y33/os161/blob/master/kern/vm/copyinout.c#L118) function to ensure that the the block of user memory provided does not overflow to the kernel address space.

7.  What is the purpose of `userptr_t`?
	>It distinguishes a pointer in the kernel space from a pointer in the user space for type safety.


### `kern/arch/mips` Traps and System Calls
1.  What is the numerical value of the exception code for a MIPS system call?
    >[8.](https://github.com/s00y33/os161/blob/master/kern/arch/mips/include/trapframe.h#L89)

2.  How many bytes is an instruction in MIPS? Try to answer this by reading `syscall` carefully, not by looking somewhere else.
	>[4 bytes](https://github.com/s00y33/os161/blob/master/kern/arch/mips/syscall/syscall.c#L141), the amount the program counter is advanced after syscall returns.

3.  Why do you probably want to change the implementation of `kill_curthread`?
	>The `kill_curthread` implementation panics on error instead of properly killing the thread.

4.  What would be required to implement a system call that took more than 4 arguments?
	>As stated in the comment: "if you run out of registers (which happens quickly with 64-bit values) further arguments must be fetched from the user-level stack, starting at `sp+16` to skip over the slots for the registerized values, with `copyin`".

### Support Code for User Programs
1.  What is the purpose of the SYSCALL macro?
	> The `SYSCALL` macro loads the syscall number into v0, the register the kernel expects to find it in, and jump to the shared syscall code.

2.  What is the MIPS instruction that actually triggers a system call? (Answer this by reading the source in this directory, not looking somewhere else.)
	>The [`syscall`](https://github.com/s00y33/os161/blob/master/userland/lib/libc/arch/mips/syscalls-mips.S#L84) instruction.

3.  OS/161 supports 64 bit values, `lseek` takes and returns a 64 bit offset value. Thus, `lseek` takes a 32 bit file handle (`arg0`), a 64 bit offset (`arg1`), a 32 bit whence (`arg3`), and needs to return a 64 bit offset. In `void syscall(struct trapframe *tf)` where will you find each of the three arguments (in which registers) and how will you return the 64 bit offset?
	>By the  calling conventions for syscalls, the first four 32-bit arguments are passed in argument registers a0-a3, and 64-bit arguments are passed in *aligned* pairs of registers. In this case, `arg0`, `arg1`, and `arg2` will be in registers a1, a2-a3, and `sp+16` respectively. Since the return value is 64-bit, it will be passed back on registers v0 and v1.

## Syscall Documentation

### Rule of Thumb
The user-level interface for system calls are defined in [`userland/include/unistd.h`](https://github.com/s00y33/os161/blob/master/userland/include/unistd.h). In [`kern/include/syscall.h`](https://github.com/s00y33/os161/blob/master/kern/include/syscall.h), we define the prototype of the in-kernel system call handlers. Following the calling conventions for syscalls, a rule thumb when doing so is that the in-kernel implementation should return errno on error, and 0 on success. The actual return value expected by the user-level interface (e.g. count of bytes written in syscall [`write`](https://github.com/s00y33/os161/blob/master/userland/include/unistd.h#L123)) should be returned via an additional argument, where we then pass in the pointer of the [`retval`](https://github.com/s00y33/os161/blob/master/kern/arch/mips/syscall/syscall.c#L100) in the syscall dispatcher. After defining the prototype, we implement the system call `x` in `kern/syscall/x_syscall.c`.

### write

 - libc prototype: `ssize_t write(int fd, const void *buf, size_t buflen);`
 - handler prototype: `int write(int fd, userptr_t *ubuf, size_t buflen, int *retval);`
 - TODO: verify type of buflen and retval in kernel
