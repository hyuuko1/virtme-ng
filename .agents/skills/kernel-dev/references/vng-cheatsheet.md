# virtme-ng (vng) 常用高级参数速查手册

本文档为 `kernel-dev` 技能的进阶参考，收录内核调试与测试常用的 `vng` 参数组合。

---

## 1. 核心运行参数

| 参数 | 说明 | 示例 | 适用场景 |
| :--- | :--- | :--- | :--- |
| `-r <kimg>` | 指定已编译好的内核镜像路径 | `-r /code/linux/o/arch/x86/boot/bzImage` | 直接运行指定内核，跳过任何编译/下载 |
| `--rwdir <path>` | 读写挂载宿主机目录到虚拟机同名路径 | `--rwdir /code/linux/o/vng-test` | 测试脚本与日志持久化落盘 |
| `--user <user>` | 指定虚拟机内登录用户 | `--user root` | 获取原生 root 特权，无需 sudo 密码 |
| `--mods none` | 禁用外部内核模块探测 | `--mods none` | 内核功能全内置（`=y`），加速启动 |
| `--mods auto` | 自动探测并挂载模块（默认） | `--mods auto` | 需要测试内核自带动态模块时 |
| `--dry-run` | 命令行预览 | | |

---

## 2. 硬件资源与拓扑配置

| 参数 | 格式与说明 | 典型用法 |
| :--- | :--- | :--- |
| `-p <cpus>`, `--cpus` | 限制虚拟机 vCPU 数量 | `-p 1`（单核复现并发/竞态漏洞）<br>`-p 4`（多核性能测试） |
| `-m <mem>`, `--memory` | 设置虚拟机内存大小 | `-m 512M`（低内存压测）<br>`-m 4G`（大内存基准测试） |
| `--numa` | 模拟多 NUMA 节点拓扑 | `--numa 2G,cpus=0-1 --numa 2G,cpus=2-3` |

---

## 3. 内核命令行注入 (`--append`)

通过 `--append` 可以向 guest 内核传递 cmdline 参数，用于早期调试或开启特定检测机制：

```bash
# 开启早期串口输出与详细日志级别
vng -r /code/linux/o/arch/x86/boot/bzImage --append "earlyprintk=serial,ttyS0 console=ttyS0 loglevel=8"

# 开启 SLUB 调试检测内存越界/UAF
vng -r /code/linux/o/arch/x86/boot/bzImage --append "slub_debug=FZPU"

# 打印内核 initcall 执行耗时
vng -r /code/linux/o/arch/x86/boot/bzImage --append "initcall_debug"
```

---

## 4. 网络模式配置

| 参数 | 行为 | 适用场景 |
| :--- | :--- | :--- |
| `-n user` | 用户态 NAT 网络（默认，自动分配 DHCP） | 虚拟机内需要访问外部网络 |
| `-n none` | 完全禁用网络 | 隔离纯计算测试，缩短启动时间 |
| `--no-dhcp` | 保留网卡但不启动 DHCP 客户端 | 需手动配置 IP 或自建虚拟网络拓扑 |

---

## 5. 调试与跟踪支持

* **GDB 源码级单步调试**：
  ```bash
  # 宿主机启动虚拟机并监听 GDB 端口（默认 1234）
  vng -r /code/linux/o/arch/x86/boot/bzImage --gdb --append "nokaslr"

  # 宿主机另起终端连接调试
  gdb /code/linux/o/vmlinux -ex "target remote :1234"
  ```
* **控制台回退模式**：
  若遇到 Rust 版本 `virtme-ng-init` 与某些极老发行版 rootfs 冲突，可添加 `--no-virtme-ng-init` 回退至 Bash 版 init 脚本。
