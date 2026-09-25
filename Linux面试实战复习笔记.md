# Linux 面试实战复习笔记

> 本笔记基于 WSL2 + Ubuntu 环境下的实战练习整理，每个知识点都配有真实跑过的命令和输出验证。

---

## 一、awk + sort + uniq 组合统计（高频送分题）

### 场景：统计访问日志中出现次数最多的IP

```bash
awk '{print $1}' logs/access.log | sort | uniq -c | sort -rn
```

**分步理解：**

| 步骤 | 命令 | 作用 |
|---|---|---|
| 1 | `awk '{print $1}' file` | 提取每行第1列（`$1`=第1字段，`$0`=整行） |
| 2 | `sort` | 排序，让相同内容的行彼此相邻 |
| 3 | `uniq -c` | 去除**相邻**重复行，`-c` 加上出现次数 |
| 4 | `sort -rn` | `-r`=降序，`-n`=按数字而非字符串比较 |

**核心考点：为什么 uniq 前必须 sort？**

`uniq` 只去重**相邻**的行。实测对比：

```bash
# 不排序，直接uniq（错误示范）
awk '{print $1}' logs/access.log | uniq -c
# 结果：同一个IP被拆成好几段分别计数，而不是合并成一次
```

**进阶：当计数相同时按第二列再排序**

```bash
awk '{print $1}' logs/access.log | sort | uniq -c | sort -rn -k2
```
`-k2` 表示以第2列（这里是IP地址）作为排序的第二关键字。

---

## 二、grep 日志排查

```bash
grep "500" logs/access.log        # 基础搜索
grep -n "500" logs/access.log     # -n 显示行号
grep -v "200" logs/access.log     # -v 反选，找出不匹配的行
grep -c "500" logs/access.log     # -c 只统计匹配行数
```

| 参数 | 全称/含义 |
|---|---|
| `-n` | number，显示行号 |
| `-v` | invert，反选 |
| `-c` | count，只统计数量 |
| `-r` | 递归搜索目录 |
| `-i` | 忽略大小写 |

---

## 三、经典送分题：文件删了但磁盘空间没释放

**背景**：`df` 显示磁盘满了，但 `du` 加起来对不上——原因是文件被删除后，仍有进程持有它的文件描述符（fd），数据未真正释放。

### 完整复现流程

```bash
# 1. 造一个大文件
dd if=/dev/zero of=bigfile.log bs=1M count=50

# 2. 启动一个进程占用它（模拟正在读日志的服务）
tail -f bigfile.log > /dev/null 2>&1 &
echo $!    # 记录PID

# 3. 用lsof确认占用
lsof -p <PID> | grep bigfile

# 4. 删除文件
rm bigfile.log
ls bigfile.log    # 报错：文件不存在

# 5. 再次lsof查看——见证奇迹
lsof -p <PID> | grep bigfile
# 输出会带 (deleted) 标记：
# tail  <PID>  caikai  3r  REG  8,48  52428800  231  /path/bigfile.log (deleted)

# 6. 杀掉进程，空间才真正释放
kill <PID>
```

### lsof 输出字段速查

| 字段 | 含义 |
|---|---|
| 进程名/PID/用户 | 占用文件的进程信息 |
| `3r` | 文件描述符编号 + 权限（r=只读） |
| `REG` | 文件类型（普通文件） |
| 文件大小（字节） | 占用的实际空间 |
| inode号 | 文件在文件系统里的"身份证号"，只要inode被引用，数据就还在 |
| `(deleted)` | 标记：文件名已从目录删除，但inode数据未释放 |

**`dd` 命令参数解释：**

```bash
dd if=/dev/zero of=bigfile.log bs=1M count=50
```

| 参数 | 含义 |
|---|---|
| `if=` | input file，输入源（`/dev/zero` = 无限吐出0字节的特殊设备） |
| `of=` | output file，输出目标 |
| `bs=1M` | 每个块大小1MB |
| `count=50` | 共50个块 = 50MB |

---

## 四、后台任务与重定向

```bash
tail -f bigfile.log > /dev/null 2>&1 &
```

| 部分 | 含义 |
|---|---|
| `tail -f` | 持续监听文件新增内容，永不主动退出 |
| `> /dev/null` | 把标准输出（stdout, 编号1）丢进"黑洞"设备，不显示 |
| `2>&1` | 把标准错误（stderr, 编号2）也重定向到**当前**标准输出指向的地方 |
| 结尾 `&` | 放入后台运行，不阻塞终端 |

**顺序很重要（面试常考陷阱）：**
- `> /dev/null 2>&1` —— 正确，stdout和stderr都会被丢弃
- `2>&1 > /dev/null` —— 错误，stderr会先被重定向到"当时的屏幕"，之后stdout才被丢到黑洞，结果stderr依然显示在屏幕上

**特殊变量：**

| 变量 | 含义 |
|---|---|
| `$!` | 最近一个后台任务的PID |
| `$$` | 当前脚本/进程自己的PID |
| `$0` | 当前脚本自己的名字/路径 |
| `$1`, `$2`... | 脚本的第1、2...个命令行参数 |

---

## 五、重定向符号

```bash
echo "内容" > file.txt    # 覆盖写入
echo "内容" >> file.txt   # 追加写入
```

---

## 六、Here Document（`cat << EOF`）

```bash
cat > file.txt << 'EOF'
多行内容...
EOF
```

| 部分 | 含义 |
|---|---|
| `<<` | Here Document开始符号，后面的多行文本作为标准输入喂给前面的命令 |
| `EOF` | 自定义的结束标记词（可以换成任意词），必须顶格单独一行才能识别为"结束" |
| 加单引号 `'EOF'` | **不做**变量替换和命令替换，内容原样保存 |
| 不加引号 `EOF` | **会做**变量替换（`$VAR`）和命令替换（`$(cmd)`） |

### 实测对比

```bash
export MY_SECRET="真实密码123"

# 不加引号 —— $MY_SECRET 和 $(date) 都被替换成真实内容
cat > test1.txt << EOF
密码: $MY_SECRET
日期: $(date)
EOF

# 加引号 —— 原样保留，不替换
cat > test2.txt << 'EOF'
密码: $MY_SECRET
日期: $(date)
EOF
```

**结论**：写脚本示例、正则表达式等包含 `$` 符号的内容时，一定要用带引号的 `'EOF'`，否则可能被意外替换甚至泄露环境变量。

---

## 七、find 批量清理文件

```bash
# 先查看（不删），确认无误
find . -name "*.log" -mtime +7

# 确认没问题后再真正删除
find . -name "*.log" -mtime +7 -delete
```

| 部分 | 含义 |
|---|---|
| `-name "*.log"` | 按文件名匹配（`*` 通配符） |
| `-mtime +7` | 修改时间超过7天前（`+`=大于，`-`=小于，不加符号=精确等于） |
| `-delete` | 删除找到的文件，**必须放在所有过滤条件最后**，避免误删 |

**安全习惯**：永远先跑不带 `-delete` 的版本预览结果，确认无误后再加上 `-delete`。

**find 与 ls/grep 的区别**：`find` 默认**递归**查找子目录，`ls`（不加参数）只看当前层，`grep`不加`-r`也不递归。

---

## 八、chmod 权限体系

### 三组权限的读法

```
-rwxr-xr-x
```

| 位置 | 分组 | 含义 |
|---|---|---|
| 第1位 | 文件类型 | `-`=普通文件，`d`=目录，`l`=软链接 |
| 2-4位 | owner（属主） | 文件所有者的权限 |
| 5-7位 | group（同组用户） | 同组其他用户的权限 |
| 8-10位 | other（其他人） | 所有其他用户的权限 |

`r`=读，`w`=写，`x`=执行

### 符号模式

```bash
chmod u+x file.sh    # u=user(属主) 加执行权限
chmod g+w file.sh     # g=group(同组) 加写权限
chmod o+x file.sh     # o=other(其他人) 加执行权限
chmod a+x file.sh     # a=all(所有人)
```

### 数字模式（面试必考手算）

| 权限 | 数值 |
|---|---|
| r | 4 |
| w | 2 |
| x | 1 |

```
chmod 755 file.sh
```
= owner(7=4+2+1=rwx) + group(5=4+0+1=r-x) + other(5=4+0+1=r-x)
= `-rwxr-xr-x`

### 实测验证：不同身份的权限差异

用 `sudo -u nobody 命令` 可以模拟"以其他用户身份"执行命令，验证 other 权限是否生效：

```bash
sudo -u nobody ./deploy.sh
# 如果other没有x权限 -> Permission denied
# 加上 chmod o+x 之后 -> 执行成功，whoami显示nobody
```

---

## 九、ps / top 进程排查

### top 批处理模式（方便截图/记录）

```bash
top -bn1 | head -15
```
`-b`=batch mode（只打印一次不刷新），`-n1`=只刷新1次

### top 输出字段

| 列名 | 含义 |
|---|---|
| PID | 进程ID |
| USER | 运行用户 |
| PR / NI | 优先级 / nice值（越小越优先） |
| VIRT / RES / SHR | 虚拟内存 / 实际物理内存 / 共享内存 |
| S | 状态：R运行 S睡眠 Z僵尸 D不可中断睡眠 |
| %CPU / %MEM | 占用百分比 |
| COMMAND | 进程名 |

### ps 查看指定进程

```bash
ps -ef | grep <PID> | grep -v grep
```

---

## 十、kill 信号：SIGTERM vs SIGKILL（对照实验验证）

### 普通 kill（SIGTERM，信号15）—— 可以被进程"无视"

```bash
cat > stubborn.sh << 'EOF'
#!/bin/bash
trap 'echo "收到SIGTERM，但我选择无视你！"' SIGTERM
echo "顽固进程启动，PID: $$"
while true; do sleep 1; done
EOF
chmod +x stubborn.sh
./stubborn.sh &
kill $!
# 结果：脚本打印无视消息，进程依然存活（ps -p 还能查到）
```

`trap '命令' 信号名` —— 捕获指定信号，执行自定义命令而非默认的"终止进程"行为。

### kill -9（SIGKILL，信号9）—— 内核强制执行，无法拦截

```bash
kill -9 $STUBBORN_PID
# 结果：[1]+ Killed，立即终止，进程没有任何挣扎机会
```

### 对比总结

| 信号 | 编号 | 能否被拦截 | 特点 |
|---|---|---|---|
| SIGTERM（`kill`） | 15 | 能（可用trap捕获/忽略） | "请求"式，给进程清理资源的机会 |
| SIGKILL（`kill -9`） | 9 | 不能，内核强制执行 | "命令"式，可能导致数据未保存、资源未释放 |

**原则**：优先用普通 `kill`，只有确认进程无视SIGTERM或卡在D状态时才上 `kill -9`。

---

## 十一、综合实战：完整的日志清理脚本

融合了参数处理、容错校验、find批量操作、安全删除流程、审计日志记录：

```bash
#!/bin/bash
# ===== 日志清理脚本 =====
# 用法: ./cleanup_logs.sh <目标目录> <保留天数>

TARGET_DIR=$1
DAYS=$2

# 参数校验：如果两个参数没给全，打印用法并退出
if [ -z "$TARGET_DIR" ] || [ -z "$DAYS" ]; then
  echo "用法: $0 <目标目录> <保留天数>"
  exit 1
fi

# 校验目录是否存在
if [ ! -d "$TARGET_DIR" ]; then
  echo "错误: 目录 $TARGET_DIR 不存在"
  exit 1
fi

# 统计符合条件（超过N天）的文件数量
COUNT=$(find "$TARGET_DIR" -name "*.log" -mtime +"$DAYS" | wc -l)

if [ "$COUNT" -eq 0 ]; then
  echo "没有超过 $DAYS 天的日志文件，无需清理"
  exit 0
fi

echo "发现 $COUNT 个超过 $DAYS 天的日志文件，清单如下："
find "$TARGET_DIR" -name "*.log" -mtime +"$DAYS" -exec ls -lh {} \;

# 统计将要释放的总空间
TOTAL_SIZE=$(find "$TARGET_DIR" -name "*.log" -mtime +"$DAYS" -exec du -ch {} + | tail -1 | awk '{print $1}')
echo "预计释放空间: $TOTAL_SIZE"

# 执行删除
find "$TARGET_DIR" -name "*.log" -mtime +"$DAYS" -delete

# 记录操作日志
echo "$(date '+%Y-%m-%d %H:%M:%S') - 用户 $(whoami) 清理了 $COUNT 个文件，释放约 $TOTAL_SIZE" >> ~/practice/cleanup_audit.log

echo "清理完成！已记录到 cleanup_audit.log"
```

### 脚本里的关键语法点

| 语法 | 含义 |
|---|---|
| `#!/bin/bash` | Shebang，指定解释器 |
| `$1` `$2` | 命令行参数 |
| `[ -z "$VAR" ]` | 判断字符串是否为空 |
| `[ ! -d "$DIR" ]` | 判断路径是否**不是**目录 |
| `\|\|` | 逻辑或 |
| `exit 1` / `exit 0` | 退出并返回状态码（0=成功，非0=失败，供外部脚本判断） |
| `$(命令)` | 命令替换，把命令输出存入变量 |
| `-exec ... {} \;` | find对每个匹配文件执行命令，`{}`是占位符 |
| `du -ch {} +` | `+`一次性传入所有文件（比`\;`逐个调用更高效），`-c`打印合计行 |
| `>>` | 追加写入（用于审计日志） |

### 语法检查小技巧

```bash
bash -n script.sh && echo "语法没问题"
```
`-n` = 只检查语法，不实际执行。

---

## 十二、其他速记知识点

### bash 是什么

`bash` 是一个 **Shell**（命令行解释器）——一个可执行程序，位于 `/usr/bin/bash`，负责接收命令、解析、调用内核执行、返回结果。

```bash
which bash      # 查看路径
echo $0         # 查看当前shell自身
```

### export 环境变量

```bash
export VAR="value"
```
把变量提升为**环境变量**，不仅当前shell能用，其**子进程**也能继承读取到。不加`export`的普通变量只在当前shell内部可见，子进程/子脚本读不到。

### 换行输入的几种方式

| 场景 | 方法 |
|---|---|
| 命令太长想手动换行 | 行尾加反斜杠 `\` 再回车 |
| Here Document 多行输入模式 | 直接回车即可 |
| 命令行编辑中插入换行符（不提交） | `Ctrl+V` 然后 `Enter` |
| vim/nano 编辑器里 | 直接回车 |

### Here Document 卡住了怎么办

如果不小心进入了 `>` 续行提示符出不来：
- 正常结束：单独一行顶格输入结束标记词（如 `EOF`）后回车
- 放弃本次输入：按 `Ctrl + C`

---

## 面试高频追问速记

- **`kill` 和 `kill -9` 区别？** SIGTERM可被trap捕获优雅退出，SIGKILL内核强制无法拦截
- **软链接 vs 硬链接？** `ln -s`创建软链接（路径引用）vs `ln`创建硬链接（同一inode）
- **`>` 和 `>>` 区别？** 覆盖 vs 追加
- **为什么uniq前要sort？** uniq只去重相邻行
- **df和du结果对不上？** 文件被删但进程fd仍占用，看lsof的`(deleted)`标记
- **僵尸进程怎么产生的？** 子进程已终止但父进程没有wait()回收，需要处理父进程
- **`.bashrc` 和 `.bash_profile` 区别？** 前者每次开新shell都加载，后者只在登录时加载一次
