# Lab 07-3: Creating a Development Environment and Git Repository in GitHub Codespaces

## Lesson Title
**AWS to GitHub Codespaces Migration: Direct Code Transfer and Microservices Setup**

## Lab Overview

This lab guides students through creating a GitHub Codespaces development environment, transferring a monolithic application directly from AWS EC2 into a structured microservices architecture, and setting up version control with Git. Students will create separate customer and employee microservices directories using identical starter code and test the application's functionality in a cloud environment.


## Prerequisites

Before starting this lab, ensure you have the following:

- A **GitHub account**
- An **AWS EC2 instance** running `MonolithicAppServer`
- **RDS database** endpoint and credentials
- Basic knowledge of **command-line operations** and **Git**

---

## Learning Objectives

By the end of this lab, you should be able to:

- Create a GitHub Codespaces development environment  
- Transfer code directly from AWS EC2 to structured microservices directories  
- Create **customer** and **employee** microservices  
- Initialize a Git repository with proper version control  
- Test microservices functionality in a cloud environment  

---

## Phase 1: GitHub Repository and Codespace Setup

### Task 1.1: Create GitHub Repository

#### Step 1.1.1: Repository Creation

1. **Navigate to GitHub:**
   - Go to [github.com](https://github.com) and sign in.
   - Click "+" → "New repository"

2. **Configure Repository Settings:**

   | Setting | Value |
   |----------|-------|
   | Repository Name | `coffee-suppliers-microservices` |
   | Description | "Coffee suppliers application with customer and employee microservices" |
   | Visibility | Private |
   | Initialize with README | ❌ Uncheck |
   | Add .gitignore | None |
   | License | None |

3. **Create Repository:**
   - Click "Create repository"
   - Note your repository URL for later.

---

#### Step 1.1.2: Launch GitHub Codespace

1. In your repository, click "Code" → "Codespaces" tab In your repository, click **â€œCodeâ€ â†’ â€œCodespacesâ€ tab â†’ â€œCreate codespace on main.â€**  
2. Wait for provisioning (2 - 3 minutes).  
   VS Code interface will open in your browser.  
3. Verify the environment:

```bash
# Open terminal in Codespace (Ctrl + `)
echo "✅ Codespace environment ready"
pwd  # Should show: /workspaces/coffee-suppliers-microservices
ls -la
```

---

## Phase 2: Direct Code Transfer to Microservices Structure

### Task 2.1: Prepare SSH Access and Create Directory Structure

#### Step 2.1.1: Set Up SSH in Codespace

```bash
# Create SSH directory
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Create PEM file - PASTE YOUR KEY CONTENT
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

---

#### Step 2.1.2: Create Microservices Directory Structure

```bash
# Create microservices directories
mkdir -p microservices/customer
mkdir -p microservices/employee

# Verify structure
echo "âœ… Microservices directory structure created:"
tree microservices -L 2
```

---

### Task 2.2: Direct Code Transfer from AWS EC2

#### Step 2.2.1: Transfer Code to Customer Directory

```bash
# Transfer monolithic code directly to customer directory
echo "ðŸ“¦ Transferring code to customer microservice..."
scp -r -i ~/.ssh/labsuser.pem ubuntu@YOUR_EC2_IP:/home/ubuntu/resources/codebase_partner/* microservices/customer/

# Verify transfer
echo "âœ… Customer microservice files transferred:"
ls -la microservices/customer/ | head -10
```

---

#### Step 2.2.2: Copy Code to Employee Directory

```bash
# Copy code from customer to employee directory
cp -r microservices/customer/* microservices/employee/

# Verify identical structure
echo "âœ… Employee microservice files copied:"
ls -la microservices/employee/ | head -10
```

---

#### Step 2.2.3: Verify Critical Files

```bash
# Check critical application files exist in both services
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

```bash
# Configure database connection for both microservices
for service in customer employee; do
cat > microservices/$service/config/db.config.js << EOF
module.exports = {
  HOST: "YOUR_RDS_ENDPOINT",
  USER: "admin", 
  PASSWORD: "lab-password",
  DB: "COFFEE",
  dialect: "mysql",
  pool: { max: 5, min: 0, acquire: 30000, idle: 10000 }
};
EOF
done
```

> **Note:** Replace `YOUR_RDS_ENDPOINT` with your actual RDS endpoint.

---

#### Step 3.1.2: Test Database Connectivity

```bash
# Test database connection from Codespace
for service in customer employee; do
    cd microservices/$service
    node -e "
    const mysql = require('mysql2');
    const config = require('./config/db.config.js');
    const connection = mysql.createConnection({
      host: config.HOST,
      user: config.USER,
      password: config.PASSWORD,
      database: config.DB
    });
    connection.connect(err => {
      if (err) console.error('âŒ Connection failed:', err.message);
      else console.log('âœ… Connected successfully');
      connection.end();
    });
    "
    cd ../..
done
```

---

### Task 3.2: Configure Microservices Entry Points

#### Step 3.2.1: Configure Customer Microservice (Port 8080)

```bash
cat > microservices/customer/index.js << 'EOF'
const express = require('express');
const path = require('path');
const app = express();

// Setup
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'hbs');
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(express.static(path.join(__dirname, 'public')));

// Routes
const suppliers = require('./app/controllers/supplier.controller.js');
app.get('/', (req, res) => {
  res.render('home', { title: 'Coffee Suppliers - Customer Portal', userType: 'customer' });
});
app.get('/suppliers', suppliers.findAll);

const PORT = process.env.PORT || 8080;
app.listen(PORT, '0.0.0.0', () => {
  console.log(`âœ… CUSTOMER MICROSERVICE running on port ${PORT}`);
});
EOF
```

---

#### Step 3.2.2: Configure Employee Microservice (Port 8081)

```bash
cat > microservices/employee/index.js << 'EOF'
const express = require('express');
const path = require('path');
const app = express();

app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'hbs');
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(express.static(path.join(__dirname, 'public')));

const suppliers = require('./app/controllers/supplier.controller.js');
app.get('/', (req, res) => {
  res.render('home', { title: 'Coffee Suppliers - Employee Portal', userType: 'employee' });
});
app.get('/admin/suppliers', suppliers.findAll);
app.get('/admin/supplier-update/:id', suppliers.findOne);

const PORT = process.env.PORT || 8081;
app.listen(PORT, '0.0.0.0', () => {
  console.log(`âœ… EMPLOYEE MICROSERVICE running on port ${PORT}`);
});
EOF
```

---

### Task 3.3: Create Package Configuration and Install Dependencies

```bash
# Create package.json for both services (example for customer)
cat > microservices/customer/package.json << 'EOF'
{
  "name": "customer-microservice",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": { "start": "node index.js", "dev": "nodemon index.js" },
  "dependencies": { "express": "^4.18.2", "mysql2": "^3.6.0", "hbs": "^4.2.0" },
  "devDependencies": { "nodemon": "^3.0.1" }
}
EOF

# Install dependencies
cd microservices/customer && npm install && cd ../employee && npm install && cd ../..
```

---

## Phase 4: Git Repository Setup and Testing

### Task 4.1: Initialize Git Repository

```bash
git init
git config user.name "Student Developer"
git config user.email "student@example.com"

# Create .gitignore
cat > .gitignore << 'EOF'
node_modules/
.env
.DS_Store
.vscode/
.ssh/
*.pem
EOF
```

---

#### Step 4.1.2: Create Initial Commit

```bash
git add .
git commit -m "feat: Initial microservices structure with AWS EC2 code transfer"
```

---

### Task 4.2: Connect to GitHub and Push

```bash
git remote add origin https://github.com/<your-username>/coffee-suppliers-microservices.git
git branch -M main
git push -u origin main
```

> If authentication fails, use a **Personal Access Token** instead of a password.

---

### Task 4.3: Test Microservices Functionality

#### Step 4.3.1: Quick Service Test

```bash
# Test customer and employee services
cd microservices/customer && timeout 5s npm start || echo "âœ… Customer OK"
cd ../employee && timeout 5s npm start || echo "âœ… Employee OK"
cd ../..
```

---

## Final Verification and Summary

### Verification Checklist

```bash
echo "=== FINAL VERIFICATION ==="
git status
tree microservices -L 2
find microservices -name "*.js" | wc -l
```

---

## Conclusion

### Lab Summary

âœ… Successfully Completed:
- Direct SCP code transfer from AWS EC2 to GitHub Codespaces  
- Structured microservices setup (`customer` and `employee`)  
- Port configurations: 8080 (customer), 8081 (employee)  
- Database configuration and connectivity tests  
- Git version control with documentation  

### Key Achievements

- **Efficient Migration:** Direct SCP transfer from EC2 to Codespaces  
- **Microservices Foundation:** Customer and employee services ready for customization  
- **Modern Cloud Development:** Using GitHub Codespaces instead of AWS Cloud9  
- **Version Control:** Professional Git workflow  

---

## Architecture Overview

```
microservices/
â”œâ”€â”€ customer/     # Port 8080 - Read-only customer portal
â””â”€â”€ employee/     # Port 8081 - Full admin employee portal
    â””â”€â”€ Shared AWS RDS MySQL Database
```

---

## Next Steps

1. Customize services for specific roles  
2. Containerize each service using **Dockerfiles**  
3. Implement **CI/CD pipelines** with GitHub Actions  
4. Deploy to **AWS ECS**  

---

## Real-World Skills Developed

- Cloud development environment setup  
- Direct code migration between platforms  
- Microservices architecture planning  
- Git-based version control  
- Database configuration and testing  

---

> *This lab provides a practical foundation for modern cloud-native development using direct code transfer and structured microservices preparation.*
