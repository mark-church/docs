### Swarm Cluster Traffic Architecture
In a Docker Swarm cluster there are different types of network traffic that provide communications for various cluster functions. The three types of cluster traffic communicate on different ports. They are as follows:

#### Swarm Management Plane
Management plane traffic is the control messages between the Swarm controllers and the workers. This kind of traffic sends various commands between workers and controllers such as a new task that is sent to a worker to be executes or a heartbeat that communicates the health of a worker to the controllers. The default port for management traffic is `tcp 2377` (unless configured otherwise) and flows directly between workers and managers.

#### Swarm Control Plane
Control plane traffic carries network information between all nodes in the cluster. This includes information such as the location of containers in the cluster and DNS to IP mappings. The port used for control traffic is not configurable and uses `tcp & udp 7946`. The flow of this traffic is directly between worker nodes. Control plane traffic is also scoped to nodes which belong to the same Docker network. This reduces the amount of control plane traffic and aids in security. Two worker nodes that do not belong to the same Docker network will not communicate control plane traffic.

#### Swarm Data Plane
The data plane carries application traffic between containers or from containers to hosts outside of the cluster. The transport for data plane traffic will differ depending on the type of Docker networking driver that is being used for containers. If using a driver like the `bridge` driver, then application traffic will be exposed on and use whatever ports are configured by the operater. In the case of the `overlay` driver, application traffic is transported using VXLAN. This utilizes port `udp 4798` and flows directly between the worker nodes where the communicating containers are.


![Docker Network Control Plane](./img/controlplane.png)

## Out of Band Management for Docker Swarm
Segmentation of cluster management and control traffic from application traffic is a good practice that helps increase cluster stability and security. This ensures that critical cluster control traffic cannot be adversely affected by congestion from applications.  Proper design of an out of band management network should enforce network isolation and not allow application traffic to traverse the management network. 

The following example shows how Swarm parameters can be used to segment traffic types. The hosts that are being used each have two network interfaces, with one designated for application traffic and the other for control & mgmt traffic.

```
$ docker -v
Docker version 17.06.0-ce-rc1, build 7f8486a

$ ifconfig | grep -B 2 'inet addr'
eth1      Link encap:Ethernet  HWaddr 08:00:27:97:d8:88
          inet addr:192.168.1.10  Bcast:192.168.1.255  Mask:255.255.255.0
--

eth2      Link encap:Ethernet  HWaddr 08:00:27:2e:74:3f
          inet addr:192.168.2.10  Bcast:192.168.2.255  Mask:255.255.255.0
...

$ docker swarm init --advertise-addr eth1 --datapath-addr eth2
```


