#test
> [!NOTE] Module 1 Assignment 
> **Problem Statement:** 
> You are assigned a task to install a web server, in order to control multiple worker processes from one master process. After you've done your research you found out that NGINX is the best suitable option in this scenario. Update the host system and install NGINX. Check the version installed.

This will update the list of available packages and their versions, then upgrade the currently installed packages to their latest versions.
```bash
sudo apt update && sudo apt upgrade -y
```
![[Pasted image 20230818230058.png]]

We install NGINX using the package manager:
```bash
sudo apt install nginx -y
```
![[Pasted image 20230818230256.png]]

After installation, starting the NGINX service:
```bash
sudo systemctl start nginx
```
![[Pasted image 20230818230716.png]]

Ensure that NGINX starts automatically at boot:
```bash
sudo systemctl enable nginx
```
![[Pasted image 20230818230412.png]]

Checking version:
```bash
nginx -v
```
![[Pasted image 20230818230510.png]]

Getting the IP of Nginx Server:  
![[Pasted image 20230819001838.png]]

Testing the service:  
![[Pasted image 20230819001617.png]]

