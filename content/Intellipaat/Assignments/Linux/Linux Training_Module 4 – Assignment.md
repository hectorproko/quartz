---
tags:
  - Bash
---

> [!NOTE] Module 4 - Assignment 
> **Problem Statement:** 
> Write a script to echo the following message before installing httpd in the system. The message is "You are using the system as" Change the file permission and test the script.


1. Using nano we create the script
```bash
nano install_httpd.sh
```
The contents of the script:
```bash
#!/bin/bash

# Echo the username
echo "You are using the system as $(whoami)"

# Install httpd
sudo apt update && sudo apt install -y httpd
```
*I'm using Ubuntu*

2. Make the script executable
```bash
chmod +x install_httpd.sh
```
![[Pasted image 20230820013308.png]]

3. Executing the script
```bash
./install_httpd.sh
```
![[Pasted image 20230820013352.png]]




