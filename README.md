# Scalable Web Application on AWS with ALB and Auto Scaling

A production-grade, highly available web application architecture on AWS. EC2 instances run in private subnets across two Availability Zones behind an Application Load Balancer, with CloudFront for static asset caching and a Multi-AZ RDS database backend.

![Architecture diagram](docs/architecture.png)

Save the architecture diagram as `docs/architecture.png` so the image above renders.


---

## Table of contents

- [Overview](#overview)
- [Architecture](#architecture)
- [AWS services used](#aws-services-used)
- [Request flow](#request-flow)
- [Network design](#network-design)
- [Security](#security)
- [High availability and scalability](#high-availability-and-scalability)
- [Monitoring and alerting](#monitoring-and-alerting)
- [Deployment guide](#deployment-guide)
- [Validation checklist](#validation-checklist)
- [Cleanup](#cleanup)
- [Learning outcomes](#learning-outcomes)
- [Possible improvements](#possible-improvements)

---

## Overview

This project deploys a web application using EC2 instances inside a VPC with public and private subnets across two Availability Zones.

Design goals:

- *High availability:* every tier is spread across two AZs.
- *Scalability:* an Auto Scaling group adds and removes EC2 instances based on load.
- *Security:* all compute and database resources live in private subnets. There is no bastion host. Access is through AWS Systems Manager Session Manager.
- *Performance:* CloudFront caches static assets from an S3 origin to reduce latency.
- *Observability:* CloudWatch alarms send notifications through SNS.

---

## Architecture

| Layer | Components |
|---|---|
| Edge | Route 53, CloudFront, S3 origin for static assets, WAF |
| Entry | Internet Gateway, Application Load Balancer |
| Public subnets (x2) | NAT Gateway in each AZ, public route table |
| Private subnets (x2) | EC2 instances in an Auto Scaling group, RDS Multi-AZ, private route table |
| Access | Systems Manager Session Manager |
| Monitoring | CloudWatch, SNS |
| Network security | Security Groups, NACLs |

---

## AWS services used

| Service | Role in this project |
|---|---|
| *VPC* | Isolated network with public and private subnets, route tables, NAT Gateways, Security Groups and NACLs |
| *EC2 + Auto Scaling* | Application servers launched from a Launch Template, scaled with target tracking policies |
| *Application Load Balancer* | Layer 7 routing and target group health checks |
| *AWS WAF* | Protects the ALB with rules for the OWASP Top 10 |
| *CloudFront* | Caches static assets and reduces latency |
| *S3* | Origin for static assets served through CloudFront |
| *RDS (MySQL or PostgreSQL)* | Multi-AZ database with automated failover |
| *Route 53* | DNS with health checks |
| *Systems Manager* | Session Manager for secure, bastion-free instance access |
| *CloudWatch + SNS* | Dashboards, alarms and notifications |

---

## Request flow

1. A user requests the site. *Route 53* resolves the domain name.
2. The request goes to *CloudFront*.
   - Static assets are served from the cache, backed by the *S3 origin*.
   - Dynamic requests are forwarded to the *ALB*.
3. Traffic enters the VPC through the *Internet Gateway*.
4. *WAF* inspects the traffic before it reaches the *ALB*.
5. The ALB forwards requests to healthy *EC2 instances* in the private subnets.
6. EC2 instances read from and write to the *RDS primary* instance.
7. RDS replicates synchronously to the *standby* in the second AZ and fails over automatically if needed.
8. EC2 instances reach the internet for updates and patches through the *NAT Gateway* in their AZ.

---

## Network design

- *Two Availability Zones*, each with one public and one private subnet.
- *Public subnets* hold the NAT Gateways and the ALB nodes.
- *Private subnets* hold the EC2 instances and RDS.
- *Route tables:*

  | Route table | Destination | Target |
  |---|---|---|
  | Public | 0.0.0.0/0 | Internet Gateway |
  | Private (per AZ) | 0.0.0.0/0 | NAT Gateway in the same AZ |

- *NACLs* are attached to the subnets as an additional stateless layer of control.

Choose CIDR blocks that suit your environment, for example a /16 VPC split into /24 subnets.

---

## Security

### Security groups

| Security group | Inbound | Source |
|---|---|---|
| ALB-SG | HTTP/HTTPS (80/443) | Internet (or CloudFront) |
| EC2-SG | App port (for example 80/8080) | ALB-SG |
| RDS-SG | DB port (3306 MySQL or 5432 PostgreSQL) | EC2-SG |

### Other controls

- WAF with AWS managed rule sets for the OWASP Top 10.
- All EC2 and RDS resources are in private subnets with no public IPs.
- Instances are managed through *Session Manager*. There are no SSH keys, no open port 22 and no bastion host. Attach the AmazonSSMManagedInstanceCore policy to the instance role.
- NACLs add subnet-level filtering.

---

## High availability and scalability

- *Multi-AZ everywhere:* subnets, NAT Gateways, EC2 instances and RDS span two AZs.
- *Auto Scaling group* launches instances from a Launch Template and replaces unhealthy ones.
- *Target tracking policy* keeps a metric, such as average CPU utilization, near a target value.
- *ALB health checks* route traffic only to healthy targets.
- *RDS Multi-AZ* provides a synchronous standby and automated failover.
- *Route 53 health checks* support DNS-level failover decisions.

---

## Monitoring and alerting

- *CloudWatch* collects metrics from the ALB, EC2, RDS and the Auto Scaling group.
- *CloudWatch alarms* trigger on conditions such as high CPU, unhealthy host count and RDS storage or connection thresholds.
- *SNS* delivers alarm notifications by email or SMS.
- *CloudWatch dashboards* give a single view of application health.

---

## Deployment guide

The steps below are in recommended build order. Use the console, CloudFormation, Terraform or the CLI.

1. *VPC:* create the VPC, two public subnets and two private subnets across two AZs.
2. *Internet access:* attach an Internet Gateway, create the public route table (0.0.0.0/0 → IGW) and associate it with the public subnets.
3. *NAT Gateways:* create one per AZ in the public subnets. Create a private route table per AZ (0.0.0.0/0 → NAT) and associate it with the private subnets.
4. *Security groups and NACLs:* create ALB-SG, EC2-SG and RDS-SG and configure the NACLs.
5. *RDS:* create a DB subnet group from the private subnets, then launch a Multi-AZ MySQL or PostgreSQL instance with RDS-SG.
6. *IAM:* create an EC2 instance role with the SSM managed policy.
7. *Launch Template:* define the AMI, instance type, instance profile, EC2-SG and user data to install the app.
8. *ALB:* create the ALB in the public subnets, a target group with health checks and a listener (HTTP/HTTPS).
9. *Auto Scaling group:* create the ASG in the private subnets, attach it to the target group and add a target tracking policy.
10. *WAF:* create a web ACL with the OWASP-related managed rules and associate it with the ALB.
11. *S3 and CloudFront:* create the static assets bucket and a distribution with the S3 and ALB origins.
12. *Route 53:* create the DNS record for the domain and configure health checks.
13. *Monitoring:* create CloudWatch alarms and a dashboard, plus an SNS topic with subscriptions.

---

## Validation checklist

- [ ] The site loads through the Route 53 domain name
- [ ] Static assets are served from CloudFront (check the X-Cache response header)
- [ ] ALB target group shows all instances as healthy
- [ ] Instances have no public IPs and are reachable only through Session Manager
- [ ] Private instances can reach the internet through the NAT Gateway
- [ ] Terminating an instance triggers the ASG to launch a replacement
- [ ] A load test scales the ASG out and back in
- [ ] Rebooting the RDS primary with failover promotes the standby
- [ ] A WAF test rule blocks a sample malicious request
- [ ] A CloudWatch alarm triggers an SNS notification

---

## Cleanup

To avoid ongoing charges, delete resources in this order:

1. CloudFront distribution (disable first) and S3 bucket contents
2. Auto Scaling group, Launch Template, ALB and target group
3. RDS instance (take a final snapshot if needed)
4. NAT Gateways and release their Elastic IPs
5. WAF web ACL, CloudWatch alarms, dashboards and SNS topic
6. Route 53 records and health checks
7. VPC, subnets, route tables, Internet Gateway and security groups

NAT Gateways, RDS Multi-AZ and the ALB are billed hourly. Tear them down when you finish testing.


---

## Learning outcomes

- Design a VPC with correct subnet, route table and NAT Gateway configuration
- Build a highly available architecture across multiple Availability Zones
- Configure ALB listener rules and target group health checks
- Implement Auto Scaling with target tracking and step scaling policies
- Secure applications with WAF, Security Groups and private subnets
- Use Systems Manager Session Manager as a bastion-free access alternative

---

## Possible improvements

- Add HTTPS with an ACM certificate on CloudFront and the ALB
- Add step scaling policies alongside target tracking
- Store database credentials in Secrets Manager
- Add VPC Flow Logs and ALB access logs
- Define the whole stack as infrastructure as code (Terraform or CloudFormation)
- Add a CI/CD pipeline for application deployments
