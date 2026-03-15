# 08、Shell 脚本

## Q1. Shell 脚本的基础语法是什么？

```bash
#!/bin/bash
# 第一行是 Shebang，指定解释器

# ==================== 变量 ====================
name="world"
echo "Hello, $name"
echo "Hello, ${name}!"    # 推荐加大括号，避免歧义

# 只读变量
readonly PI=3.14
# 删除变量
unset name

# 命令替换（两种等价写法）
date_str=$(date +%Y%m%d)
date_str=`date +%Y%m%d`

# 特殊变量
$0   # 脚本名称
$1   # 第一个参数
$@   # 所有参数（每个独立）
$*   # 所有参数（合并为一个）
$#   # 参数个数
$?   # 上一条命令的退出码（0=成功，非0=失败）
$$   # 当前脚本的PID
$!   # 最后一个后台进程的PID

# ==================== 引号 ====================
echo "$name"    # 双引号：变量和命令替换会展开
echo '$name'    # 单引号：完全字面量，不展开
```

---

## Q2. Shell 中的条件判断怎么写？

```bash
# ==================== if 语句 ====================
# 字符串比较
if [ "$str" = "hello" ]; then
    echo "match"
elif [ "$str" = "world" ]; then
    echo "world"
else
    echo "no match"
fi

# 数值比较（不能用 = > <，要用 -eq -gt 等）
if [ "$num" -gt 10 ]; then
    echo "greater than 10"
fi

# 文件/目录判断
if [ -f "/etc/hosts" ]; then echo "是普通文件"; fi
if [ -d "/etc" ]; then echo "是目录"; fi
if [ -e "/path" ]; then echo "存在（文件或目录）"; fi
if [ -r "/path" ]; then echo "可读"; fi
if [ -w "/path" ]; then echo "可写"; fi
if [ -x "/path" ]; then echo "可执行"; fi
if [ -s "/path" ]; then echo "文件非空"; fi
if [ -L "/path" ]; then echo "是符号链接"; fi

# 逻辑运算
if [ condition1 ] && [ condition2 ]; then ...  # AND
if [ condition1 ] || [ condition2 ]; then ...  # OR
if ! [ condition ]; then ...                    # NOT

# [[ ]] 双括号（bash 扩展，更强大）
if [[ "$str" == hello* ]]; then echo "以hello开头"; fi  # 支持通配符
if [[ "$str" =~ ^[0-9]+$ ]]; then echo "是数字"; fi      # 支持正则

# (( )) 算术表达式
if (( num > 10 && num < 100 )); then echo "在10到100之间"; fi
```

**常用比较运算符速查**：
```
字符串: = != -z(空) -n(非空)
数值:   -eq -ne -lt -le -gt -ge
        等于 不等 小于 小于等 大于 大于等
```

---

## Q3. Shell 中的循环怎么写？

```bash
# ==================== for 循环 ====================
# 遍历列表
for item in apple banana cherry; do
    echo "Fruit: $item"
done

# 遍历数组
fruits=("apple" "banana" "cherry")
for item in "${fruits[@]}"; do
    echo "$item"
done

# C风格for
for ((i=0; i<10; i++)); do
    echo "i=$i"
done

# 遍历文件
for file in /var/log/*.log; do
    echo "Processing: $file"
done

# 遍历命令输出
for pid in $(pgrep java); do
    echo "Java PID: $pid"
done

# ==================== while 循环 ====================
count=0
while [ $count -lt 5 ]; do
    echo "count=$count"
    ((count++))
done

# 读取文件每一行
while IFS= read -r line; do
    echo "$line"
done < /etc/hosts

# ==================== until 循环 ====================
# 直到条件为真才停止
until [ -f "/tmp/done.flag" ]; do
    echo "waiting..."
    sleep 1
done
```

---

## Q4. Shell 函数怎么定义和使用？

```bash
# 定义函数（两种写法等价）
function greet() {
    local name=$1         # local 声明局部变量，避免污染全局
    echo "Hello, $name!"
    return 0              # 返回状态码（0-255，只能是整数）
}

greet2() {
    echo "Hello, $1!"
}

# 调用函数
greet "World"
greet2 "Linux"

# 获取函数"返回值"（用命令替换，因为return只能返回整数）
get_date() {
    echo $(date +%Y%m%d)   # 通过标准输出传递字符串"返回值"
}
today=$(get_date)
echo "Today is: $today"

# 检查函数执行结果
deploy_app() {
    # ... 部署逻辑
    if [ $? -ne 0 ]; then
        echo "ERROR: 部署失败"
        return 1
    fi
    return 0
}

if deploy_app; then
    echo "部署成功"
else
    echo "部署失败，退出"
    exit 1
fi
```

---

## Q5. Shell 脚本的错误处理最佳实践是什么？

```bash
#!/bin/bash
# 最佳实践：在脚本开头加这三行
set -e          # 遇到错误立即退出（不是0退出码就停止）
set -u          # 使用未定义变量时报错退出
set -o pipefail # 管道中任意命令失败都算失败

# 等价简写
set -euo pipefail

# 错误处理函数
error_exit() {
    echo "错误: $1" >&2    # 输出到 stderr
    exit 1
}

# trap 捕获信号和退出
cleanup() {
    echo "正在清理临时文件..."
    rm -f /tmp/myapp_temp_*
}
trap cleanup EXIT          # 脚本退出时执行（无论正常还是异常）
trap 'error_exit "脚本在第 $LINENO 行出错"' ERR   # 出错时打印行号

# 安全的文件操作
TMP_FILE=$(mktemp)         # 创建临时文件（自动生成唯一名）
trap "rm -f $TMP_FILE" EXIT  # 确保退出时删除

# 检查命令是否存在
command -v docker &>/dev/null || error_exit "请先安装 docker"

# 检查root权限
[ "$(id -u)" -eq 0 ] || error_exit "请用 root 运行此脚本"
```

---

## Q6. Shell 中如何处理字符串？

```bash
# ==================== 字符串操作 ====================
str="Hello, World"

# 获取长度
echo ${#str}           # 12

# 截取子串
echo ${str:0:5}        # Hello（从第0位取5个字符）
echo ${str:7}          # World（从第7位到结尾）
echo ${str: -5}        # World（倒数5个字符）

# 替换
echo ${str/o/0}        # Hell0, World（替换第一个）
echo ${str//o/0}       # Hell0, W0rld（替换所有）
echo ${str/World/Linux} # Hello, Linux

# 删除前缀/后缀（# 删前缀，% 删后缀，## %% 贪婪匹配）
path="/usr/local/bin/java"
echo ${path#*/}        # usr/local/bin/java（删除最短前缀 /）
echo ${path##*/}       # java（删除最长前缀，只剩文件名）
echo ${path%/*}        # /usr/local/bin（删除最短后缀）
echo ${path%%/*}       # 空（删除最长后缀）

# 大小写转换（bash 4.0+）
echo ${str,,}          # hello, world（全小写）
echo ${str^^}          # HELLO, WORLD（全大写）

# 变量默认值
echo ${name:-"default"}      # name 未定义或为空时用 default
echo ${name:="default"}      # 同上，且赋值给 name
echo ${name:?"必须设置name"} # 未定义时报错退出
```

---

## Q7. Shell 中的数组如何使用？

```bash
# ==================== 普通数组 ====================
# 定义
fruits=("apple" "banana" "cherry")
fruits[3]="date"

# 访问元素
echo ${fruits[0]}         # apple
echo ${fruits[@]}         # 所有元素
echo ${#fruits[@]}        # 元素个数

# 遍历
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# 添加元素
fruits+=("elderberry")

# 删除元素
unset fruits[1]           # 删除索引1的元素（不压缩，会有空洞）

# ==================== 关联数组（字典）====================
declare -A config
config["host"]="localhost"
config["port"]="8080"
config["env"]="production"

echo ${config["host"]}
echo ${!config[@]}        # 所有键
echo ${config[@]}         # 所有值

for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done
```

---

## Q8. 写一个实用的 Shell 脚本示例

**Java 应用部署脚本**：
```bash
#!/bin/bash
set -euo pipefail

# ==================== 配置 ====================
APP_NAME="myapp"
APP_DIR="/app/${APP_NAME}"
JAR_FILE="${APP_DIR}/${APP_NAME}.jar"
LOG_DIR="/var/log/${APP_NAME}"
PID_FILE="/var/run/${APP_NAME}.pid"
JAVA_OPTS="-Xms512m -Xmx1g -XX:+HeapDumpOnOutOfMemoryError"
PROFILE="production"

# ==================== 颜色输出 ====================
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'  # No Color

log_info()  { echo -e "${GREEN}[INFO]${NC} $1"; }
log_warn()  { echo -e "${YELLOW}[WARN]${NC} $1"; }
log_error() { echo -e "${RED}[ERROR]${NC} $1" >&2; }

# ==================== 函数 ====================
get_pid() {
    [ -f "$PID_FILE" ] && cat "$PID_FILE" || echo ""
}

is_running() {
    local pid=$(get_pid)
    [ -n "$pid" ] && kill -0 "$pid" 2>/dev/null
}

start() {
    if is_running; then
        log_warn "应用已在运行（PID=$(get_pid)）"
        return 0
    fi
    
    log_info "启动 ${APP_NAME}..."
    mkdir -p "$LOG_DIR"
    
    nohup java $JAVA_OPTS \
        -jar "$JAR_FILE" \
        --spring.profiles.active="$PROFILE" \
        > "${LOG_DIR}/app.log" 2>&1 &
    
    local pid=$!
    echo $pid > "$PID_FILE"
    
    # 等待启动（最多30秒）
    local count=0
    while ! is_running && [ $count -lt 30 ]; do
        sleep 1
        ((count++))
    done
    
    if is_running; then
        log_info "启动成功（PID=$pid）"
    else
        log_error "启动失败，查看日志: tail -f ${LOG_DIR}/app.log"
        exit 1
    fi
}

stop() {
    local pid=$(get_pid)
    if [ -z "$pid" ] || ! kill -0 "$pid" 2>/dev/null; then
        log_warn "应用未在运行"
        return 0
    fi
    
    log_info "优雅停止应用（PID=$pid，等待最多30秒）..."
    kill -TERM "$pid"
    
    local count=0
    while kill -0 "$pid" 2>/dev/null && [ $count -lt 30 ]; do
        sleep 1
        ((count++))
    done
    
    if kill -0 "$pid" 2>/dev/null; then
        log_warn "优雅停止超时，强制杀死..."
        kill -9 "$pid"
    fi
    
    rm -f "$PID_FILE"
    log_info "应用已停止"
}

status() {
    if is_running; then
        log_info "运行中（PID=$(get_pid)）"
    else
        log_info "未运行"
    fi
}

# ==================== 主流程 ====================
case "${1:-}" in
    start)   start ;;
    stop)    stop ;;
    restart) stop; start ;;
    status)  status ;;
    *)
        echo "用法: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac
```

---

## Q9. Shell 中常见的调试技巧有哪些？

```bash
# 开启调试模式（显示每条执行的命令）
bash -x script.sh         # 执行时加 -x
set -x                    # 脚本内开启（从此行开始）
set +x                    # 关闭调试

# 开启语法检查（不执行，只检查语法）
bash -n script.sh

# 查看变量和表达式
echo "变量值: [$变量名]"   # 用[]包裹，方便看空格
set -v                    # 显示原始输入行（展开前）

# 在错误点打印调用栈
err_report() {
    echo "错误在第 $1 行"
    local i=0
    while caller $i; do ((i++)); done  # 打印调用栈
}
trap 'err_report $LINENO' ERR

# 测试特定函数（注释掉主体，只测函数）
# main 函数最佳实践：防止 source 时被执行
main() {
    # 脚本主逻辑
    start_service
}

# 只在直接执行时运行 main，被 source 时不运行
[[ "${BASH_SOURCE[0]}" == "${0}" ]] && main "$@"
```

> **追问：Shell 脚本和 Python 脚本如何选择？**
> **用 Shell**：调用系统命令、文件处理、简单自动化、部署脚本、快速一次性任务；**用 Python**：复杂逻辑（条件/循环多层嵌套）、需要数据结构（字典/列表/对象）、HTTP 请求/数据库操作、错误处理要求高、需要单元测试、代码需要长期维护。  
> 经验法则：Shell 脚本超过100行就考虑改用 Python；Shell 里出现大量 `awk`/`sed` 嵌套就该用 Python 了。

---

## Q10. 如何在 Shell 中实现并发和进程管理？

```bash
# ==================== 后台执行 ====================
sleep 10 &          # 后台执行，返回PID
PID=$!              # 获取最后一个后台进程的PID
wait $PID           # 等待特定进程结束
wait                # 等待所有后台进程结束

# ==================== 并发处理多个任务 ====================
process_file() {
    local file=$1
    echo "处理: $file"
    sleep 1  # 模拟耗时
}

# 并发处理，最多5个并发
MAX_JOBS=5
count=0
for file in /data/files/*.txt; do
    process_file "$file" &
    ((count++))
    if [ $count -ge $MAX_JOBS ]; then
        wait -n 2>/dev/null || wait   # wait -n等待任一完成（bash 4.3+）
        ((count--))
    fi
done
wait  # 等待所有完成

# ==================== xargs 并发 ====================
# 更简单的并发方式
find /data -name "*.log" | xargs -P 4 -I{} gzip {}
# -P 4 表示最多4个并发进程
# -I{} 表示用文件名替换{}

# ==================== 管道和 subshell ====================
# 注意：管道中的变量赋值在 subshell 中，不影响父 shell
count=0
echo "a b c" | while read word; do
    ((count++))  # 这里的count修改不会传到外面！
done
echo "count=$count"  # 输出0，不是3

# 解决方案1：进程替换
while read word; do
    ((count++))
done < <(echo "a b c" | tr ' ' '\n')
echo "count=$count"  # 输出3
```
