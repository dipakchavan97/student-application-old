# 🏗️ Project: 3-Tier Student Application (Ubuntu Edition)

This guide provides a detailed, step-by-step walkthrough for deploying a 3-tier web application (Web, Application, and Database tiers) on AWS using **Ubuntu** EC2 instances. 

## 🌐 Phase 1: Networking Prerequisite (VPC Setup)
Before provisioning servers, establish a secure, isolated network topology.

### 1. Create the VPC
* **Name:** `VPC-3-tier`
* **CIDR:** `192.168.0.0/16`

### 2. Create Subnets
* **Public-Subnet-Nginx:** `192.168.1.0/24` (Web Tier)
* **Private-Subnet-Tomcat:** `192.168.2.0/24` (App Tier)
* **Private-Subnet-Database:** `192.168.3.0/24` (DB Tier)

### 3. Configure Gateways
* **Internet Gateway (IGW):** Create `IGW-3-tier` and attach it to your VPC.
* **NAT Gateway:** Create `NAT-3-tier` inside your `Public-Subnet-Nginx` and allocate an Elastic IP.

### 4. Configure Route Tables
* **RT-Public:** Associate with the Public Subnet. Add route `0.0.0.0/0` targeting the **IGW**.
* **RT-Private:** Associate with both Private Subnets. Add route `0.0.0.0/0` targeting the **NAT Gateway**.

---

## 🖥️ Phase 2: Compute & Database Provisioning

### 1. Create EC2 Instances (Select Ubuntu 24.04 or 22.04 LTS AMI)
1. **Nginx-Server-Public** * Subnet: Public-Subnet-Nginx
   * Security Group: Allow SSH (22) and HTTP (80) from Anywhere (0.0.0.0/0).
2. **Tomcat-Server-Private**
   * Subnet: Private-Subnet-Tomcat
   * Security Group: Allow SSH (22) and Custom TCP (8080) from the Nginx Server's Security Group.

### 2. Provision the RDS Database
* **Engine:** MySQL (Free Tier)
* **DB Instance Identifier:** `database-1`
* **Credentials:** Username `admin` / Password `Passwd123$`
* **Network:** Select `VPC-3-tier`. Set **Public Access** to `No`.
* **Security Group:** Edit the attached RDS security group to allow inbound MySQL/Aurora (3306) traffic strictly from the Tomcat server's Security Group.

---

## 🗄️ Phase 3: Database Initialization
We will use the public Nginx server as a Bastion Host (Jump Box) to securely configure the private database.

### 1. Connect to the Bastion (Nginx Server)
Connect via SSH to your Nginx public IP. Create your key pair file to access the private instances:
```bash
vim 3-tier-key.pem
# Paste your private key, save (:wq), and restrict permissions:
chmod 400 3-tier-key.pem
```

2. Install MySQL Client and Connect
Install the client tools on the Ubuntu Bastion host:

```Bash
sudo apt update
sudo apt install mysql-client -y```
Connect to your RDS instance (replace <rds-endpoint> with your actual AWS RDS endpoint):

```Bash
mysql -h <rds-endpoint> -u admin -pPasswd123$
```
3. Create the Schema and Table
Execute the following SQL commands:

```Bash
SHOW DATABASES;
CREATE DATABASE studentapp;
USE studentapp;

CREATE TABLE IF NOT EXISTS students(
    student_id INT NOT NULL AUTO_INCREMENT,  
    student_name VARCHAR(100) NOT NULL,  
    student_addr VARCHAR(100) NOT NULL,   
    student_age VARCHAR(3) NOT NULL,      
    student_qual VARCHAR(20) NOT NULL,     
    student_percent VARCHAR(10) NOT NULL,   
    student_year_passed VARCHAR(10) NOT NULL,  
    PRIMARY KEY (student_id)  
);

SHOW TABLES;
exit;```
⚙️ Phase 4: Application Tier Setup (Tomcat)
From the Nginx Bastion, SSH into your private Tomcat server:

```Bash
ssh -i 3-tier-key.pem ubuntu@<private-ip-of-tomcat>```
1. Install Java
Switch to root and install the default Java Development Kit:

```Bash
sudo -i
apt update
apt install default-jdk -y```
2. Download and Extract Tomcat 9
Use the permanent Apache archive link to avoid 404 errors:

```Bash
curl -O [https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.112/bin/apache-tomcat-9.0.112.tar.gz](https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.112/bin/apache-tomcat-9.0.112.tar.gz)
tar -xzvf apache-tomcat-9.0.112.tar.gz -C /opt/```
3. Download Application Payload & DB Driver
```Bash
cd /opt/apache-tomcat-9.0.112/webapps
curl -O [https://s3-us-west-2.amazonaws.com/studentapi-cit/student.war](https://s3-us-west-2.amazonaws.com/studentapi-cit/student.war)

cd ../lib
curl -O [https://s3-us-west-2.amazonaws.com/studentapi-cit/mysql-connector.jar](https://s3-us-west-2.amazonaws.com/studentapi-cit/mysql-connector.jar)```
4. Configure Database Connection
Edit the Tomcat context configuration:

```Bash
cd /opt/apache-tomcat-9.0.112/conf
vim context.xml```
Insert the following JDBC connection string around line 21 (inside the <Context> block). Ensure you replace RDS-ENDPOINT with your actual endpoint.

```XML
<Resource name="jdbc/TestDB" auth="Container" type="javax.sql.DataSource"
          maxTotal="100" maxIdle="30" maxWaitMillis="10000"
          username="admin" password="Passwd123$" driverClassName="com.mysql.jdbc.Driver"
          url="jdbc:mysql://RDS-ENDPOINT:3306/studentapp"/>```
5. Start Tomcat
Bash
```cd ../bin
chmod +x catalina.sh
./catalina.sh start
exit```
🌐 Phase 5: Web Tier Setup (Nginx Reverse Proxy)
You should now be back on your Nginx server.

1. Install Nginx
```Bash
sudo -i
apt update
apt install nginx -y```
2. Remove Default Ubuntu Page
Ubuntu places a default site block that interferes with reverse proxies. Remove it:

```Bash
rm /etc/nginx/sites-enabled/default```
3. Configure the Reverse Proxy
Open the main configuration file:

```Bash
vim /etc/nginx/nginx.conf```
Delete the existing content and paste the following configuration. Make sure to replace <private-ip-of-tomcat> with your Tomcat server's actual private IP:

Nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    gzip on;
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;

    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        
        server_name _;

        # 1. Handle the application traffic and form submissions
        location /student/ {
            proxy_pass http://<private-ip-of-tomcat>:8080/student/;
            
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # 2. Automatically redirect root IP visits to the application path
        location = / {
            return 301 /student/;
        }
    }
}
4. Test and Start Nginx
Bash
nginx -t
systemctl restart nginx
systemctl enable nginx
✅ Phase 6: Validation
Open a web browser and navigate to the Public IP of your Nginx Server.

Nginx will automatically redirect you to /student/.

Fill out the student registration form and click submit.

If the data saves successfully and appears in the table, your Web -> App -> DB traffic flow is fully operational!
