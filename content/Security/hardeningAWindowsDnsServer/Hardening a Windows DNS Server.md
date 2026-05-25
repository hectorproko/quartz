---
tags:
  - hardening
  - windows
  - security
  - applocker
  - server2022
linkedin: "False"
quartz: "False"
refactored: "True"
pluralsight: "True"
hands-on: "True"
hardlinked: "True"
---
## Overview

In this lab, I hardened a Windows Server (`172.31.24.20`) configured to serve as a DNS server. The goal of system hardening is to reduce the attack surface of a server by removing or disabling anything that isn't required for it to perform its primary function, in this case, resolving DNS queries. This follows the principle of **least functionality**: a server should only do what it needs to do, and nothing more.

The hardening steps I performed covered four main areas:

- Removing unnecessary server roles (IIS / HTTP)
- Hardening and renaming the default Administrator account
- Disabling unnecessary user accounts and services
- Using **AppLocker** to restrict which applications can execute on the server

Before making any changes, I established a baseline by scanning the server to see its current exposure.

---

## Environment

| Component     | Details                            |
| ------------- | ---------------------------------- |
| Target Server | Windows Server 2022 `172.31.24.20` |
| Scanning Host | Linux console `172.31.24.30`       |
| Tool Used     | Nmap 7.80                          |

---

## Step 1 - Baseline Scan with Nmap

Before hardening anything, I ran an Nmap scan against the Windows Server to get a clear picture of which ports were open and what services were exposed. This is important because you can't effectively reduce your attack surface if you don't first know what that surface looks like.

The `-Pn` flag tells Nmap to skip the ping probe and treat the host as online. This is necessary because Windows Servers typically block ICMP ping by default, which would cause a standard scan to report the host as down even when it isn't.

```bash
nmap -Pn 172.31.24.20
```

**Output:**

```
pslearner@ip-172-31-24-30:~$ nmap -Pn 172.31.24.20
Starting Nmap 7.80 ( https://nmap.org ) at 2026-05-11 01:25 UTC
Nmap scan report for ip-172-31-24-20.ec2.internal (172.31.24.20)
Host is up (0.042s latency).
Not shown: 994 filtered ports
PORT     STATE SERVICE
22/tcp   open  ssh
53/tcp   open  domain
80/tcp   open  http
135/tcp  open  msrpc
445/tcp  open  microsoft-ds
3389/tcp open  ms-wbt-server

Nmap done: 1 IP address (1 host up) scanned in 23.42 seconds
```

**Analysis:** Port `53` (DNS) is expected and required. Port `80` (HTTP) should not be open on a DNS server, it means a web server role (IIS) is installed and actively listening for connections. Ports `135` ([[Remote Procedure Call (RPC)|RPC]]) and `3389` (RDP) are left open in this lab environment for management access, but in a production environment these would typically be restricted or disabled as well.
<!--
**From a hardening perspective**, port 135 is worth knowing about because it has historically been the target of serious exploits (the MS03-026 vulnerability — the one the Blaster worm used — attacked RPC). In a production DNS server you'd want to firewall it off from untrusted networks even if you can't disable the service entirely, since Windows depends on RPC internally.

In your lab it was left open because remote management tools needed it, which is a common real-world tradeoff.
-->
---

## Step 2 - Removing the IIS Role

The Nmap scan revealed that port `80` was open because the **Web Server (IIS)** role was installed. A DNS server has no need to serve web content, so IIS is unnecessary here and should be removed. Leaving unused roles installed increases the attack surface, even if IIS isn't actively being used, its presence means there are code paths and services that could potentially be exploited.

I opened **Server Manager**, which confirmed three installed roles: **DNS**, **File and Storage Services**, and **Web Server (IIS)**. DNS is required. File and Storage Services is a default Windows role that cannot be removed. IIS is the one that needs to go.

![[Pasted image 20260510212755.png]]

To remove it, I navigated to **Manage → Remove Roles and Features**, selected the server `ps-win-1`, deselected **Web Server (IIS)**, and allowed the server to restart automatically to complete the removal.

![[Pasted image 20260510212955.png]]

After the restart, IIS and the HTTP listener are completely gone from the server.

---

## Step 3 - Hardening User Accounts

### 3.1 - Reviewing the Administrators Group

With the unnecessary role removed, the next focus was on user accounts. I opened **Computer Management → Local Users and Groups → Groups** and reviewed the **Administrators** group. Any member of this group has complete, unrestricted access to the machine. At this point, both the default `Administrator` account and the `pslearner` account were members.

The default `Administrator` account is a well-known target for attackers. Because Windows always creates it with the same name, an attacker already knows half of what they need to authenticate, they just have to brute-force the password. Renaming it forces an attacker to discover both the username and the password, which significantly raises the bar.

### 3.2 - Renaming the Default Administrator Account

I right-clicked the **Administrator** account in **Local Users and Groups → Users** and renamed it to `psadmin`. This removes the predictability of the default name while preserving the account's existing permissions and group memberships.

I then set a strong password for the account.

![[Pasted image 20260510213634.png]]

> **Best practice note:** In a non-domain environment like this one, each administrator should have their own uniquely named account in the Administrators group. This enables non-repudiation, you can audit which administrator performed which actions. In a domain environment, unique domain admin accounts would be added to the **Domain Administrators** group instead.

### 3.3 - Disabling Unnecessary Accounts

After handling the Administrator account, I reviewed all other enabled accounts. The **WDAGUtilityAccount** (Windows Defender Application Guard) was enabled but is not needed for a DNS server to function. I opened its properties and checked **Account is disabled**.

![[Pasted image 20260510213903.png]]

Disabling rather than deleting accounts is a common practice, it preserves the account history while ensuring it cannot be used to log in. Disabled accounts are identified in the UI by a downward-pointing arrow on the account icon.

---

## Step 4 - Disabling Unnecessary Services

With accounts hardened, I moved on to Windows Services. Many services are enabled by default that have no relevance to DNS functionality. Running unnecessary services wastes resources and, more importantly, creates additional attack vectors. The principle here is the same as with roles: if it's not needed, it should be off.

I opened **Services** from the Start menu and reviewed what was running. Three services were identified as unnecessary for a DNS server and disabled by changing their **Startup type** to **Disabled**:

- **ActiveX Installer** - Installs ActiveX controls from the internet; not needed on a server
- **Downloaded Maps Manager** - Manages offline maps; entirely irrelevant on a server
- **Print Spooler** - Manages print jobs; a DNS server has no printing function (and notably, the Print Spooler has a history of serious vulnerabilities like PrintNightmare)

![[Pasted image 20260510214158.png]]

I also set the **Application Identity** service to **Automatic** and started it. This service is required for **AppLocker** (covered in the next step) to function. Without it, AppLocker rules are evaluated but not enforced.

---

## Step 5 - Restricting Applications with AppLocker

The final and most powerful hardening technique in this lab was configuring **AppLocker**. AppLocker is a Windows feature that lets you define explicit rules for which applications are allowed to run. Any application not covered by an allow rule is blocked by default, this is a **whitelist** model, which is far more secure than a blacklist approach.

### 5.1 - Generating Default Allow Rules

I opened **Local Security Policy → Application Control Policies → AppLocker → Executable Rules**, right-clicked, and selected **Automatically Generate Rules**. This scans the `C:\Program Files` directory and creates allow rules for all currently installed, legitimate programs.

![[Pasted image 20260510214801.png]]

I named this rule set `Default Programs` and created the rules. This gives AppLocker a clean baseline of what is allowed to run.

![[Pasted image 20260510215300.png]]

### 5.2 - Creating a Deny Rule for Wireshark

To demonstrate how deny rules work, I located the Wireshark entry in the rule list (`Default Programs: WIRESHARK signed by ...`), opened its properties, and changed the **Action** from **Allow** to **Deny**.

![[Pasted image 20260510215707.png]]

Deny rules take priority over allow rules. Any rule marked as **Deny** is shown with a red circle icon in the AppLocker interface so it can be quickly identified.

After applying the deny rule, double-clicking Wireshark on the desktop produces the following result:

![[Pasted image 20260510221154.png|500]]

> **Production note:** In a real environment, rather than creating a deny rule for everyone, you would create an allow rule scoped to only the users or groups who legitimately need the application. The deny-all approach here is for demonstration purposes.

#### Troubleshooting - AppLocker Rules Not Enforcing

After creating the Wireshark deny rule, Wireshark opened normally with no block message. Reloading the policy and restarting the server had no effect.

**Root cause:** The **Application Identity** service had stopped. AppLocker depends on this service to evaluate and enforce rules at runtime. If the service isn't running, AppLocker rules are completely ignored, no errors are shown and applications launch as if no rules exist.

**Resolution steps:**

1. Open **Local Security Policy** and navigate to **AppLocker** (the parent folder above Executable Rules)
2. Right-click **AppLocker** and select **Properties**
3. On the **Executable rules** tab, confirm the **Configured** checkbox is checked
4. Ensure the dropdown is set to **Enforce rules** (not **Audit only**)

![[Pasted image 20260510221050.png]]

5. Open **Services**, locate **Application Identity**, and start it (set Startup type to **Automatic** so it persists across reboots)

![[Pasted image 20260510220352.png]]

After starting the Application Identity service and confirming the AppLocker policy was set to **Enforce rules**, the Wireshark deny rule took effect immediately.

---

## Step 6 - Final Verification Scan

With all hardening steps complete, I ran the Nmap scan one final time to confirm the changes were reflected in the server's network exposure.

```bash
nmap -Pn 172.31.24.20
```

**Output:**

```
pslearner@ip-172-31-24-30:~$ nmap -Pn 172.31.24.20
Starting Nmap 7.80 ( https://nmap.org ) at 2026-05-11 02:20 UTC
Nmap scan report for ip-172-31-24-20.ec2.internal (172.31.24.20)
Host is up (0.042s latency).
Not shown: 995 filtered ports
PORT     STATE SERVICE
22/tcp   open  ssh
53/tcp   open  domain
135/tcp  open  msrpc
445/tcp  open  microsoft-ds
3389/tcp open  ms-wbt-server

Nmap done: 1 IP address (1 host up) scanned in 23.16 seconds
```

**Port 80 (HTTP) is gone.** The server no longer advertises an HTTP listener, confirming that removing the IIS role was successful. Port `53` (DNS) remains open as expected. The attack surface has been meaningfully reduced from what it was at the start.

---

## Summary

|Hardening Action|Reason|
|---|---|
|Removed IIS (Web Server role)|A DNS server doesn't need to serve web content; port 80 had no business being open|
|Renamed default Administrator account|Eliminates a known username, forcing attackers to discover both credentials|
|Disabled WDAGUtilityAccount|Not needed for DNS functionality; unused accounts are unnecessary risk|
|Disabled ActiveX Installer, Downloaded Maps Manager, Print Spooler|Services unrelated to DNS operation; Print Spooler in particular has a history of exploitable vulnerabilities|
|Configured AppLocker with Enforce mode|Whitelists known-good applications; blocks anything else from executing|

This lab reinforced a fundamental mindset in system hardening: **start from a position of denial and explicitly allow only what is needed.** Whether it's open ports, installed roles, enabled accounts, running services, or executable applications, if it isn't required for the server's primary function, it should be removed or disabled.

<!--

```
## OBJECTIVE 2

### Hardening Windows Systems

This challenge will help you understand how to harden a Windows Server with the IP address of 172.31.24.20 to function as a DNS Server. This means that the server should have the appropriate accounts, services, roles, and applications to perform its primary function as a DNS server. Throughout this lab, you will ensure that the default administrator account is hardened, the appropriate administrator groups are applied to any new accounts, and disable unnecessary services and applications that aren't needed for the server to function properly. Before you begin the configuration of the Windows Server, it's a good idea to get a baseline of the server in its current state.

To do this, you'll use the NMAP to scan the **Windows Server** to verify which ports and protocols are currently open.

1. Still on the **NMAP Console**, enter the command `nmap -Pn 172.31.24.20` to see which ports are open on the **Windows Server**.
    
    **Note:** The `-Pn` option does not use ping, since that is normally disabled on Windows Servers.![](https://labs.pluralsight.com/labkeep/lab_assets/79481775-0f5c-4167-a8c8-06bfb132d72f)**Analysis:** You can see the open ports on the Windows Server are **53** (DNS), **80** (http), **135** (RDP), and **3389** (RDP). To function as a DNS server, port 53 should be open, but port 80 is not required, and should be disabled. In a production environment, you may also want to disable ports 135 and 3389, but they are required in this lab environment.
    
    Now that you have seen which ports are open on this server, you need to start hardening it!

```
pslearner@ip-172-31-24-30:~$ nmap -Pn 172.31.24.20
Starting Nmap 7.80 ( https://nmap.org ) at 2026-05-11 01:25 UTC
Nmap scan report for ip-172-31-24-20.ec2.internal (172.31.24.20)
Host is up (0.042s latency).
Not shown: 994 filtered ports
PORT     STATE SERVICE
22/tcp   open  ssh
53/tcp   open  domain
80/tcp   open  http
135/tcp  open  msrpc
445/tcp  open  microsoft-ds
3389/tcp open  ms-wbt-server

Nmap done: 1 IP address (1 host up) scanned in 23.42 seconds
pslearner@ip-172-31-24-30:~$
```

1. Navigate to the **Windows Server**.
    
2. Click the **Start** icon, then click on **Server Manager**.
    
    **Note:** If you see a message to "Try Windows Admin Center and Azure Arc today" you can click **Don't show this message again** and close out of the window by clicking the **X**.![](https://labs.pluralsight.com/labkeep/lab_assets/174a0e86-4373-4387-bd39-83a8b0a107e4)**Analysis**: There are three roles that are installed, **DNS**, **File and Storage Services**, and **IIS**. **DNS** is required for this server to function as a DNS server and **File and Storage Services** is installed by default and cannot be removed. However, **IIS** is the role that is listening for connections coming in on port 80, so that needs to be removed.
![[Pasted image 20260510212755.png]]
1. In the top right corner of **Server Manager**, click **Manage**, and then click on **Remove Roles and Features**.
    
2. In the wizard that opens, click **Next**, ensure the server **ps-win-1** is selected and click **Next**.
    
3. Deselect **Web Server (IIS)**, click **Next**, and click **Next** again.
    ![[Pasted image 20260510212955.png]]
4. Then check the box next to **Restart the destination server automatically if required**. When prompted with a pop-up asking if you want to allow automatic restarts, select **Yes**, and click **Remove**.
    
    **Note:** Removing the **IIS** role does require a restart. The removal of the role and the restart should take about five minutes. When the Server has restarted, go back into **Server Manager**.
    
    Now that the unnecessary role has been removed, another hardening technique is to disable any unnecessary users and user groups.
    
5. Click the **Start** icon, and then click on **Server Manager**.
    
6. In the top right corner of **Server Manager**, click **Tools**, and then click on **Computer Management**.
    
7. If it's not already, expand **System Tools**, then expand **Local Users and Groups**. Click on **Groups**, double-click on **Administrators** to see the group's properties.![](https://labs.pluralsight.com/labkeep/lab_assets/5a2933aa-7bdc-4504-8dd5-f9885a884a55)**Analysis**: In the **Description** section, you can see that anyone is who a member of this group has complete and unrestricted access to the computer and domain. In the **Members** section, **pslearner** (the account you are using to access this lab) and the default **Administrator** account are both members. This means that both of these accounts have complete and unrestricted access to this server.
    
    **Note:** If this server was a domain controller, anyone who was a member of the **Administrators** group would have complete and unrestricted access to the domain.
    
8. Click the **OK** button to close the **Administrators Properties**, then click on the **Users** folder and double-click on the **Administrator** account.
    
    **Analysis:** When Windows Server is installed, the **Administrator** account with membership to the **Administrators** group is created by default, and it is always named 'Administrator.' This poses a security risk, since a malicious actor already knows the username, so now all they have to do is try to crack the password. Disabling the default Administrator account and using a unique account name will help mitigate these attacks, since a malicious actor would have to figure out both the account name and the password.
    
9. Click **Cancel**, then right-click on the **Administrator** account and click **Rename**. Enter `psadmin` for the name, and hit **Enter** on your keyboard.
    
    **Note:** Since this server is not part of a domain, it is best practice to have multiple, unique accounts that are part of the **Administrator** group for each administrator. This allows the ability to log the actions of each individual administrator, providing non-repudiation. Now there are two accounts, **pslearner** and **psadmin**. If this server was part of a domain, the unique domain administrator accounts would be added to the **Domain** **Administrators** group.
    ![[Pasted image 20260510213634.png]]
10. Right-click on the **psadmin** account, and click **Set Password**. When prompted, click **Proceed**.
    
11. Enter `Pluralsight123!` for the password, and then click **Ok**, and click **Ok** again.
    
    **Note:** It is also common practice to simply create a new administrator account, add it to the Administrators group, and then disable the default Administrator account.
    
    Now that the default Administrator account has been changed, and the password has been changed, the next step is to disable any accounts that aren't necessary for the server to perform it's intended functions. The only other account that is enabled is the **WDAGUtilityAccount**, which is not needed for the for this server to function as an FTP server.
    ![[Pasted image 20260510213903.png]]
12. Double click on **WDAGUtilityAccount**, check **Account is disabled**, and then click **OK**.![](https://labs.pluralsight.com/labkeep/lab_assets/c0fded3c-a63c-475b-a04f-000c76eb3368)**Analysis:** All accounts that are not necessary for this server to function as a DNS server have been disabled. You can verify they are disabled by the down arrow icon that appears next to the username.
    
    While in this instance we did disable the **WDAGUtilityAccount**, there may be times where that account is necessary to function. It's up to you as the system administrator to determine which accounts are necessary for the server to function properly. In the context of this lab, the goal of the previous step was to show you how to disable accounts that aren't necessary.
    
    The next set of tasks will disable services and roles that are not necessary for the Windows Server to function as a DNS server.
    
13. Click the **Start** icon, and type `services` and click on the search result of **Services**.
    
    **Note:** Every service that is not required for a DNS server is outside the scope of this lab. It will be up to you to determine which services are not required and disable them. However, you will walk through the steps on how to disable some services.![](https://labs.pluralsight.com/labkeep/lab_assets/d1b7e813-b4d5-488d-92e2-04c04fb1c10c)**Analysis:** The **ActiveX Installer** service is currently set to start up automatically, and is not necessary for a DNS server.
    
14. Double-click on the **ActiveX Installer** service.![](https://labs.pluralsight.com/labkeep/lab_assets/31d83c7a-f22a-46e1-8ad3-26dfe7415342)**Analysis:** In the dialog box that appears, you will be able to see the service name, as well as a description of what the service is for. Additionally, you can configure the service to start automatically, manually, or to be disabled. You can also manually start services in this dialog box.
    
15. Change the **Startup type** from **Automatic** to **Disabled**, since this service is not required, then click **OK** to close the dialog box.
    ![[Pasted image 20260510214158.png]]
16. Perform the previous two steps for **Downloaded Maps Manager** and **Print Spooler**, as these two services are not needed for DNS functionality.
    
    **Analysis:** Once you've determined which services are not needed, another hardening technique is to use an application call **AppLocker** to restrict which applications can be run on the server. In order for **AppLocker** to function properly, the **Application Identity Service** needs to be running.
    
17. **Double-click** on the **Application Identity** service for app locker, change the **Startup type** to **Automatic**, then click **Start** to start the service. Once the service has started, click **OK** to close the dialog box.
    
    **Note:** You might get an error saying that access is denied. In this lab environment, that's to be expected, but in a production environment the **Application Identity** service should be set to automatically start. For now, just click **Cancel** since the service is already running.
    
    Now that only the allowed services have been defined, the last major step is to define which applications are allowed to run on this server. You will use **AppLocker** to create explicit **Allow** rules. Once **AppLocker** is enabled, any application that isn't explicitly allowed will not be able to be executed.
    
18. Click **Start**, type `local security policy`, then click **Local Security Policy**.
    
19. In the **Local Security Policy** console that appears, expand **Application Control Policies**, then expand **AppLocker**.
    
20. Click **Executable Rules** to highlight it, then right-click it and select **Automatically Generate Rules**.![](https://labs.pluralsight.com/labkeep/lab_assets/d1e5f870-9b77-435d-bc5b-a69c97ccf6cb)**Analysis:** By default, the users that this rule will apply to will be **Everyone**. You could change this to only apply to a specific user, or a group of users. The default folder is **C:\Program Files**, but it would also be good to repeat this process to generate rules for any programs that aren't 64-bit, which would be located at **C:\Program Files (x86)**.
    ![[Pasted image 20260510214801.png]]
21. Change value in **Name to identify this set of rules** to `Default Programs`, click **Next**, leave the default and click **Next** again.
    
    **Note:** The generation of these rules will take about 2 minutes.
    
22. Once the rules have been generated, click **View rules that will be automatically created**. Scroll through and see which applications are allowed by default, click **ok** to close.
    
23. Then click **Create** to create the rules.
    ![[Pasted image 20260510215300.png]]
    **Note**: It is best practice to remove applications that are not needed by anyone who is using the server. However, sometimes applications need to be run by certain administrators or processes. As a test, you will first run the **Wireshark** application, and then disable the **Wireshark** application and verify that it cannot be opened..
    
24. Minimize all windows and double-click on the **Wireshark** application that is on the desktop. It should open. **Close** the **Wireshark** application.
    
25. Go back to the **Local Security Policy** window, and sort the list of applications by **Name** by clicking on **Name** in the top row. Search for `Default Programs: WIRESHARK signed by ...` and **double-click** that rule.
    
26. Change the **Action** to **Deny** and click **OK**.
    ![[Pasted image 20260510215707.png]]
    **Note:** This rule is applied to everyone for example purposes, but in a production environment, you'd create an allow rule and specify only the people that need to use this application.![](https://labs.pluralsight.com/labkeep/lab_assets/d22109ca-9884-4e59-a1aa-e5e6f71e5d0b)**Analysis:** Any rules that are set to **Deny** have an icon depicting a red circle with a line through it so you can quickly see which rules are **Allow** rules and which rules are **Deny** rules.
    
27. Double-click on the **Wireshark** application that is on the desktop.![](https://labs.pluralsight.com/labkeep/lab_assets/44458866-52eb-47aa-a141-ac3df09fbd02)**Analysis:** You see a message stating that **This app has been blocked by your system administrator.** 
    ❌i was able to open it no problem,relaoded, and restarted the server still nothing
    ✅ appplicaiton identity services was stopped
    ![[Pasted image 20260510220352.png]]
    - In the **Local Security Policy** window, right-click on **AppLocker** (the parent folder of Executable Rules).
    
- Select **Properties**.
    
- On the **Executable rules** tab, make sure the **Configured** checkbox is checked.
    
- Ensure the dropdown is set to **Enforce rules** (not "Audit only").
    ![[Pasted image 20260510221050.png]]
![[Pasted image 20260510221154.png]]

That is all the tasks that you will need to complete to harden the Windows Server. As a final check, you should see what the **NMAP** scan shows.
    
2. Connect to the **NMAP Console** to verify that the hardening techniques are visible.
    
3. Enter `nmap -Pn 172.31.24.20` to verify that the **Windows Server** is no longer listening for **HTTP** connections.![](https://labs.pluralsight.com/labkeep/lab_assets/ad247249-fbbe-4afb-8e27-47adc832d76d)**Analysis:** Port 80 (http) is no longer open and listening for connections.
    

Congratulations! You have just finished this challenge. You now have a solid understanding of how to harden a Windows Server. The next challenge awaits.

```
pslearner@ip-172-31-24-30:~$ nmap -Pn 172.31.24.20
Starting Nmap 7.80 ( https://nmap.org ) at 2026-05-11 02:20 UTC
Nmap scan report for ip-172-31-24-20.ec2.internal (172.31.24.20)
Host is up (0.042s latency).
Not shown: 995 filtered ports
PORT     STATE SERVICE
22/tcp   open  ssh
53/tcp   open  domain
135/tcp  open  msrpc
445/tcp  open  microsoft-ds
3389/tcp open  ms-wbt-server

Nmap done: 1 IP address (1 host up) scanned in 23.16 seconds
pslearner@ip-172-31-24-30:~$
```


[[Loki]]