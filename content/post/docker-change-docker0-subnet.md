---
categories:
  - "administration"
  - "containers"
keywords:
  - "docker"
tags:
  - "docker"
  - "network"
  - "bridge"
date: "2016-02-02"
title: "Docker Networking - Change docker0 Subnet"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-docker.jpg"

---

When you install docker, by default it will create a bridged interface `docker0` with a `172.17.0.0/16` subnet for container networking. It will also create a **MASQUERADE** rule on your **POSTROUTING** iptables chain for container NAT. If this subnet is being used elsewhere on your network, then you should change this default subnet to avoid losing connectivity to these other networks:
<!--more-->

Lets view this chain, stop docker and remove this subnet from your `docker0` interface:

```
[root@arch1 ~]# iptables -nL POSTROUTING -t nat
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0

[root@arch1 ~]# systemctl stop docker
[root@arch1 ~]# ip link set dev docker0 down
[root@arch1 ~]# ip addr del 172.17.0.1/16 dev docker0
```

Now we need to configure a new subnet for `docker0` to use. All we need to do is reconfigure `docker0` then start the docker daemon back up and docker will pick up the change and reconfigure the iptables chain:

```
[root@arch1 ~]# ip addr add 172.17.77.1/24 dev docker0
[root@arch1 ~]# ip link set dev docker0 up
[root@arch1 ~]# systemctl start docker

[root@arch1 ~]# iptables -nL POSTROUTING -t nat
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  172.17.77.0/24       0.0.0.0/0           

[root@arch1 ~]# docker network ls | grep bridge            
f96903f1d9e1        bridge              bridge              

[root@arch1 ~]# docker network inspect f96903f1d9e1
[
    {
        "Name": "bridge",
        "Id": "f96903f1d9e1e278878402b56abae411b0c85f17ab6d3b61c7e56833d623ae3d",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.77.0/24",
                    "Gateway": "172.17.77.1"
                }
            ]
        },
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        }
    }
]
```

## MAKING THE CHANGE PERSISTENT
To make this change **PERMANENT**, we need to change the way we start up the docker deamon by modifying either the init script or systemd unit file and pass the `--bip` option:

```
cat /usr/lib/systemd/system/docker.service | grep -i ExecStart
ExecStart=/usr/bin/docker daemon --bip=172.17.77.1/24 -H fd://
```

### UPDATE - 2016-04-12
Docker now supports the ability to read configuration files from a json file, which by default, is located at `/etc/docker/daemon.json`. So instead of modifying the flags of the initialization of the docker daemon, we can simply create this file with the following content:

```
{
  "bip": "172.17.77.1/24"
}
```
