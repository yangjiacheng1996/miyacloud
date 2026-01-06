# Kolla-Ansible 网络平面配置详解

## 概述

Kolla-Ansible 是 OpenStack 的容器化部署工具，它将 OpenStack 各个服务封装在 Docker 容器中，通过 Ansible 进行自动化部署。在部署过程中，Kolla-Ansible 会创建多个独立的网络平面（Network Plane），每个网络平面负责特定类型的流量，以实现网络隔离和安全管理。

本文档详细介绍 Kolla-Ansible 支持的网络平面类型、作用以及如何在 `globals.yml` 中进行配置。

## 网络平面类型

Kolla-Ansible 总共包含 **6 个主要网络平面**，每个平面都有其特定的用途和配置要求：

| 网络平面 | 默认值 | 用途 |
|---------|--------|------|
| `network_interface` | 无（必配） | 基础接口，为其他接口提供默认值 |
| `api_interface` | `network_interface` | 管理网络，用于 OpenStack 服务间通信 |
| `tunnel_interface` | `network_interface` | 隧道网络，用于 VXLAN/GRE 封装流量 |
| `neutron_external_interface` | 无（必配） | 外部网络，用于 Neutron 浮动 IP 和提供商网络 |
| `docker_interface` | `network_interface` | Docker 内部网络，用于容器间通信 |
| `storage_interface` | `network_interface` | 存储网络，用于 Ceph、iSCSI 等存储流量 |

### 1. network_interface（基础接口）

**用途**：作为其他网络接口的默认回退值，本身不直接使用。

**配置示例**：
```yaml
# /etc/kolla/globals.yml
network_interface: "eth0"
```

### 2. api_interface（API 管理网络）

**用途**：OpenStack 服务之间的内部通信网络，包括数据库连接、服务间 API 调用等。这是 OpenStack 控制平面的核心网络。

**配置示例**：
```yaml
api_interface: "{{ network_interface }}"
# 或自定义
api_interface: "eth1"
```

### 3. tunnel_interface（隧道网络）

**用途**：Neutron 组件用于处理虚拟机之间的隧道流量，支持 VXLAN 和 GRE 封装。在计算节点和网络节点上用于创建隧道端点。

**配置示例**：
```yaml
tunnel_interface: "{{ network_interface }}"
# 或自定义
tunnel_interface: "eth2"
```

### 4. neutron_external_interface（外部网络接口）

**用途**：Neutron 外部网络接口，用于：
- 浮动 IP（Floating IP）流量
- 提供商网络（Provider Networks）
- 外部网关流量

此接口上会创建 `br-ex` 网桥。

**配置示例**：
```yaml
neutron_external_interface: "eth3"
```

**重要**：此接口不能是已配置的 IP 接口，应该是物理网卡或未配置 IP 的接口。

### 5. docker_interface（Docker 网络接口）

**用途**：Docker 内部网络，用于 Kolla 容器之间的通信。通常配置在独立的网段上。

**配置示例**：
```yaml
docker_interface: "{{ network_interface }}"
# 或自定义（推荐使用独立接口）
docker_interface: "docker0"
```

### 6. storage_interface（存储网络接口）

**用途**：存储流量专用网络，用于：
- Ceph 集群通信
- iSCSI 流量
- NFS 存储流量
- Cinder 块存储流量

**配置示例**：
```yaml
storage_interface: "{{ network_interface }}"
# 或自定义
storage_interface: "eth4"
```

## globals.yml 完整配置示例

以下是生产环境推荐的 `globals.yml` 网络配置示例：

```yaml
# /etc/kolla/globals.yml

# ========== 网络接口配置 ==========

# 基础网络接口（所有其他接口的默认值）
network_interface: "eth0"

# API 管理网络（服务间通信）
api_interface: "{{ network_interface }}"
# api_interface: "eth1"  # 可选：使用独立接口

# 隧道网络（VXLAN/GRE 流量）
tunnel_interface: "{{ network_interface }}"
# tunnel_interface: "eth2"  # 可选：使用独立接口

# Neutron 外部网络接口（浮动 IP 和提供商网络）
neutron_external_interface: "eth3"

# Docker 内部网络
docker_interface: "{{ network_interface }}"
# docker_interface: "docker0"  # 默认使用 docker0 网桥

# 存储网络（存储流量）
storage_interface: "{{ network_interface }}"
# storage_interface: "eth4"  # 可选：使用独立接口

# ========== VIP 地址配置 ==========

# 内部 VIP 地址（用于 OpenStack 服务访问）
kolla_internal_vip_address: "192.168.1.250"

# 外部 VIP 地址（用于外部访问 Dashboard 等）
kolla_external_vip_address: "10.0.0.250"

# 外部网络接口（用于 VIP）
kolla_external_vip_interface: "{{ neutron_external_interface }}"
# kolla_external_vip_interface: "eth3"

# ========== 隧道网络配置 ==========

# 隧道类型（vxlan 或 gre）
neutron隧道类型: "vxlan"

# 隧道端口范围
neutron_tunneling_port_range: "8472"
```

## 独立网络平面配置示例

对于生产环境，建议使用独立网络接口以实现更好的隔离：

```yaml
# /etc/kolla/globals.yml

# 独立网络平面配置
network_interface: "eth0"           # 管理基础
api_interface: "eth1"               # API 管理网络 - 192.168.10.0/24
tunnel_interface: "eth2"            # 隧道网络 - 192.168.20.0/24
neutron_external_interface: "eth3"  # 外部网络 - 物理网络
docker_interface: "docker0"         # Docker 网络 - 172.17.0.0/16
storage_interface: "eth4"           # 存储网络 - 192.168.50.0/24

# VIP 配置
kolla_internal_vip_address: "192.168.10.250"
kolla_external_vip_address: "10.0.0.250"
kolla_external_vip_interface: "eth3"

# Neutron 配置
neutron_plugin_agent: "openvswitch"
neutron_external_interface: "eth3"
```

## 网络平面与 OpenStack 服务的关系

| 网络平面 | 相关 OpenStack 服务 | 流量类型 |
|---------|-------------------|---------|
| api_interface | Keystone, Nova, Neutron, Cinder | API 调用、数据库连接 |
| tunnel_interface | Neutron | VXLAN/GRE 隧道流量 |
| neutron_external_interface | Neutron | 浮动 IP、路由流量 |
| storage_interface | Cinder, Glance, Nova | 块存储、镜像存储 |
| docker_interface | 所有容器化服务 | 容器间控制流量 |

## 常见配置场景

### 场景 1：All-in-One 部署

```yaml
# 单节点部署简化配置
network_interface: "eth0"
neutron_external_interface: "eth0"  # 使用同一接口
kolla_internal_vip_address: "192.168.1.100"
```

### 场景 2：多节点生产部署

```yaml
# 生产环境完整配置
network_interface: "eno1"
api_interface: "eno2"
tunnel_interface: "eno3"
neutron_external_interface: "eno4"
docker_interface: "docker0"
storage_interface: "eno5"

kolla_internal_vip_address: "10.10.10.250"
kolla_external_vip_address: "203.0.113.250"
```

### 场景 3：IPv6 配置

```yaml
# IPv6 配置示例
network_interface: "eth0"
api_interface: "eth0"
tunnel_interface: "eth0"
neutron_external_interface: "eth1"

# IPv6 VIP 配置
kolla_internal_vip_address: "fd00::250"
kolla_external_vip_address: "2001:db8::250"
```

## 配置验证

部署前可以使用以下命令验证网络配置：

```bash
# 检查 kolla-ansible 的网络配置
kolla-ansible prechecks -e @/etc/kolla/globals.yml

# 查看网络配置清单
kolla-ansible inventory -e @/etc/kolla/globals.yml
```

## 注意事项

1. **接口顺序**：确保正确配置接口顺序，先配置 `network_interface`，再配置其他接口
2. **IP 配置**：`neutron_external_interface` 不应配置 IP 地址
3. **接口独立性**：生产环境建议每个网络平面使用独立物理接口
4. **网段规划**：提前规划好各个网络平面的网段，避免冲突
5. **VLAN 标记**：如果使用 VLAN，需要确保物理接口支持 VLAN 标记

## 相关文档

- [Kolla-Ansible 官方文档](https://docs.openstack.org/kolla-ansible/latest/)
- [OpenStack 网络指南](https://docs.openstack.org/neutron/latest/)
