# AGENTS.md

virtme-ng 仓库 Agent 操作指南。

若要使用 vng 启动内核进行测试，请使用 kernel-dev 技能。

## 系统架构与调用链路

virtme-ng 采用两层分层架构，`virtme_ng` 负责面向用户的高级编排，底层通过调用 `virtme/commands/` 对应的独立命令完成虚拟化与配置生成：

```
[用户 / AI Agent / CLI]
       │
       ▼
vng / virtme-ng (`virtme_ng/run.py`)  <─── vng-mcp (`virtme_ng/mcp.py`)
       │
       ├──► 内核配置：调用 `virtme-configkernel` (`virtme/commands/configkernel.py`)
       │
       ├──► 内核源码构建：本地 `make` 或远程 SSH (`--build-host`)
       │
       └──► 虚拟机启动：调用 `virtme-run` (`virtme/commands/run.py`)
                                │
                                ├──► 组装底层 QEMU 参数 (`qemuargs`，设备/内存/网络/挂载)
                                │
                                └──► 启动 QEMU 进程
                                         │
                                         ▼
                             [Guest 虚拟机内 PID 1]
                                 ├── 首选: `virtme_ng_init` (Rust 静态编译)
                                 └── 回退: `virtme-init` (Bash 脚本)
```

### 1. 顶层编排与 CLI (`virtme_ng/` 及根入口)

- **`vng` / `virtme-ng`**：CLI 用户主入口（指向 `virtme_ng.run:main`）。
- **`virtme_ng/run.py`**：顶层流程编排引擎。负责命令行参数解析（`make_parser`）、内核获取（支持 git 切换及 mainline 下载）、组织参数并以子进程方式**调用 `virtme-configkernel` 与 `virtme-run`**。
- **`vng-mcp` & `virtme_ng/mcp.py`**：Model Context Protocol (MCP) 服务端，向 Agent 暴露内核编译配置与测试运行工具。
    - _Agent 注意事项_：构建内核时必须通过宿主 Shell 执行 `vng -v --build`，切勿在虚拟机内执行编译。
- **`virtme_ng/mainline.py`**：Ubuntu/Debian mainline 预编译内核的检索、自动下载与 deb 解压。
- **`virtme_ng/utils.py` & `spinner.py`**：子进程封装、系统配置读取（`~/.config/virtme-ng/`）与终端进度指示。

### 2. 底层虚拟化命令与核心库 (`virtme/`)

- **`virtme/commands/`**（底层独立命令实现层，均由 `virtme_ng/run.py` 调度调用）：
    - **`virtme/commands/run.py` (`virtme-run`)**：**底层虚拟化核心，真正负责 QEMU 参数组装与子进程管理**。解析 CPU/内存/NUMA，配置 virtiofsd 守护进程与 9p/virtio-blk 共享挂载，拼接完整 `qemuargs` 启动并监控 QEMU。
    - **`virtme/commands/configkernel.py` (`virtme-configkernel`)**：内核配置命令。校验并生成满足测试需求的最小化内核 `.config`（合并 defconfig 与 virtme 必要配置项）。
    - **`virtme/commands/mkinitramfs.py` (`virtme-mkinitramfs`)**：独立生成 initramfs 镜像的 CLI 命令包装。
- **`virtme/` 支撑模块**：
    - **`virtme/qemu_helpers.py`**：QEMU 能力检测、设备参数转义与选项构造辅助。
    - **`virtme/architectures.py`**：多架构（x86_64, aarch64, arm, riscv, s390x 等）CPU 机器类型、控制台与启动参数映射。
    - **`virtme/kernel_image.py`**：内核镜像格式探测与解压（bzImage、vmlinux、zImage 等）。
    - **`virtme/mkinitramfs.py` & `cpiowriter.py`**：initramfs 内存盘生成逻辑与纯 Python CPIO 归档写入器。
    - **`virtme/modfinder.py` & `virtmods.py`**：内核模块依赖分析、目录查找与必备虚拟化模块列表。

### 3. Guest 虚拟机环境 (`virtme_ng_init/` 与 `virtme/guest/`)

- **`virtme_ng_init/` (Rust)**：极速 PID 1 guest init 进程。静态编译安装为 `virtme/guest/bin/virtme-ng-init`。负责 guest 启动时挂载核心文件系统（rootfs、virtiofs/9p、devtmpfs、proc、sysfs、run 等）、DHCP 网络配置与终端设置，最终 `exec` 进入测试命令或交互 Shell。
- **`virtme/guest/virtme-init` (Bash)**：备用 Bash 版 guest init 脚本（缺失静态二进制或指定 `--no-virtme-ng-init` 时回退）。
- **`virtme/guest/virtme-*-script`**：虚拟机内特定服务的辅助脚本（网络 dhcp、sshd 代理、音频与 snapd 等）。

### 4. 辅助体系 (`tools/`, `docs/`)

- **`tools/`**：Nix Flake 与包构建定义（`flake.nix`, `package.nix`）。
- **`docs/agents/`**：供 Agent 协作调用的本地工单规范、分流标签与领域模型文档。

## 构建与开发

本项目已使用 `BUILD_VIRTME_NG_INIT=1 uv tool install -e .` 安装。

## 验证与门禁

提交代码前确保以下检查全部通过：

- **自动化 Lint & Format**：运行 `pre-commit run --all-files`（统一调用 `ruff` 检查及格式化、`cargo clippy --deny warnings`、`cargo fmt`、`shellcheck` 与 `shfmt`）。
- **Rust 单元测试**：在 `virtme_ng_init/` 目录执行 `cargo test`。
- **内核启动冒烟测试**：在具备 QEMU/KVM 支持的环境下，运行 `vng -r -- uname -r` 验证端到端内核启动。

## 提交规范

- **强制 DCO 签名**：所有 commit 必须附带 DCO 认证签名，执行 `git commit -s` 生成 `Signed-off-by: Full Name <email>`。
- **Commit 标题格式**：遵循 `<subsystem>: <imperative summary>` 结构（例如 `vng, run: fix module dir detection`、`virtme-init: support NixOS guests`）。
- **Commit 正文**：详细阐明修改背景、问题起因、排查结论及具体方案。

## Agent skills

### Issue tracker

本地 Markdown 工单（保存在 `.scratch/<feature>/` 目录下）。参见 `docs/agents/issue-tracker.md`。

### Triage labels

使用 5 个标准 Triage 角色标签（`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`）。参见 `docs/agents/triage-labels.md`。

### Domain docs

单上下文架构（根目录 `GLOSSARY.md` 与 `docs/adr/`）。参见 `docs/agents/domain.md`。
