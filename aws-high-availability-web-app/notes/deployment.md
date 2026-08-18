# Deployment Notes

## Region and baseline

- AWS Region: `us-east-1` (N. Virginia)
- Primary OS: Amazon Linux 2023, x86_64
- Primary learning instance type: `t3.micro`
- Root volume: 8 GiB gp3
- SSH key pair: `phase3-ec2-key`

## Lab 3.1 — EC2 launch

1. Selected Amazon Linux 2023.
2. Selected `t3.micro`.
3. Created `phase3-ec2-key`.
4. Used the default VPC.
5. Enabled public IPv4 addressing.
6. Allowed SSH only from the current administrator IP.
7. Used an 8 GiB gp3 root volume.
8. Used IMDSv2 (`V2 only`).
9. Kept detailed monitoring disabled and CPU credit specification at Standard.

## Lab 3.2 — SSH and Linux orientation

Connected from Windows PowerShell with OpenSSH:

```powershell
ssh -i "C:\path\to\phase3-ec2-key.pem" ec2-user@<PUBLIC-IP>
```

Basic Linux checks included:

```bash
whoami
pwd
ls
cat /etc/os-release
hostname -I
free -h
nproc
id
sudo whoami
```

## Lab 3.3 — Nginx

1. Allowed HTTP port 80 in the EC2 Security Group.
2. Installed Nginx with DNF.
3. Enabled and started Nginx with systemd.
4. Replaced the default web page with a custom HTML page.
5. Verified the application in a normal web browser.

## Lab 3.4 — Security Group experiment

1. Confirmed Nginx was running.
2. Removed HTTP 80 from the Security Group.
3. Confirmed the webpage stopped being reachable.
4. Restored HTTP 80.
5. Confirmed the webpage worked again.

Conclusion: the application can be running while the network firewall still blocks access.

## Lab 3.5 — User Data

Created a temporary EC2 with User Data that:

- installed Nginx
- started Nginx
- created a custom page

The page was reachable without manually SSHing into the instance. The temporary instance was then terminated.

## Lab 3.6 — Custom AMI

Created a reusable AMI from the manually configured web server. The AMI was tested by launching a temporary instance and verifying that the Nginx configuration and webpage were present without manual installation.

## Lab 3.7 — Elastic IP

Allocated and associated an Elastic IP with the original test instance to understand stable public addressing. The test resource was cleaned up afterward.

## Lab 3.8 — Application Load Balancer

### Target Group

- Name: `phase3-web-tg`
- Target type: Instances
- Protocol: HTTP
- Port: 80
- Protocol version: HTTP1
- Health check protocol: HTTP
- Health check path: `/`
- Success code: `200`

### ALB

- Name: `phase3-alb`
- Scheme: Internet-facing
- IP address type: IPv4
- Listener: HTTP :80
- Default action: forward to `phase3-web-tg`
- Multiple Availability Zones enabled
- ALB Security Group allowed TCP 80 from the Internet

### Validation

- Verified healthy targets.
- Gave backend servers unique response markers.
- Repeated ALB requests to observe distribution.
- Stopped Nginx on one target and observed an unhealthy state.
- Restored Nginx and observed recovery.

### Troubleshooting event

The first ALB test returned HTTP 503 because the targets were in `us-east-1c` while the ALB initially had only `us-east-1a` and `us-east-1b` enabled. Enabling the target Availability Zone on the ALB resolved the issue.

## Lab 3.9 — Launch Template + Auto Scaling Group

### Launch Template

- Name: `phase3-web-lt`
- Instance type: `t3.micro`
- Key pair: `phase3-ec2-key`
- Root volume: 8 GiB gp3
- AMI: tested Phase 3 web-server AMI
- User Data used to generate a hostname-based page
- EC2 Security Group allowed SSH from My IP and HTTP from the ALB Security Group

### Auto Scaling Group

- Name: `phase3-web-asg`
- Desired capacity: 2
- Minimum capacity: 1
- Maximum capacity: 2
- Connected to `phase3-web-tg`
- Enabled EC2 and ELB health checks
- Used multiple Availability Zones
- No dynamic target-tracking policy for this learning lab

### Failure and replacement test

1. Waited for two healthy ASG-managed instances.
2. Terminated one ASG-managed instance manually.
3. Observed the ASG detect capacity below desired state.
4. ASG launched a replacement instance from the Launch Template.
5. Replacement became healthy and returned the ASG to desired capacity 2.

## Cleanup

After all screenshots and validation were complete:

1. Deleted the ALB.
2. Deleted the Auto Scaling Group and its managed instances.
3. Deleted the Target Group.
4. Terminated the earlier standalone lab EC2 instances.
5. Checked that no Phase 3 ALB or EC2 runtime remained.
6. The Launch Template was deleted accidentally during cleanup; this was done after the lab and did not affect the captured evidence.
