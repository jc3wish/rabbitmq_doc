# rpm安装

rpm安装比较简单,并且还一次性帮你把基础的脚也安装好了,生产环境还是比较推荐这个方式

安装步骤:

```
wget http://packages.erlang-solutions.com/erlang-solutions-1.0-1.noarch.rpm
rpm -Uvh erlang-solutions-1.0-1.noarch.rpm

```


如果这个地方这个源安装不了,请手动配置如下:
```

echo "[erlang-solutions]
#name=Centos $releasever - $basearch - Erlang Solutions
#baseurl=http://packages.erlang-solutions.com/rpm/centos/$releasever/$basearch
#gpgcheck=0
#gpgkey=http://packages.erlang-solutions.com/debian/erlang_solutions.asc
#enabled=1" >  /etc/yum.repos.d/erlang_solutions.repo

```


**这里一定要先把 erl yum 一下，再remove掉，要不然不这么做的话，后面rabbitmq会安装不了**

```
yum install erlang -y
yum remove erlang -y
yum groupremove erlang -y
rm -f /usr/bin/erl

#下载指定的erlang版本并安装

wget wget http://erlang.org/download/otp_src_19.1.tar.gz
yum -y install make gcc gcc-c++ kernel-devel m4 ncurses-devel openssl-devel unixODBC unixODBC-devel wxWidgets wxWidgets-devel
yum install unixODBC unixODBC-devel -y
yum install wxWidgets wxWidgets-devel -y

tar -zxvf otp_src_19.1.tar.gz
cd otp_src_19.1
./configure --without-javac
make
make install
cd ../
#这里一定要做做一个软连接，要不然会找不到erl的
ln -s /usr/local/bin/erl /usr/bin/erl

#下提rabbitmq指定版本rpm安装包
wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.1/rabbitmq-server-3.6.1-1.noarch.rpm
rpm --import https://www.rabbitmq.com/rabbitmq-signing-key-public.asc
yum install rabbitmq-server-3.6.1-1.noarch.rpm -y

mkdir -p /data/rabbitmq
mkdir -p /data/rabbitmq/log/rabbit-mgmt
mkdir -p /data/rabbitmq/mnesia

chown -R rabbitmq:root /data/rabbitmq

NodeName=rabbitmqMyNodeName

touch -f /etc/rabbitmq/rabbitmq.config
echo "[
{mnesia, [{dump_log_write_threshold, 10000},{dump_log_time_threshold,20000}]},
{rabbit, [
    {num_tcp_acceptors,25},
    {vm_memory_high_watermark_paging_ratio, 0.5},
    {vm_memory_high_watermark, 0.5},
    {loopback_users, []},
    {queue_master_locator,<<\"min-masters\">>}
]},
{rabbitmq_management,
[{listener, [{port, 15672},{ip, \"0.0.0.0\"}]},
{http_log_dir,  \"/data/rabbitmq/log/rabbit-mgmt\"},
{rates_mode,    none}
]}
]." > /etc/rabbitmq/rabbitmq.config

echo "NODENAME=$NodeName@$NodeName
RABBITMQ_NODE_IP_ADDRESS=0.0.0.0
RABBITMQ_NODE_PORT=5672
RABBITMQ_LOG_BASE=/data/rabbitmq/log
RABBITMQ_MNESIA_BASE=/data/rabbitmq/mnesia" > /etc/rabbitmq/rabbitmq-env.conf

chkconfig rabbitmq-server on

#这里设置，这名字自己修改，并且要自己做好host

echo “27.0.0.1   $NodeName” >> /etc/hosts

```


```



**用于集群使用, 如果是集群部署需要这里是需要操作的**
```
COOKIE=CQIRJLLIHWWVXCSRHQUH
echo $COOKIE > /var/lib/rabbitmq/.erlang.cookie
chown R rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
chmod a-w /var/lib/rabbitmq/.erlang.cookie
chmod o-r /var/lib/rabbitmq/.erlang.cookie
chmod g-r /var/lib/rabbitmq/.erlang.cookie
```

**启动rabbitmq插件**

rabbitmq-plugins enable rabbitmq_management
rabbitmq-plugins enable rabbitmq_shovel
rabbitmq-plugins enable rabbitmq_shovel_management


**开启5672 .15672. 4369 三个端口的防火墙**

```
echo "-A INPUT -p tcp -m state --state NEW -m tcp --dport 15672 -j ACCEPT" >> /etc/sysconfig/iptables
echo "-A INPUT -p tcp -m tcp --dport 5672 -j ACCEPT" >> /etc/sysconfig/iptables
echo "-A INPUT -p tcp -m tcp --dport 25672 -j ACCEPT" >> /etc/sysconfig/iptables
echo "-A INPUT -p tcp -m tcp --dport 4369 -j ACCEPT" >> /etc/sysconfig/iptables

service iptables restart

```

#加入集群，这个请自己决定要不要
#rabbitmqctl stop_app
#rabbitmqctl join_cluster node1@node1
#rabbitmqctl start_app
