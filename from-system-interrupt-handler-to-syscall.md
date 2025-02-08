---
title: 从中断处理到内核中的系统调
toc: true
---
[toc]

# 从中断处理到内核中的系统调

## 基本原理
* 中断处理  
用户态程序陷入内核后，中断处理函数负责根据用户态程序传出的数据，选择相应的处理程序。
* 系统调用表  
系统调用表是一个函数指针数组，存储各个内核实现了的系统调用的函数指针，可以通过该表查到内核实现的系统调用函数。
* 返回用户态  
内核态代码处理完相应事务后，执行特定汇编指令，使处理回到用户态模式。

用户在调用系统调用接口后，通过特定指令陷入到内核，此时和系统调用相关的中断处理函数被触发，中断处理函数通过传入的信息，在系统调用表中找到相应的系统调用函数，并开始执行。处理完成后在执行相应的汇编指令，返回到用户态。

## 从中断处理函数到具体的函数调用接口(以 arm64 架构 openat 为例)
### 中断处理
中断发生后，中断处理函数会到中断向量表中找到相应的处理程序  
#### 中断向量表
中断向量表用于存放中断处理函数的地址。
```s
/*
 * Exception vectors.
 */
	.pushsection ".entry.text", "ax"

	.align	11
SYM_CODE_START(vectors)
	kernel_ventry	1, t, 64, sync		// Synchronous EL1t
	kernel_ventry	1, t, 64, irq		// IRQ EL1t
	kernel_ventry	1, t, 64, fiq		// FIQ EL1t
	kernel_ventry	1, t, 64, error		// Error EL1t

	kernel_ventry	1, h, 64, sync		// Synchronous EL1h
	kernel_ventry	1, h, 64, irq		// IRQ EL1h
	kernel_ventry	1, h, 64, fiq		// FIQ EL1h
	kernel_ventry	1, h, 64, error		// Error EL1h

	kernel_ventry	0, t, 64, sync		// Synchronous 64-bit EL0
	kernel_ventry	0, t, 64, irq		// IRQ 64-bit EL0
	kernel_ventry	0, t, 64, fiq		// FIQ 64-bit EL0
	kernel_ventry	0, t, 64, error		// Error 64-bit EL0

	kernel_ventry	0, t, 32, sync		// Synchronous 32-bit EL0
	kernel_ventry	0, t, 32, irq		// IRQ 32-bit EL0
	kernel_ventry	0, t, 32, fiq		// FIQ 32-bit EL0
	kernel_ventry	0, t, 32, error		// Error 32-bit EL0
SYM_CODE_END(vectors)
```
`kernel_ventry` 宏负责定义中断向量，例如定义从用户态切花至内核态的中断向量：
```s
// 定义中断向量 el0t_64_sync
kernel_ventry	0, t, 64, sync		// Synchronous 64-bit EL0`
```

#### 中断处理函数
中断处理函数汇编代码定义：
```S
// el: 运行级别
// ht: 异常或中断类型
// regsize: 寄存器大小
// label: 标识符

	.macro entry_handler el:req, ht:req, regsize:req, label:req
SYM_CODE_START_LOCAL(el\el\ht\()_\regsize\()_\label)
	kernel_entry \el, \regsize
	mov	x0, sp
	bl	el\el\ht\()_\regsize\()_\label\()_handler
	.if \el == 0
	b	ret_to_user
	.else
	b	ret_to_kernel
	.endif
SYM_CODE_END(el\el\ht\()_\regsize\()_\label)
	.endm
```
对于系统调用来说，会在中断向量表中找到中断向量 `el0t_64_sync`，然后通过该向量进入中断处理函数 `el0t_64_sync_handler`
### 系统调用表
#### 系统调用表的定义  

```c
// 系统调用表的大小
#define __NR_syscalls 451

// 系统调用表初始化
#undef __SYSCALL
// 在 #include <asm/unistd.h> 中调用，用以为 sys_call_table 的元素赋值
#define __SYSCALL(nr, sym)	[nr] = __arm64_##sym,

const syscall_fn_t sys_call_table[__NR_syscalls] = {
    // gcc 扩展语法，将 sys_call_table[] 中的所有元素用 __arm64_sys_ni_syscall 填充
	[0 ... __NR_syscalls - 1] = __arm64_sys_ni_syscall,
// 嵌套地 include 头文件，大致是为 sys_call_table 赋有效值的，即使用有效的系统调用地址填充该表
#include <asm/unistd.h>
};
```

#### 系统调用表赋有效值  

在定义系统调用表时，其引入了头文件(`#include <asm/unistd.h`)，里面都一些宏定义，然后又嵌套的引入了其他的头文件。

其中被嵌套引用的还有 `include/uapi/asm-generic/unistd.h`，在该头文件中使用了宏 `__SYSCALL` ，这是一条赋值语句，指定了系统调用表中系统调用编号和内核实现的系统调用的函数名之间的关系, 对于 `openat`：  
```c
// 系统调用编号
#define __NR_openat 56

// 指定系统调用号对应的函数
__SYSCALL(__NR_openat, sys_openat)
```
宏展开后为：
```c
[56] = __arm64_sys_openat,
```

如此可见，在系统调用表中和 `openat` 相关的系统调用就是 `sys_call_table[__NR_openat]`，即 `sys_call_table[56]`，而这个数组元素指向 `__arm64_sys_openat`

### 内核系统调用的实现
* 内核中系统调用接口的实现使用了宏, 对于 `openat`：  
```c
SYSCALL_DEFINE4(openat, int, dfd, const char __user *, filename, int, flags,
		umode_t, mode)
{
	if (force_o_largefile())
		flags |= O_LARGEFILE;
	return do_sys_open(dfd, filename, flags, mode);
}

SYSCALL_DEFINE4(openat, int, dfd, const char __user *, filename, int, flags, umode_t, mode)
```

SYSCALL_DEFINE4 调用流程如下：
```c
#define SYSCALL_DEFINE4(name, ...) SYSCALL_DEFINEx(4, _##name, __VA_ARGS__)

// ->
#define SYSCALL_DEFINEx(x, sname, ...)				\
	SYSCALL_METADATA(sname, x, __VA_ARGS__)			\
	__SYSCALL_DEFINEx(x, sname, __VA_ARGS__)

// ->
#define __SYSCALL_DEFINEx(x, name, ...)						\
	asmlinkage long __arm64_sys##name(const struct pt_regs *regs);		\
	ALLOW_ERROR_INJECTION(__arm64_sys##name, ERRNO);			\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__));		\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__));	\
	asmlinkage long __arm64_sys##name(const struct pt_regs *regs)		\
	{									\
		return __se_sys##name(SC_ARM64_REGS_TO_ARGS(x,__VA_ARGS__));	\
	}									\
	static long __se_sys##name(__MAP(x,__SC_LONG,__VA_ARGS__))		\
	{									\
		long ret = __do_sys##name(__MAP(x,__SC_CAST,__VA_ARGS__));	\
		__MAP(x,__SC_TEST,__VA_ARGS__);					\
		__PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));		\
		return ret;							\
	}									\
	static inline long __do_sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```

宏展开后的结果
```c
// 函数声明 
asmlinkage long __arm64_sys_openat(const struct pt_regs *regs);		
ALLOW_ERROR_INJECTION(__arm64_sys_openat, ERRNO);			
static long __se_sys_openat(__MAP(x,__SC_LONG,__VA_ARGS__));		
static inline long __do_sys_openat(__MAP(x,__SC_DECL,__VA_ARGS__));	

// 函数定义
asmlinkage long __arm64_sys_openat(const struct pt_regs *regs)		
{									
	return __se_sys_openat(SC_ARM64_REGS_TO_ARGS(x,__VA_ARGS__));	
}									
static long __se_sys_openat(__MAP(x,__SC_LONG,__VA_ARGS__))		
{									
	long ret = __do_sys_openat(__MAP(x,__SC_CAST,__VA_ARGS__));	
	__MAP(x,__SC_TEST,__VA_ARGS__);					
	__PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));		
	return ret;							
}									
static inline long __do_sys_openat(__MAP(x,__SC_DECL,__VA_ARGS__))
{
	if (force_o_largefile())
		flags |= O_LARGEFILE;
	return do_sys_open(dfd, filename, flags, mode);
}
```
可以看出，对于 `openat` 来说，通过 `SYSCALL_DEFINE4` 定义了三个函数：
* 全局函数：__arm64_sys_openat，其实现依赖 __se_sys_openat;
* 静态函数：__se_sys_openat, 其实现依赖 __do_sys_openat
* 内联函数：__do_sys_openat

最关键的就是 **__do_sys_openat** 了，其在实现时依赖 do_sys_open 接口，如此就开始调用更深层次的内核接口了。


### 内核系统调流程
##### 触发中断陷入内核
用户程序调用特定指令(`svc 0`)陷入到内核;
#### 中断处理函数被触发
中断被触发后，cpu 切换至内核模式，从中断向量表中找到向量 `el0t_64_sync`, 该向量指向 `el0t_64_sync_handler`, 所以跳到 `el0t_64_sync_handler` 开始执行:
```c
asmlinkage void noinstr el0t_64_sync_handler(struct pt_regs *regs)
{
	unsigned long esr = read_sysreg(esr_el1);

	switch (ESR_ELx_EC(esr)) {
    // 系统调用相关
	case ESR_ELx_EC_SVC64:
		el0_svc(regs);
		break;
	}
    ...
}
```
#### 进入系统调用相关的处理程序
进入中断处理程序后，开始判断中断类型，并根据中断类型进入相应的处理流程，对于系统调用来说，开始执行 `el0_svc` 函数：
```c
static void noinstr el0_svc(struct pt_regs *regs)
{
	enter_from_user_mode(regs);
	cortex_a76_erratum_1463225_svc_handler();
	do_el0_svc(regs);
	exit_to_user_mode(regs);
}

// -> do_el0_svc(regs);

// -> el0_svc_common, 传入 系统调用表，可以从表中找到相应的系统调用接口
el0_svc_common(regs, regs->regs[8], __NR_syscalls, sys_call_table);

// -> invoke_syscall
static void invoke_syscall(struct pt_regs *regs, unsigned int scno,
			   unsigned int sc_nr,
			   const syscall_fn_t syscall_table[])
{
    ...
	if (scno < sc_nr) {
		syscall_fn_t syscall_fn;
        // 通过系统调用号和系统调用表找到相应的系统调用处理函数
		syscall_fn = syscall_table[array_index_nospec(scno, sc_nr)];
        // 执行系统调用代码
		ret = __invoke_syscall(regs, syscall_fn);
	} else {
		ret = do_ni_syscall(regs, scno);
	}
}
```
#### 返回用户态
中断处理函数汇编代码定义：
```S

// el: 运行级别
// ht: 异常或中断类型
// regsize: 寄存器大小
// label: 标识符

	.macro entry_handler el:req, ht:req, regsize:req, label:req
	bl	el\el\ht\()_\regsize\()_\label\()_handler
    ...
	.if \el == 0
	b	ret_to_user
	.else
	b	ret_to_kernel
	.endif
SYM_CODE_END(el\el\ht\()_\regsize\()_\label)
	.endm
```
由中断处理函数的定义可知，在中断处理函数处理结束之后，会运行`b ret_to_user` 切换到用户态:
```S
SYM_CODE_START_LOCAL(ret_to_user)
	ldr	x19, [tsk, #TSK_TI_FLAGS]	// re-check for single-step
	enable_step_tsk x19, x2
#ifdef CONFIG_GCC_PLUGIN_STACKLEAK
	bl	stackleak_erase_on_task_stack
#endif
	kernel_exit 0
SYM_CODE_END(ret_to_user)
```
ret_to_user 将会调用 kernel_exit :
```S
	.macro	kernel_exit, el
    ...
    eret
    ...
```
进而调用 `eret` 回到用户态，完美。
