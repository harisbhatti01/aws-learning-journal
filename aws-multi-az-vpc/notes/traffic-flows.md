# Traffic Flows

## 1. Internet to Application

```text
Internet
   |
   v
Internet Gateway
   |
   v
Public Subnets
   |
   v
Application Load Balancer
   |
   | HTTP/80
   v
Application Security Group
   |
   v
Private EC2
```

The ALB terminates the public connection and forwards the request to a healthy private application instance.

## 2. Private Application to Internet

```text
Private EC2
   |
   v
Private Route Table
   |
   | 0.0.0.0/0
   v
NAT Gateway
   |
   v
Internet Gateway
   |
   v
Internet
```

The private instance initiates the outbound connection. The workload does not require a public IP address.

## 3. Private Application to VPC Resource

```text
Private EC2
   |
   v
Route Table
   |
   | 10.0.0.0/16
   v
local
   |
   v
VPC Resource
```

The VPC local route handles traffic whose destination is inside the VPC CIDR.

## 4. ALB Health Check

```text
ALB
  |
  | HTTP / on port 80
  v
Private EC2
  |
  v
Nginx
  |
  v
HTTP 200
```

Only healthy targets are eligible for normal ALB routing.

## 5. Auto Scaling Replacement

```text
ASG-managed EC2
       |
       | instance terminated
       v
Auto Scaling Group detects capacity loss
       |
       v
Launch Template
       |
       v
New private EC2 instance
       |
       v
Target Group health check
       |
       v
Healthy target -> ALB traffic restored
```
