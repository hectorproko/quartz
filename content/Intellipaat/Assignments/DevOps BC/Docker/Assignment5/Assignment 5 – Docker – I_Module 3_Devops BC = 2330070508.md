---
tags:
  - docker
title: Integrating a Custom HTML File into an Ubuntu-Apache Docker Container
---
<!--
🚀 **Deep Dive into Docker: Creating Custom Web Pages in Containers!** In the latest assignment of my DevOps training, I enhanced my Docker skills by customizing a web service inside a Docker container. The task involved creating a custom HTML page and integrating it into a Dockerized Apache2 web server. This hands-on exercise deepened my understanding of Docker image customization, including crafting Dockerfiles and managing web content within containers. Successfully implementing this showcased my ability to tailor containerized applications for specific needs, a crucial skill in modern web development.

#Docker #DevOps #Containerization #WebDevelopment #ProfessionalDevelopment
-->


> [!info] Module 3: Docker Part 1 Assignment - 5
> **Training Tasks To Be Performed:** 
> 1. Create a sample HTML file 
> 2. Use the Dockerfile from the [[Assignment 4 – Docker – I_Module 3_Devops BC = 2330070508|previous task]] 
> 3. Replace this sample HTML file inside the Docker container with the default page

#### Steps:

🔹 **1. I create a sample HTML file.**

First, I'll create a simple HTML file named `my_page.html`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>My Sample Page</title>
</head>
<body>
    <h1>Welcome to My Sample Docker Page!</h1>
    <p>This is a sample page running inside a Docker container.</p>
</body>
</html>
```

---

🔹 **2. I use the Dockerfile from the previous task.**

I ensure that the Dockerfile from the previous assignment is in the current directory. The Dockerfile content:

```Dockerfile
# Using the Ubuntu base image
FROM ubuntu:latest

# Updating the package list and installing Apache2
RUN apt-get update && apt-get -y install apache2

# Exposing port 80 for Apache2
EXPOSE 80

# Command to run Apache2 in the foreground
CMD ["apachectl", "-D", "FOREGROUND"]
```

---

🔹 **3. I replace the sample HTML file inside the Docker container with the default page.**

To replace the default Apache page with `my_page.html`, I'll need to modify the Dockerfile to copy the sample HTML file into the container at the correct location.

I add the following line to the Dockerfile:
```Dockerfile
COPY my_page.html /var/www/html/index.html
```

The Dockerfile now looks like:
```Dockerfile
# Using the Ubuntu base image
FROM ubuntu:latest

# Updating the package list and installing Apache2
RUN apt-get update && apt-get -y install apache2

# Copying our sample HTML file to replace the default Apache page
COPY my_page.html /var/www/html/index.html

# Exposing port 80 for Apache2
EXPOSE 80

# Command to run Apache2 in the foreground
CMD ["apachectl", "-D", "FOREGROUND"]
```

Now, when I build and run a container from this Dockerfile, it will display my custom HTML page instead of the default Apache page.

---

To check the results, I can:

- Build the Docker image: `docker build -t my_apache_image2 .`
  <br>![[Pasted image 20231031131233.png]]
  
  <br>![[Pasted image 20231031142318.png]]
  
- Run the container: `docker run -d -p 80:80 my_apache_image`
  <br>![[Pasted image 20231031142654.png]]
- Access the browser and navigate to `http://localhost`.

> [!success]
> I see "Welcome to My Sample Docker Page!" displayed.
>   <br>![[Pasted image 20231031142735.png]]



