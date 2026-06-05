 > [!info] Module 4: Puppet Assignment - 1
> **Tasks To Be Performed:** 
> 1. Setup Puppet master-slave using 3 nodes 
> 2. Installing Apache on slaves using manifests 

---
### Step 1:
[[Installing Puppet]]

---
### Step 2:

```yaml
class apache {
  package {
    'httpd':
      ensure => installed,
      provider => (osfamily == 'RedHat' ? 'yum' : 'apt'),
  }

  service {
    'httpd':
      ensure => running,
      enable => true,
      name   => 'httpd',
  }

  file {
    '/var/www/html/index.html':
      ensure => present,
      content => template('apache/index.html.erb'),
      mode    => '0644',
      owner   => 'root',
      group   => 'root',
      notify => Service['httpd'],
  }
}
```

> [!NOTE]- explanation
> The Puppet code snippet you provided defines a basic Puppet class for managing the Apache HTTP Server (commonly referred to as `httpd`). This class ensures that the Apache package is installed and that the Apache service is running and enabled to start on boot. Let's go through the code to understand its structure and functionality:
> 
> 1. **Class Declaration**:
>     
>     puppetCopy code
>     
>     `class apache {`
>     
>     This line begins the definition of a Puppet class named `apache`. Classes in Puppet are collections of resources that are managed as a single unit.
>     
> 2. **Package Resource**:
>     
>     puppetCopy code
>     
>     `package { 'httpd':   ensure   => installed,   provider => (osfamily == 'RedHat' ? 'yum' : 'apt'), }`
>     
>     This block defines a `package` resource to manage the Apache package (`httpd`):
>     
>     - `ensure => installed` ensures that the package is installed. If it's not installed, Puppet will install it.
>     - `provider => (osfamily == 'RedHat' ? 'yum' : 'apt')` sets the package provider based on the operating system family. If the `osfamily` fact equals `'RedHat'`, it uses `yum` as the provider; otherwise, it defaults to `apt`. This is a simple way to support both RedHat-based (like CentOS) and Debian-based (like Ubuntu) systems.
> 3. **Service Resource**:
>     
>     puppetCopy code
>     
>     `service { 'httpd':   ensure => running,   enable => true,   name   => 'httpd', }`
>     
>     This block manages the Apache service:
>     
>     - `ensure => running` ensures that the Apache service is running. If it's not running, Puppet will start it.
>     - `enable => true` ensures that the Apache service is enabled to start on boot.
>     - `name => 'httpd'` explicitly sets the name of the service to manage as `httpd`.

I think the path was alredy there
```
sudo mkdir -p /etc/puppetlabs/code/environments/production/manifests
```

> [!tip] path was already there, after install
> ```bash
> ubuntu@ip-10-0-1-244:~$ sudo tree /etc/puppetlabs/code/
> /etc/puppetlabs/code/
> ├── environments
> │   └── production
> │       ├── data
> │       ├── environment.conf
> │       ├── hiera.yaml
> │       ├── manifests
> │       └── modules
> └── modules
> 
> 6 directories, 2 files
> ```


Created `site.pp` inside manifests, 

Needed to add hostnames in `etc/hosts` for the agents

What I have so far
```bash
ubuntu@ip-10-0-1-249:/etc/puppetlabs/code/environments$ tree
.
└── production
    ├── data
    ├── environment.conf
    ├── hiera.yaml
    ├── manifests
    │   └── site.pp
    └── modules
        └── apache
            ├── apache.pp
            └── templates
                └── index.html.erb

6 directories, 5 files
ubuntu@ip-10-0-1-249:/etc/puppetlabs/code/environments$ cat production/manifests/site.pp
node 'agent' {
  include apache
}

node 'agent2' {
  include apache
}
ubuntu@ip-10-0-1-249:/etc/puppetlabs/code/environments$ cat production/modules/apache/apache.pp
class apache {
  package {
    'httpd':
      ensure => installed,
      provider => (osfamily == 'RedHat' ? 'yum' : 'apt'),
  }

  service {
    'httpd':
      ensure => running,
      enable => true,
      name   => 'httpd',
  }

  file {
    '/var/www/html/index.html':
      ensure => present,
      content => template('apache/index.html.erb'),
      mode    => '0644',
      owner   => 'root',
      group   => 'root',
      notify => Service['httpd'],
  }
}
ubuntu@ip-10-0-1-249:/etc/puppetlabs/code/environments$ cat production/modules/apache/templates/index.html.erb
<!DOCTYPE html>
<html>
<head>
  <title>Welcome to Apache!</title>
</head>
<body>
  <h1>Welcome to Apache!</h1>
  <p>This is a simple default webpage from Puppet.</p>
</body>
</html>


ubuntu@ip-10-0-1-249:/etc/puppetlabs/code/environments$
```


I'll add the [[Installing Puppet#^60d9d7|agents]] in `/etc/hosts`



### Issue
```
ubuntu@ip-10-0-1-7:~$ puppet agent --test
Info: Refreshed CRL: BC:58:63:59:1A:85:0D:DA:5C:0C:CF:B5:9E:70:29:8F:B8:10:A6:76:18:43:CC:0E:5E:61:39:FA:CF:16:AA:C7
Info: Creating a new RSA SSL key for ip-10-0-1-7.ec2.internal
Info: csr_attributes file loading from /home/ubuntu/.puppetlabs/etc/puppet/csr_attributes.yaml
Info: Creating a new SSL certificate request for ip-10-0-1-7.ec2.internal
Info: Certificate Request fingerprint (SHA256): 88:DD:F2:75:33:D8:59:47:A8:3F:76:11:32:EC:D4:08:D5:1C:0C:F2:24:21:DA:A5:88:4D:2B:CF:EE:4A:9E:6D
Info: Downloaded certificate for ip-10-0-1-7.ec2.internal from https://puppet:8140/puppet-ca/v1
Error: The certificate for 'CN=ip-10-0-1-7.ec2.internal' does not match its private key
Error: The certificate for 'CN=ip-10-0-1-7.ec2.internal' does not match its private key
ubuntu@ip-10-0-1-7:~$
```
<br>![[Pasted image 20231127094333.png]]
used sudo, got new error

> [!tip] correct command with sudo
> ```
>sudo /opt/puppetlabs/puppet/bin/puppet agent --test
>```

```
ubuntu@ip-10-0-1-112:~$ sudo /opt/puppetlabs/bin/puppet agent --test
Info: Refreshing CA certificate
Info: CA certificate is unmodified, using existing CA certificate
Info: Refreshing CRL
Info: CRL is unmodified, using existing CRL
Info: Using environment 'production'
Info: Retrieving pluginfacts
Info: Retrieving plugin

Error: Could not retrieve catalog from remote server: Error 500 on SERVER: Server Error: Could not find node statement with name 'default' or 'ip-10-0-1-112.ec2.internal' on node ip-10-0-1-112.ec2.internal
Warning: Not using cache on failed catalog
Error: Could not retrieve catalog; skipping run
ubuntu@ip-10-0-1-112:~$

```

the new error node looks for its confuration on the server based on its hostname which is given by `aws ip-10-0-1-112.ec2.internal` this hostname is pingable.

This is the fully qualified domain name of the agent node which we can find with `hostname --fqdn`

Now new error i think becaus of syntax
```
Error: Could not retrieve catalog from remote server: Error 500 on SERVER: Server Error: Evaluation Error: Error while evaluating a Function Call, Could not find class ::apache for ip-10-0-1-112.ec2.internal (file: /etc/puppetlabs/code/environments/production/manifests/site.pp, line: 6, column: 3) on node ip-10-0-1-112.ec2.internal
Warning: Not using cache on failed catalog
Error: Could not retrieve catalog; skipping run
```
this is due to the moduel structure


Had an error validating `init.pp`
```bash
sudo /opt/puppetlabs/bin/puppet parser validate /etc/puppetlabs/code/environments/production/modules/apache/manifests/init.pp
```

New error from stuff  in `init.pp`

finally works
<br>![[Pasted image 20231127124114.png]]
<br>![[Pasted image 20231127124356.png]]<br>![[Pasted image 20231127124448.png]]
<br>![[Pasted image 20231127124545.png]]


### Lets try agent1
Spinning new agent1 to replace `10.0.1.7` just for testing, end 108
<mark style="background: #BBFABBA6;">was able to duplicate the install</mark>


### This what worked

> [!NOTE] Module structure
> ```bash
> ubuntu@ip-10-0-1-249:/etc/puppetlabs/code/environments$ tree
> .
> └── production
>     ├── data
>     ├── environment.conf
>     ├── hiera.yaml
>     ├── manifests
>     │   └── site.pp
>     └── modules
>         └── apache
>             ├── manifests
>             │   └── init.pp
>             └── templates
>                 └── index.html.erb
> 
> 7 directories, 5 files
> ```


> [!NOTE]- site.pp
> ```bash
> node 'ip-10-0-1-108.ec2.internal' {
>   include apache
> }
> 
> node 'ip-10-0-1-112.ec2.internal' {
>   include apache
> }
> ```

==change to==

> [!NOTE]- site.pp
> ```bash
> node 'default' {
>   include apache
> }



> [!NOTE]- init.pp
> ```bash
> class apache {
> 
>   package { 'apache2':
>     ensure => installed,
>   }
> 
>   service {
>     'apache2':
>       ensure => running,
>       enable => true,
>       name   => 'apache2',
>   }
> 
>   file {
>     '/var/www/html/index.html':
>       ensure => present,
>       content => template('apache/index.html.erb'),
>       mode    => '0644',
>       owner   => 'root',
>       group   => 'root',
>       notify => Service['apache2'],
>   }
> }
> ```

`index.html.erb`
```html
<!DOCTYPE html>
<html>
<head>
  <title>Welcome to Apache!</title>
</head>
<body>
  <h1>Welcome to Apache!</h1>
  <p>This is a simple default webpage from Puppet.</p>
</body>
</html>
```