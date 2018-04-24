# Automatic-deployment
automatic deployment on private cloud platform based on openstack

# Automatic-deployment
## 项目简介
我们希望实现在少人或无人干预下，通过操作系统、openstack环境的自动化部署，实现服务器即插即用，简化云计算基础设施的管理、缩短设备上线的准备时间和减少现场运维人员的工作。我们的系统是基于flask-web开发，主要功能包括主机发现、操作系统、openstack环境自动安装以及集群管理、基于web-ssh的远程运维。我负责的部分是openstack云计算操作系统的自动部署流程设计与实现、对cobbler接口的二次开发以及web-ssh远程运维。

最终我们成功实现了对服务器进行操作系统安装、以及openstack自动部署并能够在web端对集群的节点进行查看、配置和管理。整个系统被封装为一个docker镜像来提供一键式的环境部署安装。

## 项目用途
- 文档记录，留档
- 记录学习笔记，总结归纳

## 内容涉及
- cobbler使用
- kolla项目学习
- web-ssh
- docker学习
- ssh公钥注入
