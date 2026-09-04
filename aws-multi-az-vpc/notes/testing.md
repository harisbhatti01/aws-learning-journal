# Testing and Validation

## Test 1 — VPC Addressing

Verified the `10.0.0.0/16` VPC contains four non-overlapping `/24` subnets and that the public/private subnet pairs span two Availability Zones.

## Test 2 — Public Routing

Verified the public route table contains:

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> Internet Gateway
```

## Test 3 — Private Routing

Verified the private route table contains:

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> NAT Gateway
```

## Test 4 — ALB Health

Verified the target group health check uses:

```text
Protocol: HTTP
Path: /
Port: traffic port
Success code: 200
```

The two application targets were observed in a healthy state.

## Test 5 — Application Access

Opened the ALB DNS endpoint and verified that the Nginx application page was returned from the private application tier.

## Test 6 — Private Application Placement

Verified that Auto Scaling instances were launched into the private subnets rather than the public subnets.

## Test 7 — Outbound Connectivity

Verified private-subnet outbound connectivity through the NAT Gateway.

## Test 8 — Instance Failure and Replacement

Terminated an ASG-managed application instance and verified that the Auto Scaling Group launched a replacement instance.

## Test 9 — Load Balancer Health Protection

Verified that unhealthy targets were excluded from ALB traffic and that healthy replacement instances were admitted after successful health checks.

## Test 10 — Security Controls

Verified that application HTTP access is restricted to the ALB security group rather than allowing Internet-wide access directly to the EC2 application tier.

## Diagnostic Lesson

EC2 infrastructure status checks and ALB target health checks are separate mechanisms. An instance can pass EC2 system/instance checks while still failing the ALB application health check if the application or network path is not functioning correctly.
