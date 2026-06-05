
### EC2 Instance

- Instance Name: Jenkins
- AMI: Ubuntu Server 22.04 LTS
- Instance Type: t2.micro
- Key Pair: daro.io
- Public IP: Enabled
- Security Group: allow inbound SSH(22), Jenkins(8080)
- User Data: 
```bash
#!/bin/bash

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl enable jenkins
```
[Jenkins Install Document](https://www.jenkins.io/doc/book/installing/linux/)

<br>![[Pasted image 20231104201931.png]]
Getting "Initial Admin Password"
<br>![[Pasted image 20231104210115.png]]
We input the "Initial Admin Password"
<br>![[Pasted image 20231104210148.png|470]]
We install the suggested plugins
<br>![[Pasted image 20231104210426.png|400]]
Creating our first user
<br>![[Pasted image 20231104211939.png|400]]<br>![[Pasted image 20231104212351.png]] ^80125f

%%Add credentials or ssh key to Jenkins to connect to my GitHub account. 
In our assignments we used public Repos so was not needed%%

### Create credentials

Need to setup **ssh** credentials to be able to connect to salve nodes (EC2s) using **ssh** connection `.pem key`

**Steps:**

1. I navigate to `Manage Jenkins > Credentials > System > Global credentials (unrestricted)`
2. Click button "Add Credentials"
   <br>![[Pasted image 20231105151726.png]]
   
   
1. Select **SSH Username with private key**.
2. Enter a name (ID) for the credentials.
3. In the **Username** field, enter the username of the SSH user `ubuntu` on the slave node.
4. In the **Private Key** field, enter the contents of the `.pem` key file. You can copy and paste the contents of the file, or you can upload the file.
5. Click **Create** to create the credentials.
   <br>![[newcredentials.png]]

> [!done]
>    <br>![[daro.io ubuntu ssh credentials.png]]


### Setting up slaves


Manage Jenkins > Security
Make sure we select Random and save
<br>![[Pasted image 20231108091839.png]]


#### To add agent
Manage Jenkins > Nodes
Click button "New Node"
<br>![[Pasted image 20231105110505.png]]

<br>![[Pasted image 20231108093020.png]]


In the configuration, I focus on the following settings:

- Remote Root Directory: The workspace where Jenkins executes builds.
- Launch Method: 
    - Host: The private IP of the EC2 instance.
    - Credentials: [[daro.io ubuntu ssh credentials.png|previously created]]
    - Host Key Verification Strategy: Set to 'Non-verifying Verification Strategy.'"
<br>![[slavenode.png]]


I have installed Java on 'Slave1' as it is a prerequisite for operation.
<br>![[Pasted image 20231108094344.png]] ^7992da


Now we have an **In sync** `Slave1` Agent

> [!done]
> <br>![[Pasted image 20231108094036.png]]


