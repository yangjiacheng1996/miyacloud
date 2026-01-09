以下是对手敲命令的一个记录。
用了一个虚拟机做实验，cpu是8 vcpu host-passthrough，虚拟机里可以嵌套虚拟化。
32GB Ram，系统盘200GB，数据盘1TB
# ip 修改
```
# 网卡名称还原
cat /etc/default/grub
>>>
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"

update-grub
reboot

# 主机名
vim /etc/hostname
kolla-aio

# 网络设备查看
# eth0是默认网口，eth1是虚拟机的外部网络网口，会和br-ex桥接，运行Flat扁平网络
root@kolla-aio:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:01:36:6a brd ff:ff:ff:ff:ff:ff
    altname enp1s0
    inet 10.0.1.145/24 brd 10.0.1.255 scope global dynamic eth0
       valid_lft 3474sec preferred_lft 3474sec
    inet6 fe80::5054:ff:fe01:366a/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 52:54:00:60:19:4b brd ff:ff:ff:ff:ff:ff
    altname enp9s0

# 修改ip
vim /etc/network/interface
>>>>>>>>>>>>>>>>>>
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug eth0
iface eth0 inet static
    address 10.0.1.10
    netmask 255.255.255.0
    gateway 10.0.1.1
    dns-nameservers 10.0.1.1

allow-hotplug eth1
iface eth1 inet manual

systemctl restart networking

# chrony时间同步
apt update && sudo apt install chrony -y
vim /etc/chrony/chrony.conf 
>>>>>>>>>>>>>>追加
server ntp1.aliyun.com iburst
server ntp2.aliyun.com iburst

systemctl daemon-reload
systemctl enable chrony
systemctl restart chrony
chronyc sources -v    

# 禁用其他时间同步服务，防止干扰
systemctl stop ntp
systemctl disable ntp
systemctl stop systemd-timesyncd
systemctl disable systemd-timesyncd

# 创建一个名叫cinder-volumes的VG卷，用于存放cinder的虚拟机块设备
apt install -y lvm2
pvcreate /dev/vdb
vgcreate cinder-volumes /dev/vdb

mkdir -p /etc/kolla
chown $USER:$USER /etc/kolla

# 配置清华pip源
mkdir ~/.pip
vim ~/.pip/pip.conf
>>>>>>>>>>>>>>>>>
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple

```

# 安装kolla-ansible
```
apt update
apt install -y git python3-dev libffi-dev gcc libssl-dev libdbus-glib-1-dev

apt install -y python3 python3-venv python3-pip
apt install -y ufw
cd /etc/kolla
python3 -m venv venv-kolla-ansible
source venv-kolla-ansible/bin/activate

pip install -U pip
pip install kolla-ansible==21.0.0

pip list|grep ansible
ansible-core       2.19.5
kolla-ansible      21.0.0
```
其他工具包也要安装
```
pip install dbus-python docker

```
# 安装ansible
因为已经安装了ansible-core，所以让pip根据ansible-core的版本自动安装合适的ansible版本。
```
pip install ansible

pip list|grep ansible
ansible            12.3.0
ansible-core       2.19.5
kolla-ansible      21.0.0

# 安装kolla-ansible依赖
kolla-ansible install-deps
```

# 准备配置文件
```
cp -r /etc/kolla/venv-kolla-ansible/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/inventory/all-in-one .

# 测试inventory文件是否可用
ansible all -i all-in-one  -m ping 
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
[WARNING]: Host 'localhost' is using the discovered Python interpreter at '/etc/kolla/venv-kolla-ansible/bin/python3.11', but future installation of another Python interpreter could cause a different interpreter to be discovered. See https://docs.ansible.com/ansible-core/2.19/reference_appendices/interpreter_discovery.html for more information.
localhost | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/etc/kolla/venv-kolla-ansible/bin/python3.11"
    },
    "changed": false,
    "ping": "pong"
}
```
# 生成密码文件
```
kolla-genpwd
```

# 编辑global.yml（至关重要）
```
vim /etc/kolla/globals.yml
>>>>>>>>>>>>>>>>
kolla_base_distro: "debian"
network_interface: "eth0"
neutron_external_interface: "eth1"
kolla_internal_vip_address: "10.0.1.250"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
# 如果你创建的VG名称不是 "cinder-volumes"，则需要显式指定
# cinder_volume_group: "my-volumes"

```

# 开始部署
bootstrap server,一些准备工作，包括安装docker，禁用防火墙，创建kolla用户，检查SSL证书。
```
kolla-ansible bootstrap-servers -i ./all-in-one

Bootstrapping servers
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details

PLAY [Gather facts for all hosts] ***************************************************************************************************************************************************************************

TASK [Group hosts to determine when using --limit] **********************************************************************************************************************************************************
ok: [localhost]

TASK [Gather facts] *****************************************************************************************************************************************************************************************
ok: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: all_using_limit_True

PLAY [Gather facts for all hosts (if using --limit)] ********************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role baremetal] *********************************************************************************************************************************************************************************

TASK [openstack.kolla.etc_hosts : Include etc-hosts.yml] ****************************************************************************************************************************************************
included: /root/.ansible/collections/ansible_collections/openstack/kolla/roles/etc_hosts/tasks/etc-hosts.yml for localhost

TASK [openstack.kolla.etc_hosts : Ensure localhost in /etc/hosts] *******************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.etc_hosts : Ensure hostname does not point to 127.0.1.1 in /etc/hosts] ****************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.etc_hosts : Generate /etc/hosts for all of the nodes] *********************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.etc_hosts : Check whether /etc/cloud/cloud.cfg exists] ********************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.etc_hosts : Disable cloud-init manage_etc_hosts] **************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.baremetal : Ensure unprivileged users can use ping] ***********************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.baremetal : Set firewall default policy] **********************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.baremetal : Check if firewalld is installed] ******************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.baremetal : Disable firewalld] ********************************************************************************************************************************************************
skipping: [localhost] => (item=firewalld) 
skipping: [localhost]

TASK [openstack.kolla.packages : Install packages] **********************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.packages : Remove packages] ***********************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : include_tasks] ***************************************************************************************************************************************************************
included: /root/.ansible/collections/ansible_collections/openstack/kolla/roles/docker/tasks/install.yml for localhost

TASK [openstack.kolla.docker : include_tasks] ***************************************************************************************************************************************************************
included: /root/.ansible/collections/ansible_collections/openstack/kolla/roles/docker/tasks/repo-Debian.yml for localhost

TASK [openstack.kolla.docker : Install CA certificates and gnupg packages] **********************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Ensure apt sources list directory exists] ************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Ensure apt keyrings directory exists] ****************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Install docker apt gpg key] **************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Install docker apt pin] ******************************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker : Ensure old docker repository absent] *****************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Enable docker apt repository] ************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Update the apt cache] ********************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Check which containers are running] ******************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Check if docker systemd unit exists] *****************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Mask the docker systemd unit on Debian/Ubuntu] *******************************************************************************************************************************
changed: [localhost]

TASK [openstack.kolla.docker : Install packages] ************************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Start docker] ****************************************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker : Wait for Docker to start] ****************************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker : Ensure containers are running after Docker upgrade] **************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker : Ensure docker config directory exists] ***************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Write docker config] *********************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : meta] ************************************************************************************************************************************************************************

TASK [openstack.kolla.docker : Get Docker API version] ******************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Parse Docker system info] ****************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Determine if Docker uses containerd image store] *****************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Copying over containerd config] **********************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Remove old docker options file] **********************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker : Ensure docker service directory exists] **************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker : Configure docker service] ****************************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker : Ensure the path for CA file for private registry exists] *********************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker : Ensure the CA file for private registry exists] ******************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker : Flush handlers] **************************************************************************************************************************************************************

TASK [openstack.kolla.docker : Start and enable docker] *****************************************************************************************************************************************************
changed: [localhost]

TASK [openstack.kolla.docker : include_tasks] ***************************************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.kolla_user : Ensure groups are present] ***********************************************************************************************************************************************
skipping: [localhost] => (item=docker) 
skipping: [localhost] => (item=sudo) 
skipping: [localhost] => (item=kolla) 
skipping: [localhost]

TASK [openstack.kolla.kolla_user : Create kolla user] *******************************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.kolla_user : Add public key to kolla user authorized keys] ****************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.kolla_user : Grant kolla user passwordless sudo] **************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker_sdk : Get Python] **************************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker_sdk : Check if Python environment is externally managed] ***********************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker_sdk : Set docker_sdk_python_externally_managed fact] ***************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker_sdk : include_tasks] ***********************************************************************************************************************************************************
included: /root/.ansible/collections/ansible_collections/openstack/kolla/roles/docker_sdk/tasks/install.yml for localhost

TASK [openstack.kolla.docker_sdk : Ensure apt sources list directory exists] ********************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker_sdk : Ensure apt keyrings directory exists] ************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker_sdk : Install osbpo apt gpg key] ***********************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker_sdk : Enable osbpo apt repository] *********************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker_sdk : Install packages] ********************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.docker_sdk : Check if virtualenv is a directory] **************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker_sdk : Check if packaging is already installed] *********************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker_sdk : Install packaging into virtualenv] ***************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker_sdk : Install latest pip and packaging in the virtualenv] **********************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.docker_sdk : Install docker SDK for python using pip] *********************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.baremetal : Ensure node_config_directory directory exists] ****************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.apparmor_libvirt : include_tasks] *****************************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.baremetal : Change state of selinux] **************************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.baremetal : Set https proxy for git] **************************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.baremetal : Set http proxy for git] ***************************************************************************************************************************************************
skipping: [localhost]

TASK [openstack.kolla.baremetal : Copying over kolla.target] ************************************************************************************************************************************************
ok: [localhost]

TASK [openstack.kolla.baremetal : Configure ceph for zun] ***************************************************************************************************************************************************
skipping: [localhost]

PLAY RECAP **************************************************************************************************************************************************************************************************
localhost                  : ok=42   changed=2    unreachable=0    failed=0    skipped=27   rescued=0    ignored=0   

```

预检查，检查服务器是否达到部署openstack的条件
```
kolla-ansible prechecks -i ./all-in-one

Pre-deployment checking
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details

PLAY [Gather facts for all hosts] ***************************************************************************************************************************************************************************

TASK [Group hosts to determine when using --limit] **********************************************************************************************************************************************************
ok: [localhost]

TASK [Gather facts] *****************************************************************************************************************************************************************************************
ok: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: all_using_limit_True

PLAY [Gather facts for all hosts (if using --limit)] ********************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Group hosts based on configuration] *******************************************************************************************************************************************************************

TASK [Group hosts based on Kolla action] ********************************************************************************************************************************************************************
ok: [localhost]

TASK [Group hosts based on enabled services] ****************************************************************************************************************************************************************
ok: [localhost] => (item=enable_aodh_False)
ok: [localhost] => (item=enable_barbican_False)
ok: [localhost] => (item=enable_blazar_False)
ok: [localhost] => (item=enable_ceilometer_False)
ok: [localhost] => (item=enable_ceph_rgw_False)
ok: [localhost] => (item=enable_cinder_True)
ok: [localhost] => (item=enable_cloudkitty_False)
ok: [localhost] => (item=enable_collectd_False)
ok: [localhost] => (item=enable_cyborg_False)
ok: [localhost] => (item=enable_designate_False)
ok: [localhost] => (item=enable_etcd_False)
ok: [localhost] => (item=enable_glance_True)
ok: [localhost] => (item=enable_gnocchi_False)
ok: [localhost] => (item=enable_grafana_False)
ok: [localhost] => (item=enable_hacluster_False)
ok: [localhost] => (item=enable_heat_True)
ok: [localhost] => (item=enable_horizon_True)
ok: [localhost] => (item=enable_influxdb_False)
ok: [localhost] => (item=enable_ironic_False)
ok: [localhost] => (item=enable_iscsid_True)
ok: [localhost] => (item=enable_keystone_True)
ok: [localhost] => (item=enable_kuryr_False)
ok: [localhost] => (item=enable_letsencrypt_False)
ok: [localhost] => (item=enable_loadbalancer_True)
ok: [localhost] => (item=enable_magnum_False)
ok: [localhost] => (item=enable_manila_False)
ok: [localhost] => (item=enable_mariadb_True)
ok: [localhost] => (item=enable_masakari_False)
ok: [localhost] => (item=enable_memcached_True)
ok: [localhost] => (item=enable_mistral_False)
ok: [localhost] => (item=enable_multipathd_False)
ok: [localhost] => (item=enable_neutron_True)
ok: [localhost] => (item=enable_nova_True)
ok: [localhost] => (item=enable_octavia_False)
ok: [localhost] => (item=enable_opensearch_False)
ok: [localhost] => (item=enable_opensearch_dashboards_False)
ok: [localhost] => (item=enable_openvswitch_True_enable_ovs_dpdk_False)
ok: [localhost] => (item=enable_ovn_False)
ok: [localhost] => (item=enable_placement_True)
ok: [localhost] => (item=enable_prometheus_False)
ok: [localhost] => (item=enable_rabbitmq_True)
ok: [localhost] => (item=enable_valkey_False)
ok: [localhost] => (item=enable_skyline_False)
ok: [localhost] => (item=enable_tacker_False)
ok: [localhost] => (item=enable_telegraf_False)
ok: [localhost] => (item=enable_trove_False)
ok: [localhost] => (item=enable_watcher_False)
ok: [localhost] => (item=enable_zun_False)

PLAY [Apply role prechecks] *********************************************************************************************************************************************************************************

TASK [prechecks : Checking loadbalancer group] **************************************************************************************************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [prechecks : include_tasks] ****************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/prechecks/tasks/host_os_checks.yml for localhost

TASK [prechecks : Checking host OS distribution] ************************************************************************************************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [prechecks : Checking host OS release or version] ******************************************************************************************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [prechecks : Checking if CentOS is Stream] *************************************************************************************************************************************************************
skipping: [localhost]

TASK [prechecks : Fail if not running on CentOS Stream] *****************************************************************************************************************************************************
skipping: [localhost]

TASK [prechecks : include_tasks] ****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [prechecks : Ensure /etc/localtime exist] **************************************************************************************************************************************************************
ok: [localhost]

TASK [prechecks : Fail if /etc/localtime is absent] *********************************************************************************************************************************************************
skipping: [localhost]

TASK [prechecks : Ensure /etc/timezone exist] ***************************************************************************************************************************************************************
ok: [localhost]

TASK [prechecks : Fail if /etc/timezone is absent] **********************************************************************************************************************************************************
skipping: [localhost]

TASK [prechecks : include_tasks] ****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [prechecks : Checking if system uses systemd] **********************************************************************************************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [prechecks : Checking Docker version] ******************************************************************************************************************************************************************
ok: [localhost]

TASK [prechecks : Checking empty passwords in passwords.yml. Run kolla-genpwd if this task fails] ***********************************************************************************************************
ok: [localhost]

TASK [prechecks : Check if nscd is running] *****************************************************************************************************************************************************************
ok: [localhost]

TASK [prechecks : Fail if nscd is running] ******************************************************************************************************************************************************************
skipping: [localhost]

TASK [prechecks : Validate that internal and external vip address are different when TLS is enabled only on either the internal and external network] *******************************************************
skipping: [localhost]

TASK [prechecks : Validate that enable_ceph is disabled] ****************************************************************************************************************************************************
[WARNING]: Deprecation warnings can be disabled by setting `deprecation_warnings=False` in ansible.cfg.
[DEPRECATION WARNING]: The `bool` filter coerced invalid value '' (str) to False. This feature will be removed from ansible-core version 2.23.
skipping: [localhost]

TASK [prechecks : Validate that enable_redis is disabled] ***************************************************************************************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [prechecks : Checking docker SDK version] **************************************************************************************************************************************************************
ok: [localhost]

TASK [prechecks : Checking dbus-python package] *************************************************************************************************************************************************************
ok: [localhost]

TASK [prechecks : Checking Ansible version] *****************************************************************************************************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

TASK [prechecks : Check if config_owner_user existed] *******************************************************************************************************************************************************
ok: [localhost]

TASK [prechecks : Check if config_owner_group existed] ******************************************************************************************************************************************************
ok: [localhost]

TASK [prechecks : Check if ansible user can do passwordless sudo] *******************************************************************************************************************************************
ok: [localhost]

TASK [prechecks : Check if external mariadb hosts are reachable from the load balancer] *********************************************************************************************************************
skipping: [localhost] => (item=localhost) 
skipping: [localhost]

TASK [prechecks : Check if external database address is reachable from all hosts] ***************************************************************************************************************************
skipping: [localhost]

PLAY [Apply role common] ************************************************************************************************************************************************************************************

TASK [common : include_tasks] *******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/common/tasks/precheck.yml for localhost

TASK [service-precheck : common | Validate inventory groups] ************************************************************************************************************************************************
skipping: [localhost] => (item=kolla-toolbox) 
skipping: [localhost]

PLAY [Apply role cron] **************************************************************************************************************************************************************************************

TASK [cron : include_tasks] *********************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/cron/tasks/precheck.yml for localhost

TASK [service-precheck : cron | Validate inventory groups] **************************************************************************************************************************************************
skipping: [localhost] => (item=cron) 
skipping: [localhost]

PLAY [Apply role fluentd] ***********************************************************************************************************************************************************************************

TASK [fluentd : include_tasks] ******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/fluentd/tasks/precheck.yml for localhost

TASK [service-precheck : fluentd | Validate inventory groups] ***********************************************************************************************************************************************
skipping: [localhost] => (item=fluentd) 
skipping: [localhost]

PLAY [Apply role loadbalancer] ******************************************************************************************************************************************************************************

TASK [loadbalancer : include_tasks] *************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/loadbalancer/tasks/precheck.yml for localhost

TASK [service-precheck : loadbalancer | Validate inventory groups] ******************************************************************************************************************************************
skipping: [localhost] => (item=haproxy) 
skipping: [localhost] => (item=proxysql) 
skipping: [localhost] => (item=keepalived) 
skipping: [localhost] => (item=haproxy-ssh) 
skipping: [localhost]

TASK [loadbalancer : Get container facts] *******************************************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Group hosts by whether they are running keepalived] ************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Group hosts by whether they are running HAProxy] ***************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Group hosts by whether they are running ProxySQL] **************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Set facts about whether we can run HAProxy and keepalived VIP prechecks] ***************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking if external haproxy certificate exists] ***************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Assert that external haproxy certificate exists] ***************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking if internal haproxy certificate exists] ***************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Assert that internal haproxy certificate exists] ***************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking the kolla_external_vip_interface is present] **********************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking the kolla_external_vip_interface is active] ***********************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking if kolla_internal_vip_address and kolla_external_vip_address are not pingable from any node] **********************************************************************************
ok: [localhost] => (item=10.0.1.250)
ok: [localhost] => (item=10.0.1.250)

TASK [loadbalancer : Checking free port for HAProxy stats] **************************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for HAProxy monitor (api interface)] ********************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for HAProxy monitor (vip interface)] ********************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for ProxySQL admin (api interface)] *********************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for ProxySQL admin (vip interface)] *********************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for ProxySQL prometheus exporter (api interface)] *******************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for ProxySQL prometheus exporter (vip interface)] *******************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking if kolla_internal_vip_address is in the same network as api_interface on all nodes] *******************************************************************************************
ok: [localhost]

TASK [loadbalancer : Getting haproxy stat] ******************************************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Setting haproxy stat fact] *************************************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Aodh API HAProxy] ***********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Barbican API HAProxy] *******************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Blazar API HAProxy] *********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Ceph RadosGW HAProxy] *******************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Cinder API HAProxy] *********************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Cloudkitty API HAProxy] *****************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Cyborg API HAProxy] *********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Designate API HAProxy] ******************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Glance API HAProxy] *********************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Gnocchi API HAProxy] ********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Grafana server HAProxy] *****************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Heat API HAProxy] ***********************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Heat API CFN HAProxy] *******************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Horizon HAProxy] ************************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Ironic API HAProxy] *********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Keystone Internal HAProxy] **************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Keystone Public HAProxy] ****************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Magnum API HAProxy] *********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Manila API HAProxy] *********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for MariaDB HAProxy/ProxySQL] ***************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Masakari API HAProxy] *******************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Mistral API HAProxy] ********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Neutron Server HAProxy] *****************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Nova API HAProxy] ***********************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Nova Metadata HAProxy] ******************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Nova NoVNC HAProxy] *********************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Nova Serial Proxy HAProxy] **************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Nova Spice HTML5 HAProxy] ***************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Nova Placement API HAProxy] *************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Octavia API HAProxy] ********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for OpenSearch HAProxy] *********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for OpenSearch Dashboards HAProxy] **********************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for RabbitMQ Management HAProxy] ************************************************************************************************************************************
ok: [localhost]

TASK [loadbalancer : Checking free port for Tacker Server HAProxy] ******************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Trove API HAProxy] **********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Watcher API HAProxy] ********************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Checking free port for Zun API HAProxy] ************************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Check if firewalld is running] *********************************************************************************************************************************************************
skipping: [localhost]

TASK [loadbalancer : Fail if firewalld is not running] ******************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : aodh] **********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : barbican] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : blazar] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : ceph-rgw] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : cinder] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : cloudkitty] ****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : cyborg] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : designate] *****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : etcd] **********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : glance] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : gnocchi] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : grafana] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : heat] **********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : horizon] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : influxdb] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : ironic] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : keystone] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : letsencrypt] ***************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : magnum] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : manila] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : mariadb] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : masakari] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : memcached] *****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : mistral] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : neutron] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : placement] *****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : nova] **********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : nova-cell] *****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : octavia] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : opensearch] ****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : prometheus] ****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : rabbitmq] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : skyline] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : tacker] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : trove] *********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : watcher] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : zun] ***********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : loadbalancer] **************************************************************************************************************************************************************************
skipping: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: enable_opensearch_True

PLAY [Apply role opensearch] ********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_letsencrypt_True

PLAY [Apply role letsencrypt] *******************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_collectd_True

PLAY [Apply role collectd] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_influxdb_True

PLAY [Apply role influxdb] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_telegraf_True

PLAY [Apply role telegraf] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_valkey_True

PLAY [Apply role valkey] ************************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role mariadb] ***********************************************************************************************************************************************************************************

TASK [mariadb : Group MariaDB hosts based on shards] ********************************************************************************************************************************************************
ok: [localhost] => (item=localhost)

TASK [mariadb : include_tasks] ******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/mariadb/tasks/precheck.yml for localhost

TASK [service-precheck : mariadb | Validate inventory groups] ***********************************************************************************************************************************************
skipping: [localhost] => (item=mariadb) 
skipping: [localhost]

TASK [mariadb : Get container facts] ************************************************************************************************************************************************************************
ok: [localhost]

TASK [mariadb : Checking free port for MariaDB] *************************************************************************************************************************************************************
ok: [localhost]

TASK [mariadb : Checking free port for MariaDB WSREP] *******************************************************************************************************************************************************
ok: [localhost]

TASK [mariadb : Checking free port for MariaDB IST] *********************************************************************************************************************************************************
ok: [localhost]

TASK [mariadb : Checking free port for MariaDB SST] *********************************************************************************************************************************************************
ok: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: mariadb_restart

PLAY [Restart mariadb services] *****************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: mariadb_start

PLAY [Start mariadb services] *******************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: mariadb_bootstrap_restart

PLAY [Restart bootstrap mariadb service] ********************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply mariadb post-configuration] *********************************************************************************************************************************************************************

TASK [Include mariadb post-deploy.yml] **********************************************************************************************************************************************************************
skipping: [localhost]

TASK [Include mariadb post-upgrade.yml] *********************************************************************************************************************************************************************
skipping: [localhost]

PLAY [Apply role memcached] *********************************************************************************************************************************************************************************

TASK [memcached : include_tasks] ****************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/memcached/tasks/precheck.yml for localhost

TASK [service-precheck : memcached | Validate inventory groups] *********************************************************************************************************************************************
skipping: [localhost] => (item=memcached) 
skipping: [localhost]

TASK [memcached : Get container facts] **********************************************************************************************************************************************************************
ok: [localhost]

TASK [memcached : Checking free port for Memcached] *********************************************************************************************************************************************************
ok: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: enable_prometheus_True

PLAY [Apply role prometheus] ********************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role prometheus-node-exporters] *****************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role iscsi] *************************************************************************************************************************************************************************************

TASK [iscsi : include_tasks] ********************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/iscsi/tasks/precheck.yml for localhost

TASK [service-precheck : iscsi | Validate inventory groups] *************************************************************************************************************************************************
skipping: [localhost] => (item=iscsid) 
skipping: [localhost] => (item=tgtd) 
skipping: [localhost]

TASK [iscsi : Get container facts] **************************************************************************************************************************************************************************
ok: [localhost]

TASK [iscsi : Checking free port for iscsi] *****************************************************************************************************************************************************************
ok: [localhost]

TASK [iscsi : Check supported platforms for tgtd] ***********************************************************************************************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}
[WARNING]: Could not match supplied host pattern, ignoring: enable_multipathd_True

PLAY [Apply role multipathd] ********************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role rabbitmq] **********************************************************************************************************************************************************************************

TASK [rabbitmq : include_tasks] *****************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/rabbitmq/tasks/precheck.yml for localhost

TASK [service-precheck : rabbitmq | Validate inventory groups] **********************************************************************************************************************************************
skipping: [localhost] => (item=rabbitmq) 
skipping: [localhost]

TASK [rabbitmq : Get container facts] ***********************************************************************************************************************************************************************
ok: [localhost]

TASK [rabbitmq : Checking free port for RabbitMQ] ***********************************************************************************************************************************************************
ok: [localhost]

TASK [rabbitmq : Checking free port for RabbitMQ Management] ************************************************************************************************************************************************
ok: [localhost]

TASK [rabbitmq : Checking free port for RabbitMQ Cluster] ***************************************************************************************************************************************************
ok: [localhost]

TASK [rabbitmq : Checking free port for RabbitMQ EPMD] ******************************************************************************************************************************************************
ok: [localhost]

TASK [rabbitmq : Check if all rabbit hostnames are resolvable] **********************************************************************************************************************************************
ok: [localhost] => (item=localhost)

TASK [rabbitmq : Check if each rabbit hostname resolves uniquely to the proper IP address] ******************************************************************************************************************
skipping: [localhost] => (item=[{'changed': False, 'stdout': '10.0.1.10       STREAM kolla-aio\n10.0.1.10       DGRAM  \n10.0.1.10       RAW    ', 'stderr': '', 'rc': 0, 'cmd': ['getent', 'ahostsv4', 'kolla-aio'], 'start': '2026-01-09 14:51:52.377333', 'end': '2026-01-09 14:51:52.383762', 'delta': '0:00:00.006429', 'msg': '', 'invocation': {'module_args': {'_raw_params': 'getent ahostsv4 kolla-aio', '_uses_shell': False, 'expand_argument_vars': True, 'stdin_add_newline': True, 'strip_empty_ends': True, 'cmd': None, 'argv': None, 'chdir': None, 'executable': None, 'creates': None, 'removes': None, 'stdin': None}}, 'stderr_lines': [], 'failed': False, 'item': 'localhost', 'ansible_loop_var': 'item'}, '10.0.1.10       STREAM kolla-aio']) 
skipping: [localhost] => (item=[{'changed': False, 'stdout': '10.0.1.10       STREAM kolla-aio\n10.0.1.10       DGRAM  \n10.0.1.10       RAW    ', 'stderr': '', 'rc': 0, 'cmd': ['getent', 'ahostsv4', 'kolla-aio'], 'start': '2026-01-09 14:51:52.377333', 'end': '2026-01-09 14:51:52.383762', 'delta': '0:00:00.006429', 'msg': '', 'invocation': {'module_args': {'_raw_params': 'getent ahostsv4 kolla-aio', '_uses_shell': False, 'expand_argument_vars': True, 'stdin_add_newline': True, 'strip_empty_ends': True, 'cmd': None, 'argv': None, 'chdir': None, 'executable': None, 'creates': None, 'removes': None, 'stdin': None}}, 'stderr_lines': [], 'failed': False, 'item': 'localhost', 'ansible_loop_var': 'item'}, '10.0.1.10       DGRAM  ']) 
skipping: [localhost] => (item=[{'changed': False, 'stdout': '10.0.1.10       STREAM kolla-aio\n10.0.1.10       DGRAM  \n10.0.1.10       RAW    ', 'stderr': '', 'rc': 0, 'cmd': ['getent', 'ahostsv4', 'kolla-aio'], 'start': '2026-01-09 14:51:52.377333', 'end': '2026-01-09 14:51:52.383762', 'delta': '0:00:00.006429', 'msg': '', 'invocation': {'module_args': {'_raw_params': 'getent ahostsv4 kolla-aio', '_uses_shell': False, 'expand_argument_vars': True, 'stdin_add_newline': True, 'strip_empty_ends': True, 'cmd': None, 'argv': None, 'chdir': None, 'executable': None, 'creates': None, 'removes': None, 'stdin': None}}, 'stderr_lines': [], 'failed': False, 'item': 'localhost', 'ansible_loop_var': 'item'}, '10.0.1.10       RAW    ']) 
skipping: [localhost]

TASK [rabbitmq : Check if TLS certificate exists for RabbitMQ] **********************************************************************************************************************************************
skipping: [localhost]

TASK [rabbitmq : Check if TLS key exists for RabbitMQ] ******************************************************************************************************************************************************
skipping: [localhost]

TASK [rabbitmq : List RabbitMQ queues] **********************************************************************************************************************************************************************
skipping: [localhost]

TASK [rabbitmq : Check if RabbitMQ quorum queues need to be configured] *************************************************************************************************************************************
skipping: [localhost]

TASK [rabbitmq : Check if RabbitMQ quorum queues for transient queues need to be configured] ****************************************************************************************************************
skipping: [localhost]

TASK [rabbitmq : Check if RabbitMQ streams need to be configured] *******************************************************************************************************************************************
skipping: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: rabbitmq_restart

PLAY [Restart rabbitmq services] ****************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply rabbitmq post-configuration] ********************************************************************************************************************************************************************

TASK [Include rabbitmq post-deploy.yml] *********************************************************************************************************************************************************************
skipping: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: enable_etcd_True

PLAY [Apply role etcd] **************************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role keystone] **********************************************************************************************************************************************************************************

TASK [keystone : include_tasks] *****************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/keystone/tasks/precheck.yml for localhost

TASK [service-precheck : keystone | Validate inventory groups] **********************************************************************************************************************************************
skipping: [localhost] => (item=keystone) 
skipping: [localhost] => (item=keystone-fernet) 
skipping: [localhost] => (item=keystone-httpd) 
skipping: [localhost] => (item=keystone-ssh) 
skipping: [localhost]

TASK [keystone : Get container facts] ***********************************************************************************************************************************************************************
ok: [localhost]

TASK [keystone : Checking free port for Keystone Public] ****************************************************************************************************************************************************
ok: [localhost]

TASK [keystone : Checking free port for Keystone SSH] *******************************************************************************************************************************************************
ok: [localhost]

TASK [keystone : Checking fernet_token_expiry] **************************************************************************************************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}
[WARNING]: Could not match supplied host pattern, ignoring: enable_ceph_rgw_True

PLAY [Apply role ceph-rgw] **********************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role glance] ************************************************************************************************************************************************************************************

TASK [glance : include_tasks] *******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/glance/tasks/precheck.yml for localhost

TASK [service-precheck : glance | Validate inventory groups] ************************************************************************************************************************************************
skipping: [localhost] => (item=glance-api) 
skipping: [localhost] => (item=glance-tls-proxy) 
skipping: [localhost]

TASK [glance : Get container facts] *************************************************************************************************************************************************************************
ok: [localhost]

TASK [glance : Checking free port for Glance API] ***********************************************************************************************************************************************************
ok: [localhost]

TASK [glance : Check if S3 configurations are defined] ******************************************************************************************************************************************************
skipping: [localhost] => (item=glance_backend_s3_url) 
skipping: [localhost] => (item=glance_backend_s3_bucket) 
skipping: [localhost] => (item=glance_backend_s3_access_key) 
skipping: [localhost] => (item=glance_backend_s3_secret_key) 
skipping: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: enable_ironic_True

PLAY [Apply role ironic] ************************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role cinder] ************************************************************************************************************************************************************************************

TASK [cinder : include_tasks] *******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/cinder/tasks/precheck.yml for localhost

TASK [service-precheck : cinder | Validate inventory groups] ************************************************************************************************************************************************
skipping: [localhost] => (item=cinder-api) 
skipping: [localhost] => (item=cinder-scheduler) 
skipping: [localhost] => (item=cinder-volume) 
skipping: [localhost] => (item=cinder-backup) 
skipping: [localhost]

TASK [cinder : Get container facts] *************************************************************************************************************************************************************************
ok: [localhost]

TASK [cinder : Checking free port for Cinder API] ***********************************************************************************************************************************************************
ok: [localhost]

TASK [cinder : Checking at least one valid backend is enabled for Cinder] ***********************************************************************************************************************************
skipping: [localhost]

TASK [cinder : Checking LVM volume group exists for Cinder] *************************************************************************************************************************************************
ok: [localhost]

TASK [cinder : Checking for coordination backend if Ceph backend is enabled] ********************************************************************************************************************************
skipping: [localhost]

TASK [cinder : Check if S3 configurations are defined] ******************************************************************************************************************************************************
skipping: [localhost] => (item=cinder_backup_s3_url) 
skipping: [localhost] => (item=cinder_backup_s3_bucket) 
skipping: [localhost] => (item=cinder_backup_s3_access_key) 
skipping: [localhost] => (item=cinder_backup_s3_secret_key) 
skipping: [localhost]

TASK [cinder : Check if Lightbits configurations are defined] ***********************************************************************************************************************************************
skipping: [localhost] => (item=lightos_api_address) 
skipping: [localhost] => (item=lightos_jwt) 
skipping: [localhost]

TASK [cinder : Check if cinder_cluster_name is configured for HA configurations] ****************************************************************************************************************************
skipping: [localhost]

TASK [cinder : Check if cinder_cluster_name is configured and configuration is non-HA] **********************************************************************************************************************
ok: [localhost] => {
    "changed": false,
    "msg": "All assertions passed"
}

PLAY [Apply role placement] *********************************************************************************************************************************************************************************

TASK [placement : include_tasks] ****************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/placement/tasks/precheck.yml for localhost

TASK [service-precheck : placement | Validate inventory groups] *********************************************************************************************************************************************
skipping: [localhost] => (item=placement-api) 
skipping: [localhost]

TASK [placement : Get container facts] **********************************************************************************************************************************************************************
ok: [localhost]

TASK [placement : Checking free port for Placement API] *****************************************************************************************************************************************************
ok: [localhost]

PLAY [Apply role openvswitch] *******************************************************************************************************************************************************************************

TASK [openvswitch : include_tasks] **************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/openvswitch/tasks/precheck.yml for localhost

TASK [service-precheck : openvswitch | Validate inventory groups] *******************************************************************************************************************************************
skipping: [localhost] => (item=openvswitch-db-server) 
skipping: [localhost] => (item=openvswitch-vswitchd) 
skipping: [localhost]

TASK [openvswitch : Get container facts] ********************************************************************************************************************************************************************
ok: [localhost]

TASK [openvswitch : Checking free port for OVSDB] ***********************************************************************************************************************************************************
ok: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: enable_openvswitch_True_enable_ovs_dpdk_True

PLAY [Apply role ovs-dpdk] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_ovn_True

PLAY [Apply role ovn-controller] ****************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role ovn-db] ************************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Bootstrap nova API databases] *************************************************************************************************************************************************************************

TASK [Bootstrap deploy] *************************************************************************************************************************************************************************************
skipping: [localhost]

TASK [Bootstrap upgrade] ************************************************************************************************************************************************************************************
skipping: [localhost]

PLAY [Bootstrap nova cell databases] ************************************************************************************************************************************************************************

TASK [Bootstrap deploy] *************************************************************************************************************************************************************************************
skipping: [localhost]

TASK [Bootstrap upgrade] ************************************************************************************************************************************************************************************
skipping: [localhost]

PLAY [Apply role nova] **************************************************************************************************************************************************************************************

TASK [nova : include_tasks] *********************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/nova/tasks/precheck.yml for localhost

TASK [service-precheck : nova | Validate inventory groups] **************************************************************************************************************************************************
skipping: [localhost] => (item=nova-api) 
skipping: [localhost] => (item=nova-metadata) 
skipping: [localhost] => (item=nova-scheduler) 
skipping: [localhost] => (item=nova-super-conductor) 
skipping: [localhost]

TASK [nova : Get container facts] ***************************************************************************************************************************************************************************
ok: [localhost]

TASK [nova : Checking free port for Nova API] ***************************************************************************************************************************************************************
ok: [localhost]

TASK [nova : Checking free port for Nova Metadata] **********************************************************************************************************************************************************
ok: [localhost]

PLAY [Apply role nova-cell] *********************************************************************************************************************************************************************************

TASK [nova-cell : include_tasks] ****************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/nova-cell/tasks/precheck.yml for localhost

TASK [service-precheck : nova | Validate inventory groups] **************************************************************************************************************************************************
skipping: [localhost] => (item=nova-libvirt) 
skipping: [localhost] => (item=nova-ssh) 
skipping: [localhost] => (item=nova-novncproxy) 
skipping: [localhost] => (item=nova-spicehtml5proxy) 
skipping: [localhost] => (item=nova-serialproxy) 
skipping: [localhost] => (item=nova-conductor) 
skipping: [localhost] => (item=nova-compute) 
skipping: [localhost] => (item=nova-compute-ironic) 
skipping: [localhost]

TASK [nova-cell : Get container facts] **********************************************************************************************************************************************************************
ok: [localhost]

TASK [nova-cell : Checking available compute nodes in inventory] ********************************************************************************************************************************************
skipping: [localhost]

TASK [nova-cell : Checking free port for Nova NoVNC Proxy] **************************************************************************************************************************************************
ok: [localhost]

TASK [nova-cell : Checking free port for Nova Serial Proxy] *************************************************************************************************************************************************
skipping: [localhost]

TASK [nova-cell : Checking free port for Nova Spice HTML5 Proxy] ********************************************************************************************************************************************
skipping: [localhost]

TASK [nova-cell : Checking free port for Nova SSH (API interface)] ******************************************************************************************************************************************
ok: [localhost]

TASK [nova-cell : Checking free port for Nova SSH (migration interface)] ************************************************************************************************************************************
skipping: [localhost]

TASK [nova-cell : Checking free port for Nova Libvirt] ******************************************************************************************************************************************************
ok: [localhost]

TASK [nova-cell : Checking that host libvirt is not running] ************************************************************************************************************************************************
ok: [localhost]

TASK [nova-cell : Checking that nova_libvirt container is not running] **************************************************************************************************************************************
skipping: [localhost]

PLAY [Refresh nova scheduler cell cache] ********************************************************************************************************************************************************************

TASK [nova : Refresh cell cache in nova scheduler] **********************************************************************************************************************************************************
skipping: [localhost]

PLAY [Reload global Nova super conductor services] **********************************************************************************************************************************************************

TASK [nova : Reload nova super conductor services to remove RPC version pin] ********************************************************************************************************************************
skipping: [localhost]

PLAY [Reload Nova cell services] ****************************************************************************************************************************************************************************

TASK [nova-cell : Reload nova cell services to remove RPC version cap] **************************************************************************************************************************************
skipping: [localhost] => (item=nova-conductor) 
skipping: [localhost] => (item=nova-compute) 
skipping: [localhost] => (item=nova-compute-ironic) 
skipping: [localhost] => (item=nova-novncproxy) 
skipping: [localhost] => (item=nova-serialproxy) 
skipping: [localhost] => (item=nova-spicehtml5proxy) 
skipping: [localhost]

PLAY [Reload global Nova API services] **********************************************************************************************************************************************************************

TASK [nova : Reload nova API services to remove RPC version pin] ********************************************************************************************************************************************
skipping: [localhost] => (item=nova-scheduler) 
skipping: [localhost] => (item=nova-api) 
skipping: [localhost]

PLAY [Run Nova API online data migrations] ******************************************************************************************************************************************************************

TASK [nova : Run Nova API online database migrations] *******************************************************************************************************************************************************
skipping: [localhost]

PLAY [Run Nova cell online data migrations] *****************************************************************************************************************************************************************

TASK [nova-cell : Run Nova cell online database migrations] *************************************************************************************************************************************************
skipping: [localhost]

PLAY [Apply role neutron] ***********************************************************************************************************************************************************************************

TASK [neutron : include_tasks] ******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/neutron/tasks/precheck.yml for localhost

TASK [service-precheck : neutron | Validate inventory groups] ***********************************************************************************************************************************************
skipping: [localhost] => (item=neutron-server) 
skipping: [localhost] => (item=neutron-rpc-server) 
skipping: [localhost] => (item=neutron-periodic-worker) 
skipping: [localhost] => (item=neutron-ovn-maintenance-worker) 
skipping: [localhost] => (item=neutron-openvswitch-agent) 
skipping: [localhost] => (item=neutron-dhcp-agent) 
skipping: [localhost] => (item=neutron-l3-agent) 
skipping: [localhost] => (item=neutron-sriov-agent) 
skipping: [localhost] => (item=neutron-mlnx-agent) 
skipping: [localhost] => (item=neutron-eswitchd) 
skipping: [localhost] => (item=neutron-metadata-agent) 
skipping: [localhost] => (item=neutron-ovn-metadata-agent) 
skipping: [localhost] => (item=neutron-bgp-dragent) 
skipping: [localhost] => (item=neutron-infoblox-ipam-agent) 
skipping: [localhost] => (item=neutron-metering-agent) 
skipping: [localhost] => (item=ironic-neutron-agent) 
skipping: [localhost] => (item=neutron-ovn-agent) 
skipping: [localhost]

TASK [neutron : Get container facts] ************************************************************************************************************************************************************************
ok: [localhost]

TASK [neutron : Checking free port for Neutron Server] ******************************************************************************************************************************************************
ok: [localhost]

TASK [neutron : Checking number of network agents] **********************************************************************************************************************************************************
skipping: [localhost]

TASK [neutron : Checking tenant network types] **************************************************************************************************************************************************************
ok: [localhost] => (item=vxlan) => {
    "ansible_loop_var": "item",
    "changed": false,
    "item": "vxlan",
    "msg": "All assertions passed"
}

TASK [neutron : Checking whether Ironic enabled] ************************************************************************************************************************************************************
skipping: [localhost]

TASK [neutron : Checking if neutron's dns domain has proper value] ******************************************************************************************************************************************
skipping: [localhost]

TASK [neutron : Get container facts] ************************************************************************************************************************************************************************
ok: [localhost]

TASK [neutron : Get container volume facts] *****************************************************************************************************************************************************************
ok: [localhost]

TASK [neutron : Check for ML2/OVN presence] *****************************************************************************************************************************************************************
skipping: [localhost]

TASK [neutron : Check for ML2/OVS presence] *****************************************************************************************************************************************************************
skipping: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: enable_kuryr_True

PLAY [Apply role kuryr] *************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_hacluster_True

PLAY [Apply role hacluster] *********************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role heat] **************************************************************************************************************************************************************************************

TASK [heat : include_tasks] *********************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/heat/tasks/precheck.yml for localhost

TASK [service-precheck : heat | Validate inventory groups] **************************************************************************************************************************************************
skipping: [localhost] => (item=heat-api) 
skipping: [localhost] => (item=heat-api-cfn) 
skipping: [localhost] => (item=heat-engine) 
skipping: [localhost]

TASK [heat : Get container facts] ***************************************************************************************************************************************************************************
ok: [localhost]

TASK [heat : Checking free port for Heat API] ***************************************************************************************************************************************************************
ok: [localhost]

TASK [heat : Checking free port for Heat API CFN] ***********************************************************************************************************************************************************
ok: [localhost]

PLAY [Apply role horizon] ***********************************************************************************************************************************************************************************

TASK [horizon : include_tasks] ******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/horizon/tasks/precheck.yml for localhost

TASK [service-precheck : horizon | Validate inventory groups] ***********************************************************************************************************************************************
skipping: [localhost] => (item=horizon) 
skipping: [localhost]

TASK [horizon : Get container facts] ************************************************************************************************************************************************************************
ok: [localhost]

TASK [horizon : Checking free port for Horizon] *************************************************************************************************************************************************************
ok: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: enable_magnum_True

PLAY [Apply role magnum] ************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_mistral_True

PLAY [Apply role mistral] ***********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_manila_True

PLAY [Apply role manila] ************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_gnocchi_True

PLAY [Apply role gnocchi] ***********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_ceilometer_True

PLAY [Apply role ceilometer] ********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_aodh_True

PLAY [Apply role aodh] **************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_barbican_True

PLAY [Apply role barbican] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_cyborg_True

PLAY [Apply role cyborg] ************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_designate_True

PLAY [Apply role designate] *********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_trove_True

PLAY [Apply role trove] *************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_watcher_True

PLAY [Apply role watcher] ***********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_grafana_True

PLAY [Apply role grafana] ***********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_cloudkitty_True

PLAY [Apply role cloudkitty] ********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_tacker_True

PLAY [Apply role tacker] ************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_octavia_True

PLAY [Apply role octavia] ***********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_zun_True

PLAY [Apply role zun] ***************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_blazar_True

PLAY [Apply role blazar] ************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_masakari_True

PLAY [Apply role masakari] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_skyline_True

PLAY [Apply role skyline] ***********************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY RECAP **************************************************************************************************************************************************************************************************
localhost                  : ok=114  changed=0    unreachable=0    failed=0    skipped=138  rescued=0    ignored=0   

```

下载容器镜像
```
kolla-ansible pull -i ./all-in-one

Pulling Docker images
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details

PLAY [Gather facts for all hosts] ***************************************************************************************************************************************************************************

TASK [Group hosts to determine when using --limit] **********************************************************************************************************************************************************
ok: [localhost]

TASK [Gather facts] *****************************************************************************************************************************************************************************************
ok: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: all_using_limit_True

PLAY [Gather facts for all hosts (if using --limit)] ********************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Group hosts based on configuration] *******************************************************************************************************************************************************************

TASK [Group hosts based on Kolla action] ********************************************************************************************************************************************************************
ok: [localhost]

TASK [Group hosts based on enabled services] ****************************************************************************************************************************************************************
ok: [localhost] => (item=enable_aodh_False)
ok: [localhost] => (item=enable_barbican_False)
ok: [localhost] => (item=enable_blazar_False)
ok: [localhost] => (item=enable_ceilometer_False)
ok: [localhost] => (item=enable_ceph_rgw_False)
ok: [localhost] => (item=enable_cinder_True)
ok: [localhost] => (item=enable_cloudkitty_False)
ok: [localhost] => (item=enable_collectd_False)
ok: [localhost] => (item=enable_cyborg_False)
ok: [localhost] => (item=enable_designate_False)
ok: [localhost] => (item=enable_etcd_False)
ok: [localhost] => (item=enable_glance_True)
ok: [localhost] => (item=enable_gnocchi_False)
ok: [localhost] => (item=enable_grafana_False)
ok: [localhost] => (item=enable_hacluster_False)
ok: [localhost] => (item=enable_heat_True)
ok: [localhost] => (item=enable_horizon_True)
ok: [localhost] => (item=enable_influxdb_False)
ok: [localhost] => (item=enable_ironic_False)
ok: [localhost] => (item=enable_iscsid_True)
ok: [localhost] => (item=enable_keystone_True)
ok: [localhost] => (item=enable_kuryr_False)
ok: [localhost] => (item=enable_letsencrypt_False)
ok: [localhost] => (item=enable_loadbalancer_True)
ok: [localhost] => (item=enable_magnum_False)
ok: [localhost] => (item=enable_manila_False)
ok: [localhost] => (item=enable_mariadb_True)
ok: [localhost] => (item=enable_masakari_False)
ok: [localhost] => (item=enable_memcached_True)
ok: [localhost] => (item=enable_mistral_False)
ok: [localhost] => (item=enable_multipathd_False)
ok: [localhost] => (item=enable_neutron_True)
ok: [localhost] => (item=enable_nova_True)
ok: [localhost] => (item=enable_octavia_False)
ok: [localhost] => (item=enable_opensearch_False)
ok: [localhost] => (item=enable_opensearch_dashboards_False)
ok: [localhost] => (item=enable_openvswitch_True_enable_ovs_dpdk_False)
ok: [localhost] => (item=enable_ovn_False)
ok: [localhost] => (item=enable_placement_True)
ok: [localhost] => (item=enable_prometheus_False)
ok: [localhost] => (item=enable_rabbitmq_True)
ok: [localhost] => (item=enable_valkey_False)
ok: [localhost] => (item=enable_skyline_False)
ok: [localhost] => (item=enable_tacker_False)
ok: [localhost] => (item=enable_telegraf_False)
ok: [localhost] => (item=enable_trove_False)
ok: [localhost] => (item=enable_watcher_False)
ok: [localhost] => (item=enable_zun_False)
[WARNING]: Could not match supplied host pattern, ignoring: kolla_action_precheck

PLAY [Apply role prechecks] *********************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role common] ************************************************************************************************************************************************************************************

TASK [common : include_tasks] *******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/common/tasks/pull.yml for localhost

TASK [service-images-pull : common | Pull images] ***********************************************************************************************************************************************************
changed: [localhost] => (item=kolla-toolbox)

PLAY [Apply role cron] **************************************************************************************************************************************************************************************

TASK [cron : include_tasks] *********************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/cron/tasks/pull.yml for localhost

TASK [service-images-pull : cron | Pull images] *************************************************************************************************************************************************************
changed: [localhost] => (item=cron)

PLAY [Apply role fluentd] ***********************************************************************************************************************************************************************************

TASK [fluentd : include_tasks] ******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/fluentd/tasks/pull.yml for localhost

TASK [service-images-pull : fluentd | Pull images] **********************************************************************************************************************************************************
changed: [localhost] => (item=fluentd)

PLAY [Apply role loadbalancer] ******************************************************************************************************************************************************************************

TASK [loadbalancer : include_tasks] *************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/loadbalancer/tasks/pull.yml for localhost

TASK [service-images-pull : loadbalancer | Pull images] *****************************************************************************************************************************************************
changed: [localhost] => (item=haproxy)
changed: [localhost] => (item=proxysql)
changed: [localhost] => (item=keepalived)

TASK [include_role : aodh] **********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : barbican] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : blazar] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : ceph-rgw] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : cinder] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : cloudkitty] ****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : cyborg] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : designate] *****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : etcd] **********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : glance] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : gnocchi] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : grafana] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : heat] **********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : horizon] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : influxdb] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : ironic] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : keystone] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : letsencrypt] ***************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : magnum] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : manila] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : mariadb] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : masakari] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : memcached] *****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : mistral] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : neutron] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : placement] *****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : nova] **********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : nova-cell] *****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : octavia] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : opensearch] ****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : prometheus] ****************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : rabbitmq] ******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : skyline] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : tacker] ********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : trove] *********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : watcher] *******************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : zun] ***********************************************************************************************************************************************************************************
skipping: [localhost]

TASK [include_role : loadbalancer] **************************************************************************************************************************************************************************
skipping: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: enable_opensearch_True

PLAY [Apply role opensearch] ********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_letsencrypt_True

PLAY [Apply role letsencrypt] *******************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_collectd_True

PLAY [Apply role collectd] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_influxdb_True

PLAY [Apply role influxdb] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_telegraf_True

PLAY [Apply role telegraf] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_valkey_True

PLAY [Apply role valkey] ************************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role mariadb] ***********************************************************************************************************************************************************************************

TASK [mariadb : Group MariaDB hosts based on shards] ********************************************************************************************************************************************************
ok: [localhost] => (item=localhost)

TASK [mariadb : include_tasks] ******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/mariadb/tasks/pull.yml for localhost

TASK [service-images-pull : mariadb | Pull images] **********************************************************************************************************************************************************
changed: [localhost] => (item=mariadb)
[WARNING]: Could not match supplied host pattern, ignoring: mariadb_restart

PLAY [Restart mariadb services] *****************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: mariadb_start

PLAY [Start mariadb services] *******************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: mariadb_bootstrap_restart

PLAY [Restart bootstrap mariadb service] ********************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply mariadb post-configuration] *********************************************************************************************************************************************************************

TASK [Include mariadb post-deploy.yml] **********************************************************************************************************************************************************************
skipping: [localhost]

TASK [Include mariadb post-upgrade.yml] *********************************************************************************************************************************************************************
skipping: [localhost]

PLAY [Apply role memcached] *********************************************************************************************************************************************************************************

TASK [memcached : include_tasks] ****************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/memcached/tasks/pull.yml for localhost

TASK [service-images-pull : memcached | Pull images] ********************************************************************************************************************************************************
changed: [localhost] => (item=memcached)
[WARNING]: Could not match supplied host pattern, ignoring: enable_prometheus_True

PLAY [Apply role prometheus] ********************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role prometheus-node-exporters] *****************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role iscsi] *************************************************************************************************************************************************************************************

TASK [iscsi : include_tasks] ********************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/iscsi/tasks/pull.yml for localhost

TASK [service-images-pull : iscsi | Pull images] ************************************************************************************************************************************************************
changed: [localhost] => (item=iscsid)
changed: [localhost] => (item=tgtd)
[WARNING]: Could not match supplied host pattern, ignoring: enable_multipathd_True

PLAY [Apply role multipathd] ********************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role rabbitmq] **********************************************************************************************************************************************************************************

TASK [rabbitmq : include_tasks] *****************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/rabbitmq/tasks/pull.yml for localhost

TASK [service-images-pull : rabbitmq | Pull images] *********************************************************************************************************************************************************
changed: [localhost] => (item=rabbitmq)
[WARNING]: Could not match supplied host pattern, ignoring: rabbitmq_restart

PLAY [Restart rabbitmq services] ****************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply rabbitmq post-configuration] ********************************************************************************************************************************************************************

TASK [Include rabbitmq post-deploy.yml] *********************************************************************************************************************************************************************
skipping: [localhost]
[WARNING]: Could not match supplied host pattern, ignoring: enable_etcd_True

PLAY [Apply role etcd] **************************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role keystone] **********************************************************************************************************************************************************************************

TASK [keystone : include_tasks] *****************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/keystone/tasks/pull.yml for localhost

TASK [service-images-pull : keystone | Pull images] *********************************************************************************************************************************************************
changed: [localhost] => (item=keystone)
changed: [localhost] => (item=keystone-fernet)
changed: [localhost] => (item=keystone-ssh)
[WARNING]: Could not match supplied host pattern, ignoring: enable_ceph_rgw_True

PLAY [Apply role ceph-rgw] **********************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role glance] ************************************************************************************************************************************************************************************

TASK [glance : include_tasks] *******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/glance/tasks/pull.yml for localhost

TASK [service-images-pull : glance | Pull images] ***********************************************************************************************************************************************************
changed: [localhost] => (item=glance-api)
[WARNING]: Could not match supplied host pattern, ignoring: enable_ironic_True

PLAY [Apply role ironic] ************************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role cinder] ************************************************************************************************************************************************************************************

TASK [cinder : include_tasks] *******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/cinder/tasks/pull.yml for localhost

TASK [service-images-pull : cinder | Pull images] ***********************************************************************************************************************************************************
changed: [localhost] => (item=cinder-api)
changed: [localhost] => (item=cinder-scheduler)
changed: [localhost] => (item=cinder-volume)
changed: [localhost] => (item=cinder-backup)

PLAY [Apply role placement] *********************************************************************************************************************************************************************************

TASK [placement : include_tasks] ****************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/placement/tasks/pull.yml for localhost

TASK [service-images-pull : placement | Pull images] ********************************************************************************************************************************************************
changed: [localhost] => (item=placement-api)

PLAY [Apply role openvswitch] *******************************************************************************************************************************************************************************

TASK [openvswitch : include_tasks] **************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/openvswitch/tasks/pull.yml for localhost

TASK [service-images-pull : openvswitch | Pull images] ******************************************************************************************************************************************************
changed: [localhost] => (item=openvswitch-db-server)
changed: [localhost] => (item=openvswitch-vswitchd)
[WARNING]: Could not match supplied host pattern, ignoring: enable_openvswitch_True_enable_ovs_dpdk_True

PLAY [Apply role ovs-dpdk] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_ovn_True

PLAY [Apply role ovn-controller] ****************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role ovn-db] ************************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Bootstrap nova API databases] *************************************************************************************************************************************************************************

TASK [Bootstrap deploy] *************************************************************************************************************************************************************************************
skipping: [localhost]

TASK [Bootstrap upgrade] ************************************************************************************************************************************************************************************
skipping: [localhost]

PLAY [Bootstrap nova cell databases] ************************************************************************************************************************************************************************

TASK [Bootstrap deploy] *************************************************************************************************************************************************************************************
skipping: [localhost]

TASK [Bootstrap upgrade] ************************************************************************************************************************************************************************************
skipping: [localhost]

PLAY [Apply role nova] **************************************************************************************************************************************************************************************

TASK [nova : include_tasks] *********************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/nova/tasks/pull.yml for localhost

TASK [service-images-pull : nova | Pull images] *************************************************************************************************************************************************************
changed: [localhost] => (item=nova-api)
ok: [localhost] => (item=nova-metadata)
changed: [localhost] => (item=nova-scheduler)

PLAY [Apply role nova-cell] *********************************************************************************************************************************************************************************

TASK [nova-cell : include_tasks] ****************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/nova-cell/tasks/pull.yml for localhost

TASK [service-images-pull : nova_cell | Pull images] ********************************************************************************************************************************************************
changed: [localhost] => (item=nova-libvirt)
changed: [localhost] => (item=nova-ssh)
changed: [localhost] => (item=nova-novncproxy)
changed: [localhost] => (item=nova-conductor)
changed: [localhost] => (item=nova-compute)

PLAY [Refresh nova scheduler cell cache] ********************************************************************************************************************************************************************

TASK [nova : Refresh cell cache in nova scheduler] **********************************************************************************************************************************************************
skipping: [localhost]

PLAY [Reload global Nova super conductor services] **********************************************************************************************************************************************************

TASK [nova : Reload nova super conductor services to remove RPC version pin] ********************************************************************************************************************************
skipping: [localhost]

PLAY [Reload Nova cell services] ****************************************************************************************************************************************************************************

TASK [nova-cell : Reload nova cell services to remove RPC version cap] **************************************************************************************************************************************
skipping: [localhost] => (item=nova-conductor) 
skipping: [localhost] => (item=nova-compute) 
skipping: [localhost] => (item=nova-compute-ironic) 
skipping: [localhost] => (item=nova-novncproxy) 
skipping: [localhost] => (item=nova-serialproxy) 
skipping: [localhost] => (item=nova-spicehtml5proxy) 
skipping: [localhost]

PLAY [Reload global Nova API services] **********************************************************************************************************************************************************************

TASK [nova : Reload nova API services to remove RPC version pin] ********************************************************************************************************************************************
skipping: [localhost] => (item=nova-scheduler) 
skipping: [localhost] => (item=nova-api) 
skipping: [localhost]

PLAY [Run Nova API online data migrations] ******************************************************************************************************************************************************************

TASK [nova : Run Nova API online database migrations] *******************************************************************************************************************************************************
skipping: [localhost]

PLAY [Run Nova cell online data migrations] *****************************************************************************************************************************************************************

TASK [nova-cell : Run Nova cell online database migrations] *************************************************************************************************************************************************
skipping: [localhost]

PLAY [Apply role neutron] ***********************************************************************************************************************************************************************************

TASK [neutron : include_tasks] ******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/neutron/tasks/pull.yml for localhost

TASK [service-images-pull : neutron | Pull images] **********************************************************************************************************************************************************
changed: [localhost] => (item=neutron-server)
ok: [localhost] => (item=neutron-rpc-server)
ok: [localhost] => (item=neutron-periodic-worker)
changed: [localhost] => (item=neutron-openvswitch-agent)
changed: [localhost] => (item=neutron-dhcp-agent)
changed: [localhost] => (item=neutron-l3-agent)
changed: [localhost] => (item=neutron-metadata-agent)
[WARNING]: Could not match supplied host pattern, ignoring: enable_kuryr_True

PLAY [Apply role kuryr] *************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_hacluster_True

PLAY [Apply role hacluster] *********************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY [Apply role heat] **************************************************************************************************************************************************************************************

TASK [heat : include_tasks] *********************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/heat/tasks/pull.yml for localhost

TASK [service-images-pull : heat | Pull images] *************************************************************************************************************************************************************
changed: [localhost] => (item=heat-api)
changed: [localhost] => (item=heat-api-cfn)
changed: [localhost] => (item=heat-engine)

PLAY [Apply role horizon] ***********************************************************************************************************************************************************************************

TASK [horizon : include_tasks] ******************************************************************************************************************************************************************************
included: /etc/kolla/venv-kolla-ansible/share/kolla-ansible/ansible/roles/horizon/tasks/pull.yml for localhost

TASK [service-images-pull : horizon | Pull images] **********************************************************************************************************************************************************
changed: [localhost] => (item=horizon)
[WARNING]: Could not match supplied host pattern, ignoring: enable_magnum_True

PLAY [Apply role magnum] ************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_mistral_True

PLAY [Apply role mistral] ***********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_manila_True

PLAY [Apply role manila] ************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_gnocchi_True

PLAY [Apply role gnocchi] ***********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_ceilometer_True

PLAY [Apply role ceilometer] ********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_aodh_True

PLAY [Apply role aodh] **************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_barbican_True

PLAY [Apply role barbican] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_cyborg_True

PLAY [Apply role cyborg] ************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_designate_True

PLAY [Apply role designate] *********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_trove_True

PLAY [Apply role trove] *************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_watcher_True

PLAY [Apply role watcher] ***********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_grafana_True

PLAY [Apply role grafana] ***********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_cloudkitty_True

PLAY [Apply role cloudkitty] ********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_tacker_True

PLAY [Apply role tacker] ************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_octavia_True

PLAY [Apply role octavia] ***********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_zun_True

PLAY [Apply role zun] ***************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_blazar_True

PLAY [Apply role blazar] ************************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_masakari_True

PLAY [Apply role masakari] **********************************************************************************************************************************************************************************
skipping: no hosts matched
[WARNING]: Could not match supplied host pattern, ignoring: enable_skyline_True

PLAY [Apply role skyline] ***********************************************************************************************************************************************************************************
skipping: no hosts matched

PLAY RECAP **************************************************************************************************************************************************************************************************
localhost                  : ok=41   changed=18   unreachable=0    failed=0    skipped=51   rescued=0    ignored=0   

```
中国大陆无法直接访问dockerhub，我想知道，在我没有配置国内源的情况下，kolla-ansible是如何成功下载镜像的？daemon.json没有配置，globals.yml也没有国内源的信息。
```
docker images 
                                                                                                                                                                                         i Info →   U  In Use
IMAGE                                                                      ID             DISK USAGE   CONTENT SIZE   EXTRA
quay.io/openstack.kolla/cinder-api:2025.2-debian-bookworm                  4ba623737a67       2.15GB          494MB        
quay.io/openstack.kolla/cinder-backup:2025.2-debian-bookworm               d9565a83f424       2.16GB          496MB        
quay.io/openstack.kolla/cinder-scheduler:2025.2-debian-bookworm            69b8f8977880       2.15GB          494MB        
quay.io/openstack.kolla/cinder-volume:2025.2-debian-bookworm               d03796e87ef8       2.18GB          499MB        
quay.io/openstack.kolla/cron:2025.2-debian-bookworm                        03f1bca3a9a6        526MB          136MB        
quay.io/openstack.kolla/fluentd:2025.2-debian-bookworm                     470c86d0e2ed       1.01GB          248MB        
quay.io/openstack.kolla/glance-api:2025.2-debian-bookworm                  0a455b36e988        1.8GB          434MB        
quay.io/openstack.kolla/haproxy:2025.2-debian-bookworm                     d7927de28988        541MB          139MB        
quay.io/openstack.kolla/heat-api-cfn:2025.2-debian-bookworm                7d0aee826469       1.76GB          410MB        
quay.io/openstack.kolla/heat-api:2025.2-debian-bookworm                    70884abbbb4d       1.76GB          410MB        
quay.io/openstack.kolla/heat-engine:2025.2-debian-bookworm                 013964e54808       1.76GB          410MB        
quay.io/openstack.kolla/horizon:2025.2-debian-bookworm                     9b3bdd5c3b26       2.05GB          457MB        
quay.io/openstack.kolla/iscsid:2025.2-debian-bookworm                      0c8b46d4c21a        542MB          140MB        
quay.io/openstack.kolla/keepalived:2025.2-debian-bookworm                  47db6b4426f5        541MB          140MB        
quay.io/openstack.kolla/keystone-fernet:2025.2-debian-bookworm             16b932d5d6c7       1.72GB          408MB        
quay.io/openstack.kolla/keystone-ssh:2025.2-debian-bookworm                9de28186fa9b       1.73GB          409MB        
quay.io/openstack.kolla/keystone:2025.2-debian-bookworm                    0717fbfd5443       1.76GB          421MB        
quay.io/openstack.kolla/kolla-toolbox:2025.2-debian-bookworm               43ce307afcbb       1.57GB          403MB        
quay.io/openstack.kolla/mariadb-server:2025.2-debian-bookworm              b2d9ea56a0d6       1.03GB          216MB        
quay.io/openstack.kolla/memcached:2025.2-debian-bookworm                   e207bcc33239        527MB          136MB        
quay.io/openstack.kolla/neutron-dhcp-agent:2025.2-debian-bookworm          a2ec4692b62f       2.01GB          468MB        
quay.io/openstack.kolla/neutron-l3-agent:2025.2-debian-bookworm            d0bdee91be5e       2.02GB          469MB        
quay.io/openstack.kolla/neutron-metadata-agent:2025.2-debian-bookworm      4c8ed8ed8239       2.01GB          468MB        
quay.io/openstack.kolla/neutron-openvswitch-agent:2025.2-debian-bookworm   f89667f6df66       2.01GB          468MB        
quay.io/openstack.kolla/neutron-server:2025.2-debian-bookworm              3556d5ff707f       2.03GB          470MB        
quay.io/openstack.kolla/nova-api:2025.2-debian-bookworm                    6cd3e0ff597a          2GB          467MB        
quay.io/openstack.kolla/nova-compute:2025.2-debian-bookworm                75bdc02a39a5       2.48GB          583MB        
quay.io/openstack.kolla/nova-conductor:2025.2-debian-bookworm              36462f7f9afb          2GB          467MB        
quay.io/openstack.kolla/nova-libvirt:2025.2-debian-bookworm                a797f5625beb       1.56GB          387MB        
quay.io/openstack.kolla/nova-novncproxy:2025.2-debian-bookworm             3d6de52bf499       2.17GB          505MB        
quay.io/openstack.kolla/nova-scheduler:2025.2-debian-bookworm              bd6c2b431411          2GB          467MB        
quay.io/openstack.kolla/nova-ssh:2025.2-debian-bookworm                    dfa436a3966d       2.01GB          469MB        
quay.io/openstack.kolla/openvswitch-db-server:2025.2-debian-bookworm       07ba7685a927        568MB          148MB        
quay.io/openstack.kolla/openvswitch-vswitchd:2025.2-debian-bookworm        714b60bba1e7        568MB          148MB        
quay.io/openstack.kolla/placement-api:2025.2-debian-bookworm               713c3c43a0ad       1.63GB          391MB        
quay.io/openstack.kolla/proxysql:2025.2-debian-bookworm                    e8134c1a96db        727MB          191MB        
quay.io/openstack.kolla/rabbitmq:2025.2-debian-bookworm                    c18bfd80c7db        656MB          186MB        
quay.io/openstack.kolla/tgtd:2025.2-debian-bookworm                        52c00688044a        524MB          136MB        

```
quay.io是红帽企业级镜像仓库源，所以国内可以直接下载。

部署容器，启动OpenStack核心组件和internal VIP
```
kolla-ansible deploy -i ./all-in-one

PLAY RECAP **************************************************************************************************************************************************************************************************
localhost                  : ok=459  changed=299  unreachable=0    failed=0    skipped=293  rescued=0    ignored=1   
```
然后浏览器登录vip地址，我的是10.0.1.250。


# 部署后操作
安装OpenStack Cli客户端
```
pip install python-openstackclient 
```
生成/etc/kolla/clouds.yaml文件作为admin用户的凭证。
```
kolla-ansible post-deploy -i /etc/kolla/all-in-one

Post-Deploying Playbooks
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details

PLAY [Determining whether we need become=true] **************************************************************************************************************************************************************

TASK [Get stats of /etc/kolla] ******************************************************************************************************************************************************************************
ok: [localhost]

TASK [Set become fact] **************************************************************************************************************************************************************************************
ok: [localhost]

PLAY [Creating clouds.yaml file on the deploy node] *********************************************************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Template out clouds.yaml] *****************************************************************************************************************************************************************************
changed: [localhost]

PLAY [Creating admin openrc file on the deploy node] ********************************************************************************************************************************************************

TASK [Gathering Facts] **************************************************************************************************************************************************************************************
ok: [localhost]

TASK [Template out admin-openrc.sh] *************************************************************************************************************************************************************************
changed: [localhost]

TASK [Template out admin-openrc-system.sh] ******************************************************************************************************************************************************************
changed: [localhost]

TASK [Template out public-openrc.sh] ************************************************************************************************************************************************************************
changed: [localhost]

TASK [Template out public-openrc-system.sh] *****************************************************************************************************************************************************************
changed: [localhost]

TASK [octavia : Template out octavia-openrc.sh] *************************************************************************************************************************************************************
skipping: [localhost]

PLAY RECAP **************************************************************************************************************************************************************************************************
localhost                  : ok=9    changed=5    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0  
```
获取admin用户的密码
```
cat /etc/kolla/clouds.yaml |grep password
```
初始化OpenStack环境。因为kolla刚部署的环境里面什么都没有，需要初始化来创建一个基础设置，管理员少折腾一点。以下步骤是有必要执行的。会启动一个cirros虚拟机做测试。
```
/etc/kolla/venv-kolla-ansible/share/kolla-ansible/init-runonce


Checking for locally available cirros image.
None found, downloading cirros image (version 0.6.3).
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 20.6M  100 20.6M    0     0  44837      0  0:08:03  0:08:03 --:--:-- 50584
Creating glance image.
+------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                                                                                                  |
+------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| checksum         | 87617e24a5e30cb3b87fda8c0764838f                                                                                                                                                       |
| container_format | bare                                                                                                                                                                                   |
| created_at       | 2026-01-09T08:35:52Z                                                                                                                                                                   |
| disk_format      | qcow2                                                                                                                                                                                  |
| file             | /v2/images/d3fde45e-332e-4ae4-a28c-5df675237313/file                                                                                                                                   |
| id               | d3fde45e-332e-4ae4-a28c-5df675237313                                                                                                                                                   |
| min_disk         | 0                                                                                                                                                                                      |
| min_ram          | 0                                                                                                                                                                                      |
| name             | cirros                                                                                                                                                                                 |
| owner            | 784266e338b34285b046b52060057d89                                                                                                                                                       |
| properties       | os_hash_algo='sha512', os_hash_value='9a9bce0083a00939ec17c11febbfc767aa211aaa54f51e75c5a8b271a9b5637c77205a518b7a2007cb391d23cceb01e0e4e8d64832317151bc85b734b92a7be0',               |
|                  | os_hidden='False', os_type='linux', owner_specified.openstack.md5='', owner_specified.openstack.object='images/cirros', owner_specified.openstack.sha256='', stores='file'             |
| protected        | False                                                                                                                                                                                  |
| schema           | /v2/schemas/image                                                                                                                                                                      |
| size             | 21692416                                                                                                                                                                               |
| status           | active                                                                                                                                                                                 |
| tags             |                                                                                                                                                                                        |
| updated_at       | 2026-01-09T08:35:53Z                                                                                                                                                                   |
| virtual_size     | 117440512                                                                                                                                                                              |
| visibility       | public                                                                                                                                                                                 |
+------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
Configuring neutron.
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| created_at              | 2026-01-09T08:35:56Z                 |
| description             |                                      |
| distributed             | False                                |
| enable_ndp_proxy        | None                                 |
| external_gateway_info   | null                                 |
| flavor_id               | None                                 |
| ha                      | False                                |
| id                      | 29567511-7a3a-400e-80ce-bebccad0f548 |
| name                    | demo-router                          |
| project_id              | 784266e338b34285b046b52060057d89     |
| revision_number         | 1                                    |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tags                    |                                      |
| updated_at              | 2026-01-09T08:35:56Z                 |
+-------------------------+--------------------------------------+
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2026-01-09T08:35:59Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 1f33dd22-2dbd-422b-a7fd-702ca1b32146 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_qinq              | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1450                                 |
| name                      | demo-net                             |
| port_security_enabled     | True                                 |
| project_id                | 784266e338b34285b046b52060057d89     |
| provider:network_type     | vxlan                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | 402                                  |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2026-01-09T08:35:59Z                 |
+---------------------------+--------------------------------------+
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 10.0.0.2-10.0.0.254                  |
| cidr                 | 10.0.0.0/24                          |
| created_at           | 2026-01-09T08:36:02Z                 |
| description          |                                      |
| dns_nameservers      | 8.8.8.8                              |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | True                                 |
| gateway_ip           | 10.0.0.1                             |
| host_routes          |                                      |
| id                   | c0afdefb-54c4-499b-ae01-944e36fb6af7 |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | demo-subnet                          |
| network_id           | 1f33dd22-2dbd-422b-a7fd-702ca1b32146 |
| project_id           | 784266e338b34285b046b52060057d89     |
| revision_number      | 0                                    |
| router:external      | False                                |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2026-01-09T08:36:02Z                 |
+----------------------+--------------------------------------+
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2026-01-09T08:36:09Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 38fcd1b9-9756-48fc-bbdb-94240a374e92 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_qinq              | None                                 |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | public1                              |
| port_security_enabled     | True                                 |
| project_id                | 784266e338b34285b046b52060057d89     |
| provider:network_type     | flat                                 |
| provider:physical_network | physnet1                             |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | External                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2026-01-09T08:36:09Z                 |
+---------------------------+--------------------------------------+
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 10.0.2.150-10.0.2.199                |
| cidr                 | 10.0.2.0/24                          |
| created_at           | 2026-01-09T08:36:11Z                 |
| description          |                                      |
| dns_nameservers      |                                      |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | False                                |
| gateway_ip           | 10.0.2.1                             |
| host_routes          |                                      |
| id                   | b3785d87-0802-48e0-ae9e-17bd27814f1c |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | public1-subnet                       |
| network_id           | 38fcd1b9-9756-48fc-bbdb-94240a374e92 |
| project_id           | 784266e338b34285b046b52060057d89     |
| revision_number      | 0                                    |
| router:external      | True                                 |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2026-01-09T08:36:11Z                 |
+----------------------+--------------------------------------+
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| belongs_to_default_sg   | True                                 |
| created_at              | 2026-01-09T08:36:21Z                 |
| description             |                                      |
| direction               | ingress                              |
| ether_type              | IPv4                                 |
| id                      | e9311784-022e-4354-8e55-f1a35e6c3467 |
| normalized_cidr         | 0.0.0.0/0                            |
| port_range_max          | None                                 |
| port_range_min          | None                                 |
| project_id              | 784266e338b34285b046b52060057d89     |
| protocol                | icmp                                 |
| remote_address_group_id | None                                 |
| remote_group_id         | None                                 |
| remote_ip_prefix        | 0.0.0.0/0                            |
| revision_number         | 0                                    |
| security_group_id       | 97ef4274-ab21-40c8-ade7-cabb75d3829e |
| updated_at              | 2026-01-09T08:36:21Z                 |
+-------------------------+--------------------------------------+
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| belongs_to_default_sg   | True                                 |
| created_at              | 2026-01-09T08:36:23Z                 |
| description             |                                      |
| direction               | ingress                              |
| ether_type              | IPv4                                 |
| id                      | 87a8bee7-79aa-457c-9836-9336d3d9f51c |
| normalized_cidr         | 0.0.0.0/0                            |
| port_range_max          | 22                                   |
| port_range_min          | 22                                   |
| project_id              | 784266e338b34285b046b52060057d89     |
| protocol                | tcp                                  |
| remote_address_group_id | None                                 |
| remote_group_id         | None                                 |
| remote_ip_prefix        | 0.0.0.0/0                            |
| revision_number         | 0                                    |
| security_group_id       | 97ef4274-ab21-40c8-ade7-cabb75d3829e |
| updated_at              | 2026-01-09T08:36:23Z                 |
+-------------------------+--------------------------------------+
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| belongs_to_default_sg   | True                                 |
| created_at              | 2026-01-09T08:36:25Z                 |
| description             |                                      |
| direction               | ingress                              |
| ether_type              | IPv4                                 |
| id                      | 99c0a883-1f79-4875-b43a-940e8c283e01 |
| normalized_cidr         | 0.0.0.0/0                            |
| port_range_max          | 8000                                 |
| port_range_min          | 8000                                 |
| project_id              | 784266e338b34285b046b52060057d89     |
| protocol                | tcp                                  |
| remote_address_group_id | None                                 |
| remote_group_id         | None                                 |
| remote_ip_prefix        | 0.0.0.0/0                            |
| revision_number         | 0                                    |
| security_group_id       | 97ef4274-ab21-40c8-ade7-cabb75d3829e |
| updated_at              | 2026-01-09T08:36:25Z                 |
+-------------------------+--------------------------------------+
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| belongs_to_default_sg   | True                                 |
| created_at              | 2026-01-09T08:36:27Z                 |
| description             |                                      |
| direction               | ingress                              |
| ether_type              | IPv4                                 |
| id                      | 767d1e07-a94a-49a2-8338-406f6f636d02 |
| normalized_cidr         | 0.0.0.0/0                            |
| port_range_max          | 8080                                 |
| port_range_min          | 8080                                 |
| project_id              | 784266e338b34285b046b52060057d89     |
| protocol                | tcp                                  |
| remote_address_group_id | None                                 |
| remote_group_id         | None                                 |
| remote_ip_prefix        | 0.0.0.0/0                            |
| revision_number         | 0                                    |
| security_group_id       | 97ef4274-ab21-40c8-ade7-cabb75d3829e |
| updated_at              | 2026-01-09T08:36:27Z                 |
+-------------------------+--------------------------------------+
Generating ssh key.
Generating public/private ecdsa key pair.
Your identification has been saved in /root/.ssh/id_ecdsa
Your public key has been saved in /root/.ssh/id_ecdsa.pub
The key fingerprint is:
SHA256:+BVKQ0VB9CIAc0ETrVJigeciGjM0QWBQNGdFHUn1Yp4 root@kolla-aio
The key's randomart image is:
+---[ECDSA 256]---+
|**=.**X*+OB.     |
|.o.++o.++  o     |
|. .+ o .+ = o    |
|= . o .o * =     |
|.= . .. S E      |
|.      . .       |
|        .        |
|                 |
|                 |
+----[SHA256]-----+
Configuring nova public key and quotas.
+-------------+-------------------------------------------------+
| Field       | Value                                           |
+-------------+-------------------------------------------------+
| created_at  | None                                            |
| fingerprint | 7f:74:30:41:f9:25:12:24:60:02:c0:b1:18:9a:25:40 |
| id          | mykey                                           |
| is_deleted  | None                                            |
| name        | mykey                                           |
| type        | ssh                                             |
| user_id     | 1b6703611c624839aea494f67a8c9742                |
+-------------+-------------------------------------------------+
+----------------------------+---------+
| Field                      | Value   |
+----------------------------+---------+
| OS-FLV-DISABLED:disabled   | False   |
| OS-FLV-EXT-DATA:ephemeral  | 0       |
| description                | None    |
| disk                       | 1       |
| id                         | 1       |
| name                       | m1.tiny |
| os-flavor-access:is_public | True    |
| properties                 |         |
| ram                        | 512     |
| rxtx_factor                | 1.0     |
| swap                       | 0       |
| vcpus                      | 1       |
+----------------------------+---------+
+----------------------------+----------+
| Field                      | Value    |
+----------------------------+----------+
| OS-FLV-DISABLED:disabled   | False    |
| OS-FLV-EXT-DATA:ephemeral  | 0        |
| description                | None     |
| disk                       | 20       |
| id                         | 2        |
| name                       | m1.small |
| os-flavor-access:is_public | True     |
| properties                 |          |
| ram                        | 2048     |
| rxtx_factor                | 1.0      |
| swap                       | 0        |
| vcpus                      | 1        |
+----------------------------+----------+
+----------------------------+-----------+
| Field                      | Value     |
+----------------------------+-----------+
| OS-FLV-DISABLED:disabled   | False     |
| OS-FLV-EXT-DATA:ephemeral  | 0         |
| description                | None      |
| disk                       | 40        |
| id                         | 3         |
| name                       | m1.medium |
| os-flavor-access:is_public | True      |
| properties                 |           |
| ram                        | 4096      |
| rxtx_factor                | 1.0       |
| swap                       | 0         |
| vcpus                      | 2         |
+----------------------------+-----------+
+----------------------------+----------+
| Field                      | Value    |
+----------------------------+----------+
| OS-FLV-DISABLED:disabled   | False    |
| OS-FLV-EXT-DATA:ephemeral  | 0        |
| description                | None     |
| disk                       | 80       |
| id                         | 4        |
| name                       | m1.large |
| os-flavor-access:is_public | True     |
| properties                 |          |
| ram                        | 8192     |
| rxtx_factor                | 1.0      |
| swap                       | 0        |
| vcpus                      | 4        |
+----------------------------+----------+
+----------------------------+-----------+
| Field                      | Value     |
+----------------------------+-----------+
| OS-FLV-DISABLED:disabled   | False     |
| OS-FLV-EXT-DATA:ephemeral  | 0         |
| description                | None      |
| disk                       | 160       |
| id                         | 5         |
| name                       | m1.xlarge |
| os-flavor-access:is_public | True      |
| properties                 |           |
| ram                        | 16384     |
| rxtx_factor                | 1.0       |
| swap                       | 0         |
| vcpus                      | 8         |
+----------------------------+-----------+
+----------------------------+---------+
| Field                      | Value   |
+----------------------------+---------+
| OS-FLV-DISABLED:disabled   | False   |
| OS-FLV-EXT-DATA:ephemeral  | 0       |
| description                | None    |
| disk                       | 1       |
| id                         | 6       |
| name                       | m2.tiny |
| os-flavor-access:is_public | True    |
| properties                 |         |
| ram                        | 512     |
| rxtx_factor                | 1.0     |
| swap                       | 0       |
| vcpus                      | 2       |
+----------------------------+---------+

Done.

To deploy a demo instance, run:

openstack --os-cloud=kolla-admin server create \
    --image cirros \
    --flavor m1.tiny \
    --key-name mykey \
    --network demo-net \
    demo1
```
从初始化的返回可以看出，脚本创建了一个叫public1的公共网络，用户稍后可以创建路由器和subnet，绑定浮动IP，来让虚拟机访问外网。
```
cat /etc/kolla/neutron-server/ml2_conf.ini
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security

[ml2_type_vlan]
network_vlan_ranges =

[ml2_type_flat]
flat_networks = physnet1

[ml2_type_vxlan]
vni_ranges = 1:1000

```
注意public1的物理设备就是physnet1。