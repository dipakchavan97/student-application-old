
# Student Application Setup Guide


---

## Requirements
- **Ports:** `8080`, `22`, `3306`
- **Applications/Commands:** `git`, `unzip`

---

## Database Setup (MySQL/MariaDB)

### Install MariaDB
```bash
# yum install mariadb105-server -y
# systemctl start mariadb
# systemctl enable mariadb
# mysql_secure_installation
````

* Default username: **root**
* Set a password for the root user.
  Example login:

```bash
mysql -u root -p1234
```

### Create Database & Table

```sql
-- Create Database
CREATE DATABASE studentapp;

-- Switch to database
USE studentapp;

-- Create Table
CREATE TABLE IF NOT EXISTS students (
    student_id INT NOT NULL AUTO_INCREMENT,
    student_name VARCHAR(100) NOT NULL,
    student_addr VARCHAR(100) NOT NULL,
    student_age VARCHAR(3) NOT NULL,
    student_qual VARCHAR(20) NOT NULL,
    student_percent VARCHAR(10) NOT NULL,
    student_year_passed VARCHAR(10) NOT NULL,
    PRIMARY KEY (student_id)
);
```

Exit MySQL:

```bash
exit
```

---

## Install Java

```bash
yum install java-1.8* -y
```

---

## Install Tomcat

1. **Download Tomcat (v8.5.93)**

```bash
wget https://dlcdn.apache.org/tomcat/tomcat-8/v8.5.93/bin/apache-tomcat-8.5.93.zip
```

2. **Unzip & Navigate**

```bash
unzip apache-tomcat-8.5.93.zip
cd apache-tomcat-8.5.93
```

3. **Make Scripts Executable**

```bash
cd bin
chmod +x *.sh
```

4. **Tomcat Start/Stop**

```bash
./catalina.sh start
./catalina.sh stop
```

---

## Setup Student Application

1. **Clone Repository**

```bash
yum install git -y
cd
git clone https://github.com/abhipraydhoble/Student_App-Project.git
```

2. **Stop Tomcat**

```bash
cd apache-tomcat-8.5.93/bin
./catalina.sh stop
```

3. **Copy Application Files**

```bash
cp Student_App-Project/student.war apache-tomcat-8.5.93/webapps/
cp Student_App-Project/mysql-connector.jar apache-tomcat-8.5.93/lib/
```

4. **Modify Tomcat `context.xml`**

```bash
cd apache-tomcat-8.5.93/conf
vim context.xml
```

Add the following at **line 21**:

```xml
<Resource name="jdbc/TestDB" auth="Container" type="javax.sql.DataSource"
          maxTotal="100" maxIdle="30" maxWaitMillis="10000"
          username="USERNAME" password="PASSWORD" driverClassName="com.mysql.jdbc.Driver"
          url="jdbc:mysql://DB-ENDPOINT:3306/DATABASE"/>
```

Replace:

* `USERNAME` → MySQL username (e.g., `root`)
* `PASSWORD` → MySQL password (e.g., `1234`)
* `DB-ENDPOINT` → Database host (e.g., `localhost`)
* `DATABASE` → Database name (e.g., `studentapp`)

5. **Start Tomcat**

```bash
cd apache-tomcat-8.5.93/bin
./catalina.sh start
```

---

## Access Application

* [http://localhost:8080/student](http://localhost:8080/student)
* `http://<SERVER-IP>:8080/student`

---

✅ Your **Student Application** should now be running!


graph TD
    classDef public fill:#e2f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef private fill:#fff3e0,stroke:#e65100,stroke-width:2px;
    classDef db fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;
    classDef internet fill:#f3e5f5,stroke:#6a1b9a,stroke-width:2px;

    User((End User)):::internet

    subgraph AWS Cloud ["☁️ AWS Cloud"]
        subgraph VPC ["VPC-3-tier (192.168.0.0/16)"]
            
            IGW[Internet Gateway]:::internet
            
            subgraph Public_Tier ["Public Subnet (192.168.1.0/24)"]
                NAT[NAT Gateway]:::public
                Nginx["Nginx Server (Ubuntu)<br/>Web Tier / Bastion<br/>Ports: 80, 22"]:::public
            end

            subgraph App_Tier ["Private Subnet (192.168.2.0/24)"]
                Tomcat["Tomcat Server (Java/WAR)<br/>App Tier<br/>IP: 192.168.2.113<br/>Ports: 8080, 22"]:::private
            end

            subgraph DB_Tier ["Private Subnet (192.168.3.0/24)"]
                RDS[("RDS MySQL (database-1)<br/>Database Tier<br/>Port: 3306")]:::db
            end

            %% Network Routing
            IGW --- |Internet Access| Nginx
            NAT -.-> |Outbound Internet| Tomcat
            
            %% Traffic Flow
            Nginx ==>|Proxy Pass http://192.168.2.113:8080| Tomcat
            Tomcat ==>|JDBC Connection| RDS
        end
    end

    User ==>|HTTP Request (Port 80)| IGW



Do you want me to also **add architecture flow diagram + troubleshooting steps** into this README so it looks more professional?
```
