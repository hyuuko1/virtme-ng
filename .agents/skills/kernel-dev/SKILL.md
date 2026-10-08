---
name: kernel-dev
description: Linux 内核开发、编译与测试指南。当需要配置或编译内核/模块、使用 vng 启动虚拟机测试、排查内核问题或撰写内核文档时调用。
---

# Linux 内核开发与测试指南

本技能规范了从内核编译、配置修改，到使用 `vng` 进行测试与问题排查的完整工程闭环。

---

## 一、 核心铁律

1. **root 用户权限规则**：
   - **宿主机世界**：**绝对禁止在宿主机执行 `sudo` 或任何提权命令**。若遇到权限不足等无法解决的问题，必须立即停止并提示用户手动处理。
   - **虚拟机世界**：默认以 `root` 运行（`--user root`）。Agent 拥有虚拟机内的全量控制权（可自由执行 `insmod`、`dmesg` 等）。
2. **零多余防御检查 (Fail-Fast)**：
   - **启动前**：**禁止**运行 `which vng`、`ls -l bzImage`、`python3 -c 'import virtme_ng;'` 等防御性检查命令。无需检查命令或文件是否存在，直接执行 `vng ...` 命令，若失败则通过错误输出排查问题或求助用户。
   - **执行后**：场景 A 批处理模式下，`vng` 退出即代表虚拟机已关机且 QEMU 进程已完全回收，**禁止**运行 `ps aux | grep qemu` 等后置存活探针。
3. **绝对禁止在虚拟机内部编译代码**（编译极慢且浪费资源）。所有内核代码、模块及测试程序必须在宿主机完成编译。
4. 虚拟机内临时挂载或临时文件放置于可写目录 `/tmp/`（根文件系统默认挂载为只读）。
5. **严禁**在 `--remote-cmd` 中添加 `&` 试图在 VM 内后台运行，在虚拟机内使用 `&` 会导致 SSH 客户端因继承 I/O 管道而卡死挂起（或断连时子进程异常终止）。长耗时任务应直接使用宿主机的 `run_command` 工具调用执行命令，原生支持转为后台执行并在任务完成时自动发送通知唤醒 Agent 无需轮询。

---

## 二、 代码编写与构建规范

- 内核代码在 `/code/linux`。内核构建产物统一存放于 `/code/linux/o/` 目录下。
- 若要写代码进行实验，必须新建 git 仓库，用 git 管理代码。
- 执行复杂长串命令时，优先写脚本，若没必要写脚本，为了可读性，长串命令应换行。
- 写脚本时要考虑失败退出时的回退。
- 禁止将脚本写到 /tmp 目录。
- 必须良好组织 代码、脚本和日志等等文件 的目录结构，必要时新建目录，避免无关的文件混在一起。

### 1. 编译内核与模块

编译时必须使用以下四个选项：

- `LLVM=1`：使用 Clang/LLVM 工具链编译
- `O=o`：编译产物（包括 `.config`）输出至 `o/` 目录
- `LOCALVERSION=`：禁用内核版本自定义后缀
- `-j$(nproc)`：使用多线程并发编译

```bash
# 在 /code/linux 目录下执行
# 1. 编译内核镜像 (产物为 /code/linux/o/arch/x86/boot/bzImage)
make LLVM=1 O=o LOCALVERSION= -j$(nproc) bzImage

# 2. 编译内核模块
make LLVM=1 O=o LOCALVERSION= -j$(nproc) modules
```

### 2. 内核配置修改规则

若需要修改内核配置（`.config`），**必须先备份原 `.config`**，备份文件名必须附带时间戳：

```bash
cp /code/linux/o/.config /code/linux/o/.config.$(date '+%Y%m%d_%H%M%S').bak
```

### 3. 外部模块构建规范

外部内核模块同样必须使用 Clang 编译，模块源码目录下需提供如下格式的 `Makefile`：

```makefile
obj-m := mod.o
mod-objs := a.o b.o

KDIR ?= /code/linux
M    := $(shell pwd)

all:
	$(MAKE) LLVM=1 -C $(KDIR) O=o M=$(M) -j$(nproc) modules

clean:
	$(MAKE) LLVM=1 -C $(KDIR) O=o M=$(M) -j$(nproc) clean
```

#### 4. 外部内核模块编写规范

- 不要调用未被导出的内核符号。

---

## 三、 测试准备与产物持久化

为了防止虚拟机崩溃退出后丢失日志，统一使用宿主机的读写共享目录。

1. **共享持久化目录**：
   - 固定使用 `--rwdir /code/linux/o/vng-test/` 读写挂载，存放测试用例、脚本与输出日志。
   - 宿主机预先创建该目录：`mkdir -p /code/linux/o/vng-test`。
   - **缓存刷新规则**：当宿主机写入新内容后，若要想虚拟机读到新内容，需要在虚拟机内执行 `echo 3 > /proc/sys/vm/drop_caches` 刷新缓存。
2. **测试脚本模板规范**：
   写在宿主机 `/code/linux/o/vng-test/<dir>/<test_name>.sh` 的脚本必须具备时间感知与内核日志体检能力：

```bash
#!/usr/bin/env bash
set -euo pipefail

# 时间戳打印辅助函数
log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

start_time=$SECONDS
log "=== 开始执行测试: $(basename "$0") ==="

# 清理历史 dmesg（防止前序测试干扰）
dmesg -C 2>/dev/null || true

# 执行宿主机已编译好的测试用例
/code/linux/o/vng-test/my_test_bin

# 内核健康体检 (Kernel Hygiene Check)
log "检查内核是否存在告警或崩溃堆栈..."
if dmesg | grep -E -q "kernel BUG|WARNING:|Oops|BUG:|Call Trace:|KASAN:|lockdep"; then
    log "ERROR: 检测到内核告警或崩溃！"
    dmesg -l warn,err
    exit 1
fi

duration=$((SECONDS - start_time))
log "=== 测试顺利通过，耗时 ${duration}s ==="
```

---

## 四、 虚拟机测试执行工作流

内核镜像统一指定为 `/code/linux/o/arch/x86/boot/bzImage`。根据测试性质选择以下两种场景之一：

```
                [已具备 /code/linux/o/arch/x86/boot/bzImage]
                                     │
                     ┌───────────────┴───────────────┐
                     ▼                               ▼
             【场景 A：单次批处理】             【场景 B：常驻 SSH 探索】
             (回归/固定/自动化用例)             (交互调试/现场保留/多步摸索)
```

### 场景 A：单次批处理模式 (Ephemeral Batch Mode)

适用于流程确定、可一键跑完的回归验证用例。执行完毕后虚拟机自动退出。

1. **宿主机准备工作**：在宿主机编译好测试二进制，并将测试脚本写入 `/code/linux/o/vng-test/<dir>/<test_name>.sh`（添加执行权限 `chmod +x`）。若用例较简单，无需写脚本，直接用命令即可。
2. **启动并执行**：
   ```bash
   vng -r /code/linux/o/arch/x86/boot/bzImage --user root --rwdir /code/linux/o/vng-test -- /code/linux/o/vng-test/<dir>/<test_name>.sh
   ```
3. **完成判定**：命令退出码为 `0` 且控制台未打印内核告警。`vng` 会自动透传脚本退出码；若内核发生 Panic，`vng` 自动以错误码退出。

---

### 场景 B：常驻 SSH 探索模式 (Stateful Daemon + SSH)

适用于调试未知 Bug、需要逐步排查、或依赖多步骤保留中间状态的探索性测试。

1. **启动后台虚拟机**：
   使用 `run_command` 工具启动后台守护进程：
   - `CommandLine`: `vng -r /code/linux/o/arch/x86/boot/bzImage --user root --ssh 3 --rwdir /code/linux/o/vng-test`
     - 解读：
       - 启动虚拟机，通过 virtiofs 只读共享宿主机的根目录，部分目录在虚拟机内 overlay 挂载，upperdir 在虚拟机的 tmpfs，写文件不影响宿主机。
       - 使用内核 `/code/linux/o/arch/x86/boot/bzImage`。
       - 虚拟机内登录为 root 用户。
       - 虚拟机的 vsock CID 为 3，可通过 vsock 进行 SSH 连接。
       - 将宿主机的 `/code/linux/o/vng-test` 共享给虚拟机，并挂载为可读写目录，写文件对宿主机可见。
   - `IsDaemon`: `true`
   - `WaitMsBeforeAsync`: `3000`
2. **就绪探测**：
   执行轻量探针确认虚拟机及 SSH 服务完全就绪：

   ```bash
   vng --ssh-client 3 -- uname -r
   ```

   - 若返回内核版本字符串，说明握手成功，进入测试步骤。
   - 若连接失败，**切勿无休止重试**，查阅下一节的自愈响应协议。

3. **交互式执行测试**：
   使用 SSH 客户端逐步下发命令。因为已指定 `--user root`，命令默认以 root 身份执行。

   ```bash
   # 加载模块或执行测试
   vng --ssh-client 3 --remote-cmd "insmod /code/linux/o/vng-test/my_mod.ko"
   vng --ssh-client 3 --remote-cmd "/code/linux/o/vng-test/test_step.sh"
   # 宿主机写入新内容后，在虚拟机内刷新缓存以读到新内容
   vng --ssh-client 3 --remote-cmd "echo 3 > /proc/sys/vm/drop_caches"
   ```

4. **必须执行的环境清理**：
   测试结束或排查完毕后，**必须彻底强制关闭虚拟机**以释放 VSOCK CID 3 及系统资源：
   ```bash
   vng --ssh-client 3 -- poweroff -f
   ```
   客户端返回 `client_loop: send disconnect: Connection reset by peer` 或退出码 `255` 为预期内的断连信号，表示虚拟机已断电退出。
   若虚拟机已死锁失去响应，调用 `manage_task` 工具对后台启动任务执行 `kill`。

---

## 五、 自愈响应

秉持“启动前不检查、遇错后精准自愈”原则。发生错误时，对照下表执行自愈：

| 错误特征                                                      | 根本原因                                           | 自愈与处置动作                                                                                      |
| :------------------------------------------------------------ | :------------------------------------------------- | :-------------------------------------------------------------------------------------------------- |
| `guest-cid=3 ... address already in use` 或 QEMU 端口绑定失败 | 上一次运行的 QEMU 进程残留，占用了 VSOCK CID 3     | 1. 执行 `pkill -f "guest-cid=3"` 杀灭孤儿进程<br>2. **仅重试 1 次**启动命令                         |
| `cannot find kernel image` 或找不到 `bzImage` 文件            | 内核尚未编译或输出目录被清理                       | **立即停下来询问用户**是否需要编译内核，得到确认后再编译                                            |
| SSH 执行时报 `Connection refused` 或探测超时                  | 内核在早启动期崩溃或驱动死锁导致 SSH 未启动        | 1. **严禁盲目轮询**<br>2. 调阅后台 QEMU 任务的标准输出日志<br>3. 用 `vng -v -r...` 查看详细启动日志 |
| `vng: error: argument --qemu-opts/-o: expected one argument`  | QEMU 参数开头的 `-` 被宿主机 `argparse` 误判为选项 | 在参数字符串首位增加一个空格，例如 `--qemu-opts " -device..."`                                      |

---

## 六、 日志

排查测试问题时，必须清楚区分两类日志的采集途径：

1. **用户态业务日志**：
   - 存放在宿主机挂载的 `/code/linux/o/vng-test/` 目录下。
   - 适用于分析测试用例自身打印的测试统计、性能指标及测试步骤输出。
2. **内核崩溃与串口日志**：
   - 内核 Panic 或 Oops 时，文件系统通常已不可写或无法同步，`--rwdir` 中不会有由 guest 写入的崩溃日志文件。
   - **崩溃堆栈通过以下方式捕获**：
     - 场景 A：直接查看 `vng -r ...` 命令的标准输出/错误。
     - 场景 B：调阅启动 `vng ... --ssh 3` 的后台任务日志。

---

## 七、 文档撰写规范

在内核开发过程中撰写技术总结或记录文档时：

- 文档**必须**写入到 `/code/blog/docs/` 中的某个子目录，**严禁**写入到 `/code/linux/` 仓库内。
- 撰写文档时，**严禁参考 `/code/blog/docs/` 中的其他文档**，保持独立客观。

---

## 八、 进阶参考与附录

- 有关限制 CPU/内存规格（`-p`, `-m`）、注入启动参数（`--append "slub_debug=FZPU"`）、日志等产物持久化落盘以及 GDB 调试等高级用法，查阅：[vng 常用高级参数速查手册](references/vng-cheatsheet.md)
- 若要看 vng 的源码，请查阅 /code/other/virtme-ng/AGENTS.md

## 九、 其他

- `/code/linux/o/vng-test` 只是一个默认的示例路径，不是必须要将代码脚本日志放在此路径，请根据实际情况，使用用户要求的目录。
