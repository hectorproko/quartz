---
tags:
  - docker
title: Building a Docker Swarm Cluster with Apache Container Replication
---
<!--
🚀 **Enhanced Docker Deployment: Creating a Docker Swarm Cluster!** I've completed an advanced assignment in my DevOps training focused on Docker Swarm. This involved setting up a Docker Swarm cluster with three nodes, deploying an Apache container with four replicas across the cluster, and managing network and security configurations. This project provided me with hands-on experience in orchestrating multi-container deployments using Docker Swarm, reinforcing my understanding of container orchestration and load balancing in a distributed environment.

#Docker #DevOps #DockerSwarm #ContainerOrchestration #ProfessionalDevelopment
-->

> [!info] Docker Part 2 Assignment - 4
> **Tasks To Be Performed:**
> 1. Create a Docker swarm cluster with 3 nodes 
> 2. Deploy an Apache container with 4 replicas


### Step 1: Creating Security Group 

1. **Swarm Management Ports:**
    - TCP port 2377: This port facilitates communication between manager nodes and worker nodes. It's essential for all nodes in the Swarm cluster.
      
2. **Overlay Network Ports (if used):**
    - UDP and TCP ports 7946: These ports are utilized for control plane communication among nodes in the overlay network.
    - UDP port 4789: This port is employed for transporting overlay network data traffic. It's specifically used when using overlay networks with Docker services.
      
3. **Service Published Ports:**
    - In my Swarm setup, I have a web service running so I need to open the specific port configured for this service `80`.

<br>![[dockerswarm security group.png]]
SSH access is set to allow connections from any source (`0.0.0.0`), for remote access to perform the activities I'm documenting. Additionally, to enhance security, all ports related to Docker are confined within the VPC CIDR range (`10.0.0.0/16`). Furthermore, HTTP connections are permitted from any source (`0.0.0.0`).

### Step 2: EC2 Instances

I will create three EC2 instances, each running Ubuntu 22.04, and configure them to use the specified security group. Additionally, I will enable public IP addresses for these instances to facilitate SSH access.  


Installing Docker in all instances:
```bash
#!/bin/bash
sudo apt-get update -y
sudo apt-get install docker.io -y
```

Documenting Docker version:
```bash
Docker version 24.0.5, build 24.0.5-0ubuntu1~22.04.1
```


> [!info] Instances Private IPs
> master `10.0.1.199`
> worker1 `10.0.1.176`
> worker2 `10.0.1.38`



On the **master** node, I initiate the Docker Swarm with the following command:
```bash
docker swarm init --advertise-addr=10.0.1.199
```
*This command initializes the Swarm and designates `10.0.1.199` as the advertised address, which is the address that other nodes in the Swarm will use to connect to the master node for management and coordination.*

<br>![[Pasted image 20231117120103.png]]
*Additionally, it outputs a Docker Swarm join token, which is used by worker nodes to join the Swarm.*




In worker1 and worker2, I use the token by running the `docker swarm join` command to join them to the cluster.
```bash
docker swarm join --token SWMTKN-1-57n206fb6tainmxdd8okzegjn70afvoyomi32o4dntg9bpqixo-761de69y7sbxrn1j2epll6ilf 10.0.1.199:2377
```


I verify the nodes in the swarm by using the `docker node ls` command.
<br>![[Pasted image 20231117120824.png]]



### Step 3: Deploying Apache Container with 4 Replicas
Now that I have set up a Docker Swarm cluster, I can deploy an Apache container with 4 replicas. On the master node, I will create a `docker-compose.yml` file with the following content:

```yaml
version: '3.8'

services:
  apache:
    image: httpd:latest
    deploy:
      replicas: 4
    ports:
      - "80:80"
```

I deploy the service:
```bash
docker stack deploy -c docker-compose.yml my-apache-stack
```

This configuration will create 4 replicas of the Apache container and distribute them across the Swarm nodes.
<br>![[Pasted image 20231117121847.png]]


I check the container distribution:
```bash
docker service ps my-apache-stack_apache
```
<br>![[Pasted image 20231117122941.png]]

To verify that the Apache page is up and running, I will use the public IP addresses of worker2 and the master node and access it through a web browser.

> [!success]
> <br>![[Pasted image 20231117123423.png]]
> <br>![[Pasted image 20231117123438.png]]
