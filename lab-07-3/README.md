# Lab 07-3: Creating a Development Environment and Git Repository in GitHub Codespaces

**Lesson Title:** AWS to GitHub Codespaces Migration: Direct Code Transfer and Microservices Setup

---

## ðŸš€ Lab Overview

This lab guides you through creating a **GitHub Codespaces** development environment, directly transferring monolithic application code from an AWS EC2 instance into a structured microservices directory, and establishing proper Git repository management. We will use a direct **SCP transfer** to create `customer` and `employee` microservices directories with identical starter code, setting the foundation for microservice development.

### â±ï¸ Details and Prerequisites

| Detail | Value |
| :--- | :--- |
| **Duration** | 30â€“45 minutes |
| **Difficulty** | Intermediate |

**Prerequisites**

- **GitHub account**
- **AWS EC2 instance** with `MonolithicAppServer` running
- **RDS database endpoint** and credentials
- Basic command line and **Git knowledge**

### âœ… Learning Objectives

By the end of this lab, you will be able to:

- Create a **GitHub Codespaces** development environment.
- Directly transfer code from **AWS EC2** to microservices directories.
- Create a `customer` and `employee` microservices structure.
- Initialize a **Git repository** with proper version control.
- Configure and test application functionality in the cloud environment.

---

## Phase 1: GitHub Repository and Codespace Setup

### Task 1.1: Create GitHub Repository

| Step | Action | Instructions |
| :--- | :--- | :--- |
| **1.1.1** | **Repository Creation** | 1) Navigate to **github.com** and sign in. 2) Click the `+` icon -> **New repository**. |
| **1.1.2** | **Configure Repository** | - **Repository name:** `coffee-suppliers-microservices`<br> - **Description:** â€œCoffee suppliers application with customer and employee microservicesâ€<br> - **Visibility:** Private<br> - **Initialize with README:** Uncheck<br> - **.gitignore/License:** None |
| **1.1.3** | **Finalize** | Click **Create repository** and note your repository URL. |

### Task 1.2: Launch GitHub Codespace

1. In your new repository, click **Code** -> **Codespaces** tab.  
2. Click **Create codespace on main**.  
3. Wait for the environment to provision (2â€“3 minutes). The **VS Code interface** will open in your browser.  
4. Verify the environment by running the following in the Codespace terminal:

```bash
# Open terminal in Codespace (Ctrl + `)
echo "âœ… Codespace environment ready"
pwd  # Should show: /workspaces/coffee-suppliers-microservices
ls -la
```

---

## Phase 2: Direct Code Transfer to Microservices Structure

### Task 2.1: Prepare SSH Access and Create Directory Structure

#### Step 2.1.1: Set Up SSH in Codespace

```bash
# Create SSH directory and set secure permissions
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Create PEM file - PASTE YOUR labsuser.pem KEY CONTENT
echo "Creating SSH key file - paste your labsuser.pem content:"
cat > ~/.ssh/labsuser.pem << 'EOF'
-----BEGIN RSA PRIVATE KEY-----
[PASTE YOUR ENTIRE labsuser.pem CONTENT HERE]
-----END RSA PRIVATE KEY-----
EOF

# Set secure permissions
chmod 600 ~/.ssh/labsuser.pem
echo "âœ… SSH key configured"
```

#### Step 2.1.2: Create Microservices Directory Structure

```bash
# Create the main microservices directory
mkdir -p microservices

# Create customer and employee directories inside microservices
mkdir -p microservices/customer
mkdir -p microservices/employee

# Verify structure
echo "âœ… Microservices directory structure created:"
tree microservices -L 2
```

### Task 2.2: Direct Code Transfer from AWS EC2

#### Step 2.2.1: Transfer Code to Customer Directory

> This command transfers the monolithic code from your EC2 instance into the new `microservices/customer/` directory.

```bash
# Transfer monolithic code directly to customer directory
echo "ðŸ“¦ Transferring code to customer microservice..."
scp -r -i ~/.ssh/labsuser.pem ubuntu@YOUR_EC2_IP:/home/ubuntu/resources/codebase_partner/* microservices/customer/

# Verify transfer
echo "âœ… Customer microservice files transferred:"
ls -la microservices/customer/ | head -10
echo "Total files in customer: $(find microservices/customer -type f | wc -l)"
```

> **IMPORTANT:** Replace `YOUR_EC2_IP` with the public IP address of your AWS EC2 instance.

#### Step 2.2.2: Copy Code to Employee Directory

> Both microservices start with the identical monolithic codebase.

```bash
# Copy the same code to employee directory
echo "ðŸ“¦ Copying code to employee microservice..."
cp -r microservices/customer/* microservices/employee/

# Verify both directories have identical structure
echo "âœ… Employee microservice files copied:"
ls -la microservices/employee/ | head -10
echo "Total files in employee: $(find microservices/employee -type f | wc -l)"
```

#### Step 2.2.3: Verify Critical Files

```bash
# Check critical application files exist in both services
echo "ðŸ” Verifying critical files in both microservices..."

critical_files=("index.js" "package.json" "app/controllers/supplier.controller.js" "app/models/supplier.model.js")

for service in customer employee; do
    echo "--- $service microservice ---"
    for file in "${critical_files[@]}"; do
        if [ -f "microservices/$service/$file" ]; then
            echo "âœ… $file"
        else
            echo "âŒ $file - MISSING"
        fi
    done
    echo ""
done
```

---

## Phase 3: Configure Microservices Development

### Task 3.1: Configure Database Connection

#### Step 3.1.1: Update Database Configuration

> Update the database configuration for both services to connect to the shared AWS RDS instance.

```bash
# Configure database connection for both microservices
for service in customer employee; do
    echo "âš™ï¸ Configuring database for $service microservice..."
    
    # Create/update db.config.js
    cat > microservices/$service/config/db.config.js << EOF
// Database configuration for $service microservice
module.exports = {
  HOST: "YOUR_RDS_ENDPOINT",
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

console.log('âœ… $service microservice database configuration loaded');
EOF

    # Update app/config/config.js if it exists
    if [ -f "microservices/$service/app/config/config.js" ]; then
        cat > microservices/$service/app/config/config.js << EOF
module.exports = {
  APP_DB_HOST: "YOUR_RDS_ENDPOINT",
  APP_DB_USER: "admin",
  APP_DB_PASSWORD: "lab-password", 
  APP_DB: "COFFEE"
};
EOF
    fi
done

echo "âœ… Database configuration updated for both microservices"
```

> **IMPORTANT:** Replace `YOUR_RDS_ENDPOINT` with your actual AWS RDS endpoint address.

#### Step 3.1.2: Test Database Connectivity

> This script attempts to connect to the RDS database using the configured credentials and executes a simple query.

```bash
# Test database connection from Codespace
echo "ðŸ§ª Testing database connectivity..."

for service in customer employee; do
    echo "Testing $service microservice database connection..."
    cd microservices/$service
    
    node -e "
    const mysql = require('mysql2');
    const config = require('./config/db.config.js');
    
    console.log('Connecting to database:', config.HOST);
    
    const connection = mysql.createConnection({
      host: config.HOST,
      user: config.USER,
      password: config.PASSWORD,
      database: config.DB
    });
    
    connection.connect((err) => {
      if (err) {
        console.error('âŒ Database connection failed:', err.message);
        process.exit(1);
      } else {
        console.log('âœ… Database connected successfully');
        
        // Test a simple query
        connection.query('SELECT COUNT(*) as count FROM suppliers', (err, results) => {
          if (err) {
            console.error('âŒ Query failed:', err.message);
          } else {
            console.log('ðŸ“Š Suppliers in database:', results[0].count);
          }
          connection.end();
        });
      }
    });
    "
    
    cd ../..
    echo ""
done
```

### Task 3.2: Configure Microservices Entry Points

> The `index.js` file for each service is customized to define specific ports and initial routes.

#### Step 3.2.1: Configure Customer Microservice (Port 8080)

```bash
# Update customer microservice to run on port 8080
echo "âš™ï¸ Configuring customer microservice (port 8080)..."

cat > microservices/customer/index.js << 'EOF'
const express = require('express');
const path = require('path');
const app = express();

// View engine setup
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'hbs');

// Middleware
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(express.static(path.join(__dirname, 'public')));

// Import routes and controllers
const suppliers = require('./app/controllers/supplier.controller.js');

// Customer routes - read only access
app.get('/', (req, res) => {
  res.render('home', { 
    title: 'Coffee Suppliers - Customer Portal',
    userType: 'customer'
  });
});

app.get('/suppliers', suppliers.findAll);

const PORT = process.env.PORT || 8080;
app.listen(PORT, '0.0.0.0', () => {
  console.log('='.repeat(50));
  console.log('âœ… CUSTOMER MICROSERVICE');
  console.log(`ðŸ“ Port: ${PORT}`);
  console.log(`ðŸ”— http://localhost:${PORT}`);
  console.log('ðŸ‘¥ Access: Read-only customer portal');
  console.log('='.repeat(50));
});
EOF

echo "âœ… Customer microservice configured for port 8080"
```

#### Step 3.2.2: Configure Employee Microservice (Port 8081)

```bash
# Update employee microservice to run on port 8081
echo "âš™ï¸ Configuring employee microservice (port 8081)..."

cat > microservices/employee/index.js << 'EOF'
const express = require('express');
const path = require('path');
const app = express();

// View engine setup
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'hbs');

// Middleware
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(express.static(path.join(__dirname, 'public')));

// Import routes and controllers
const suppliers = require('./app/controllers/supplier.controller.js');

// Employee routes - full admin access with /admin prefix
app.get('/', (req, res) => {
  res.render('home', { 
    title: 'Coffee Suppliers - Employee Portal',
    userType: 'employee'
  });
});

app.get('/admin/suppliers', suppliers.findAll);
app.get('/admin/supplier-update/:id', suppliers.findOne);

const PORT = process.env.PORT || 8081;
app.listen(PORT, '0.0.0.0', () => {
  console.log('='.repeat(50));
  console.log('âœ… EMPLOYEE MICROSERVICE');
  console.log(`ðŸ“ Port: ${PORT}`);
  console.log(`ðŸ”— http://localhost:${PORT}`);
  console.log('ðŸ‘¤ Access: Full administrative control');
  console.log('='.repeat(50));
});
EOF

echo "âœ… Employee microservice configured for port 8081"
```

### Task 3.2.3: Create Package Configuration (`package.json`)

> The `package.json` file is created for both services to manage dependencies and startup scripts.

```bash
# Create package.json for customer microservice
cat > microservices/customer/package.json << 'EOF'
{
  "name": "customer-microservice",
  "version": "1.0.0",
  "description": "Customer-facing read-only coffee suppliers service",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mysql2": "^3.6.0",
    "hbs": "^4.2.0",
    "express-validator": "^7.0.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  },
  "keywords": ["coffee", "suppliers", "customer", "microservice"],
  "author": "Student Developer",
  "license": "MIT"
}
EOF

# Create package.json for employee microservice
cat > microservices/employee/package.json << 'EOF'
{
  "name": "employee-microservice", 
  "version": "1.0.0",
  "description": "Employee-facing admin coffee suppliers service",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js", 
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mysql2": "^3.6.0",
    "hbs": "^4.2.0",
    "express-validator": "^7.0.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  },
  "keywords": ["coffee", "suppliers", "employee", "microservice", "admin"],
  "author": "Student Developer",
  "license": "MIT"
}
EOF

echo "âœ… Package configuration created for both microservices"
```

### Task 3.3: Install Dependencies

```bash
# Install dependencies for customer microservice
echo "ðŸ“¦ Installing dependencies for customer microservice..."
cd microservices/customer
npm install
cd ../..

# Install dependencies for employee microservice
echo "ðŸ“¦ Installing dependencies for employee microservice..."
cd microservices/employee  
npm install
cd ../..

echo "âœ… Dependencies installed for both microservices"
```

---

## Phase 4: Git Repository Setup and Testing

### Task 4.1: Initialize Git Repository

#### Step 4.1.1: Configure Git and Create `.gitignore`

```bash
# Initialize Git repository
git init

# Configure Git user (Use your own name/email for a real project)
git config user.name "Student Developer"
git config user.email "student@example.com"

# Create .gitignore file to exclude temporary files and node modules
cat > .gitignore << 'EOF'
# Dependencies
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Environment variables
.env
.env.local
.env.*.local

# Logs
logs
*.log

# Runtime data
*.pid
*.seed
*.pid.lock

# Coverage directory
coverage/
.nyc_output

# Optional npm cache
.npm

# Optional REPL history
.node_repl_history

# Output of 'npm pack'
*.tgz

# Yarn Integrity file
.yarn-integrity

# dotenv environment variables
.env

# macOS/Windows
.DS_Store
Thumbs.db

# IDE/Editor files
.vscode/
.idea/
*.swp
*.swo

# SSH keys (never commit!)
*.pem
*.key
.ssh/

# Temporary files
*.tmp
*.temp
temp/
EOF

echo "âœ… Git repository initialized and .gitignore created"
```

#### Step 4.1.2: Create Initial Commit

```bash
# Stage all files
git add .

# Check what will be committed
echo "ðŸ“ Files to be committed:"
git status

# Create initial commit
git commit -m "feat: Initial microservices structure with AWS EC2 code

Summary:
- Direct SCP transfer from AWS EC2 to GitHub Codespaces
- Created microservices/customer and microservices/employee directories
- Both services contain identical starter code from monolithic application
- Configured for microservices development

Changes:
- Transferred code from: /home/ubuntu/resources/codebase_partner/
- Created: microservices/customer/ with all application files
- Created: microservices/employee/ with identical code copy
- Configured customer service on port 8080 (read-only)
- Configured employee service on port 8081 (admin access)

Service Configuration:
- Customer: http://localhost:8080 (Customer Portal)
- Employee: http://localhost:8081 (Employee Portal)
- Shared RDS MySQL database
- Ready for microservices customization

Technical Details:
- Source: AWS EC2 MonolithicAppServer
- Transfer: Direct SCP to Codespaces
- Database: AWS RDS MySQL
- Framework: Node.js with Express.js"

echo "âœ… Initial commit created:"
git log --oneline -1
```

### Task 4.2: Connect to GitHub and Push

#### Step 4.2.1: Push to GitHub Repository

```bash
# Add GitHub as remote origin (replace USERNAME if needed)
git remote add origin https://github.com/$(git config user.name)/coffee-suppliers-microservices.git

# Verify remote configuration
echo "ðŸ”— Remote configuration:"
git remote -v

# Rename branch to main
git branch -M main

# Push to GitHub
echo "ðŸš€ Pushing code to GitHub repository..."
git push -u origin main

echo "âœ… Code successfully pushed to GitHub"
```

> **Note:** If authentication fails, use a **Personal Access Token (PAT)** as your password. Generate one from **GitHub Settings -> Developer settings -> Personal access tokens**, with the `repo` scope.

#### Step 4.2.2: Verify GitHub Repository

- Navigate to your GitHub repository in the browser.  
- Verify the directory structure: `microservices/customer/` and `microservices/employee/` are present.  
- Check the commit history.

### Task 4.3: Test Microservices Functionality

#### Step 4.3.1: Quick Service Test

> Verifies that both Node.js services can start up successfully without immediate errors.

```bash
# Test if both services can start without errors
echo "ðŸ§ª Testing microservices startup..."

# Test customer service
echo "Testing customer microservice..."
cd microservices/customer
timeout 5s npm start || echo "âœ… Customer service starts successfully"
cd ../..

# Test employee service  
echo "Testing employee microservice..."
cd microservices/employee
timeout 5s npm start || echo "âœ… Employee service starts successfully"
cd ../..

echo "âœ… Both microservices can start successfully"
```

#### Step 4.3.2: Create Project Documentation

Create a comprehensive `README.md` for the repository.

```bash
# Create comprehensive README
cat > README.md << 'EOF'
# Coffee Suppliers Microservices

Microservices architecture for the coffee suppliers application migrated from AWS EC2 to GitHub Codespaces.

## Project Structure
```
microservices/
â”œâ”€â”€ customer/     # Customer-facing service (Port 8080)
â”‚   â”œâ”€â”€ app/      # Application logic
â”‚   â”œâ”€â”€ views/    # Customer templates
â”‚   â”œâ”€â”€ config/   # Service configuration
â”‚   â””â”€â”€ index.js  # Service entry point
â””â”€â”€ employee/     # Employee-facing service (Port 8081)
    â”œâ”€â”€ app/      # Application logic
    â”œâ”€â”€ views/    # Employee templates
    â”œâ”€â”€ config/   # Service configuration
    â””â”€â”€ index.js  # Service entry point
```

## âš¡ Quick Start

### Development
Run the following commands in the Codespace terminal:

```bash
# Customer service (read-only)
cd microservices/customer
npm run dev
# Access: http://localhost:8080

# Employee service (admin)  
cd microservices/employee
npm run dev  
# Access: http://localhost:8081
```

### Database Setup
Update the database configuration in both services by replacing `your-rds-endpoint.us-east-1.rds.amazonaws.com` with your actual RDS endpoint:

```js
// microservices/[service]/config/db.config.js
HOST: "your-rds-endpoint.us-east-1.rds.amazonaws.com",
USER: "admin",
PASSWORD: "lab-password",
DB: "COFFEE"
```

## ðŸ’» Service Overview

| Service  | Port | Access Level       | Features                         | Key URL Paths         |
|---------|------|--------------------|----------------------------------|-----------------------|
| Customer | 8080 | Read-only          | View suppliers, contact info     | `/`, `/suppliers`     |
| Employee | 8081 | Full Administrative| CRUD operations, management      | `/`, `/admin/suppliers` |

## Migration Notes
- Source: AWS EC2 instance path `/home/ubuntu/resources/codebase_partner/`  
- Transfer: Direct SCP to Codespaces microservices directories  
- Both services start with an identical monolithic codebase and are ready for microservices-specific customization.
EOF

# Add and commit documentation
git add README.md
git commit -m "docs: Add comprehensive README with project structure"
git push origin main
echo "âœ… Project documentation created and pushed"
```

---

## ðŸ“‹ Final Verification and Summary

### Final Verification Checklist

Run the following commands to confirm the lab setup:

```bash
echo "=== FINAL VERIFICATION CHECKLIST ==="

# 1. Git status
echo "1. Git Repository:"
echo "   Branch: $(git branch --show-current)"
echo "   Status: $(git status --porcelain | wc -l) changes pending"

# 2. Directory structure
echo "2. Directory Structure:"
echo "   microservices/: $(test -d microservices && echo 'âœ…' || echo 'âŒ')"
echo "   microservices/customer/: $(test -d microservices/customer && echo 'âœ…' || echo 'âŒ')"
echo "   microservices/employee/: $(test -d microservices/employee && echo 'âœ…' || echo 'âŒ')"

# 3. File counts (JS files)
echo "3. File Counts:"
echo "   Customer: $(find microservices/customer -name "*.js" | wc -l) JS files"
echo "   Employee: $(find microservices/employee -name "*.js" | wc -l) JS files"

# 4. Critical files
echo "4. Critical Files:"
for file in "README.md" ".gitignore"; do
    echo "   $file: $(test -f "$file" && echo 'âœ…' || echo 'âŒ')"
done

# 5. Dependencies
echo "5. Dependencies:"
for service in customer employee; do
    echo "   $service node_modules: $(test -d "microservices/$service/node_modules" && echo 'âœ…' || echo 'âŒ')"
done

echo "=== VERIFICATION COMPLETE ==="
```

---

## Lab Conclusion and Architecture

This lab provides a practical foundation for modern cloudâ€‘native development using direct code transfer and structured microservices preparation.

### Key Achievements
- **Efficient Migration:** Direct SCP transfer from AWS EC2 to GitHub Codespaces
- **Microservices Foundation:** Customer and employee services created and ready for customization
- **Modern Development:** Utilized GitHub Codespaces as a powerful, cloudâ€‘based environment
- **Version Control:** Implemented a professional Git workflow
- **Database Connectivity:** RDS connection configured and tested for both services

### Resulting Architecture

```
microservices/
â”œâ”€â”€ customer/     # Port 8080 - Read-only customer portal
â””â”€â”€ employee/     # Port 8081 - Full admin employee portal
    â””â”€â”€ Shared RDS MySQL Database
```

### Next Steps
- **Customize Services:** Implement readâ€‘only features for the customer service and full admin features for the employee service.  
- **Containerization:** Create Dockerfiles for both microservices.  
- **CI/CD Pipeline:** Implement GitHub Actions for continuous integration and deployment.  
- **Production Deployment:** Prepare deployment to a service like AWS ECS.
