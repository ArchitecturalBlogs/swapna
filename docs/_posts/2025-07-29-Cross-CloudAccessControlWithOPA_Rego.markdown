---
layout: post
title:  "Cross-cloud access control with OPA Rego"
date:   2025-07-29 01:39:00 - 0700
categories: #OPA #Rego #PolicyAsCode #CloudSecurity #AWS #Azure #S3 #BlobStorage #IAM #Authorization #DevSecOps #ZeroTrust #CloudNative #MultiCloud
---
## Cross-cloud access control with OPA Rego – AWS S3 & Azure Storage

## Introduction

I set up authorization policies using the Open Policy Agent (OPA) with Rego to do fine-grained access control in multi-cloud environments on:

* AWS S3 Buckets
* Azure Blob Storage Containers

## Use Case:
Rego policies were defined to control access based on user identity, roles, and action types.
Rules enforced include:
* Deny write access to public S3 buckets
* Block delete actions on Azure containers unless explicitly allowed.

Input claims, which come from a JWT/identity provider, were utilized to make dynamic access decisions.
The set of Rego policies were tested using opa test --verbose and enhanced with print() statements for debugging, which proved to be very helpful in validating conditions across different identity providers and storage models.

This approach can also be extended to multiple stages of the cloud lifecycle:

* At runtime: Intercept API calls before execution using tools like AWS API Gateway or Azure API Management, and enforce decisions through OPA policies.
* During provisioning: Integrate with Terraform (via tools like OPA Terraform Validator or Sentinel/OPA plugins) to enforce policies before infrastructure is created, preventing misconfigurations at source.
* Post-deployment: Continuously scan and evaluate existing cloud resources (S3 buckets, Azure storage containers, IAM roles, etc.) against Rego policies to detect drift or violations.

This enables unified, consistent, and automated governance across AWS and Azure — whether it's proactive prevention or reactive detection.

The result: centralized, consistent, and scalable access checks in a zero-trust architecture.

#OPA #Rego #PolicyAsCode #CloudSecurity #AWS #Azure #S3 #BlobStorage #IAM #Authorization #DevSecOps #ZeroTrust #CloudNative #MultiCloud