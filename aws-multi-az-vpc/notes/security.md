# Security Notes

## Security Groups

### `phase4-alb-sg`

The load balancer is the public entry point. HTTP access is permitted from the Internet because the ALB is intentionally Internet-facing.

### `phase4-app-sg`

The application security group restricts HTTP access to the ALB security group rather than the public Internet.

This creates a trust boundary:

```text
Internet -> ALB -> ALB SG -> App SG -> Private EC2
```

The EC2 layer therefore does not need a public HTTP rule.

## Network ACLs

Network ACLs were used as subnet-level controls in addition to the stateful Security Group controls attached to the application instances.

The key architectural distinction is:

- Security Group: instance/ENI-level, stateful, allow rules.
- Network ACL: subnet-level, stateless, ordered allow/deny rules.

## Private Subnets

Application instances are deployed without direct Internet-facing placement. Their outbound Internet connectivity uses the NAT Gateway, while inbound application traffic is expected to originate from the ALB.

## Endpoint Connectivity

VPC endpoint connectivity was tested as a private AWS-service access pattern. This demonstrates that AWS services do not always require a public Internet path when an appropriate VPC endpoint is available.
