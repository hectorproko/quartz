[https://drive.google.com/file/d/1VIRGzOQltf8kNsWr7Pqj96VJXctqZn_p/view?usp=sharing](https://drive.google.com/file/d/1VIRGzOQltf8kNsWr7Pqj96VJXctqZn_p/view?usp=sharing)


[[commands]]
<br>![[Devops Project 2.docx]]

the link website is not working properly

1EC2 controller, install 2 things, jenkins, terraform (to create 2 slave)

2 new slaves with help of terraform

in slave 1, kubernetes, java, docker
<br>![[Pasted image 20231211194520.png]]

lauches EC2 ubuntu, called "terraform", this is the controller

looks on how to instlal jeknins 
<br>![[Pasted image 20231211195203.png]]

creates scripts where he also add java 
<br>![[Pasted image 20231211195334.png]]

--- time 33 ---
==he shares the links==
install terraform

<br>![[Pasted image 20231211195829.png]]
done with controller


Now he uses terraform to generate two slaves, kub-master, kub-slave
`nano main.tf`

time 38 explains terraform main.tr
simple 2 instnaces, can reuse other assign just add tags with anem
<br>![[Pasted image 20231211200340.png]]


kubemaster
install java
<br>![[Pasted image 20231211200834.png]]
also in kube-slave, java

has a whole command for kubeadm
<br>![[Pasted image 20231211201038.png]]



also runs it in kube-slave
<br>![[Pasted image 20231211201148.png]]


to create the cluster
<br>![[Pasted image 20231211201301.png]]

generates the token that he will paste on the slave

create services? nodeport
<br>![[Pasted image 20231211202008.png]]
<br>![[Pasted image 20231211202047.png]]


pastes the token in kube-slave

in master
<br>![[Pasted image 20231211202215.png]]
he said to use -w to autosacling, first he only saw one worker than the second worker appeared, said it will keep increasing, way to end was Crtl + C

install docker.io in kube-master

uses public ip of terrafrom instace, to use jenkins, the install user creation etc, initialize it

adds kube-master as a node in Jenkins
In ID you can write anything
<br>![[Pasted image 20231211202835.png]]

---time 1:00 hour---
--- time 1:26 hour --- 

in jenins he makes crednetial "manag credential" add credential

username dockerhub account username and password and creates

gives you or shows you credential id
<br>![[Pasted image 20231211212358.png]]

copy this id
<br>![[Pasted image 20231211212413.png]]

with this id we are going to push it

<br>![[Pasted image 20231211212453.png]]


1. first hello
<br>![[Pasted image 20231211212900.png]]


<br>![[Pasted image 20231211213232.png]]
adds teh credentials with the id we copied

1:39 he goes over the stages


there is  docker push images he checks in  dockerhub