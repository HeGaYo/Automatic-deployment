这篇文章主要是记录我配置cobbler，并成功让一台虚拟机通过PXE启动后安装自制的系统的过程。
> 两台主机（位于同一个网段）：

> 一台作为管理服务器，配置cobbler服务器。
> 一台作为客户机，启用PXE安装。

配置cobbler服务器的过程：

1. 安装cobbler和cobbler-web，debmirror还有createrepo也一并安装
```
$ sudo apt-get install cobbler cobbler-web debmirror createrepo
```
2. 检查cobbler是否安装成功
```
$ curl -I localhost/cobbler/
```
其实就是看localhost/cobbler/能否访问，如果可以，会返回HTTP 200 OK.

3. 检查cobbler的配置
```
$ cobbler check
```
4. 同步cobbler的配置
```
$ cobbler sync
```
一旦修改了cobbler的配置，就需要通过cobbler sync来同步，重新加载配置文件，不然白改

5. 初步配置cobbler 运行 cobbler check检查配置，根据提示修复错误
同时配置debmirror.conf配置文件
```
$ cp /usr/share/doc/debmirror/examples/debmirror.conf /etc/ 
```
注释掉`@dists`和`@arches` 两行
然后使配置生效
```
$ sudo cobbler sync
```

6. 配置cobbler-web的密码，可以修改用户密码
默认的访问信息——用户名:cobbler 密码：空
```
Htdigest /etc/cobbler/users.digest "Cobbler" cobbler
Changing password for user cobbler in realm Cobbler
New password:
Re-type new password:
```
7. 安装dnsmasq和tftp-hpa
```
$ sudo apt-get install dnsmasq tftp-hpa
```

8. 接下来就是修改各配置文件：

   8.1. 配置cobbler接管DHCP,DNS和tFTP服务 修改配置文件`/etc/cobbler/settings`
   ```
    manage_dhcp: 1
    manage_dns: 1
    manage_tftpd: 1
    restart_dhcp: 1
    restart_dns: 1
    pxe_just_once: 1
    next_server: <server's IP address>
    server: <server's IP address>
   ```
   使用`$ openssl passwd -1`生成密码，修改 `default_password_crypted: ""`
   8.2.修改配置文件`/etc/cobbler/modules.conf`

   ```
    [authentication]
    module = authn_configfile
    [authorization]
    module = authz_allowall
    [dns]
    module = manage_dnsmasq # uses dnsmasq
    [dhcp]
    module = manage_dnsmasq # uses dnsmasq
    [tftpd]
    module = manage_in_tftpd  # defaut, uses the system's tftp server, in this example, use tftpd-hpa
    ```
  8.3. 修改dhcp模板，`/etc/cobbler/dhcp.template`

   ```
   subnet 192.168.1.0 netmask 255.255.255.0 {
         option routers             192.168.1.1;
         option domain-name-servers 192.168.1.210,192.168.1.211;
         option subnet-mask         255.255.255.0;
         filename                   "/pxelinux.0";
         default-lease-time         21600;
         max-lease-time             43200;
         next-server                $next_server;
        ...
}

   ```
    8.3. 修改dnsmasq模板：`/etc/cobbler/dnsmasq.template`

  ```
  read-ethers
  addn-hosts = /var/lib/cobbler/cobbler_hosts
  #domain=
  dhcp-range=192.168.88.100,192.168.88.254
  dhcp-option=3,$next_server
  dhcp-lease-max=1000
  dhcp-authoritative
  dhcp-boot=pxelinux.0
  dhcp-boot=net:normalarch,pxelinux.0
  dhcp-boot=net:ia64,$elilo
  $insert_cobbler_system_definitions

  ```
  8.4. 修改tFTP模板：`/etc/cobbler/tftpd.template`
  ```
  {
        disable                = no
        socket_type       = dgram
        protocol              = udp
        wait                     = yes
        user                     = $user
        server                  = $binary
        server_args        = -B 1380 $args
        per_source         = 11
        cps                        = 100 2
        flags                      = IPv4
}

  ```
   8.5. 根据需求修改`/etc/lib/cobbler/distro_signatures.json`文件。如果所安装的系统的签名不存在则需要添加。
9. 在新制作的镜像中需要由一个`version`文件，内容为`tinycore`，作为标志文件使用。
10. 修改完后重启cobbler，
```
$ sudo service cobbler restart
$ sudo service apache2 restart
```
或者使用
```
cobbler signature update
```

11. 导入镜像

 11.1. 将镜像挂载到一个目录下
 ```
 mount -o loop,ro minbase_vinzor.iso /mnt/minbase
 ```
 11.2. 通过cobbler import导入镜像
 ```
 Cobbler import --name=minbase --arch=x86_64 --path=/mnt/minbase
 ```
 一般导入之后会默认产生一个profile以及distro。
 11.3. 查看distro, profile以及system的相关的信息,可加入参数 `--name=xxxx`单独输出相关的信息
 ```
 cobbler distro list
 cobbler profile list
 cobbler system list
 cobbler distro report
 cobbler profile report
 cobbler system report
 ```
 11.4. 通过cobbler system add的命令注册system

12. 当然啦，如果你熟悉cobbler-web的操作的话，第11步骤完全可以通过web端操作，比较简单。
