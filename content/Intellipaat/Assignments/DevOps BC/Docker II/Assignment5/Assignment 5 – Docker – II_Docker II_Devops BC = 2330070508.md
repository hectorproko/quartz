---
tags:
  - docker
title: Creating Overlay Networks and Testing Container Connectivity
---
<!--
🚀 **Advancing in Docker Orchestration: Mastering Overlay Networks and Container Communication!** I've completed an intriguing assignment in my DevOps training, where I focused on Docker's overlay networks. The task involved deploying two containers within an overlay network in a Docker Swarm cluster and ensuring they could communicate with each other. This exercise was pivotal in understanding how overlay networks facilitate seamless container communication across different nodes, enhancing my skills in Docker Swarm management and network orchestration.

#Docker #DevOps #OverlayNetworks #ContainerCommunication #ProfessionalDevelopment
-->

> [!info] Docker Part 2 Assignment - 5
> **Tasks To Be Performed:** 
> 1. Use the previous assignment’s deployment 
> 2. Deploy any 2 containers in a overlay network 
> 3. Try pinging each of the containers from within the containers



### Step 1: Previous Deployment
Using previous [[Assignment 4 – Docker – II_Docker II_Devops BC = 2330070508|Assignment 4 – Docker – II]]'s Deployment

> [!info] Instances Private IPs
> master `10.0.1.199`
> worker1 `10.0.1.176`
> worker2 `10.0.1.38`


### Step 2: Deploy Two Containers in an Overlay Network

In this step, I will deploy two containers within an overlay network in the Docker Swarm cluster. An overlay network enables seamless communication between containers across different nodes in the Swarm. To achieve this, I will use the following commands:
```bash
# Create an overlay network (if not already created)
docker network create --driver overlay my-overlay-network

# Deploying two containers in the overlay network
docker service create --name container-1 --network my-overlay-network alpine ping 8.8.8.8
docker service create --name container-2 --network my-overlay-network alpine ping 8.8.8.8
```

<br>![[Pasted image 20231117145641.png]]


### Step 3: Ping Containers from Within Containers



I find out which node I need to ssh into
```bash
docker service ps <container-1>
```

<br>![[Pasted image 20231117150323.png]]
<br>![[Pasted image 20231117154014.png]]
So the containers are in worker1 `10.0.1.176` and worker2 `10.0.1.38`  


I SSH into worker1  and subsequently log into 'container-1,' enabling me to ping 'container-2'.

> [!success]
> <br>![[Pasted image 20231117153511.png]]

This time, I SSH into worker2, which allows me to log into 'container-2.' From within 'container-2,' I execute a ping command to reach 'container-1'.

> [!success]
> <br>![[Pasted image 20231117154220.png]]


