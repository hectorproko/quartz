---
title: Load Balancer Solution with Apache
tags:
  - Linux
  - LoadBalancer
  - "#Apache"
---
*~~(old [Project 8](https://github.com/hectorproko/LOAD-BALANCER-SOLUTION-WITH-APACHE))~~*

> [!Info]
> Project 8: Adding Load Balancer to [**Project 7:** DevOps Tooling Website Solution](https://github.com/hectorproko/Devops-Tooling-Website-Solution/blob/main/Project7_Step.md)  
> 
> In our set up in **Project 7** we had 2 Web Servers and each of them had its own public IP address and public DNS name. We had to access them by using different URLs, which is not a nice user experience to remember addresses/names of even 2 server.
> In order to hide all this complexity and to have a single point of access with a single public IP address/name, a **Load Balancer** can be used. A Load Balancer (LB) distributes clients’ requests among underlying Web Servers and makes sure that the load is distributed in an optimal way.
> 
> Technologies/Tools used:
> * AWS (EC2)
> * Ubuntu
> * Apache (Load Balancer)
> * GitBash

## Configure Apache As A Load Balancer

 1. Create an Ubuntu Server 20.04  
[Creating Ubuntu instance in AWS (**EC2**)](https://github.com/hectorproko/RepeatableSteps_tutorials/blob/main/AWS_Ubuntu_Instnace.md)

2. Open TCP port 80 creating an **Inbound Rule** in Security Group.  
[Open Ports in AWS (**EC2**)](https://github.com/hectorproko/RepeatableSteps_tutorials/blob/main/OpenPortAWS.md)

## Install Apache Load Balancer
``` bash
#Install apache2
sudo apt update
sudo apt install apache2 -y
sudo apt-get install libxml2-dev
```
``` bash
#Enable following modules: a2enmod is an apache thing
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic
```
``` bash
#Restart apache2 service
sudo systemctl restart apache2
Make sure apache2 is up and running
sudo systemctl status apache2
```
## Configure Load Balancing to point traffic to both Web Servers

We need to edit file **000-default.conf** and add a configuration

```bash
sudo vi /etc/apache2/sites-available/000-default.conf
```
My configuration looks like this
```bash
<VirtualHost *:80>
    <Proxy "balancer://mycluster">
        BalancerMember http://172.31.86.103:80 loadfactor=5 timeout=1 
        BalancerMember http://172.31.86.62:80 loadfactor=5 timeout=1
        ProxySet lbmethod=bytraffic
        # ProxySet lbmethod=byrequests
    </Proxy>
    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/

    #ServerAdmin webmaster@localhost
    #DocumentRoot /var/www/html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
The above **Private IPs** belong to my Web Servers
```
http://172.31.86.103:80
http://172.31.86.62:80
```
The balancing method I used is **bytraffic**  which distributes incoming load between both Web Servers according to current traffic load. We can control in which proportion the traffic must be distributed by **loadfactor** parameter.
Traffic will be disctributed **evenly** between them since we set **loadfactor** to the same value for both servers.

Restart **Apache** to re-load configuration
``` bash
sudo systemctl restart apache2
```

To verify that our configuration works we access the **LB** from a browswer using its **Public IP** or **Public DNS name**  

We should see a Red Hat default page since the Load Balancer (Ubuntu Machine) is redirecting the traffic to the Web Servers which are using Read Hat

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LOAD-BALANCER-SOLUTION-WITH-APACHE/main/images/site.png)

To make sure both servers are receiving **HTTP GET requests** from the **Load Balancer** we check the server's log file for new records.

**Note**: In [**Project 7**](https://github.com/hectorproko/Devops-Tooling-Website-Solution/blob/main/Project7_Step.md#prepare-nfs-server) we mounted **/var/log/httpd/** from Web Servers to the **/mnt/logs** on NFS server. For this test we unmount it to make sure that each Web Server has its own log directory.  

Now I'll log in to each Web Server using **SSH** and run the following command which displays the logs live
``` bash 
sudo tail -f /var/log/httpd/access_log
```

The following terminals shows both logs side by side as I refresh the page on the browser

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LOAD-BALANCER-SOLUTION-WITH-APACHE/main/images/lbgetrequest.png)



