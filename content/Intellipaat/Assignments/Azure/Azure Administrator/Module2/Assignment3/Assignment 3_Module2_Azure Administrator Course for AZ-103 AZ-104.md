---
tags:
  - azure
title: Setting Up Azure File Shares on Windows and Linux Systems
---
<!--
**Project Highlight: Azure File Share Creation and Mounting!** I've just completed an insightful assignment in the Azure Administrator course, focusing on Azure Storage. My task was to create a file share in Azure Storage and then mount it on both Windows and Linux systems. This assignment involved navigating the Azure portal to set up the file share, obtaining access credentials, and executing mount commands on different operating systems. Successfully completing this task has enhanced my understanding of Azure's file services and cross-platform integration, showcasing the versatility and practicality of Azure Storage solutions in a multi-environment setup.
-->

> [!info] Module 2: Assignment - 3
> **Tasks To Be Performed:** 
> 1. Create a file share in Azure Storage 
> 2. Mount this file share on Windows and Linux

> [!attention] Prerequisite:
> **Linux VM** refencing steps in [[Assignment 1_Module4_Azure Administrator Course for AZ-103 AZ-104|Assignment 1: Module 4]]
> **Windows VM** refencing steps in [[Assignment 2_Module4_Azure Administrator Course for AZ-103 AZ-104|Assignment 2: Module 4]]
> 

### Step 1: Create a File Share in Azure Storage

**I Navigate to the Azure Portal**:

**I Access the Storage Account** `hectorstorage12345`

**I Create the File Share**: In the storage account overview, I navigate to the "File shares" section and click on "+ File share". I give the file share a name `myfileshare` and specify the quota (the maximum size the file share can grow to). After entering the details, I click "Create" to create the file share.

### Step 2: Mount the File Share on Windows and Linux

<br>![[Pasted image 20231206143016.png]]

#### On Windows:
**I Get the Access Credentials**: In the Azure Portal, within my file share, I click on "Connect". This opens a pane with scripts to mount the file share. I select the Windows option and copy the script.

Script given to me:
```powershell
$connectTestResult = Test-NetConnection -ComputerName hectorstorage12345.file.core.windows.net -Port 445
if ($connectTestResult.TcpTestSucceeded) {
    # Save the password so the drive will persist on reboot
    cmd.exe /C "cmdkey /add:`"hectorstorage12345.file.core.windows.net`" /user:`"localhost\hectorstorage12345`" /pass:`"vLrPX5ip459mPnXpwBu24qho8n/oZgzhzXvh37upYismRIGIprAr1CRcCsRTP+RHVgHGX3/GDhb2+AStF45TFQ==`""
    # Mount the drive
    New-PSDrive -Name Z -PSProvider FileSystem -Root "\\hectorstorage12345.file.core.windows.net\myfileshare" -Persist
} else {
    Write-Error -Message "Unable to reach the Azure storage account via port 445. Check to make sure your organization or ISP is not blocking port 445, or use Azure P2S VPN, Azure S2S VPN, or Express Route to tunnel SMB traffic over a different port."
}
```
   
**I Run the Script on My Windows Machine**: On my Windows computer, I open PowerShell as an Administrator and paste the script. This script mounts the Azure file share as a network drive.

Script output:
<br>![[Pasted image 20231206143816.png]]

I navigate to "This PC" and verify share drive
<br>![[Pasted image 20231206143917.png]]


#### On Linux:
**I Prepare My Linux System**: I ensure that the `cifs-utils` package is installed on my Linux system for mounting SMB/CIFS shares. I run:
```bash
sudo apt-get update
sudo apt-get install cifs-utils
```
**I Get the Mount Command**: Back in the Azure Portal, in the "Connect" pane for the file share, I select the Linux script and copy it.
```bash
sudo mkdir /mnt/myfileshare
if [ ! -d "/etc/smbcredentials" ]; then
sudo mkdir /etc/smbcredentials
fi
if [ ! -f "/etc/smbcredentials/hectorstorage12345.cred" ]; then
    sudo bash -c 'echo "username=hectorstorage12345" >> /etc/smbcredentials/hectorstorage12345.cred'
    sudo bash -c 'echo "password=vLrPX5ip459mPnXpwBu24qho8n/oZgzhzXvh37upYismRIGIprAr1CRcCsRTP+RHVgHGX3/GDhb2+AStF45TFQ==" >> /etc/smbcredentials/hectorstorage12345.cred'
fi
sudo chmod 600 /etc/smbcredentials/hectorstorage12345.cred

sudo bash -c 'echo "//hectorstorage12345.file.core.windows.net/myfileshare /mnt/myfileshare cifs nofail,credentials=/etc/smbcredentials/hectorstorage12345.cred,dir_mode=0777,file_mode=0777,serverino,nosharesock,actimeo=30" >> /etc/fstab'
sudo mount -t cifs //hectorstorage12345.file.core.windows.net/myfileshare /mnt/myfileshare -o credentials=/etc/smbcredentials/hectorstorage12345.cred,dir_mode=0777,file_mode=0777,serverino,nosharesock,actimeo=30
```


**I Mount the File Share on My Linux Machine**: On my Linux machine, I open the terminal and paste the script. This script will mount the Azure file share at a specified mount point in my Linux filesystem.

I verify the mount with `df -h`
<br>![[Pasted image 20231206145148.png]]

### Verifying File Share

**Linux:**
From Linux machine I create a file inside `/mnt/myfileshare` named `file_created_in_linux`
<br>![[Pasted image 20231206145814.png]]

**Windows:**
I check in Windows VM and see the same file
<br>![[Pasted image 20231206145905.png]]

**Azure Portal:**
I check in the portal
<br>![[Pasted image 20231206150050.png]]