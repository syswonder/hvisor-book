# 在龙芯3A6000主板（7A2000）上启动 hvisor

韩喻泷 <wheatfox17@icloud.com>

更新时间：2025.3.3

请参考文档中 3A5000 的流程，需要注意，在编译 hvisor 时需要指定对应的板卡为 3A6000，目前对 3A6000 新板卡的支持仍在开发中，请从 <https://github.com/enkerewpo/hvisor> 进行代码克隆。

```bash
git clone https://github.com/enkerewpo/hvisor
cd hvisor
make ARCH=loongarch64 FEATURES=platform_3a6000
```