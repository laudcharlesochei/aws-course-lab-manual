# Project Overview and AWS Microservices Implementation

## Project Overview and Objectives

In this project, you're challenged to use at least 11 AWS offerings, including some that might be new to you, to build a microservices and continuous integration and continuous development (CI/CD) solution. Throughout various AWS Academy courses, you have completed hands-on labs. You have used different AWS services and features to build a variety of solutions.

In many parts of the project, step-by-step guidance is not provided. This is intentional. These specific sections of the project are meant to challenge you to practice skills that you have acquired throughout your learning experiences prior to this project. In some cases, you might be challenged to use resources to independently learn new skills.

### Learning Objectives

By the end of this project, you should be able to do the following:

- Recognize how a Node.js web application is coded and deployed to run and connect to a relational database where the application data is stored.
- Create an AWS Cloud9 integrated development environment (IDE) and a code repository (repo) in which to store the application code.
- Split the functionality of a monolithic application into separate containerized microservices.
- Use a container registry to store and version control containerized microservice Docker images.
- Create code repositories to store microservice source code and CI/CD deployment assets.
- Create a serverless cluster to fulfill cost optimization and scalability solution requirements.
- Configure an Application Load Balancer and multiple target groups to route traffic between microservices.
- Create a code pipeline to deploy microservices containers to a blue/green cluster deployment.
- Use the code pipeline and code repository for CI/CD by iterating on the application design.

## The Lab Environment and Monitoring Your Budget

This environment is long lived. When the session timer runs to 0:00, the session will end, but any data and resources that you created in the AWS account will be retained. If you later launch a new session (for example, the next day), you will find that your work is still in the lab environment. Also, at any point before the session timer reaches 0:00, you can choose **Start Lab** again to extend the current session time.

**Important:** Monitor your lab budget in the lab interface. When you have an active lab session, the latest known remaining budget information displays at the top of this screen. The remaining budget that you see might not reflect your most recent account activity. If you exceed your lab budget, your lab account will be disabled, and all progress and resources will be lost. If you exceed the budget, please contact your educator for assistance.

If you ever want to reset the lab environment to the original state, before you started the lab, use the **Reset** option above these instructions. Note that this won't reset your budget.

**Warning:** If you reset the environment, you will permanently delete everything that you have created or stored in this AWS account.

### AWS Service Restrictions

In this lab environment, access to AWS services and service actions are restricted to the ones that are needed to complete the project. You might encounter errors if you attempt to access other services or perform actions beyond the ones that are described in this lab.

## Scenario

The owners of a café corporation with many franchise locations have noticed how popular their gourmet coffee offerings have become.

Customers (the café franchise location managers) cannot seem to get enough of the high-quality coffee beans that are needed to create amazing cappuccinos and lattes in their cafés.

Meanwhile, the employees in the café corporate office have been challenged to consistently source the highest-quality coffee beans. Recently, the leaders at the corporate office learned that one of their favorite coffee suppliers wants to sell her company. The café corporate managers jumped at the opportunity to buy the company. The acquired coffee supplier runs a coffee supplier listings application on an AWS account.

![Current Monolithic Architecture](./images/scenario-one.png)
*Figure 1: Current monolithic application architecture showing the single-tier design*


<div align="center">
  <img src="https://github.com/laudcharlesochei/aws-course-lab-manual/blob/master/images/scenario-one.png" alt="Current Monolithic Architecture" width="600">
  <br>
  <em>Figure 1: Current monolithic application architecture - all components run as a single unit</em>
</div>

The coffee suppliers application currently runs as a monolithic application. It has reliability and performance issues. That is one of the reasons that you have recently been hired to work in the café corporate office. In this project, you perform tasks that are associated with software development engineer (SDE), app developer, and cloud support engineer roles.

You have been tasked to split the monolithic application into microservices, so that you can scale the services independently and allocate more compute resources to the services that experience the highest demand, with the goal of avoiding bottlenecks. A microservices design will also help avoid single points of failure, which could bring down the entire application in a monolithic design. With services isolated from one another, if one microservice becomes temporarily unavailable, the other microservices might remain available.

You have also been challenged to develop a CI/CD pipeline to automatically deploy updates to the production cluster that runs containers, using a blue/green deployment strategy.

## Solution Requirements

The solution must meet the following requirements:

| Requirement | Description |
|-------------|-------------|
| **R1 - Design** | The solution must have an architecture diagram. |
| **R2 - Cost optimized** | The solution must include a cost estimate. |
| **R3 - Microservices-based architecture** | Ensure that the solution is functional and deploys a microservice-based architecture. |
| **R4 - Portability** | The solution must be portable so that the application code isn't tied to running on a specific host machine. |
| **R5 - Scalability/resilience** | The solution must provide the ability to increase the amount of compute resources that are dedicated to serving requests as usage patterns change, and the solution must use routing logic that is reliable and scalable. |
| **R6 - Automated CI/CD** | The solution must provide a CI/CD pipeline that can be automatically invoked when code is updated and pushed to a version-controlled code repository. |

## Lab Project Tips

- Knowledge of Linux Bash commands would be helpful, but it isn't mandatory.
- **Tip:** One online Linux Bash reference is the Linux Man Pages on linux.die.net. You can search or browse for commands to learn more about them.
- Finally, you are encouraged to be resourceful as you complete this project. For example, reference the AWS Documentation or a search engine if you need an answer to a technical question.
- As you work through the project, you will find other tips to help you complete specific phases or tasks.

## Approach

The following table describes the phases of the project:

| Phase | Detail | Solution Requirement |
|-------|--------|---------------------|
| 1 | Create an architecture diagram and cost estimate for the solution. | R1, R2 |
| 2 | Analyze the design of the monolithic application and test the application. | R3 |
| 3 | Create a development environment on AWS Cloud9, and check the monolithic source code into CodeCommit. | R3 |
| 4 | Break the monolithic design into microservices, and launch test Docker containers. | R3, R4 |
| 5 | Create ECR repositories to store Docker images. Create an ECS cluster, ECS task definitions, and CodeDeploy application specification files. | R3 |
| 6 | Create target groups and an Application Load Balancer that routes web traffic to them. | R5 |
| 7 | Create ECS services. | R5 |
| 8 | Configure applications and deployments groups in CodeDeploy, and create two CI/CD pipelines by using CodePipeline. | R5, R6 |
| 9 | Modify your microservices and scale capacity, and use the pipelines that you created to deploy iterative improvements to production by using a blue/green deployment strategy. |

## Sample Application Structure

Here's a basic example of a Node.js application structure you'll be working with:

```javascript
// server.js - Basic Express server
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({ message: 'Coffee Suppliers API', status: 'running' });
});

app.listen(port, () => {
  console.log(`Coffee suppliers app running on port ${port}`);
});
```

## Key AWS Services You'll Use

```yaml
Core Services:
  - AWS Cloud9: Development environment
  - Amazon ECR: Container registry
  - Amazon ECS: Container orchestration
  - Application Load Balancer: Traffic routing
  - AWS CodeCommit: Source code repository
  - AWS CodePipeline: CI/CD pipeline
  - AWS CodeDeploy: Blue/green deployments
  - Amazon RDS: Database service
  - IAM: Security and permissions
```

**How to Copy Code:**
- Hover over any code block above
- Click the "Copy" button that appears in the top-right corner
- The code will be copied to your clipboard
- Paste it into your development environment

## Next Steps

Proceed to the next unit to begin Phase 1: Creating your architecture diagram and cost estimate.
