[https://us02web.zoom.us/rec/share/9DpuGabp9qeA_T0aJRlbruNaKz_ahtQWEMp5d_HkfA30r8CWyF_Hr9xzmSlDUNdA.YEDWDWRdIvwox24s](https://us02web.zoom.us/rec/share/9DpuGabp9qeA_T0aJRlbruNaKz_ahtQWEMp5d_HkfA30r8CWyF_Hr9xzmSlDUNdA.YEDWDWRdIvwox24s) 

Passcode: SE.k57?1


### assignment 1
Instances master t2.medium
slaves t2.micro

ubuntu ==20.4== confirm


all 3 machines
sudo apt update
sudo apt install docker.io -y

kubeadm init, only thing in master

creates script file
<br>![[Pasted image 20231113074350.png]]

in master
`kubeadm init`
follows instruction to run
<br>![[Pasted image 20231113074845.png]]


network pod
<br>![[Pasted image 20231113075052.png]]

> [!NOTE]
> **Weave Net (`weave-daemonset-k8s.yaml`)**:
> - **Weave Net** is another networking solution that creates a virtual network that connects Docker

  
  
then the toke on the slaves

In master he checks and slave is connected
<br>![[Pasted image 20231113075236.png]]

adds another slave

---
deployment could be done with comand or yaml
<br>![[Pasted image 20231113075807.png]]
goes here to see deployment yaml file

create with nano in the severs and paste it
<br>![[Pasted image 20231113075910.png]]

changes to only nginx, instead of specific version
replicas is already set to 3
kubectl apply -f assign1.yaml
kubectl get deploy

one line command instead of yaml
``kubectl create deployment nginx-deployment-1 --image=nginx --replicas=3``

so see pods `kubectl get pods`

### assignment 2

creates assign2.yml
<br>![[Pasted image 20231113081749.png]]
The selector is looking for keywor app: nginx
Our deployment .yaml has this

service nodeport created
<br>![[Pasted image 20231113082019.png]]

<br>![[Pasted image 20231113082111.png]]

### Assignment 3

> [!attention]
> In OneNote look into CREATE A REPLICA SET sort of dicuss this topic

<br>![[Pasted image 20231113083023.png]]

will change the number replicas for both

`kubectl edit deploy` opens editor
<br>![[Pasted image 20231113083130.png]]
<br>![[Pasted image 20231113083205.png]]
in the edit thing both deploytment appear need to scroll

<br>![[Pasted image 20231113083217.png]]

kubectl get deploy, both show 5 once he edits both
kubectl get pod both have 5 after edit both

### assignment 4

<br>![[Pasted image 20231113085007.png]]

edits the service
<br>![[Pasted image 20231113085058.png]]
<br>![[Pasted image 20231113085205.png]]

<mark style="background: #FFB8EBA6;">I think this is wrong</mark>
> He is adamant on saying that we can not update the deploy with .yaml, because we need to delete the old, is not editing, you would be recreating, if the comamnd has `-f` will be force wont tell that it is repeating? He tried it with f and the changes just happened no message, maybe in background was destroy and created again

<br>![[Pasted image 20231113085555.png]]

### Assignment 5
1:27
Creaing an ingress
 we wont be able to verify the changes in the browser?

ingress service the nginx deployment


Creates new instance, with minikube beacuse with kubeadm
He used Ubuntu 20.4
t2.medium

minikube, only master is also salve, one node architecture

sudo apt udpate
also creates script file
<br>![[Pasted image 20231113090746.png]]
add -y to docker

after install
<br>![[Pasted image 20231113091119.png]]
the master and slave

<br>![[Pasted image 20231113093858.png]]

will use one line commadn to nodeport
<br>![[Pasted image 20231113094005.png]]

<br>![[Pasted image 20231113094022.png]]
still not accessible from the broswer
<br>![[Pasted image 20231113094059.png]]

needs port forwarding
using porforward commadn
<br>![[Pasted image 20231113094326.png]]
all IPs and https, NOT http

he put also 80 later
<br>![[Pasted image 20231113094449.png]]
*if you cancel this operation it wont be working*

make sure its http in broswer
<br>![[Pasted image 20231113094750.png]]


creates ingress file
<br>![[Pasted image 20231113095005.png]]

<br>![[Pasted image 20231113095103.png]]

creating an ingress for the nodeport
we are not forwarding the ingress directly we are forwarding teh ingress nginx controller, -n name-space this particula name space was created automatically

<br>![[Pasted image 20231113095602.png]]
this time is for 443

if i leave it with http
<br>![[Pasted image 20231113095657.png]]

<br>![[Pasted image 20231113095807.png]]

did it again
<br>![[Pasted image 20231113095850.png]]
for http is also giving smae
<br>![[Pasted image 20231113095921.png]]

so this is succesful cuse is not bad connection message?