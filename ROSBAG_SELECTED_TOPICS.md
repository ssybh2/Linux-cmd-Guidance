# ROS 2 rosbag：录制指定话题、重复录制多次

本教程适用于 ROS 2 Humble / Ubuntu：**只录制自己指定的话题**，每次实验单独保存一个 rosbag，随后查看录制结果。示例来自 STM32H750 + EtherCAT 从站 `sn2883650`；换成其他工程时，修改话题名和工作空间路径即可。

> **重要**：录制前先启动发布对应话题的节点（例如 EtherCAT 主站），并确认话题确实在发布消息。rosbag 是录制工具，不会自行启动 IMU、遥控器或电机控制程序。

## 1. 加载 ROS 环境，确认话题

按实际路径修改工作空间；以下路径是本例：

```bash
cd /home/hby/one/ZLT
source /opt/ros/humble/setup.bash
source install/setup.bash

ros2 topic list | grep '/ecat/sn2883650/'
```

本例需要录制的四个话题：

| 应用 | 数据 | 话题 |
| --- | --- | --- |
| app1 | 遥控器 | `/ecat/sn2883650/app1/read` |
| app2 | CAN1 IMU | `/ecat/sn2883650/app2/read` |
| app3 | CAN2 IMU | `/ecat/sn2883650/app3/read` |
| app4 | DShot 指令 | `/ecat/sn2883650/app4/write` |

需要时逐个检查消息类型、发布者、频率：

```bash
ros2 topic info /ecat/sn2883650/app1/read
ros2 topic type /ecat/sn2883650/app2/read
ros2 topic hz /ecat/sn2883650/app2/read
# 观察完持续运行的 hz 命令后按 Ctrl+C
```

**注意**：如果 RC → DShot 控制节点运行在 `dry_run:=true`，它不会发布 app4 的真实 DShot 消息；此时 rosbag 可以记录前三个话题，但 app4 的消息数可能为 0。**不要仅为了让 rosbag 有 app4 数据而启用电机输出。**

## 2. 录制指定的四个话题（每次生成新文件夹）

建议把 bag 保存到工作空间外的独立目录，不要放在 `src/` 下：

```bash
mkdir -p /home/hby/one/rosbags

# 带日期、时间和纳秒的名字，避免多次录制时目录名重复
BAG_DIR="/home/hby/one/rosbags/h750_$(date +%Y%m%d_%H%M%S_%N)"

echo "本次 rosbag：$BAG_DIR"

ros2 bag record \
    -o "$BAG_DIR" \
    /ecat/sn2883650/app1/read \
    /ecat/sn2883650/app2/read \
    /ecat/sn2883650/app3/read \
    /ecat/sn2883650/app4/write
```

终端运行期间就是正在录制。**完成本次实验后按一次 `Ctrl+C`，等待 rosbag 退出并写完元数据**，不要直接拔电或强行杀进程。

示例目录（实际文件名随 rosbag 存储格式变化）：

```text
/home/hby/one/rosbags/
├── h750_20260920_180000_123456789/
│   ├── metadata.yaml
│   └── h750_20260920_180000_123456789_0.db3
└── h750_20260920_181500_987654321/
    ├── metadata.yaml
    └── h750_20260920_181500_987654321_0.db3
```

## 3. 录制第二次、第三次……

**重新执行第 2 节中从 `BAG_DIR=...` 开始的命令**。每运行一次，`date` 都会生成新的目录名称，不会覆盖之前的实验。

如果需要频繁操作，可一次性创建可重复使用的脚本：

```bash
cat > /home/hby/one/record_h750.sh <<'EOF'
#!/usr/bin/env bash
set -e

source /opt/ros/humble/setup.bash
source /home/hby/one/ZLT/install/setup.bash

BAG_ROOT="/home/hby/one/rosbags"
mkdir -p "$BAG_ROOT"

# 使用日期时间和纳秒避免碰撞；目录已存在时 rosbag 也会拒绝覆盖
BAG_DIR="$BAG_ROOT/h750_$(date +%Y%m%d_%H%M%S_%N)"

echo "开始录制：$BAG_DIR"
echo "按 Ctrl+C 结束当前录制；等命令结束后再次运行脚本即可录下一次。"

ros2 bag record \
    -o "$BAG_DIR" \
    /ecat/sn2883650/app1/read \
    /ecat/sn2883650/app2/read \
    /ecat/sn2883650/app3/read \
    /ecat/sn2883650/app4/write
EOF

chmod +x /home/hby/one/record_h750.sh
```

以后每次实验，在新终端运行：

```bash
/home/hby/one/record_h750.sh
```

录制结束按 `Ctrl+C`，等待返回命令提示符。下一次**重新运行同一脚本**即可。需要录制其他话题时，修改脚本末尾的四行话题路径。

## 4. 查看有哪些录制与是否真的记录到消息

```bash
ls -lht /home/hby/one/rosbags
```

挑选**实际存在**的某次目录查看，例如：

```bash
ros2 bag info /home/hby/one/rosbags/h750_20260920_180000_123456789
```

重点检查录制时长、每个话题的消息类型和消息数量。某个话题不存在、没有发布者或没有消息时，该话题可能无法录到有效数据。

若只想查看最近一次的目录（目录名按时间排序）：

```bash
BAG_DIR="$(find /home/hby/one/rosbags -mindepth 1 -maxdepth 1 -type d -name 'h750_*' | sort | tail -n 1)"
if [ -n "$BAG_DIR" ]; then
    echo "$BAG_DIR"
    ros2 bag info "$BAG_DIR"
else
    echo "还没有找到 h750_* 录制目录"
fi
```

> 如果你以 `root` 运行录制，生成的文件可能归 root 所有；以后用 `hby` 用户分析时若遇到 `Permission denied`，先检查文件归属，不要为了方便而放宽整个工作空间的权限。

## 5. 可选：打包为 ZIP 方便传到 Windows

**先结束所有 rosbag 录制进程**，避免压缩到尚未写完的数据库。然后：

```bash
cd /home/hby/one
sudo apt-get install -y zip
zip -r "rosbags_$(date +%Y%m%d_%H%M%S).zip" rosbags/
ls -lh rosbags_*.zip
```

建议每次生成**新名字**的 ZIP，避免重复复用旧压缩包后误以为里面含有刚录制的数据。后续可按本仓库的 [Python HTTP 文件服务器与 Windows 下载教程](./PYTHON_HTTP_WINDOWS_DOWNLOAD.md) 传输 ZIP。

## 6. 常见问题

- **`ros2: command not found`**：先 `source /opt/ros/humble/setup.bash`；确认 ROS 2 已安装。
- **没有指定的话题**：检查主站/传感器节点是否已启动、终端是否 source 同一 ROS 2 环境及 `ROS_DOMAIN_ID` 是否一致。
- **DShot 话题消息数为 0**：`dry_run:=true` 时这是预期行为；不应通过启动电机来满足录包要求。
- **`Output folder already exists`**：不要删除旧录制；重新执行时间戳命令生成新目录。
- **上一条录制没结束**：等待 `Ctrl+C` 正常收尾；不要在相同输出目录并行启动第二个 recorder。

参考：[ROS 2 Humble ros2 bag 命令与录制指南](https://docs.ros.org/en/humble/Tutorials/Beginner-CLI-Tools/Recording-And-Playing-Back-Data/Recording-And-Playing-Back-Data.html)。
