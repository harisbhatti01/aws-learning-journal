# AWS Highly Available Web Application

A hands-on AWS Phase 3 project that demonstrates how to build a small highly available web application using Amazon EC2, an Application Load Balancer, a Target Group, an Auto Scaling Group, a Launch Template, User Data, and Security Groups.

## Project objective

Build a web application that can:

- serve HTTP traffic through an Application Load Balancer
- distribute requests across multiple EC2 instances
- detect unhealthy targets through load balancer health checks
- automatically replace failed EC2 instances through an Auto Scaling Group
- keep the architecture simple and cost-conscious

## Architecture

```text
                         Internet
                            |
                         HTTP :80
                            |
                            v
                 +-----------------------+
                 | Application Load      |
                 | Balancer (ALB)        |
                 | phase3-alb             |
                 +-----------+-----------+
                             |
                       HTTP :80
                             |
                             v
                 +-----------------------+
                 | Target Group           |
                 | phase3-web-tg          |
                 | Health check: /        |
                 +-----------+-----------+
                             |
                    +--------+--------+
                    |                 |
                    v                 v
              +-----------+     +-----------+
              | EC2 node  |     | EC2 node  |
              | Nginx     |     | Nginx     |
              +-----------+     +-----------+
                    ^                 ^
                    |                 |
                    +--------+--------+
                             |
                             v
                 +-----------------------+
                 | Auto Scaling Group    |
                 | phase3-web-asg        |
                 | desired=2             |
                 | min=1, max=2         |
                 +-----------------------+
                             |
                             v
                 +-----------------------+
                 | Launch Template       |
                 | phase3-web-lt         |
                 +-----------------------+
```

See the editable diagram in [`architecture/architecture.drawio`](architecture/architecture.drawio).

## AWS services used

| Service | Purpose |
|---|---|
| Amazon EC2 | Compute instances running Amazon Linux 2023 and Nginx |
| Amazon EBS | Root storage for the EC2 instances |
| Amazon VPC | Networking, subnets, routing, and Availability Zones |
| Security Groups | Firewall rules for ALB and EC2 traffic |
| Application Load Balancer | Public HTTP entry point and request distribution |
| Target Group | Pool of EC2 targets and HTTP health checks |
| Auto Scaling Group | Maintains desired EC2 capacity and replaces failed instances |
| EC2 Launch Template | Reusable instance configuration for the ASG |
| EC2 User Data | Boot-time automation for web-node configuration |
| AMI | Reusable image used as a known-good server baseline |

## Build summary

### 1. EC2 foundation

- Launched Amazon Linux 2023 on `t3.micro`.
- Used an 8 GiB gp3 root volume.
- Created `phase3-ec2-key` for SSH access.
- Restricted SSH to the administrator's current public IP.
- Verified Linux access from Windows PowerShell using OpenSSH.

### 2. Nginx web server

- Installed Nginx manually on the initial web server.
- Opened TCP 80 in the Security Group.
- Replaced the default Nginx page with a custom page.
- Verified the page through the EC2 public address.

### 3. User Data automation

A temporary EC2 instance was launched with User Data that installed and started Nginx and generated a custom webpage automatically. The instance was then cleaned up.

This demonstrated the difference between manual provisioning and boot-time automation.

### 4. Custom AMI

Created a custom AMI containing the configured Linux/Nginx web-server state and launched a temporary instance from it to verify that the configuration was reusable.

### 5. Application Load Balancer

Created an internet-facing ALB and a Target Group using HTTP port 80 health checks.

The ALB was configured across multiple Availability Zones. An early lab test produced a `503 Service Temporarily Unavailable` because the target instances were in an Availability Zone not initially enabled on the ALB. Enabling that Availability Zone allowed the targets to become usable.

### 6. Load balancing test

Two web servers were given unique response markers. Repeated requests through the ALB showed that traffic could be served by either backend instance.

### 7. Health-check failure test

Nginx was intentionally stopped on one target. The Target Group marked it unhealthy, and the ALB stopped routing normal traffic to it. Nginx was then restarted and the target returned to a healthy state.

### 8. Auto Scaling Group

Created:

- Launch Template: `phase3-web-lt`
- Auto Scaling Group: `phase3-web-asg`
- Desired capacity: 2
- Minimum capacity: 1
- Maximum capacity: 2

The ASG was connected to the Target Group so newly launched instances could be evaluated by the ALB health checks.

### 9. Automatic replacement test

One ASG-managed EC2 instance was manually terminated. The Auto Scaling Group detected that actual capacity had fallen below desired capacity and automatically launched a replacement. The replacement became healthy and returned the ASG to the desired capacity of two instances.

## Security Group design

The intended final traffic flow was:

```text
Internet
   |
   | HTTP :80
   v
ALB Security Group
   |
   | HTTP :80 from ALB SG
   v
EC2 Security Group
   |
   v
Nginx
```

SSH access remained restricted to the administrator's current public IP for temporary administration.

## Why ALB + ASG?

### Application Load Balancer

The ALB provides a stable public entry point, distributes HTTP requests across healthy targets, and performs health checks.

### Auto Scaling Group

The ASG maintains the requested number of EC2 instances and can replace failed instances automatically.

### Together

```text
ALB = traffic distribution + health-aware routing
ASG = capacity management + instance replacement
Launch Template = instance creation recipe
Target Group = backend target pool + health checks
```

## Cost-control strategy

This project was intentionally kept small:

- used `t3.micro` instances
- used an 8 GiB gp3 root volume
- kept ASG maximum capacity at 2
- did not create a NAT Gateway
- did not create RDS, EFS, Route 53 hosted zones, CloudFront, WAF, or Global Accelerator
- kept the ALB short-lived
- terminated temporary test instances
- deleted the ALB, Target Group, ASG, and manually created lab instances after validation
- released the test Elastic IP when it was no longer needed

## Cleanup

After validation, the following paid or runtime resources were removed:

- Application Load Balancer
- Target Group
- Auto Scaling Group
- ASG-managed EC2 instances
- Earlier standalone EC2 test instances

The Launch Template was also deleted accidentally during cleanup. This does not affect the completed lab because all experiments and screenshots had already been captured.

The AMI may also be retained or removed separately depending on whether it is still useful for later study; its underlying snapshot can consume storage when retained.

## Evidence

Place the captured screenshots in `screenshots/` and use these recommended names:

- `alb-working.png`
- `target-health.png`
- `asg-running.png`
- `instance-replacement.png`
- `cost-check.png`

Do not publish private keys, credentials, account IDs, or other sensitive information.

## Lessons learned

- An EC2 instance's public IP and private IP serve different purposes.
- A Security Group controls whether network traffic can reach an instance; it does not start or stop the application.
- Port 22 is used for SSH administration, while port 80 is used for HTTP.
- Nginx can be installed manually or automated through User Data.
- An AMI is a reusable server image.
- A Target Group is the ALB's backend target pool and health-check mechanism.
- An ALB provides a public entry point and routes to healthy targets.
- An Auto Scaling Group maintains desired capacity and replaces failed instances.
- ALB and ASG together provide the foundation for a highly available web application.
- Resource cleanup is part of cloud engineering because running managed resources can continue to generate cost.

## Future improvements

A production-grade version could add:

- HTTPS with AWS Certificate Manager
- private EC2 subnets with managed egress
- stricter least-privilege IAM roles
- observability and CloudWatch alarms
- infrastructure as code
- a CI/CD pipeline
- managed domain and DNS

Those improvements were intentionally excluded from this Phase 3 learning project.
