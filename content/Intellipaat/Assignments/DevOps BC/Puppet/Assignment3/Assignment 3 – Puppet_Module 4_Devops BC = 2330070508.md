 > [!info] Module 4: Puppet Assignment - 3
> **Tasks To Be Performed:** 
> 1. Use the previous module’s deployment 
> 2. If Apache is installed change index.html to “apache is installed” 
> 3. If NGINX is installed change index.html to “nginx is installed” 

edited site.pp
```bash
node default {

  # Check if Apache is installed, and if so, change the index.html content
  exec { 'index_apache_installed':
    path    => '/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games',
    command => "sudo echo 'apache is installed' > /var/www/html/index.html",
    onlyif  => '/bin/which apache2',
  }

  # Check if NGINX is installed, and if so, change the index.html content
  exec { 'index_nginx_installed':
    path    => '/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games',
    command => "sudo echo 'nginx is installed' > /var/www/html/index.html",
    onlyif  => '/bin/which nginx',
  }

}
```


```
sudo systemctl restart puppetserver
```

<br>![[Pasted image 20231127220332.png]]

<br>![[Pasted image 20231127220312.png]]

Issue:
Not having correct `path` made it not not work
NOT WORK: `path => ['/bin', '/usr/bin', '/usr/sbin'],`
Solution:
Changed it to `path    => '/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games',`
