# Kolla-Ansible 快速入门指南（部署/评估）

> **原文版本**: kolla-ansible 20.3.1.dev7
> **文档地址**: [Quick Start for deployment/evaluation](https://docs.openstack.org/project-deploy-guide/kolla-ansible/2025.1/quickstart.html)

本指南提供使用 Kolla Ansible 在裸机服务器或虚拟机上部署 OpenStack 的分步说明。对于开发者，我们还提供 [开发者快速入门指南](https://docs.openstack.org/kolla-ansible/2025.1/user/quickstart-development.html)。

---

## 推荐阅读

在运行 Kolla Ansible 之前，建议先学习 [Ansible](https://docs.ansible.com/) 和 [Docker](https://docs.docker.com/) 的基础知识。

---

## 主机要求

主机必须满足以下最低要求：

- 2 个网络接口
- 8GB 内存
- 40GB 磁盘空间

有关支持的宿主操作系统的详细信息，请参阅[支持矩阵](https://docs.openstack.org/kolla-ansible/2025.1/user/support-matrix)。Kolla Ansible 支持受支持操作系统提供的默认 Python 3.x 版本。更多信息请参阅[测试运行时](https://docs.openstack.org/project-deploy-guide/kolla-ansible/2025.1/tested-runtimes)。

---

## 安装依赖

本节中通常使用系统包管理器的命令必须以 root 权限运行。

通常建议使用虚拟环境来安装 Kolla Ansible 及其依赖项，以避免与系统 site-packages 冲突。请注意，这与用于远程执行的虚拟环境的使用无关，这部分内容在[虚拟环境](https://docs.openstack.org/kolla-ansible/2025.1/user/virtual-environments.html)中描述。

### 步骤 1: 更新软件包索引（Debian/Ubuntu）

```bash
sudo apt update
```

### 步骤 2: 安装 Python 构建依赖

**对于 CentOS 或 Rocky：**

```bash
sudo dnf install git python3-devel libffi-devel gcc openssl-devel python3-libselinux
```

**对于 Debian 或 Ubuntu：**

```bash
sudo apt install git python3-dev libffi-dev gcc libssl-dev libdbus-glib-1-dev
```

### 安装虚拟环境依赖

**步骤 3: 安装虚拟环境依赖**

**对于 CentOS 或 Rocky：** 不需要做任何操作。

**对于 Debian 或 Ubuntu：**

```bash
sudo apt install python3-venv
```

**步骤 4: 创建虚拟环境并激活**

```bash
python3 -m venv /path/to/venv
source /path/to/venv/bin/activate
```

在运行依赖虚拟环境中安装包的任何命令之前，应先激活虚拟环境。

**步骤 5: 确保安装最新版本的 pip**

```bash
pip install -U pip
```

---

## 安装 Kolla-Ansible

### 步骤 1: 使用 pip 安装 kolla-ansible 及其依赖项

```bash
pip install git+https://opendev.org/openstack/kolla-ansible@|KOLLA_BRANCH_NAME|
```

### 步骤 2: 创建 `/etc/kolla` 目录

```bash
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
```

### 步骤 3: 复制配置文件

将 `globals.yml` 和 `passwords.yml` 复制到 `/etc/kolla` 目录：

```bash
cp -r /path/to/venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
```

### 步骤 4: 复制清单文件

将 `all-in-one` 清单文件复制到当前目录：

```bash
cp /path/to/venv/share/kolla-ansible/ansible/inventory/all-in-one .
```

---

## 安装 Ansible Galaxy 依赖

安装 Ansible Galaxy 依赖：

```bash
kolla-ansible install-deps
```

---

## 准备初始配置

### 清单（Inventory）

下一步是准备清单文件。清单是一个 Ansible 文件，我们在其中指定主机及其所属的组。我们可以使用它来定义节点角色和访问凭据。

Kolla Ansible 附带 `all-in-one` 和 `multinode` 示例清单文件。它们的区别在于，前者已准备好在 localhost 上部署单节点 OpenStack。在本指南中，我们将展示 `all-in-one` 安装。

### Kolla 密码

部署中使用的密码存储在 `/etc/kolla/passwords.yml` 文件中。此文件中的所有密码都是空白的，必须手动填写或通过运行随机密码生成器来填写：

```bash
kolla-genpwd
```

### Kolla globals.yml

`globals.yml` 是 Kolla Ansible 的主要配置文件，默认存储在 `/etc/kolla/globals.yml` 文件中。部署 Kolla Ansible 需要设置以下几个选项：

#### 1. 镜像选项

用户必须指定用于部署的镜像。在本指南中，我们将使用 [Quay.io](https://quay.io/organization/openstack.kolla) 提供的预构建镜像。要了解有关构建机制的更多信息，请参阅[构建容器镜像](https://docs.openstack.org/kolla/2025.1/admin/image-building.html)。

Kolla 提供了多种 Linux 发行版的容器镜像选择：

- CentOS Stream (`centos`)
- Debian (`debian`)
- Rocky (`rocky`)
- Ubuntu (`ubuntu`)

对于新手，我们推荐使用 Rocky Linux 9 或 Ubuntu 24.04。

```bash
kolla_base_distro: "rocky"
```

#### 2. AArch64 选项

Kolla 提供了 x86-64 和 aarch64 架构的镜像。它们不是"多架构"的，因此 aarch64 用户需要定义 "openstack_tag_suffix" 设置：

```bash
openstack_tag_suffix: "-aarch64"
```

这样将使用为 aarch64 架构构建的镜像。

#### 3. 网络配置

Kolla Ansible 需要设置一些网络选项。我们需要设置 OpenStack 使用的网络接口。

首先需要设置的接口是 "network_interface"。这是多种管理类型网络的默认接口。

```bash
network_interface: "eth0"
```

第二个需要的接口是专用于 Neutron 外部（或公共）网络的接口，可以是 vlan 或 flat，具体取决于网络的创建方式。该接口应该处于活动状态但没有 IP 地址。否则，实例将无法访问外部网络。

```bash
neutron_external_interface: "eth1"
```

要了解有关网络配置的更多信息，请参阅[网络概述](https://docs.openstack.org/kolla-ansible/2025.1/admin/production-architecture-guide.html#network-configuration)。

接下来，我们需要提供用于管理流量的浮动 IP。此 IP 将由 keepalived 管理以提供高可用性，应设置为连接到 `network_interface` 的管理网络中**未使用**的地址。如果您使用现有的 OpenStack 安装进行部署，请确保在 VM 的配置中允许该 IP。

```bash
kolla_internal_vip_address: "10.1.0.250"
```

#### 4. 启用附加服务

默认情况下，Kolla Ansible 提供了一个裸计算工具包，但它确实支持大量附加服务。要启用它们，请将 `enable_*` 设置为 "yes"。

Kolla 现在支持许多 OpenStack 服务，[可用服务列表](https://github.com/openstack/kolla-ansible/blob/master/README.rst#openstack-services)。有关服务配置的更多信息，请参阅[服务参考指南](https://docs.openstack.org/kolla-ansible/2025.1/reference/index.html)。

#### 5. 多个 globals 文件

为了更精细地控制，现在可以使用多个 yml 文件来启用主 `globals.yml` 文件中的任何选项。只需在 `/etc/kolla/` 下创建一个名为 `globals.d` 的目录，并将所有相关的 `*.yml` 文件放在那里。`kolla-ansible` 脚本会自动将所有这些文件作为参数添加到 `ansible-playbook` 命令中。

一个示例用例是：如果操作员想在初始部署之后的某个阶段启用 cinder 及其所有选项，而不篡改现有的 `globals.yml` 文件。这可以通过使用单独的 `cinder.yml` 文件实现，该文件放在 `/etc/kolla/globals.d/` 目录下，并在其中添加所有相关选项。

#### 6. 虚拟环境

建议使用虚拟环境在远程主机上执行任务。这部分内容在[虚拟环境](https://docs.openstack.org/kolla-ansible/2025.1/user/virtual-environments.html)中介绍。

---

## 部署

配置完成后，我们可以进入部署阶段。首先需要设置基本的主机级依赖项，如 docker。

Kolla Ansible 提供了一个 playbook，将安装所有正确版本的服务。

以下假设使用 `all-in-one` 清单。如果使用不同的清单（如 `multinode），请相应地替换 `-i` 参数。

### 步骤 1: 引导服务器安装 kolla 部署依赖项

```bash
kolla-ansible bootstrap-servers -i ./all-in-one
```

### 步骤 2: 对主机进行部署前检查

```bash
kolla-ansible prechecks -i ./all-in-one
```

### 步骤 3: 执行实际的 OpenStack 部署

```bash
kolla-ansible deploy -i ./all-in-one
```

当这个 playbook 完成后，OpenStack 应该已经启动、运行并且可以正常使用了！如果在执行过程中发生错误，请参阅[故障排除指南](https://docs.openstack.org/kolla-ansible/2025.1/user/troubleshooting.html)。

---

## 使用 OpenStack

### 步骤 1: 安装 OpenStack CLI 客户端

```bash
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/|KOLLA_OPENSTACK_RELEASE|
```

### 步骤 2: 生成 clouds.yaml 文件

OpenStack 需要一个 `clouds.yaml` 文件来设置 admin 用户的凭据。生成此文件：

```bash
kolla-ansible post-deploy
```

> **注意**: 该文件将生成在 `/etc/kolla/clouds.yaml` 中，您可以通过将其复制到 `/etc/openstack` 或 `~/.config/openstack`，或设置 `OS_CLIENT_CONFIG_FILE` 环境变量来使用它。

### 步骤 3: 运行初始化脚本

根据您安装 Kolla Ansible 的方式，有一个脚本可以创建示例网络、镜像等。

> **警告**: 您可以随意将以下 `init-runonce` 脚本用于演示目的，但请注意，为了使用您的云，**不**一定必须运行它。根据您的自定义设置，它可能无法工作，或可能与您要创建的资源冲突。已被警告。

```bash
/path/to/venv/share/kolla-ansible/init-runonce
```

---

## 相关文档

- [Kolla Ansible 部署指南](../kolla-ansible-2025.md)
- [多节点部署](./kolla-ansible-Multi.md)
- [故障排除指南](https://docs.openstack.org/kolla-ansible/2025.1/user/troubleshooting.html)
- [服务参考指南](https://docs.openstack.org/kolla-ansible/2025.1/reference/index.html)
