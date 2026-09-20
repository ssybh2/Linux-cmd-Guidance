# Python 3 临时 HTTP 文件服务器：Ubuntu 文件在 Windows 浏览器下载

本文说明如何让 Ubuntu 临时提供一个 ZIP 下载链接，并从同一网络中的 Windows 电脑用 Edge/Chrome 下载。**Python 3 标准库自带 `http.server`，不需要安装额外 Web 框架。**

示例文件：

```text
/home/hby/one/rosbags.zip
```

目标是用 Windows 下载这个 ZIP。实际使用时请根据自己的文件位置修改路径。

> **安全提醒**：`python3 -m http.server` 是方便的临时文件共享方式，**没有账号、密码或 HTTPS**。知道服务器 IP、能连上对应端口的人都可能访问共享目录中的文件。仅在可信网络短时间使用；**不要直接共享整个工作空间、家目录或含有密钥的目录**，不要把端口暴露到公网。下载完成后按 `Ctrl+C` 关闭服务器。

## 1. 准备专门的下载目录，只放需要分享的文件

先检查 ZIP 确实存在：

```bash
ls -lh /home/hby/one/rosbags.zip
```

**强烈建议先创建独立共享目录**，不要直接把 `/home/hby/one` 作为 HTTP 根目录：

```bash
SHARE_DIR="/home/hby/one/http_download"
mkdir -p "$SHARE_DIR"
cp /home/hby/one/rosbags.zip "$SHARE_DIR/rosbags.zip"
ls -lh "$SHARE_DIR"
```

仅放需要下载的文件；不要放私人配置、SSH 密钥、工作空间源码或其他不准备共享的数据。若 ZIP 是旧版本，请先结束 rosbag 录制，重新生成 ZIP，然后**重新复制到共享目录**。

如果还没有 ZIP，可以参照 [指定话题 rosbag 录制教程](./ROSBAG_SELECTED_TOPICS.md) 的打包章节。注意：不要在 rosbag 数据库还在写入时压缩它。

## 2. Ubuntu 启动临时 HTTP 文件服务器

检查 Python 3：

```bash
python3 --version
```

**在 Ubuntu 终端运行**：

```bash
python3 -m http.server 8000 \
    --bind 0.0.0.0 \
    --directory /home/hby/one/http_download
```

显示类似：

```text
Serving HTTP on 0.0.0.0 port 8000 ...
```

表示 HTTP 服务正在监听 8000 端口。此终端要保持运行；**`0.0.0.0` 是监听所有 IPv4 网络接口，不是 Windows 浏览器中要输入的访问地址**。

如果想仅绑定到可访问的局域网接口，可把 `--bind 0.0.0.0` 改为 `--bind <Ubuntu 的局域网 IP>`。无论哪种方式，都只应在可信网络使用。

## 3. 查找 Ubuntu 的正确 IP 地址

**另开一个 Ubuntu 终端**：

```bash
hostname -I
ip -br -4 addr
```

`hostname -I` 可能输出多个地址，请选 **Windows 能访问的那块网卡的 IPv4 地址**。例如，如果 Ubuntu 局域网地址是 `192.168.1.100`，则实际下载链接为：

```text
http://192.168.1.100:8000/rosbags.zip
```

请替换 `192.168.1.100` 为本机查询得到的地址，**不要在 Windows 输入 `0.0.0.0`**。

如果 Windows 与 Ubuntu 在不同子网，或 Ubuntu 在 NAT/虚拟机中，它们可能无法直接互访，需要先解决路由、桥接或端口转发问题；仅启动 Python 并不能自动打通网络。

## 4. 在 Windows 浏览器打开并下载

在 Windows 的 Edge / Chrome 地址栏输入：

```text
http://<Ubuntu-IP>:8000/rosbags.zip
```

例如（仅作示例）：

```text
http://192.168.1.100:8000/rosbags.zip
```

回车后浏览器会下载 `rosbags.zip`。也可以先在浏览器中打开 `http://<Ubuntu-IP>:8000/` 查看共享目录中的文件，但这会列出共享目录中所有可访问的文件名，因此前面才推荐建立专用目录。

> **特别情况：Ubuntu 是 Windows 电脑上的 WSL**。在支持 localhost 端口转发的 WSL 2 网络配置下，可以先从 Windows 测试 `http://localhost:8000/rosbags.zip`；若失败，再按 WSL 的实际网络模式与端口转发方式排查。不要假定所有 WSL/虚拟机场景都能自动用 localhost 访问。

## 5. 可选：验证服务是否正常

先在 **Ubuntu 的另一个终端**测试：

```bash
curl -I http://127.0.0.1:8000/rosbags.zip
```

一般应返回 `HTTP/1.0 200 OK` 或等价的成功状态。也可以用浏览器访问 Ubuntu 的 `http://127.0.0.1:8000/rosbags.zip`；`127.0.0.1` 只代表**当前运行浏览器的那台电脑自己**，Windows 上的 `127.0.0.1` 通常不是另一台 Ubuntu 机器。

Windows 若下载失败，可在 PowerShell 检查连通性：

```powershell
Test-NetConnection 192.168.1.100 -Port 8000
```

将示例 IP 替换成实际 Ubuntu 地址。若端口测试不通，检查是否在同一可互访网络、Linux 防火墙、Wi-Fi 客户端隔离、虚拟机 NAT/端口转发，以及 Python 服务是否仍在运行。仅当确认局域网访问确有需要时，才考虑添加**限定来源地址**的防火墙规则；不要直接向公网开放端口。

## 6. 下载完成后立即关闭服务器

回到运行 HTTP 服务的 Ubuntu 终端按：

```text
Ctrl+C
```

如果之后需要再次下载，重新执行第 2 节命令即可。需要换新的 ZIP 时先把新文件复制进 `http_download`，检查文件大小与内容再分享。

## 7. 常见错误

| 现象 | 检查方向 |
| --- | --- |
| `Address already in use` | 8000 端口已被占用；结束旧 HTTP 服务，或换成 8001，Windows 链接端口也要同步修改 |
| 浏览器显示 `404 File not found` | 检查 ZIP 实际文件名及是否在 `--directory` 指定的目录内，URL 大小写需一致 |
| Ubuntu 本机可访问、Windows 无法访问 | 检查是否用了 `127.0.0.1` / `0.0.0.0` 当远程访问地址、是否有防火墙/网络隔离/NAT |
| 下载的 ZIP 没有最新 rosbag | 上一次 ZIP/共享目录副本可能是旧文件；在录制完成后重新打包并复制 |
| 下载成功但无法解压 | 等待录制正常停止，重新生成 ZIP；先在 Ubuntu 本机检查 ZIP 是否完整 |
| `Permission denied` | 检查 ZIP 与共享目录的读取权限；如果以前由 root 创建，先确认用户及文件归属，不要把家目录直接设置为全员可写 |

## 8. 一分钟命令备忘

Ubuntu（共享目录里只放需要公开给当前网络的文件）：

```bash
mkdir -p /home/hby/one/http_download
cp /home/hby/one/rosbags.zip /home/hby/one/http_download/rosbags.zip
hostname -I
python3 -m http.server 8000 --bind 0.0.0.0 \
    --directory /home/hby/one/http_download
```

Windows 浏览器：

```text
http://<Ubuntu的实际IPv4地址>:8000/rosbags.zip
```

下载完成回到 Ubuntu 的服务终端按 `Ctrl+C`。

参考：[Python 3 官方文档：`http.server`](https://docs.python.org/3/library/http.server.html)。
