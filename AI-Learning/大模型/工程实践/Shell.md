---
aliases: [Shell, Bash, 脚本]
tags: [AI/Learning, 工具, Shell]
created: 2026-06-13
---

> [!tip] 背诵版（100字）
> Shell 是 Linux 命令行解释器，Bash 最常用。变量用 `=` 赋值、`$` 取值；条件用 `[ ]` 或 `test`；循环有 `for`/`while`/`until`；函数用 `function` 或直接定义名；数组用 `()` 声明；`echo`/`read` 处理 IO；`grep`/`sed`/`awk` 是文本处理三剑客。脚本以 `#!/bin/bash` 开头，`chmod +x` 后执行。

## 目录
- [[#一、Shell 概述]]
- [[#二、变量]]
- [[#三、条件判断]]
- [[#四、循环]]
- [[#五、函数]]
- [[#六、数组]]
- [[#七、输入输出]]
- [[#八、文本处理三剑客]]
- [[#九、脚本调试与最佳实践]]

## 一、Shell 概述
Shell 是 Linux 的**命令行解释器**，负责将用户命令翻译为内核系统调用。常见 Shell：`bash`（最常用）、`sh`、`zsh`、`dash`。
```bash
cat /etc/shells       # 查看系统支持的 Shell
echo $SHELL            # 查看当前 Shell
```
脚本以 `#!/bin/bash` 开头，两种执行方式：
```bash
bash ./hello.sh        # 方式一：不需要 +x 权限
chmod +x hello.sh
./hello.sh             # 方式二：需要 +x 权限
```

## 二、变量
### 系统预定义变量
| 变量 | 含义 |
|------|------|
| `$PATH` | 命令搜索路径 |
| `$HOME` | 当前用户主目录 |
| `$PWD` | 当前工作目录 |
| `$SHELL` | 当前 Shell 解释器路径 |
| `$USER` | 当前用户名 |
### 自定义变量
```bash
A=5                     # 定义（= 两侧不能有空格）
echo $A                 # 使用：5
unset A                 # 删除
readonly B=2            # 只读变量（不能修改和 unset）
D="I love shell"        # 值有空格必须加引号
```
**命名规则**：字母/数字/下划线，不能以数字开头，环境变量建议全大写。
### 特殊变量
| 变量 | 含义 |
|------|------|
| `$0` | 脚本名称 |
| `$1`~`$9` / `${10}` | 第 1~9 / 10+ 个参数 |
| `$#` | 参数个数 |
| `$*` | 所有参数（整体） |
| `$@` | 所有参数（逐个区分） |
| `$?` | 上条命令返回状态（0=成功） |
> **`$*` vs `$@`**：加引号后，`"$*"` 是一个整体字符串，`"$@"` 保持各参数独立，循环遍历时用 `"$@"`。
### 环境变量（全局）
```bash
export MY_VAR=100       # 提升为环境变量，子进程可继承
```
### 算术运算
```bash
S=$[(2+3)*4]            # 20，$[] 和 $(()) 均可
echo $((10 / 3))        # 3（整数除法）
```

## 三、条件判断
### 基本语法
```bash
test condition           # 写法1
[ condition ]            # 写法2（推荐），[ ] 与条件之间必须有空格
```
### 整数比较
| 运算符 | 含义 |
|--------|------|
| `-eq` | 等于 | `-ne` | 不等于 |
| `-lt` | 小于 | `-le` | 小于等于 |
| `-gt` | 大于 | `-ge` | 大于等于 |
```bash
[ 23 -lt 22 ] && echo "yes" || echo "no"   # no
```
### 文件测试
| 运算符 | 含义 |
|--------|------|
| `-e` | 文件存在 | `-f` | 是常规文件 | `-d` | 是目录 |
| `-r` | 有读权限 | `-w` | 有写权限 | `-x` | 有执行权限 |
```bash
[ -e /etc/passwd ] && echo "存在"
[ -d /tmp ] && echo "是目录"
```
### 字符串比较
```bash
[ -z "$str" ]     # 字符串为空    [ -n "$str" ]     # 字符串非空
[ "$a" = "$b" ]   # 相等          [ "$a" != "$b" ]  # 不等
```
### if / case
```bash
# if 多分支
if [ $1 -lt 18 ]; then
    echo "未成年"
elif [ $1 -lt 60 ]; then
    echo "成年人"
else
    echo "老年人"
fi

# case 语句（;; 相当于 break，* 相当于 default）
case $1 in
    "1") echo "one" ;;
    2)   echo "two" ;;
    *)   echo "other" ;;
esac
```

## 四、循环
### for 循环
```bash
# C 风格
for ((i=1; i<=100; i++)); do sum=$[sum+$i]; done
# in 列表
for i in cls mly wls; do echo "hello $i"; done
# 遍历命令结果
for file in $(seq 1 10); do echo $file; done
```
### while / until
```bash
# while（条件为真时执行）
i=1; while [ $i -le 5 ]; do echo $i; i=$[i+1]; done
# until（条件为假时执行，与 while 相反）
i=1; until [ $i -gt 5 ]; do echo $i; i=$[i+1]; done
```
### break / continue
```bash
for i in {1..10}; do
    [ $i -eq 3 ] && continue   # 跳过 3
    [ $i -eq 8 ] && break      # 到 8 停止
    echo $i
done   # 1 2 4 5 6 7
```

## 五、函数
```bash
# 定义（function 和 () 只能省略一个）
sum() {
    SUM=$[$1 + $2]
    echo $SUM          # 用 echo 返回结果
}
sum 3 5                # 8
result=$(sum 10 20)    # 通过 $() 捕获返回值
```
- `return` 只能返回 0-255 的整数，实战建议用 `echo` 输出 + `$(func)` 捕获
- Shell 逐行执行，函数**必须在调用之前定义**

## 六、数组
```bash
arr=(1 2 3 4 5)               # 定义
echo ${arr[0]}                 # 访问第 1 个元素：1
echo ${arr[@]}                 # 所有元素
echo ${#arr[@]}                # 数组长度：5
for item in ${arr[@]}; do echo $item; done   # 遍历
arr[0]=100                     # 修改
arr+=(6 7 8)                   # 追加
unset arr[2]                   # 删除第 3 个元素
```

## 七、输入输出
### echo / read
```bash
echo -e "a\tb"                 # 解析转义字符
echo -n "no newline"           # 不换行
read -t 7 -p "请输入:" NAME    # 带提示和超时（-p 提示，-t 秒数）
```
### 重定向与管道
| 符号 | 含义 |
|------|------|
| `>` / `>>` | 覆盖 / 追加写入 |
| `2>` | 重定向错误输出 |
| `&>` | 同时重定向标准和错误输出 |
| `<` | 重定向输入 |
| `\|` | 管道（前命令输出作为后命令输入） |
```bash
ls > list.txt; ls >> list.txt   # 覆盖/追加
ps aux | grep nginx             # 管道
ls -la | wc -l                  # 统计文件数
```

## 八、文本处理三剑客
### grep -- 文本搜索
```bash
grep -i "hello" file   # 忽略大小写    grep -n "hello" file   # 显示行号
grep -v "hello" file   # 反选          grep -r "hello" ./dir  # 递归搜索
```
**正则常用**：`^` 行首 | `$` 行尾 | `.` 任意字符 | `*` 前字符 0+ 次 | `[abc]` 字符集 | `[0-9]` 数字范围
### sed -- 流编辑器
```bash
sed 's/old/new/g' file       # 全局替换（不改原文件）
sed -i 's/old/new/g' file    # -i 直接修改文件
sed '/pattern/d' file        # 删除匹配行
sed '3d' file                # 删除第 3 行
sed '2a\new line' file       # 第 2 行后追加
```
### awk -- 文本分析
```bash
awk -F: '{print $1, $7}' /etc/passwd                    # 按 : 分隔，取列
awk -F: '/^root/{print $1, $7}' /etc/passwd             # 匹配 pattern 才执行
awk -F: 'BEGIN{print "user"} {print $1} END{print "done"}' file  # BEGIN/END
awk -v offset=1 -F: '{print $3+offset}' /etc/passwd     # -v 传变量
```
**内置变量**：`NR` 行号 | `NF` 列数 | `FILENAME` 文件名

## 九、脚本调试与最佳实践
```bash
bash -x script.sh    # 打印每条命令及结果（最常用调试）
bash -n script.sh    # 只检查语法，不执行
set -x / set +x      # 在脚本中开关调试
```
**最佳实践**：
| 实践 | 说明 |
|------|------|
| `#!/bin/bash` | 首行加 shebang |
| `"$VAR"` | 变量加引号防空值展开 |
| `$?` | 关键命令后检查返回值 |
| `$#` | 用参数个数做校验 |
| `set -euo pipefail` | 推荐脚本开头三件套：遇错退出 + 未定义变量报错 + 管道失败传播 |
| `command && ok \|\| fail` | 条件执行简写 |
| `(cd /tmp && ls)` | 子 shell 不影响当前环境 |
| `command &` | 后台运行 |

---
> **一句话总结**：Shell 脚本 = 变量 + 条件 + 循环 + 函数 + 文本处理，掌握 `if`/`for`/`while`/`echo`/`read`/`grep`/`sed`/`awk` 即可覆盖 90% 的日常运维和数据处理场景。
