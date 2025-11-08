# Phase 4: Using VS Code with GitHub Codespaces for Microservices Deployment

## Lab Overview
This lab guides you through decomposing a monolithic coffee suppliers application into two microservices (customer-facing and employee/admin) and containerizing them using Docker.  
Since AWS Cloud9 is no longer available to new customers and AWS has discontinued onboarding new customers to CodeCommit, we'll use **GitHub Codespaces** as our development environment and **GitHub** for version control.

### Key Technologies Used
- GitHub Codespaces (Cloud Development Environment)
- VS Code
- Docker Containers
- Node.js Microservices
- MySQL Database (AWS RDS or Local)
- GitHub for Version Control

### Architecture Overview
| Component | Description | Port |
|------------|--------------|------|
| Customer Microservice | Read-only access to supplier data | 8080 |
| Employee Microservice | Full CRUD operations with /admin route prefix | 8081 |
| Database | AWS RDS MySQL or local MySQL instance | - |

---

## Step 1: Environment Setup in GitHub Codespaces

### 1.1 Access GitHub Codespaces
1. Navigate to your GitHub repository  
2. Click the **Code** button and select the **Codespaces** tab  
3. Click **Create codespace on main**  
4. Wait for the environment to initialize

### 1.2 Verify Development Environment
```bash
# Check Node.js and npm versions
node --version
npm --version

# Verify Git configuration
git config --list
```

### 1.3 Upload Project Files to Codespaces
- In Codespaces File Explorer, right-click and select **Upload**
- Select your project files/folders from your local machine
- Alternatively, drag and drop files directly into the file explorer

Verify file structure:
```
microservices/
├── customer/
│   ├── app/
│   ├── views/
│   └── package.json
└── employee/
    ├── app/
    ├── views/
    └── package.json
```

---

## Step 2: Database Configuration

### 2.1 Option A: Connect to AWS RDS Database
```bash
sudo apt update
sudo apt install mysql-client -y

# Test RDS connection
mysql -h YOUR_RDS_ENDPOINT -u admin -p -e "SHOW DATABASES;"
# Password: lab-password
```

### 2.2 Option B: Set Up Local MySQL Database
```bash
sudo apt update
sudo apt install mysql-server -y

sudo service mysql start
sudo systemctl enable mysql

sudo mysql -u root << 'EOF'
CREATE DATABASE COFFEE;
CREATE USER 'coffee_user'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON COFFEE.* TO 'coffee_user'@'localhost';
FLUSH PRIVILEGES;

USE COFFEE;
CREATE TABLE suppliers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(50),
    description TEXT,
    address VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO suppliers (name, email, phone, description, address, city, state) VALUES
('Mountain Coffee Co.', 'contact@mountaincoffee.com', '555-0101', 'Organic mountain-grown beans', '123 Mountain Rd', 'Denver', 'CO'),
('Valley Roasters', 'info@valleyroasters.com', '555-0102', 'Premium dark roast specialists', '456 Valley Ave', 'Seattle', 'WA'),
('Urban Brew', 'sales@urbanbrew.com', '555-0103', 'Fair trade urban coffee', '789 City St', 'Portland', 'OR');
EOF
```

### 2.3 Configure Environment Variables
Create `.env` files for both microservices:

**Customer Microservice (.env):**
```bash
cd /workspaces/your-repo/microservices/customer
cat > .env << 'EOF'
APP_DB_HOST=localhost
APP_DB_USER=coffee_user
APP_DB_PASSWORD=password
APP_DB_NAME=COFFEE
APP_PORT=8080
EOF
```

**Employee Microservice (.env):**
```bash
cd ../employee
cat > .env << 'EOF'
APP_DB_HOST=localhost
APP_DB_USER=coffee_user
APP_DB_PASSWORD=password
APP_DB_NAME=COFFEE
APP_PORT=8081
EOF
```

---

## Step 3: Modify Customer Microservice (Read-Only)

### 3.1 Update Supplier Model
**File:** `customer/app/models/supplier.model.js`
```javascript
const sql = require("./db.js");

const Supplier = function(supplier) {
    this.name = supplier.name;
    this.email = supplier.email;
    this.phone = supplier.phone;
    this.description = supplier.description;
};

Supplier.getAll = result => {
    sql.query("SELECT * FROM suppliers", (err, res) => {
        if (err) {
            result(null, err);
            return;
        }
        result(null, res);
    });
};

Supplier.findById = (id, result) => {
    sql.query(`SELECT * FROM suppliers WHERE id = ${id}`, (err, res) => {
        if (err) {
            result(err, null);
            return;
        }
        if (res.length) {
            result(null, res[0]);
            return;
        }
        result({ kind: "not_found" }, null);
    });
};

module.exports = Supplier;
```

### 3.2 Update Navigation
**File:** `customer/views/nav.html`
```html
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
    <a class="navbar-brand" href="#">Coffee suppliers</a>
    <button class="navbar-toggler" type="button" data-toggle="collapse" 
            data-target="#navbarNavAltMarkup" aria-controls="navbarNavAltMarkup" 
            aria-expanded="false" aria-label="Toggle navigation">
        <span class="navbar-toggler-icon"></span>
    </button>
    <div class="collapse navbar-collapse" id="navbarNavAltMarkup">
        <div class="navbar-nav">
            <a class="nav-link" href="/">Customer home</a>
            <a class="nav-link" href="/suppliers">List of suppliers</a>
            <a class="nav-link" href="/admin/suppliers">Administrator link</a>
        </div>
    </div>
</nav>
```

### 3.3 Update Supplier List View (Read-Only)
- Remove â€œAdd new supplierâ€ button  
- Remove edit/delete options  
- Keep only the supplier display table

---

## Step 4: Modify Employee Microservice (Admin Functions)

### 4.1 Update Employee Controller
**File:** `employee/app/controller/supplier.controller.js`  
Update all redirect calls to use `/admin` prefix:
```javascript
// Change all redirects from:
res.redirect("/suppliers");
// To:
res.redirect("/admin/suppliers");
```

### 4.2 Update Employee Routes
**File:** `employee/index.js`
```javascript
require('dotenv').config();
const express = require("express");
// ... other imports

// ADD /admin PREFIX TO ALL ROUTES
app.get("/admin", (req, res) => {
    res.render("home", {});
});

app.get("/admin/suppliers/", supplier.findAll);
app.get("/admin/supplier-add", (req, res) => {
    res.render("supplier-add", {});
});
app.post("/admin/supplier-add", supplier.create);
app.get("/admin/supplier-update/:id", supplier.findOne);
app.post("/admin/supplier-update", supplier.update);
app.post("/admin/supplier-remove/:id", supplier.remove);

// Update port
const app_port = process.env.APP_PORT || 8081;
app.listen(app_port, () => {
    console.log(`Coffee suppliers employee microservice is running on port ${app_port}.`);
});
```

### 4.3 Update Employee Views
Update form actions in HTML files to use `/admin` prefix:

**File:** `employee/views/supplier-add.html`
```html
<form action="/admin/supplier-add" method="POST">
```

**File:** `employee/views/supplier-update.html`
```html
<form action="/admin/supplier-update" method="POST">
<form action="/admin/supplier-remove/{{id}}" method="POST">
```

### 4.4 Update Employee Navigation
**File:** `employee/views/nav.html`
```html
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
    <img src="/img/espresso.jpg" width="200">
    <div><a class="navbar-brand page-title" href="/admin">Manage coffee suppliers</a></div>
    <div class="collapse navbar-collapse" id="navbarSupportedContent">
        <ul class="navbar-nav mr-auto">
            <li class="nav-item active">
                <a class="nav-link" href="/admin/suppliers">Administrator home</a>
                <a class="nav-link" href="/admin/suppliers">Suppliers list</a>
                <a class="nav-link" href="/">Customer home</a>
            </li>
        </ul>
    </div>
</nav>
```

---

## Step 5: Create Docker Containers
**Customer Dockerfile:**
```dockerfile
FROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 8080
CMD ["npm", "run", "start"]
```

**Employee Dockerfile:**
```dockerfile
FROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 8081
CMD ["npm", "run", "start"]
```

---

## Step 6: Test and Verify
```bash
docker build --tag customer ./customer
docker build --tag employee ./employee
docker run -d --name customer_1 -p 8080:8080 --env-file .env customer
docker run -d --name employee_1 -p 8081:8081 --env-file .env employee
docker ps
```

Visit:  
- Customer: `https://your-codespace-8080.app.github.dev/`  
- Employee: `https://your-codespace-8081.app.github.dev/admin`

---

## Conclusion
âœ… Both microservices configured and containerized  
âœ… Database connection verified  
âœ… Admin panel functional with CRUD operations  
âœ… Code pushed to GitHub ready for ECS deployment
