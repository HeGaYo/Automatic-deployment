doc for preseed_ssh-key injected
------

由于新安装的系统没有`/proc/cmdline` 中的数据，我们无法在客户机中获取`URL`。因此在ClipAgent.py中通过`scp`之类的操作来获取`authorized_keys`的做法不太好，可能实现不了。因此我尝试在preseed文件中添加东西，使得在装系统的过程中就能将SSH公钥注入。

参考文档： http://www.itdadao.com/articles/c15a337907p0.html

该文档主要讲的是cobbler中的SNIPPET怎么用，后面的公钥注入还是靠自己尝试出来。

preseed文件目前暂时使用`cobbler`自带的`sample.seed`，在`/var/lib/cobbler/kickstarts/`目录下。一般`cobbler import`如果要修改对应的seed文件，有两个方法：第一个是直接在cobbler_web页面上改，很方便；第二个是通过`cobbler profile edit ...`， 异曲同工。主要修改的文件为`preseed_late_default`：在`/var/lib/cobbler/scripts/`目录下。
还用到了本地源获取文件，将`~/.ssh/authorized_keys`放到了`/var/lib/cobbler/svc/`目录下。在preseed文件中可以用到`$http_server`参数获取管理服务器的URL。

sample.seed文件中有一段代码：
```
d-i preseed/early_command string wget -O- \
   http://$http_server/cblr/svc/op/script/$what/$name/?script=preseed_early_default | \
   /bin/sh -s

d-i preseed/late_command string wget -O- \
   http://$http_server/cblr/svc/op/script/$what/$name/?script=preseed_late_default | \
   chroot /target /bin/sh -s
```
表示的是在preseed文件执行之前与之后需要执行的语句，对应的文件在`/var/lib/cobbler/scripts/`目录下的`preseed_early_default`与`preseed_late_default`。

后半段代码的逻辑是首先获取到`preseed_late_default`文件，然后转换路径到target目录下。因为在推荐引导文件的时候它的根目录并不是系统的根目录，系统的根目录其实是在根目录下面的target下。因此需要转换路径到target目录下再执行`preseed_late_default`脚本中的内容。

下面来看看`preseed_late_default`下的内容：
```
# Start preseed_late_default
# This script runs in the chroot /target by default
$SNIPPET('post_install_network_config_deb')
$SNIPPET('late_apt_repo_config')
$SNIPPET('post_run_deb')
$SNIPPET('download_config_files')
$SNIPPET('kickstart_done')
#新加的内容的位置
# End preseed_late_default
```
这些`$SNIPPET(...)`对应的运行文件在`/var/lib/cobbler/SNIPPET`目录下。值得一提的是`$SNIPPET(kickstart_done)`。这个是在系统装完后需要重启时，启动cobbler的`nopxe`的设置，使得下一次进入系统时不会进入PXE菜单而是直接进入安装好的系统。如果没有这个设置的话就是漫长的循环，最终是装不好系统的!

了解了这些之后我们就可以在`preseed_late_default`脚本下加东西了。切记，你已经在preseed文件下切换目录到了`target`下了，脚本执行的路径也是在`target`下的。
```
$SNIPET(test)
set srv = $getVar('http_server', '')
cd /root/
mkdir .ssh
wget http://$http_server/cobbler/svc/authorized_keys
echo $svc > /root/.ssh/test
```
无需分号分割，只能执行简单的shell语句。这里希望做的事情就是创建`.ssh`目录，并且获取本地源上的`authorized_keys`。然后将一个参数写入到`test`中。

这些都设置好之后就开始装系统了...漫长的等待...


问题1： 貌似在`preseed_late_default`文件下无法获取$http_server的参数值诶...getVar也尝试了，好像也没用


修改方法：
http://purplepalmdash.github.io/2015/06/29/insert-public-key-into-cobbler-deployed-system/

直接在/var/lib/cobbler/snippet文件夹下添加文件publickey_root
```
cd /root
mkdir --mode=700 .ssh
cat >> .ssh/authorized_keys << "PUBLIC_KEY"
ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA3B3GtGuKY0l2Ak9+WSkorY7R+Cx5/u3RMua/7GrvP05IPywQdkR+mqwdRydNjyhB96nHlYZtr8Fbfn5iwqn0j8dz8wmTZicBNeRqIdbe/YUje5NjXxDXjYda63VfDhpgzJ53KICTx6pBhGaeOKS/U5HqCpDbF7ODP8siU7bRhk1LkIQ6VwZYUg7b0oR+Sw6XJ31Z7gs4CWF6zfjfQQoF7EoMA+dnqvt2K4PQPXNSBJQx3qb9jyXIXvo333PcfIX6mD1TW1wDAIXLm4qz4mi7C8Ax9h+T/D98r08WX360vC5Tzr8feXMs6H4il4s4Ftq7RVoqCNKmG3AB1LTp4AQAzw== root@z_WHServer
PUBLIC_KEY
chmod 600 .ssh/authorized_keys
cat >> .ssh/config <<EOF
StrictHostKeyChecking no
UserKnownHostsFile /dev/null
EOF
```
然后直接在`preseed_late_default`文件下添加`$SNIPPET(publickey_root)`

然后就成功啦~
