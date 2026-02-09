# AWS Application Load Balancer with Auto Scaling and Docker

A step-by-step guide to deploying a Dockerized Node.js application on AWS EC2 with automatic scaling and load balancing.

## Architecture Overview

This project deploys a containerized backend application that runs a Node.js API on port 3000, uses MongoDB for data storage, automatically scales based on CPU usage at 50 percent threshold, distributes traffic using an Application Load Balancer, and maintains 1 to 2 EC2 instances based on demand.

Tech Stack includes AWS EC2, ALB, Auto Scaling Group, Docker, Node.js, MongoDB, and Amazon Linux 2.

## Step 1 Prepare Your Application

### Application Structure

The backend is a Node.js application listening on port 3000. The database is MongoDB running on port 27017.

API Endpoints include GET slash for health check which returns App running plus DB connected, and GET slash users to fetch user data from MongoDB.

### Docker Configuration

Two containers run on each EC2 instance. First is the backend container on port 3000. Second is the MongoDB container on port 27017.

## Step 2 Create Security Groups

### ALB Security Group

Name it alb security group with the following inbound rules. Type is HTTP, port is 80, and source is 0.0.0.0/0.

### EC2 Security Group

Name it ec2 security group with the following inbound rules. First rule has type Custom TCP, port 3000, and source as the ALB security group. Second rule has type SSH, port 22, and source as your IP for administration.

## Step 3 Create Target Group

Navigate to EC2, then Target Groups, then Create target group.

Configure the target type as Instance, name as backend target group, protocol as HTTP, port as 3000, and select your VPC.

For health check settings, set path as slash, success codes as 200, interval as 30 seconds, timeout as 5 seconds, healthy threshold as 2, and unhealthy threshold as 2.

Click Create.

## Step 4 Create Application Load Balancer

Navigate to EC2, then Load Balancers, then Create Load Balancer.

Select Application Load Balancer.

Configure name as backend alb, scheme as Internet-facing, IP address type as IPv4. Select your VPC and select at least 2 public subnets in different availability zones. Select alb security group as the security group.

For the listener, set protocol as HTTP, port as 80, and default action as Forward to backend target group.

Click Create.

## Step 5 Create Launch Template

Navigate to EC2, then Launch Templates, then Create launch template.

Configure name as backend launch template, AMI as Amazon Linux 2, instance type as t2.micro, and security group as ec2 security group.

Under Advanced Details and User data, add the following script.

The script starts with hashbang bin bash. It updates the system with yum update dash y. It installs Docker with yum install docker dash y, starts the Docker service, and adds ec2-user to the docker group. It installs Docker Compose by downloading it from GitHub and making it executable. It creates an application directory at home ec2-user app and changes to that directory. You can add your docker-compose.yml or docker run commands here.

Click Create launch template.

## Step 6 Create Auto Scaling Group

Navigate to EC2, then Auto Scaling Groups, then Create Auto Scaling group.

Configure name as backend asg, launch template as backend launch template. Select your VPC and select the same subnets as the ALB.

For load balancing, attach to existing load balancer and select backend target group.

For group size, set desired capacity as 1, minimum capacity as 1, and maximum capacity as 2.

For scaling policies, set policy type as Target tracking scaling policy, metric as Average CPU utilization, and target value as 50.

Click Create Auto Scaling group.

## Step 7 Testing

### Verify Docker Containers on EC2

SSH into your EC2 instance and run docker ps to check running containers and curl http localhost 3000 to test the backend locally.

Expected output is App running plus DB connected.

### Test via Load Balancer

Get your ALB DNS name from EC2 then Load Balancers. Access in browser or via curl http ALB DNS NAME.

### Monitor Auto Scaling

Check EC2 then Auto Scaling Groups then Activity for scaling events. View Target Groups then Targets to see healthy instances. Monitor CloudWatch metrics for CPU utilization.

## Troubleshooting

### Issue 504 Gateway Timeout

Possible causes include target instances are unhealthy, incorrect health check path, port mismatch between target group and application, or security group blocking ALB traffic.

Solutions are to check target health in EC2 then Target Groups then Targets tab. Verify health check path is slash and returns 200. Confirm target group port is 3000. Ensure EC2 security group allows inbound on port 3000 from ALB security group.

### Issue Docker Port Already in Use

Error message is Bind for 0.0.0.0 3000 failed port is already allocated.

Solution is to list all containers with docker ps dash a, stop conflicting container with docker stop container id, and remove container with docker rm container id. Or kill the process using the port with sudo lsof dash i 3000 and sudo kill dash 9 PID.

### Issue Targets Not Registering

Solutions include verify EC2 instances are in the same VPC as target group, check that instances are in subnets associated with the ASG, ensure user data script completes successfully by checking var log cloud-init-output.log, and verify Docker containers start automatically.





