---
tags:
  - Bash
---

> [!NOTE] Module 9 - Assignment
> **Problem Statement:**
> You work for xyz organization. Your job work is to manage Linux-based servers.
> 
> You have been asked to:
> 1. Install MySQL on CentOS
> 2. Create a database with two tables - table1 & table2
> 3. Create a shell script which can be used to insert 4 rows of data into table1 & table2
> 

Update packages:
```bash
sudo yum update -y
```

Install mysql:
```bash
sudo yum install mysql-server -y
```

Start the service and confirm it is up
```bash
sudo systemctl start mysqld
sudo systemctl status mysqld
```
![[Pasted image 20230820161713.png]]

Run `mysql_secure_installation` for security-related operations like configuring a password
```bash
sudo mysql_secure_installation
```

Log in to the MySQL shell as the root user: `mysql -u root -p` using previously configured password  
![[Pasted image 20230820163104.png]]

#### Step 1: Create a Database
```mysql
CREATE DATABASE mydatabase;
```
#### Step 2: Create Tables
```mysql
USE mydatabase;  

CREATE TABLE table1 (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(255),   
  age INT 
);  
  
CREATE TABLE table2 (
  id INT PRIMARY KEY AUTO_INCREMENT,
  product VARCHAR(255),   
  price FLOAT
);
```
![[Pasted image 20230820163828.png]]


### 3. Create a Shell Script for Inserting Data:

Created script named `mysql_script.sh` with the following content using `vi`:
```bash
#!/bin/bash

# Log in to the MySQL server and insert data into table1 and table2
mysql -u root -p -D 'mydatabase' <<EOF

INSERT INTO table1 (name, age) VALUES ('Alice', 30);
INSERT INTO table1 (name, age) VALUES ('Bob', 40);
INSERT INTO table1 (name, age) VALUES ('Charlie', 50);
INSERT INTO table1 (name, age) VALUES ('Dave', 60);

INSERT INTO table2 (product, price) VALUES ('Laptop', 1000);
INSERT INTO table2 (product, price) VALUES ('Phone', 500);
INSERT INTO table2 (product, price) VALUES ('TV', 1500);
INSERT INTO table2 (product, price) VALUES ('Camera', 800);

EOF
```

Giving script execution permissions:
```bash
chmod +x insert_data.sh
```
![[Pasted image 20230820164634.png]]

Executing the script:
```bash
./insert_data.sh
```
Prompts for password  
![[Pasted image 20230820165002.png]]


**Confirming input:**
1. Logging to MySQL shell as the root user:
```bash
mysql -u root -p
```
*prompted for password*

2. Switch to database:
```mysql
USE mydatabase;
```

3. Checking data in `table1`:
```mysql
SELECT * FROM table1;
```

4. Checking data in `table2`:
```mysql
SELECT * FROM table2;
```
![[Pasted image 20230820165153.png]]


