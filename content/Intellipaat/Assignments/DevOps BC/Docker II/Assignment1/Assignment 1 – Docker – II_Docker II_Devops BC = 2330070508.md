---
tags:
  - docker
title: Managing Apache2 Containers and Creating Volumes
---
<!--
🚀 **Deepening Docker Expertise: Volume Management and Advanced Container Deployment!** I recently accomplished an exciting assignment in my DevOps course focused on Docker volume management. The project involved launching an Apache2 container from a previously created image and creating a Docker volume to manage web content. This exercise allowed me to explore Docker's volume capabilities, enhancing my skills in persistent data management within containers. The task underscored the importance of efficient container storage solutions in maintaining and scaling web applications.

#Docker #DevOps #Containerization #VolumeManagement #DataPersistence #ProfessionalDevelopment
-->

> [!info] Docker Part 2 Assignment - 1
> **Tasks To Be Performed:** 
> 1. Launch the Apache2 container created in previous module 
> 2. Create a Docker volume on /var/www/html 
>  
> Please submit the commands in order to complete this assignment. 


### Step 1:
I will use the Apache2 container that we pushed to DockerHub in [[Case Study 1 – Containerization Using Docker – Part 1_Module 3_Devops BC = 2330070508|Case Study 1 – Containerization Using Docker – Part 1]]
<br>![[my_apache_image2 in dockerhub.png|500]]

### Step 2:


First, I create a volume: `volume-sample`
```bash
docker volume create volume-sample
```

Then, I verify its creation by running:
```
docker volume ls
```

<br>![[Pasted image 20231116130344.png]]

Now, I run a container from the image that was [[my_apache_image2 in dockerhub.png|pushed]] to DockerHub in [[Case Study 1 – Containerization Using Docker – Part 1_Module 3_Devops BC = 2330070508|Case Study 1 – Containerization Using Docker – Part 1]], and I attach the newly created Docker volume named `volume-sample`.



I run a Docker container and attached the newly created volume to it
```bash
docker run -it --mount source="volume-sample",destination="/var/www/html" -p 8080:80 -d hectorproko/my_docker_image:my_apache_image2
```


<!--
Big messup here, see like you attached a wrong volume, that i dont think even existed in your environemnt it was from the lesson
<br>![[Pasted image 20231116155033.png]]
-->
I verify the container works by `curl`ing the page from localhost
<br>![[Pasted image 20231116155018.png]]



