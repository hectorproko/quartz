---
tags:
  - AWS
title: Migrating MySQL to PostgreSQL using AWS Database Migration Service
---
<!--
**Mini-Project: Streamlined Data Migration with AWS Database Migration Service!** In this hands-on project, I ventured into the realm of database migration using AWS's Database Migration Service (DMS). My task was to migrate a MySQL database to a PostgreSQL database hosted on RDS, ensuring a seamless transfer of data. The process involved setting up an RDS MySQL database, creating a replication instance, configuring source and target endpoints, executing the migration task, and verifying the integrity of the migrated data. This project allowed me to practically implement and understand the intricacies of migrating databases across different DBMS, enhancing my proficiency in AWS DMS and cloud migration strategies.
-->
 

> [!info] Module 11: Migration Assignment - 3
> **Problem Statement:**
> You work for XYZ Corporation. Your corporation wants to move their infrastructure to the cloud to improve the performance and availability of the application hosted. You are given the opportunity to accomplish the tasks for successful migration. 
> 
> **Tasks To Be Performed:**
> 1. Launch an RDS MySQL database. Login and insert some data into it. 
> 2. Use Database Migration (DMS) System to migrate the MySQL database into an RDS PostgreSQL database.




### MySQL Database Setup and Data Generation

We set up an RDS instance with the MySQL Engine and an Aurora Cluster for Postgres. MySQL serves as our source database, while Postgres acts as our target.
<br>![[Pasted image 20231018154908.png]]

I will connect to these databases using an EC2 instance by selecting the "Set up EC2 connection" option for each database.
<br>![[Pasted image 20231017154145.png]]

#### Connecting to the MySQL Instance

The provided MySQL script will generate a database and some sample data to migrate.
```mysql
CREATE DATABASE testdb;
USE testdb;
CREATE TABLE sample (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(255));
INSERT INTO sample (name) VALUES ('data1'), ('data2'), ('data3');
```

Connecting to the MySQL instance hosted on Amazon RDS.
```
mysql -h mysql.cnk8gecauqro.us-east-1.rds.amazonaws.com -P 3306 -u admin -p
```
<br>![[Pasted image 20231018155600.png]]

### Creating a Replication Instance

To begin the data migration process using the AWS Database Migration Service (DMS), I first need to create a Replication Instance. Here's the step-by-step procedure:

1. Start by searching for "DMS" in the AWS Management Console.
2. Once inside DMS, I navigate to the `Migrate data` section.
3. Then, I choose `Replication instances`.
4. On this screen, I'll click on the button titled "Create replication instance".

<br>![[Pasted image 20231017164539.png]]

### Setting Up Source and Target Endpoints

To set up the endpoints for our migration process, here's what I do:

For the MySQL Source Endpoint: 

1. In the AWS DMS console, I navigate to the left panel and click on `Migrate data`.
2. Next, I select `Endpoints`.
3. I then click on the "Create endpoint" button.
4. In the setup, I ensure I select MySQL as the source type and provide the necessary database details.

For the PostgreSQL Target Endpoint: 

1. Still within the `Endpoints` section, I click on another "Create endpoint" button.
2. This time, I set it up for our PostgreSQL database, specifying it as our target.
3. I provide the required Amazon Aurora PostgreSQL details and finish the configuration.

<br>![[Pasted image 20231018154846.png]]

### Database Migration Task
  
To initiate the actual data migration process, here's what I'm doing: 

1. In the AWS DMS console, I head over to the left panel and select `Migrate data`.
2. I then choose `Database migration tasks`.
3. To kick off the migration, I click on the "Create task" button.
4. I pick the previously created replication instance
5. I ensure that the correct source (MySQL) and target (PostgreSQL) endpoints are selected.
6. With the task configured, I begin the migration. As seen in the dashboard, the task with the identifier `mysql-postgres` has successfully completed a full load, transferring 100% of the data from the MySQL source to the PostgreSQL target. This confirms our data has been moved successfully.
<br>![[Pasted image 20231018154924.png]]

### Verifying Data Migration in PostgreSQL

To verify the data migration, I'm connecting to the PostgreSQL instance using the provided command. This will allow me to access the PostgreSQL database and inspect the tables and data to ensure everything was migrated accurately from the MySQL source.
```
psql -h postgres2-instance-1.cnk8gecauqro.us-east-1.rds.amazonaws.com -U postgres
```

> [!success]
> <br>![[Pasted image 20231018155052.png]]






