---
title: linux 中的系统调用——如何从用户态到内核态
toc: true
---
[toc]

# linux 中的系统调用
## 工作原理
* 整体流程  
用户态程序将必要的参数传入到特定寄存器，然后通过出发软中断进入内核态，接着执行软中断的中断处理函数，该处理函数将根据特定寄存器的内容判断要做哪些事情，做完后，就返回并切换到用户态程序。  
* 中断处理举例  
这个软中断处理函数的名字叫 system_call()，它的实现与具体的硬件体系结构紧密相关。  
在 x86-64 的系统中，预定义的软中断是 128, 可通过 int 0x80 指令触发，触发后就会执行 system_call() 处理函数，该函数在 entry_64.S 中用汇编实现。  
除了 int 指令外，x86 处理器还有 sysenter , syscall 指令(参照 [Intel x86 操作码表和参考][Intel x86 操作码表和参考])。

## 从用户程序调用接口到陷入 linux kernel(以 x86_64 架构下的 fopen() 接口为例)
### 从用户程序调用接口到 glibc 库
* 引用头文件: `stdio.h`
* 函数原型
```c
FILE *fopen(const char *restrict pathname, const char *restrict mode);
```
### glibc 库内部调用
用户程序引入 stdio.h 头文件，并开始调用 fopen 接口，在 glibc 库中的调用流程如下：
```c
// 1. -> stdio.h:fopen :
#   define fopen(fname, mode) _IO_new_fopen (fname, mode)

// 2. -> iofopen:__fopen_internal
FILE *
_IO_new_fopen (const char *filename, const char *mode)
{
  return __fopen_internal (filename, mode, 1);
}

// 3. -> iofopen: __fopen_internal
FILE *
__fopen_internal (const char *filename, const char *mode, int is32)
{
    // ...
  if (_IO_file_fopen ((FILE *) new_f, filename, mode, is32) != NULL)
    return __fopen_maybe_mmap (&new_f->fp.file);
    // ...
}

// 4. -> fileops.c:_IO_file_fopen
versioned_symbol (libc, _IO_new_file_fopen, _IO_file_fopen, GLIBC_2_1);

// 5. -> fileops.c:_IO_new_file_fopen
FILE *
_IO_new_file_fopen (FILE *fp, const char *filename, const char *mode,
		    int is32not64)
{
    // ...
  result = _IO_file_open (fp, filename, omode|oflags, oprot, read_write,
			  is32not64);
    // ...
}

// 6. -> fileops.c:_IO_file_open
FILE *
_IO_file_open (FILE *fp, const char *filename, int posix_mode, int prot,
	       int read_write, int is32not64)
{
    // ...
    fdesc = __open (filename, posix_mode | (is32not64 ? 0 : O_LARGEFILE), prot);
  if (fdesc < 0)
    // ...

//7. -> open64.c:__open
strong_alias (__libc_open64, __open)

// 8. -> open64.c:__libc_open64
__libc_open64 (const char *file, int oflag, ...)
{
    // ...
  return SYSCALL_CANCEL (openat, AT_FDCWD, file, oflag | O_LARGEFILE, mode);
    // ...
}
```

到了这里可以看到， fopen 接口是通过 openat 实现的，接下来的操作是通过宏的形式开始的，这部分看着蛮头大的，可以参考 [glibc源码分析][glibc源码分析] 这篇帖子。


```
// 9. -> syscall.h:SYSCALL_CANCEL
# define SYSCALL_CANCEL(...) \
  __SYSCALL_CANCEL_CALL (__VA_ARGS__)

// 10. -> syscall.h:__SYSCALL_CANCEL_CALL
#define __SYSCALL_CANCEL_CALL(...) \
  __SYSCALL_CANCEL_DISP (__SYSCALL_CANCEL, __VA_ARGS__) // 获取参数的个数，并拼接上__SYSCALL_CANCEL，openat 有4 个参数，所以宏展开为__SYSCALL_CANCEL4

// 11. -> syscall.h:__SYSCALL_CANCEL4 
#define __SYSCALL_CANCEL4(name, a1, a2, a3, a4) \
  __syscall_cancel (__SSC (a1), __SSC (a2), __SSC (a3),	__SSC(a4), 0, 0, __SYSCALL_CANCEL7_ARG __NR_##name) //  x86_64 中 openat 的 __NR_openat 被定义为 257

// 12. -> cancellation.c:__syscall_cancel
// 13. -> cancellation.c:__internal_syscall_cancel 
__internal_syscall_cancel (__syscall_arg_t a1, __syscall_arg_t a2, ... __syscall_arg_t nr)
{
...
  if (SINGLE_THREAD_P || !cancel_enabled (ch) || cancel_exiting (ch))
    {
      result = INTERNAL_SYSCALL_NCS_CALL (nr, a1, a2, a3, a4, a5, a6
					  __SYSCALL_CANCEL7_ARCH_ARG7);
                      ...
    }
    ...
}

```
### 从 glibc 库 到 linux kernel
对于 x86_64 架构下的 fopen 来说，在glibc 中调用了一大推接口和宏定义，最终调到 __internal_syscall_cancel 接口，其内部调用宏 INTERNAL_SYSCALL_NCS_CALL, 然后一些转换后调到宏 ，内容容下：
```
// -> internal_syscall6
#undef internal_syscall6
#define internal_syscall6(number, arg1, arg2, arg3, arg4, arg5, arg6) \
({									\
    unsigned long int resultvar;					\
    TYPEFY (arg6, __arg6) = ARGIFY (arg6);			 	\
    TYPEFY (arg5, __arg5) = ARGIFY (arg5);			 	\
    TYPEFY (arg4, __arg4) = ARGIFY (arg4);			 	\
    TYPEFY (arg3, __arg3) = ARGIFY (arg3);			 	\
    TYPEFY (arg2, __arg2) = ARGIFY (arg2);			 	\
    TYPEFY (arg1, __arg1) = ARGIFY (arg1);			 	\
    register TYPEFY (arg6, _a6) asm ("r9") = __arg6;			\
    register TYPEFY (arg5, _a5) asm ("r8") = __arg5;			\
    register TYPEFY (arg4, _a4) asm ("r10") = __arg4;			\
    register TYPEFY (arg3, _a3) asm ("rdx") = __arg3;			\
    register TYPEFY (arg2, _a2) asm ("rsi") = __arg2;			\
    register TYPEFY (arg1, _a1) asm ("rdi") = __arg1;			\
    asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number), "r" (_a1), "r" (_a2), "r" (_a3), "r" (_a4),		\
      "r" (_a5), "r" (_a6)						\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
    (long int) resultvar;						\
})
```
至此，glibc 中的调用就算结束了，通过上面的处理后，就将需要的数据(系统调用号，参数等)存放在相应的寄存器中，然后执行 `syscall` 指令陷入内核，经由内核处理。

## I think
我一直都是喜欢刨根问底的性子，但很多时候由于知识储备有限只能对纠结的问题含恨而终，对于linux的系统调用如何从用户态切换到内核态的问题也是如此。而这次，又尝试着走了一边，算是一定程度上弥补了这个缺憾。当然这次流程走的也很粗糙，但也算是个好的开始。


[glibc源码分析]: https://zhuanlan.zhihu.com/p/28984642
[Intel x86 操作码表和参考]: https://shell-storm.org/x86doc/
