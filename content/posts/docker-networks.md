+++ 
draft = false
date = 2024-04-09T22:02:31+05:00
title = "Docker Networks"
description = "An overview of docker networks"
slug = "docker-networks"
authors = ["Ahmed Nadeem"]
tags = ["docker", "networks"]
categories = []
series = []
+++

Recently, as part of a project, I deployed two backend services that communicated with each other, only to realize a noob mistake I made. In my haste I forgot the simple fact that Docker containers are completely isolated and what I was trying to do required some network configuration as well. This lead me to do some research on Docker networking modes to configure my containers accordingly, and although I was aware of some basic networking modes like bridge and host, I was certainly surprised by some newer network drivers that were available to use in the documentation. This blog post will hopefully provide an overview of all these networking modes, so instead of jumping the ship and choosing Kubernetes directly to solve all your networking woes, you'll find some solace with Docker as well :wink:

## Bridge

![Alternative text for accessibility](/images/avatar.jpeg)

In networking a bridge is a hardware or software device that is responsible for forwarding traffic between network segments. The main advantage of a bridge was to reduce collisions that occur in a network by segmenting it into smaller networks. The bridge network in Docker is similar and it is allow communication between containers within the same network. Bridge networks can also be separated into two types: Default bridge and User-defined. The default network is assigned to containers if no network is specified, this however is considered as a legacy approach. One key difference between them is that user-defined bridge networks allow DNS resolution, while for default networks you can only connect containers using their IP addresses, or by using the ```--link``` option, which is also legacy as per the docs. Moreover, default bridge networks aren't a good practice since there is no isolation if you have multiple containers and they can all access each other on the default network. Another caveat with default networks is that you'll have to restart containers if you want to connect them using ```--link```, while the connecting and disconnecting to user-defined bridge networks is as trivial as running commands like ```docker network connect my-network my-container``` and ```docker network disconnect my-network my-container```

## Host

With the host network mode, your container does not get a separate IP address and no port forwarding is needed, it shares the same network settings as the Docker host. So if your Postgres database is running on port 5432 in a container, you will be able to connect at localhost on the same port and any other application connecting to it will access the it via it's port on localhost. On the surface this seems counter-intuitive since it defeats the purpose of containers, which is isolation, but this is useful in certain situations, for example, if a container needs to handle a large number of ports, or if performance is needed - since there would be no network address translation in this case. **Note: this mode only works on Linux**.

## None

The None network mode completely isolates the container. This is useful if you want to run batch jobs, or any task that does require network calls - I am assuming that's a very small subset.

## IPvlan

IPvlan is a new networking virtualization technique in Docker that provides significant performance gains. Since using bridge networks results in additional network hops when accessing the container, the IPvlan mode bypasses the Linux bridge and connects the container with the NIC directly using a network gateway. So for instance, if we have a network interface on our VM at 192.23.4.40/24, we can create an IPvlan network that assigns IP addresses to containers from this CIDR range directly. By default the gateway IP is the first assignable IP address from the CIDR block, otherwise we can specify the IP while creating the IPvlan network. We can create it using the command:

```
docker network create -d ipvlan \
    --subnet=192.168.30.0/24 \
    --gateway=192.168.30.1 \
    -o parent=eth0.30 \
    -o ipvlan_mode=l2 ipvlan30
```

Now, when we create a container with this network we also don't need to specify any port mappings since the container will have the IP address of a network interface which can be directly accessed from the host.

## Macvlan

In some situations, especially with legacy applications, you expect to be directly connected with a physical network. Macvlan allows you to assign separate MAC addresses to your container's virtual network interface. In the case of IPvlan the MAC addresses are the same.

## Overlay

When you deploy your applications, it is likely that your services reside in separate physical or virtual machines, in which case you'll need overlay network driver to connect your containers. Overlay network mode allows you to connect distributed applications from two different Docker daemons, ideally when the data is encrypted and you're using Docker Swarm to manage your distributed services. However, according to the docs, you could also connect individual containers, which are not part of a Docker Swarm deployment. However, I have never used this mode, if you plan to deploy distributed services, you might as well use Kubernetes since it is built to handle networking at scale.