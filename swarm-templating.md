#Using Swarm Templates for Service Introspection

Swarm Templating can support many common use cases where one might need to inject information about a service to the service itself. This allows for some introspection use cases such as sending the service name or node name to a service task.


The Swarm templating options are detailed in the [docs.](https://docs.docker.com/engine/reference/commandline/service_create/)


####Introspection of the Service Name
In this example we send in the service name to the service via environment variables. Using Swarm's built-service discovery, service name introspection allows a service task to resolve to itself or the other tasks within the service.

```
$ docker service create --replicas 2 \
--env 'SERVICE={{.Service.Name}}'  \
--network ov \
alpine sh -c 'ping $SERVICE'

$ docker logs 287
PING stupefied_keller (10.0.4.2): 56 data bytes
64 bytes from 10.0.4.2: seq=0 ttl=64 time=0.041 ms
64 bytes from 10.0.4.2: seq=1 ttl=64 time=0.127 ms
...
```

Swarm backs service names by a distributed VIP that load balances to individual tasks within the service. The VIP `10.0.4.2` will load balance between both replicas within the service. This kind of introspection may be useful for members of clustered applications that need to contact each other on startup.

####Introspection of the Service Task Number
In most respects tasks of a service are idential. Task slots are unique sequential numbers starting from 1 that identify the tasks of a service. Introspection of the task slot can be useful to clustered applications who may need a persistent unique ID. The following service echos its task number and exits. Swarm re-schedules the tasks and we can see that the task slot number persists in the newly re-scheduled task.


```
$ docker service create --replicas 2 --name slot \
--env 'SLOT={{.Task.Slot}}' \
alpine sh -c 'echo $SLOT'

$ docker service logs slot
slot.1.dsi60s6xnswl@moby    | 1
slot.2.sl4vzijjosfw@moby    | 2
slot.1.etzlywzqntl2@moby    | 1
slot.2.nli2neswwn2d@moby    | 2
slot.1.0s8kdiul6rcn@moby    | 1
```


