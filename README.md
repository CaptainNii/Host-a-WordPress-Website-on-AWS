# 🖥️ Host a WordPress Website on AWS

## 📌 Project Overview

This project demonstrates how to deploy and host a **highly available, secure, and scalable WordPress website** on **Amazon Web Services (AWS)** using a combination of AWS services and DevOps best practices.

The solution is designed to ensure:

* **High availability** using multiple Availability Zones
* **Scalability** with an Auto Scaling Group
* **Security** through network segmentation and HTTPS encryption
* **Shared storage** using Amazon EFS
* **Managed database** using Amazon RDS MySQL

---

## 📐 Architecture Overview

The architecture leverages the following AWS components:

1. **Virtual Private Cloud (VPC)** – Custom VPC with public and private subnets across two Availability Zones.
2. **Internet Gateway (IGW)** – Provides internet access to resources in public subnets.
3. **Security Groups** – Network firewalls controlling inbound and outbound traffic.
4. **Public Subnets** – Hosting the NAT Gateway and Application Load Balancer.
5. **Private Subnets** – Hosting EC2 web servers, RDS database, and EFS.
6. **EC2 Instance Connect Endpoint** – Secure connection to both public and private resources.
7. **NAT Gateway** – Allows private instances to access the internet.
8. **Application Load Balancer (ALB)** – Distributes incoming requests across multiple EC2 instances.
9. **Auto Scaling Group (ASG)** – Automatically adjusts the number of EC2 instances.
10. **Amazon EFS** – Shared file system for WordPress uploads and media.
11. **Amazon RDS (MySQL)** – Managed database for WordPress.
12. **AWS Certificate Manager (ACM)** – Secures communication via HTTPS.
13. **Amazon SNS** – Notifications for Auto Scaling events.
14. **Amazon Route 53** – DNS management for the domain.

---

## 📂 Repository Contents

* **Reference Architecture Diagram** – Visual overview of the AWS setup.
* **Manual Installation Script** – Script to set up WordPress on a single EC2 instance.
* **Auto Scaling Launch Template Script** – Script for automated deployments.

---

## ⚙️ Prerequisites

Before deployment, ensure you have:

* AWS CLI configured (`aws configure`)
* A domain name in **Route 53**
* An EFS file system created in the same VPC
* An RDS MySQL instance set up
* Proper IAM permissions for EC2, EFS, RDS, and ALB

---

## 🚀 Deployment Guide

### **1️⃣ Manual WordPress Installation on EC2**

**Replace** `fs-xxxxxxxxxxxxxx` with your EFS DNS name.

```bash
# Switch to root
sudo su

# Update packages
sudo yum update -y

# Create HTML directory
sudo mkdir -p /var/www/html

# Set EFS variable
EFS_DNS_NAME=fs-064e9505819af10a4.efs.us-east-1.amazonaws.com

# Mount EFS
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport "$EFS_DNS_NAME":/ /var/www/html

# Install Apache
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install PHP 8 and extensions
sudo dnf install -y \
php php-cli php-cgi php-curl php-mbstring php-gd php-mysqlnd php-gettext php-json php-xml \
php-fpm php-intl php-zip php-bcmath php-ctype php-fileinfo php-openssl php-pdo php-tokenizer

# Install MySQL server
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install -y mysql-community-server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Set permissions
sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www
sudo find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;
sudo chown apache:apache -R /var/www/html

# Download and configure WordPress
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
sudo vi /var/www/html/wp-config.php

# Restart Apache
sudo service httpd restart
```

---

### **2️⃣ Auto Scaling Launch Template Script**

Use this script in your ASG launch template’s **User Data** field:

```bash
#!/bin/bash
sudo yum update -y

# Install Apache
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install PHP and extensions
sudo dnf install -y \
php php-cli php-cgi php-curl php-mbstring php-gd php-mysqlnd php-gettext php-json php-xml \
php-fpm php-intl php-zip php-bcmath php-ctype php-fileinfo php-openssl php-pdo php-tokenizer

# Install MySQL server
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install -y mysql-community-server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Mount EFS
EFS_DNS_NAME=fs-02d3268559aa2a318.efs.us-east-1.amazonaws.com
echo "$EFS_DNS_NAME:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a

# Set permissions
chown apache:apache -R /var/www/html

# Restart Apache
sudo service httpd restart
```

---

## 🔐 Security Best Practices

* Restrict **SSH access** to specific IPs in the Security Group.
* Use **ACM certificates** for HTTPS.
* Keep **RDS private** with no public access.
* Regularly **patch and update** EC2 instances.
* Use **IAM roles** instead of hardcoding credentials.

---

## 📢 Notifications

Amazon SNS is configured to send notifications for scaling activities in the Auto Scaling Group.

---

