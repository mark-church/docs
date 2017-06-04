```
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
7fs5izsl0rpqay7a540mslg9u *   node1               Ready               Active              Leader
a4ea2vg78szy98e5t62aiuiet     node2               Ready               Active
gvhwwbvmo2ysyzcuitcwvv5o1     node3               Ready               Active
```

Deploying a service in the host network namespace ...

```
$ docker service create --network host --replicas 3 --name swarm-host-test chrch/docker-pets:1.0
$ curl localhost:5000
<html>
  <head>
    <link rel='stylesheet' type='text/css' href="../static/style.css">
    <title>Docker PaaS</title>
  </head>
```
  
Deploying a service on local-scope bridge networks ...

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

Deploying a service on local-scope macvlan networks ...

Create the local config on each node

```
node1 $ docker network create --config-only --subnet 172.28.128.0/24 --gateway 172.28.128.1 -o parent=eth1 --ip-range 172.28.128.32/27 mv-config1
node2 $ docker network create --config-only --subnet 172.28.128.0/24 --gateway 172.28.128.1 -o parent=eth1 --ip-range 172.28.128.64/27 mv-config1
node3 $ docker network create --config-only --subnet 172.28.128.0/24 --gateway 172.28.128.1 -o parent=eth1 --ip-range 172.28.128.96/27 mv-config1
```

Instantiate the macvlan network globally and deploy the service

```
node1 $ docker network create -d macvlan --scope swarm --config-from mv-config1 --attachable mvlan1
node1 $ docker service create --replicas 3 --network swarm-macvlan --name swarm-macvlan-test chrch/docker-pets:1.0
```

Ping the container macvlan IP from a different host

```
node1 $ docker exec -it swarm-macvlan-test.2.ya09dwzqpkgknwzipemia1mtr ip add sho eth0
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



