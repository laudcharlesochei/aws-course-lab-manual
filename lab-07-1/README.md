# Lab 7-1: Monolithic Application Analysis

## Overview
In this lab, you will analyze the existing monolithic coffee suppliers application to understand its architecture, runtime environment, and database connectivity before beginning the microservices migration.

---

## Part 1: Initial Application Verification

### Task 1.1: Access and Test the Monolithic Application

#### Step 1: Navigate to EC2 Console
1. Go to the **AWS Management Console**
2. Search for **"EC2"** in the services search bar and select it
3. In the left sidebar, click **"Instances"**

#### Step 2: Locate the Application Server
- Look for an instance named **"MonolithicAppServer"** in the instances list
- Check the **"Name"** column if not immediately visible

#### Step 3: Copy Public IP Address
```text
1. Select the MonolithicAppServer instance checkbox
2. In the details panel, locate "Public IPv4 address"
3. Click the copy icon next to the address
```

#### Step 4: Test Application Access
- Open a new browser tab
- Paste the IP address with `http://` prefix (not https)
- Example: `http://54.210.134.58` (use your actual IP)
- Handle security warning by clicking **"Advanced"** → **"Proceed to [IP]"**

#### Step 5: Explore Application Features
- Click **"List of suppliers"** (URL: `/suppliers`)
- Click to **add a new supplier** (URL: `/supplier-add`)
- **Edit an existing supplier** (URL: `/supplier-update/1`)
- Test form submissions and data persistence

---

## Part 2: Server Connection and Process Analysis

### Task 2.1: Connect to EC2 Instance

#### Step 1: Establish Connection
1. In EC2 Console, select **MonolithicAppServer** instance
2. Click **"Connect"** button at top
3. Select **"EC2 Instance Connect"** tab
4. Click **"Connect"** - new terminal tab will open

### Task 2.2: Analyze Running Processes

#### Step 1: Check Port 80 Activity
```bash
sudo lsof -i :80
```

**Expected Output:**
```text
COMMAND  PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
node    1234 ubuntu   21u  IPv4  12345      0t0  TCP *:http (LISTEN)
```

**Key Observations:**
- **Command:** `node` - Node.js application
- **PID:** `1234` - Process ID (yours will differ)
- **User:** `ubuntu` - Running user account
- **Port:** `http` (port 80)

#### Step 2: Analyze Node.js Processes
```bash
ps -ef | head -1; ps -ef | grep node
```

**Expected Output:**
```text
UID        PID  PPID  C STIME TTY          TIME CMD
ubuntu    1234     1  0 10:30 ?        00:00:10 node /home/ubuntu/resources/codebase_partner/index.js
```

**Key Observations:**
- **UID:** `ubuntu` - Process owner
- **PID:** `1234` - Matches lsof output
- **CMD:** Shows application entry point: `index.js`

---

## Part 3: Application Structure Analysis

### Task 3.1: Explore Application Directory

#### Step 1: Navigate to Application
```bash
cd ~/resources/codebase_partner
```

#### Step 2: List Application Files
```bash
ls -la
```

**Expected Structure:**
```text
total 120
drwxrwxr-x 6 ubuntu ubuntu  4096 Mar 15 10:00 .
drwxr-xr-x 3 ubuntu ubuntu  4096 Mar 15 09:55 ..
-rw-rw-r-- 1 ubuntu ubuntu   287 Mar 15 10:00 app.js
-rw-rw-r-- 1 ubuntu ubuntu  1234 Mar 15 10:00 index.js
-rw-rw-r-- 1 ubuntu ubuntu   567 Mar 15 10:00 package.json
drwxrwxr-x 2 ubuntu ubuntu  4096 Mar 15 10:00 views
drwxrwxr-x 2 ubuntu ubuntu  4096 Mar 15 10:00 controllers
drwxrwxr-x 2 ubuntu ubuntu  4096 Mar 15 10:00 models
drwxrwxr-x 2 ubuntu ubuntu  4096 Mar 15 10:00 config
```

### Task 3.2: Critical Analysis Questions

**Answer these based on your observations:**

1. **How is the application running?**
   - Node.js process directly on EC2 instance
   - Listening on port 80 (HTTP)
   - Running under 'ubuntu' user account

2. **How was it installed?**
   - Likely deployed via scripts/manual installation
   - Node.js runtime pre-installed
   - Code located at: `/home/ubuntu/resources/codebase_partner/`

3. **What prerequisites were needed?**
   - Node.js runtime environment
   - NPM packages from `package.json`
   - Database connectivity libraries

4. **Where is data stored?**
   - Amazon RDS MySQL database (external)
   - Configuration in `config/` directory

---

## Part 4: Database Connectivity Analysis

### Task 4.1: Locate RDS Database

#### Step 1: Find Database Endpoint
1. Open new browser tab to **RDS Console**
2. Search for **"RDS"** and select it
3. Click **"Databases"** in left sidebar
4. Click on the database instance
5. Copy the **Endpoint** from "Connectivity & security" section

**Endpoint Format:** `database-1.xxxxxxxx.us-east-1.rds.amazonaws.com`

### Task 4.2: Test Database Connection

#### Step 1: Verify Database Accessibility
```bash
nmap -Pn YOUR_RDS_ENDPOINT
```

**Replace `YOUR_RDS_ENDPOINT` with actual endpoint**

**Expected Output:**
```text
Starting Nmap 7.60 ( https://nmap.org ) at 2024-03-15 11:00 UTC
Nmap scan report for database-1.xxxxxxxx.us-east-1.rds.amazonaws.com (10.0.1.123)
Host is up (0.0020s latency).
Not shown: 999 filtered ports
PORT     STATE SERVICE
3306/tcp open  mysql
```

**Confirmation:** MySQL database accessible on port 3306

#### Step 2: Connect to MySQL Database
```bash
mysql -h YOUR_RDS_ENDPOINT -u admin -p
```

**When prompted for password, enter:**
```text
lab-password
```

**Successful Connection Indication:**
```text
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 12345
Server version: 8.0.28 Source distribution

mysql>
```

### Task 4.3: Explore Database Schema

#### Step 1: Show Databases
```sql
SHOW DATABASES;
```

#### Step 2: Use COFFEE Database
```sql
USE COFFEE;
```

#### Step 3: Show Tables
```sql
SHOW TABLES;
```

#### Step 4: View Supplier Data
```sql
SELECT * FROM suppliers;
```

**Expected Data:**
```text
+----+-------------------+------------------+----------------+------------------------+
| id | name              | email            | phone          | description           |
+----+-------------------+------------------+----------------+------------------------+
|  1 | Mountain Coffee   | info@mtncoffee.com | 555-0101       | Organic mountain beans|
|  2 | Valley Roasters   | contact@valleyroast.com | 555-0102 | Premium dark roast   |
+----+-------------------+------------------+----------------+------------------------+
```

### Task 4.4: Clean Up
```sql
EXIT;
```

Close EC2 Instance Connect tab and any application browser tabs.

---

## Part 5: Documentation and Findings

### Key Architecture Findings

| Component | Observation | Implication |
|-----------|-------------|-------------|
| **Application Runtime** | Node.js on EC2, port 80, ubuntu user | Traditional monolithic deployment |
| **Process Confirmation** | Matching PIDs from lsof/ps commands | Application actively serving requests |
| **Database Connectivity** | RDS MySQL, COFFEE.suppliers table | External data persistence |
| **Architecture Pattern** | Monolithic design | Combined web server + application logic |

### Critical Architecture Understanding
- **Monolithic Design:** All features in single application
- **Combined Concerns:** Web server and business logic intertwined
- **External Data:** Database separate from application server
- **Direct Deployment:** Application runs directly on EC2 instance

---

## Troubleshooting Guide

### Instance Location Issues
**If you can't find MonolithicAppServer:**
1. Verify lab is active (clicked "Start Lab")
2. Check correct AWS region (usually us-east-1)
3. Confirm instance state is "running"
4. Use EC2 filter: Type "MonolithicAppServer" in search

### Connection Problems
- **nmap shows port 3306 closed:** Check RDS instance status
- **MySQL connection fails:** Verify endpoint and password
- **No processes on port 80:** Application may not be running
- **COFFEE database missing:** Application not initialized properly

### Expected Instance Properties
- **Name:** `MonolithicAppServer`
- **Location:** `LabVPC` (default VPC for lab)
- **Status:** Pre-installed and running
- **Access:** Automatically created when lab starts

---

## Completion Checklist
- [ ] Successfully accessed web application via public IP
- [ ] Connected to EC2 instance via Instance Connect
- [ ] Identified Node.js process running on port 80
- [ ] Explored application directory structure
- [ ] Located and connected to RDS database
- [ ] Verified database schema and sample data
- [ ] Documented architectural observations

**This completes the comprehensive analysis of the monolithic application architecture.**
