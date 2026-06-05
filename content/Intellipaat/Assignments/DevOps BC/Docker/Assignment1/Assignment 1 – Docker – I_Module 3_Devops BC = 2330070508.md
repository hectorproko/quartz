---
tags:
  - docker
title: Deploying an Ubuntu Container with Apache2 and Port Mapping
---
<!--
🚀 **Exploring Docker: Practical Assignment in Container Management!** I've recently accomplished an assignment in my DevOps training, centered on Docker. The task involved pulling an Ubuntu container, running it while mapping port 80, installing Apache2, and ensuring its accessibility via a browser. This hands-on experience was crucial in mastering Docker container management and deploying web services within a containerized environment. It demonstrated my ability to navigate Docker's capabilities, from container setup to web server installation, reinforcing my skills in modern DevOps practices.

#Docker #DevOps #Containerization #Apache2 #CloudComputing #ProfessionalDevelopment
-->
> [!info] Module 3: Docker Part 1 Assignment - 1
> **Training Tasks To Be Performed:** 
> 1. Pull Ubuntu container 
> 2. Run this container and map port 80 on the local 
> 3. Install Apache2 on this container 
> 4. Check if you are able to access the Apache page on your browser 
 

> [!NOTE] Prerequisite:
> Before I begin with the Docker tasks, I've ensured that:
> - I have Docker Desktop installed on my Windows machine.
> - I'll be running all the Docker commands in cmd.


1. I pull the Ubuntu container.
```
docker pull ubuntu
```

2. I run the Ubuntu container and map port 80 on my local machine to port 80 on the container.

> **Note to myself:** Before mapping port 80, I ensure that no other services are using it on my host machine to avoid conflicts. If there are issues, I'll address those first.
   
```
docker run -it -p 80:80 ubuntu
```
<br>![[Pasted image 20231030233016.png]]

==Once executed, I am inside the container's shell.==

3. I install Apache2 on this container.
First, I update the package lists:
```
apt update -y --fix-missing
```

Then, I install the Apache2 package:
```
apt install apache2 -y
```

---

4. I start the Apache2 service inside the container.
```
service apache2 start
```

To ensure Apache2 is actively running, I check its status:
```
service apache2 status
```
<br>![[Pasted image 20231030233321.png]]

---

5. I open my web browser to access the Apache2 default page.**

> [!success]
> By navigating to `http://localhost`, I  see the Apache2 Ubuntu default page.
> <br>![[Pasted image 20231030233510.png]]
> 

---

