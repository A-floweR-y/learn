# 包管理工具

常见 Linux 服务器上主要会碰到两套：**apt**（Advanced Packaging Tool）和 **dnf**（Dandified YUM）。

- apt：Debian 家族（Debian / Ubuntu / Mint）
- dnf：Red Hat 家族（Fedora / RHEL / CentOS Stream / Rocky）

> dnf 是 yum 的后继。新系统上 `yum` 多半是 dnf 的软链或包装，打 `yum` 实际走 dnf。老系统（如 CentOS 7）只有 yum，没有 dnf。

不用记，上机器看有哪个：

```bash
which apt   # 有路径 → 用 apt
which dnf   # 有路径 → 用 dnf
```

## 常用命令

装、卸、升级几乎都要 `sudo`。搜索不用。

| 功能 | APT | DNF |
| --- | --- | --- |
| 安装（已装则升级） | `sudo apt install <包名>` | `sudo dnf install <包名>` |
| 卸载（保留配置） | `sudo apt remove <包名>` | `sudo dnf remove <包名>` |
| 卸载并清配置 | `sudo apt purge <包名>` | 没有对应命令，`remove` 后配置常要手动删 |
| 刷新包目录（不安装） | `sudo apt update` | 一般不用单独做，`install` / `upgrade` 会自己刷新 |
| 按目录升级所有已装包 | `sudo apt upgrade` | `sudo dnf upgrade` |
| 搜索 | `apt search <关键词>` | `dnf search <关键词>` |

apt 装东西或升级前先 `update`，否则用的是本机过期目录。
