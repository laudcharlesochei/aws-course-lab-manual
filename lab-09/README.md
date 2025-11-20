# Lab 09: Configuring Microservices and Testing in Docker Containers

## Lab Overview
In this phase, you will transform the **monolithic coffee suppliers application** into two separate microservices(customer and employee) containerize them using Docker, and test them locally. 
This setup replaces AWS Cloud9 with GitHub Codespaces and uses Docker Desktop for local container testing.

---

## Prerequisites
Before beginning, ensure you have:
- A **GitHub account** with repository access
- **Docker Desktop** installed locally
- **VS Code** with the Docker extension
- Basic knowledge of **Node.js** and **Docker**

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
### 1.4 Database Configuration - Option A: Connect to AWS RDS Database
```bash
sudo apt update
sudo apt install mysql-client -y

# Test RDS connection
mysql -h YOUR_RDS_ENDPOINT -u admin -p -e "SHOW DATABASES;"
# Password: lab-password
```

### 1.5 Database Configuration - Option B: Set Up Local MySQL Database
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

### 1.6 Configure Environment Variables
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

## Step 2: Modify Customer Microservice (Read-Only)

### 2.1 Update Customer Controller
**File:** `customer/app/controller/supplier.controller.js`
```javascript
const Supplier = require("../models/supplier.model.js");
const { body, validationResult } = require("express-validator");

exports.findAll = (req, res) => {
  Supplier.getAll((err, data) => {
    if (err)
      res.render("500", { message: "There was a problem retrieving the list of suppliers" });
    else
      res.render("supplier-list-all", { suppliers: data });
  });
};

exports.findOne = (req, res) => {
  Supplier.findById(req.params.id, (err, data) => {
    if (err) {
      if (err.kind === "not_found") {
        res.status(404).send({ message: `Not found Supplier with id ${req.params.id}.` });
      } else {
        res.render("500", { message: `Error retrieving Supplier with id ${req.params.id}` });
      }
    } else res.render("supplier-update", { supplier: data });
  });
};
```

### 2.2 Update Customer Model
**File:** `customer/app/models/supplier.model.js`
- **Keep:** `Supplier.getAll`, `Supplier.findById`
- **Remove:** `Supplier.create`, `Supplier.updateById`, `Supplier.remove`

### 2.3 Update Customer Navigation
**File:** `customer/views/nav.html`
```html
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
  <a class="navbar-brand" href="#">Coffee suppliers</a>
  <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarNavAltMarkup">
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

### 2.4 Update Customer Views
**File:** `customer/views/supplier-list-all.html`
- Remove "Add a new supplier" button
- Remove edit/delete action buttons
- Keep only the read-only table of supplier information

**Delete these files:**
`supplier-add.html`, `supplier-form-fields.html`, `supplier-update.html`

### 2.5 Update Customer Configuration
**File:** `customer/index.js`
```javascript
// Read-only routes
app.get("/", (req, res) => res.render("home", {}));
app.get("/suppliers/", supplier.findAll);
app.get("/supplier-update/:id", supplier.findOne);

// Comment out write operations
// app.get("/supplier-add", ...);
// app.post("/supplier-add", ...);
// app.post("/supplier-update", ...);
// app.post("/supplier-remove/:id", ...);

// Set port for Docker
const app_port = process.env.APP_PORT || 8080;
app.listen(app_port, () => {
  console.log(`Coffee suppliers customer microservice is running on port ${app_port}.`);
});
```

---

## Step 3: Modify Employee Microservice (Admin Functions)

### 3.1 Update Employee Controller
**File:** `employee/app/controller/supplier.controller.js`  
Update redirect paths:
```javascript
// BEFORE
res.redirect('/suppliers');

// AFTER
res.redirect('/admin/suppliers');
```

### 3.2 Update Employee Routes
**File:** `employee/index.js`
```javascript
// Admin-prefixed routes
app.get('/admin/suppliers', suppliers.findAll);
app.get('/admin/supplier-update/:id', suppliers.findOne);
app.get('/admin/supplier-add', (req, res) => res.render('supplier-add', {}));
app.post('/admin/supplier-add', suppliers.create);
app.post('/admin/supplier-update', suppliers.update);
app.post('/admin/supplier-remove/:id', suppliers.remove);

// Home route
app.get('/admin', (req, res) => res.render('home'));

// Port for local testing
const app_port = process.env.APP_PORT || 8081;
app.listen(app_port, () => {
  console.log(`Coffee suppliers employee microservice is running on port ${app_port}.`);
});
```

### 3.3 Update Employee Views
Update form actions and navigation links to use `/admin` prefix.

### 3.4 Update Employee Navigation
**File:** `employee/views/nav.html`
```html
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
  <a class="navbar-brand" href="#">Manage coffee suppliers</a>
  <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarNavAltMarkup">
    <span class="navbar-toggler-icon"></span>
  </button>
  <div class="collapse navbar-collapse" id="navbarNavAltMarkup">
    <div class="navbar-nav">
      <a class="nav-link" href="/admin/suppliers">Administrator home</a>
      <a class="nav-link" href="/admin/supplier-add">Add a new supplier</a>
      <a class="nav-link" href="/">Customer home</a>
    </div>
  </div>
</nav>
```

---

## Step 4: Docker Container Setup

### 4.1 Create Customer Dockerfile
**File:** `customer/Dockerfile`
```dockerfile
FROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 8080
CMD ["npm", "run", "start"]
```

### 4.2 Create Employee Dockerfile
**File:** `employee/Dockerfile`
```dockerfile
FROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 8081
CMD ["npm", "run", "start"]
```

### 4.3 Build Docker Images
```bash
cd customer
docker build --tag customer .
cd ../employee
docker build --tag employee .
docker images
```

### 4.4 Run Containers with Database Configuration
Retrieve **RDS Database Details**: From AWS Console â†’ RDS â†’ Databases â†’ Note Endpoint, Username, Password, and Database Name (COFFEE).
```bash
# Run Customer Container
docker run -d --name customer_1 -p 8080:8080   -e APP_DB_HOST="your-rds-endpoint.us-east-1.rds.amazonaws.com"   -e APP_DB_USER="admin"   -e APP_DB_PASSWORD="lab-password"   -e APP_DB_NAME="COFFEE" customer

# Run Employee Container
docker run -d --name employee_1 -p 8081:8081   -e APP_DB_HOST="your-rds-endpoint.us-east-1.rds.amazonaws.com"   -e APP_DB_USER="admin"   -e APP_DB_PASSWORD="lab-password"   -e APP_DB_NAME="COFFEE" employee
```

### 4.5 Verify Container Status
```bash
docker ps
docker logs customer_1
docker logs employee_1
```

---

## Step 5: Testing Microservices

### 5.1 Test Customer Microservice (Read-Only)
```bash
curl http://localhost:8080/
curl http://localhost:8080/suppliers
```
**Expected:**
- Supplier list displays
- No â€œAdd new supplierâ€ button
- No edit/delete options
- Administrator link visible

### 5.2 Test Employee Microservice (Admin)
```bash
curl http://localhost:8081/admin/suppliers
```
**Expected:**
- Full admin interface with CRUD features
- â€œAdd new supplierâ€ button functional
- Customer home link available

### 5.3 Test Database Connectivity
```bash
docker exec customer_1 node -e "
 const mysql = require('mysql');
 const connection = mysql.createConnection({
   host: process.env.APP_DB_HOST,
   user: process.env.APP_DB_USER,
   password: process.env.APP_DB_PASSWORD,
   database: process.env.APP_DB_NAME
 });
 connection.connect(err => {
   if (err) console.log('Database connection failed:', err.message);
   else console.log('Database connected successfully');
   connection.end();
 });
"
```

---

## Step 6: Prepare for ECS Deployment

### 6.1 Standardize Employee Port
**File:** `employee/index.js`
```javascript
const app_port = process.env.APP_PORT || 8080;
```

**File:** `employee/Dockerfile`
```dockerfile
EXPOSE 8080
```

### 6.2 Rebuild Employee Image
```bash
cd employee
docker build --tag employee .
```

---

## Step 7: Commit and Push Changes
```bash
git status
git diff
git add .
git commit -m "feat: Complete microservices decomposition - Customer: read-only, port 8080 - Employee: /admin routes, CRUD enabled, port 8080 - Added Dockerfiles for both services - Tested containers locally and prepared for ECS deployment"
git push origin dev
```

---

## Troubleshooting
| Issue | Solution |
|--------|-----------|
| **Database connection failed** | Check container logs and verify environment variables |
| **Port conflicts** | Stop and remove existing containers: `docker stop customer_1 employee_1 && docker rm customer_1 employee_1` |
| **Application errors** | Check logs with `docker logs customer_1` or test endpoints using `curl` |
| **Missing dependencies** | Rebuild with `docker build --no-cache --tag customer .` |

---

## Conclusion

### Achievements
Decomposed monolithic app into two microservices  
Implemented clear separation of concerns (read-only vs read-write)  
Containerized both services using Docker  
Verified database connectivity  
Prepared both microservices for ECS deployment

### Architecture Summary
| Component | Port | Function |
|------------|------|-----------|
| Customer Microservice | 8080 | Read-only operations |
| Employee Microservice | 8080 | Full CRUD under /admin path |
| Shared Database | - | AWS RDS MySQL instance |
| Deployment | - | Docker containers, ECS-ready |
