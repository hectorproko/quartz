
> [!NOTE] Module 2 — Assignment
> #### Problem Statement:
> You work for XYZ organization, and your work environment is UNIX/Linux-based. Your assignment consists of the following tasks:
> 
> 1. **Create 3 Users**: Create three users who will have access to the server.
> 2. **Create a Group**: Establish a group named "webdev" and add the newly created users to this group.
> 3. **Folder and File Permissions**: Create a folder named "dev" and place a file within it. Set permissions for the file such that it can only be accessed by members of the "webdev" group.
> 4. **User Removal**: Remove one user from the "webdev" group and subsequently delete that user from the system.
> 5. **File Relocation and Ownership Change**: Move the file from the "dev" folder to another folder. Change the file's group ownership thereafter.
> 



1. **Create 3 Users**: Create three users who will have access to the server.
```bash
sudo useradd user1
sudo useradd user2
sudo useradd user3
```

Confirming: 
```bash
getent passwd | grep 'user1\|user2\|user3'
```
![[Pasted image 20230819235423.png]]
Set their passwords:
```bash
sudo passwd user1
sudo passwd user2
sudo passwd user3
```
![[Pasted image 20230819235738.png]]

2. **Create a Group**: Establish a group named "webdev" and add the newly created users to this group.
```bash
sudo groupadd webdev
```
Adding the users to group dev:
```bash
sudo usermod -aG webdev user1
sudo usermod -aG webdev user2
sudo usermod -aG webdev user3
```
Confirming:
```bash
getent group webdev
```
![[Pasted image 20230820003007.png]]


3. **Folder and File Permissions**: Create a folder named "dev" and place a file within it. Set permissions for the file such that it can only be accessed by members of the "webdev" group.
```bash
mkdir dev #Create the folder "dev"
cd dev #Change to the "dev" directory:
touch myfile.txt #Create a file inside "dev"
ls -l #To see the permissions and ownership
```
![[Pasted image 20230820000915.png]]
Changing group ownership of the file to "webdev":
```bash
sudo chown :webdev myfile.txt
```
Changing the file permissions so that only group members can read the file:
```bash
sudo chmod 040 myfile.txt #only group can read
```
![[Pasted image 20230820002033.png]]

4. **User Removal**: Remove one user from the "webdev" group and subsequently delete that user from the system.

Removing user user3 from the group:
```bash
sudo gpasswd -d user3 webdev #remove from group
```
Removal and confirmation:  
![[Pasted image 20230820003434.png]]
Deleting the user:
```bash
sudo userdel -r user3
```
Deletion and confirmation:  
![[Pasted image 20230820003820.png]]

5. **File Relocation and Ownership Change**: Move the file from the "dev" folder to another folder. Change the file's group ownership thereafter.

Creating "prod" folder:
```bash
mkdir ../prod
```
Moving the file there:
```bash
mv myfile.txt ../prod/
```
Changing the group ownership to "prod":
```bash
sudo chown :newgroup ../prod/myfile.txt
```
Confirming:  
![[Pasted image 20230820004623.png]]



