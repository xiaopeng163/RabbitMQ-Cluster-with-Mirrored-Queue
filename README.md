# RabbitMQ-Cluster-with-Mirrored-Queue

## Environments Preparation

### Three RabbitMq nodes

Three CentOS 7 VM. Hostname are: `controller1`, `controller2` and `controller3`, and make sure
you `/etc/hosts` configure right.


### Install RabbitMQ server

```
% yum install rabbitmq-server
```

## Create RabbitMQ cluster

### On controller1

Start RabbitMQ server on node controller1.

```
% service rabbitmq-server start
```

Copy the elang cookie from controller1 to the other nodes: controller2, controller3

```
% scp /var/lib/rabbitmq/.erlang.cookie root@controller2:/var/lib/rabbitmq/.erlang.cookie
% scp /var/lib/rabbitmq/.erlang.cookie root@controller3:/var/lib/rabbitmq/.erlang.cookie
```

### On controller2

Make sure that the cookie copy from controller1 is owned by user `rabbitmq`, group `rabbitmq` and has mode 400 on both node.

```
% chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
% chmod 400 /var/lib/rabbitmq/.erlang.cookie
```
Start the RabbitMQ service.

```
% chkconfig rabbitmq-server on
% service rabbitmq-server start
```

Join to a cluster.

```
% rabbitmqctl stop_app
Stopping node rabbit@controller2 ...
...done.
% rabbitmqctl join_cluster rabbit@controller1
Clustering node rabbit@controller2 with rabbit@controller1 ...
...done.
% rabbitmqctl start_app
Starting node rabbit@controller2 ...
...done.
```

### On controller3

Do the same thing as you do on controller2


Go to controller1, and run:

```
% rabbitmqctl cluster_status
Cluster status of node rabbit@controller1 ...
[{nodes,[{disc,[rabbit@controller1,rabbit@controller2,rabbit@controller3]}]},
 {running_nodes,[rabbit@controller3,rabbit@controller2,rabbit@controller1]},
 {cluster_name,<<"rabbit@controller1">>},
 {partitions,[]}]
...done.
```

## Set Policy for mirrored queue


### RabbitMQ Management tools

Enable rabbitmq managment plugin and restart it

```
% rabbitmq-plugins enable rabbitmq_management
% service rabbitmq-server restart

```

Go to http://controller1:15672/#/ to declare a queue named `127.0.0.1` on controller1.


### Set Policy for mirror queue

Policy where all queues are mirrored to all nodes in the cluster:

```
% rabbitmqctl set_policy ha-all "." '{"ha-mode":"all"}'
```

Then you will see the queue on controller2 and controller3.


## Install haproxy

```
% yum install haproxy
```

Edit /etc/haproxy/haproxy.cfg and create a simple TCP proxy for RabbitMQ.

```
global
    daemon

defaults
    mode tcp
    maxconn 10000
    timeout connect 5s
    timeout client 100s
    timeout server 100s

listen rabbitmq 10.15.85.141:5670
    mode tcp
    balance roundrobin
    server controller1 controller1:5672 check inter 5s rise 2 fall 3
    server controller2 controller2:5672 check inter 5s rise 2 fall 3
    server controller3 controller3:5672 check inter 5s rise 2 fall 3
```

```
% service haproxy start
```
