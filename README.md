# Linux Command Guidance

这个仓库用于整理 Linux / ROS 2 开发中常用的命令与排错教程。

## 教程目录

### [ST-Link + OpenOCD 固件烧录教程](./STLINK_OPENOCD.md)

Ubuntu 下使用 ST-Link 和 OpenOCD 给 STM32 烧录 `.elf` 固件，包括连接检测、烧录、校验和常见错误排查。

### [Linux `chown` 工作空间权限修复教程](./CHOWN.md)

讲解 `chown` 的命令格式，以及如何修复 ROS 2 工作空间、Git 仓库、`build/install/log` 被 root 占有后出现的 `Permission denied` 问题。

---

以后新增教程时，也可以继续以独立 `.md` 文件的形式放在仓库中，并在这里添加入口链接。
