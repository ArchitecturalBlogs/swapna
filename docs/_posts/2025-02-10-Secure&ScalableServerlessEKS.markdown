---
layout: post
title:  "Secure Serverless EKS: A DevSecOps Journey with Fargate Profile"
date:   2025-02-10 11:47:00 - 0600
categories: EKS, Serverless, Fargate profile, terraform, helm, frontend, backend, devsecops
---
## Deploying Secure Frontend and Backend Services to EKS Using Fargate with Terraform, GitHub Workflows, and DevSecOps

## Introduction

In modern cloud-native architectures, security is paramount. This article integrates DevSecOps principles into the process of deploying frontend and backend services from a master repository to an AWS EKS (Elastic Kubernetes Service) cluster using AWS Fargate profiles. The infrastructure is provisioned using the standard terraform-aws-modules/eks/aws Terraform module, with security enhancements such as KMS encryption, IAM security policies, and IRSA. Additionally, we provision a VPC, NAT Gateway, and Elastic IP using the terraform-aws-modules/vpc/aws module to ensure a scalable and secure network environment. GitHub Actions automate CI/CD pipelines, while Helm ensures secure and consistent deployments.

## Prerequisites

Ensure you have the following prerequisites in place:

1. An AWS account with necessary IAM permissions

2. AWS CLI, kubectl, and Helm installed

3. Terraform installed

4. GitHub repository with separate directories for frontend and backend code

5. GitHub Actions enabled

6. Docker installed for local development and testing

## Infrastructure Architecture

### EKS Cluster with Fargate Profile
The architecture includes a serverless EKS cluster with Fargate profiles for compute workloads. The cluster is deployed in a highly available VPC spanning multiple availability zones. Key components include:

* Private Subnets: Pods are deployed in private subnets, ensuring they do not have direct internet access.

* NAT Gateway: Provides internet access for private subnet resources to pull dependencies securely.

* Elastic IPs: Ensures stable and consistent public IPs for NAT Gateway connectivity.

* Application Load Balancer (ALB): Handles external traffic routing through Ingress Controller and Load Balancer Controller installed on the EKS cluster.

This setup ensures security, scalability, and controlled access to internet resources while allowing external access to deployed applications.

![EKS Serverless Cluster Architecture](/swapna/images/EKSServerlessClusterArchitecture.png)

### VPC Configuration

The Virtual Private Cloud (VPC) is designed using the terraform-aws-modules/vpc/aws module and includes:

* Public Subnets: Used for the ALB and NAT Gateway.

* Private Subnets: Used for Fargate workloads and EKS pods.

* NAT Gateway: Enables internet access for private subnets without exposing internal resources.

* Elastic IPs: Ensures persistent public IP addresses for NAT Gateway.

### Provisioning EKS with Load Balancer Controller

The EKS cluster is created with security best practices:

* KMS encryption for secrets and cluster data.

* IAM Roles for Service Accounts (IRSA) to securely access AWS services.

* Cluster creator admin permissions for operational flexibility.

### Install IAM Role for Service Account (IRSA)
This allows Kubernetes services to interact with AWS resources securely.

* Deploy AWS Load Balancer Controller using Helm

* Manages the provisioning of AWS ALBs/NLBs dynamically within the EKS environment.

* Ensures secure and efficient traffic routing to applications deployed in private subnets.

## DevSecOps Workflow

### Infrastructure as Code(IaC): 
Infrastructure is managed entirely through Terraform, ensuring consistency across environments. The GitHub Actions workflow automates Terraform deployment and applies security checks before provisioning the infrastructure.

![EKS Serverless Cluster Creation/Upgrade Github Workflow](/swapna/images/IaC-EKSClusterCreation&Deploy.png)

### CI/CD(Continous Integration and Deployments):
The pipeline includes:

* Build and Push: Builds the application and runs unit tests.

* Security Scanning: Scans Docker images for vulnerabilities before deployment.

* Automated Deployments: Deploys applications securely using Helm charts.

![EKS Services Continous integration and deployments](/swapna/images/CICD-EKS_Deployments.png)

Conclusion

With the terraform-aws-modules/vpc/aws module managing networking and terraform-aws-modules/eks/aws handling cluster provisioning, we can efficiently deploy and scale frontend and backend services on AWS EKS with Fargate. DevSecOps principles, including encryption, IAM security, and automated vulnerability scans, ensure a robust security posture for cloud-native applications.

For complete source code, refer to the **[GitHub Repository](https://github.com/DevOpsProjectFinal/masterrepo)**