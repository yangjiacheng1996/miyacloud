# Kolla-Ansible 多节点部署指南

> 基于 [Kolla-Ansible 2025.1 多节点部署文档](https://docs.openstack.org/project-deploy-guide/kolla-ansible/2025.1/multinode.html)

## 1. 部署Registry

### 1.1 为什么需要Registry

Docker Registry是一个本地托管的Registry，用于替代从公共Registry拉取镜像。对于多节点部署，建议使用本地Registry。

**特点：**
- 只需部署一个Registry实例
- Registry服务支持HA特性

### 1.2 部署简单Registry

在当前主机上部署简单的Registry：

```bash
docker run -d \
  --network host \
  --name registry \
  --restart=always \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:4000 \
  -v registry:/var/lib/registry \
  registry:2
```

**说明：**
- 使用端口4000避免与Keystone冲突
- 如果Registry不与Keystone在同一主机上，可以省略`-e`参数

### 1.3 配置Kolla使用Registry

编辑`globals.yml`文件，添加以下配置：

```yaml
docker_registry: 192.168.1.100:4000
docker_registry_insecure: yes
```

> 注意：`192.168.1.100:4000`是Registry监听的IP地址和端口

## 2. 编辑Inventory文件

### 2.1 Inventory文件位置

Ansible Inventory文件包含了确定哪些服务将部署到哪些主机上的所有信息。

- Kolla-Ansible目录：`ansible/inventory/multinode`
- 如果通过pip安装：位于`/usr/share/kolla-ansible`

### 2.2 配置主机组

将IP地址或主机名添加到相应的组，该组关联的服务将部署到该主机上。

**必须配置的主机组：**
- control
- network
- compute
- monitoring
- storage

### 2.3 配置SSH认证参数

```ini
[control]
# 这些主机名必须可从部署主机解析
control01      ansible_ssh_user=<ssh-username> ansible_become=True ansible_private_key_file=<path/to/private-key-file>
192.168.122.24 ansible_ssh_user=<ssh-username> ansible_become=True ansible_private_key_file=<path/to/private-key-file>
```

### 2.4 重要参数说明

| 参数 | 说明 |
|------|------|
| `ansible_ssh_user` | SSH用户名 |
| `ansible_become` | 是否提升权限（sudo） |
| `ansible_private_key_file` | 私钥文件路径 |
| `ansible_ssh_pass` | SSH密码（与私钥二选一） |

> ⚠️ Ansible使用SSH连接部署主机和目标主机

### 2.5 高级服务分组

对于更高级的角色，可以编辑每个组关联的服务：

```ini
[kibana:children]
control

[elasticsearch:children]
control

[loadbalancer:children]
network
```

> ⚠️ 注意：某些服务必须分组在一起，更改这些分组可能会破坏部署

## 3. 主机和组变量

### 3.1 变量配置方式

Kolla-Ansible配置通常存储在`globals.yml`文件中，该文件中的变量适用于所有主机。

### 3.2 在Inventory文件中直接定义

```ini
# 主机变量示例
[control]
control01 api_interface=eth3

# 组变量示例
[control:vars]
api_interface=eth4
```

### 3.3 使用host_vars和group_vars目录

推荐使用独立的YAML文件来管理变量：

```
inventory/
├── group_vars/
│   └── control
├── host_vars/
│   └── control01
└── multinode
```

### 3.4 变量优先级（重要）

根据Ansible变量优先级规则：

1. `globals.yml`中的变量具有最高优先级
2. `ansible/group_vars/all.yml`定义全局默认值
3. Inventory文件的`group_vars/all`优先级较低
4. Inventory的`group_vars/*`优先级更高

> ⚠️ 必须在不同主机之间有所不同的变量不能放在`globals.yml`中

## 4. 部署Kolla

### 4.1 部署前检查

```bash
kolla-ansible prechecks -i <path/to/multinode/inventory/file>
```

**重要提示：**
- RabbitMQ不支持IP地址，因此`api_interface`的IP地址应可通过主机名解析
- 确保所有RabbitMQ集群主机能事先解析彼此的主机名

### 4.2 执行部署

```bash
kolla-ansible deploy -i <path/to/multinode/inventory/file>
```

### 4.3 验证配置

```bash
kolla-ansible validate-config -i <path/to/multinode/inventory/file>
```

### 4.4 特殊注意事项

#### Keepalived虚拟路由器ID

如果在同一个第2层网络中运行多个keepalived集群，需要编辑`/etc/kolla/globals.yml`并指定`keepalvd_virtual_router_id`：

```yaml
keepalived_virtual_router_id: <unique-id>
```

- `keepalived_virtual_router_id`应该是唯一的
- 取值范围：0-255

#### Glance文件后端限制

如果Glance配置使用`file`作为后端，只会启动一个`glance_api`容器：

```yaml
# 当/etc/kolla/globals.yml中未指定其他后端时，默认启用file
```

### 4.5 配置验证说明

由于配置生成的性质，验证只能在首次部署后进行：

- 某些验证需要访问运行中的容器
- 验证任务位于各Ansible角色目录下：`kolla-ansible/ansible/roles/$role/tasks/config_validate.yml`
- 大多数OpenStack服务的验证通过特殊角色`service-config-validate`完成

## 5. 部署架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                    Kolla-Ansible 多节点架构                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   Control   │◄──►│   Network   │◄──►│   Compute   │     │
│  │   Nodes     │    │   Nodes     │    │   Nodes     │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│         │                  │                  │             │
│         └──────────────────┼──────────────────┘             │
│                            │                                │
│                    ┌───────▼───────┐                        │
│                    │   Registry   │                        │
│                    │   (可选)      │                        │
│                    └───────────────┘                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 6. 快速参考命令

```bash
# 1. 部署Registry
docker run -d --network host --name registry --restart=always \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:4000 -v registry:/var/lib/registry registry:2

# 2. 预检查
kolla-ansible prechecks -i /path/to/multinode

# 3. 部署
kolla-ansible deploy -i /path/to/multinode

# 4. 验证配置
kolla-ansible validate-config -i /path/to/multinode
```

## 相关文档

- [Kolla-Ansible 快速入门](./kolla-ansible-AIO.md)
- [OpenStack Kolla-Ansible 官方文档](https://docs.openstack.org/kolla-ansible/latest/)
- [Ansible Inventory 文档](https://docs.ansible.com/ansible/latest/intro_inventory.html)
- [Ansible 变量优先级](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#ansible-variable-precedence)

---

*文档最后更新：2025-12-19*
