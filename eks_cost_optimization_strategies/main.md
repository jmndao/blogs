# Spot the Savings: EKS Cost Optimization Strategies with Terraform

My goal with this post is to share my experience with EKS cost optimization strategies. I will show you how to use Terraform to implement these strategies. 


Let's start with a quick overview of the EKS cost model.

## EKS Components

AWS EKS is a managed Kubernetes service which is globally comprised of three components:
- Control Plane
- Worker Nodes
- Networking

## EKS Cost Model

EKS is a managed Kubernetes service. It is a managed control plane that runs on EC2 instances. You pay for the control plane and the EC2 instances that run your workloads.

The control plane is charged at $0.10 per hour. This sums up to a monthly (30 days) bill of $72 for the control plane only.

The EC2 instances are charged at the regular rates. The EC2 rates depend on the instance type and the region. For example, the EC2 rate for a t3.medium instance in the us-east-1 region is $0.0416 per hour. 







