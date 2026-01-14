# Hvisor 虚拟机间通信协议规范

## 基本协议

基本协议是指所有通信协议都需要遵循的规范。

## 配置⽂件格式

每个参与通信的zone必须在⾃⼰的配置⽂件中增加共享内存区域的配置，例如：

```shell
"ivc_configs": [
    {
        "ivc_id": 0,
        "peer_id": 1,
        "control_table_ipa": "0xd0000000",
        "shared_mem_ipa": "0xd0001000",
        "rw_sec_size": "0",
        "out_sec_size": "0x1000",
        "interrupt_num": 66,
        "max_peers": 2
    }，
],
```

`ivc_configs`是⼀个数组，每个元素规定⼀个通信区域，⼀个zone最多可以有2个通信区域：
- ivc_id：inter vm communication id，⼀起通信的zones的ivc_id必须相同。
- peer_id：每个参与通信的zone称为peer，这⾥指定peer的编号。
- control_table_ipa：control table的IPA起始地址。
- shared_mem_ipa：共享内存IPA起始地址。
- rw_sec_size：read/write section的⼤⼩。
- out_sec_size：output section的⼤⼩
- interrupt_num：该zone通信时⽤到的中断号，注意该中断号需要添加到配置⽂件中的
interrupts属性中
- max_peers：该通信区域中，参与通信的zone数量的最⼤值。

## 控制表格式

控制表⽤于`zone`参与⼀个通信区域的通信。虚拟机通过`hypercall`可获取共享内存的地址、控制表的地址和对应的`ivc_id`。


```shell
// 以下为只读字段
ivc_id：u32，指⽰该控制表所属的ivc id
max_peers： u32，指⽰ivc id下的通信区域最多⽀持的peer数量
RW Sec Size： u32，指⽰Read/Write Section⼤⼩
Out Sec Size：u32，指⽰Output Section⼤⼩
peer id：u32，该zone在该通信区域下的peer id
// 以下为只写字段
ipi_invoke：u32，写⼊peer id，向该peer发送中断通知
```

## 共享内存格式

共享内存的基本格式：
```shell
+-----------------------------+ -
|                             | :
| Output Section for peer n-1 | : Output Section Size
| (n = Maximum Peers)         | :
+-----------------------------+ -
:                             :
:                             :
:                             :
+-----------------------------+ -
|                             | :
|  Output Section for peer 1  | : Output Section Size
|                             | :
+-----------------------------+ -
|                             | :
   Output Section for peer 0    : Output Section Size
|                             | :
+-----------------------------+ -
|                             | :
|  Read/Write Section         | : RW Section Size
|                             | :
+-----------------------------+   <-- Shared memory base address
```

共享内存分有两个区域：`RW Section`和`Output Section`。

- RW Section：对所有peers均可读写，⼤⼩由配置⽂件指定，可以为0。可⽤于多个peer通信时互
斥机制。
- Output Section：每个peer均有⼀个Output Section，且⼤⼩相同，由配置⽂件指定⼤⼩。Output Section for peer x对于peer x可读可写，对于其他peers只读，⽤于peer x向其他peer发送
数据使⽤。


## 虚拟机与共享内存交互机制

虚拟机需要执⾏`id`为`5`的`hypercall`，获取该虚拟机可进⾏通信的信息，格式为：

```shell
struct ivc_info {
    __u64 len; // 以下各个数组的实际⻓度，也是通信域的个数
    __u64 ivc_ct_ipas[CONFIG_MAX_IVC_CONFIGS]; // control table 控制表的ipa
    __u64 ivc_shmem_ipas[CONFIG_MAX_IVC_CONFIGS]; // 共享内存ipa
    __u32 ivc_ids[CONFIG_MAX_IVC_CONFIGS]; // 通信域id，ivc id
    __u32 ivc_irqs[CONFIG_MAX_IVC_CONFIGS]; // 各个通信域下，该zone对应的物理中断号
}__attribute__((packed));
```

之后通过控制表`control table`，即可获取通信的相关信息进⾏通信。


## 运行示例

`root linux` 中`ivc`配置如下：

```shell
//可根据需要在对应平台下的board.rs添加如下配置，这⾥以qemu-gicv3为例
//hvisor/platform/aarch64/qemu-gicv3/board.rs
pub const ROOT_ZONE_IRQS: [u32; 9] = [33, 64, 77, 79, 35, 36, 37, 38, 65];//添加中断号65
    pub const ROOT_ZONE_IVC_CONFIG: [HvIvcConfig; 1] = [
    HvIvcConfig {
    ivc_id: 0,
    peer_id: 0,
    control_table_ipa: 0xd0000000,
    shared_mem_ipa: 0xd0001000,
    rw_sec_size: 0,
    out_sec_size: 0x1000,
    interrupt_num: 65,
    max_peers: 2,
    },
];

```
在`zone0`的设备树中添加`hvisor_ivc_device`节点，设备树中的中断号要与`ivc`配置中的`interrupt_num`⼀致，这⾥为`0x21`是因为ARM架构要加上⼤⼩为`32`的偏移（0x21+32=65）：

```shell
hvisor_ivc_device {
    compatible = "hvisor";
    interrupt-parent = <0x01>;
    interrupts = <0x00 0x21 0x01>;
};
```

`non-root linux`中`ivc`配置如下：
```shell
//在zone1的虚拟机配置⽂件添加`ivc`配置，以`zone1-linux.json`为例
"interrupts": [66, 76, 78], //添加中断号66
"ivc_configs": [
    {
    "ivc_id": 0,
    "peer_id": 1,
    "control_table_ipa": "0xd0000000",
    "shared_mem_ipa": "0xd0001000",
    "rw_sec_size": "0",
    "out_sec_size": "0x1000",
    "interrupt_num": 66,
    "max_peers": 2
    }
],
```
在`zone1`的设备树中添加`hvisor_ivc_device`节点，设备树中的中断号要与`ivc`配置中的`interrupt_num`⼀致，这⾥为`0x22`是因为`ARM`架构要加上⼤⼩为`32`的偏移（0x22+32=66）：

```shell
hvisor_ivc_device {
    compatible = "hvisor";
    interrupt-parent = <0x01>;
    interrupts = <0x00 0x22 0x01>;
};
```

在运⾏`hvisor-tool/tools/ivc_demo`测试`ivc`功能时，需要分别在`root linux`和`non-root linux`中执⾏`insmod ivc.ko`创建`/dev/hivc0`设备
