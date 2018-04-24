redis是key-value存储的，放在内存中，并在磁盘持久化的数据结构存储系统，它可以用作数据库、缓存和消息中间件。
value的格式可以使string，set，list，map（即python里面的dict），sorted set（有序集合）


Redis的安装
apt-get install update
apt-get install redis-server
wget https://pypi.python.org/packages/source/r/redis/redis-2.9.1.tar.gz
tar xvzf redis-2.9.1.tar.gz
cd redis-2.9.1
python setup.py install


Redis测试安装成功
import python无误即可


Redis连接
r = redis.Redis(host = config.REDIS_IP,\
port = config.REDIS_PORT,\
password = config.REDIS_PASSWORD,\
db = config.REDIS_DB)


Redis通过本机客户端访问远程服务器段
Redis默认连接主机ip为127.0.0.1，端口号为6379
如果需要访问远程IP的计算机，远程计算徐需要安装Redis并且打开其服务器 redis.server.exe
客户机运行redis.cli.exe。


Redis添加数据
import jsonimport redis
r = redis.StrictRedis(host='localhost', port=6379, db=0)
 d={
	"product name":"vmware",
	…
	"uuid":"df2dfkja435jfl90"
    }
json_str = json.dumps(d)r.set('dmi_info', json_str)
json_str1 = r.get(’dmi_info')

data = json.loads(json_str)

连接池连接、Pipeline的用法
Redis使用connection pool来管理一个redis server的所有连接，避免每次建立和释放连接的开销。每个redis实例都会维护一个自己的连接池，可以建立一个连接池，作为参数Redis，这样可以实现多个Redis实例共享一个连接池。

Pipeline机制：可以在一次请求中执行多个命令，避免多次的往返时延
import redis    pool = redis.ConnectionPool(host='127.0.0.1', port=9212)  r = redis.Redis(connection_pool=pool)  pipe = r.pipeline()  pipe.set('one', 'first')  pipe.set('two', 'second')  pipe.execute() 


常用的存取
r.set ("name","jane")
r.get("name")

r.hset("usr","name","jane")
r.hset("usr","age","17")
r.hgetall("usr")——返回一个字典类型
r.delete("usr")


redis-server
redis-cli
/etc/redis/redis.conf


客户端检测连接
1、连接之前可以先检测服务是否启动；
2、测试启动 redis-cli ping 返回PONG，启动成功


停止Redis
redis-cli shutdown
redis-cli -p 6380 shutdown

