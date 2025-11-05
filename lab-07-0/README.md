# Lab 7-1: AWS to GitHub Codespaces Migration

## Lab Overview
This lab guides you through migrating a monolithic coffee suppliers application from AWS EC2 to GitHub Codespaces, establishing RDS database connectivity, and setting up a modern development workflow.

**Lab Duration:** 60-90 minutes  
**Difficulty Level:** Intermediate  
**Prerequisites:** Basic AWS knowledge, GitHub account, understanding of Node.js and MySQL

### Learning Objectives
By completing this lab, you will be able to:
- Migrate applications from AWS EC2 to cloud development environments
- Establish and test RDS database connectivity
- Configure GitHub Codespaces for development
- Set up proper version control and project structure
- Troubleshoot common migration issues

---

## Part 1: AWS Environment Preparation

### Task 1.1: Access AWS Resources

#### Step 1: Locate EC2 Instance
1. Navigate to **AWS Console** → **EC2** → **Instances**
2. Find the **MonolithicAppServer** instance
3. Copy the **Public IPv4 address** (format: `44.199.191.4`)

#### Step 2: Download SSH Key
1. In AWS Academy Lab interface, find **"AWS Details"** panel
2. Click **"labsuser.pem"** to download the SSH key
3. Save to secure location: `D:\MicroserviceApplications\microservice_aws\.ssh\`

### Task 1.2: Configure SSH Access

#### Step 1: Set Up SSH Directory
```bash
# Create SSH directory structure
mkdir -p D:\MicroserviceApplications\microservice_aws\.ssh
```

#### Step 2: Fix PEM File Permissions (Windows)
```cmd
# Run in Admin Command Prompt
icacls "D:\MicroserviceApplications\microservice_aws\.ssh\labsuser.pem" /reset
icacls "D:\MicroserviceApplications\microservice_aws\.ssh\labsuser.pem" /inheritance:r
icacls "D:\MicroserviceApplications\microservice_aws\.ssh\labsuser.pem" /grant:r "%username%:F"

# Verify permissions
icacls "D:\MicroserviceApplications\microservice_aws\.ssh\labsuser.pem"
```

#### Step 3: Create SSH Config File
Create `D:\MicroserviceApplications\microservice_aws\.ssh\config`:

```text
Host monolithic-app-server
    HostName YOUR_EC2_PUBLIC_IP
    User ubuntu
    IdentityFile D:\MicroserviceApplications\microservice_aws\.ssh\labsuser.pem
```

---

## Part 2: GitHub Repository Setup

### Task 2.1: Create GitHub Repository

#### Step 1: Initialize Repository
1. Go to [github.com](https://github.com) and sign in
2. Click **"+"** → **"New repository"**
3. Configure repository settings:
   - **Repository name:** `coffee-suppliers-app`
   - **Description:** "Monolithic Coffee Suppliers Application migrated from AWS"
   - **Visibility:** Private
   - **Initialize with README:** Uncheck
4. Click **"Create repository"**

### Task 2.2: Launch GitHub Codespace

#### Step 1: Create Codespace
1. Navigate to your new repository
2. Click **"Code"** → **"Codespaces"** tab
3. Click **"Create codespace on main"**
4. Wait 2-3 minutes for environment provisioning

#### Step 2: Verify Environment
```bash
# In Codespace terminal
pwd
# Should show: /workspaces/coffee-suppliers-app

# Check available tools
node --version
npm --version
git --version
```

---

## Part 3: Code Transfer and Project Setup

### Task 3.1: Upload SSH Key to Codespaces

#### Step 1: Set Up SSH in Codespace
```bash
# Create .ssh directory
mkdir .ssh

# Set proper permissions
chmod 700 .ssh
```

#### Step 2: Add PEM File
1. Open `labsuser.pem` from your local machine
2. Copy entire contents
3. In Codespace, create new file: `.ssh/labsuser.pem`
4. Paste contents and save
5. Set permissions: `chmod 600 .ssh/labsuser.pem`

### Task 3.2: Transfer Application Code

#### Step 1: Download from EC2 Instance
```bash
# Replace YOUR_EC2_IP with actual IP
scp -r -i .ssh/labsuser.pem ubuntu@YOUR_EC2_IP:/home/ubuntu/resources/codebase_partner/ ./

# Verify transfer completed
ls -la
ls -la codebase_partner/
```

#### Step 2: Reorganize Project Structure
```bash
# Move all files to root directory
mv codebase_partner/* .
rm -rf codebase_partner

# Verify project structure
find . -name "*.js" -o -name "*.json" | head -10
ls -la app/ views/ controllers/ models/ config/
```

---

## Part 4: Database Configuration

### Task 4.1: Locate RDS Database

#### Step 1: Find Database Endpoint
1. **AWS Console** → **RDS** → **Databases**
2. Click on your database instance
3. Copy the **Endpoint** (format: `database-1.xxxxxx.us-east-1.rds.amazonaws.com`)
4. Note credentials:
   - **Username:** `admin`
   - **Password:** `lab-password`
   - **Database:** `COFFEE`

### Task 4.2: Test Database Connectivity

#### Step 1: Install MySQL Client
```bash
# Install MySQL client in Codespace
sudo apt update
sudo apt install mysql-client -y
```

#### Step 2: Test Connection
```bash
# Test connection to RDS (replace with your endpoint)
mysql -h YOUR_RDS_ENDPOINT -u admin -p

# Enter password when prompted: lab-password
```

#### Step 3: Verify Database
```sql
-- Verify database and tables
USE COFFEE;
SHOW TABLES;
SELECT * FROM suppliers;
EXIT;
```

### Task 4.3: Configure Application Database

#### Step 1: Update Config Files

**File:** `app/config/config.js`
```javascript
module.exports = {
  APP_DB_HOST: "your-rds-endpoint.us-east-1.rds.amazonaws.com",
  APP_DB_USER: "admin", 
  APP_DB_PASSWORD: "lab-password",
  APP_DB: "COFFEE"
};
```

**File:** `config/db.config.js`
```javascript
module.exports = {
  HOST: "your-rds-endpoint.us-east-1.rds.amazonaws.com",
  USER: "admin",
  PASSWORD: "lab-password", 
  DB: "COFFEE",
  dialect: "mysql",
  pool: {
    max: 5,
    min: 0,
    acquire: 30000,
    idle: 10000
  }
};
```

---

## Part 5: Application Setup

### Task 5.1: Install Dependencies

#### Step 1: Install Node.js Packages
```bash
# Install project dependencies
npm install

# If package.json is missing, install required packages manually
npm init -y
npm install express mysql2 hbs express-validator
```

### Task 5.2: Configure Application

#### Step 1: Update Port Configuration
**File:** `index.js` (around line 45)
```javascript
// Change from port 80 to 3000 for development
app.listen(3000, () => {
  console.log("Coffee suppliers application is running on port 3000.");
});
```

#### Step 2: Verify Database Connection
**File:** `app/models/db.js`
```javascript
const mysql = require("mysql2");
const dbConfig = require("../../config/db.config.js");

// Create connection pool
const connection = mysql.createPool({
  host: dbConfig.HOST,
  user: dbConfig.USER,
  password: dbConfig.PASSWORD,
  database: dbConfig.DB
});

module.exports = connection;
```

---

## Part 6: Version Control Setup

### Task 6.1: Configure Git

#### Step 1: Set Git Configuration
```bash
# Configure Git (Codespace may have auto-configured)
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Check current status
git status
```

### Task 6.2: Create .gitignore File

```bash
# Create comprehensive .gitignore for Node.js
cat > .gitignore << EOF
node_modules/
.env
.DS_Store
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.vscode/
.idea/
coverage/
dist/
build/
.ssh/
config/local*.js
EOF
```

### Task 6.3: Initial Commit and Push

#### Step 1: Commit Changes
```bash
# Stage all files
git add .

# Check what will be committed
git status

# Initial commit
git commit -m "feat: Migrate monolithic coffee suppliers app to Codespaces

- Complete application code from AWS EC2
- RDS database configuration
- Updated for Codespaces development environment
- Ready for further development and microservices decomposition"
```

#### Step 2: Push to GitHub
```bash
# Push to GitHub
git push origin main
```

---

## Part 7: Application Testing

### Task 7.1: Start the Application

#### Step 1: Launch Application
```bash
# Start the Node.js application
npm start

# Alternative: Use nodemon for development
npm install -g nodemon
nodemon index.js
```

### Task 7.2: Configure Port Forwarding

#### Step 1: Set Up Public Access
1. In Codespace, go to **Ports** tab
2. Locate port **3000**
3. Right-click → **"Change Port Visibility"** → **"Public"**
4. Click the globe icon 🌐 to open in browser

### Task 7.3: Verify Application Functionality

#### Application Testing Checklist
- [ ] **Home Page:** Navigate to `/` - displays main application page
- [ ] **List Suppliers:** Click "List of suppliers" - displays all suppliers from RDS
- [ ] **Add Supplier:** Test "Add a new supplier" functionality
- [ ] **Edit Supplier:** Verify edit operations work
- [ ] **Database Persistence:** Restart application and verify data persists

#### Step 2: Test Endpoints
```bash
# Test all major functionality
curl http://localhost:3000/
curl http://localhost:3000/suppliers

# Check application logs for errors
# Monitor terminal output for any issues
```

---

## Part 8: Troubleshooting Guide

### Common Issues and Solutions

#### Issue 1: Database Connection Problems
**Symptoms:** "ECONNREFUSED" or "ER_ACCESS_DENIED_ERROR"

**Solutions:**
```bash
# Test RDS connectivity
mysql -h RDS_ENDPOINT -u admin -p

# Verify security groups allow your IP
# Check credentials in config files
# Ensure database exists
```

#### Issue 2: Port Forwarding Problems
**Symptoms:** Cannot access application in browser

**Solutions:**
- Verify port 3000 is set to "Public" in Codespace Ports tab
- Check if application is running: `ps aux | grep node`
- Restart application and refresh browser

#### Issue 3: Module Not Found Errors
**Symptoms:** "Cannot find module 'express'"

**Solutions:**
```bash
# Reinstall dependencies
rm -rf node_modules
npm install

# Verify package.json exists and has dependencies
cat package.json
```

#### Issue 4: SSH Transfer Failures
**Symptoms:** "Permission denied" during SCP transfer

**Solutions:**
```bash
# Verify PEM file permissions in Codespace
chmod 600 .ssh/labsuser.pem

# Test SSH connection first
ssh -i .ssh/labsuser.pem ubuntu@EC2_IP "echo 'Connection successful'"
```

---

## Part 9: Alternative Development Methods

### Comparison of Development Environments

| Method | Pros | Cons | Best For |
|--------|------|------|----------|
| **GitHub Codespaces** ✅ | Cloud-based, pre-configured, integrated workflow | Requires internet, learning curve | Modern development, collaboration |
| **EC2 Instance Connect** | No setup, direct access | Limited tools, no version control | Quick testing, debugging |
| **VS Code + Remote SSH** | Full IDE features, direct access | Network dependency, setup complexity | Power users, existing workflows |
| **Local Development** | Familiar environment, offline work | Environment setup complexity | Personal preference, offline needs |

### Method 1: EC2 Instance Connect (Quick Testing)
```bash
# Direct access to MonolithicAppServer
AWS Console → EC2 → Instances → MonolithicAppServer → Connect → EC2 Instance Connect

# Work directly on the instance
cd /home/ubuntu/resources/codebase_partner
npm start
# Access via: http://EC2_PUBLIC_IP
```

### Method 2: GitHub Codespaces (Recommended)
- ✅ Cloud-based development environment
- ✅ Pre-configured tools and runtimes
- ✅ Integrated with GitHub workflow
- ✅ Accessible from any device
- ✅ Automatic environment setup

---

## Part 10: Conclusion and Next Steps

### Lab Summary
In this lab, you successfully:
- [ ] Migrated monolithic application from AWS EC2 to GitHub Codespaces
- [ ] Established secure RDS database connectivity
- [ ] Set up modern development workflow using GitHub
- [ ] Configured application for cloud-based development
- [ ] Implemented proper version control practices

### Key Learnings
- **AWS to GitHub Migration:** Transitioned from AWS-specific services to portable development environments
- **RDS Connectivity:** Maintained database connectivity while changing development platforms
- **Cloud Development:** Leveraged Codespaces for consistent, accessible development environments
- **Troubleshooting Skills:** Developed problem-solving approaches for common migration issues

### Next Steps
1. **Containerization:** Create Dockerfile for consistent deployment
2. **CI/CD Pipeline:** Implement GitHub Actions for automated testing
3. **Microservices Decomposition:** Begin splitting into customer/employee services
4. **Environment Management:** Implement proper configuration management
5. **Monitoring:** Add logging and performance monitoring

### Real-World Application
This lab demonstrates essential skills for modern cloud development:
- Adapting to platform changes (Cloud9 discontinuation)
- Maintaining application functionality during migration
- Implementing portable development practices
- Leveraging cloud-based development environments

**Remember:** The ability to migrate applications between environments and maintain functionality is a critical skill in modern software development, especially as cloud services evolve and change.

---

## Completion Checklist
- [ ] Successfully downloaded and configured SSH access
- [ ] Created GitHub repository and Codespace
- [ ] Transferred application code from EC2 instance
- [ ] Configured RDS database connectivity
- [ ] Installed dependencies and configured application
- [ ] Set up version control and committed code
- [ ] Tested application functionality
- [ ] Resolved any troubleshooting issues

**Congratulations on completing the AWS to GitHub Codespaces migration!**
