# 如何启动Non Root RTOS

## 在RK3588上启动AArch64/AArch32 Zephyr虚拟机

目前hvisor支持在RK3588 PC开发板上，启动Zephyr作为Non Root RTOS虚拟机。Zephyr可以以两种模式启动：AArch32和AArch64。

具体启动与移植相关的文档，请见：[在rk3588上通过hvisor启动64/32位zephyr](https://blog.syswonder.org/#/2025/20250531_Zephyr_on_hvisor)

## 在NXP imx8mp上启动Xiuos

NXP imx8mp开发板上可在M核上启动Xiuos，并通过RPMsg与Linux通信，具体流程请见：[在NXP上通过RPMsg实现Linux与Xiuos通信](https://blog.syswonder.org/#/2025/20250217_RPMSG_on_NXP)
