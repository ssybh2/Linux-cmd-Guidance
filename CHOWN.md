# Linux `chown` 工作空间权限修复指南

这篇教程用于解决 Linux / ROS 2 工作空间中常见的权限问题，例如：

```text
Permission denied
cannot open .git/FETCH_HEAD
Operation not permitted
CMake / colcon 无法写入 build、install、log
```

这些问题经常出现在曾经使用过：

```bash
sudo git ...
sudo colcon build
sudo su
```

之后，因为部分文件或目录变成了 `root` 所有，普通用户无法继续修改。

---

## 1. `chown` 是什么

`chown` 的含义是：

```text
change owner
```

也就是修改文件或目录的所有者。

最常用格式：

```bash
sudo chown -R 用户名:用户组 目录路径
```

例如：

```bash
sudo chown -R hby:hby /home/hby/one
```

含义：

```text
sudo                         使用管理员权限执行
chown                        修改所有者
-R                           递归处理整个目录以及里面所有文件
hby:hby                      用户:用户组
/home/hby/one                需要修改权限的目录
```

---

## 2. 给整个 ROS 2 工作空间改回普通用户

例如工作空间：

```text
/home/hby/one
```

用户：

```text
hby
```

执行：

```bash
sudo chown -R hby:hby /home/hby/one
```

然后检查：

```bash
ls -ld /home/hby/one
```

正常应看到类似：

```text
drwxr-xr-x ... hby hby ... /home/hby/one
```

---

## 3. 已经进入工作空间时怎么写

假设当前已经在：

```bash
cd /home/hby/one
```

可以直接执行：

```bash
sudo chown -R hby:hby .
```

其中：

```text
.
```

表示当前目录。

因此：

```bash
sudo chown -R hby:hby .
```

就是把当前目录以及里面所有内容都改成 `hby:hby`。

---

## 4. 如果当前已经是 root

如果终端显示：

```text
root@computer:/home/hby/one#
```

说明当前已经是 root。

这时不需要再写 `sudo`：

```bash
chown -R hby:hby /home/hby/one
```

或者：

```bash
cd /home/hby/one
chown -R hby:hby .
```

完成后建议退出 root：

```bash
exit
```

回到普通用户后再继续：

```text
hby@computer:~$
```

---

## 5. 不要在 root 下使用 `$USER`

普通用户终端里：

```bash
echo $USER
```

可能输出：

```text
hby
```

因此可以写：

```bash
sudo chown -R $USER:$USER /home/hby/one
```

但是如果先执行：

```bash
sudo su
```

此时：

```bash
echo $USER
```

通常会变成：

```text
root
```

这时千万不要执行：

```bash
chown -R $USER:$USER /home/hby/one
```

否则实际上等于：

```bash
chown -R root:root /home/hby/one
```

会把工作空间继续变成 root 所有。

在 root 环境下应该明确写普通用户名：

```bash
chown -R hby:hby /home/hby/one
```

---

## 6. 只修复 ROS 2 的 `build/install/log`

如果源码权限正常，只是之前使用 `sudo colcon build` 导致编译目录属于 root，可以只修复：

```bash
cd /home/hby/one
sudo chown -R hby:hby build install log
```

如果有些目录可能不存在：

```bash
sudo chown -R hby:hby build install log 2>/dev/null || true
```

其中：

```text
2>/dev/null
```

用于隐藏“目录不存在”之类的错误输出。

```text
|| true
```

表示即使前面的命令返回错误，也不要让整条命令链中断。

---

## 7. ROS 2 工作空间彻底清理后重新编译

如果工作空间曾经复制、重命名、移动过，例如：

```text
旧路径：/home/hby/foot_ws
新路径：/home/hby/Foot_ws0
```

可能出现：

```text
CMakeCache.txt directory ... is different than the directory ... where CMakeCache.txt was created
```

这不是单纯的文件权限问题，而是旧的 CMake 缓存仍然记录着之前的绝对路径。

最直接的处理方式：

```bash
cd /home/hby/Foot_ws0
sudo chown -R hby:hby .
rm -rf build install log
colcon build --symlink-install
```

注意：

```text
Linux 区分大小写
```

所以：

```text
foot_ws
Foot_ws
Foot_ws0
```

都是不同路径。

---

## 8. Git 报 `.git/FETCH_HEAD: Permission denied`

例如：

```text
error: cannot open .git/FETCH_HEAD: Permission denied
```

通常说明 `.git` 目录中的文件属于 root。

先查看：

```bash
ls -ld .git
ls -l .git/FETCH_HEAD
```

如果看到：

```text
root root
```

修复整个仓库：

```bash
sudo chown -R hby:hby /home/hby/你的仓库
```

例如：

```bash
sudo chown -R hby:hby /home/hby/ZLT
```

然后再执行：

```bash
git pull
```

---

## 9. 查看当前用户名和用户组

查看当前用户：

```bash
whoami
```

例如：

```text
hby
```

查看 UID、GID 和所属组：

```bash
id
```

可能得到：

```text
uid=1000(hby) gid=1000(hby) groups=1000(hby),27(sudo),...
```

这里说明：

```text
用户名 = hby
主用户组 = hby
```

因此 `chown` 可以写成：

```bash
sudo chown -R hby:hby /path/to/workspace
```

---

## 10. 查看文件到底属于谁

查看目录：

```bash
ls -ld /home/hby/one
```

查看目录里面的文件：

```bash
ls -l /home/hby/one
```

查看 Git 目录：

```bash
ls -ld /home/hby/one/.git
```

查找当前工作空间中所有属于 root 的文件：

```bash
find /home/hby/one -user root -print
```

如果输出很多文件，就说明这个工作空间确实被 root 写过。

---

## 11. 最常用的几条命令

### 整个工作空间改回 `hby`

```bash
sudo chown -R hby:hby /home/hby/one
```

### 当前目录全部改回 `hby`

```bash
sudo chown -R hby:hby .
```

### 只处理 ROS 2 编译目录

```bash
sudo chown -R hby:hby build install log
```

### 当前已经是 root

```bash
chown -R hby:hby /home/hby/one
```

### 查看权限

```bash
ls -ld /home/hby/one
```

### 查找 root 所有的文件

```bash
find /home/hby/one -user root -print
```

---

## 12. ROS 2 / Git 使用建议

正常开发时，以下命令尽量都使用普通用户运行：

```bash
git clone
git pull
git checkout
git switch
colcon build
ros2 run
ros2 launch
```

尽量不要使用：

```bash
sudo git pull
sudo git clone
sudo colcon build
```

也不要为了方便长期在：

```bash
sudo su
```

之后进行 ROS 2 开发。

某些确实需要 root 权限的程序，例如直接访问特定硬件、实时调度或网络接口的程序，可以单独以 root 权限启动，但不要因此把整个源码工作空间和编译目录都放在 root 用户下维护。

---

## 最短备忘录

遇到 ROS 2 工作空间 `Permission denied`：

```bash
cd /home/hby/你的工作空间
sudo chown -R hby:hby .
```

如果工作空间曾经移动、复制或改名，同时清理旧缓存：

```bash
rm -rf build install log
colcon build --symlink-install
```

记住：

```text
chown = 修改文件所有者
-R    = 递归修改整个目录
```

通用格式：

```bash
sudo chown -R <用户名>:<用户组> <目录路径>
```
