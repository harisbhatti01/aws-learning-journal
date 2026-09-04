# Cost and Cleanup Notes

## Cost-Aware Design

The project used a deliberately small infrastructure footprint.

The most important paid networking component was the NAT Gateway, so it was used only for the required private-subnet outbound connectivity validation. The ALB and EC2 resources were also kept to a small learning-oriented capacity.

## Resources Requiring Attention

- NAT Gateway
- Elastic IP associated with the NAT Gateway
- Application Load Balancer
- EC2 instances
- Auto Scaling Group capacity
- EBS volumes and snapshots created during testing
- Any temporary endpoint, peering, or networking resources created for demonstrations

## Cleanup Principle

Billable resources should not be left running after testing. The environment should be reviewed through the AWS console and billing views after teardown rather than relying on memory.

## Security and Cost Hygiene

The repository contains architecture and documentation only. AWS account credentials, private keys, access tokens, account secrets, and sensitive billing information must not be committed.
