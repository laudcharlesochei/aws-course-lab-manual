# Configuring Microservices and Testing in Docker Containers

This lab uses **GitHub Codespaces** as the development environment and **GitHub** for version control.

Phase 4: Using VS
Code with GitHub
Codespaces for Codespace
Microservices
Dep loyment
Lab Overview
This lab guides you through decomposing a
monolithic coffee suppliers application into two
microservices (customer-facing and
employee/admin) and containerizing them using
Docker.
Since AWS Cloud9 is no longer available to new
customers and AWS has discontinued
onboarding new customers to CodeCommit,
we'll use GitHub Codespaces Codespace as our
development environment and GitHub for
version control.
Key Technologies Used
GitHub Codespaces (Cloud Development Codespace
Environment)
GitHub Codespaces
Docker Containers
Node.js Microservices
MySQL Database (AWS RDS or Local)
GitHub for Version Control
Architecture Overview08/11/2025, 20:39 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-6/ 1/11
Component Description Port
Customer
MicroserviceRead-only access
to supplier data8080
Employee
MicroserviceFull CRUD
operations with
/admin route
prefix8081
DatabaseAWS RDS MySQL
or local MySQL
instance-
Step 1: Environment
Setup in GitHub
Codespaces Codespace
1.1 Access GitHub Codespaces Codespace
1. Navigate to your GitHub repository
2. Click the Code button and select the
Codespaces Codespace tab
3. Click Create codespace on main codespace
4. Wait for the environment to initialize
1.2 Verify Dev elopment
Environment
# Check Node.js and npm versions
node --version
npm --version
# Verify Git configuration
git config --listbash08/11/2025, 20:39 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-6/ 2/11
1.3 Upload Project Files to
Codespaces Codespace
In Codespaces File Explorer, right-click and Codespace
select Upload
Select your project files/folders from your
local machine
Alternatively, drag and drop files directly
into the file explorer
Verify file structure:
Step 2: Database
Configuration
2.1 Option A: Connect to AWS
RDS Databasemicroservices/
â”œâ”€â”€ customer/
â”‚   â”œâ”€â”€ app/
â”‚   â”œâ”€â”€ views/
â”‚   â””â”€â”€ package.json
â””â”€â”€ employee/
    â”œâ”€â”€ app/
    â”œâ”€â”€ views/
    â””â”€â”€ package.json
sudo apt update
sudo apt install mysql-client -y
# Test RDS connectionbash08/11/2025, 20:39 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-6/ 3/11
2.2 Option B: Set Up Local
MySQL Databasemysql -h YOUR_RDS_ENDPOINT -u adm
# Password: lab-password
sudo apt update
sudo apt install mysql-server -y
sudo service mysql start
sudo systemctl enable mysql
sudo mysql -u root << 'EOF'
CREATE DATABASE COFFEE;
CREATE USER 'coffee_user'@'localho
GRANT ALL PRIVILEGES ON COFFEE.* T
FLUSH PRIVILEGES;
USE COFFEE;
CREATE TABLE suppliers (
    id INT AUTO_INCREMENT PRIMARY 
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(50),
    description TEXT,
    address VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(50),
    created_at TIMESTAMP DEFAULT C
);
INSERT INTO suppliers (name, emai
('Mountain Coffee Co.', 'contact@m
('Valley Roasters', 'info@valleyro
('Urban Brew', 'sales@urbanbrew.co
EOFbash08/11/2025, 20:39 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-6/ 4/11
2.3 Configure Environment
Variables
Create .env files for both microservices:
Customer Microservice (.env):
Employee Microservice (.env):
Step 3: Modify Customer
Microservice (Read-Only)
3.1 Update Supplier Model
File:
customer/app/models/supplier.model.jscd /workspaces/your-repo/microserv
cat > .env << 'EOF'
APP_DB_HOST=localhost
APP_DB_USER=coffee_user
APP_DB_PASSWORD=password
APP_DB_NAME=COFFEE
APP_PORT=8080
EOFbash
cd ../employee
cat > .env << 'EOF'
APP_DB_HOST=localhost
APP_DB_USER=coffee_user
APP_DB_PASSWORD=password
APP_DB_NAME=COFFEE
APP_PORT=8081
EOFbash08/11/2025, 20:39 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-6/ 5/11
3.2 Update Navigation
File: customer/views/nav.html sql  
   
    name  suppliername
    email  supplieremail
    phone  supplierphone
    description  supplierd
Supplier     
    sql
         err 
             err
            
        
         res
    
Supplier    
    sql
         err 
            err 
            
        
         reslength 
             res
            
        
          
    
moduleexports  Supplierconst=require("./db.js");
constSupplier=function(supplie
this.= .;
this.= .;
this.= .;
this. = .
};
.getAll=result=>{
.query("SELECT * FROM supp
if(){
result(null,);
return;
}
result(null,);
});
};
.findById=(id result, )
.query(`SELECT * FROM supp
if(){
result(,null);
return;
}
if(.){
result(null,[0]);
return;
}
result({kind:"not_found
});
};
. = ;javascript08/11/2025, 20:39 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-6/ 6/11
3.3 Update Supplier List View
(Read-Only)
Remove "Add new supplier" button
Remove edit/delete options
Keep only the supplier display table
Step 4: Modify Employee
Microservice (Admin
Functions)
4.1 Update Employee
Controller
File:
employee/app/controller/supplier.controller.js
Update all redirect calls to use /admin prefix:    
    
        
    
    
        
            
            
            
        
     nav<classnavbar navbar-expand- ="
  a<classnavbar-brand =" "href=
 
            
             button< classnavbar-toggler ="
data-target#navbarN ="
aria-expandedfalse=""
 span<classnavbar-toggl ="
button</>
 div<classcollapse navbar-co ="
 div<classnavbar-nav =" ">
  a<classnav-link =" "h
  a<classnav-link =" "h
  a<classnav-link =" "h
div</>
div</>
nav</>html08/11/2025, 20:39 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-6/ 7/11
4.2 Update Employee Routes
File: employee/index.js
4.3 Update Employee Viewsres
res// Change all redirects from:
.redirect("/suppliers");
// To:
.redirect("/admin/suppliers");javascript
 express  
app    
    res  
app  supp
app  
    res  
app  s
app
app
app
 app_port  processenv
app app_port   
    consolerequire('dotenv').config();
const =require("express"
// ... other imports
// ADD /admin PREFIX TO ALL ROUTES
.get("/admin",(req res,)=>{
.render("home",{});
});
.get("/admin/suppliers/",
.get("/admin/supplier-add",(r
.render("supplier-add",{}
});
.post("/admin/supplier-add",
.get("/admin/supplier-update/:
.post("/admin/supplier-update"
.post("/admin/supplier-remove/
// Update port
const = ..APP_
.listen( ,()=>{
.log(`Coffee suppliers 
});javascript08/11/2025, 20:39 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-6/ 8/11
Update form actions in HTML files to use
/admin prefix:
File: employee/views/supplier-add.html
File: employee/views/supplier-
update.html
4.4 Update Employee
Navigation
File: employee/views/nav.html form<action/admin/supplier-add ="html
 form<action/admin/supplier-upd ="
 form<action/admin/supplier-remo ="html
    
    
    
        
            
                
                
                
            
        
     nav<classnavbar navbar-expand- ="
  img<src/img/espresso.jpg =" "w
div<> a<classnavbar-brand p ="
 div<classcollapse navbar-co ="
 ul<classnavbar-nav mr- ="
 li<classnav-item a ="
 a<classnav-lin ="
 a<classnav-lin ="
 a<classnav-lin ="
li</>
ul</>
div</>
nav</>html08/11/2025, 20:39 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-6/ 9/11
Step 5: Create Docker
Containers
Customer Dockerfile:
Employee Dockerfile:
Step 6: Test and VerifyFROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 8080
CMD ["npm", "run", "start"]dockerfile
FROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 8081
CMD ["npm", "run", "start"]dockerfile
docker build --tag customer ./cus
docker build --tag employee ./emp
docker run -d --name customer_1 -
docker run -d --name employee_1 -
docker psbash08/11/2025, 20:39 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-6/ 10/11
Conclusion
Both microservices configured and
containerized
Database connection verified
Admin panel functional with CRUD operations
Code pushed to GitHub ready for ECS
deployment08/11/2025, 20:39 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-6/ 11/11


Lab 07-4: Configuring
Microservices and
Testing in Docker
Containers
Lab Overview
In this phase, you will transform the monolithic
coffee suppliers application into two separate
microservices(customer and employee)
containerize them using Docker, and test them
locally. This setup replaces AWS Cloud9 with
GitHub Codespaces and uses Docker Desktop for
local container testing.
Prerequisites
Before beginning, ensure you have:
A GitHub account with repository access
Docker Desktop installed locally
GitHub Codespaces with the Docker extension
Basic knowledge of Node.js and Docker
Step 1: Environment
Setup
1.1 Clone Your Repository08/11/2025, 20:40 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-5/ 1/13
1.2 Open in GitHub Codespaces or
Codespaces
Option A (Local): Open the folder in VS
Code.
Option B (Codespaces): Create a new
codespace from your GitHub repository.
1.3 Verify Project Structure
Step 2: Modify Customer
Microservice (Read-Only)
2.1 Update Customer
Controllergit clone https://github.com/YOUR
cd coffee-suppliers-microservicesbash
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
    â””â”€â”€ package.json08/11/2025, 20:40 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-5/ 2/13
File:
customer/app/controller/supplier.controller.js
2.2 Update Customer Model
File:
customer/app/models/supplier.model.js
Keep: Supplier.getAll,
Supplier.findById
Remove: Supplier.create,
Supplier.updateById,
Supplier.remove Supplier  
  body validationResult  
exports     
  Supplier   
     err
      res   
    
      res
  
exports     
  Supplier reqparamsid
     err 
       errkind  
        res  
        
        res   
      
      res
  const =require("../mode
const{, }
.findAll=(req res,)=>{
.getAll((err data,)=>
if()
.render("500",{message
else
.render("supplier-list-a
});
};
.findOne=(req res,)=>{
.findById(..
if(){
if(.==="not_found
.status(404).send({me
}else{
.render("500",{messa
}
}else.render("supplier-u
});
};javascript08/11/2025, 20:40 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-5/ 3/13
2.3 Update Customer
Navigation
File: customer/views/nav.html
2.4 Update Customer Views
File: customer/views/supplier-list-
all.html
Remove "Add a new supplier" button
Remove edit/delete action buttons
Keep only the read-only table of supplier
information
Delete these files: supplier-add.html,
supplier-form-fields.html, supplier-
update.html
2.5 Update Customer
Configuration
File: customer/index.js  
  
    
  
  
    
      
      
      
    
   nav<classnavbar navbar-expand- ="
  a<classnavbar-brand =" "href#="
  button< classnavbar-toggler =" "
 span<classnavbar-toggler-i ="
button</>
 div<classcollapse navbar-col ="
 div<classnavbar-nav =" ">
  a<classnav-link =" "href/="
  a<classnav-link =" "href/="
  a<classnav-link =" "href/="
div</>
div</>
nav</>html08/11/2025, 20:40 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-5/ 4/13
Step 3: Modify Employee
Microservice (Admin
Functions)
3.1 Update Employee
Controller
File:
employee/app/controller/supplier.controller.js
Update redirect paths:app    res
app  supplierf
app  s
 app_port  processenv
app app_port   
  console// Read-only routes
.get("/",(req res,)=>.re
.get("/suppliers/", .
.get("/supplier-update/:id",
// Comment out write operations
// app.get("/supplier-add", ...);
// app.post("/supplier-add", ...)
// app.post("/supplier-update", .
// app.post("/supplier-remove/:id
// Set port for Docker
const = ..APP_
.listen( ,()=>{
.log(`Coffee suppliers c
});javascript
res// BEFORE
.redirect('/suppliers');javascript08/11/2025, 20:40 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-5/ 5/13
3.2 Update Employee Routes
File: employee/index.js
3.3 Update Employee Views
Update form actions and navigation links to use
/admin prefix.
3.4 Update Employee
Navigation
File: employee/views/nav.htmlres// AFTER
.redirect('/admin/suppliers');
app  suppl
app
app  
app  s
app
app
app    r
 app_port  processenv
app app_port   
  console// Admin-prefixed routes
.get('/admin/suppliers',
.get('/admin/supplier-update/:
.get('/admin/supplier-add',(r
.post('/admin/supplier-add',
.post('/admin/supplier-update'
.post('/admin/supplier-remove/
// Home route
.get('/admin',(req res,)=>
// Port for local testing
const = ..APP_
.listen( ,()=>{
.log(`Coffee suppliers em
});javascript08/11/2025, 20:40 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-5/ 6/13
Step 4: Docker Container
Setup
4.1 Create Customer
Dockerfile
File: customer/Dockerfile
4.2 Create Employee
Dockerfile  
  
    
  
  
    
      
      
      
    
   nav<classnavbar navbar-expand- ="
  a<classnavbar-brand =" "href#="
  button< classnavbar-toggler =" "
 span<classnavbar-toggler-i ="
button</>
 div<classcollapse navbar-col ="
 div<classnavbar-nav =" ">
  a<classnav-link =" "href/="
  a<classnav-link =" "href/="
  a<classnav-link =" "href/="
div</>
div</>
nav</>html
FROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 8080
CMD ["npm", "run", "start"]dockerfile08/11/2025, 20:40 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-5/ 7/13
File: employee/Dockerfile
4.3 Build Docker Images
4.4 Run Containers with
Database Configuration
Retrieve RDS Database Details: From AWS
Console Ã¢ â€  Ê¼ RDS Ã¢ â€  Ê¼ Databases Ã¢ â€  Ê¼ Note
Endpoint, Username, Password, and Database
Name (COFFEE).
4.5 Verify Container StatusFROM node:11-alpine
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . .
RUN npm install
EXPOSE 8081
CMD ["npm", "run", "start"]dockerfile
cd customer
docker build --tag customer .
cd ../employee
docker build --tag employee .
docker imagesbash
# Run Customer Container
docker run -d --name customer_1 -
# Run Employee Container
docker run -d --name employee_1 -bash08/11/2025, 20:40 My Lab Manual
https://laudcharlesochei.github.io/aws-course-lab-manual/#/lab-07-5/ 8/13
Step 5: Testing
Microservices
5.1 Test Customer
Microservice (Read-Only)
Expected:
Supplier list displays
No "Add new supplierÃ¢â‚¬
