# Multi-AZ Secure AWS VPC

## Overview

This project implements a secure, multi-AZ AWS application network using a custom VPC, public and private subnets, controlled Internet access, layered security controls, private application instances, an Application Load Balancer, and Auto Scaling.

The project was built as an end-to-end AWS networking and application-infrastructure implementation. The objective was to design the network, implement the routing and security model, deploy the application tier in private subnets, expose it through a public load balancer, and validate failure recovery and traffic flow.

## Architecture

The environment is based on a single custom VPC using `10.0.0.0/16`.

```text
                                      INTERNET
                                          |
                                          v
                                +-------------------+
                                | Internet Gateway  |
                                |    phase4-igw     |
                                +---------+---------+
                                          |
                         +----------------+----------------+
                         |                                 |
                  Availability Zone A              Availability Zone B
                         |                                 |
                 +-------+-------+                 +-------+-------+
                 |               |                 |               |
          Public-A         Private-A          Public-B         Private-B
       10.0.1.0/24       10.0.2.0/24         10.0.3.0/24      10.0.4.0/24
                 |               |                 |               |
                 +-------+-------+                 +-------+-------+
                         |                                 |
                         +------------+   +----------------+
                                      |   |
                              +-------v---v-------+
                              | Application       |
                              | Load Balancer     |
                              | phase4 ALB        |
                              +---------+---------+
                                        |
                              +---------+---------+
                              |                   |
                       Private-A EC2       Private-B EC2
                       ASG-managed         ASG-managed
                              |                   |
                              +---------+---------+
                                        |
                                  NAT Gateway
                                   phase4-nat
                                        |
                                Internet Gateway
                                        |
                                     Internet
```

The editable architecture diagram is in `architecture/vpc-architecture.drawio`, with a PNG export in `architecture/vpc-architecture.png`.

## Network Design

| Component | Configuration |
|---|---|
| VPC | `phase4-vpc` — `10.0.0.0/16` |
| Public Subnet A | `10.0.1.0/24` |
| Private Subnet A | `10.0.2.0/24` |
| Public Subnet B | `10.0.3.0/24` |
| Private Subnet B | `10.0.4.0/24` |
| Public Route Table | `phase4-public-rt` |
| Private Route Table | `phase4-private-rt` |
| Internet Gateway | `phase4-igw` |
| NAT Gateway | `phase4-nat` |
| NAT Elastic IP | `phase4-elasticIP` |

Public subnets use a default route to the Internet Gateway. Private subnets use the VPC local route for internal communication and the NAT Gateway for controlled outbound Internet access.

## Application Tier

The application tier was moved into the private subnets and exposed through an Internet-facing Application Load Balancer.

### Load Balancer

- Application Load Balancer
- Internet-facing
- Deployed across the two public subnets
- HTTP listener on port 80
- Target group: `phase4-app-tg`
- ALB security group: `phase4-alb-sg`

### Application Security Group

`phase4-app-sg` controls the private EC2 instances.

The intended application rule is:

```text
HTTP / TCP 80
Source: phase4-alb-sg
```

This keeps the application tier inaccessible directly from the Internet while allowing the load balancer to reach the application instances.

### Launch Template

`phase4-app-lt` provides repeatable EC2 configuration for the Auto Scaling Group.

The launch template uses Amazon Linux and bootstraps Nginx through User Data. The application page identifies the Phase 4 application and the private EC2 instance serving the request.

### Auto Scaling Group

`phase4-app-asg` maintains application capacity across the two private subnets.

The final implementation used a small, controlled capacity so that the project could demonstrate multi-AZ placement and automatic replacement without creating unnecessary infrastructure.

### Target Group

`phase4-app-tg` performs HTTP health checks on `/` using port 80. Only healthy targets receive traffic from the Application Load Balancer.

## Security Architecture

The project uses multiple layers of network control:

1. Internet-facing access terminates at the ALB rather than directly at the EC2 instances.
2. The ALB security group permits public HTTP access.
3. The application security group permits HTTP only from the ALB security group.
4. The application instances are deployed in private subnets.
5. Network ACLs provide subnet-level filtering.
6. Private outbound Internet access is provided through the NAT Gateway rather than a direct Internet Gateway route on the private subnets.
7. VPC endpoint connectivity was configured to demonstrate private access to AWS services.

## Traffic Flows

### Internet to application

```text
Client
  -> Internet Gateway
  -> Public ALB
  -> ALB Security Group
  -> VPC local routing
  -> Application Security Group
  -> Private EC2
```

### Private EC2 to Internet

```text
Private EC2
  -> Private Route Table
  -> NAT Gateway
  -> Internet Gateway
  -> Internet
```

### Private EC2 to another VPC resource

Traffic for destinations inside the VPC uses the VPC's local route. VPC peering was also tested as a separate private connectivity mechanism.

### AWS service access

A VPC endpoint was configured to demonstrate private AWS service access without requiring the application to use a public Internet path for that service.

## Validation Performed

The implementation was validated through the following operational tests:

- Confirmed the VPC and four subnet CIDR design.
- Verified public and private route-table associations.
- Verified Internet Gateway routing for public subnets.
- Verified NAT Gateway routing for private-subnet outbound access.
- Verified ALB listener and target-group health checks.
- Confirmed private EC2 instances were reachable from the ALB through private VPC routing.
- Confirmed application instances were not exposed directly as Internet-facing servers.
- Confirmed Auto Scaling successfully maintained application capacity.
- Terminated an ASG-managed instance and verified replacement behavior.
- Verified unhealthy targets were removed from ALB traffic using target health checks.
- Tested network security controls and subnet-level filtering.
- Tested VPC endpoint connectivity and VPC peering during the networking phase.

## Operational Lessons

The project reinforced several production-oriented networking principles:

- A subnet is not public simply because it is named public; its route table must provide Internet routing through an Internet Gateway.
- A private subnet can communicate with the Internet for outbound connections through a NAT Gateway without exposing the workload directly to inbound Internet traffic.
- ALB-to-EC2 traffic can remain completely private inside the VPC through local routing.
- Security groups should describe trust relationships between application tiers rather than exposing application servers broadly.
- Availability across multiple Availability Zones must be reflected in subnet placement and compute capacity.
- Load-balancer health and EC2 infrastructure health are different checks and must be diagnosed separately.
- Auto Scaling should be validated through actual replacement behavior rather than only checking that an ASG exists.

## Project Evidence

The `screenshots/` directory is intended to contain the evidence captured during implementation. Add the real screenshots from the AWS console after cloning or copying this repository.

Recommended evidence files:

```text
screenshots/
├── 01-vpc-details.png
├── 02-subnets.png
├── 03-route-tables.png
├── 04-internet-gateway.png
├── 05-nat-gateway.png
├── 06-security-groups.png
├── 07-network-acl.png
├── 08-vpc-endpoint.png
├── 09-alb.png
├── 10-target-group-healthy.png
├── 11-auto-scaling-group.png
├── 12-private-instances.png
├── 13-alb-application-working.png
├── 14-private-outbound-connectivity.png
├── 15-instance-replacement.png
└── 16-final-cleanup.png
```

## Repository Structure

```text
aws-multi-az-vpc/
├── README.md
├── architecture/
│   ├── vpc-architecture.drawio
│   └── vpc-architecture.png
├── screenshots/
└── notes/
    ├── architecture.md
    ├── deployment.md
    ├── security.md
    ├── traffic-flows.md
    ├── testing.md
    └── cost-notes.md
```

## Outcome

The completed environment demonstrates the ability to design, implement, secure, validate, and document a multi-AZ AWS network rather than only configure individual AWS networking services in isolation.
