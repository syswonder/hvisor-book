# VirtIO-SCMI

## 概述

VirtIO-SCMI 是 hvisor 中实现的一套虚拟化框架，通过 VirtIO 传输层在 Root Zone（Zone0）与 Non-Root Zone（ZoneU）之间传递 SCMI（System Control and Management Interface）协议消息。它允许 Non-Root Zone 安全地访问受控的硬件资源，包括时钟、复位域和电源域。

### 设计目标

传统硬件直通方式将物理设备直接分配给特定虚拟机，存在资源独占和安全风险。VirtIO-SCMI 通过虚拟化层提供统一的资源访问抽象：

- Zone0 掌握真实的硬件资源
- ZoneU 通过 VirtIO-SCMI 协议请求资源操作
- hvisor-tool 负责消息转发和访问控制

### 支持的协议

| 协议 ID | 协议名称 | 功能 |
|---------|----------|------|
| 0x10 | Base | 版本查询、协议列表、厂商信息 |
| 0x11 | Power Domain | 电源域属性查询、电源状态控制 |
| 0x14 | Clock | 时钟属性、频率管理、时钟启用/禁用 |
| 0x16 | Reset | 复位域属性、复位操作 |

## 整体架构

```
ZoneU (Linux)
  ┌──────────────────────┐
  │  Clock/Reset/Power   │  SCMI 客户端驱动
  │  (SCMI Protocol)     │
  └────────┬─────────────┘
           │ VirtIO Queue
  ┌────────▼─────────────┐
  │   hvisor-tool        │  VirtIO 设备模拟
  │  virtio_scmi +       │  SCMI 协议处理
  │  scmi_core +         │
  │  clock/power/reset   │
  └────────┬─────────────┘
           │ ioctl (ko_fd)
  ┌────────▼─────────────┐
  │  Zone0 Linux         │
  │  hvisor.ko           │  SCMI Server
  │  (SCMI 驱动)          │  硬件操作代理
  └────────┬─────────────┘
           │
  ┌────────▼─────────────┐
  │  硬件 (Clock/Reset/  │
  │  Power Controller)   │
  └──────────────────────┘
```

### 用户态工具层（hvisor-tool）

位于 `tools/virtio/devices/scmi/`，负责 VirtIO 设备模拟和 SCMI 协议处理：

| 文件 | 功能 |
|------|------|
| virtio_scmi.c | VirtIO 设备入口，处理队列中断与描述符链 |
| scmi_core.c | SCMI 协议核心，per-device 协议管理、响应上下文、统一 ioctl |
| base.c | Base 协议实现（版本、厂商、协议列表） |
| power.c | Power Domain 协议实现 |
| clock.c | Clock 协议实现 |
| reset.c | Reset 协议实现 |

### 内核驱动层（hvisor.ko）

集成在 Root Zone 的 hvisor 内核模块中，负责实际硬件操作：

- 通过 `of_clk_get` / `__of_reset_control_get` / `genpd_dev_pm_attach_by_id` 获取 Linux 子系统句柄
- 执行真实的时钟频率设置、复位操作、电源状态切换
- 通过 ioctl 与用户态工具通信

### SCMI 消息格式

SCMI 使用打包消息头（Packed Message Header）：

```
  31          28 27          18 17      10 9    8 7             0
 +--------------+--------------+----------+------+---------------+
 |   Reserved   |   Token ID   |ProtocolID| Type |  Message ID   |
 +--------------+--------------+----------+------+---------------+
```

| 字段 | 位宽 | 说明 |
|------|------|------|
| Message ID | 8 bits | 协议内消息标识 |
| Message Type | 2 bits | 0=Command, 2=Delayed Response, 3=Notification |
| Protocol ID | 8 bits | 协议标识 |
| Token ID | 10 bits | 请求/响应配对标识 |

### 响应上下文

VirtIO-SCMI 使用 `scmi_resp_ctx` 响应上下文追踪响应写入状态，替代直接操作 raw iovec。上下文记录底层响应缓冲、已写入字节数和总容量：

- `scmi_resp_ctx_init()` — 从响应 iovec 初始化上下文
- `scmi_resp_write()` — 安全写入响应数据（带溢出检测）
- `scmi_make_response()` — 写入 SCMI 响应头 + 状态码

VirtIO Used Ring 更新时使用 `ctx.written` 而非 guest 提供的缓冲区长度，保证通知 guest 的字节数与实际写入一致。

## 虚拟 ID ABI

### 资源映射机制

每个 SCMI 设备实例维护三组 ID 数组，定义于 `SCMIDev` 结构体：

```c
struct virtio_scmi_config {
    /* 预留：virtio 规范规定 virtio-scmi 无配置数据（"no configuration data"），
     * 此成员仅满足框架"设备结构体首成员为配置结构体"的要求 */
    uint64_t reserved[8];
};

struct scmi_dev_protocol_entry {
    uint8_t protocol_id;
    int (*handler)(struct virtio_scmi_dev *dev, uint8_t msg_id,
                   uint16_t token, const struct iovec *req_iov,
                   struct scmi_resp_ctx *ctx);
};

typedef struct virtio_scmi_dev {
    struct virtio_scmi_config config; /* 首成员：VirtIO 配置空间 */
    uint32_t *clock_ids;              /* 物理时钟 DT 索引数组，下标为 SCMI 虚拟 ID */
    uint32_t clock_count;
    uint32_t *reset_ids;              /* 物理复位 DT 索引数组 */
    uint32_t reset_count;
    uint32_t *power_ids;              /* 物理电源域 DT 索引数组 */
    uint32_t power_count;
    struct scmi_dev_protocol_entry protocols[SCMI_MAX_PROTOCOLS];
    int protocol_count;
} SCMIDev;
```

`protocols` 为 per-device 协议注册表，注册规则见"协议注册"节。

映射规则：数组**下标**为 SCMI 虚拟 ID，数组**元素值**为物理资源在 Zone0 hvisor 设备树节点中的 DT 索引。

```
SCMI 虚拟 ID 0 → clock_ids[0] = 5 → of_clk_get(hvisor_node, 5)
SCMI 虚拟 ID 1 → clock_ids[1] = 9 → of_clk_get(hvisor_node, 9)
```

### JSON 配置

设备配置通过 `virtio_cfg.json` 定义：

```json
{
    "type": "scmi",
    "addr": "0xff9c0000",
    "len": "0x200",
    "irq": "0x44",
    "status": "enable",
    "clock_ids": [0, 1, 2, 3, 4, 5, 6, 7],
    "reset_ids": [0, 1, 2, 3, 4, 5],
    "power_ids": [0, 1, 2]
}
```

配置项说明：

| 配置项 | 类型 | 说明 |
|--------|------|------|
| type | string | 设备类型 "scmi" |
| addr | string | VirtIO-MMIO 基地址 |
| len | string | MMIO 区域大小 |
| irq | string | 中断号（全局中断号，等于 GIC SPI 号加 32） |
| status | string | 设备状态，"enable" 表示启用 |
| clock_ids | array | 物理时钟 DT 索引数组，下标为 SCMI 虚拟 ID |
| reset_ids | array | 物理复位 DT 索引数组 |
| power_ids | array | 物理电源域 DT 索引数组 |

### 单 Zone 配置

单个 Non-Root Zone 使用唯一的 SCMI VirtIO 设备，所有资源由该 Zone 独占或共享：

```json
{
    "zones": [
        {
            "id": 1,
            "devices": [
                {
                    "type": "scmi",
                    "addr": "0xff9c0000",
                    "len": "0x200",
                    "irq": "0x44",
                    "status": "enable",
                    "clock_ids": [0, 1, 2, 3, 4, 5, 6, 7],
                    "reset_ids": [0, 1, 2, 3, 4, 5],
                    "power_ids": [0, 1, 2]
                }
            ]
        }
    ]
}
```

ZoneU 设备树中，`virtio_mmio` 节点指向该 SCMI 设备的 MMIO 地址，`clocks`/`resets`/`power-domains` 的 cell 值即为 `clock_ids`/`reset_ids`/`power_ids` 数组的下标（SCMI 虚拟 ID）。

### 多 Zone 配置

多个 Non-Root Zone 各自拥有独立的 SCMI VirtIO 设备，每个设备使用不同的 MMIO 地址和中断号，且可以指定不同的资源子集：

```json
{
    "zones": [
        {
            "id": 1,
            "devices": [
                {
                    "type": "scmi",
                    "addr": "0xff9c0000",
                    "len": "0x200",
                    "irq": "0x44",
                    "status": "enable",
                    "clock_ids": [0, 1, 2, 3, 4, 5, 6, 7],
                    "reset_ids": [0, 1, 2, 3, 4, 5],
                    "power_ids": [0, 1, 2]
                }
            ]
        },
        {
            "id": 2,
            "devices": [
                {
                    "type": "scmi",
                    "addr": "0xffa10000",
                    "len": "0x200",
                    "irq": "0x66",
                    "status": "enable",
                    "clock_ids": [8, 9, 10, 11],
                    "reset_ids": [0],
                    "power_ids": [3]
                }
            ]
        }
    ]
}
```

ZoneU 各自设备树中，`virtio_mmio` 节点地址分别对应 `0xff9c0000` 和 `0xffa10000`，各自引用各自的 SCMI 虚拟 ID。

Zone0 hvisor 设备树节点的 `clocks`/`resets`/`power-domains` 属性需包含所有 Zone 需要的物理资源。各 Zone 通过 ID 数组选择自己能访问的子集，从而实现资源隔离：

```
Zone0 hvisor DT node clocks 属性: [clk0, clk1, clk2, clk3, clk4, ...]
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
              Zone1 clock_ids          Zone2 clock_ids     未分配
              [0, 1] → clk0, clk1      [2, 3, 4] → clk2-4  安全隔离
```

### 设备树配置

#### Root Zone 设备树

Root Zone 的设备树需在 hvisor 节点中列出所有由 SCMI 管理的硬件资源。`clocks`/`resets`/`power-domains` 属性中的每个条目对应一个 DT 索引（从 0 开始），该索引被 `virtio_cfg.json` 中的 ID 数组引用：

```dts
/* Root Zone 的 hvisor 设备节点 */
hvisor_virtio_device {
    compatible = "hvisor";
    interrupts = <0x00 0x20 0x01>;

    /* DT 索引 0 → clk 控制器，参数 3
     * DT 索引 1 → clk 控制器，参数 5  */
    clocks = <&clk 3>, <&clk 5>;

    /* DT 索引 0 → rst 控制器，参数 523 */
    resets = <&rst 523>;

    /* DT 索引 0 → power 控制器，参数 2
     * DT 索引 1 → power 控制器，参数 7  */
    power-domains = <&power 2>, <&power 7>;
};
```

`virtio_cfg.json` 中的 ID 数组值即为该节点中对应属性的 DT 索引：

| JSON 配置 | 含义 | 最终解析结果 |
|-----------|------|-------------|
| `"clock_ids": [0, 1]` | SCMI 虚拟 ID 0 → DT 索引 0 | `of_clk_get(hvisor_node, 0)` → clk 控制器, 参数 3 |
| | SCMI 虚拟 ID 1 → DT 索引 1 | `of_clk_get(hvisor_node, 1)` → clk 控制器, 参数 5 |
| `"reset_ids": [0]`   | SCMI 虚拟 ID 0 → DT 索引 0 | `__of_reset_control_get(np, NULL, 0, ...)` → rst 控制器, 参数 523 |
| `"power_ids": [0, 1]` | SCMI 虚拟 ID 0 → DT 索引 0 | `genpd_dev_pm_attach_by_id(&pdev->dev, 0)` → power 控制器, 参数 2 |

DT 索引和控制器参数的区别：

- **DT 索引**：`clocks`/`resets`/`power-domains` 属性中条目的序号（从 0 开始），在 hvisor 节点中定义
- **控制器参数**：每个条目中的 cell 值（如 rst 控制器的 `523`），由对应的硬件控制器驱动程序解释，对 SCMI 层透明

#### ZoneU 设备树

Non-Root Zone 的设备树定义 SCMI VirtIO 传输通道和协议节点。SCMI 协议节点中的 cell 值为 **SCMI 虚拟 ID**，对应 `virtio_cfg.json` 中 ID 数组的下标：

```dts
/* ZoneU 设备树 */
firmware {
    scmi {
        compatible = "arm,scmi-virtio";
        #address-cells = <1>;
        #size-cells = <0>;

        scmi_clk: protocol@14 {
            reg = <0x14>;
            #clock-cells = <1>;
        };

        scmi_rst: protocol@16 {
            reg = <0x16>;
            #reset-cells = <1>;
        };

        scmi_pwr: protocol@11 {
            reg = <0x11>;
            #power-domain-cells = <1>;
        };
    };
};

/* VirtIO-MMIO 传输通道，地址/中断与 virtio_cfg.json 对应。
 * 注意：interrupts 中为 GIC SPI 号（0x24 = 36），
 * 而 virtio_cfg.json 的 "irq": "0x44"（68）为全局中断号，两者相差 32。 */
virtio_mmio@ff9c0000 {
    compatible = "virtio,mmio";
    reg = <0x0 0xff9c0000 0x0 0x200>;
    interrupt-parent = <&gic>;
    interrupts = <0x0 0x24 0x1>;
    dma-coherent;
};
```

ZoneU 内的设备通过标签引用 SCMI 资源，cell 值为 SCMI 虚拟 ID：

```dts
&ethernet {
    clocks = <&scmi_clk 0>;        /* SCMI 虚拟 ID 0 → clock_ids[0] → 物理 clk 3 */
    resets = <&scmi_rst 0>;        /* SCMI 虚拟 ID 0 → reset_ids[0]  → 物理 rst 523 */
    power-domains = <&scmi_pwr 0>; /* SCMI 虚拟 ID 0 → power_ids[0] → 物理 power 2 */
};
```

#### 完整映射链路

以 reset 为例，从 ZoneU 设备到物理硬件的完整映射路径：

```
ZoneU ethernet 节点
  resets = <&scmi_rst 0>
       │
       ▼ SCMI 虚拟 ID 0
virtio_cfg.json reset_ids[0] = 0
       │
       ▼ DT 索引 0
Zone0 hvisor_virtio_device 节点
  resets = <&rst 523>
       │
       ▼ 控制器参数 523
rst 复位控制器 → 硬件复位信号 #523
```

### 协议注册

协议处理函数按设备注册。在 JSON 配置解析阶段，根据各资源数组长度决定是否注册对应协议：

```
scmi_dev_register_protocol(dev, SCMI_PROTO_ID_BASE,   base_handler);
if (clock_count > 0)
    scmi_dev_register_protocol(dev, SCMI_PROTO_ID_CLOCK, clock_handler);
if (reset_count > 0)
    scmi_dev_register_protocol(dev, SCMI_PROTO_ID_RESET, reset_handler);
if (power_count > 0)
    scmi_dev_register_protocol(dev, SCMI_PROTO_ID_POWER, power_handler);
```

## 消息处理流程

### 请求处理

```
ZoneU
  │ 写入 VirtIO TX 队列
  ▼
hvisor-tool virtq_tx_handle_one_request()
  │ 解析 SCMI 消息头 (protocol_id, msg_id, token)
  │ 初始化 scmi_resp_ctx
  ▼
scmi_handle_message()
  │ per-device 协议表查找
  ▼
clock/power/reset handler
  │ 验证 ID (clk_phys_id/pwr_phys_id/rst_phys_id)
  │ 写入 ioctl 参数 (物理 DT 索引)
  ▼
hvisor_scmi_ioctl_cmd()
  │ ioctl(ko_fd, ...)
  ▼
Zone0 hvisor.ko
  │ of_clk_get / clk_set_rate / genpd_dev_pm_attach_by_id
  ▼
硬件操作完成，返回结果
```

### 关键实现要点

- **描述符链解析**：通过 `process_descriptor_chain_buf()` 将描述符链解析为 `VirtioRequest`，可读缓冲（请求）进入 `out_iov`、可写缓冲（响应）进入 `in_iov`；SCMI 请求要求恰好一个可读缓冲和一个可写缓冲
- **消息头解析**：请求缓冲首 4 字节为打包的 SCMI 消息头，通过位域结构体 `scmi_msg_header` 直接解析出 protocol_id / msg_id / msg_type / token
- **协议分发**：`scmi_handle_message()` 在 per-device 协议表中按 protocol_id 查找处理器并分发
- **响应完成**：协议处理完成后，以 `ctx.written`（实际写入字节数）更新 VirtIO Used Ring；非法请求以长度 0 完成，坏描述符被消费并以长度 0 完成，避免请求永久阻塞队列

## Zone0 驱动集成

Zone0 的 hvisor 内核驱动模块在初始化时通过 `hvisor_scmi_init()` 扫描 hvisor 设备树节点，注册对各子系统资源的访问能力。初始化依次处理时钟、复位、电源域三个子系统，任一步失败即回滚已初始化的子系统并返回错误；模块卸载时由 `hvisor_scmi_cleanup()` 统一释放资源。

驱动程序提供以下关键能力：

- **时钟缓存**：在 `clock_init()` 中通过 `of_count_phandle_with_args` 获取时钟数量，运行时按需通过 `of_clk_get` 懒加载并缓存时钟句柄
- **复位缓存**：通过 `__of_reset_control_get` 获取复位控制器句柄并进行缓存
- **电源域**：通过 `genpd_dev_pm_attach_by_id` 绑定电源域控制器

### Server 依赖的 Linux API 表

| 子系统 | API | 引入版本 | 用途 |
|--------|-----|---------|------|
| **Clock** | `clk_prepare_enable` | 3.1 | 启用时钟 |
| | `clk_disable_unprepare` | 3.1 | 禁用时钟 |
| | `clk_get_rate` | 2.6 | 获取时钟频率 |
| | `clk_set_rate` | 2.6 | 设置时钟频率 |
| | `__clk_is_enabled` | 3.1 | 查询时钟启用状态 |
| **Reset** | `__of_reset_control_get` | 4.0 (5 参) / 5.2 (6 参) | 获取复位控制器句柄 |
| | `__fwnode_reset_control_get` | 7.1 | 获取复位控制器句柄（新内核） |
| | `reset_control_assert` | 4.0 | 断言复位 |
| | `reset_control_deassert` | 4.0 | 去断言复位 |
| | `reset_control_reset` | 4.0 | 触发复位脉冲 |
| | `reset_control_acquire` | 5.2 | 获取复位控制权 |
| | `reset_control_release` | 5.2 | 释放复位控制权 |
| | `reset_control_put` | 4.0 | 释放复位句柄 |
| **Power Domain** | `genpd_dev_pm_attach_by_id` | 5.1 | 通过索引绑定电源域 |
| | `dev_pm_domain_detach` | 4.11 | 解绑电源域 |
| | `pm_runtime_resume_and_get` | 5.10 | 恢复电源域（≥5.10） |
| | `pm_runtime_get_sync` | 2.6.32 | 恢复电源域（<5.10 fallback） |
| | `pm_runtime_put_sync` | 2.6.32 | 暂停电源域 |
| | `pm_runtime_active` | 2.6.36 | 查询电源域状态 |

版本兼容性说明：

- **Linux 5.4**：`genpd_dev_pm_attach_by_id`（≥5.1）可用；`pm_runtime_resume_and_get`（≥5.10）走 `#else` fallback 到 `pm_runtime_get_sync`；`__of_reset_control_get` 走 6 参版本（≥5.2）
- **Linux 6.1**：所有 API 完整可用；`__fwnode_reset_control_get` 走 7.1 新路径
- **老内核**：如需支持 <5.1 的内核，需为 `genpd_dev_pm_attach_by_id` 添加 fallback（回退到 `of_genpd_add_device`）

## 配置与使用

### 前提条件

1. Root Zone 设备树中需包含 `hvisor_virtio_device` 节点，其 `clocks`、`resets`、`power-domains` 属性列出所有交由 SCMI 管理的硬件资源
2. hvisor 内核模块需加载并正常运行

### 构建

```bash
make ARCH=arm64 LOG=LOG_INFO KDIR=/path/to/linux-kernel
```

### 启动

```bash
# 启动 VirtIO 守护进程
nohup ./hvisor virtio start /path/to/virtio_cfg.json &

# 启动 Non-Root Zone
./hvisor zone start /path/to/zone.json
```

## 注意事项

1. ID 数组中的值必须为有效的 DT 索引，范围为 `[0, N)`，其中 N 为 Zone0 hvisor 节点对应属性的条目数
2. SCMI 虚拟 ID 不能超过对应 ID 数组的长度减一
3. Non-Root Zone 设备树中的 SCMI 协议 cell 值即为虚拟 ID，与 ID 数组下标一一对应
4. Zone0 的 hvisor 设备树节点需要包含所有 Non-Root Zone 可能访问的资源

> 本文档描述协议与配置层面的约定。实现细节以 hvisor-tool 仓库 `tools/virtio/devices/scmi/` 目录下的源码为准。
