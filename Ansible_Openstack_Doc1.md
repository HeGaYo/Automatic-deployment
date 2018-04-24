#目录
[TOC]

官方文档链接
https://docs.openstack.org/project-deploy-guide/openstack-ansible/ocata/overview-osa.html
## 概览
ansible 提供一个自动化平台健华系统和应用的部署。ansible通过SSH来管理系统。
ansible使用的是playbooks，用的是YAML语言。

**hosts分类**

- deployment hosts: 运行ansible playbooks
- target hosts：安装openstack服务以及设施等组件

**安装的流程**

1. Prepare deployment host
2. Prepare target hosts
3. Configure deployment
4. Run playbooks
5. Verify Openstack operation

## 安装需求
**Software requirement**（这里只考虑ubuntu)

- ubuntu(16.04)
- linux kernel version 3.13.0-34-generic 或者更新的内核版本
- 支持SSH，支持公钥验证
- 时钟同步 支持NTP
- python2.7.*x*
- en_US.UTF-8 as the locale

**CPU requirement**

- compute host：借助硬件虚拟化扩展的多核处理器
- infrastructure(storage) host：最好性能的多核处理器

**Disk requirement**
|host类型|需求|
|---|--|
|deployment hosts|10GB磁盘空间支持openstack-ansilbe仓库以及额外的必需软件|
|compute hosts|依赖于主机上运行的总实例个数，compute node至少需要1TB的磁盘空间，并且磁盘提供较低时延、更高IO吞吐量，例如SSD驱动|
|storage hosts|运行cinder的节点在openstack环境汇总使用最多磁盘空间，和compute hosts一样，需要较低时延、更高IO吞吐量的磁盘|
|infrastructure (storage) hosts|包含存储量服务，例如镜像服务glance，mariaDB等，这些hosts至少需要100GB的磁盘空间|
|logging hosts|保存各个容器的log信息，至少保留50GB的磁盘空间来保存log信息|

**Network recommendations**
部署Openstack环境可以只用一个物理网卡，但是只适用于一个小的环境。
在生产环境中：
- bonded network interface：提高性能和可靠性
- VLAN offloading：通过增加或者去除vlan 标签来提高硬件性能
- gigabit or 10 Gigabit Ethernet, 支持更高的网速，并且提高使用块存储服务的存储性能
- jumbo frames（巨型帧）, 允许每个包中发送更多的数据以提高网络性能

## 服务架构
Openstack是一个灵活部署配置的模型，能够部署所有的服务到一个单独的LXC(linux container)中或者直接部署在指定的hosts上。
可以让安装者根据特定的需求选择怎么更好的部署openstack
**基础设施服务**

- mariaDB with Galera：底层数据库，多主线(multimaster)数据库配置
- rabbitmq：用于高级消息队列协议(AMQP).
- memcached：内存缓存，加速事务的执行
- repository
- load balance
- utility container
- log aggregation host
- unbound DNS container

**Openstack service**

- Bare Metal (ironic)
- Block Storage (cinder)
- Compute (nova)
- Container Infrastructure Management (magnum)
- Dashboard (horizon)
- Data Processing (sahara)
- Identity (keystone)
- Image (glance)
- Networking (neutron)
- Object Storage (swift)
- Orchestration (heat)
- Telemetry (aodh, ceilometer, gnocchi)

## 网络架构
在openstack-ansible中，在target host上的网络要自动配置，因为每个user case都是不同的。
openstack-ansible在容器中用网桥来连接主机上物理、逻辑网络接口与虚拟网络接口。不同类型的target hosts有不同的网桥。
| target host 类型 | 网桥  | 描述 | 配置文件路径 |
|--|-----|----|----|
|LXC internal|`lxcbr0`|必须的。openstack-ansible一般会自动配置它，提供与容器的外部连接。此桥不直接连接主机上的任何物理或逻辑接口，因为iptables处理连接性。它连接到每个容器中的eth0|provider network 目录下 的openstack_user_config.yml|
|Container management|`br-mgmt`|必须的。提供了基础设施和openstack服务之间的管理和通信。该桥连接一个物理或者逻辑接口，通常是一个绑定VLAN子接口，它还附加到容器的eth1上|openstack_user_config.yml|
|Storage|`br-storage`|可选的。它为OpenStack服务和块存储设备之间的块存储设备提供了隔离访问。桥接在一个物理或逻辑接口上，通常是一个绑定VLAN子接口。它还附加到每个关联容器中的eth2|openstack_user_config.yml
|OpenStack Networking tunnel|`br-vxlan`|如果环境被配置为允许项目创建虚拟网络，则需要brl -vxlan桥。它为虚拟(VXLAN)隧道网络提供了接口。桥接在物理或逻辑接口上，通常是一个绑定1 VLAN子接口。它还附加到每个关联容器中的eth10。|openstack_user_config.yml|
|OpenStack Networking provider|`br-vlan`|必须的。它为VLAN标记或flat(无VLAN标记)网络提供基础设施。桥连接到物理或逻辑接口，通常是bond1。它在每个关联的容器中附加到eth11中VLAN类型网络。因为它只管理二层连接，因此它没有被分配ip地址。|openstack_user_config.yml|


## 存储架构
* **Block Storage (cinder)**
块存储(cinder)服务在一个环境中管理存储设备上的卷。在生产环境中，该设备通过存储协议(例如，NFS、iSCSI或Ceph RBD)向存储网络(`brc-storage`)和管理网络(`br-mgmt`)存储管理API提供存储。**实例通过计算主机上的管理程序通过存储网络连接到卷。**
![这里写图片描述](http://img.blog.csdn.net/20180308171250335?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGd5YW4yNQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
描述：
    - 首先通过指定的`cinder-volume`服务利用合适的cinder-driver来创建一个卷，通过使用一个向管理网络提交的API来创建。
    - 紧接着，等卷创建完成后，`nova-compute`服务通过存储网络将计算主机管理程序和卷连接。
    - 最后当计算主机管理程序连接上卷后，将卷作为本地硬件设备向实例展示。
* **Object Storage (swift)**
对象存储(swift)服务实现了一个高度可用的、分布式的、最终一致的object/blob存储，可以通过HTTP/HTTPS访问。
![](http://img.blog.csdn.net/20180308171410245?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGd5YW4yNQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
客户端通过管理网络上的负载均衡器(`br-mgmt`)访问`swift-proxy`服务。`swift-proxy`服务通过存储网络(`brc-Storage`)在对象存储主机上与帐户、容器和对象服务进行通信。对象存储主机之间的复制通过复制网络(`brl-repl`)完成。
* **Image storage (glance)**
镜像服务(glance)可以配置为在glance_store驱动程序支持的各种存储后端存储镜像。
![](http://img.blog.csdn.net/20180308171428404?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGd5YW4yNQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
描述：
    - 当客户端请求镜像时，`glance-api`服务通过存储网络(`brc-storage`)访问存储设备上的适当存储，并将其拉入缓存中。当再次请求相同的镜像时，它将直接从缓存提供给客户机。
    - 当一个实例被安排在一个计算主机上进行创建时，`nova-compute`服务会从管理网络(`br-mgmt`)中请求来自`glance -api`服务的镜像。
    - 在检索镜像之后，`nova-compute`服务将映像存储在其自己的镜像缓存中。当使用同一映像创建另一个实例时，将从本地基础镜像缓存中检索镜像。
* **Ephemeral storage (nova)**
当计算服务中的各种类型被配置为提供根或临时磁盘的实例时，nova-compute服务使用它的临时磁盘存储位置来管理这些分配。
![](http://img.blog.csdn.net/20180308171439104?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaGd5YW4yNQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
描述：
    - 计算主机配置为访问存储设备。计算主机通过使用存储协议(例如，NFS、iSCSI或Ceph RBD)通过存储网络(`brc-storage`)访问存储空间。
    - `nova-compute`服务配置管理程序，将已分配的实例磁盘作为设备提供给实例。
    - hypervisor将磁盘作为一个设备呈现给实例。
