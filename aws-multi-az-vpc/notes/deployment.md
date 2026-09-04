# Deployment Notes

## Deployment Sequence

The environment was assembled incrementally so each networking dependency could be verified before the application layer was introduced.

### Network foundation

1. Created the custom VPC `phase4-vpc` with CIDR `10.0.0.0/16`.
2. Created two public subnets and two private subnets across two Availability Zones.
3. Created separate public and private route tables and associated the correct subnets.
4. Attached `phase4-igw` to the VPC.
5. Added the public default route `0.0.0.0/0` to the Internet Gateway.
6. Created the public NAT Gateway `phase4-nat` with the allocated Elastic IP.
7. Added the private default route `0.0.0.0/0` to the NAT Gateway.

### Security layer

8. Configured network ACL rules for the subnet boundaries.
9. Created `phase4-alb-sg` for Internet-facing load-balancer access.
10. Created `phase4-app-sg` for private application instances.
11. Restricted application HTTP access to traffic originating from the ALB security group.

### Application layer

12. Created the Application Load Balancer across the two public subnets.
13. Created `phase4-app-tg` with an HTTP health check on `/`.
14. Created the `phase4-app-lt` Launch Template with Amazon Linux and Nginx bootstrap User Data.
15. Created `phase4-app-asg` using the private subnets across both Availability Zones.
16. Attached the Auto Scaling Group to the existing target group.
17. Verified that the ASG-created application targets became healthy.

### Validation

18. Verified the application through the ALB DNS endpoint.
19. Verified private application placement.
20. Verified outbound Internet access for private resources through NAT.
21. Tested failure and replacement behavior by terminating an ASG-managed instance.
22. Verified that the ALB continued routing to healthy targets during replacement.
