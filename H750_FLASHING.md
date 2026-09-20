# STM32H750 固件编译与烧录教程（ST-Link + OpenOCD）

本文记录一次实际完成的流程：在 Ubuntu 上克隆 [AIMEtherCAT/EcatV2_AX58100_H750_Universal](https://github.com/AIMEtherCAT/EcatV2_AX58100_H750_Universal)，编译 STM32H750 的 ELF 固件，经 ST-LINK/V2.1 + OpenOCD 连接控制板、排查复位暂停失败、写入并校验内部 Flash。

**适用范围**：该项目的 STM32H750（Cortex-M7）固件。本文的路径、链接地址、OpenOCD target 及 Flash 容量不能不经核对就照搬到其他芯片或其他固件工程。

> **安全提示**：Flash 编程会擦除目标扇区中的旧固件及其中可能保存的数据。先备份重要内容，确认连接的是目标控制板，并断开电机/执行机构的动力电源；目标板必须保留正常的逻辑供电。**烧录成功不等于固件能正常启动或能安全驱动电机。**

## 0. 本次环境与实测结果

| 项目 | 本次情况 |
| --- | --- |
| 上游源码 | [AIMEtherCAT/EcatV2_AX58100_H750_Universal](https://github.com/AIMEtherCAT/EcatV2_AX58100_H750_Universal) |
| 本地工作区 | `~/CommonH750/EcatV2_AX58100_H750_Universal` |
| 目标芯片 | STM32H750；OpenOCD 报告 `STM32H74x/75x - Rev: V` |
| 调试器 | ST-LINK/V2.1，USB VID:PID `0483:3752` |
| OpenOCD | `0.11.0` |
| 接口 | SWD，实测调试时钟 `400 kHz` |
| 内部 Flash | `0x08000000` 起，`128 KB`，单 Bank |
| 编译配置 | CMake + Ninja；`Release` |
| ELF 输出 | `build/Release/EcatV2_AX58100_H750_Universal.elf` |

本次实测：`flash write_image erase` 报告写入 **97,728 bytes**，`verify_image` 报告校验 **102,376 bytes**，OpenOCD 退出码 **0**。**这只能证明烧录与校验成功；未在本次日志中确认烧录后应用程序能正常启动或 EtherCAT 已进入 OP。**

## 1. 新建独立工作区并克隆原版源码

不要与之前修改过的工程混用。第一次执行：

```bash
mkdir -p ~/CommonH750
cd ~/CommonH750

git clone https://github.com/AIMEtherCAT/EcatV2_AX58100_H750_Universal.git
cd EcatV2_AX58100_H750_Universal

git remote -v
git status --short --branch
```

如果已经克隆成功，后续直接进入工程，不要再执行 `git clone`：

```bash
cd ~/CommonH750/EcatV2_AX58100_H750_Universal
```

## 2. 安装编译工具并初始化 SOES 子模块

原版工程提供 `CMakePresets.json`、`cmake/gcc-arm-none-eabi.cmake`，GitHub Actions 也使用 ARM GNU 工具链编译 Release。SOES 是 Git 子模块；**普通的 `git clone` 不会自动拉取其内容**。

```bash
sudo apt-get update
sudo apt-get install -y \
    git cmake ninja-build \
    gcc-arm-none-eabi binutils-arm-none-eabi \
    openocd usbutils

cd ~/CommonH750/EcatV2_AX58100_H750_Universal
git submodule update --init --recursive

arm-none-eabi-gcc --version
cmake --version
ninja --version
openocd --version
```

如果子模块下载失败，先检查网络并重新执行 `git submodule update --init --recursive`，不要在依赖缺失的状态下继续编译。

## 3. 编译 Release ELF 并检查产物

```bash
cd ~/CommonH750/EcatV2_AX58100_H750_Universal

cmake --preset Release
cmake --build --preset Release --parallel 4

ELF="build/Release/EcatV2_AX58100_H750_Universal.elf"
test -s "$ELF" && {
    realpath "$ELF"
    ls -lh "$ELF"
    file "$ELF"
    arm-none-eabi-size "$ELF"
}
```

**以 CMake 编译退出码和 ELF 检查共同判断是否编译成功。** 如果 CMake 失败，先修复报错，不要把以前残留的 ELF 当作新产物。

本次生成的 ELF 路径：

```text
/home/hby/CommonH750/EcatV2_AX58100_H750_Universal/build/Release/EcatV2_AX58100_H750_Universal.elf
```

本次的 `file` 输出显示它是 ARM EABI5 的 32 位 ELF。`arm-none-eabi-size` 输出：

```text
   text    data     bss     dec     hex
  97484    4884  116872  219240   35868
```

`.bss` 是未初始化 RAM 段，不应把 `text + data + bss` 的 `dec` 总和直接理解为写入 Flash 的字节数。

## 4. 接线及检测 ST-Link

关断目标板及执行机构动力电源后核对调试线；根据实际 ST-Link 型号和控制板引脚标识接线：

```text
ST-Link          STM32H750 控制板
SWDIO     <-->   SWDIO
SWCLK     --->   SWCLK
GND       -----  GND
VTref     <---   目标板 3.3V 电平参考
NRST      <-->   NRST（如果两端都引出，推荐连接）
```

VTref 是目标电压参考信号；**不要假设 ST-Link 的任意 3.3V 引脚都可以安全给目标板供电**。控制板需按其硬件设计正确供电，ST-Link 与目标板必须共地。

连接 USB 后执行：

```bash
lsusb | grep -Ei 'ST-Link|STMicroelectronics|0483:375'
openocd --version
```

本次看到：

```text
Bus 003 Device 003: ID 0483:3752 STMicroelectronics ST-LINK/V2.1
Open On-Chip Debugger 0.11.0
```

`lsusb` 识别到 ST-Link 只证明 USB 侧连通，**不能单独证明 SWD 已经连接到 MCU**。如出现 `LIBUSB_ERROR_ACCESS`，再检查 USB 设备权限或临时用 `sudo openocd ...` 诊断；不要把整套构建命令都改用 sudo。

## 5. 检查 OpenOCD 与 CPU 连接

原版工程有 `stm32h750b-disco.cfg`，引用 `interface/stlink.cfg` 和 `target/stm32h7x.cfg`，并包含 `reset_config none`。这里首先使用直接指定接口及目标的形式，以便看清实际配置。

**先只做调试连接测试，不烧录：**

```bash
cd ~/CommonH750/EcatV2_AX58100_H750_Universal

openocd \
    -f interface/stlink.cfg \
    -f target/stm32h7x.cfg \
    -c "adapter speed 400" \
    -c "init" \
    -c "halt" \
    -c "targets" \
    -c "shutdown"
```

如果看到类似 `stm32h7x.cpu0 ... halted`，说明可以通过 SWD 暂停 CPU。**这不等于现有固件正在正常运行**。

### 本次遇到的故障：`reset halt` 超时

本次先尝试过 `adapter speed 1000` + `reset halt`，得到：

```text
Error: timed out while waiting for target halted
TARGET: stm32h7x.cpu0 - Not halted
```

之后降低到 **400 kHz**，改为 **`halt`（不先复位）**，成功停止 CPU：

```text
target halted due to debug-request, current mode: Handler HardFault
xPSR: 0x01000003 pc: 0xfffffffe msp: 0xffffffd8
stm32h7x.cpu0 ... halted
```

含义：ST-Link/SWD 已可连接并暂停 CPU；但**暂停时旧程序处于 HardFault，不能据此判断新固件的运行情况**。如果 `halt` 也失败，检查 SWDIO、SWCLK、共地、供电、NRST 接线，尝试更低的 SWD 频率。不要将 `reset halt` 超时直接解释为芯片或 ST-Link 损坏。

仓库自带配置也可用于连接测试：

```bash
openocd \
    -f ./stm32h750b-disco.cfg \
    -c "adapter speed 400" \
    -c "init" \
    -c "halt" \
    -c "targets" \
    -c "shutdown"
```

注意：修改复位配置并不保证修复暂停问题；能否使用硬件复位还取决于实际 NRST 接线。

## 6. 烧录前只读检查 Flash

**在 CPU 能成功 `halt` 之后**执行：

```bash
cd ~/CommonH750/EcatV2_AX58100_H750_Universal

openocd \
    -f interface/stlink.cfg \
    -f target/stm32h7x.cfg \
    -c "adapter speed 400" \
    -c "init" \
    -c "halt" \
    -c "flash probe 0" \
    -c "flash info 0" \
    -c "shutdown"
```

本次实测关键信息：

```text
Device: STM32H74x/75x
flash size probed value 128
STM32H7 flash has a single bank
Bank (0) size is 128 kb, base address is 0x08000000
#0 : stm32h7x at 0x08000000, size 0x00020000
# 0: 0x00000000 (0x20000 128kB) not protected
```

原版链接脚本 `STM32H750XX_FLASH.ld` 同样定义 `FLASH` 起点 `0x08000000`、大小 `128K`。若实际探测地址、容量、保护状态或芯片不匹配，**停止烧录，先核实硬件与固件是否相符**。

## 7. 烧录 ELF 并回读校验（本次实际成功的方案）

常见的 `program firmware.elf verify reset exit` 会调用包含复位初始化的流程。本次 `reset halt` 曾失败，因此采用 **`init → halt → flash write_image erase → verify_image`**，跳过烧录前的复位暂停；不自动复位目标。

⚠️ **此步骤会擦除并改写目标 Flash。** 仅在第 6 步确认目标正确、已备份重要固件/数据、执行机构安全时执行。`flash write_image erase` 会擦除受影响的整个 Flash 扇区，扇区中不属于新镜像的旧数据也可能丢失。

```bash
cd ~/CommonH750/EcatV2_AX58100_H750_Universal

ELF="build/Release/EcatV2_AX58100_H750_Universal.elf"

if [ ! -s "$ELF" ]; then
    echo "错误：ELF 不存在或为空，停止烧录"
else
    openocd \
        -f interface/stlink.cfg \
        -f target/stm32h7x.cfg \
        -c "adapter speed 400" \
        -c "init" \
        -c "halt" \
        -c "flash write_image erase $ELF" \
        -c "verify_image $ELF" \
        -c "shutdown"

    echo "OpenOCD 退出码：$?"
fi
```

本次成功日志的关键内容：

```text
Device: STM32H74x/75x
Bank (0) size is 128 kb, base address is 0x08000000
Warn : Adding extra erase range, 0x08017dc0 .. 0x0801ffff
Warn : no flash bank found for address 0x30000000
auto erase enabled
wrote 97728 bytes from file ... .elf
verified 102376 bytes
shutdown command invoked
OpenOCD 退出码：0
```

**如何解读：**

- `wrote ... bytes`：OpenOCD 报告完成 Flash 写入。
- `verified ... bytes`：OpenOCD 报告镜像校验通过；与写入量不同不一定意味着校验失败，判断时应结合完整日志和退出码。
- `Adding extra erase range`：为 Flash 擦除扇区边界补齐范围；不能假设扇区里其他旧内容会保留。
- `no flash bank found for address 0x30000000`：本次 ELF 中涉及 RAM 地址（`0x30000000`），而不是可编程的 Flash Bank。本次写入和校验仍成功；如果其他工程出现类似警告，仍需检查 ELF 段地址与链接脚本，不应无条件忽略。
- **仅看到 `shutdown command invoked` 并不足以证明烧录成功**；必须同时检查写入、校验和退出码。

本例为 ELF 文件，不必额外手动指定 `0x08000000`；链接脚本和 ELF 已包含相关加载地址。

## 8. 烧录完成后复位与运行检查

上面的成功方案**没有**自动重启目标程序。烧录成功后，保持执行机构动力电源断开，使用控制板的物理 RESET 按键，或将控制板**逻辑电源**断电再上电，使 MCU 重新启动。

观察控制板的实际 LED/状态输出。原版工程的 [README](https://github.com/AIMEtherCAT/EcatV2_AX58100_H750_Universal/blob/main/readme.md) 记录了 ESC 初始化和 EtherCAT 状态指示灯的含义；LED 与业务功能仍需结合硬件现场验证。

需要再次通过 ST-Link 检查 CPU 时，可在复位后执行以下**只读诊断（会暂停正在运行的 CPU）**：

```bash
openocd \
    -f interface/stlink.cfg \
    -f target/stm32h7x.cfg \
    -c "adapter speed 400" \
    -c "init" \
    -c "halt" \
    -c "reg pc" \
    -c "reg msp" \
    -c "reg xpsr" \
    -c "targets" \
    -c "shutdown"
```

若再次看到 `current mode: Handler HardFault`，可以继续读取故障状态寄存器：

```bash
openocd \
    -f interface/stlink.cfg \
    -f target/stm32h7x.cfg \
    -c "adapter speed 400" \
    -c "init" \
    -c "halt" \
    -c "mdw 0xE000ED28 1" \
    -c "mdw 0xE000ED2C 1" \
    -c "mdw 0xE000ED34 1" \
    -c "mdw 0xE000ED38 1" \
    -c "shutdown"
```

依次对应 CFSR、HFSR、MMFAR、BFAR；MMFAR/BFAR 只有在对应有效位成立时才能作为有效的故障地址。**不能仅用 PC 是否以 `0x0800` 开头判断程序是否运行正常**，更应检查异常状态、任务、外设及 EtherCAT 工作状态。上述调试会暂停 CPU，检查完毕后需按硬件安全流程重新启动。

## 9. 快速备忘（仅适用于已确认硬件与 Flash 的相同环境）

```bash
# 进入已克隆的原版工程
cd ~/CommonH750/EcatV2_AX58100_H750_Universal

# 首次初始化及编译
git submodule update --init --recursive
cmake --preset Release
cmake --build --preset Release --parallel 4

# 检查 ELF 与 ST-Link
test -s build/Release/EcatV2_AX58100_H750_Universal.elf
lsusb | grep -Ei 'ST-Link|STMicroelectronics|0483:375'

# 非破坏性的 MCU/Flash 检测
openocd -f interface/stlink.cfg -f target/stm32h7x.cfg \
    -c "adapter speed 400" -c "init" -c "halt" \
    -c "flash probe 0" -c "flash info 0" -c "shutdown"

# 确认芯片、Flash 容量、备份与执行机构安全后，再执行第 7 节烧录命令
```

## 参考资料

- [上游工程及源码](https://github.com/AIMEtherCAT/EcatV2_AX58100_H750_Universal)
- [原版 CMake 构建预设](https://github.com/AIMEtherCAT/EcatV2_AX58100_H750_Universal/blob/main/CMakePresets.json)
- [原版 GitHub Actions 编译流程](https://github.com/AIMEtherCAT/EcatV2_AX58100_H750_Universal/blob/main/.github/workflows/build.yml)
- [原版 Flash 链接脚本](https://github.com/AIMEtherCAT/EcatV2_AX58100_H750_Universal/blob/main/STM32H750XX_FLASH.ld)
- [原版 OpenOCD 配置](https://github.com/AIMEtherCAT/EcatV2_AX58100_H750_Universal/blob/main/stm32h750b-disco.cfg)
- [本仓库通用 ST-Link + OpenOCD 教程（以 STM32G431 为例）](./STLINK_OPENOCD.md)
