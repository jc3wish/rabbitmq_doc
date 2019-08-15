# 源码安装

RabbitMQ的源码在github上脱管,开发者们都是将server源码及插件源码等等分开存放的.并且有不少的依懒关系 ,所以我们下载他的rabbitmq-sever源码下来编译是有问题的,我们需要下载 tags包里的 ***.tar.xz**
包

```
httpManagerPort=15674
ERL_EPMD_PORT=4369
RABBITMQ_DIST_PORT=5672
RABBITMQ_NODE_PORT=15672

rabbitmqVersion=3.6.1
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/rabbitmq_v3_6_1/rabbitmq-server-3.6.1.tar.xz
xz -d rabbitmq-server-3.6.1.tar.xz
tar -xvf rabbitmq-server-3.6.1.tar
cd rabbitmq-server-3.6.1
NODENAME=rabbitmqTestNode
make TARGET_DIR=/usr/local/rabbitmq-${rabbitmqVersion} SBIN_DIR=/usr/local/rabbitmq-${rabbitmqVersion}/sbin MAN_DIR=/usr/local/rabbitmq-${rabbitmqVersion}/man DOC_INSTALL_DIR=/usr/local/rabbitmq-${rabbitmqVersion}/doc install
mkdir -p /usr/local/rabbitmq-${rabbitmqVersion}/etc/rabbitmq

touch -f /usr/local/rabbitmq-${rabbitmqVersion}/etc/rabbitmq/rabbitmq.config
touch -f /usr/local/rabbitmq-${rabbitmqVersion}/etc/rabbitmq/rabbitmq-env.conf

dataDir=/usr/local/rabbitmq-${rabbitmqVersion}/data
mkdir -p $dataDir/log
mkdir -p $dataDir/mnesia

#rabbit_auth_backend_internal
echo "[
{mnesia, [{dump_log_write_threshold, 10000},{dump_log_time_threshold,20000}]},
{rabbit, [
    {num_tcp_acceptors,5},
    {vm_memory_high_watermark_paging_ratio, 0.5},
    {vm_memory_high_watermark, 0.4},
    {loopback_users, []}
]},
{rabbitmq_management,
[{listener, [{port, $httpManagerPort},{ip, \"0.0.0.0\"}]}
]}
]." > /usr/local/rabbitmq-${rabbitmqVersion}/etc/rabbitmq/rabbitmq.config

echo "NODENAME=$NODENAME@$NODENAME
RABBITMQ_NODE_IP_ADDRESS=0.0.0.0
RABBITMQ_NODE_PORT=$RABBITMQ_NODE_PORT
RABBITMQ_LOG_BASE=$dataDir/log
RABBITMQ_MNESIA_BASE=$dataDir/mnesia" > /usr/local/rabbitmq-${rabbitmqVersion}/etc/rabbitmq/rabbitmq-env.conf

/usr/local/rabbitmq-$rabbitmqVersion/sbin/rabbitmq-plugins enable rabbitmq_management


#启动：
ERL_EPMD_PORT=$ERL_EPMD_PORT RABBITMQ_DIST_PORT=$RABBITMQ_DIST_PORT  /usr/local/rabbitmq-$rabbitmqVersion/sbin/rabbitmq-server

```
