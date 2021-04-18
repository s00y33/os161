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
