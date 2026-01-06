# Kolla-Ansible All-in-One OpenStack 部署指南

> 基于 OpenStack Kolla-Ansible 2025.1 官方文档

## 概述

本指南提供使用 Kolla Ansible 在裸金属服务器或虚拟机上部署 OpenStack 的分步说明。

**推荐前置知识**：在运行 Kolla Ansible 之前，建议先学习 [Ansible](https://docs.ansible.com) 和 [Docker](https://docs.docker.com) 的基础知识。

---

## 1. 主机要求

| 配置项 | 最低要求 |
|--------|----------|
| 网络接口 | 2 个 |
| 内存 | 8GB |
| 磁盘空间 | 40GB |

**支持的操作系统**：详见 [支持矩阵](https://docs.openstack.org/kolla-ansible/2025.1/user/support-matrix)

---

## 2. 安装依赖

> 以下命令通常需要 root 权限执行。

### 2.1 系统包安装

**对于 Debian/Ubuntu：**
```bash
sudo apt update
sudo apt install git python3-dev libffi-dev gcc libssl-dev libdbus-glib-1-dev
```

**对于 CentOS/Rocky：**
```bash
sudo dnf install git python3-devel libffi-devel gcc openssl-devel python3-libselinux
```

### 2.2 Python 虚拟环境配置

```bash
# 安装虚拟环境依赖 (Debian/Uuntu)
sudo apt install python3-venv

# 创建并激活虚拟环境
python3 -m venv /path/to/venv
source /path/to/venv/bin/activate

# 升级 pip
pip install -U pip
```

> 使用虚拟环境安装 Kolla Ansible 及其依赖可以避免与系统 site-packages 冲突。

---

## 3. 安装 Kolla-Ansible

```bash
# 1. 使用 pip 安装 Kolla-Ansible
pip install git+https://opendev.org/openstack/kolla-ansible@|KOLLA_BRANCH_NAME|

# 2. 创建配置目录
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla

# 3. 复制配置文件
cp -r /path/to/venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla

# 4. 复制 inventory 文件
cp /path/to/venv/share/kolla-ansible/ansible/inventory/all-in-one .
```

---

## 4. 安装 Ansible Galaxy 依赖

```bash
kolla-ansible install-deps
```

---

## 5. 初始化配置

### 5.1 Inventory 文件

Kolla Ansible 提供了两个示例 inventory 文件：
- `all-in-one`：单机部署
- `multinode`：多节点部署

本指南使用 `all-in-one` 进行单节点 OpenStack 部署。

### 5.2 配置密码

密码存储在 `/etc/kolla/passwords.yml` 文件中。所有密码初始为空，需要手动填写或使用随机密码生成器：

```bash
kolla-genpwd
```

### 5.3 配置 globals.yml

`globals.yml` 是 Kolla Ansible 的主要配置文件，位于 `/etc/kolla/globals.yml`。

#### 5.3.1 镜像选项

```yaml
# 推荐使用 Rocky Linux 9 或 Ubuntu 24.04
kolla_base_distro: "rocky"
```

**可选的 Linux 发行版**：
- CentOS Stream (`centos`)
- Debian (`debian`)
- Rocky (`rocky`)
- Ubuntu (`ubuntu`)

#### 5.3.2 AArch64 架构选项

```yaml
# AArch64 架构需要设置标签后缀
openstack_tag_suffix: "-aarch64"
```

#### 5.3.3 网络配置

```yaml
# 管理网络接口
network_interface: "eth0"

# Neutron 外部网络接口（无 IP 地址）
neutron_external_interface: "eth1"

# 内部 VIP 地址（用于高可用）
kolla_internal_vip_address: "10.1.0.250"
```

> **注意**：`neutron_external_interface` 应该不带 IP 地址，否则实例将无法访问外部网络。

#### 5.3.4 启用额外服务

默认情况下 Kolla Ansible 提供基础计算服务，可以通过设置 `enable_*` 启用更多服务：

```yaml
# 例如启用 Cinder
enable_cinder: "yes"
```

**可用服务列表**：[OpenStack Services](https://github.com/openstack/kolla-ansible/blob/master/README.rst#openstack-services)

#### 5.3.5 多配置文件管理

为实现更细粒度的控制，可以在 `/etc/kolla/globals.d/` 目录下创建独立的 `.yml` 文件来启用选项：

```bash
mkdir -p /etc/kolla/globals.d
```

例如创建 `cinder.yml` 来管理 Cinder 相关配置，而不影响主 `globals.yml` 文件。

---

## 6. 部署 OpenStack

### 6.1 引导服务器

```bash
kolla-ansible bootstrap-servers -i ./all-in-one
```

### 6.2 预部署检查

```bash
kolla-ansible prechecks -i ./all-in-one
```

### 6.3 部署

```bash
kolla-ansible deploy -i ./all-in-one
```

部署完成后，OpenStack 应该已经启动并可正常运行！

如果出现错误，请参考 [故障排除指南](https://docs.openstack.org/kolla-ansible/2025.1/user/troubleshooting.html)。

---

## 7. 使用 OpenStack

### 7.1 安装 OpenStack CLI

```bash
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/|KOLLA_OPENSTACK_RELEASE|
```

### 7.2 生成 clouds.yaml

```bash
kolla-ansible post-deploy
```

配置文件将生成在 `/etc/kolla/clouds.yaml`，可以复制到：
- `/etc/openstack`
- `~/.config/openstack`
- 或设置环境变量 `OS_CLIENT_CONFIG_FILE`

### 7.3 初始化网络和镜像

```bash
/path/to/to/venv/share/kolla-ansible/init-runonce
```

> ⚠️ **警告**：`init-runonce` 脚本仅用于演示目的，不一定必须运行。根据自定义配置的不同，它可能无法工作或与您要创建的资源冲突。

---

## 8. 部署架构图

```
┌─────────────────────────────────────────────────────────┐
│                  All-in-One 节点                         │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────┐   │
│  │              Docker 容器层                        │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │   │
│  │  │  Keystone│ │   Nova   │ │ Neutron  │        │   │
│  │  │   (Id)   │ │ (Compute)│ │ (Network)│        │   │
│  │  └──────────┘ └──────────┘ └──────────┘        │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐        │   │
│  │  │  Glance  │ │  Cinder  │ │   Horizon│        │   │
│  │  │ (Image)  │ │ (Block)  │ │ (Dashboard)       │   │
│  │  └──────────┘ └──────────┘ └──────────┘        │   │
│  │  ... 更多 OpenStack 服务                         │   │
│  └─────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────┤
│  ┌────────────────┐        ┌────────────────┐          │
│  │ network_interface │      │ neutron_external │       │
│  │    (eth0)         │      │  _interface      │       │
│  │  管理网络          │      │   (eth1)         │       │
│  └────────────────┘        └────────────────┘          │
└─────────────────────────────────────────────────────────┘
```

---

## 9. 常用命令速查

| 命令 | 说明 |
|------|------|
| `kolla-ansible bootstrap-servers` | 引导服务器安装依赖 |
| `kolla-ansible prechecks` | 预部署检查 |
| `kolla-ansible deploy` | 部署 OpenStack |
| `kolla-ansible destroy` | 销毁部署 |
| `kolla-ansible reconfigure` | 重新配置 |
| `kolla-ansible upgrade` | 升级部署 |
| `kolla-genpwd` | 生成随机密码 |
| `kolla-ansible post-deploy` | 部署后配置 |
| `kolla-ansible install-deps` | 安装 Ansible Galaxy 依赖 |

---

## 10. 故障排除

遇到问题时：

1. 查看日志：`/var/log/kolla/`
2. 检查容器状态：`docker ps`
3. 参考 [官方故障排除文档](https://docs.openstack.org/kolla-ansible/2025.1/user/troubleshooting.html)

---

## 参考资料

- [Kolla-Ansible 官方文档](https://docs.openstack.org/kolla-ansible/2025.1/)
- [Kolla-Ansible GitHub](https://github.com/openstack/kolla-ansible)
- [OpenStack 官方文档](https://docs.openstack.org/)
- [支持矩阵](https://docs.openstack.org/kolla-ansible/2025.1/user/support-matrix)
- [可用服务列表](https://github.com/openstack/kolla-ansible/blob/master/README.rst#openstack-services)

---

*文档最后更新：2025-12-19*
