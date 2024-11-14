# Hypercall说明

hvisor作为Hypervisor，向上层虚拟机提供hypercall处理机制。

## 虚拟机如何执行Hypercall

虚拟机通过执行指定的汇编指令，在Arm64为`hvc`，在riscv64为`ecall`。执行汇编指令时，传入的参数分别为：

* code：hypercall id，其范围和含义详见hvisor对hypercall的处理
* arg0：虚拟机要传递的第一个参数，类型为u64
* arg1：虚拟机要传递的第二个参数，类型为u64

例如，对于riscv linux：

```c
#ifdef RISCV64

// according to the riscv sbi spec
// SBI return has the following format:
// struct sbiret
//  {
//  long error;
//  long value;
// };

// a0: error, a1: value
static inline __u64 hvisor_call(__u64 code,__u64 arg0, __u64 arg1) {
	register __u64 a0 asm("a0") = code;
	register __u64 a1 asm("a1") = arg0;
	register __u64 a2 asm("a2") = arg1;
	register __u64 a7 asm("a7") = 0x114514;
	asm volatile ("ecall"
	        : "+r" (a0), "+r" (a1)
			: "r" (a2), "r" (a7)
			: "memory");
	return a1;
}
#endif
```

对于arm64 linux：

```c
#ifdef ARM64
static inline __u64 hvisor_call(__u64 code, __u64 arg0, __u64 arg1) {
	register __u64 x0 asm("x0") = code;
	register __u64 x1 asm("x1") = arg0;
	register __u64 x2 asm("x2") = arg1;

	asm volatile ("hvc #0x4856"
	        : "+r" (x0)
			: "r" (x1), "r" (x2)
			: "memory");
	return x0;
}
#endif /* ARM64 */
```

## hvisor对hypercall的处理

当虚拟机执行hypercall后，CPU会进入hvisor指定的异常处理函数：`hypercall`。之后hvisor根据hypercall传入的参数`code`、`arg0`、`arg1`，继续调用不同的处理函数，分别为：

| code | 调用函数             | 参数说明                                                   | 函数简介                                      |
| ---- | -------------------- | ---------------------------------------------------------- | --------------------------------------------- |
| 0    | hv_virtio_init       | arg0：共享内存起始地址                                     | 用于root zone初始化virtio跳板机制             |
| 1    | hv_virtio_inject_irq | 无                                                         | 用于root zone将virtio设备中断发送给其他虚拟机 |
| 2    | hv_zone_start        | arg0：虚拟机配置文件地址；arg1：配置文件大小               | 用于root zone启动一个虚拟机                   |
| 3    | hv_zone_shutdown     | arg0：要关闭的虚拟机id                                     | 用于root zone关闭一个虚拟机                   |
| 4    | hv_zone_list         | arg0：表示虚拟机信息的数据结构地址；arg1：虚拟机信息的数量 | 用于root zone查看整个系统所有虚拟机信息       |
| 5    | hv_ivc_info          | arg0：ivc信息的起始地址                                    | 用于一个zone查看自己所在的通信域信息          |



