# Highly Available Web Application on AWS

This project is a small web application deployment built on AWS to understand how multiple EC2 instances can be placed behind an Application Load Balancer and maintained by an Auto Scaling Group.

The work started with a single Linux web server and gradually moved toward a setup that can handle an instance failure without manual replacement.

## Overview

The application uses Amazon Linux 2023 and Nginx on EC2. An Application Load Balancer provides the public entry point, a Target Group performs HTTP health checks, and an Auto Scaling Group maintains two web servers.

```text
Internet
    |
    | HTTP :80
    v
Application Load Balancer
    |
    | HTTP :80
    v
Target Group
    |
    +-------------------+
    |                   |
    v                   v
EC2 web server      EC2 web server
    |                   |
    +---------+---------+
              |
       Auto Scaling Group
       desired: 2
       min: 1
       max: 2
```

The editable architecture diagram is available in [`architecture/architecture.drawio`](architecture/architecture.drawio), with a PNG version in [`architecture/architecture.png`](architecture/architecture.png).

## What I built

The deployment was completed in stages:

1. Launched Amazon Linux 2023 on a small EC2 instance.
2. Connected to the instance from Windows PowerShell using SSH.
3. Installed Nginx and served a custom web page.
4. Tested Security Group behavior by opening and closing HTTP access.
5. Used EC2 User Data to automate Nginx installation on a temporary server.
6. Created a reusable AMI from a configured web server and verified that it could be used to launch another working server.
7. Allocated and associated an Elastic IP to understand stable public addressing.
8. Created an Application Load Balancer and Target Group and placed two web servers behind them.
9. Added HTTP health checks and tested what happened when Nginx stopped on one target.
10. Created a Launch Template and Auto Scaling Group with a desired capacity of two instances.
11. Connected the Auto Scaling Group to the Target Group.
12. Terminated one ASG-managed instance and verified that Auto Scaling launched a replacement automatically.
13. Removed the temporary and paid resources after validation.

## AWS services and their roles

| Service | Role in the project |
|---|---|
| Amazon EC2 | Runs the Linux web servers and Nginx |
| Amazon EBS | Provides the root disk for EC2 |
| Amazon VPC | Provides the network, subnets and Availability Zones |
| Security Groups | Control HTTP and SSH access |
| Application Load Balancer | Provides the public HTTP entry point and distributes requests |
| Target Group | Keeps the backend instance list and performs HTTP health checks |
| Auto Scaling Group | Maintains the required number of web servers and replaces failed instances |
| Launch Template | Stores the EC2 configuration used by the Auto Scaling Group |
| Amazon Machine Image | Provides a reusable server image |
| EC2 User Data | Automates first-boot configuration |

## Network and security design

The public-facing component is the load balancer. The web servers are intended to receive HTTP traffic from the load balancer rather than directly from the Internet.

```text
Internet
   |
   | HTTP :80
   v
ALB Security Group
   |
   | HTTP :80
   v
EC2 Security Group
   |
   v
Nginx
```

SSH access was kept restricted to the administrator's current public IP for temporary administration.

The two main Security Group rules for a web server were:

- TCP 22 from the administrator's current IP address.
- TCP 80 from the load balancer Security Group.

The ALB Security Group allowed TCP 80 from the Internet.

## Why these components were used

### Application Load Balancer

The ALB gives the application one public endpoint. It forwards requests to healthy EC2 targets instead of requiring users to connect to individual servers.

### Target Group

The Target Group contains the backend instances and checks the `/` path over HTTP on port 80. An instance that fails the health check is removed from normal traffic until it becomes healthy again.

### Auto Scaling Group

The Auto Scaling Group keeps the application at the desired capacity. The group was configured with:

```text
Desired: 2
Minimum: 1
Maximum: 2
```

When one managed instance was terminated, the group detected the lower capacity and launched a replacement from the Launch Template.

### Launch Template

The Launch Template stores the configuration needed to create another web server, including the AMI, instance type, storage, key pair, Security Group and boot-time User Data.

### User Data

User Data was used to automate first-boot configuration. In the automated web-server test, it installed Nginx, enabled the service and created a simple web page without requiring a manual SSH configuration step.

## Load balancing and failure tests

Two backend web servers were given different response markers so that repeated requests through the ALB could be used to observe traffic reaching different targets.

A separate health-check test stopped Nginx on one target. The Target Group reported that instance as unhealthy and the ALB stopped sending normal traffic to it. Nginx was restarted and the target returned to a healthy state.

The Auto Scaling test went one step further. One ASG-managed instance was terminated manually. The Auto Scaling Group launched a replacement and the new instance became healthy again.

These tests demonstrated two related but different responsibilities:

```text
ALB + Target Group  -> routes traffic only to healthy targets
ASG                 -> maintains the required number of instances
```

## Cost control

The environment was deliberately kept small. The EC2 instances used the `t3.micro` type with 8 GiB gp3 root volumes. The Auto Scaling Group was limited to a maximum of two instances. No NAT Gateway was created, and services such as RDS, EFS, Route 53 hosted zones, CloudFront, WAF and Global Accelerator were not needed for this project.

The load balancer and temporary test instances were removed after the tests were completed. The earlier standalone EC2 instances were also terminated during cleanup.

## Evidence

The project was supported by screenshots of the main stages, including:

- successful SSH access
- running Nginx
- the custom web page
- the load balancer serving the application
- healthy Target Group members
- the Auto Scaling Group and its instances
- automatic instance replacement
- the final cost/resource check

Sensitive information such as private keys, credentials and other account secrets should not be included in the repository.

## Lessons learned

- A public IP and a private IP serve different purposes in an AWS VPC.
- Port 22 is used for SSH administration, while port 80 is used for HTTP.
- A Security Group can block access even when the application itself is running.
- User Data is useful for first-boot configuration and repeatable server setup.
- An AMI provides a reusable server baseline.
- A Target Group combines backend registration with health checking.
- An ALB provides a single public entry point and distributes traffic across healthy targets.
- An Auto Scaling Group maintains capacity and can replace failed instances automatically.
- Keeping the ALB and EC2 instances in more than one Availability Zone improves resilience to an Availability Zone problem.
- Cloud resources should be removed when testing is complete so that temporary infrastructure does not continue generating cost.

## Possible next improvements

A production version could add HTTPS with AWS Certificate Manager, private subnets for the application servers, least-privilege IAM roles, CloudWatch alarms, infrastructure as code and a CI/CD pipeline.
