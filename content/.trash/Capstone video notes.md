for VM we have 1 subnet and for App Gateway we have different subnet

says to use link <br>![[Pasted image 20231218101707.png]]
the other one is outdated

---time 38---

Creates VM1 east1 
changes to 18 LTS, scripts command
<br>![[Pasted image 20231218102604.png]]

<br>![[Pasted image 20231218102637.png]]

Creates VNet1 as he creates VM1, he add 1 more subnet, subnet_apg1 for app gateway, ports open 80 443 22
<br>![[Pasted image 20231218103017.png]]

Creates VM2, same Vnet2, same subnet

---
VM3
west-us
Ubuntu 18
same port open
Create VNet2
<br>![[Pasted image 20231218104642.png]]

VM4 same thing as VM3


<br>![[Pasted image 20231218105316.png]]<br>![[Pasted image 20231218105356.png]]

---
---time 1 hour ---
Now we need to create storage account where we put our upload data and error page

Creates new storage account
<br>![[Pasted image 20231218110037.png]]

we need to create container named upload
make sure it is NOT private, so Bob

we can upload the page directly in the container but intructions say go through "Static website"
<br>![[Pasted image 20231218110328.png]]

skips and goes to create error file

saves teh error file from repo as a file in his desktop
error.html
<br>![[Pasted image 20231218110806.png]]
Save changes

You get "Primary endpoint" and "Secondary endpoint"

Now upload the error file
<br>![[Pasted image 20231218111046.png]]
gives option to upload

so the file appear in container $web and i think is suppose to be in upload also? doesnt seem like it


with help primary endpoint i can fetch what we have uploaded


---
---time 1:17--
Creating application gateway1 chosing VNet1 automatically choose gateway subnet 
<br>![[Pasted image 20231218115500.png]]
front end gives i name

back end poot, two pools because 2 VMs
<br>![[Pasted image 20231218115656.png]]

rule

<br>![[Pasted image 20231218120501.png]]<br>![[Pasted image 20231218120526.png]]
<br>![[Pasted image 20231218120622.png]]

so pool2 was in top but pool1 was "a path" or target, not what we did in our assigment
<br>![[Pasted image 20231218120841.png]]
<br>![[Pasted image 20231218120724.png]]


<br>![[Pasted image 20231218120946.png]]

==--- time 1:40--==

Creates second application gateway
Vnet2, west US, gateway subnet


Frontends, new public ip
<br>![[Pasted image 20231218124051.png]]


backend
pool3 and pool4, adds VM

Configuraiton, same t hing
<br>![[Pasted image 20231218124154.png]]<br>![[Pasted image 20231218124225.png]]

<br>![[Pasted image 20231218124333.png]]


==--1:49--==

had issue with a VM not  having public IP, he deleted a bunch of IPs, then gives VMs public IPs


Creates gateway 2, did not finishe last time

he run sudo update on all VMs, Putty into each one

second gateway
<br>![[Pasted image 20231218131243.png]]

---
Now he will start running scripts in the VMs

runs script vm2.sh in VM4 and VM2

run script vm1.sh in VM1 and VM3

---
about to edit the file config.py in VM1 and VM3
<br>![[Pasted image 20231218155741.png]]

---
Creates traffic manager

Wehn we connect app gateway to VM we need pools, when we connect Traffic managers to app gateways we need endpoint

we go app gateways, 
<br>![[Pasted image 20231218160328.png]]<br>![[Pasted image 20231218160339.png]]
<br>![[Pasted image 20231218160400.png]]
not sure why he gave it a DNS if he did not use it


Then we go to out traffic manager find "Endpoints"
<br>![[Pasted image 20231218160605.png]]
<br>![[Pasted image 20231218160630.png]]


<br>![[Pasted image 20231218160736.png]]
<br>![[Pasted image 20231218160724.png]]

<br>![[Pasted image 20231218160811.png]]

then we upload something