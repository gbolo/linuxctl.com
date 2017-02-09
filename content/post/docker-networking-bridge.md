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
date: "2016-03-16"
title: "Docker Networking Options - Bridge"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-docker.jpg"

---

By Default, when docker is installed, it will automatically create a bridged network which represents the `docker0` interface which is present in all Docker installations. We can view our docker networks using the `docker network ls` command:
<!--more-->

```
[root@c7-1 ~]# docker network ls
NETWORK ID          NAME                DRIVER
0249796ab0b7        none                null                
47b6faccef28        host                host                
1ecba84224cb        bridge              bridge
```

the Docker daemon connects containers to this network by default. This network also has a subnet associated with it. We can view more details about this network by issuing the `docker network inspect` command:
```
[root@c7-1 ~]# docker network inspect 1ecba84224cb
[
    {
        "Name": "bridge",
        "Id": "1ecba84224cbcc476cc1c1b08f625ac33f9424647664538e545a5c3b1298b08d",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.42.1"
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

[root@c7-1 ~]# ifconfig docker0
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.42.1  netmask 255.255.0.0  broadcast 0.0.0.0
```

We can also notice that the gateway for this network is our `docker0` interface on the host. We can also create additional networks on our host, such as a **bridge network** or an **overlay network**. The problem with using the built in default bridge is that If you want to communicate with container names you must connect the containers via the legacy `docker run --link`. Also we can create pockets of isolation for our containers using multiple of these bridged networks. Let's take a look at creating a new bridged network with our own subnet, which allows us to use some new features that are not present with the default bridge:
```
[root@c7-1 ~]# docker network create --subnet=192.168.77.0/24 --gateway=192.168.77.1 www
749a3fdd6eb9cfae661e4257fde682f4978936acb5696f865bbfe2f5f7868ca0

[root@c7-1 ~]# docker network ls
NETWORK ID          NAME                DRIVER
0249796ab0b7        none                null                
47b6faccef28        host                host                
1ecba84224cb        bridge              bridge              
749a3fdd6eb9        www                 bridge              

[root@c7-1 ~]# docker network inspect 749a3fdd6eb9
[
    {
        "Name": "www",
        "Id": "749a3fdd6eb9cfae661e4257fde682f4978936acb5696f865bbfe2f5f7868ca0",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.77.0/24",
                    "Gateway": "192.168.77.1"
                }
            ]
        },
        "Containers": {},
        "Options": {}
    }
]

[root@c7-1 ~]# ifconfig | grep -iB1 192.168.77.1
br-749a3fdd6eb9: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.77.1  netmask 255.255.255.0  broadcast 0.0.0.0
```

To use this new network, we can add the `--net=www` option when we run our container. We can also use **[Network-scoped alias](https://docs.docker.com/engine/userguide/networking/work-with-networks/#network-scoped-alias)** for name resolution in our containers with the option `--net-alias name`. Let's say I wanted to make 3 containers which talk to each other to deliver my web application such as:

> **nginx --> php-fpm --> mysql**

I can now put these 3 containers in this bridged network and create a separate network for my minecraft container. We also have the flexibility to put the nginx container into multiple bridged networks, since it may need to proxy requests for containers providing different services which are isolated from each other.

```
[root@c7-1 ~]# docker run -itd --name=nginx --net=www --ip=192.168.77.2 --net-alias=nginx busybox
17ab90c3994c635afcd96e128ed2124c21923c202594e5185b6bf0ba4299b80c
[root@c7-1 ~]# docker run -itd --name=php-fpm --net=www --net-alias=php-fpm busybox
de5b86ca5009287e70c66cfb5cca3979f667c39ea2ee1c639f1334a709045a14
[root@c7-1 ~]# docker run -itd --name=mysql --net=www --net-alias=mysql busybox
e4679102aa2afd7a9803374209ab615b3a7c63596c12df2660463edb15a2465f

[root@c7-1 ~]# docker network inspect www
[
    {
        "Name": "www",
        "Id": "749a3fdd6eb9cfae661e4257fde682f4978936acb5696f865bbfe2f5f7868ca0",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.77.0/24",
                    "Gateway": "192.168.77.1"
                }
            ]
        },
        "Containers": {
            "17ab90c3994c635afcd96e128ed2124c21923c202594e5185b6bf0ba4299b80c": {
                "Name": "nginx",
                "EndpointID": "c6f23376071ec35497ddc76a8eb68d0044fa19f542f42ed4644e857762b25496",
                "MacAddress": "02:42:c0:a8:4d:02",
                "IPv4Address": "192.168.77.2/24",
                "IPv6Address": ""
            },
            "de5b86ca5009287e70c66cfb5cca3979f667c39ea2ee1c639f1334a709045a14": {
                "Name": "php-fpm",
                "EndpointID": "c5a6df5c236f5bd070e2d25edcf52a6634a3d4d8e59705deecd19980ecf165e8",
                "MacAddress": "02:42:c0:a8:4d:03",
                "IPv4Address": "192.168.77.3/24",
                "IPv6Address": ""
            },
            "e4679102aa2afd7a9803374209ab615b3a7c63596c12df2660463edb15a2465f": {
                "Name": "mysql",
                "EndpointID": "1dbf62a4d49a7ce356fea6356a0c283002f227927180f1215798425269b63594",
                "MacAddress": "02:42:c0:a8:4d:04",
                "IPv4Address": "192.168.77.4/24",
                "IPv6Address": ""
            }
        },
        "Options": {}
    }
]

```

Each container on this network can reach each other by the name defined in the option `--net-alias=name`. Now lets connect nginx container to another bridge network and spin up another container which is part of it:
```
[root@c7-1 ~]# docker network connect minecraft nginx

[root@c7-1 ~]# docker run -itd --name=minecraft --net=minecraft --net-alias=minecraft busybox
0e48fad03075fd3a39636b66de13f6201a3b4767808b2b62d537eaa567bc2b1a

[root@c7-1 ~]# docker network inspect minecraft
[
    {
        "Name": "minecraft",
        "Id": "72740d4610049bb48782947474fac6d4e25ecfede427ba07c3f1e5077cd48b50",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.78.0/24",
                    "Gateway": "192.168.78.1"
                }
            ]
        },
        "Containers": {
            "0e48fad03075fd3a39636b66de13f6201a3b4767808b2b62d537eaa567bc2b1a": {
                "Name": "minecraft",
                "EndpointID": "e28dfdd943c275a0f95f5a908f80505a5afebf2fabd787610fa67ed5cdbd13c4",
                "MacAddress": "02:42:c0:a8:4e:03",
                "IPv4Address": "192.168.78.3/24",
                "IPv6Address": ""
            },
            "17ab90c3994c635afcd96e128ed2124c21923c202594e5185b6bf0ba4299b80c": {
                "Name": "nginx",
                "EndpointID": "3bb0f6b4962d418b6ffe0844a280b75ccd38a984ea581f14b9effe02ea55e461",
                "MacAddress": "02:42:c0:a8:4e:02",
                "IPv4Address": "192.168.78.2/24",
                "IPv6Address": ""
            }
        },
        "Options": {}
    }
]
```
## Testing Connectivity
Now that we have everything setup, lets experiment with connectivity between containers:
```
[root@c7-1 ~]# docker attach nginx

/ # ping -c1 mysql
PING mysql (192.168.77.4): 56 data bytes
64 bytes from 192.168.77.4: seq=0 ttl=64 time=0.307 ms

--- mysql ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.307/0.307/0.307 ms
/ # ping -c1 php-fpm
PING php-fpm (192.168.77.3): 56 data bytes
64 bytes from 192.168.77.3: seq=0 ttl=64 time=0.134 ms

--- php-fpm ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.134/0.134/0.134 ms
/ # ping -c1 minecraft
PING minecraft (192.168.78.3): 56 data bytes
64 bytes from 192.168.78.3: seq=0 ttl=64 time=0.150 ms

--- minecraft ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.150/0.150/0.150 ms

[root@c7-1 ~]# docker attach minecraft

/ # ping mysql
ping: bad address 'mysql'
/ # ping -c1 192.168.77.4
PING 192.168.77.4 (192.168.77.4): 56 data bytes

--- 192.168.77.4 ping statistics ---
1 packets transmitted, 0 packets received, 100% packet loss

```

As expected, our nginx container can talk to the other 4 because it's in both networks. The other containers can only resolve and ping other containers in their own network. Mission accomplished!
