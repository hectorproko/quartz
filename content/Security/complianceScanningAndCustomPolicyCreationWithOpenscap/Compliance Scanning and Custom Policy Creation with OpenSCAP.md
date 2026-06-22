---
tags:
  - draft
  - security
  - "#compliance"
  - "#openscap"
  - "#RHEL"
  - hardening
  - RHEL7
linkedin: "True"
quartz: "True"
refactored: "True"
pluralsight: "True"
hands-on: "True"
hardlinked: "True"
title: Compliance Scanning and Policy Creation with OpenSCAP
---
## Overview

Security compliance isn't just about passing audits, it's about knowing exactly where your systems stand against a **defined security baseline** and having a repeatable process to verify it. In this lab, I used **OpenSCAP** and **SCAP Workbench** on a RHEL 7 host to do two things:

1. Run a compliance scan against the **[[C2S (Common Cloud Security) profile]]**, a hardened baseline for Red Hat Enterprise Linux 7.
2. Build and run a **custom scan policy** tailored to specific internal security rules, then export the results as a shareable HTML report.

These two tasks reflect a realistic workflow: you start by scanning against a known standard to understand your baseline, then you refine your policies to match what actually matters in your environment.

---

## Environment

- **OS:** Red Hat Enterprise Linux 7
- **Tool:** [[SCAP Workbench]] (GUI frontend for [[OpenSCAP]]) 
- **Access:** Remote desktop via VNC on TCP port `5901`
- **Connection:**
    - **Windows:** [VNC Viewer](https://www.realvnc.com/en/connect/download/viewer/)

![[Pasted image 20260516084131.png]]

---

## Part 1 - Running a Baseline C2S Compliance Scan

### Why This Matters

Before hardening anything, you need a picture of your current state. The **[[C2S (Common Cloud Security) profile|C2S profile]]** is based on the [[CIS (Center for Internet Security)|Center for Internet Security (CIS)]] Level 2 recommendations adapted for RHEL. Running this scan tells you which controls are passing, which are failing, and what needs attention, without changing a single configuration.

---

### Step 1 - Install SCAP Workbench

SCAP Workbench provides a graphical interface over the OpenSCAP engine, making it easier to select profiles, run scans, and export results. Install it with:

```bash
sudo yum install -y scap-workbench
```

---

### Step 2 - Load the RHEL 7 Content

Open SCAP Workbench from:

**Applications → System Tools → SCAP Workbench**

![[Pasted image 20260516091739.png|400]]

When prompted to _Select content to load_, choose **RHEL7** and click **Load Content**. This loads the pre-bundled SCAP data stream, which contains all the rule definitions, checks, and remediation scripts for RHEL 7.

---

### Step 3 - Select the C2S Profile and Run the Scan

1. From the **Profile** dropdown, select **C2S for Red Hat Enterprise Linux 7**
2. Set the **Target** to **Local Machine**
3. Click **Scan**

The scanner iterates through each rule in the profile, checking things like SSH configuration, password policies, audit logging settings, and firewall rules, and marks each as `pass`, `fail`, or `notchecked`.

Once complete, close the **Diagnostics** window.

![[Pasted image 20260516092026.png]]

---

### Step 4 - Export the Results as an HTML Report

Click **Save Results → HTML Report**, name the file `scan_results.html`, and save it.

![[Pasted image 20260516092136.png|600]]

---

### Step 5 - View the Report

Open Firefox and navigate to:

```
file:///home/cloud_user/scan_results.html
```

The HTML report gives a clear breakdown of the scan: a summary scorecard at the top, followed by a rule-by-rule table showing pass/fail status, severity, and descriptions.

![[Pasted image 20260516092732.png]]

**Example output - what to expect in the report:**

| Rule                                                      | Result   | Severity |
| --------------------------------------------------------- | -------- | -------- |
| Ensure SSH Protocol is set to 2                           | **pass** | High     |
| `Disable Telnet Service`                                  | **fail** | High     |
| Enable Auditing for Processes Which Start Prior to auditd | **pass** | Medium   |
| Verify firewalld Enabled                                  | **fail** | Medium   |

> A typical first scan on an unconfigured host will show a mix of passes and failures. The value is in having a documented, repeatable baseline, not expecting perfection on the first run.

---

## Part 2 - Creating and Running a Custom Scan Policy

### Why This Matters

Pre-built profiles like C2S cover a broad set of controls, but real environments often have specific concerns. Maybe you're focused on a particular set of services your team manages, or you want a lighter scan that targets only the rules relevant to a compliance requirement you're actively working toward. Creating a **custom profile** lets you scope your scans precisely, reduce noise, and audit exactly what you care about.
<!--
we are curating rules from an existing profile C2S, aka selecting some rules, not creating new rules
these new selected rules create a new custom policy, profile
Writing a **net new rule from scratch** would mean authoring the OVAL XML yourself — defining exactly what file to check, what registry key to read, what service state to query. That's a much deeper task and a different skill set entirely.
-->
---

### Step 1 - Create a New Profile from the RHEL7 Baseline

In SCAP Workbench (with RHEL7 content loaded), click the **Customize** button next to the Profile dropdown.

When prompted for a **New Profile ID**, enter:

```
xccdf_org.ssgproject.custom_profile_1
```

^58f5d4

This ID follows the [[XCCDF naming convention]], which ensures the profile is properly identified in exported scan results and XML files.

![[Pasted image 20260516092938.png|600]]

---

### Step 2 - Select Only the Rules You Need

In the customization window:

1. Click **Deselect All** - this clears the inherited rules from the base profile, giving you a clean slate.
2. Navigate the rule tree and check only the rules relevant to your policy:

|Path|Rule|
|---|---|
|Services → Obsolete Services → Telnet|✅ **Uninstall telnet-server Package**|
|Services → FTP Server → Disable vsftpd if Possible|✅ **Uninstall vsftpd Package**|
|System Settings → Network Configurations and Firewalls → firewalld|✅ **Verify firewalld Enabled**|
|System Settings → Network Configurations and Firewalls → firewalld|✅ **Install firewalld**|

![[Pasted image 20260516093504.png]]

**Why these rules?**

- **Telnet and FTP** are legacy, unencrypted protocols. Their presence on a modern host is a direct security risk, credentials sent over these services are transmitted in plaintext.
- **firewalld** is the standard host-based firewall for RHEL. Verifying it's installed and active ensures there's a layer of network-level protection even if the host is misconfigured elsewhere.

Click **OK** to apply the customization.

![[Pasted image 20260516093833.png|600]]

---

### Step 3 - Save the Custom Policy as XML

Navigate to **File → Save Customization Only**, name the file `custom_profile_1.xml`, and click **Save**.

This exports the profile as a standalone **XCCDF XML** file. Saving it separately is important, it means you can version-control the policy, share it with teammates, or load it into other OpenSCAP tools independently of the base content.

---

### Step 4 - Scan with the Custom Profile

1. Set **Target** to **Local Machine**
2. Click **Scan**

Because this profile only contains 4 rules instead of hundreds, the scan completes quickly. Close the **Diagnostics** window when done.

![[Pasted image 20260516094054.png]]

---

### Step 5 - Export and Review the Custom Scan Report

Click **Save Results → HTML Report**, name it `scan_results.html`, and open it in Firefox.

![[Pasted image 20260516094353.png]]

The report now reflects only your custom rules, making it easy to share a focused compliance snapshot with your team or attach to a ticket.

**Example output - custom scan results:**

|Rule|Result|Severity|
|---|---|---|
|Uninstall telnet-server Package|**pass**|High|
|Uninstall vsftpd Package|**pass**|High|
|Verify firewalld Enabled|**fail**|Medium|
|Install firewalld|**fail**|Medium|

> In this example, the host correctly has telnet and vsftpd removed, but firewalld isn't active, a clear, actionable remediation target.

---

## Key Takeaways

|Concept|What I Practiced|
|---|---|
|**Baseline scanning**|Ran a full C2S compliance scan and reviewed an HTML results report|
|**Custom policy creation**|Built a scoped XCCDF profile targeting specific services and firewall rules|
|**Profile export**|Saved a custom profile as XML for reuse and version control|
|**Reporting**|Generated shareable HTML reports from both scan types|

OpenSCAP is a powerful addition to any security operations or compliance workflow. The ability to scan against industry-standard profiles out of the box, then layer in custom policies specific to your environment, makes it flexible enough for both formal audits and day-to-day hygiene checks.

---


%%
# POST
Continuing with security hardening topics. Recently got some solid hands-on time with **OpenSCAP** and SCAP Workbench. I started by running a compliance scan specifically against the **baseline C2S (Common Cloud Security) profile** for RHEL 7.

Then I went ahead and built my own **custom scan policy** tailored to specific internal security rules, for example, disabling legacy services like Telnet and FTP, and ensuring firewalld is active. Finally, generated clean, professional HTML reports for both scans.

It was a very informative hand-on that demystifies compliance scanning and policy creation. 

[https://hectorproko.github.io/quartz/Security/complianceScanningAndCustomPolicyCreationWithOpenscap/Compliance-Scanning-and-Custom-Policy-Creation-with-OpenSCAP](https://hectorproko.github.io/quartz/Security/complianceScanningAndCustomPolicyCreationWithOpenscap/Compliance-Scanning-and-Custom-Policy-Creation-with-OpenSCAP)

`#CyberSecurity #Compliance #OpenSCAP #RHEL #SecurityHardening #InfoSec`

%%


%%# Drafts

# Run an OpenSCAP Compliance Scan on a Host


## Lab Overview

In this lab, we will be installing OpenSCAP and scanning a host for compliance. OpenSCAP is a powerful tool used to scan hosts to **validate compliance** with **predetermined rule sets**. This allows us to identify where we fall out of compliance and remediate the identified issues.

## Solution

We will connect to our lab server using VNC. The IP address and credentials are provided in the hands-on lab page.

VNC connections will be different for each operating system:

- For Mac users:
    - Open Finder
    - Press **Command+K** on your keyboard to bring up the _Connect to server_ window
        - Alternatively, expand **Go** in the menu at the top of the screen and click **Connect to Server**
    - In the _Connect to Server_ window, connect to `vnc://<IP_ADDRESS>:5901`, making sure to replace `<IP_ADDRESS>` with the IP address you are provided on the hands-on lab page
- Windows users will need to install an application like [VNC Viewer](https://www.realvnc.com/en/connect/download/viewer/) to connect.

### Install SCAP Workbench

1. To install SCAP Workbench, run the following command from Terminal on the lab server:

```
sudo yum install -y scap-workbench
```

### Scan the localhost for C2S compliance and create a report

1. Open SCAP-Workbench

- **Applications Menu** -> **System Tools** -> **SCAP Workbench**
![[Pasted image 20260516091739.png]]

1. Choose **RHEL7** when prompted to _Select content to load:_, then click the **Load Content** button
2. From the _Profile_ drop down, select **C2S for Red Hat Enterprise Linux 7**
3. Click the radial button next to **Local Machine** for the _Target_
4. Click the **Scan** button at the bottom to start the scan
5. Once the scan is complete click **Close** in the _Diagnostics_ window
   ![[Pasted image 20260516092026.png]]
6. Click the _Save Results_ drop down button and select **HTML Report**
   ![[Pasted image 20260516092136.png]]
7. Type "scan_results.html" in the name and click **Save**


### View the report

1. Open **Firefox** on the lab server and navigate to `file:///home/cloud_user/scan_results.html`
![[Pasted image 20260516092732.png]]
## Conclusion

Congratulations — you've completed this hands-on lab!


# Creating a Custom Scan Policy with OpenSCAP

_This course is not approved or sponsored by Red Hat._

## Introduction

In this hands-on lab, we will use the SCAP Workbench tool to create a custom policy and scan a host with it. SCAP Workbench comes with preconfigured rule sets, from which we can create our own custom policies we'll use to scan our environment for compliance with internal security policies.

## Solution

We will connect to our lab server using VNC. The IP address and credentials are provided in the hands-on lab page.

**Note:** Please give the lab an extra two minutes before you try to connect with VNC on tcp port 5901.

VNC connections will be different for each operating system:

- For Mac users:
    
    - Open Finder
        
    - Press **Command+K** on your keyboard to bring up the _Connect to server_ window
        
        - Alternatively, expand **Go** in the menu at the top of the screen and click **Connect to Server**
    - In the _Connect to Server_ window, connect to `vnc://<IP_ADDRESS>:5901`, making sure to replace `<IP_ADDRESS>` with the IP address you are provided on the hands-on lab page
        
- Windows users will need to install an application like [VNC Viewer](https://www.realvnc.com/en/connect/download/viewer/) to connect.
    
![[Pasted image 20260516084131.png]]
### Create a Custom OpenSCAP Policy

1. Open SCAP Workbench by navigating to **Applications** > **System Tools** > **SCAP Workbench**.
    
2. Set _Select content to load:_ to **RHEL7**.
    
3. Click **Load Content**.
    
4. Click the **Customize** button next to _Profile_.
    
5. Provide a _New Profile ID_ of "xccdf_org.ssgproject.custom_profile_1", and click **OK**.
    ![[Pasted image 20260516092938.png]]
6. In the customizing window:
    
    1. Click **Deselect All** at the top.
    2. Under _Services_ > _Obsolete Services_ > _Telnet_, check the box next to **Uninstall telnet-server Package**.
    
    3. Under _Services_ > _FTP Server_ > _Disable vsftpd if Possible_, check the box next to **Uninstall vsftpd Package**.
       ![[Pasted image 20260516093504.png]]
    4. Under _System Settings_ > _Network Configurations and Firewalls_ > _firewalld_ > _Inspect and Activate Default firewalld Rules_, check the boxes to **Verify firewalld Enabled** and **Install firewalld**.
       
1. Click the **OK** button at the bottom of the customization window.
    ![[Pasted image 20260516093833.png]]
2. In the SCAP Workbench window, navigate to **File** > **Save Customization Only**.
    
3. Name the customization "custom_profile_1.xml", and click **Save**.
    

### Scan the Localhost with a Custom Profile

1. In the SCAP Workbench window, set the _Target_ to **Local Machine**.
2. Click the **Scan** button at the bottom to start a scan using the custom profile.
3. Once the scan finished, click the **Close** button in the _Diagnostics_ window.
   ![[Pasted image 20260516094054.png]]
4. Click **Save Results** at the bottom, and select **HTML Report**.
5. Name the report "scan_results.html", and click **Save**.
pretty much the same as before think we can merge these two labs

the report
![[Pasted image 20260516094353.png]]
## Conclusion

Congratulations on completing this hands-on lab!%%