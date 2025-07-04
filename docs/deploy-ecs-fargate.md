# Deploying Node.js Server with ECS Fargate and ALB

## 1. VPC and Networking Setup

* Create a VPC with at least **2 public subnets**, one in each Availability Zone (AZ):

  * `public-subnet-1a`
  * `public-subnet-1b`
* Attach a **Route Table** and associate it with both public subnets.
* Connect the Route Table to an **Internet Gateway** for outbound internet access.



---

## 2. EC2 Instance Setup

* Launch an EC2 instance on one of the public subnet, dont forget to assign public ip
* For security group, allow inbound http, ssh, and port 3000 (where we will run our docker container)
* This will be used for building and pushing Docker images to ECR Repo
* SSH into this instance
* Install:
  * AWS CLI
  * Git
  * Docker
* Clone the app repository, then build and test it
* Dont forget to turn on docker daemon with `$ sudo systemctl enable --now docker`

```bash
$ git clone https://github.com/fasyalks2025/nodejs-ssl-server-demo
$ cd nodejs-ssl-server-demo/
$ docker build -t nodejs-ssl-server-demo .
$ docker -d -p 3000:3000 <container id>

```
then copy the public ip and add :3000 at the end e.g 74.125.68.139:3000  
the website should show, dont forget to stop the docker container

---


## 3. IAM Setup

* Create a new IAM user: `nodejs-server-user`
* Attach the policy: `AmazonEC2ContainerRegistryFullAccess`
* Generate **Access Key ID** and **Secret Access Key**
* Configure AWS CLI with these credentials:

```bash
$ aws configure
```

---

## 4. ECR (Elastic Container Registry)

* Create a **private ECR repository** to store Docker images.
* Click on **"View push commands"** and follow all of the steps:
* There should be 4 steps like below example:
1. **Authenticate Docker to your registry**
   Example:

   ```bash
   aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
   ```

2. **Build the Docker image**

   ```bash
   docker build -t nodejs-server .
   ```

3. **Tag the Docker image**

   ```bash
   docker tag nodejs-server:latest <repo-uri>:latest
   ```

4. **Push the image to ECR**

   ```bash
   docker push <repo-uri>:latest
   ```

---

## 5. ECS Cluster and Task Definition

* Create a new **ECS Cluster**
* Use **Fargate** launch type for serverless infra
* Create a **Task Definition (TD)**:

  * Use **0.5 vCPU** and **1 GB memory**
  * Set container image to your **ECR repo**
  * Set **container port** to `3000` (this is the exposed port)

```plaintext
Port Mappings:
Container port: 3000
Host port: (optional)
App protocol: HTTP
```

---

## 6. ECS Service Configuration

* Create a **Service** from the Task Definition
* Launch Type: `Fargate`
* Deployment Type: `Replica`
* Desired Tasks: 2

### Networking

* Select a VPC with **at least 2 public subnets in different AZs**
* Create a **Security Group** named `nodejs-ALB-SG`
  * Allow **inbound on for http and https from anywhere**

* Create a **Security Group** named `nodejs-ALB-to-ECS-SG`
  * Allow **inbound only on port 3000**
  * Only allow inbound **from the ALB Security Group**


---

## 7. Application Load Balancer (ALB)

* **Load Balancer Type**: Application Load Balancer
* Choose the same VPC
* Create a new **Security Group** for the ALB allowing:

  * HTTP (port 80)
  * HTTPS (port 443)

### Listener

* Listener Protocol: `HTTP`
* Listener Port: `80`

### Target Group

* Create a new Target Group:

  * Name: `nodejs-server-ALB-TG-1`
  * Protocol: `HTTP`
  * Target Type: `IP`
  * Health Check Path: `/health`
  * Deregistration delay: `300` seconds
  * Health Check Protocol: `HTTP`

---

## 8. Final
Ideally we could just choose SG on ECS Service Console when creating LB,  
but since AWS is...stupid, 
we will be assigned SG from the container
so we have to change the SG manually in:

EC2 -> Load Balancer -> nodejs-server-ALB-TG-1 -> Security Group
change the security group to `nodejs-ALB-SG`
