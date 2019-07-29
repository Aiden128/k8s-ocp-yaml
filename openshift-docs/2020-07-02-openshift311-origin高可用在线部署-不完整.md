---
layout: post
title:  centos7.4 openshift origin 3.11 高可用在线安装
category: openshift，origin
description: 
---

采用单节点模式，在centos7.4安装openshift3.11社区版，环境需联网。
若离线安装，提前拉好源和镜像。


官方文档
https://docs.okd.io/3.11/install/host_preparation.html


### 配置环境

#### 1. 系统配置  
操作系统： centos 7.4 mini install，最小化安装  
虚机共7台  
haproxy 1台 2核 8G ,可以用公有云负载代替  
master 3台 2核 8G  
node 3台 4核 16G  
网卡配置IP netmask gateway dns 

#### 2. 修改ip_forward配置
openshift节点master， infra， node ，haproxy修改/etc/sysctl.conf文件：

```bash
echo "net.ipv4.ip_forward = 1" >>/etc/sysctl.conf
 # 使配置生效
sysctl -p /etc/sysctl.conf
```

#### 3. 每台设置主机名及配置ssh免密 

```bash
# 分别设置主机名

hostnamectl set-hostname master1.ocp311.com
...
hostnamectl set-hostname node3.ocp311.com

# 每台节点修改/etc/hosts ，解析所有节点主机名，使用内网IP
LB内部IP haproxy.ocp311.com
172.16.160.11 master1.ocp311.com

...
172.16.160.15 node2.ocp311.com
172.16.160.16 node3.ocp311.com
#  配置ssh免密，只在master1执行
ssh-keygen   # 一路敲回车
ssh-copy-id haproxy.ocp311.com # 输入密码
ssh-copy-id master1.ocp311.com 
......
在master1 依次对每个节点执行
ssh xxx.ocp311.com hostname -i # 验证免密
```

#### 4. 更新yum源及设置selinux

每个节点都要做  
注意此处不要用官方的epel源，那个ansible是2.8.0的，按照官方文档，ansible版本需不低于2.5.7，不支持2.8.0，此处自己添加ansible源，安装2.7.4

```bash
# 添加ansible 源
cat > /etc/yum.repos.d/ansible.repo << EOF
[ansible]
name=ansible
baseurl=https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/
enabled=0
gpgcheck=0
EOF

# 安装 ansible
yum -y --enablerepo=ansible install ansible-2.7.4

# 安装其他需要的包及更新
yum install -y wget git net-tools bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct nfs-utils python-passlib java-1.8.0-openjdk-headless ntp

yum update -y

# 安装并启动docker 
yum install docker-1.13.1 -y
systemctl enable docker
systemctl start docker
systemctl is-active docker

# 修改selinux 
setenforce 0

# /etc/selinux/config  把SELINUX=enforcing 改为 SELINUX=permissive
sed -i -e "s/^SELINUX=enforcing/SELINUX=permissive/" /etc/selinux/config

reboot  # 必须做
``` 


### docker安装配置

#### 1. docker 安装

yum install docker-1.13.1

Verify that version 1.13 was installed:

```bash
rpm -V docker-1.13.1
docker version
```

#### 2. docker 独立存储配置，需要独立sdb磁盘，POC和测试环境忽略这步

停止docker服务

```bash
systemctl stop docker
rm -rf /var/lib/docker/
```

配置/etc/sysconfig/docker-storage-setup
```bash
cat <<EOF > /etc/sysconfig/docker-storage-setup
STORAGE_DRIVER=overlay2
DEVS=/dev/sdb
CONTAINER_ROOT_LV_NAME=dockerlv
CONTAINER_ROOT_LV_SIZE=100%FREE
# 下面这个mount的路径就比较特殊了，其实这个相关的配置文件就是使用sdb创建一个pv、vg、lv，然后格式化成xfs，最后mount一下，docker只是去使用这个路径下空间，如果docker存储的路径需要改变，这个mount的路径也需要进行修改。
CONTAINER_ROOT_LV_MOUNT_PATH=/var/lib/docker
VG=docker-vg
EOF
```

初始化docker存储启动docker
```bash
docker-storage-setup
systemctl start docker
```
最后通过docker info去查看存储相关的配置进行验证

注意点
如果此前这块盘被使用过，创建了pv，vg，lv，需依次删除lv、vg、pv，然后再使用fdisk删除sdb1分区，执行partprobe，最后再执行docker-storage-setup。 如果pv删了，vg删不掉，重启下系统。
在使用一块盘、分区、VG，lv默认使用的是所在VG的40%空间，这个要看具体
执行docker-storage-setup报错如下错误时，将WIPE_SIGNATURES=true追加至/etc/sysconfig/docker-storage-setup文件后。

ERROR: Found dos signature on device /dev/sdb at offset 0x1fe. Wipe signatures using wipefs or use WIPE_SIGNATURES=true and retry.

如果再次执行docker-storage-setup还会报如下错，删除/etc/sysconfig/docker-storage文件再次执行即可。

docker-storage-setup

#### 3. 启动docker服务
```bash
 systemctl enable docker
 systemctl start docker
 systemctl is-active docker
```

修改docker配置
/etc/sysconfig/docker 替换之前的OPTIONS

```bash
cp /etc/sysconfig/docker /etc/sysconfig/docker.bak.$(date "+%Y%m%d%H%M%S");
sed  -i s/".*OPTIONS=.*"/"OPTIONS='--log-driver=json-file --insecure-registry=172.30.0.0\/16 --insecure-registry=registry.ocp311.com:5000 --selinux-enabled --log-opt max-size=1M --log-opt max-file=3'"/g /etc/sysconfig/docker;

##Restart the Docker service:
systemctl restart docker
```
### openshift安装

#### 1. 下载ansible 脚本 
```bash
cd ~
git clone https://github.com/openshift/openshift-ansible
cd openshift-ansible
git checkout release-3.11

# 也可以手动去网站下载release-3.11的分支
```

#### 2. 准备ansible hosts文件 
```bash
# 在openshift-ansible 目录
# 修改etory/hosts文件
# 注意这个里面没有写LB，LB使用公有云负载，提前配置好，8443 端口负载到三个master的8443 

[root@master1 openshift-ansible]# cat inventory/hosts
# Create an OSEv3 group that contains the master, nodes, etcd, and lb groups.
# The lb group lets Ansible configure HAProxy as the load balancing solution.
# Comment lb out if your load balancer is pre-configured.
[OSEv3:children]
masters
nodes
etcd

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
ansible_ssh_user=root
openshift_deployment_type=origin

# uncomment the following to enable htpasswd authentication; defaults to DenyAllPasswordIdentityProvider
#openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]

# Native high availbility cluster method with optional load balancer.
# If no lb group is defined installer assumes that a load balancer has
# been preconfigured. For installation the value of
# openshift_master_cluster_hostname must resolve to the load balancer
# or to one or all of the masters defined in the inventory if no load
# balancer is present.
openshift_master_cluster_method=native
openshift_master_cluster_hostname=haproxy.ocp311.com
openshift_master_cluster_public_hostname=haproxy.ocp311.com

# 部署metrics 监控，弹性会用到
openshift_metrics_install_metrics=true
## EFK
openshift_logging_install_logging=true
#关闭prometheus，默认是安装
openshift_cluster_monitoring_operator_install=false
#
# catalog 屏蔽，用不到
ansible_service_broker_install=false
openshift_enable_service_catalog=false
template_service_broker_install=false

# apply updated node defaults
#openshift_node_groups=[{'name': 'node-config-all-in-one', 'labels': ['node-role.kubernetes.io/master=true', 'node-role.kubernetes.io/infra=true', 'node-role.kubernetes.io/compute=true'], 'edits': [{ 'key': 'kubeletArguments.pods-per-core','value': ['20']}]}]

# host group for masters
[masters]
master1.ocp311.com
master2.ocp311.com
master3.ocp311.com

# host group for etcd
[etcd]
master1.ocp311.com
master2.ocp311.com
master3.ocp311.com

# Specify load balancer host

# host group for nodes, includes region info
[nodes]
master1.ocp311.com openshift_node_group_name='node-config-master'
master2.ocp311.com openshift_node_group_name='node-config-master'
master3.ocp311.com openshift_node_group_name='node-config-master'
node1.ocp311.com openshift_node_group_name='node-config-infra'
node2.ocp311.com openshift_node_group_name='node-config-infra'
node3.ocp311.com openshift_node_group_name='node-config-compute'


下面这个在华为云预监测失败

[root@origin311 openshift-ansible-release-3.11]# cat inventory/hosts
# Create an OSEv3 group that contains the masters, nodes, and etcd groups
[OSEv3:children]
masters
nodes
etcd

# Set variables common for all OSEv3 hosts
[OSEv3:vars]
# SSH user, this user should allow ssh based auth without requiring a password
ansible_ssh_user=root

# If ansible_ssh_user is not root, ansible_become must be set to true
#ansible_become=true

openshift_deployment_type=origin   #代表开源版

# 指定安裝的 OpenShift 版本
openshift_release="3.11"
openshift_image_tag=v3.11.0
openshift_pkg_version=-3.11.0
openshift_use_openshift_sdn=true

os_sdn_network_plugin_name='redhat/openshift-ovs-networkpolicy'
# When installing osm_cluster_network_cidr and openshift_portal_net must be set.
# Sane examples are provided below.
#osm_cluster_network_cidr=10.128.0.0/14
#openshift_portal_net=172.30.0.0/16

# disable checks unsupported
 openshift_disable_check=docker_storage,memory_availability,docker_image_availability,disk_availability,docker_storage_driver

# uncomment the following to enable htpasswd authentication; defaults to AllowAllPasswordIdentityProvider
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]
# Defining htpasswd users
#openshift_master_htpasswd_users={'user1': '<pre-hashed password>', 'user2': '<pre-hashed password>'}

# default subdomain to use for exposed routes, you should have wildcard dns
# for *.apps.test.example.com that points at your infra nodes which will run
# your router
openshift_master_default_subdomain=apps.ocp311.com

#Set cluster_hostname to point at your load balancer
#将来平台的访问域名
openshift_master_cluster_method=native
openshift_master_cluster_hostname=haproxy.ocp311.com
openshift_master_cluster_public_hostname=haproxy.ocp311.com

# Cluster metrics are not set to automatically deploy.
# The metrics public URL can be set during cluster installation using
# the openshift_metrics_hawkular_hostname Ansible variable, which defaults to:
# https://hawkular-metrics.{{openshift_master_default_subdomain}}/hawkular/metrics
openshift_metrics_install_metrics=true

ansible_service_broker_install=false
openshift_enable_service_catalog=false
template_service_broker_install=false

enable_excluders=false

# registry passwd
#oreg_url=registry.ocp311.com:5000/openshift3/ose-${component}:${version}
#oreg_url=registry.ocp311.com:5000/openshift/origin-${component}:${version}
openshift_examples_modify_imagestreams=true

# Enable cockpit
osm_use_cockpit=true
#
# Set cockpit plugins
osm_cockpit_plugins=['cockpit-kubernetes']

openshift_node_groups=[{'name': 'node-config-master', 'labels': ['node-role.kubernetes.io/master=true']}, {'name': 'node-config-infra', 'labels': ['node-role.kubernetes.io/infra=true']}, {'name': 'node-config-compute', 'labels': ['node-role.kubernetes.io/compute=true,node-role.kubernetes.io/infra=true']}]

# docker config
#openshift_docker_additional_registries=registry.ocp311.com:5000
openshift_docker_insecure_registries=registry.ocp311.com:5000
#openshift_docker_blocked_registries
openshift_docker_options="--insecure-registry 172.30.0.0/16 --log-driver json-file --log-opt max-size=1M --log-opt max-file=3"

# OpenShift Router Options
# Router selector (optional)
# Router will only be created if nodes matching this label are present.
# Default value: 'node-role.kubernetes.io/infra=true'
#openshift_hosted_router_selector='node-role.kubernetes.io/infra=true'
#
# Router replicas (optional)
# Unless specified, openshift-ansible will calculate the replica count
# based on the number of nodes matching the openshift router selector.
#openshift_hosted_router_replicas=2

# Openshift Registry Options
# Registry selector (optional)
# Registry will only be created if nodes matching this label are present.
# Default value: 'node-role.kubernetes.io/infra=true'
#openshift_hosted_registry_selector='node-role.kubernetes.io/infra=true'
#
# Registry replicas (optional)
# Unless specified, openshift-ansible will calculate the replica count
# based on the number of nodes matching the openshift registry selector.
#openshift_hosted_registry_replicas=2


# openshift_cluster_monitoring_operator_install=false
# openshift_metrics_install_metrics=true
# openshift_enable_unsupported_configurations=True
#openshift_logging_es_nodeselector='node-role.kubernetes.io/infra: "true"'
#openshift_logging_kibana_nodeselector='node-role.kubernetes.io/infra: "true"'

## EFK
openshift_logging_install_logging=true


# host group for masters
[masters]
master1.ocp311.com
master2.ocp311.com
master3.ocp311.com

# host group for etcd
[etcd]
master1.ocp311.com
master2.ocp311.com
master3.ocp311.com

# host group for nodes, includes region info
[nodes]
master1.ocp311.com openshift_node_group_name='node-config-master'
master2.ocp311.com openshift_node_group_name='node-config-master'
master3.ocp311.com openshift_node_group_name='node-config-master'
node1.ocp311.com openshift_node_group_name='node-config-compute'
node2.ocp311.com openshift_node_group_name='node-config-compute'
node3.ocp311.com openshift_node_group_name='node-config-compute'
```

#### 3. 执行安装脚本

```bash
# 使用ansible脚本预检查
ansible-playbook -i inventory/hosts playbooks/prerequisites.yml

# 执行完成会在每台节点添加openshift在线源，此源下载比较慢。若使用离线源，可以手动替换。 /etc/yum.repos.d/CentOS-OpenShift-Origin311.repo

# 执行部署脚本
ansible-playbook -i inventory/hosts playbooks/deploy_cluster.yml

# 卸载脚本，如果执行出错或者想要重新部署可以执行卸载脚本
正常情况下不要执行
### ansible-playbook -i inventory/hosts playbooks/adhoc/uninstall.yml ###
```

#### 4. 部署FAQ，异常处理
1. 部署报错 Install docker excluder - yum

TASK [openshift_excluder : Install docker excluder - yum]
fatal: [node2.ocp311origin.com]: FAILED! => {"attempts": 3, "changed": false, "msg": "Failure talking to yum: 'ascii' codec can't encode characters in position 173-177: ordinal not in range(128)"}

这个报错，是因为脚本部署过程修改了/etc/resolv.conf 中nameserver，改成了本地IP，而dnsmasq未正常启动，不能访问外部域名，nameserver 被改成本地IP，这个是正常的，只要dnsmasq服务正常，就没有问题，之前的dns配置文件被移到了 /etc/dnsmasq.d/origin-upstream-dns.conf
查看dnsmasq服务发现报错信息，手动启动dnsmasq起不来，和dbus有关，可能是部署过程更新了dbus导致异常。

解决方法：重启dbus，然后重启dnsmasq并查看dnsmasq状态，还不行就重启操作系统。

2. 部署报错 Wait for control plane pods to appear

TASK [openshift_control_plane : Wait for control plane pods to appear] ************************************************************************************
Wednesday 29 May 2019  16:09:27 +0800 (0:00:00.098)       0:06:23.053 *********
FAILED - RETRYING: Wait for control plane pods to appear (60 retries left).

ansible版本问题，2.8.0 版本会卡在这地方，2.7.4正常，在github上官方有说明，不低于2.5.7，并且低于2.8.0
解决方法：
更换ansible版本，如果严格执行上述步骤，ansible版本应该是正确的，不会出现此问题
	部署停留在Verify that the console is running

3. 
TASK [openshift_web_console : Verify that the console is running] ***********************************************
Thursday 30 May 2019  14:04:12 +0800 (0:00:00.109)       0:04:22.223 **********
  1. Hosts:    origin311.ocp311.com
     Play:     OpenShift Metrics
     Task:     openshift_metrics : fail
     Message:  'keytool' is unavailable. Please install java-1.8.0-openjdk-headless on the control node
Failure summary:


  1. Hosts:    origin311.ocp311.com
     Play:     OpenShift Metrics
     Task:     generate htpasswd file for hawkular metrics
     Message:  This module requires the passlib Python library

解决方法：
因为装metrics需要
yum install java-1.8.0-openjdk-headless python-passlib 
