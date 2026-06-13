---
aliases: [Linux, Ubuntu]
tags: [AI/Learning, 工具, Linux]
created: 2026-06-13
---

> [!summary] 一句话总结
> Linux = 一切皆文件 + 命令行驱动；掌握文件操作、权限、进程、网络四条线，再会 vim 和 shell 脚本，日常开发运维够用。

---

## 一、Linux 概述

### 1.1 Linux 与 Windows 核心区别

| 维度 | Linux | Windows |
|------|-------|---------|
| 操作方式 | 命令行为主，GUI 为辅 | GUI 为主 |
| 文件系统 | 一切皆文件，单根 `/` | 多盘符 `C:\` `D:\` |
| 权限模型 | 用户/组/其他，rwx 三元组 | ACL，继承式权限 |
| 包管理 | apt / yum / pacman | exe / msi / 应用商店 |
| 服务管理 | systemd / init | 服务管理器 |
| 路径分隔符 | `/` | `\` |
| 大小写敏感 | 敏感 | 不敏感 |

### 1.2 目录结构速查

| 目录 | 用途 | 类比 |
|------|------|------|
| `/bin` | 常用命令（ls, cp...） | 系统工具箱 |
| `/sbin` | 系统管理命令（ifconfig, fdisk...） | 管理工具箱 |
| `/home` | 普通用户主目录 | `C:\Users` |
| `/root` | root 用户主目录 | 管理员专属 |
| `/etc` | 系统配置文件 | 注册表 + 各种 ini |
| `/usr` | 用户程序和库 | `C:\Program Files` |
| `/lib` / `/lib64` | 动态链接库 | `.dll` |
| `/var` | 日志、缓存、可变数据 | 日志目录 |
| `/tmp` | 临时文件（重启可能清除） | `%TEMP%` |
| `/dev` | 设备文件（硬盘、终端...） | 设备管理器 |
| `/proc` | 进程/内核信息的虚拟文件系统 | 任务管理器数据源 |
| `/mnt` / `/media` | 临时挂载点（U盘、光驱） | 挂载盘符 |
| `/opt` | 第三方可选软件安装目录 | 自定义安装 |
| `/boot` | 启动核心文件 | 启动分区 |
| `/run` | 运行时临时文件 | 运行时状态 |

### 1.3 APT 包管理（Debian/Ubuntu）

```bash
sudo apt update              # 更新软件源列表
sudo apt install <包名>      # 安装（加 -y 跳过确认）
sudo apt remove <包名>       # 卸载（保留配置）
sudo apt autoremove          # 清理无用依赖
sudo apt upgrade             # 升级所有已安装的包
sudo apt search <关键词>     # 搜索软件包
apt show <包名>              # 查看包详情
```

---

## 二、文件操作

### 2.1 导航与查看

```bash
pwd                          # 显示当前目录绝对路径
cd /path                     # 切换目录
cd                           # 回到 ~（家目录）
cd -                         # 回到上一次的目录
cd ..                        # 回到上级目录
ls                           # 列出目录内容
ls -l                        # 详细信息（= ll）
ls -la                       # 包含隐藏文件
ls -lh                       # 文件大小人类可读（K/M/G）
ll                           # ls -al 的别名（Ubuntu 默认）
```

### 2.2 创建与删除

```bash
mkdir dir1                   # 创建目录
mkdir -p a/b/c               # 递归创建多级目录
rmdir dir1                   # 删除空目录
touch file.txt               # 创建空文件 / 更新时间戳
rm file.txt                  # 删除文件
rm -r dir/                   # 递归删除目录及内容
rm -rf dir/                  # 强制递归删除（危险！）
```

### 2.3 复制与移动

```bash
cp src dest                  # 复制文件
cp -r src_dir/ dest/         # 递归复制目录
cp -p file dest              # 保留权限和时间戳
mv old new                   # 重命名
mv file /target/             # 移动文件
```

### 2.4 查找

```bash
find /path -name "*.py"      # 按名称查找
find . -type f -size +100M   # 按类型和大小查找
find . -user "root"          # 按所有者查找
find . -mtime -7             # 最近 7 天修改过的文件
```

### 2.5 软链接（符号链接 = Windows 快捷方式）

```bash
ln -s /real/path /link       # 创建软链接
rm link                      # 删除软链接（不要加 /）
cd -P link                   # 进入链接指向的实际目录
```

### 2.6 输出重定向与管道

```bash
echo "hello" > file          # 覆盖写入
echo "world" >> file         # 追加写入
command1 | command2          # 管道：前者的输出作为后者的输入
command > file 2>&1          # 合并 stdout 和 stderr 到文件
command &> file              # 同上（bash 简写）
```

### 2.7 其他

```bash
history                      # 查看历史命令
basename /path/to/file.txt   # 输出 file.txt
dirname /path/to/file.txt    # 输出 /path/to
```

---

## 三、文本处理

### 3.1 查看文件

```bash
cat file                     # 输出全部内容（适合小文件）
cat -n file                  # 带行号
head -n 20 file              # 前 20 行
tail -n 20 file              # 后 20 行
tail -f file                 # 实时追踪文件末尾（看日志常用）
tail -F file                 # 实时追踪（文件轮转时自动重开）
less file                    # 分页浏览（支持搜索，按 q 退出）
wc -l file                   # 统计行数
wc -w file                   # 统计单词数
```

### 3.2 搜索过滤

```bash
grep "pattern" file          # 搜索匹配的行
grep -r "pattern" /path      # 递归搜索目录
grep -n "pattern" file       # 显示行号
grep -i "pattern" file       # 忽略大小写
grep -c "pattern" file       # 只输出匹配行数
grep -v "pattern" file       # 反选（排除匹配的行）
grep -E "regex" file         # 扩展正则（= egrep）
```

### 3.3 字段截取与排序

```bash
cut -d: -f1,3 /etc/passwd    # 按 : 分隔，取第 1、3 列
sort file                    # 排序（按行首）
sort -n file                 # 按数值排序
sort -r file                 # 逆序
sort -t: -k3 -n file         # 按 : 分隔，第 3 列数值排序
uniq                         # 去重（相邻重复行，通常配合 sort 使用）
sort file | uniq -c          # 去重并计数
```

### 3.4 sed 流编辑器

```bash
sed 's/old/new/g' file       # 全局替换（不修改原文件，输出到终端）
sed -i 's/old/new/g' file    # 就地修改文件
sed -n '5,10p' file          # 打印第 5-10 行
sed '/^#/d' file             # 删除注释行
```

### 3.5 awk 文本分析

```bash
awk '{print $1, $3}' file    # 打印第 1、3 列
awk -F: '{print $1}' /etc/passwd  # 指定分隔符为 :
awk '$3 > 100' file          # 条件过滤（第 3 列 > 100）
awk 'BEGIN{FS=":"} NR==1{print $1}' /etc/passwd  # BEGIN 指定分隔符
```

---

## 四、权限管理

### 4.1 文件属性解读

```
- rwxr-xr-- 1 user group 1234 Jan 1 00:00 file.txt
│ │││ │││ │││
│ U  G   O
│
类型: - 文件 | d 目录 | l 链接
```

| 位置 | 含义 | 数字 |
|------|------|------|
| r (read) | 可读 | 4 |
| w (write) | 可写 | 2 |
| x (execute) | 可执行 / 可进入目录 | 1 |

**rwx 在目录上的特殊含义**：r = 能 ls，w = 能在里面增删文件，x = 能 cd 进入。

### 4.2 chmod 修改权限

```bash
chmod u+x file               # 给所有者加执行权限
chmod g-w file               # 去掉组的写权限
chmod o=r file               # 其他人设为只读
chmod a+r file               # 所有人加读权限
chmod 755 file               # rwxr-xr-x（数字法）
chmod -R 777 dir/            # 递归修改整个目录（慎用）
```

数字法速记：`7`=rwx, `6`=rw-, `5`=r-x, `4`=r--, `0`=---

### 4.3 chown / chgrp

```bash
chown user file              # 改所有者
chown user:group file        # 同时改所有者和组
chown -R user:group dir/     # 递归修改
chgrp group file             # 改所属组
```

---

## 五、用户与组管理

### 5.1 用户操作

```bash
sudo passwd root             # 为 root 设置密码（Ubuntu 默认锁定 root）
useradd -m username          # 创建用户（-m 同时创建家目录）
useradd -g groupname user    # 创建用户并指定初始组
passwd username              # 设置/修改密码
userdel username             # 删除用户（保留家目录）
userdel -r username          # 删除用户及家目录
usermod -l newname oldname   # 重命名用户
usermod -g group user        # 更改用户主组
id username                  # 查看 uid、gid、所属组
whoami                       # 当前用户
cat /etc/passwd              # 查看所有用户
```

### 5.2 组操作

```bash
groupadd groupname           # 新增组
groupdel groupname           # 删除组
groupmod -n newname oldname  # 重命名组
cat /etc/group               # 查看所有组
```

### 5.3 sudo 与 su

```bash
sudo command                 # 以 root 权限执行命令（需输入当前用户密码）
su                           # 切换到 root（保持环境）
su -                         # 切换到 root（加载 root 环境变量）
su username                  # 切换到指定用户
exit                         # 退出当前用户 / 关闭终端
```

> [!tip] 推荐做法
> 日常用 `sudo` 执行单条命令，不要长期以 root 登录。需要完整 root 环境用 `su -`。

### 5.4 配置 sudo 免密码

编辑 `/etc/sudoers`（用 `visudo` 更安全），添加：
```
username ALL=(ALL) NOPASSWD:ALL
```

---

## 六、系统管理

### 6.1 进程管理

```bash
ps -aux                      # 查看所有进程（CPU、内存、STAT 等）
ps -ef                       # 查看所有进程（含 PPID 父进程关系）
top                          # 实时进程监控（按 q 退出）
top -d 1                     # 每 1 秒刷新
top -p PID                   # 只监控指定 PID
kill PID                     # 终止进程（默认 SIGTERM=15）
kill -9 PID                  # 强制终止（SIGKILL）
killall processname          # 按名称终止所有匹配进程
```

**top 常用交互**：`P` 按 CPU 排序，`M` 按内存排序，`k` 输入 PID 杀进程。

**ps 输出中 STAT 关键状态**：`R`=运行, `S`=睡眠, `T`=停止, `Z`=僵尸, `D`=不可中断睡眠。

### 6.2 系统资源

```bash
df -h                        # 磁盘使用情况（人类可读）
du -sh dir/                  # 目录总大小
du -h --max-depth=1 dir/     # 目录下各子项大小
free -m                      # 内存使用（MB 单位）
hostname                     # 查看主机名
```

### 6.3 服务管理（systemd）

```bash
systemctl start <服务>       # 启动
systemctl stop <服务>        # 停止
systemctl restart <服务>     # 重启
systemctl status <服务>      # 查看状态
systemctl enable <服务>      # 设置开机自启
systemctl disable <服务>     # 取消开机自启
```

### 6.4 定时任务（crontab）

```bash
crontab -e                   # 编辑当前用户的定时任务
crontab -l                   # 查看当前用户的定时任务
```

cron 表达式格式：

```
分 时 日 月 周 命令
* * * * * /path/to/script.sh

# 示例
*/5 * * * *  /bin/echo "hello" >> /tmp/log.txt   # 每 5 分钟
0 2 * * *    /path/to/backup.sh                    # 每天凌晨 2 点
```

---

## 七、网络

```bash
ifconfig                     # 查看网络接口 IP（需 net-tools）
ip addr                      # 同上（iproute2 推荐方式）
ping -c 4 www.baidu.com      # 测试连通性（-c 限制次数）
curl -I https://example.com  # 查看 HTTP 响应头
curl -o file URL             # 下载文件
wget URL                     # 下载文件
wget -c URL                  # 断点续传
netstat -anp | grep :80      # 查看端口 80 占用情况
netstat -tlnp                # 列出所有监听的 TCP 端口
ss -tlnp                     # netstat 的现代替代，更快
ssh user@host                # 远程登录
ssh -p 2222 user@host        # 指定端口
scp file user@host:/path     # 上传文件到远程
scp user@host:/path file     # 从远程下载文件
scp -r dir/ user@host:/path  # 递归复制目录
```

---

## 八、Vim 编辑器

### 8.1 三种模式

```
一般模式（默认）
  │  i/I/a/A/o/O → 进入
  ↓
编辑模式（INSERT）
  │  Esc → 返回
  ↓
一般模式
  │  : / ? → 进入
  ↓
命令模式
  │  Esc → 返回
  ↓
一般模式
```

### 8.2 一般模式常用操作

| 按键 | 功能 |
|------|------|
| `i` / `a` | 光标前 / 后插入 |
| `I` / `A` | 行首 / 行尾插入 |
| `o` / `O` | 下方 / 上方新建行 |
| `x` / `dd` | 删除字符 / 删除整行 |
| `u` | 撤销 |
| `Ctrl+r` | 重做 |
| `yy` | 复制当前行 |
| `p` | 粘贴 |
| `/keyword` | 向下搜索 |

### 8.3 命令模式常用命令

```vim
:wq                          # 保存退出
:wq!                         # 强制保存退出
:q                           # 不保存退出
:q!                          # 强制退出
:w                           # 仅保存
:set nu                      # 显示行号
:noh                         # 取消搜索高亮
:n1,n2d                      # 删除第 n1 到 n2 行
:%s/old/new/g                # 全文替换
:n                           # 跳转到第 n 行
```

---

## 九、Shell 脚本基础

### 9.1 基本结构

```bash
#!/bin/bash                   # Shebang，指定解释器
# 这是注释
echo "Hello World"           # 输出
```

### 9.2 变量

```bash
name="Linux"                 # 赋值（等号两边不能有空格）
echo $name                   # 使用变量
echo ${name}                 # 同上（推荐加花括号）
readonly name="const"        # 只读变量
unset name                   # 删除变量
`command` 或 $(command)      # 命令替换，将命令输出赋给变量
```

### 9.3 条件判断

```bash
# if 语法
if [ "$a" -eq "$b" ]; then
    echo "相等"
elif [ "$a" -gt "$b" ]; then
    echo "a 更大"
else
    echo "b 更大"
fi
```

常用比较运算符：
- 数值比较：`-eq` 等于, `-ne` 不等于, `-gt` 大于, `-lt` 小于, `-ge` 大于等于, `-le` 小于等于
- 字符串比较：`=`, `!=`, `-z`（空串）, `-n`（非空）
- 文件测试：`-f` 是否文件, `-d` 是否目录, `-e` 是否存在, `-r` 是否可读, `-x` 是否可执行

### 9.4 循环

```bash
# for 循环
for i in 1 2 3 4 5; do
    echo "第 $i 次"
done

# C 风格 for
for ((i=1; i<=5; i++)); do
    echo "$i"
done

# while 循环
count=1
while [ $count -le 5 ]; do
    echo "$count"
    ((count++))
done

# until 循环（条件为假时执行）
until [ $count -gt 5 ]; do
    echo "$count"
    ((count++))
done
```

### 9.5 函数

```bash
greet() {
    local name=$1             # local 定义局部变量
    echo "Hello, $name"
}

greet "Alice"                # 调用函数
```

### 9.6 常用快捷键

| 快捷键 | 功能 |
|--------|------|
| `Tab` | 命令/路径补全 |
| `Ctrl+C` | 终止当前命令 |
| `Ctrl+Z` | 挂起当前命令（`fg` 恢复） |
| `Ctrl+R` | 反向搜索历史命令 |
| `Ctrl+A` / `Ctrl+E` | 光标移到行首 / 行尾 |
| `Ctrl+L` | 清屏（= `clear`） |
| `!!` | 重复上一条命令 |
| `!$` | 上一条命令的最后一个参数 |
