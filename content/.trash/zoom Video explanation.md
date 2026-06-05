```bash
7KVjC2!=
https://us02web.zoom.us/rec/play/y_BJeclALkYmHGx-WbJ3Ia3o22PpY33wVg-03bRcCp9bgutsLVDchYPsaAKV_FqEuKgfVf2kW5y2bMqs.eUWf7-4lI4vqjc_k?canPlayFromShare=true&from=share_recording_detail&continueMode=true&componentName=rec-play&originRequestUrl=https%3A%2F%2Fus02web.zoom.us%2Frec%2Fshare%2FLSZS_H-etz7iIpUZ8fnvQnOjcoTluVzERlB6k60Yxt1fHQkMK5S2pVdyOwcEkJYx.ouOOydQclhamPm8a
```

3hours

<br>![[Pasted image 20231130143657.png]]'


Creating instances 3
ubuntu
Project-M < ansible
Project-test
Project-prod

### Project-M
googles how to install ansible
<br>![[Pasted image 20231130152141.png]]
Security Group All traffic

create key pair in M
ssh-keygen

goes to .ssh  to find the contents of the public key id_rasa.pub.pub

In the slaves key paste the contents of the public key id_rasa.pub inside the slave's authorize_keys file

Now master can connect to slaves

adding the IPs of slaves in ansible test and prod, file hosts
<br>![[Pasted image 20231130153457.png]]

testing connections
`ansible -m ping all`

---
Create playbook that installs the require softare

<br>![[Pasted image 20231130154643.png]]

createas scripts
jenkins.io docs, weekly release code,
master.sh
<br>![[Pasted image 20231130154901.png]]

salves, difference is instead of jenkins, docker
<br>![[Pasted image 20231130154946.png]]

<br>![[Pasted image 20231130155404.png]]

`ansible-playbook play.yaml --syntax-check`

---
Add nodes to jenkins

1:31

He adds agents via ssh, /remote root directory: /home/ubuntu/jenkins

createing ssh key by copying contents of .pem

non veryfhing something strategy the same thing we have done before
slave2 we use copy existing node

---
==time 2:08==

Forked repo

Creates Docker file in the repo folder website
<br>![[Pasted image 20231130193329.png]]
<br>![[Pasted image 20231130193348.png]]


Creates new branch

---
Creating Jobs in Jenkins

<br>![[Pasted image 20231130202539.png]]
job1 is for slave1 using develop

Job1
Freestyle Project
Restrict where this project can be run: Slave1 (PRoject test)
Source Code Mangement: Git, URL, uses HTTPS
Branch Scpeifier: Develop


location of docker file, in Project test
<br>![[Pasted image 20231130203505.png]]
<br>![[Pasted image 20231130212925.png]]
This fails if there is no container to delete, so job needs to run at least one without first command

Runs the job check website
<br>![[Pasted image 20231130212618.png]]

He says all jobs are the same only which slave it is running on and what branch

enables webhook
<br>![[Pasted image 20231130213203.png]]

---
Job2

Master branch
adds commands
<br>![[Pasted image 20231130212925.png]]
comments out the first command
Port 83 changed to 82, c1 changed to c2, and job1 to job2

---
Job3
changed to port 80
<br>![[Pasted image 20231130214404.png]]

Complete 
<br>![[godno.gif|200]]