---
tags:
  - azure
title: Setting Up Docker on a VM and Managing a Web Application Repository
---
<!--
**Project Discovery: Docker Utilization in Azure VMs!** I'm excited to share my latest achievement from the Azure Administrator course, where I successfully installed Docker on a Linux VM and utilized it to manage a web application. The assignment involved creating a VM, installing Docker, pulling the 'hshar/webapp' repository, and making modifications to the Docker image. This exercise provided me with valuable hands-on experience in using Docker within Azure, highlighting the integration of container technology with cloud services. It was an insightful exploration into managing web applications in a containerized environment, enhancing my skills in Azure and Docker.

#Azure #Docker #VirtualMachine #ContainerTechnology #WebApplicationManagement #AzureAdministrator #ProfessionalDevelopment
-->

> [!info] Module 5: Assignment - 1
> **Tasks To Be Performed:** 
> 1. Install a Docker using VM 
> 2. Pull hshar/webapp (https://hub.docker.com/r/hshar/webapp) repository 
> 3. Create a new file in this repository


### Step 1: Install Docker on a Virtual Machine

1. **I Create a Linux VM**:
    
    - Will use the VM from [[Assignment 5_Module4_Azure Administrator Course for AZ-103 AZ-104|Assignment 5: Module 4]] 
      
      
1. **I Install Docker Using `docker.io`**: ^f947ef
    - I run the following commands to update the package index and install Docker from the Ubuntu repositories:
      `sudo apt-get update`
      `sudo apt-get install docker.io`
    - After the installation, I start and enable Docker to ensure it's running and set to start on boot:
      `sudo systemctl start docker `
      `sudo systemctl enable docker`
    - I verify Installation
      `sudo docker --version`
      <br>![[Pasted image 20231207195403.png]]

### Step 2: Pull the `hshar/webapp` Docker Repository

1. **I Pull the Docker Image**:
    - I use the Docker command to pull the `hshar/webapp` image from Docker Hub:
      `sudo docker pull hshar/webapp`
      <br>![[Pasted image 20231207195804.png]]

### Step 3: Create a New File in the Repository

1. **I Run a Container from the Pulled Image**:
    
    - I start a Docker container using the `hshar/webapp` image:
      `sudo docker run -dit --name webapp-container hshar/webapp`
  
2. **I Access the Container's Shell**:
    
    - I access the shell of the running container:
      `sudo docker exec -it webapp-container bash`
  
3. **I Create a New File in the Container**:
    
    - Inside the container, I navigate to the desired directory.
    - I create a new file `touch newfile.txt`  

<br>![[Pasted image 20231210212045.png]]
### Step 4: Commit the Changes to Create a New Docker Image

1. **Commit the Container to a New Image**:
    
    - Use the `docker commit` command to create a new image from the container's current state.
    - The syntax is `docker commit [CONTAINER_ID] [new_image_name]:[tag]`.
2. **Verify the New Image**:
    
    - Use `docker images` to list all images and verify that your new image (`mynewimage:latest`) is there.

```bash
azureuser@VMCustomImage:~$ sudo docker commit f88acb1dcab07b1ae0 hshar/webapp_updated:latest
sha256:5d9250a92e2766f6b591a7ab647888a836ef9da9ea779a098abcce830ee2e284
azureuser@VMCustomImage:~$ sudo docker images
REPOSITORY             TAG       IMAGE ID       CREATED          SIZE
hshar/webapp_updated   latest    5d9250a92e27   8 seconds ago    303MB
<none>                 <none>    853e35c08c74   30 seconds ago   303MB
hshar/webapp           latest    0cbc1f535ed8   4 years ago      303MB
azureuser@VMCustomImage:~$
```

<br>![[Pasted image 20231210212722.png]]

