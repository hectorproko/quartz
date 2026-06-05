---
title: Configure Nginx as a Load Balancer with TLS
tags:
  - LoadBalancer
  - TLS
  - Nginx
  - Linux
---
*~~(old [Project 10](https://github.com/hectorproko/LOAD-BALANCER-SOLUTION-WITH-NGINX-AND-SSL-TLS)  )~~*

> [!info]
> PROJECT 10: LOAD BALANCER SOLUTION WITH NGINX AND SSL/TLS
> 
> To know different alternative solutions for the same problem. We will implement a **Nginx Load Balancing** Web Solution with secured HTTPS connection with periodically updated SSL/TLS certificates.
> 
> In this project we will register our website with LetsEnrcypt Certificate Authority, to automate certificate issuance and use a shell client recommended by LetsEncrypt – certbot.
> 
> ![[darey.io/NGINX LOAD-BALANCER with TLS/images/architecture.png]]
> 

## Provision the environment
 1. Create an Ubuntu Server 20.04  
[Creating Ubuntu instance in AWS (**EC2**)](https://github.com/hectorproko/RepeatableSteps_tutorials/blob/main/AWS_Ubuntu_Instnace.md)

2. Open TCP port **80** and **443** creating an **Inbound Rule** in Security Group.  
[Open Ports in AWS (**EC2**)](https://github.com/hectorproko/RepeatableSteps_tutorials/blob/main/OpenPortAWS.md)


## Configure Nginx as a Load Balancer
``` bash
#Install Nginx
sudo apt update
sudo apt install nginx
```
Will update **/etc/hosts** file for local DNS with Web Servers’ names **Web1**, **Web2** and their **local IP addresses**. This is my hosts file

``` bash
ubuntu@ip-172-31-91-30:~$ sudo vi /etc/hosts
ubuntu@ip-172-31-91-30:~$ cat /etc/hosts
    127.0.0.1 localhost
    172.31.87.158 Web1 #<< Added these two lines
    172.31.81.144 Web2 #<<
    ...
```


We configure **LB** by updating config file **nginx.conf** 
```bash
sudo vi /etc/nginx/nginx.conf
```
Make sure to comment-out line **include /etc/nginx/sites-enabled/*;**
``` bash
#include /etc/nginx/sites-enabled/*;
```

Sections **upstream** and **server** should look like this
``` bash
upstream myproject {
    server Web1 weight=5; #using Local DNS names Web1,2 instead of IPs
    server Web2 weight=5;
  }
server {
    listen 80;
    server_name www.domain.com;
    location / {
      proxy_pass http://myproject;
    }
  }

}
```

## Register a new domain name and configure secured connection using SSL/TLS certificates
Let us make necessary configurations to make connections to our Tooling Web Solution secured!
In order to get a valid SSL certificate – you need to register a new domain name. I registered hectorsdomainforproject.de

Assign an Elastic IP to your Nginx LB server and associate your domain name with this Elastic IP. To avoid new public ip after restart
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LOAD-BALANCER-SOLUTION-WITH-NGINX-AND-SSL-TLS/main/images/elasticip.png)  

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LOAD-BALANCER-SOLUTION-WITH-NGINX-AND-SSL-TLS/main/images/elasticip2.png)  

Update A record in your registrar to point to Nginx LB using Elastic IP address
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LOAD-BALANCER-SOLUTION-WITH-NGINX-AND-SSL-TLS/main/images/mydomains.png)  

Configure Nginx to recognize your new domain name
Update your nginx.conf with server_name www.<your-domain-name.com> instead of server_name www.domain.com. When requesting


```bash
server {
    server_name www.hectorsdomainforproject.de;
    location / {
      proxy_pass http://myproject;
    }
```
Make sure [snapd](https://snapcraft.io/snapd) service is active and running.
```
sudo systemctl status snapd
```

Install **certbot** and request for an **SSL/TLS** certificate

``` perl
ubuntu@ip-172-31-91-30:~$ sudo snap install --classic certbot
certbot 1.20.0 from Certbot Project (certbot-eff✓) installed
ubuntu@ip-172-31-91-30:~$
```

#### Request your certificate [certbot instructions](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal) 

``` bash
sudo ln -s /snap/bin/certbot /usr/bin/certbot  #created symbolic link
sudo certbot --nginx #get a certificate and have Certbot edit your nginx configuration automatically to serve it, turning on HTTPS access in a single step.
```

``` bash
ubuntu@ip-172-31-91-30:/etc/nginx$ sudo certbot --nginx
Saving debug log to /var/log/letsencrypt/letsencrypt.log

Which names would you like to activate HTTPS for?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: www.hectorsdomainforproject.de
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel):
Requesting a certificate for www.hectorsdomainforproject.de

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/www.hectorsdomainforproject.de/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/www.hectorsdomainforproject.de/privkey.pem
This certificate expires on 2022-01-16.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for www.hectorsdomainforproject.de to /etc/nginx/nginx.conf
Congratulations! You have successfully enabled HTTPS on https://www.hectorsdomainforproject.de

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
ubuntu@ip-172-31-91-30:/etc/nginx$
```


Testing secure access to the Web Solution by trying to reach  
*https:/hectorsdomainforproject.de* 

We can see a lock icon in the browser’s search string.  

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LOAD-BALANCER-SOLUTION-WITH-NGINX-AND-SSL-TLS/main/images/site.png)  
By clicking on the lock icon we see the details of the certificate issued for the website.

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/LOAD-BALANCER-SOLUTION-WITH-NGINX-AND-SSL-TLS/main/images/certificate.png)    

By default, **LetsEncrypt** certificate is valid for 90 days, so it is recommended to renew it at least every 60 days or more frequently.  

You can test renewal command in **dry-run mode**
``` bash
ubuntu@ip-172-31-91-30:/etc/nginx$ sudo certbot renew --dry-run
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/www.hectorsdomainforproject.de.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Account registered.
Simulating renewal of an existing certificate for www.hectorsdomainforproject.de

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations, all simulated renewals succeeded:
  /etc/letsencrypt/live/www.hectorsdomainforproject.de/fullchain.pem (success)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
ubuntu@ip-172-31-91-30:/etc/nginx$
```

Best practice is to have a scheduled job that runs **renew command** periodically. We can configure a **cronjob** to run the command twice a day.
To do so, lets edit the crontab file with the following command:
``` 
crontab -e
```
we add the following line:  
``` bash
* */12 * * *   root /usr/bin/certbot renew > /dev/null 2>&1 
```
