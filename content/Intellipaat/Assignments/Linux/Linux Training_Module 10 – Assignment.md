
> [!NOTE] Module 10 - Assignment
> **Problem Statement:**
> You work for xyz organization. Your job work is to manage Linux-based servers.
> 
> You have been asked to:
> 1. Add a website with its IP address in the /etc/hosts file
> 2. Ping it to check whether it is working fine
> 3. Connect to that server using ftp and transfer a file from your pc to that server
> 4. Install a web server and allow access to the port number of that server


> [!tip] FTP Portion
> For this lab I created an FTP Server at https://sftpcloud.io/tools/free-ftp-server
> ![[Pasted image 20230821214953.png|600]]
> 
> Resolved the hostname `eu-central-1.sftpcloud.io` to IP at https://www.nsookup.io/
> 
> ![[Pasted image 20230821215042.png|600]]
> 


1. **Mapping `5.75.179.162` to `ftp_server`**  
![[Pasted image 20230821215223.png]]
2. **Pinging `ftp_server`**  
![[Pasted image 20230821215628.png]]



3. **Using `5.75.179.162` I'll connect to the FTP server where I'll upload `test` file**  

*Could have used `ftp_server` instead of IP to connect as well*  
![[Pasted image 20230821221659.png]]

![[Pasted image 20230821215441.png]]


4. **Install a web server and allow access to the port number of that server**  

In terminal I run:
```bash
sudo apt update -y #update packages
sudo apt install apache2 -y #install web server apache
```

Checking web server is up:
```bash
sudo systemctl status apache2
```
![[Pasted image 20230822205646.png]]

Getting server IP:  
![[Pasted image 20230822210024.png]]

Enabling firewall
```bash
sudo ufw enable
```
![[Pasted image 20230822205932.png]]

If I check in browser page does not appear  
![[Pasted image 20230822210234.png]]

Opening the necessary ports:
```bash
sudo ufw allow 80,443/tcp
```
![[Pasted image 20230822210318.png]]

When I check again page appears:  
![[Pasted image 20230822210349.png]]

