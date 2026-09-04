# Architecture Notes

## Design Goal

Provide a resilient application network with public entry points, private application servers, controlled outbound Internet access, and layered network security.

## Addressing

The VPC uses `10.0.0.0/16` and is divided into four `/24` subnets across two Availability Zones:

| Availability Zone | Public | Private |
|---|---|---|
| AZ-A | `10.0.1.0/24` | `10.0.2.0/24` |
| AZ-B | `10.0.3.0/24` | `10.0.4.0/24` |

## Routing Model

The public route table contains:

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> phase4-igw
```

The private route table contains:

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> phase4-nat
```

The local route supports communication between VPC resources without requiring separate subnet-to-subnet routes.

## Application Placement

The ALB is placed in the public subnets. The EC2 application instances are placed in the private subnets through the Auto Scaling Group. This separates Internet-facing ingress from the application tier.

## Availability

Two Availability Zones are used so that application capacity is not dependent on a single zone. The Auto Scaling Group maintains multiple application instances and the ALB routes traffic only to healthy targets.

## Network Components

- Custom VPC
- Four subnets across two Availability Zones
- Public and private route tables
- Internet Gateway
- NAT Gateway
- Network ACLs
- Security Groups
- VPC endpoint connectivity
- VPC peering
- Application Load Balancer
- Target Group
- Launch Template
- Auto Scaling Group
