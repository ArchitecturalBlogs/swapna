---
layout: post
title:  "Policy Engine using OPA (Open Policy Agent)"
date:   2024-12-12 12:18:25 -0600
categories: Policy Engines, Open Policy Agent (OPA), Access Control, Firewall Rules
---
## From Rules to Action: Exploring the Power of Policy Engines using Open Policy Agent (OPA)

## What is a Policy Engine?
A policy engine is a tool that helps automate decisions based on rules. For example, it can decide who can access certain data or actions in a system. By using predefined rules, policy engines ensure everything runs securely and consistently, without the need for manual checks.

## Why Use OPA?
Open Policy Agent (OPA) is an open-source tool that helps you enforce policies across different systems. It makes decision-making easier, faster, and more consistent, ensuring security and compliance in a scalable way. OPA is flexible and can be integrated with a wide range of systems like Kubernetes, APIs, and cloud platforms.  
Learn more about OPA: [OPA Website](https://www.openpolicyagent.org)

## Overview of OPA
OPA is a general-purpose policy engine that allows you to define, manage, and enforce rules (policies) in your systems. It uses a simple, flexible language called Rego to write those rules. OPA is designed to be lightweight and scalable, making it a great choice for modern cloud applications, microservices, and infrastructure management.

## How OPA Works
OPA makes decisions based on data it receives. You give OPA some data (like who the user is or what resource they’re trying to access), and it checks that against its policies to decide if the action should be allowed or denied. The decision is then sent back to the system.

## Key Concepts in OPA
1. **Rego: The Policy Language**  
OPA uses a language called Rego to write policies. Rego is simple to understand and lets you define rules that control access to data or resources.

   For example, a rule might say:  
   Allow access if the user is an admin:
   ```rego
   allow {
       input.user == "admin"
       input.resource == "sensitive-data"
   }
    ```
In this case, only users who are admins are allowed access to sensitive data.

2. OPA Architecture
OPA can be used in many different environments. You can run it alongside your applications or as a centralized service. It evaluates policies in real time and returns a decision based on the rules you’ve set.

## Usaecases for OPA
1. Access Control: Role Based Access Control(RBAC) and Attributes Based Access Control(ABAC)
2. Security and Compliance: Policies to verify encrption or authorized users are allowed to access certain resources.
3. It can be used in other decision making ona defined rules which less complexity.

## Other products for Policy engines
There are several paid policy engines and commercial solutions that provide policy enforcement, authorization, and governance capabilities, often with additional enterprise features. Some of them I have listed below.

1. Auth0 (Authorization Core and Fine-Grained Access Control)
Type: SaaS-based identity and access management.
Website: auth0.com
2. Okta (Adaptive Access Policies)
Type: Identity and Access Management (IAM).
Website: okta.com
3. Axiomatics Policy Server (APS)
Type: Attribute-Based Access Control (ABAC) engine.
4. IBM ODM
Type:  Business rules for decision-making.
Website: IBM ODM


## Usecase for RBAC(Role Based Access Control)
Let's take an exmaple for RBAC with 2 users who login to a website:
1. Admins have full access to all features like view, edit, delete.
2. User/Viewer have view-only access.

## Steps to Implement OPA for Role-Based Access Management

1. Install OPA
Install OPA on your system or include it as part of your application's deployment.

For macOS (using Homebrew):
brew install opa
For Linux (using curl):
```bash
curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64
chmod +x opa
sudo mv opa /usr/local/bin/
```
Check the installation:
```bash
opa version
```
2. Write the RBAC and ABAC Policy in Rego
Create a policy file named rbac.rego. This policy checks the role of the user and grants permissions accordingly.

RBAC Policy Example (Rego)
```rego
package authz

# Default deny all access
default allow = false

# Admins have full access to all actions
allow {
    input.role == "admin"
}

# Users (non-admins) can only perform view actions
allow {
    input.role == "user"
    input.action == "view"
}

# Attribute-Based Access Control (ABAC)
# Example: Only allow access if the user is in the allowed department AND during office hours
allow {
    input.role == "employee"
    input.action == "view"
    input.department == "sales"
    is_office_hours()
}

# Helper function to check office hours (8 AM to 6 PM)
is_office_hours() {
    input.time >= "08:00"  # Start time
    input.time <= "18:00"  # End time
}
```

* RBAC Rules:
Admins (role == "admin") can access all features.
Users (role == "user") can only perform the view action.
* ABAC Rule:
Employees (role == "employee") must meet specific conditions:
They are part of the "sales" department (input.department == "sales").
They access resources only during office hours (is_office_hours()).
* Helper Function:
The is_office_hours() function ensures access is restricted to a time window (8:00 AM - 6:00 PM)

3. Test the Policy Locally
Start the OPA server to load the policy and test it with different inputs.

Run OPA as a Server
```bash
opa run --server
```
OPA will now be running at http://localhost:8181.

Test the Policy Using Input Data

Use curl to test the policy for different roles.

* Admin Role (Full Access):
```bash
curl -X POST http://localhost:8181/v1/data/authz/allow \
-d '{ "role": "admin", "action": "edit" }'
```
Response:
```bash
{
  "result": true
}
```
* User Role (View Only):
```bash
curl -X POST http://localhost:8181/v1/data/authz/allow \
-d '{ "role": "user", "action": "view" }'
```
Response:
```bash
{
  "result": true
}
```
User Role Trying Edit Access:
```bash
curl -X POST http://localhost:8181/v1/data/authz/allow \
-d '{ "role": "user", "action": "edit" }'
```bash
Response:
```bash
{
  "result": false
}
```
* Employee Role (ABAC - Allowed During Office Hours):
```bash
curl -X POST http://localhost:8181/v1/data/authz/allow \
-d '{ "role": "employee", "action": "view", "department": "sales", "time": "09:30" }'
```bash
Response:
```bash
{ "result": true }
```
* Employee Role (Outside Office Hours):
```bash
curl -X POST http://localhost:8181/v1/data/authz/allow \
-d '{ "role": "employee", "action": "view", "department": "sales", "time": "19:00" }'
```
Response:
```bash
{ "result": false }
```
* Unauthorized Department:
```bash
curl -X POST http://localhost:8181/v1/data/authz/allow \
-d '{ "role": "employee", "action": "view", "department": "hr", "time": "10:00" }'
```
Response:
```bash
{ "result": false }
```

4. Integrate OPA with Your Application
To enforce the policy in your application, make an HTTP request to OPA's API with the user's role and action as input.

Example Integration in Node.js

Here's how you can call OPA to make access control decisions.
```node
const fetch = require('node-fetch');

// Function to check access
async function checkAccess(role, action) {
    const response = await fetch('http://localhost:8181/v1/data/authz/allow', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ role: role, action: action }),
    });
    const result = await response.json();
    return result.result; // true or false
}

// Example Usage
(async () => {
    const role = "user"; // Could be "admin" or "user"
    const action = "edit"; // Action the user wants to perform

    const isAllowed = await checkAccess(role, action);
    if (isAllowed) {
        console.log("Access granted");
    } else {
        console.log("Access denied");
    }
})();
```

5. Deploy OPA with Your Application
You can deploy OPA as a sidecar container (in Kubernetes) or a standalone service alongside your application.

Docker Example:

Create a Docker container with OPA and the policy file:
```bash
docker run -p 8181:8181 -v $(pwd):/policies openpolicyagent/opa:latest run --server /policies/rbac.rego
```
Your application can now send policy evaluation requests to http://localhost:8181/v1/data/authz/allow.
```bash
curl -X PUT --data-binary @rbac.rego http://localhost:8181/v1/policies/rbac
```
## Conclusion
OPA(Open Policy Agent) offers multiple solutions to business rules with higher complexity. I know of the case where we have achieved this same model using the Serverless Architecture using API Gatewy, Lambda and Dynamodb. We need to choose wisely based on the use case like performance, how often you need your code changes to be deployed to have the access controls to be applied, scalibility, potability and cost. OPA can be considered for the following defined usecase.
* Centralized, externalized policy management.
* Complex, fine-grained policies (RBAC, ABAC).
* Scalability and performance.
* Auditing and compliance for large-scale systems.
* Lower Cost.