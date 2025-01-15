# Containerized-Application-Deployment-on-ECS-with-CI-CD-Pipeline
This guide outlines the step-by-step process to deploy a Spring Boot web application to AWS ECS Fargate using GitHub, AWS networking, security configurations, and CI/CD pipelines.

---

## Phase 1: Deploy Spring Boot Web App to AWS ECS Fargate

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
*Screenshots*

---

## Phase 2: CI/CD Pipeline Setup

### Part 1: AWS CodeBuild

1. **Create a Build Project:**
   - Project Name: `Paypal-build`
   - Source Provider: GitHub.
   - Repository: Link to your repository.
   - Privileged Mode: Enabled (for Docker builds).
   - Subnets: Both private subnets.
   - Security Groups: `ECS-InternetFacing-LB`
   - Environment Variables:
     - `AWS_DEFAULT_REGION`, `AWS_ACCOUNT_ID`, `REPOSITORY_URI`, `IMAGE_TAG`, `CONTAINER_NAME`, `IMAGE_REPO_NAME`, `MYSQL_HOST`, `MYSQL_USER`, `MYSQL_PASSWORD`.
   - Build Specification: Use a `buildspec.yml` file.

2. **Update IAM Role:**
   - Add the following policies to the CodeBuild service role:
     - `AmazonEC2ContainerRegistryFullAccess`
     - `AmazonEC2ContainerRegistryPowerUser`

3. **Monitor the Build:**
   - Verify the Docker image is pushed to ECR.

---

### Part 2: AWS CodePipeline

1. **Create a Pipeline:**
   - Pipeline Name: `Paypal-ecs-pipeline`
   - Source Provider: GitHub.
   - Build Provider: AWS CodeBuild (select `Paypal-build`).
   - Deploy Provider: Amazon ECS.
     - Cluster Name: `Paypal-App-Cluster`
     - Service Name: `Paypal-Service`.

2. **Add Manual Approval Stage:**
   - Stage Name: `Review`.
   - Action Name: `Manual-Approval`.

3. **Test Pipeline:**
   - Make changes to your repository and push to GitHub.
   - Monitor the pipeline execution.
   - Validate changes reflected in the application.

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

This document provides a comprehensive guide to deploying a Spring Boot application on AWS ECS Fargate, along with CI/CD pipeline automation. Follow the outlined steps to ensure a seamless deployment and robust infrastructure setup.

---

## Screenshots

1. Load Balancer Test: `alb-test.jpg`
2. Build Success: `build-success.jpg`
3. Review Stage: `review-stage.jpg`
4. Change Reflected: `change-reflected.png`


