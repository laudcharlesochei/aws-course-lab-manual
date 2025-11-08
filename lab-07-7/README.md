# Lab: Configuring, Deploying and Testing Microservices Using GitHub Codespaces and Docker  

## Lab Overview

In this lab, you will transform the monolithic Coffee Suppliers application into two separate microservices  - a **Customer-facing (read-only)** service and an **Employee/Admin (full CRUD)** service. You will containerize both services using Docker and test them locally or in **GitHub Codespaces**.

Since AWS Cloud9 is no longer available for new users, this lab uses **GitHub Codespaces** as the cloud development environment and **GitHub** for version control.  
You will also prepare the setup for **Amazon ECS deployment**.

---

## Key Technologies
- GitHub Codespaces (Cloud IDE)
- Node.js Microservices
- Docker Containers
- MySQL Database (AWS RDS or Local)
- GitHub Version Control

---

## Architecture Overview

| Component | Description | Port |
|------------|--------------|------|
| Customer Microservice | Read-only access to supplier data | 8080 |
| Employee Microservice | Full CRUD operations under /admin prefix | 8081 |
| Database | AWS RDS MySQL or local MySQL instance | - |

---

## Step 1: Environment Setup

### 1.1 Clone and Open Repository
```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/coffee-suppliers-microservices
cd coffee-suppliers-microservices
```

**Option A (Local):**  
Open in Docker-enabled environment.

**Option B (Cloud):**  
Open repository in **GitHub Codespaces**.  
Click **Code â†’ Codespaces â†’ Create codespace on main.**

### 1.2 Verify Environment
```bash
node --version
npm --version
git config --list
```

### 1.3 Verify Project Structure
```
microservices/
â”œâ”€â”€ customer/
â”‚   â”œâ”€â”€ app/
â”‚   â”œâ”€â”€ views/
â”‚   â”œâ”€â”€ index.js
â”‚   â””â”€â”€ package.json
â””â”€â”€ employee/
    â”œâ”€â”€ app/
    â”œâ”€â”€ views/
    â”œâ”€â”€ index.js
    â””â”€â”€ package.json
```

---

## Step 2: Database Configuration

### Option A â€“ Connect to AWS RDS
```bash
sudo apt update
sudo apt install mysql-client -y
mysql -h YOUR_RDS_ENDPOINT -u admin -p
```

### Option B â€“ Local MySQL Setup
```bash
sudo apt update
sudo apt install mysql-server -y
sudo service mysql start
sudo mysql -u root
```

**Create database and user:**
```sql
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

INSERT INTO suppliers (name, email, phone, description, address, city, state)
VALUES
('Mountain Coffee Co.', 'contact@mountaincoffee.com', '555-1234', 'High-altitude beans', '123 Hill Rd', 'Denver', 'CO'),
('Valley Roasters', 'info@valleyroasters.com', '555-5678', 'Organic fair-trade roasters', '45 Valley Dr', 'Portland', 'OR'),
('Urban Brew', 'sales@urbanbrew.com', '555-7890', 'Premium city blends', '9 Main St', 'Seattle', 'WA');
```

### 2.3 Configure Environment Variables

**Customer Microservice (.env)**
```
APP_DB_HOST=localhost
APP_DB_USER=coffee_user
APP_DB_PASSWORD=password
APP_DB_NAME=COFFEE
APP_PORT=8080
```

**Employee Microservice (.env)**
```
APP_DB_HOST=localhost
APP_DB_USER=coffee_user
APP_DB_PASSWORD=password
APP_DB_NAME=COFFEE
APP_PORT=8081
```

---

## Step 3: Modify Customer Microservice (Read-Only)

### 3.1 Update Controller
**File:** `customer/app/controller/supplier.controller.js`

Keep:
- `Supplier.getAll`
- `Supplier.findById`

Remove:
- `Supplier.create`
- `Supplier.updateById`
- `Supplier.remove`

### 3.2 Update Views
- Remove `supplier-add.html`, `supplier-update.html`, and any â€œAddâ€, â€œEditâ€, or â€œDeleteâ€ buttons.
- Keep only the read-only supplier table.

### 3.3 Update Navigation
**File:** `customer/views/nav.html`  
Keep only **Home** and **Supplier List** links.  
Add **Admin Portal** link (points to Employee microservice).

---

## Step 4: Modify Employee Microservice (Admin)

### 4.1 Update Routes and Controllers
**File:** `employee/app/controller/supplier.controller.js`

Update all redirects:
```javascript
// Before
res.redirect('/suppliers');

// After
res.redirect('/admin/suppliers');
```

### 4.2 Update Index.js
Add `/admin` prefix to all routes:
```javascript
app.get('/admin', (req, res) => res.render('home'));
app.get('/admin/suppliers', ...);
app.get('/admin/supplier-add', ...);
app.post('/admin/supplier-add', ...);
app.get('/admin/supplier-update/:id', ...);
app.post('/admin/supplier-update', ...);
app.post('/admin/supplier-remove/:id', ...);
```

### 4.3 Update Navigation
**File:** `employee/views/nav.html`  
- Update links to `/admin/...` paths.  
- Add logo or branding for the admin panel.

---

## Step 5: Docker Container Setup

### Customer Dockerfile
```dockerfile
FROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 8080
CMD ["npm", "run", "start"]
```

### Employee Dockerfile
```dockerfile
FROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 8081
CMD ["npm", "run", "start"]
```

### Build and Run Containers
```bash
docker build -t customer ./customer
docker build -t employee ./employee

docker run -d --name customer_1 -p 8080:8080 customer
docker run -d --name employee_1 -p 8081:8081 employee

docker ps
```

---

## Step 6: Testing and Verification

### Customer Microservice
Visit: **http://localhost:8080**
- âœ… Supplier list loads  
- âŒ No Add/Edit/Delete buttons

### Employee Microservice
Visit: **http://localhost:8081/admin**
- âœ… CRUD operations available  
- âœ… Admin dashboard functional

### Database Connection Test
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
  if (err) console.log('Database connection failed');
  else console.log('Database connected successfully');
  connection.end();
});
"
```

---

## Step 7: Commit and Push Changes
```bash
git status
git add .
git commit -m "feat: microservices containerization and testing complete"
git push origin main
```

---

## Troubleshooting

| Issue | Possible Solution |
|--------|--------------------|
| Database connection failed | Check `.env` variables and container logs |
| Port conflicts | Run `docker stop $(docker ps -q)` and restart containers |
| Missing dependencies | Rebuild image using `--no-cache` flag |
| Application errors | Run `docker logs <container_name>` |

---

## Conclusion

âœ… Decomposed monolithic app into two microservices  
âœ… Implemented clear separation of concerns (read-only vs admin)  
âœ… Containerized services using Docker  
âœ… Verified local and database connectivity  
âœ… Prepared for ECS deployment  

### Next Steps:
- Push Docker images to Amazon ECR  
- Deploy to Amazon ECS  
- Configure Application Load Balancer  
- Enable Auto-scaling and Service Discovery
