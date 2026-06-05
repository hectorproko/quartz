---
tags:
  - AWS
  - CloudFormation
  - IaC
title: Automating S3 Bucket Deployment and Versioning with CloudFormation in AWS
---
<!--
**Mini-Project: Automating S3 Bucket Deployment with AWS CloudFormation!** In this project, I used AWS CloudFormation to automate the deployment of an S3 bucket, named “intellipaat-hector”, specifically focusing on enabling versioning for enhanced data management and security. I wrote a YAML CloudFormation template to define the bucket's properties and configurations. Upon deploying the template, I successfully created the bucket with versioning enabled, demonstrating my capability in using infrastructure as code to streamline AWS resource deployment. This project was a practical exercise in enhancing operational efficiency using AWS CloudFormation.
-->

> [!info] Module 8: CloudFormation Assignment - 1 
> **Training Problem Statement:** 
> You work for XYZ Corporation. Your team is asked to deploy similar architecture multiple times for testing, development, and production purposes. Implement CloudFormation for the tasks assigned to you below. 
> 
> **Tasks To Be Performed:** 
> 1. Create a template which can create an S3 bucket named “Intellipaat-” 
> 2. The template should be able to enable versioning for the bucket created.


1. **Crafting the CloudFormation Template:** 
I fired up my trusted text editor and initiated a new document. I named this file `s3_bucket_template.yaml`. 

At the very beginning of my file, I penned down the following to define the template's version and its purpose:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CloudFormation template to create an S3 bucket with versioning enabled.
```

2. **Laying Out the Resource Details:** 
Diving into the core part, I began with the `Resources` section to define my S3 bucket:
```yml
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: intellipaat-hector
      VersioningConfiguration:
        Status: Enabled
```

3. **Saving and Deploying My Work:** 
- Once satisfied with the configurations, I saved the `s3_bucket_template.yaml` file. 
- Navigating back to the CloudFormation section in the AWS console, I clicked on "Create Stack". 
- I then uploaded the `s3_bucket_template.yaml` and closely followed the on-screen instructions. 
  <br>![[Pasted image 20231007162942.png]]
  <br>![[Pasted image 20231007163618.png]]

`s3_bucket_template.yaml`
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CloudFormation template to create an S3 bucket with versioning enabled.

Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: intellipaat-hector
      VersioningConfiguration:
        Status: Enabled
```

<br>![[Pasted image 20231007183507.png]]

4. **Validating My Implementation:** 

> [!success]
> As expected, I found my freshly minted bucket, "intellipaat-hector", and confirmed that versioning was indeed enabled.
> 
> <br>![[Pasted image 20231007183820.png]]
> 
> 
