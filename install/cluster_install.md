#集群部署

RabbitMQ的集群依懒于elang语言本身的分布式集群.
erlang语言是天生的分布式语言, 它可以通过cookie的配置,然后进行节点之间的通信.这感觉像是我们自己做的应用一样.

这个没错,erlang语方自身支持这个集群

操作步骤:

**在所有rabbitmq节点上设置,将cookie写到/var/lib/rabbitmq/.erlang.cookie 文件,并且设这个文件只有可读权限**

```
COOKIE=CQIRJLLIHWWVXCSRHQUH
echo $COOKIE > /var/lib/rabbitmq/.erlang.cookie
chown R rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
chmod a-w /var/lib/rabbitmq/.erlang.cookie
chmod o-r /var/lib/rabbitmq/.erlang.cookie
chmod g-r /var/lib/rabbitmq/.erlang.cookie

```

然后启动所有rabbitmq节点
```
/etc/init.d/rabbitmq-server start

```
### 案例

假如机器部署情况如下:

10.4.4.101 rabbitmq101

10.4.4.102  rabbitmq102

10.4.4.103  rabbitmq103

以 rabbitmq101 作cluster集群主节点

在 10.4.4.102，10.4.4.103 机器上运行

```
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbitmq101@rabbitmq101
rabbitmqctl start_app

```




