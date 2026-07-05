---
tags:
linkedin: "False"
quartz: "False"
refactored: "False"
pluralsight: "True"
hands-on: "True"
completed: "False"
hardlinked: "True"
title: SELinux  Index
---
# 🚧 🔧

### Section 1: SELinux Modes and Boolean-Based Service Configuration
- [[Rotating Between the 3 SELinux Modes]]
- [[Configuring SELinux - Enabling Service Communication and Enforcing Security Policy]] (Zabbix/httpd) comes with prerequise stuff tht we dotn mention

### Section 2: Security Contexts
- [[Display and Restore File and Directory Security Contexts with SELinux]] comes with prerequise stuff tht we dotn mention
- [[Troubleshooting SELinux on Files and Directories]]
- [[Troubleshooting SELinux Issues]]
- [[Enhancing SELinux and Compliance with RHEL System Roles]]

### Section 3: Log Analysis & Troubleshooting
- [[❌Converting SELinux Log File with sealert and Finding Entries for HTTP in the Log File]]

### Section 4: Ports & Labels on Apache (merged)
 - [[Change Apache Port and Give It a Proper SELinux Label]]
 - [[Finding a Problem Caused by a Misconfiguration of SELinux and Troubleshooting the Issue]] (two violations, port + file context)
 - [[Resolving SELinux Issues]] (one violation, port only)

### Section 5: Booleans & Ports on Other Services
- [[Working with SELinux Booleans and Ports]] (MariaDB)

### Section 6: SELinux Users & sudo
This section grows into a 3-lab progression, all about SELinux user identities (separate from booleans/contexts/ports):  
- [[Working with SELinux Users and Roles]]
- [[Creating Confined Users in SELinux]]
- [[Granting sudo Privileges to Confined Users]]

### Section 7: Writing a Custom SELinux Policy (capstone)
- [[Writing a Custom SELinux Policy]] (apachelogger)


