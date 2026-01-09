# Kolla-Ansible 网络配置指南

本指南将帮助您配置Kolla以满足生产环境需求。它旨在解答关于Kolla基本配置选项的一些问题本文档还包含其他有用的参考信息。

## 节点类型及其运行的服务

基本的Kolla清单由多种类型的节点组成，在Ansible中称为"组"：

- **控制节点** - 云控制节点，托管API和数据库等控制服务。该组应为奇数个节点以实现仲裁。
- **网络节点** - 托管Neutron代理以及haproxy/keepalived的节点。这些节点将具有在`kolla_internal_vip_address`中定义的浮动IP。
- **计算节点** - 用于计算服务的节点。客户虚拟机运行在此类节点上。
- **存储节点** - 用于cinder-volume和LVM的存储节点。
- **监控节点** - 托管监控服务的监控节点。

## 网络配置

### 接口配置

在Kolla中，操作员应配置以下网络接口：

- **`network_interface`** - 虽然它本身不被使用，但为以下其他接口提供所需的默认值。
- **`api_interface`** - 此接口用于管理网络。管理网络是OpenStack服务之间相互通信以及与数据库通信的网络。这里存在已知的安全风险，因此建议将此网络设为内部网络，不从外部访问。默认为`network_interface`。
- **`kolla_external_vip_interface`** - 这是面向公众的接口。当您希望HAProxy公共端点暴露在不同于内部端点的网络中时使用此接口。当`kolla_enable_tls_external`设置为yes时，必须设置此选项。默认为`network_interface`。
- **`tunnel_interface`** - Neutron用于通过隧道网络（如VxLan）进行虚拟机到虚拟机通信的接口。默认为`network_interface`。
- **`neutron_external_interface`** - Neutron需要的接口。Neutron将在其上放置br-ex。它将用于扁平网络以及标记的VLAN网络。必须单独设置。
- **`dns_interface`** - Designate和Bind9需要的接口。用于面向公众的DNS请求以及对bind9和designate mDNS服务的查询。默认为`network_interface`。
- **`bifrost_network_interface`** - Bifrost需要的接口。用于配置裸金属云主机，需要与裸金属云主机具有L2连接性，以便提供带PXE引导选项的DHCP租约。默认为`network_interface`。

### 地址族配置（IPv4/IPv6）

从Train版本开始，Kolla Ansible允许操作员使用IPv6而非IPv4部署控制平面。每个Kolla Ansible网络（由接口表示）提供两种地址族的选择。内部和外部VIP地址都可以使用IPv6地址进行配置。IPv6在所有支持的平台上都经过测试。

> **警告**
> 虽然Kolla Ansible Train需要Ansible 2.6或更高版本，但IPv6支持需要Ansible 2.8或更高版本，因为存在一个bug：https://github.com/ansible/ansible/issues/63227

> **注意**
> 目前不支持双栈。IPv4只能与IPv6在不同网络上混合使用。此约束源于服务需要通用单一地址族寻址。

例如，`network_address_family`接受`ipv4`或`ipv6`作为其值，并为所有网络定义默认地址族，类似于`network_interface`定义默认接口的方式。类似地，`api_address_family`更改API网络的地址族。当前网络列表可在`globals.yml`文件中找到。

> **注意**
> 虽然Train版本中引入的IPv6支持范围很广，但已知某些服务尚无法与IPv6一起使用或存在一些已知问题：
> - Bifrost不支持IPv6：https://storyboard.openstack.org/#!/story/2006689
> - Docker不允许IPv6注册表地址：https://github.com/moby/moby/issues/39033 - 解决方法是使用主机名
> - Ironic DHCP服务器dnsmasq目前无法自动配置为提供DHCPv6：https://bugs.launchpad.net/kolla-ansible/+bug/1848454
