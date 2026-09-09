# Highly Available Web App on EC2 (ALB + Auto Scaling)

## Goal
Deploy a scalable, highly available web application behind an Application Load Balancer with Auto Scaling, mirroring a real production service.

## Architecture

```mermaid
flowchart TB
    Internet((Internet)) --> IGW[Internet Gateway]
    IGW --> ALB[Application Load Balancer<br/>public subnets, both AZs]

    subgraph VPC["VPC — 10.0.0.0/16"]
        subgraph AZ1["Availability Zone A"]
            PubA[Public Subnet<br/>NAT Gateway]
            PrivA[Private Subnet<br/>EC2 instance - ASG]
            PrivA -.outbound via NAT.-> PubA
        end

        subgraph AZ2["Availability Zone B"]
            PubB[Public Subnet<br/>NAT Gateway]
            PrivB[Private Subnet<br/>EC2 instance - ASG]
            PrivB -.outbound via NAT.-> PubB
        end

        ALB --> PrivA
        ALB --> PrivB
    end

    TG[Target Group] -.health checks.- PrivA
    TG -.health checks.- PrivB
    CW[CloudWatch Alarm<br/>CPU > 60%] -.triggers.-> ASG[Auto Scaling Group<br/>min 2 / desired 2 / max 4]
    ASG -.manages.-> PrivA
    ASG -.manages.-> PrivB
```

**Flow:** Internet → Internet Gateway → Application Load Balancer (public subnets, spread across 2 AZs) → Target Group → EC2 instances (private subnets, spread across 2 AZs, managed by an Auto Scaling Group). Instances reach the internet outbound (for updates) through a NAT Gateway sitting in each AZ's public subnet. A CloudWatch alarm on average CPU utilization drives the ASG's scaling policy.

**Security groups:**
- `alb-sg` — inbound HTTP (80) from `0.0.0.0/0`
- `test-SG1` — inbound HTTP (80) from `alb-sg` only (no direct internet access to instances)
- `test-SG2` — inbound HTTP (80) from `alb-sg` only (no direct internet access to instances) 

## Screenshots
- ALB DNS test (curl output alternating between instance IDs)

   ![ALB DNS test showing alternating instance IDs](project/screenshot/week-2.png)
   ![ALB DNS test showing alternating instance IDs](project/screenshot/week1.png)

- ASG activity log (instance termination + replacement event)

   ![Replacement event](project/screenshot/asg-2.png)
   ![AsG](project/screenshot/asg.png)


- Target Group health (both targets showing Healthy)

   ![Target group health](project/screenshot/test-target-group.png)


## What I Learned
Built a scalable, highly available architecture with built-in redundancy and fault tolerance — spreading compute across two Availability Zones behind a load balancer, with an Auto Scaling Group automatically replacing unhealthy or terminated instances to keep the service running.

## Issues Faced
- **Target Group not picking up instances:** I ran into trouble getting my EC2 instances attached to the Target Group. Resolved by working through the registration flow properly (the instances need to be explicitly registered as targets — the ALB doesn't discover them automatically).
- **Couldn't connect to the ALB DNS name:** I was testing over HTTPS while the ALB listener was only configured for HTTP (port 80). Fixed by using `http://` instead of `https://` when testing the DNS name.
