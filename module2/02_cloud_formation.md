# AWS CloudFormation

## 1. What is AWS CloudFormation?

AWS CloudFormation is an Infrastructure as Code (IaC) service that lets you create, configure, and manage AWS resources using templates instead of manually creating each resource in the AWS Console.

For example, suppose you need to create:

- An EC2 instance
- A VPC
- Two subnets
- A security group
- An Application Load Balancer

You could create each resource manually, or define them in a CloudFormation template and let AWS create the infrastructure for you.

<img width="1400" height="620" alt="image" src="https://github.com/user-attachments/assets/95b04008-8c4d-4f37-8d86-4350311086b9" />

---

## 2. Why do we need CloudFormation?

Imagine you manually build a VPC with public and private subnets, route tables, security groups, and EC2 instances.

If you need the same architecture for development, testing, and production, recreating everything manually is time-consuming and error-prone.

CloudFormation helps by providing:

- **Automation:** create many resources from one template.
- **Consistency:** use the same template to create similar environments.
- **Repeatability:** recreate infrastructure when needed.
- **Version control:** store templates in Git and track changes.
- **Dependency management:** CloudFormation determines the order in which dependent resources need to be created.
- **Lifecycle management:** update or delete resources managed by a stack, subject to their configuration and retention policies.

---

## 3. How does CloudFormation work?

```text
1. Write a template
   Define the AWS resources in YAML or JSON
          |
          v
2. Create a stack
   Submit the template to CloudFormation
          |
          v
3. CloudFormation provisions resources
   Creates resources and handles dependencies
          |
          v
4. Manage the infrastructure
   Review, update, or delete the stack
```

---

## 4. What is a CloudFormation template?

A template is a text file written in YAML or JSON that describes the infrastructure you want.

For example, this YAML template defines a basic EC2 instance:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Create a basic EC2 instance

Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-xxxxxxxxxxxxxxxxx
      InstanceType: t3.micro
```

> This is an illustrative template. The AMI ID must be replaced with a valid, compatible AMI for your chosen AWS Region. Additional configuration may be needed for networking and access.

Let's understand the important parts:

- **Resources:** defines the AWS resources to create.
- **MyEC2Instance:** the logical name used to refer to the resource within the template.
- **Type:** specifies which AWS resource CloudFormation should create.
- **Properties:** contains the resource's configuration, such as its AMI and instance type.

---

## 5. What is a stack?

A stack is a collection of AWS resources that CloudFormation manages as a unit based on a template.

For example, a web application stack might contain:

- A VPC
- Public and private subnets
- Security groups
- EC2 instances
- An Application Load Balancer

You can update the stack by submitting an updated template. CloudFormation calculates the changes and attempts to apply them.

You can also delete a stack, but be careful: resources may be deleted with it unless retention policies or other protections apply.

---

## 6. Important distinction: Template vs. Stack

| Template | Stack |
|----------|-------|
| Defines the desired infrastructure | Represents the deployed infrastructure managed by CloudFormation |
| A YAML or JSON file | An AWS resource managed by CloudFormation |
| Can be reused to create multiple environments | Represents one particular deployment |
| Describes what should be created | Tracks the resources and operations for that deployment |

> **Analogy:** Think of a template as a blueprint and a stack as the building created from that blueprint.

---

## 7. CloudFormation vs. manually creating resources

| Manual AWS Console | CloudFormation |
|--------------------|----------------|
| Create resources one at a time | Define resources in a template |
| Easy to make inconsistent configurations | Reuse templates for consistency |
| Repeating deployments takes time | Reuse templates to automate deployments |
| Changes can be hard to track | Templates can be version-controlled |

CloudFormation is not the only IaC tool. Terraform is another popular option, but it is a separate tool with its own workflow and state management.
