# First Attempt 



Using Ubuntu 20, (puppet has a .deb for focal)
https://www.puppet.com/docs/puppet/8/install_puppet#install_puppet


```
wget https://apt.puppetlabs.com/puppet8-release-focal.deb 
sudo dpkg -i puppet8-release-focal.deb 
sudo apt update -y
```

## Installing Puppet Server
https://www.puppet.com/docs/puppet/8/server/install_from_packages.html

Did not have Java installed
Installting 17.0.8
```
sudo apt install openjdk-17-jre-headless -y
```
<br>![[Pasted image 20231123092900.png]]

```
sudo apt-get install puppetserver -y
sudo systemctl enable puppetserver
sudo systemctl start puppetserver
sudo systemctl status puppetserver
```
So before this works need to solve the [[Assignment 1 – Puppet_Module 4_Devops BC = 2330070508#Issue|issue]]. But we need to install it so the config file appears and we can edit it

Getting the version
If im going to add the path variable now is the time
```
ubuntu@ip-10-0-1-220:~$ which puppetserver
/opt/puppetlabs/bin/puppetserver
ubuntu@ip-10-0-1-220:~$ /opt/puppetlabs/bin/puppetserver --version
puppetserver version: 8.3.0
ubuntu@ip-10-0-1-220:~$
```

^b68c6d

==added the variable path `.bashrc` above==

**Edit the Initialization File**:
Open your shell's profile script in a text editor. For most users on Linux using the Bash shell, this will be `~/.bashrc`:
```
nano ~/.bashrc
```

**Add the Export Command**:
Add the line to the end of the file:
```
export PATH="$PATH:/opt/puppetlabs/server/apps/puppetserver/bin"
```

**Save and Close the File**:
Save the changes and exit the editor.

**Apply the Changes**:
To apply the changes immediately, source the profile script:
```
source ~/.bashrc
```

- Alternatively, the changes will apply the next time you log in or open a new terminal session.
verify
<br>![[Pasted image 20231123121236.png]]
<br>![[Pasted image 20231123121929.png]]



---

```bash
ubuntu@ip-10-0-1-220:~$ sudo apt policy puppetserver
puppetserver:
  Installed: 8.3.0-1focal
  Candidate: 8.3.0-1focal
  Version table:
 *** 8.3.0-1focal 500
        500 http://apt.puppet.com focal/puppet8 amd64 Packages
        500 http://apt.puppet.com focal/puppet8 all Packages
        100 /var/lib/dpkg/status
     8.2.3-1focal 500
        500 http://apt.puppet.com focal/puppet8 amd64 Packages
        500 http://apt.puppet.com focal/puppet8 all Packages
     8.2.1-1focal 500
        500 http://apt.puppet.com focal/puppet8 amd64 Packages
        500 http://apt.puppet.com focal/puppet8 all Packages
     8.2.0-1focal 500
        500 http://apt.puppet.com focal/puppet8 amd64 Packages
        500 http://apt.puppet.com focal/puppet8 all Packages
     8.1.0-1focal 500
        500 http://apt.puppet.com focal/puppet8 amd64 Packages
        500 http://apt.puppet.com focal/puppet8 all Packages
     8.0.0-1focal 500
        500 http://apt.puppet.com focal/puppet8 amd64 Packages
        500 http://apt.puppet.com focal/puppet8 all Packages
ubuntu@ip-10-0-1-220:~$

```


No need to run `sudo ufw allow 8140/tcp `. I have firewall disabled in OS, need to open in security Group
```
ubuntu@ip-10-0-1-220:~$ sudo ufw status
Status: inactive
```

--- 
==This was not needed==
[CA subcommands](https://www.puppet.com/docs/puppet/8/puppet_server_ca_cli#puppet_server_ca_cli-ca-subcommands) need to be installed ==LEFT HERE==

```
sudo /opt/puppetlabs/puppet/bin/gem install -i /opt/puppetlabs/puppet/lib/ruby/vendor_gems puppetserver-ca
```


```
puppetserver ca list
```


## Agent
[Enable the Puppet platform on Apt](https://www.puppet.com/docs/puppet/8/install_puppet#enable_the_puppet_platform_apt)
```
wget https://apt.puppetlabs.com/puppet8-release-focal.deb 
sudo dpkg -i puppet8-release-focal.deb 
sudo apt update -y
```

Install agents
```
sudo apt-get install puppet-agent
sudo systemctl start puppet
sudo systemctl status puppet
sudo systemctl enable puppet
```

Finding version
```bash
ubuntu@ip-10-0-1-165:~$ ls /opt/puppetlabs/bin/
facter  puppet  pxp-agent
ubuntu@ip-10-0-1-165:~$ /opt/puppetlabs/bin/puppet --version
8.3.1
ubuntu@ip-10-0-1-165:~$ /opt/puppetlabs/bin/pxp-agent --version
1.15.18
ubuntu@ip-10-0-1-165:~$
```


Need to edit `/etc/hosts` in agent to find the master node
We dont have DNS running to resolved hostnames, ==on both== 

I think we see the issue here of agent looking for hostname puppet
<br>![[Pasted image 20231123115057.png]]
Make the entry, choose any
<br>![[Pasted image 20231123115730.png]]
<br>![[Pasted image 20231123115605.png]]

Restart the service `sudo systemctl restart puppet`
Now I can see connection error is gone
<br>![[Pasted image 20231123120052.png]]



#### Issue:
Staring puppet server
<br>![[Pasted image 20231123094711.png]]
moving from t2.micro to t2.small, i think is the JAVA_ARGS
<br>![[Pasted image 20231123102742.png]]
<br>![[Pasted image 20231123102907.png]]

File located in `/etc/default/puppetserver`

---
Issue:
```
ubuntu@ip-10-0-1-220:~$ puppetserver ca list
/opt/puppetlabs/puppet/lib/ruby/vendor_gems/gems/puppetserver-ca-2.6.0/lib/puppetserver/ca/utils/http_client.rb:168:in `add_file': X509_LOOKUP_load_file: system lib (OpenSSL::X509::StoreError)
        from /opt/puppetlabs/puppet/lib/ruby/vendor_gems/gems/puppetserver-ca-2.6.0/lib/puppetserver/ca/utils/http_client.rb:168:in `make_store'
        from /opt/puppetlabs/puppet/lib/ruby/vendor_gems/gems/puppetserver-ca-2.6.0/lib/puppetserver/ca/utils/http_client.rb:20:in `initialize'
        from /opt/puppetlabs/puppet/lib/ruby/vendor_gems/gems/puppetserver-ca-2.6.0/lib/puppetserver/ca/certificate_authority.rb:26:in `new'
        from /opt/puppetlabs/puppet/lib/ruby/vendor_gems/gems/puppetserver-ca-2.6.0/lib/puppetserver/ca/certificate_authority.rb:26:in `initialize'
        from /opt/puppetlabs/puppet/lib/ruby/vendor_gems/gems/puppetserver-ca-2.6.0/lib/puppetserver/ca/action/list.rb:226:in `new'
        from /opt/puppetlabs/puppet/lib/ruby/vendor_gems/gems/puppetserver-ca-2.6.0/lib/puppetserver/ca/action/list.rb:226:in `get_certs_or_csrs'
        from /opt/puppetlabs/puppet/lib/ruby/vendor_gems/gems/puppetserver-ca-2.6.0/lib/puppetserver/ca/action/list.rb:103:in `run'
        from /opt/puppetlabs/puppet/lib/ruby/vendor_gems/gems/puppetserver-ca-2.6.0/lib/puppetserver/ca/cli.rb:102:in `run'
        from /opt/puppetlabs/server/apps/puppetserver/cli/apps/ca:5:in `<main>'
```


After ruing with and without sudo `openssl x509 -in /etc/puppetlabs/puppet/ssl/ca/ca_crt.pem -text -noout
`
wonder if its a permission issue


1. **Check if the CA certificate file exists:** Verify that the file `/etc/puppetlabs/puppet/ssl/ca/ca.pem` exists. If it does not exist, you will need to restore the file from a backup or reinstall Puppet.
    
2. **Check the permissions of the CA certificate file:** Ensure that the file `/etc/puppetlabs/puppet/ssl/ca/ca.pem` has the correct permissions. The file should be readable by the user running Puppet and not writable by anyone else.
    ==Not in this locaiton==
    Moving from `/etc/puppetlabs/puppet/ssl/certs/ca.pem`
    to `/etc/puppetlabs/puppet/ssl/ca/`
    ==DID not work==
    

> [!success]
> Seems like was  indeed a permission thing
> <br>![[Pasted image 20231125192357.png]]
> but i cant use sudo puppetserver, need to use the full path
> 


1. **Check if the CA certificate file is in the expected location:** If you have moved the CA certificate file to a different location, you will need to update the Puppet configuration to point to the new location.
   `ca_cert = "/etc/puppetlabs/puppet/ssl/certs/ca.pem"`
   Teh puppet configuraiton file puppet.conf did not have an entry pointing to the ca.pem adding it 
   ==Did nto work==
    
4. **Check if the CA certificate file is corrupt:** If you suspect that the CA certificate file is corrupt, you can try to regenerate it. This will require you to delete the existing CA certificate file and then restart the Puppet server.Second Attempt

# Second Attempt
Following [[Hands-On-–-Puppet-Installation.pdf]]

Using Ubuntu 20, (puppet has a .deb for focal)  
[https://www.puppet.com/docs/puppet/8/install_puppet#install_puppet](https://www.puppet.com/docs/puppet/8/install_puppet#install_puppet)


> [!tip] Instances
> Puppet Master `10.0.1.154`
> Node1 `10.0.1.52`
> 

---
 **Installing Puppet Master**

```bash
sudo apt-get update -y
sudo apt-get install wget -y
wget https://apt.puppetlabs.com/puppet8-release-focal.deb
sudo dpkg -i puppet8-release-focal.deb
sudo apt-get update -y
```

```bash
ubuntu@ip-10-0-1-154:~$ apt policy puppet master
puppet:
  Installed: 5.5.10-4ubuntu3
  Candidate: 5.5.10-4ubuntu3
  Version table:
 *** 5.5.10-4ubuntu3 500
        500 http://us-east-1.ec2.archive.ubuntu.com/ubuntu focal/universe amd64 Packages
        100 /var/lib/dpkg/status
N: Unable to locate package master

```

```bash
sudo apt-get install puppet-master
```

**Installing Puppet Agent**
*Same as above but at the end we install `puppet` instead of `puppet-master`* 


```bash
sudo apt-get update -y
sudo apt-get install wget -y
wget https://apt.puppetlabs.com/puppet8-release-focal.deb
sudo dpkg -i puppet8-release-focal.deb
sudo apt-get update -y
```

```bash
ubuntu@ip-10-0-1-52:~$ apt policy puppet master
puppet:
  Installed: (none)
  Candidate: 5.5.10-4ubuntu3
  Version table:
     5.5.10-4ubuntu3 500
        500 http://us-east-1.ec2.archive.ubuntu.com/ubuntu focal/universe amd64 Packages
N: Unable to locate package master
```

```bash
sudo apt-get install puppet
```

**Configuring Puppet Master**
 Step1: Add the following lines in the puppet-master configuration file:

Added `JAVA_ARGS="-Xms512 Xmx512m"`

```bash
sudo vi /etc/default/puppet-master
sudo systemctl restart puppet-master
```

<br>![[Pasted image 20231125112944.png]]
<br>![[Pasted image 20231125113102.png]]
Step 2: Next open port 8140 on the Puppet Master’s firewall

```bash
ubuntu@ip-10-0-1-154:~$ sudo ufw allow 8140/tcp
Rules updated
Rules updated (v6)
ubuntu@ip-10-0-1-154:~$
```

```
ubuntu@ip-10-0-1-154:~$ sudo ufw status
Status: inactive
```


```bash
sudo systemctl restart puppet-master
```


Step 3: Make changes to the hosts file which exists in /etc/hosts. And add the Puppet Master IP address along with the name “puppet” on all nodes

Master and slave:
<br>![[Pasted image 20231125124608.png]]

Step 4: Create the following directory path:
In master:
```bash
sudo mkdir -p /etc/puppet/code/environments/production/manifests
```


**Configuring Puppet Slave:** 
Step 1: Add the entry for Puppet Master in /etc/hosts
<br>![[Pasted image 20231125124608.png]]

Step 2: Finally start the Puppet agent by using the following command. Also, enable the service, so that it starts when the computer starts.

```bash
sudo systemctl start puppet
sudo systemctl enable puppet
```

**On Master:** 
Step 1: Type the following command


> [!attention]
> We stop because in the new version of puppet in the newest version  master and slave portions are separated from package puppet, now we have puppet-server
