Goal
====
> Containers can join multiple networks which allows you to provide 
> fine grained network policy for connectivity and isolation. 
> By default a container will be created with one network attached. 
> If no network is specified then this will be the default docker0 network. 
> After the container has been created more networks can 
> be attached to a container using the `docker network connect` command.

Steps
=====
> In the following example we create two networks and attach them to the `c1` container. 
> Docker only allows a single network to be specified with the `docker run` command.
>  To connect multiple networks `docker network connect` is used to connect 
>  additional networks. If a container needs to be connected to multiple networks
>  before it runs then it is possible to attach networks to a created container
>  that has not started yet. This is done by instantiating a container 
>  with `docker create`, attaching the networks with `docker network connect`, 
>  and then running the created container with docker start. 
>  This will ensure that the container has all of the required 
>  network attachments on startup.

Step 1
----
Create the networks that you would like to attach to your container.

```
$ docker network create bluenet
$ docker network create rednet
```

Step 2a
----
Run the container. You can specify an initial network for it to start with. If no network is specified then the container will be attached to the default `docker0` network.

`$ docker run -itd --net bluenet --name c1 busybox sh`

Step 2b
----
There are some cases where it may be desirable for a container to not start until it has all the correct networks attached - for instance, an application that uses the networks immediately on startup. 

In this case it is best to create the container with `docker create`, attach the networks, and then start the container with `docker start`.

Create the container with its initial network. 

`$ docker create -it --net bluenet --name c1 busybox sh`

We can see that the container is in a `Created` but not running state.

```
$ docker ps -a
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS                  PORTS               NAMES
e616fc9965f6        busybox                "sh"                     16 seconds ago      Created                                     c1
```

Step 3
----
Attach the remaining networks.

`$ docker network connect rednet c1`

Step 3b
----
If the container has not been started yet then start the container.

`$ docker start c1`

Step 4
----
We can now verify that the running container is connected to multiple networks.

```
$ docker exec -it c1 sh

/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:1D:00:02
          inet addr:172.29.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe1d:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:16 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1296 (1.2 KiB)  TX bytes:648 (648.0 B)

eth1      Link encap:Ethernet  HWaddr 02:42:AC:1E:00:02
          inet addr:172.30.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe1e:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:16 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1296 (1.2 KiB)  TX bytes:648 (648.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

/ # ip route
default via 172.29.0.1 dev eth0
172.29.0.0/16 dev eth0  src 172.29.0.2
172.30.0.0/16 dev eth1  src 172.30.0.2  
```      

Results
===========
>  We can see above that every new network attachment Docker automatically
>  creates a new `eth` interface inside the container. Networks can be detached
>  from containers with `docker network disconnect` and the respective `eth`
>  inside the container will be removed.