# 虚拟pcie设备

### hvisor支持的虚拟设备列表

### 添加新的设备
hvisor通过 `VpciDeviceHandler` 以支持自定义的虚拟pcie设备，目前已测试在ecam pcie系统中添加虚拟设备
```
pub trait VpciDeviceHandler: Sync + Send {
    fn read_cfg(
        &self,
        dev: ArcRwLockVirtualPciConfigSpace,
        offset: PciConfigAddress,
        size: usize,
    ) -> HvResult<PciConfigAccessStatus>;
    fn write_cfg(
        &self,
        dev: ArcRwLockVirtualPciConfigSpace,
        offset: PciConfigAddress,
        size: usize,
        value: usize,
    ) -> HvResult<PciConfigAccessStatus>;
    fn vdev_init(&self, dev: VirtualPciConfigSpace) -> VirtualPciConfigSpace;
}
```

添加新设备需要
- 在`VpciDevType`中注册新的设备类型
- 为该设备类型实现`VpciDeviceHandler`

hvisor实现了一个标准的设备`StandardVdev`供参考