# ST-Link + OpenOCD 固件烧录指南

本仓库记录在 **Ubuntu 22.04 / ROS 2 Humble** 环境下，使用 **ST-LINK/V2.1 + OpenOCD** 给 STM32 烧录 `.elf` 固件的完整流程。

当前流程已在以下组合上实际验证：

- Ubuntu 22.04
- ROS 2 Humble
- ST-LINK/V2.1
- STM32G431
- `AIMEtherCAT/hipnucimu`
- OpenOCD 0.11.0

> ROS 2 Humble 和 STM32 烧录本身没有直接关系。烧录由 OpenOCD + ST-Link 完成，因此即使没有 `source /opt/ros/humble/setup.bash`，也不影响烧录。

---

## 1. 安装工具

```bash
sudo apt update
sudo apt install -y openocd usbutils unzip
```

检查 OpenOCD：

```bash
openocd --version
```

示例：

```text
Open On-Chip Debugger 0.11.0
```

---

## 2. 工程目录示例

假设工作空间位于 Linux Home 目录：

```text
/home/<username>/
└── hipnucimu/
    ├── st_nucleo_g4.cfg
    ├── Core/
    ├── Drivers/
    └── build/
        └── Release/
            ├── hipnucimu.elf
            ├── hipnucimu.bin
            └── hipnucimu.hex
```

进入工程：

```bash
cd ~/hipnucimu
```

查看文件：

```bash
ls
```

进入编译产物目录：

```bash
cd ~/hipnucimu/build/Release
```

确认 ELF 存在：

```bash
ls -l hipnucimu.elf
```

---

## 3. 先检测 ST-Link

插入 ST-Link 后执行：

```bash
lsusb | grep -i -E "ST-LINK|STMicroelectronics"
```

成功时可能看到：

```text
Bus 003 Device 004: ID 0483:3752 STMicroelectronics ST-LINK/V2.1
```

这一步只证明：

```text
Ubuntu -> USB -> ST-Link
```

连接正常，**还不能证明 STM32 已经连通**。

---

## 4. STM32G431 的 OpenOCD 配置

`AIMEtherCAT/hipnucimu` 工程中的：

```text
st_nucleo_g4.cfg
```

内容为：

```tcl
source [find interface/stlink.cfg]
source [find target/stm32g4x.cfg]
```

也就是说它同时指定了：

- 调试器：ST-Link
- Target：STM32G4

检查配置文件：

```bash
ls -l ~/hipnucimu/st_nucleo_g4.cfg
```

检查系统 OpenOCD 是否具有 STM32G4 target：

```bash
ls -l /usr/share/openocd/scripts/target/stm32g4x.cfg
```

---

## 5. 烧录前先检测 STM32

推荐先执行连接测试，不要直接烧：

```bash
sudo openocd \
  -f ~/hipnucimu/st_nucleo_g4.cfg \
  -c "adapter speed 1000; init; reset halt; targets; shutdown"
```

连接成功时重点关注：

```text
Info : STLINK V2J... VID:PID 0483:3752
Info : Target voltage: 3.2...
Info : stm32g4x.cpu: hardware has 6 breakpoints, 4 watchpoints
target halted due to debug-request
```

看到：

```text
stm32g4x.cpu
target halted due to debug-request
```

基本说明链路已经打通：

```text
Ubuntu
  |
ST-LINK/V2.1
  |
 SWD
  |
STM32G431
```

---

## 6. 烧录工作空间中编译出的 ELF

进入：

```bash
cd ~/hipnucimu/build/Release
```

确认：

```bash
ls -l hipnucimu.elf
```

烧录：

```bash
sudo openocd \
  -f ../../st_nucleo_g4.cfg \
  -c "adapter speed 1000; init; reset halt; program hipnucimu.elf verify reset exit"
```

也可以不依赖当前路径，使用绝对路径：

```bash
sudo openocd \
  -f ~/hipnucimu/st_nucleo_g4.cfg \
  -c "adapter speed 1000; init; reset halt; program ~/hipnucimu/build/Release/hipnucimu.elf verify reset exit"
```

### ELF 不需要手动指定 Flash 地址

`.elf` 文件中已经包含链接地址，因此这里不需要额外添加：

```text
0x08000000
```

---

## 7. 如何判断烧录成功

烧录日志中重点看：

```text
Programming Started
Programming Finished
Verified OK
```

其中：

- `Programming Finished`：Flash 写入完成
- `Verified OK`：OpenOCD 回读 Flash 后与 ELF 校验一致

看到 `Verified OK` 基本即可确认烧录成功。

---

## 8. 烧录从 GitHub Actions 下载的 ZIP 固件

例如 Home 目录下有：

```text
~/firmware-elf(10).zip
```

文件名中有括号，因此 shell 中建议加引号。

### 8.1 解压

```bash
cd ~
rm -rf ~/firmware-elf-10
mkdir -p ~/firmware-elf-10
unzip 'firmware-elf(10).zip' -d ~/firmware-elf-10
```

### 8.2 查找 ELF

```bash
find ~/firmware-elf-10 -type f -name "*.elf" -print
```

例如：

```text
/home/hby/firmware-elf-10/hipnucimu.elf
```

如果确认压缩包中只有一个 ELF，可以自动获取：

```bash
ELF=$(find ~/firmware-elf-10 -type f -name "*.elf" | head -n 1)
echo "$ELF"
```

### 8.3 再次检测 Target

```bash
sudo openocd \
  -f ~/hipnucimu/st_nucleo_g4.cfg \
  -c "adapter speed 1000; init; reset halt; targets; shutdown"
```

### 8.4 烧录 ZIP 中的 ELF

```bash
sudo openocd \
  -f ~/hipnucimu/st_nucleo_g4.cfg \
  -c "adapter speed 1000; init; reset halt; program \"$ELF\" verify reset exit"
```

---

## 9. 烧完后验证 MCU 是否真正运行

```bash
sudo openocd \
  -f ~/hipnucimu/st_nucleo_g4.cfg \
  -c "adapter speed 1000; init; reset run; sleep 500; halt; reg pc; reg msp; shutdown"
```

正常情况下常见：

```text
PC  -> 0x0800xxxx
MSP -> 0x200xxxxx
```

含义：

- `0x080xxxxx`：STM32 Flash 区域
- `0x200xxxxx`：STM32 SRAM 区域

如果未烧入有效程序，可能会看到类似：

```text
pc:  0xfffffffe
msp: 0xfffffffc
```

---

## 10. 常见错误：`current_target out of bounds`

错误示例：

```bash
openocd \
  -f interface/stlink.cfg \
  -c "transport select hla_swd; adapter speed 1000; init; shutdown"
```

可能得到：

```text
Error: BUG: current_target out of bounds
```

原因是这里只加载了：

```text
interface/stlink.cfg
```

也就是只告诉 OpenOCD 使用 ST-Link，却没有告诉它目标 MCU 是什么。

对于 STM32G431 还需要：

```text
target/stm32g4x.cfg
```

因此使用工程自带配置即可：

```bash
-f ~/hipnucimu/st_nucleo_g4.cfg
```

---

## 11. `Unable to match requested speed` 是否是错误？

可能看到：

```text
Unable to match requested speed 2000 kHz, using 1800 kHz
```

一般不是错误，只是 ST-Link 无法精确产生所请求的时钟，OpenOCD 自动选择了最接近的速度。

只要后续仍然出现：

```text
target halted
Programming Finished
Verified OK
```

通常不需要处理。

---

## 12. 如果突然无法连接 STM32

先把 SWD 速度降低：

```bash
sudo openocd \
  -f ~/hipnucimu/st_nucleo_g4.cfg \
  -c "adapter speed 100; init; reset halt; targets; shutdown"
```

同时检查 SWD 接线：

```text
ST-Link             STM32

SWDIO   ----------  SWDIO
SWCLK   ----------  SWCLK
GND     ----------  GND
VREF    ----------  3.3V/VREF
NRST    ----------  NRST      # 可选，但推荐
```

注意：

- `lsusb` 能识别 ST-Link，只说明电脑到 ST-Link 正常
- 必须看到 `stm32g4x.cpu` 和 `target halted`，才能确认 ST-Link 到 MCU 也正常

---

## 13. 最短版操作备忘录

### 1）检测 ST-Link

```bash
lsusb | grep -i -E "ST-LINK|STMicroelectronics"
```

### 2）检测 STM32G431

```bash
sudo openocd \
  -f ~/hipnucimu/st_nucleo_g4.cfg \
  -c "adapter speed 1000; init; reset halt; targets; shutdown"
```

### 3）烧录 ELF

```bash
sudo openocd \
  -f ~/hipnucimu/st_nucleo_g4.cfg \
  -c "adapter speed 1000; init; reset halt; program /你的/固件路径/firmware.elf verify reset exit"
```

工作空间版本：

```bash
cd ~/hipnucimu/build/Release

sudo openocd \
  -f ../../st_nucleo_g4.cfg \
  -c "adapter speed 1000; init; reset halt; program hipnucimu.elf verify reset exit"
```

---

## 14. 一条命令理解整个过程

```text
OpenOCD
  |
  +-- interface/stlink.cfg      -> 使用 ST-Link
  |
  +-- target/stm32g4x.cfg       -> 目标 STM32G4
  |
  +-- reset halt                -> 复位并暂停 MCU
  |
  +-- program firmware.elf      -> 写入 Flash
  |
  +-- verify                    -> 回读校验
  |
  +-- reset                     -> 复位运行
  |
  +-- exit                      -> 退出 OpenOCD
```

---

## 参考工程

- `AIMEtherCAT/hipnucimu`
- MCU：STM32G431
- OpenOCD 配置：`st_nucleo_g4.cfg`

> 如果换成 STM32F4、H7、G0 等其他 MCU，需要把 target 配置换成对应的 OpenOCD target，不能直接照搬 `stm32g4x.cfg`。
