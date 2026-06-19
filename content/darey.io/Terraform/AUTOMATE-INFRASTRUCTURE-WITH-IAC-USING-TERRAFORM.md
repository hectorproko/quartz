---
tags:
  - "#Terraform"
  - IaC
hardlinked: "True"
title: Automating a Multi-Tier AWS Stack with Terraform
darey.io: "True"
refactored: "True"
---

Using Terraform to provisioning project [[Project15 AWS CLOUD SOLUTION FOR 2 COMPANY WEBSITES USING A REVERSE PROXY TECHNOLOGY|AWS Solution for 2 Company Websites using a Reverse Proxy]]  

[[PART1_PROJECT_16|Part 1: Getting Started]]
[[PART2_PROJECT_17|Part 2: Building the Full Stack]]
[[PART3_PROJECT18_Backends|Part 3: Remote Backends & Modules]]
[[PART4_PROJECT19_TerraformCloud|Part 4: Terraform Cloud]]

*Migrated from [Project 16-19 (provisioning project 15)](https://github.com/hectorproko/AUTOMATE-INFRASTRUCTURE-WITH-IAC-USING-TERRAFORM-PART-1-to-4/tree/main/PBL)*

---

[Other](https://github.com/hectorproko/Terraform) 


%%
### Post
Finally got around to refactoring an old Terraform project I'd built a while back. It automates an AWS reverse-proxy architecture I'd originally set up by hand, but the text was rough. So I cleaned it all up and wrote it as a 4-part series that builds step by step: Part 1 lays the foundation (IAM, VPC, variables), Part 2 builds out the full stack (load balancers, auto scaling, EFS, RDS), Part 3 moves state to a remote S3 backend and refactors everything into modules, and Part 4 runs it all from Terraform Cloud. Each part picks up exactly where the last one left off.

`#Terraform #IaC #AWS #DevOps`
%%