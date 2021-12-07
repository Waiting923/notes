# Rabbitmq
## Rabbitmq集群初始化
1. 确认各节点rabbitmq-server服务已经正常安装启动
```shell
$ systemctl status rabbitmq-server
$ rabbitmqctl cluster_status(此时集群应为各节点脑裂状态)
```
2. 统一 .erlang.cookie，选择一个节点的erlang cookie覆盖至另外两个节点
```shell
$ cat /var/lib/rabbitmq/.erlang.cookie
```
3. 重启被覆盖的两个节点
```shell
$ systemctl restart rabbitmq-server
```
4. 把被修改的节点加入集群,两个节点依次执行
```shell
$ rabbitmqctl stop_app
$ rabbitmqctl reset
$ rabbitmqctl join_cluster rabbit@rabbit-node1
$ rabbitmqctl start_app
```
5. 加入好后确认集群状态
```shell
$ rabbitmqctl cluster_status
```
---
## Rabbitmq 开启dashboard
1. enable management plugin
```shell
$ rabbitmq-plugins enable rabbitmq_management
$ rabbitmq-plugins list
Listing plugins with pattern ".*" ...
 Configured: E = explicitly enabled; e = implicitly enabled
 | Status: * = running on rabbit@s4a-sms6
 |/
[  ] rabbitmq_amqp1_0                  3.7.10
[  ] rabbitmq_auth_backend_cache       3.7.10
[  ] rabbitmq_auth_backend_http        3.7.10
[  ] rabbitmq_auth_backend_ldap        3.7.10
[  ] rabbitmq_auth_mechanism_ssl       3.7.10
[  ] rabbitmq_consistent_hash_exchange 3.7.10
[  ] rabbitmq_event_exchange           3.7.10
[  ] rabbitmq_federation               3.7.10
[  ] rabbitmq_federation_management    3.7.10
[  ] rabbitmq_jms_topic_exchange       3.7.10
[E*] rabbitmq_management               3.7.10
[e*] rabbitmq_management_agent         3.7.10
[  ] rabbitmq_mqtt                     3.7.10
[  ] rabbitmq_peer_discovery_aws       3.7.10
[  ] rabbitmq_peer_discovery_common    3.7.10
[  ] rabbitmq_peer_discovery_consul    3.7.10
[  ] rabbitmq_peer_discovery_etcd      3.7.10
[  ] rabbitmq_peer_discovery_k8s       3.7.10
[  ] rabbitmq_random_exchange          3.7.10
[  ] rabbitmq_recent_history_exchange  3.7.10
[  ] rabbitmq_sharding                 3.7.10
[  ] rabbitmq_shovel                   3.7.10
[  ] rabbitmq_shovel_management        3.7.10
[  ] rabbitmq_stomp                    3.7.10
[  ] rabbitmq_top                      3.7.10
[  ] rabbitmq_tracing                  3.7.10
[  ] rabbitmq_trust_store              3.7.10
[e*] rabbitmq_web_dispatch             3.7.10
[  ] rabbitmq_web_mqtt                 3.7.10
[  ] rabbitmq_web_mqtt_examples        3.7.10
[  ] rabbitmq_web_stomp                3.7.10
[  ] rabbitmq_web_stomp_examples       3.7.10
```
2. 逐台重启rabbitmq
```shell
$ systemctl restart rabbitmq-server
```

3. 若启动失败，确认haproxy是否占用了15672端口，若占用将haproxy端口改为其它，如15673
```shell
2021-11-30 10:26:59.696 [error] <0.936.1> Failed to start Ranch listener rabbit_web_dispatch_sup_15672 in ranch_tcp:listen([{cacerts,'...'},{key,'...'},{cert,'...'},{port,15672}]) for reason eaddrinuse (address already in use)
2021-11-30 10:26:59.696 [error] <0.936.1> CRASH REPORT Process <0.936.1> with 0 neighbours exited with reason: {listen_error,rabbit_web_dispatch_sup_15672,eaddrinuse} in gen_server:init_it/6 line 352
2021-11-30 10:26:59.696 [error] <0.934.1> Supervisor {<0.934.1>,ranch_listener_sup} had child ranch_acceptors_sup started with ranch_acceptors_sup:start_link(rabbit_web_dispatch_sup_15672, ranch_tcp) at undefined exit with reason {listen_error,rabbit_web_dispatch_sup_15672,eaddrinuse} in context start_error
```
---
## notifications.info 报错
1. openstack组件中出现如下报错
```shell
2021-11-30 03:45:03.754 6 ERROR oslo.messaging._drivers.impl_rabbit [req-46ae2c2b-fb52-4777-a757-a93a5a45aa61 - - - - -] Unable to connect to AMQP server on 10.12.126.27:5672 after inf tries: Queue.declare: (404) NOT_FOUND - failed to perform operation on queue 'notifications.info' in vhost '/' due to timeout: NotFound: Queue.declare: (404) NOT_FOUND - failed to perform operation on queue 'notifications.info' in vhost '/' due to timeout
```
2. 需要赢重启rabbitmq，先停掉非ram主节点，后重启主节点，再启动另外两个节点(如下sms8为主节点)
```shell
$ rabbitmqctl cluster_status
Cluster status of node rabbit@s4a-sms6 ...
[{nodes,[{disc,['rabbit@sms6','rabbit@sms7','rabbit@sms8']}]},
 {running_nodes,['rabbit@sms8','rabbit@sms7','rabbit@sms6']},
 {cluster_name,<<"rabbit@sms6">>},
 {partitions,[]},
 {alarms,[{'rabbit@sms8',[]},
          {'rabbit@sms7',[]},
          {'rabbit@sms6',[]}]}]
```