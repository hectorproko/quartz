 > [!info] Module 4: Puppet Assignment - 2
> **Tasks To Be Performed:**
> 1. Add one more slave to the setup 
> 2. Add a condition, if Apache is already installed on the slaves, install NGINX The result should be 3rd slave with NGINX installed. 


I spine another EC2 instance `000`
follow the [[Installing Puppet#Agent|same steps previously]] for agents

Back in the master we sign certificate:
<br>![[Pasted image 20231127204406.png]]


I check agent `sudo systemctl status puppet`
<br>![[Pasted image 20231127192945.png]]

In master I create a new module to install nginx like i did with apache


update `site.pp`
```bash
root@ip-10-0-1-249:~# cat /etc/puppetlabs/code/environments/production/manifests/site.pp
node default {
  exec { 'nginx_if':
    command => 'apt-get install nginx -y',
    path    => '/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games',
    unless  => '/bin/which apache2',
  }
}
root@ip-10-0-1-249:~#
```
==maybe change `path`==
> [!NOTE]
> The `path` attribute in a Puppet `exec` resource specifies the list of directories that Puppet will use to search for the command being executed. This is analogous to the `PATH` environment variable in a Unix-like operating system, which the shell uses to locate executables.


In agent3 our latest
```
sudo /opt/puppetlabs/puppet/bin/puppet agent --test
```
<br>![[Pasted image 20231127204529.png]]


<br>![[Pasted image 20231127204700.png]]

We try in agent2
```
sudo /opt/puppetlabs/puppet/bin/puppet agent --test
```
<br>![[Pasted image 20231127205027.png]]
was nto installed in agent2 cuse it already had apche2 not meeting the conditions