# **Containerized Application Deployment on ECS with CI/CD Pipeline**  

## Overview
This project provides a comprehensive walkthrough for deploying a **containerized Spring Boot web application** to **AWS ECS Fargate**, leveraging key AWS services such as **GitHub**, **AWS CodePipeline**, **Amazon ECR**, **AWS CodeBuild**, and **AWS CodeDeploy**. The project ensures a secure, scalable, and automated deployment workflow with **CI/CD integration**, enabling seamless application updates and deployment management.

---

## **Table of Contents**  
1. [Architecture Diagram](#architecture-diagram)  
2. [Deploy Spring Boot Web App to AWS ECS Fargate](#deploy-spring-boot-web-app-to-aws-ecs-fargate)
   - [Part 1: Download Application Code](#part-1-Download-Application-Code)  
   - [Part 2: Networking and Security Configuration](#part-2-Networking-and-Security-Configuration)  
   - [Part 3: Amazon ECR Repository](#part-3-Amazon-ECR-Repository)  
   - [Part 4:  Amazon ECS Setup](#part-4-Amazon-ECS-Setup)
3. [Testing ECS Deployment](Testing-ECS-Deployment)
4. [CI/CD Pipeline Setup](#ci-cd-pipeline-setup)  
   - [Part 1: AWS CodeBuild](#part-1-aws-codebuild)  
   - [Part 2: AWS CodePipeline](#part-2-aws-codepipeline)  
6. [Testing the Deployment](#testing-the-deployment)  
7. [Troubleshooting Guide](#troubleshooting-guide)  
8. [Conclusion](#conclusion)  

---
## Architecture Diagram

<img src="https://github.com/gitkailash/Containerized-Application-Deployment-on-ECS-with-CI-CD-Pipeline/blob/master/assets/CI_CD Pipeline for Deploying a Spring Boot Application on AWS ECS Fargate.svg" alt="Architecture Diagram"/>


---

## Deploy Spring Boot Web App to AWS ECS Fargate

### Part 1: Download Application Code

1. Clone the GitHub repository:
   ```bash
   git clone https://github.com/gitkailash/Containerized-Application-Deployment-on-ECS-with-CI-CD-Pipeline.git
   ```
---

### Part 2: Networking and Security Configuration

#### **1. VPC:**
   - Use your default VPC.

#### **2. Subnets:**
   - Create two private subnets across two Availability Zones (AZs).

#### **3. NAT Gateway:**
   - Create a NAT Gateway in both AZs.
   - Allocate Elastic IPs (EIP) for each NAT Gateway.

#### **4. Route Tables:**
   - Create private route tables for each AZ.
   - Add routes:
     - Destination: `0.0.0.0/0`
     - Target: NAT Gateway of the respective AZ.
   - Associate private subnets with the corresponding route tables.

#### **5. Security Groups:**
   - **ECS-InternetFacing-LB:**
     - Allow `Custom TCP: 8080` from `0.0.0.0/0`.
   - **ECS-App-SG:**
     - Allow `Custom TCP: 8080` from `ECS-InternetFacing-LB` and `MyIP`.
   - **DB-SG:**
     - Allow `MySQL/Aurora` from `ECS-App-SG` and `ECS-InternetFacing-LB`.
     - Note: *Assumed that you already have RDS MySQL database. Follow this https://github.com/gitkailash/AWS-Three-Tier-Web-Architecture-Project.git link to create it*

#### **6. Database Configuration:**
   - Add the `DB-SG` to your database security group.

---

### Part 3: Amazon ECR Repository

1. Navigate to ECR and create a repository:
   - Repository Name: `paypal/spring-boot-app`

---

### Part 4: Amazon ECS Setup

#### **1. Create a Cluster:**
   - Cluster Name: `Paypal-App-Cluster`
   - Infrastructure: AWS Fargate.

#### **2. Create Task Definitions:**
   - Task Definition Family: `Paypal-task`
   - Launch Type: AWS Fargate
   - CPU: `0.5 vCPU`
   - Memory: `1 GB`
   - Container Details:
     - Name: `paypal-container`
     - Image URI: Your ECR repository URI.
     - Container Port: `8080`
     - Port Name: `http`
     - Environment Variables:
       - MYSQL_HOST: `Your database host`.
       - MYSQL_USER: `Your database username`.
       - MYSQL_PASSWORD: `Your database password`
#### **3. Create Services:**
   - Compute options: `Launch type`
   - Task Definition Family: `Paypal-task`
   - Revision: `LATEST`
   - Service Name: `Paypal-Service`
   - VPC: `default`
   - Subnets: `Both private subnets.`
   - Security Group: `ECS-App-SG`
   - Load Balancer:
     - Type: `Application Load Balancer.`
     - Load balancer name: ECS-ALB
     - Health check grace period: 300
     - Listener: Create new listener'
     - port `8080`
   - Target group: 
     - Target Group: `Create new target group`
     - Name: `ECS-ALB-TG`
     - Health Check Path: `/health`

#### **4. Update Load Balancer Security Group:**
   - Set the security group to `ECS-InternetFacing-LB` and remove others.

---

### Testing ECS Deployment

1. Copy the Load Balancer DNS name.
2. Test in your browser:
   ```
   ECS-ALB-123456789.us-east-1.elb.amazonaws.com:8080
   ```
3. Validate the application is running successfully.
   
*Validate Deployment*
![Login](/assets/login.png) 

---

## CI/CD Pipeline Setup

### Part 1: AWS CodeBuild

1. **Create a Build Project:**
   - Project Name: `Paypal-build`
   - Source Provider: `GitHub`
   - Credential: `Custom source Credential`
   - connection: `Choose your connection` *(Assumed you have connection to your github repo)*
   - Repository: `Select your repository`
   - Service role: `new service role` *(Note this role name because we will add more permission later)*
   - Additional configuration:
      - Hours: `0`
      - Minutes: `10`
      - Privileged Mode: `Enabled (for Docker builds)`
      - vpc: `default`
      - Subnets: `Both private subnets`
      - Security Groups: `ECS-InternetFacing-LB`
      - Environment Variables:
        - `AWS_DEFAULT_REGION`, `AWS_ACCOUNT_ID`, `REPOSITORY_URI`, `IMAGE_TAG`, `CONTAINER_NAME`, `IMAGE_REPO_NAME`, `MYSQL_HOST`, `MYSQL_USER`, `MYSQL_PASSWORD`.
      - Build Specification: `Use a buildspec file`

2. **Update IAM Role:**
   - Add the following policies to the CodeBuild service role:
     - `AmazonEC2ContainerRegistryFullAccess`
     - `AmazonEC2ContainerRegistryPowerUser`
   - Remember: *(Note this role name because we will add more permission later)*

3. **Monitor the Build:**
   - Verify the Docker image is pushed to ECR.

   *Validate Build Logs*
![Build-Success](/assets/Build-logs.png) 

---

### AWS CodePipeline

1. **Create a Pipeline:**
   - Creation options: `Build custom pipeline`
   - Pipeline Name: `Paypal-ecs-pipeline`
   - Source Provider: `GitHub`
   - connection: `Choose your connection`
   - Repository name: `Your github repo`
   - default Branch: `Your github branch`
   - Build provider: `Other build providers`
   - Select: `AWS CodeBuild`
   - Project name: `Paypal-build`
   - Environment Variables:
        - `AWS_DEFAULT_REGION`, `AWS_ACCOUNT_ID`, `REPOSITORY_URI`, `IMAGE_TAG`, `CONTAINER_NAME`, `IMAGE_REPO_NAME`, `MYSQL_HOST`, `MYSQL_USER`, `MYSQL_PASSWORD`.
   - Build type: `Single build`
   - Deploy Provider: `Amazon ECS`
   - Cluster Name: `Paypal-App-Cluster`
   - Service Name: `Paypal-Service`.

3. **Add Manual Approval Stage:**
   *Add stage after Build*
   - Stage Name: `Review`
   - Click add action group
   - Action Name: `Manual-Approval`
   - Action provider: `Select Manual Approval`
     Note: *Review stage will wait for your approval.*

   *Validate Review stage*
   ![Review-Stage](/assets/Review-stage.png)
   
     
5. **Test Pipeline:**
   - Make changes to your repository and push to GitHub.
   - Monitor the pipeline execution.
   - Validate changes reflected in the application.

   *After Changed Pushed*
    *Pipeline Wating For Approval*
   ![Waiting-for-approval](/assets/approval-details.png)

   *Pipeline After Approval*
   ![pipeline-after-approval](/assets/Pipeline-after-approved.png)

   *Output After Changed Push (Pipeline Test Added in Login page)*
   ![Change-after-new-push](/assets/pipeline-test-after-change-push.png)
   
---

## Troubleshooting Steps

1. **Build Failures:**
   - Check `CloudWatch Logs` for errors.
   - Ensure IAM roles have the correct permissions.
2. **Deployment Issues:**
   - Verify ECS service task definitions and security group rules.
   - Ensure NAT Gateway and subnets are correctly configured.
3. **Pipeline Failures:**
   - Confirm GitHub connection is active.
   - Validate `buildspec.yml` syntax and environment variables.

---

## Conclusion

This document provides a detailed guide to deploying a Spring Boot application on **AWS ECS Fargate** with a secure and automated **CI/CD pipeline**. By following the outlined steps, you’ve learned to containerize applications, automate builds, and manage infrastructure using AWS services like **CodePipeline**, **CodeBuild**, and **ECR**. The guide emphasizes secure networking with **VPCs**, **NAT Gateways**, and **Security Groups**, ensuring scalability, reliability, and streamlined deployments.  

The project highlights the benefits of cloud-native strategies, such as serverless resource management, automated workflows, and robust security. Future improvements could include adding monitoring with **Amazon CloudWatch**, enabling rollback mechanisms, and expanding to multi-region deployments. This guide equips you to build secure, scalable, and efficient cloud-native solutions, setting the foundation for modern application development. 🚀

---

### Screenshots
*Validate Deployment*
![Login](/assets/login.png) 

 *Validate Build Logs*
![Build-Success](/assets/Build-logs.png)

*Validate Review stage*
![Review-Stage](/assets/Review-stage.png)

*After Changed Pushed Pipeline Wating For Approval*
![Waiting-for-approval](/assets/approval-details.png)

*Review Approved*
![Approved](/assets/Approved.pngg)

*Pipeline After Approval*
![pipeline-after-approval](/assets/Pipeline-after-approved.png)

*Output After Changed Push (Pipeline Test Added in Login page)*
![Change-after-new-push](/assets/pipeline-test-after-change-push.png)

*Health Checked*

![Health-Checked](/assets/health-check.PNG)

*Dashboard*
![Dashboard](/assets/dashboard.png)

*Paypal*
![Paypal](/assets/paypal.png)

*Swagger-1*
![Swagger-1](/assets/swagger-1.png)

*Swagger-2*
![Swagger-2](/assets/swagger-2.png)


---

## Notes  

⚠️ **Important Reminder**:  
"Because we all love the thrill of an unexpected AWS bill, don't forget to *not* delete your created services after testing. Who doesn't enjoy explaining a hefty cloud bill to their manager? But hey, if you're into that sort of thing, go ahead and leave it running. 😉" 

---

## License
This project is licensed under the MIT License.
