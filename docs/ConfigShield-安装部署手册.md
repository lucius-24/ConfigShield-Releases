# ConfigShield 安装部署手册

本文档面向负责安装、试用和上线 ConfigShield 的用户，说明如何通过离线包完成部署、升级、修复、卸载、备份恢复和授权导入。

ConfigShield 当前面向用户只提供一种正式安装方式：Docker 离线包安装。离线包可以通过 GitHub Releases、网盘、客户制品库、SFTP、堡垒机或 U 盘传输到目标服务器，安装过程不需要目标服务器访问公网镜像仓库。

## 1. 产品概述

ConfigShield 是网络设备配置安全治理平台，围绕设备配置和运行状态提供以下能力：

- 设备资产统一管理。
- 配置备份和历史版本留存。
- 配置差异比较和变更追踪。
- 基础巡检和巡检结果留存。
- 风险分析、安全合规、报告输出和告警集成。
- 社区版默认授权和专业版授权导入。

平台适用于网络运维、安全运营、项目交付和审计留痕场景，帮助用户把网络设备配置纳入持续治理流程。

## 2. 版本与功能

| 功能模块 | 社区版 | 专业版 |
| --- | --- | --- |
| 设备管理 | 支持 | 支持 |
| 配置备份 | 支持 | 支持 |
| 基础差异 | 支持 | 支持 |
| 基础巡检 | 支持 | 支持 |
| 风险分析 | 不支持 | 支持 |
| 安全合规 | 不支持 | 支持 |
| 报告中心 | 不支持 | 支持 |
| 集成告警 | 不支持 | 支持 |
| AI 语义分析 | 不支持 | 支持 |
| 设备额度 | 8 台 | 授权文件指定 |

系统安装后默认进入社区版，不需要导入授权文件。需要专业版功能或更高设备额度时，在系统授权页面生成申请信息，并导入由 ULC 授权系统签发的授权文件。

## 3. 部署准备

### 3.1 操作系统

推荐目标服务器：

```text
Ubuntu Server 24.04 LTS / 26.04 LTS x86_64
```

其他 Linux 发行版只要支持 Docker Engine 24+ 和 Docker Compose v2，通常也可以运行。正式项目建议优先使用 Ubuntu Server LTS。

### 3.2 必需软件

目标服务器需要已安装 Docker Engine 和 Docker Compose v2：

```bash
docker --version
docker compose version
```

验证 Docker 可用：

```bash
docker run --rm hello-world
docker compose version
```

### 3.3 硬件建议

| 场景 | CPU | 内存 | 磁盘 | 适用范围 |
| --- | --- | --- | --- | --- |
| 最低测试/演示 | 2 core | 2 GB | 20 GB | 功能验证、短期试用、少量设备 |
| 小规模生产推荐 | 2 core | 4 GB | 50 GB | 社区版或小规模专业版 |
| 较多设备/长期留存 | 4 core | 8 GB | 100 GB+ | 更多设备、更多巡检、长期保存备份和报告 |

磁盘空间主要用于数据库、设备配置备份、巡检结果、导出报告和授权文件。生产环境应结合设备数量、备份频率和保留周期预留容量。

### 3.4 网络和端口

默认 Web/API 访问端口：

```text
18004
```

如该端口已被占用，可在安装时通过 `API_HOST_PORT` 指定其他端口。MySQL 和 Redis 不向宿主机公开端口，只在 Docker 网络内部访问。

离线安装需要提前把以下文件放到目标服务器：

```text
configshield-<version>.tar.gz
configshield-<version>.tar.gz.sha256
```

## 4. 默认安装位置

默认安装目录：

```text
/opt/configshield
```

生产环境建议使用该目录，便于统一权限、备份、迁移和交接。不要把安装目录放在 `/tmp`、源码仓库目录或容器内部路径中。

主要持久化数据位于：

```text
/opt/configshield/data/mysql
/opt/configshield/data/redis
/opt/configshield/data/backups
/opt/configshield/data/exports
/opt/configshield/license
```

升级、修复和容器重建不会主动删除这些数据。

## 5. 离线包内容

解压后的离线包目录应包含：

```text
quick-offline-install.sh
offline-install.sh
install.sh
docker-compose.release.yml
backup.sh
restore.sh
README.md
release-manifest.json
SHA256SUMS
images/configshield-images.tar
```

`images/configshield-images.tar` 包含 ConfigShield 应用镜像、MySQL 镜像和 Redis 镜像。

## 6. 安装步骤

### 6.1 校验并解压

```bash
sha256sum -c configshield-<version>.tar.gz.sha256
tar -xzf configshield-<version>.tar.gz
cd configshield-<version>
sha256sum -c SHA256SUMS
```

如果校验失败，不要继续安装，应重新获取离线包。

### 6.2 默认安装

```bash
sudo bash ./quick-offline-install.sh
```

安装完成后访问：

```text
http://<服务器IP>:18004
```

首次安装未指定管理员密码时，脚本会自动创建管理员账号并生成随机密码：

```text
初始管理员账号 / Initial admin account
访问地址 / URL: http://<服务器IP>:18004
用户名 / Username: admin
密码 / Password: <随机密码>
请立即保存该密码，后续不会再次显示。 / Please save this password now. It will not be shown again.
```

### 6.3 指定端口和目录

```bash
CONFIGSHIELD_INSTALL_DIR=/opt/configshield \
API_HOST_PORT=18004 \
sudo bash ./quick-offline-install.sh
```

测试环境如不希望写入 `/opt`，可以指定当前用户目录：

```bash
CONFIGSHIELD_INSTALL_DIR=$HOME/configshield-runtime \
API_HOST_PORT=18088 \
bash ./quick-offline-install.sh
```

### 6.4 指定初始管理员密码

```bash
CONFIGSHIELD_ADMIN_USERNAME=admin \
CONFIGSHIELD_ADMIN_PASSWORD='请替换为自己的强密码' \
CONFIGSHIELD_ADMIN_REAL_NAME='System Admin' \
sudo bash ./quick-offline-install.sh
```

## 7. 安装后检查

健康检查：

```bash
curl -fsSL http://127.0.0.1:18004/live
```

查看容器状态：

```bash
bash ./quick-offline-install.sh status
```

查看日志：

```bash
bash ./quick-offline-install.sh logs
```

## 8. 常用运维动作

```bash
bash ./quick-offline-install.sh status
bash ./quick-offline-install.sh repair
bash ./quick-offline-install.sh upgrade
bash ./quick-offline-install.sh restart
bash ./quick-offline-install.sh logs
bash ./quick-offline-install.sh uninstall
```

动作说明：

| 动作 | 说明 |
| --- | --- |
| `status` | 查看服务和容器状态。 |
| `repair` | 重新生成运行配置并修复容器状态。 |
| `upgrade` | 加载新离线包镜像并升级服务。 |
| `restart` | 重启 ConfigShield 服务。 |
| `logs` | 查看服务日志。 |
| `uninstall` | 停止并移除容器，不删除持久化数据。 |

## 9. 升级

1. 备份当前系统。
2. 获取新版本离线包。
3. 解压并进入新版本目录。
4. 执行升级：

```bash
CONFIGSHIELD_INSTALL_DIR=/opt/configshield \
sudo bash ./quick-offline-install.sh upgrade
```

升级会复用原安装目录中的数据库、备份、导出文件和授权文件。

## 10. 修复

当 Compose 文件、环境变量文件或容器状态异常时，可执行：

```bash
CONFIGSHIELD_INSTALL_DIR=/opt/configshield \
sudo bash ./quick-offline-install.sh repair
```

修复动作不会主动删除持久化数据。

## 11. 卸载

```bash
CONFIGSHIELD_INSTALL_DIR=/opt/configshield \
sudo bash ./quick-offline-install.sh uninstall
```

卸载只停止并移除容器，不删除 `/opt/configshield` 下的数据。如需彻底清理，应在确认已备份后由管理员手动删除安装目录。

## 12. 备份与恢复

### 12.1 备份

```bash
CONFIGSHIELD_INSTALL_DIR=/opt/configshield \
bash ./backup.sh
```

备份文件会包含数据库导出、应用数据、配置备份、导出文件和授权目录。

### 12.2 恢复

恢复前先完成同版本安装，然后执行：

```bash
CONFIGSHIELD_INSTALL_DIR=/opt/configshield \
bash ./restore.sh /path/to/configshield-backup-YYYYmmddTHHMMSSZ.tar.gz
```

恢复完成后检查服务状态和登录功能。

## 13. 授权

### 13.1 社区版默认授权

系统无授权文件时自动进入社区版：

```text
版本：社区版
设备额度：8 台
功能范围：设备管理、配置备份、基础差异、基础巡检
```

### 13.2 专业版授权

专业版授权流程：

1. 登录 ConfigShield。
2. 进入“系统设置”中的产品授权页面。
3. 点击“生成授权申请信息”。
4. 将申请信息提交给授权签发方。
5. 获取授权文件后，在授权页面上传。
6. 刷新授权状态，确认显示为专业版。

授权文件由 ULC 授权系统签发。ConfigShield 只负责授权文件导入、签名校验、机器绑定校验、有效期校验和功能开关控制。

## 14. 故障排查

### 14.1 端口无法访问

检查容器状态：

```bash
bash ./quick-offline-install.sh status
```

检查端口是否被占用：

```bash
sudo ss -lntp | grep 18004
```

如端口冲突，重新指定 `API_HOST_PORT` 安装或修复。

### 14.2 安装后忘记密码

如果首次安装时使用随机密码但未保存，需要由管理员进入容器或数据库执行密码重置流程。生产环境建议安装时显式指定初始管理员密码，并在首次登录后立即更换。

### 14.3 Docker 权限不足

如果当前用户不能执行 Docker 命令，可以使用 `sudo` 执行安装脚本，或由系统管理员把当前用户加入 Docker 用户组。

### 14.4 校验失败

如果 `sha256sum -c` 失败，说明离线包或校验文件不匹配。不要继续安装，应重新下载或重新传输离线包。

## 15. 安全建议

- 生产环境使用强密码，并在首次登录后修改默认管理员信息。
- 限制服务器 SSH 登录来源。
- 将 `/opt/configshield` 纳入主机备份策略。
- 定期导出备份文件并保存到独立介质。
- 授权文件和备份文件应按敏感文件管理。
