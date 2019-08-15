# Erlang安装

RabbitMQ是基于 Erlang 语言编写的,在运行RabbitMQ之前,必须先安装Erlang语言运行环境.

不同的RabbitMQ 版本可能对erlang版本也不一样.详细请查看官方文档

Erlang 里的典型的action模型,没有全局变量,并且变量只要一赋值则不可变更. Erlang里的process 之间通信全是依懒Erlang内部实的通信,Erlang集群之间节点可以通信,并且 Erlang自带ETS,DETS 及节点之间自动同步的key->value 数据库: MNESIA

```
yum install -y erlang
yum remove -y eralng
rm -f /usr/bin/erl

```

为什么一定要install 之后再remove呢,因为有些基础环境依懒,要不然后面rabbitmq会跑不起来

下载指定的erl版本

```
wget http://erlang.org/download/otp_src_19.1.tar.gz
yum -y install make gcc gcc-c++ kernel-devel m4 ncurses-devel openssl-devel unixODBC unixODBC-devel wxWidgets wxWidgets-devel
yum install unixODBC unixODBC-devel -y
yum install wxWidgets wxWidgets-devel -y

tar -zxvf otp_src_19.1.tar.gz
cd otp_src_19.1
./configure --without-javac
make
make install
ln -s /usr/local/bin/erl /usr/bin/erl

```
