# Cost Notes

## Goal

Keep Phase 3 practical work small, temporary, and easy to clean up.

## Cost-sensitive resources used

### EC2

Used small `t3.micro` instances for the learning labs. Running instances continue to consume compute resources, so idle time was minimized.

### EBS

Used 8 GiB gp3 root volumes. Stopped instances can still retain EBS storage, so storage was considered during cleanup.

### Application Load Balancer

The ALB was treated as a short-lived paid resource. It was created only after the EC2 and Target Group prerequisites were ready and was deleted immediately after validation.

### Public IPv4 / Elastic IP

An Elastic IP was used temporarily for the Elastic IP lesson. Unused public IPv4 addresses should not be left allocated unnecessarily.

## Resources deliberately NOT created

- NAT Gateway
- RDS
- EFS
- FSx
- Route 53 hosted zone
- CloudFront
- AWS WAF
- Global Accelerator
- Other managed services not required for the Phase 3 objective

This reduced both cost and architectural complexity.

## Capacity controls

The Auto Scaling Group was configured with:

```text
Desired = 2
Minimum = 1
Maximum = 2
```

The maximum was intentionally kept at 2 to avoid accidental scaling into a larger bill.

## Temporary test resources

Temporary EC2 instances were created for:

- User Data testing
- AMI validation
- Load balancing experiments

These were terminated after validation.

## Cleanup checklist

- [x] ALB deleted
- [x] Target Group deleted
- [x] Auto Scaling Group deleted
- [x] ASG-managed instances terminated by ASG deletion
- [x] Original manual EC2 lab instances terminated
- [x] Launch Template deleted during cleanup
- [x] No NAT Gateway created
- [x] No additional managed services created
- [ ] Final Billing page screenshot — add after Billing access is available

## Billing access note

Billing and Cost Management access may be unavailable to an IAM user even when the user has broad AWS service permissions. The account-level IAM access to billing setting may need to be enabled from the account/root side before a final cost screenshot can be captured.

## Final verification

The final billing check should confirm that the Phase 3 runtime resources are no longer active and that there are no unexpected resources continuing to generate charges.
