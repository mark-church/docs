# Docker EE and Contiv

```
$ sudo apt-get install -yopenvswitch-switch
```

#### Save the Master and Worker IPs

```
ucp-controller $ export NODE1_IP=$(hostname -i)
ucp-worker $ export NODE2_IP=$(hostname -i)
```

#### Deploy etcd as a container


```
docker run -d -v /usr/share/ca-certificates/:/etc/ssl/certs -p 4001:4001 -p 2380:2380 -p 2379:2379 \
 --name etcd quay.io/coreos/etcd:v2.3.8 \
 -name etcd0 \
 -advertise-client-urls http://$NODE1_IP:2379,http://$NODE1_IP:4001 \
 -listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 \
 -initial-advertise-peer-urls http://$NODE1_IP:2380 \
 -listen-peer-urls http://0.0.0.0:2380 \
 -initial-cluster-token etcd-cluster-1 \
 -initial-cluster etcd0=http://$NODE1_IP:2380,etcd1=http://$NODE2_IP:2380 \
 -initial-cluster-state new
```
 
```
 docker run -d -v /usr/share/ca-certificates/:/etc/ssl/certs -p 4001:4001 -p 2380:2380 -p 2379:2379 \
 --name etcd quay.io/coreos/etcd:v2.3.8 \
 -name etcd1 \
 -advertise-client-urls http://$NODE2_IP:2379,http://$NODE2_IP:4001 \
 -listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 \
 -initial-advertise-peer-urls http://$NODE2_IP:2380 \
 -listen-peer-urls http://0.0.0.0:2380 \
 -initial-cluster-token etcd-cluster-1 \
 -initial-cluster etcd0=http://$NODE1_IP:2380,etcd1=http://$NODE2_IP:2380 \
 -initial-cluster-state new
```




#### Verify that etcd is working
```
curl -L  https://github.com/coreos/etcd/releases/download/v2.1.0-rc.0/etcd-v2.1.0-rc.0-linux-amd64.tar.gz -o etcd-v2.1.0-rc.0-linux-amd64.tar.gz
tar xzvf etcd-v2.1.0-rc.0-linux-amd64.tar.gz
cd etcd-v2.1.0-rc.0-linux-amd64
./etcdctl
```

#### Install Contiv Docker Plugin

Install the Contiv master

```
docker plugin install contiv/v2plugin:1.0.2 plugin_role=master cluster_store=etcd://localhost:2379 ctrl_ip=${NODE1_IP} dbg_flag='-debug' --grant-all-permissions
```

Install the Contiv worker

```
docker plugin install contiv/v2plugin:1.0.2 plugin_role=worker cluster_store="etcd://localhost:2379" ctrl_ip=${NODE2_IP} dbg_flag='-debug' --grant-all-permissions
```




#### Uninstall Contiv Docker Plugin
```
$ docker plugin rm -f contiv/v2plugin:1.0.2
```