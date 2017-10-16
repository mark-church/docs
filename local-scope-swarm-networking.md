# Local Scope Swarm Networking
In 17.06 Docker provides support for local scope networks in Swarm. This includes any local scope network driver. Some examples of these are `bridge`, `host`, and `macvlan` though any local scope network driver, built-in or plug-in, will work with Swarm. Previously only swarm scope networks like `overlay` were supported.

## Lab Setup

This lab is setup using vagrant. The MACVLAN driver requires the network and interfaces to be in promiscuous mode. This mode is often not possible in cloud environments which is why vagrant is an ideal choice to test with.

Vagrantfile used for this example:

```
Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/trusty64"

  config.vm.define "node1" do |node1|
    node1.vm.hostname = 'node1'
    node1.vm.network "private_network", type: "dhcp"
    node1.vm.provision :shell, path: "bootstrap.sh"
  end
  config.vm.define "node2" do |node2|
    node2.vm.hostname = 'node2'
    node2.vm.network "private_network", type: "dhcp"
    node2.vm.provision :shell, path: "bootstrap.sh"
  end
  config.vm.define "node3" do |node3|
    node3.vm.hostname = 'node3'
    node3.vm.network "private_network", type: "dhcp"
    node3.vm.provision :shell, path: "bootstrap.sh"
  end
end
```

bootstrap.sh used for this example:

```
#!/usr/bin/env bash

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download-stage.docker.com/linux/ubuntu $(lsb_release -cs) test"
sudo apt-get update
sudo apt-get install -y linux-generic-lts-wily netcat iproute2 apt-transport-https ca-certificates curl
sudo apt-get -y install docker-ce
sudo usermod -aG docker vagrant
sudo reboot
```

Create a 3-node swarm cluster from these nodes.

```
$ docker -v
Docker version 17.06.0-ce-rc1, build 7f8486a

$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
7fs5izsl0rpqay7a540mslg9u *   node1               Ready               Active              Leader
a4ea2vg78szy98e5t62aiuiet     node2               Ready               Active
gvhwwbvmo2ysyzcuitcwvv5o1     node3               Ready               Active
```

## Host Networking
The `host` network driver simply places containers in the host network namespace. Containers will not receive their own network interfaces, but will see and use the network interfaces of the host.

```
$ docker service create --network host --replicas 3 --name swarm-host-test chrch/docker-pets:1.0
$ curl localhost:5000
<html>
  <head>
    <link rel='stylesheet' type='text/css' href="../static/style.css">
    <title>Docker PaaS</title>
  </head>
```

## Bridge Networking
  
The `bridge` driver is a simple L2 bridge that connects containers locally on a host. 

```
$ docker network create -d bridge --scope swarm swarm-bridge
k3u7o815nvvlosbv5z3mh2glq

$ docker service create --replicas 3 --network swarm-bridge --name swarm-bridge-test -p 5001:5000 chrch/docker-pets:1.0

$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                   PORTS
nsu2xo9scepk        swarm-host-test     replicated          3/3                 chrch/docker-pets:1.0
tlqjlwn27sv8        swarm-bridge-test   replicated          3/3                 chrch/docker-pets:1.0   *:5001->5000/tcp

$ curl localhost:5001
<html>
  <head>
    <link rel='stylesheet' type='text/css' href="../static/style.css">
    <title>Docker PaaS</title>
  </head> ...

```

## MACVLAN Networking
`macvlan` is a type of Docker network driver that provides the capability to give external network IPs directly to containers. This enables containers to communicate directly with nodes on the external network without NAT or overlays.

Remove the previous services from the cluster so that the ports do not collide.

```
node1 $ docker service rm $(docker service ls -q)
```

The macvlan interfaces of each Docker host and any virtual network you are using should be in promiscuous mode. If using virtualbox then the NIC of VM needs to be set as promiscuous in the settings of the network as well. 

```
$ sudo ip link set dev eth1 promisc on

$ netstat -i
Kernel Interface table
Iface   MTU Met   RX-OK RX-ERR RX-DRP RX-OVR    TX-OK TX-ERR TX-DRP TX-OVR Flg
0 BMRU
eth0       1500 0     58973      0      0 0         21393      0      0      0 BMRU
eth1       1500 0      4299      0      0 0          3301      0      0      0 BMPRU
```

A local network config is created on each host. The config holds host-specific information, such as the subnet allocated for this host's containers. `--ip-range` is used to specify a pool of IP addresses that is a subset of IPs from the subnet. This is one method of IPAM to guarantee unique IP allocations.

```
node1 $ docker network create --config-only --subnet 172.28.128.0/24 --gateway 172.28.128.1 -o parent=eth1 --ip-range 172.28.128.32/27 mv-config

node2 $ docker network create --config-only --subnet 172.28.128.0/24 --gateway 172.28.128.1 -o parent=eth1 --ip-range 172.28.128.64/27 mv-config

node3 $ docker network create --config-only --subnet 172.28.128.0/24 --gateway 172.28.128.1 -o parent=eth1 --ip-range 172.28.128.96/27 mv-config
```

Instantiate the macvlan network globally.

```
node1 $ docker network create -d macvlan --scope swarm --config-from mv-config mv-net
```
Deploy a service to the `mvlan1` network.

```
node1 $ docker service create --replicas 3 --network mv-net --name pets-macvlan chrch/docker-pets:1.0
```

We can now demonstrate multi-host connectivity by accessing a container on `node1` from `node2`. 

```
node1 $ docker exec -it pets-macvlan.2.ya09dwzqpkgknwzipemia1mtr ip add sho eth0
37: eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 02:42:ac:1c:80:21 brd ff:ff:ff:ff:ff:ff
    inet 172.28.128.33/24 scope global eth0
       valid_lft forever preferred_lft forever
       
node2 $ curl 172.28.128.33:5000
<html>
  <head>
    <link rel='stylesheet' type='text/css' href="../static/style.css">
    <title>Docker PaaS</title>
  </head> ...
```



