---
tags:
  - docker
title: Implementing Dynamic HTML File Replacement with Apache2 Container
---
<!--
🚀 **Enhancing Docker Skills: Bind Mounts and Dynamic Web Content!** I've recently tackled an interesting assignment in my DevOps course, focusing on Docker bind mounts. The task was to create a bind mount on an Apache2 container for dynamic HTML file updates. This involved running a container with a bind mount linked to a local directory, enabling real-time web content modifications. This exercise was pivotal in understanding the flexibility of Docker in web development scenarios, showcasing my ability to adapt web content dynamically in a containerized environment.

#Docker #DevOps #BindMounts #WebDevelopment #ProfessionalDevelopment
-->

> [!info] Docker Part 2 Assignment - 2
> **Tasks To Be Performed:** 
> 1. Use the Apache2 container created in previous module 
> 2. Create a bind mount on `/var/www/html` to replace html files dynamically 
> 
> Submit the commands to complete the assignment.

### Step 1: 
Using container `ID: cbda67fd188b` from [[Assignment 1 – Docker – II_Docker II_Devops BC = 2330070508|Assignment 1 – Docker – II]] 

Removing container
```
docker rm -f cbda67fd188b
```

This command recreates a container with bind mounts, linking specific source and target paths from the local host to the container.
```bash
docker run -it --mount type=bind,source=/c/Users/Hecti/Desktop/docker_vol,target=/var/www/html -p 8080:80 -d hectorproko/my_docker_image:my_apache_image2
```
<br>![[Pasted image 20231116160144.png]]


### Step 2:
Created file `/index.html` inside folder nano `docker_vol`
`nano docker_vol/index.html`
```html
<!DOCTYPE html>
<html>
<head>
<title>Page Title</title>
</head>
<body>

<h1>Created in Bind Mount</h1>


</body>
</html>
```


Restarted the container
```bash
docker restart 0f3ceee8c91693
```

Verifying the the `index.html` is being hosted by accessing the container/Apache Server 

> [!success]
> <br>![[Pasted image 20231116160825.png]]