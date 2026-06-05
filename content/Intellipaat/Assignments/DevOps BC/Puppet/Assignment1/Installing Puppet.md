

> [!quote]- resources:
> [[Hands-On-–-Puppet-Installation 1.pdf]]
> [[Hands-On-–-Using-Variables-In-Manifest.pdf]]
> [[Hands-On-–-Using-Loops-In-Manifest.pdf]]
> [[Hands-On-–-Using-Conditions-In-Manifest.pdf]]
> [[Hands-On-–-Creating-A-Manifest-In-Puppet.pdf]]
> 

Using 2 Ubuntu 20.04, (puppet has a .deb for focal)
https://www.puppet.com/docs/puppet/8/install_puppet#install_puppet

> [!tip] Instances
> puppetmaster `10.0.1.249`
> agent `10.0.1.7` (replaced?)
> agent2 `10.0.1.112`

^60d9d7

[Enable the Puppet platform on Apt](https://www.puppet.com/docs/puppet/8/install_puppet#enable_the_puppet_platform_apt) (both master and agent)
```bash
wget https://apt.puppetlabs.com/puppet8-release-focal.deb 
sudo dpkg -i puppet8-release-focal.deb 
sudo apt update -y
```

Did not have Java installed
Installting 17.0.8
```bash
sudo apt install openjdk-17-jre-headless -y
```
<br>![[Pasted image 20231125200130.png]]

```
sudo apt-get install puppetserver -y
sudo systemctl enable puppetserver
```
So before this works need to solve the [[Assignment 1 – Puppet_Module 4_Devops BC = 2330070508#Issue|issue]]. But we need to install it so the config file appears and we can edit it

I edit file located in `/etc/default/puppetserver`
<br>![[Pasted image 20231123102742.png]]
<br>![[Pasted image 20231123102907.png]]

```bash
sudo systemctl start puppetserver
sudo systemctl status puppetserver
```

<br>![[Pasted image 20231125210610.png]]

Used `find` instead of `which` not sure why did [[Assignment 1 – Puppet_Module 4_Devops BC = 2330070508#^b68c6d|work this time]]
```bash
ubuntu@ip-10-0-1-249:~$ sudo find / -type f puppetserver
find: paths must precede expression: `puppetserver'
ubuntu@ip-10-0-1-249:~$ sudo find / -type f -name puppetserver
/etc/init.d/puppetserver
/etc/default/puppetserver
/opt/puppetlabs/server/apps/puppetserver/bin/puppetserver
```

<br>![[Pasted image 20231125203855.png]]
Maybe not even mention this part
The env variable we create does not registe with sudo need to keep this in mind when we need to run it with sudo
<br>![[Pasted image 20231125204737.png]]

```bash
sudo apt policy puppetserver
```
<br>![[Pasted image 20231125205102.png]]

No need to run `sudo ufw allow 8140/tcp `. I have firewall disabled in OS, need to open in security Group
```
ubuntu@ip-10-0-1-249:~$ sudo ufw status
Status: inactive
```

Need to have DNS for hostname puppet
<br>![[Pasted image 20231125205611.png]]

trying this the command we need to use full path with sudo. As expected there are no certificate request cuse we have not agents yet
```
sudo /opt/puppetlabs/server/apps/puppetserver/bin/puppetserver ca list
```
<br>![[Pasted image 20231125205759.png]]

## Agent

First we add DNS to resovle puppet to master IP. `/etc/hosts`
<br>![[Pasted image 20231125210340.png]]

[Enable the Puppet platform on Apt](https://www.puppet.com/docs/puppet/8/install_puppet#enable_the_puppet_platform_apt)
```
wget https://apt.puppetlabs.com/puppet8-release-focal.deb 
sudo dpkg -i puppet8-release-focal.deb 
sudo apt update -y
```

Install agents
```
sudo apt-get install puppet-agent -y
sudo systemctl enable puppet
sudo systemctl start puppet
sudo systemctl status puppet
```
<br>![[Pasted image 20231125210429.png]]


Will  navigate to the directory of `puppetserver` so when i run it with sudo dont have to specify the full path.
With puppet ca list i see the certifacte requests of my 2 agents
Then we sign the certificate we can sign indidivual certifacates i opted out for `--all`
<br>![[Pasted image 20231125214041.png]]
Here appears old agent1

> [!tip] when calling agent need to use sudo
> ```
> sudo /opt/puppetlabs/puppet/bin/puppet agent --test
> ```

