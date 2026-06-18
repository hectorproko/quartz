---
tags:
  - "#Terraform"
  - IaC
hardlinked: "True"
linkedin: "False"
quartz: "False"
refactored: "True"
darey.io: "True"
hands-on: "True"
completed: "True"
title: "Automating AWS Infrastructure with Terraform - Part 4: Terraform Cloud"
---

> **Part:** 4 of 4  
> **Topics:** Terraform Cloud · VCS Integration · Workspaces · Environment Variables · Branch-based Environments · Auto Apply · Notifications

---

## **Overview**

In [[PART3_PROJECT18_Backends|Part 3: Remote Backends & Modules]] I ran Terraform from my local workstation against a remote S3 backend. This final part moves the execution itself into [**Terraform Cloud**](https://www.terraform.io/cloud), HashiCorp's managed platform for running Terraform.

Terraform Cloud runs `plan` and `apply` on remote, consistent infrastructure rather than my laptop. It stores state for me, integrates directly with a VCS provider (so runs trigger from Git commits), manages variables and secrets securely, and supports per-environment workspaces. The goal here is to connect the GitHub repository, create a workspace, wire up branch-based environments (`dev`, `test`, `prod`), and set up notifications.

---

## Step 1 - Create a Terraform Cloud Account

I create an account and **start from scratch** (rather than importing an existing setup).

![test|500](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/welcome.png)

![test|500](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/organization.png)

![[Markdown_Logo-5.png|500]]


---

## Step 2 - Connect a VCS Provider & Select the Repository

Terraform Cloud connects to a version control provider so it can pull configuration and trigger runs on commits. I connect the provider, then choose the repository to use.

![test|500](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/workspace2.png)

![[Markdown_Logo-6.png|300]]

Then I select the repo:

![[Markdown_Logo-4.png|300]]

![test|500](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/cloud.png)



---

## Step 3 - Create the Workspace

When creating the workspace:

- **Make sure to select "Environment variable"** when adding sensitive values (such as AWS credentials) so they're injected into the run environment securely.
- The **workspace name** is `terraform-cloud` — it's set automatically to match the repository name.

![](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/variables.png)

This is how it looks navigating back to the workspace overview:

![](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/workspaces.png)

A walkthrough of the run flow:

![](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/terraform_cloud_apply.gif)

![test|150](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/created.png)

![](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/terraform_cloud_destroyed.gif)

![test|300](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/triggered.png)

![](https://raw.githubusercontent.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/main/images/terraform_cloud_autoPlanning.gif)

---

## Step 4 - Configure Branch-based Environments

To separate environments, I configure three branches in the `terraform-cloud` repository:

|Branch|Environment|
|---|---|
|`dev`|Development|
|`test`|Testing|
|`prod`|Production|

Each environment maps to its own branch, so changes flow `dev` → `test` → `prod` as they're promoted.

---

## Step 5 - Auto-trigger Runs for `dev` Only

I make the necessary configuration so runs trigger **automatically only for the `dev` environment**. Development changes get fast feedback, while `test` and `prod` stay gated behind manual control.

---

## Step 6 - Email & Slack Notifications

I set up notifications for key run events (for example, a started `plan` or an errored run) and test them.

**Navigate to:** Settings → Notifications → Create a Notification

I create both an **Email** and a **Slack** notification so the team is alerted when a run starts or fails.

---

## Step 7 - Create the `terraform-cloud-dev` Workspace

For the development environment I create a dedicated workspace:

|Setting|Value|
|---|---|
|Workspace Name|`terraform-cloud-dev`|
|Apply Method|Auto apply|
|VCS branch|`dev`|

With **Auto apply** enabled and the VCS branch set to `dev`, any commit pushed to the `dev` branch triggers a run that plans **and** applies without manual confirmation.

---

## Step 8 - Trigger a Run

Finally, I create a new run to confirm everything is wired up: pushing to the `dev` branch (or queuing manually) kicks off a remote `plan`, and with auto-apply enabled on `terraform-cloud-dev` it proceeds to `apply`.

---

## Summary

Part 4 moved Terraform execution off the local machine and into a managed, collaborative workflow:

- ✅ Created a Terraform Cloud account from scratch
- ✅ Connected a VCS provider and selected the repository
- ✅ Created a VCS-backed workspace with credentials stored as environment variables
- ✅ Configured `dev` / `test` / `prod` branch-based environments
- ✅ Set auto-triggered runs for `dev` only
- ✅ Added and tested Email + Slack notifications
- ✅ Created a `terraform-cloud-dev` workspace with auto-apply on the `dev` branch

This completes the four-part series: from a manually built AWS environment, to fully automated Infrastructure as Code, refactored into modules, backed by remote state, and now driven from Terraform Cloud.

---

_GitHub Repository: [AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4)_

<!--
ORIGINAL RAW NOTES (Project 19) - preserved for reference

TERRAFORM CLOUD
- Created an account
- Start from scratch
  [media/Markdown_Logo-5.png]
  [media/Markdown_Logo-6.png]
  [media/Markdown_Logo-5.png]
  [media/Markdown_Logo-5.png]
  [media/Markdown_Logo-6.png]
- Lets create the repo to choose
  [media/Markdown_Logo-4.png]
  [media/Markdown_Logo-5.png]
  [media/Markdown_Logo-5.png]
- Make sure you select Environment variable
- Workspace name is terraform-cloud like the repo name is automatic
- This is how it looks if you navigate back
  [media/Markdown_Logo-5.png]
  [media/Markdown_Logo-1.gif]
  [media/Markdown_Logo-5.png]
  [media/Markdown_Logo.gif]
  [media/Markdown_Logo-5.png]
  [media/Markdown_Logo-2.gif]

- Configure 3 branches in your terraform-cloud repository for dev, test, prod environments
- Make necessary configuration to trigger runs automatically only for dev environment
- Create an Email and Slack notifications for certain events (e.g. started plan or errored run) and test it
  Settings > Notifications > Create a Notification

- Create a new Workspace
    Workspace Name: terraform-cloud-dev
    Step 4: Configure settings
    Apply Method: Auto apply
    VCS branch: dev
- Need to create a new run
-->

<!--
# AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-4
Project 16-19 Terraform
*==TERRAFORM CLOUD* ==
### AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM. PART 4 - TERRAFORM CLOUD  

Created an account  

Start from scratch  

![Markdown Logo](media/Markdown_Logo-5.png)  

![Markdown Logo](media/Markdown_Logo-6.png)  

![Markdown Logo](media/Markdown_Logo-5.png)  


![Markdown Logo](media/Markdown_Logo-5.png)  


![Markdown Logo](media/Markdown_Logo-6.png)  

Lets created the repo to choose  

![Markdown Logo](media/Markdown_Logo-4.png)  

![Markdown Logo](media/Markdown_Logo-5.png)  

![Markdown Logo](media/Markdown_Logo-5.png)  

Make sure you select Environment variable  

Worspace name is terraform-cloud like the repo name is automatic  



This is how it looks if you navigate back  
![Markdown Logo](media/Markdown_Logo-5.png)  


![Markdown Logo](media/Markdown_Logo-1.gif)  


![Markdown Logo](media/Markdown_Logo-5.png)  


![Markdown Logo](media/Markdown_Logo.gif)  


![Markdown Logo](media/Markdown_Logo-5.png)  


![Markdown Logo](media/Markdown_Logo-2.gif)  


Configure 3 branches in your `terraform-cloud` repository for `dev`, `test`, `prod` environments  

Make necessary configuration to trigger runs automatically only for `dev` environment  

Create an Email and Slack notifications for certain events (e.g. started `plan` or errored run) and test it  

Settings > Notifications > Create a Notification  


Create a new Workspace  
	Workspace Name: terraform-cloud-dev  
	Step 4: Configure settings  
	Apply Method: Auto apply  
	VCS branch: dev  
	
Need to create a new run  

