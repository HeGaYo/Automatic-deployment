doc for web-ssh：GateOne
--------
>##### 1.安装python2.7+, tornado 2.2+
>
>1.1 安装前提
>
>     $ python -V
>     $ wget --no-check-certificate https://bootstrap.pypa.io/get-pip.py
>     $ sudo python get-pip.py
>     $ sudo pip install tornado
>1.2 验证安装环境

>     $ python -V
>     $ python -c "import tornado; print(tornado.version)"

--------

>##### 2.GateOne的获取和安装

>2.1 通过git上源码获取

>     $ sudo apt-get install git
>     $ git clone https://github.com/liftoff/GateOne.git
>     $ cd GateOne
>     $ sudo python ./setup.py install

>2.2 安装过程中可能会遇到的问题
>
> * 安装HTML5lib时提示setuptools版本太低，需要升级

>       $ pip install -U setuptools

>将setup.py成功运行之后即可启动gateone服务，至此，gateone的安装完成

--------
>##### 3.GateOne的测试

>默认的配置文件是/etc/gateone/conf.d/10.server.conf（位置不是绝对的）

>         $sudo gateone
>默认打开的端口号为443。在浏览器中输入''https://127.0.0.1:443''。
>如果配置成功，会出现gateone的页面。点击Terminal图标，可以在web端上使用命令行、也可以通过ssh登录到其他主机上进行操作。

--------
>##### 4.GateOne的嵌入到其他应用中
>
>4.1 配置文件的修改

>我们使用`GateOne/gateone/docs/embeddings_configs/99tutorial_chapter1.conf`作为默认的配置文件，可以设置端口号以及是否需要验证等

>* 本身这个文件是有一个语法问题， `auth=none`后面要加上一个`,`,不然运行的时候会报错
>
> * `origin = [*]`表示不对源进行检查
>
> * `"port": 8000` 修改数字可以改变gateone的端口号
>
>4.2 将配置文件放到一个目录下，并运行gateone(运行时需要添加setting_dir参数)
>
>      $ mkdir /tmp/chapter1
>      $ cd GateOne
>      $ cp gateone/docs/embedding_configs/99tutorial_chapter1.conf /tmp/chapter1
>      $ ./run_gateone.py --settings_dir=/tmp/chapter1
>
>4.3 将`Gateone/gateone/static/gateone.js`复制到应用的static文件夹中
>4.4 在你想要展示的页面文件.html文件下添加以下内容：
>```
><div id="gateone"></div>
>或者调一调格式啥的
><div id="gateone_container" style="width:60em; height:30em;">
>    <div id="gateone"></div>
></div>
>```
>4.5 引入gateone.js文件
>```
><script src="https://your-gateone-server/static/gateone.js"></script>
>```
>4.6 调用`GateOne.init()`函数来进行信息传递
>```
>GateOne.init({url:"https://yourgateone-server/:port"})
>```

--------
>##### 5.gateone免密码登录(autologin)
>其实gateone的url可以变为`https://ip_addr:port/?ssh=ssh://hostname@host_ipaddr`,这样只需要输入密码即可通过SSH访问其他主机。但还是不方便，如果有很多台主机，还要记住一堆主机的密码...

> 5.1 SSH 免密码登录
>
>```
>ssh-keygen -t rsa
>cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
>ssh vinzor@172.18.229.251 cat /home/vinzor/.ssh/authorized_keys >> ~/.ssh/authorized_keys
>scp ~/.ssh/authorized_keys >vinzor@172.18.229.251:~/.ssh/authorized_keys
>sudo chmod 755 .ssh 和 chmod 644 authorized_keys
>```
>* 可能报错：Agent admitted failure to sign using the key 问题 with ssh
>
>  解决方案：```$ ssh-add ~/.ssh/id_rsa```
>
> 5.2 将`id_rsa`写入到`.default_ids`里，形成对应关系
>```
>echo id_rsa > /var/lib/gateone/user/ANOUNYOUS/.ssh/.default_ids
>```
> 5.3 修改`GateOne.init`函数，添加参数`autoconnect_url`
>```
> GateOne.init({ url:'https://127.0.0.1:port', disableTermTransitions:true, showTitle:false, showToolbar:false, fillContainer:true, autoConnectURL:'ssh://hostname@ip_addr' }); } 
>```

--------
>##### 6.一台新主机加入
> 新主机不一定有`.ssh/authorized_keys`文件,如果直接用scp命令，会提示`no such file`。因此需要先创建文件夹再执行后面的操作。
>
>将服务器的`~/.ssh/authorized_keys`复制到`hostname@ip_addr:~/.ssh/authorized_keys`后，即可使用SSH无密码登录。此时再启动gateone服务器，url填写为指定的URL，即可通过公钥验证，无需再输入密码即可登录到远程主机。



参考网址：

http://liftoff.github.io/GateOne/Developer/index.html#embeddeding-gate-one-into-other-applications

http://www.cnblogs.com/lienhua34/p/4876300.html

http://www.cnblogs.com/lienhua34/p/4884115.html

https://github.com/liftoff/GateOne/issues/591
