# AWS Systems Manager

## 1. What exactly is AWS Systems Manager?

AWS Systems Manager is a centralized AWS service for operating, managing, and automating your infrastructure.

Imagine you have deployed an application on AWS. Your infrastructure contains:

- 5 EC2 instances running your application
- 1 RDS database
- 1 Application Load Balancer
- Several configuration values needed by your application

After deployment, you still need to maintain your infrastructure. You might need to inspect servers, run commands, install updates, and manage configurations.

AWS Systems Manager helps you perform many of these operational tasks from a central place.

> **Important clarification:** Systems Manager does not replace EC2, RDS, or your other AWS services. It provides tools to manage and operate supported resources.

---

## 2. Understand the problem it solves

Let's compare two approaches.

### Without Systems Manager

1. Connect to an EC2 instance using SSH.
2. Run Linux commands manually.
3. Repeat the process for other instances.
4. Maintain separate scripts and access methods.

### With Systems Manager

1. Open the Systems Manager console.
2. Select the managed instances you need.
3. Run supported commands, open sessions, or automate tasks.
4. Review execution results centrally.

For example, if you need to check disk space on 20 Linux servers, you can use Systems Manager to run `df -h` across the selected managed instances rather than logging into each server individually.

---

## 3. How does Systems Manager communicate with EC2?

This is the most important concept to understand before learning its individual features.

There are three main components involved in managing an EC2 instance through Systems Manager:

```text
You / Administrator
(AWS Console or AWS CLI)
          |
          v
AWS Systems Manager
(Receives authorized requests and coordinates operations)
          |
          v
SSM Agent on EC2
(Communicates with Systems Manager and performs supported tasks)
```

### A. AWS Systems Manager

This is the AWS-managed service that coordinates the operation you request.

For example, you ask Systems Manager to execute a command on an EC2 instance. Systems Manager handles the request, authorization checks, and coordination with the managed instance.

### B. SSM Agent

The SSM Agent is software that runs on the EC2 instance.

It communicates with the Systems Manager service and performs supported operations requested through that service.

For example, when you request a shell command through Run Command, the agent receives the task and executes it on the instance.

Many current AWS machine images include the agent, but you should verify its installation and status for your chosen operating system.

### C. IAM permissions

IAM controls who can perform operations and which resources they can access.

There are two separate permission considerations:

- **Instance permissions:** The EC2 instance generally needs an IAM role that grants the agent permission to communicate with Systems Manager.
- **Your permissions:** Your IAM user or role needs authorization to request the operation, such as starting a session or running a command.

Having access to the Systems Manager console alone does not automatically grant access to every EC2 instance.

---

## 4. Does Systems Manager require SSH?

No. This is one of its major benefits.

Normally, SSH access to a Linux EC2 instance uses TCP port 22. With Session Manager, you can access a supported, configured instance without opening inbound port 22.

However, Systems Manager still needs network connectivity. An instance may communicate with the service through:

- Internet access


# SSM Agent

## 1. What is the SSM Agent?

The SSM Agent is software installed on an EC2 instance that allows AWS Systems Manager to communicate with and manage that instance.

Think of it this way:

- **EC2** is your virtual server.
- **Systems Manager** is the service that coordinates management tasks.
- **SSM Agent** runs on the server and carries out supported tasks.

```text
AWS Systems Manager
(Receives and coordinates your request)
          |
          v
Your EC2 instance
          |
          v
SSM Agent
(Communicates with Systems Manager and performs tasks)
```

For example, when you request a remote command to check disk space, Systems Manager coordinates the request and the agent executes the command on your instance.

---

## 2. Why is the agent important?

Without a functioning agent, Systems Manager generally cannot perform agent-based management tasks on that instance.

For Systems Manager to work, you typically need:

1. SSM Agent installed and running.
2. IAM permissions that allow the instance to communicate with Systems Manager.
3. Network connectivity to the required Systems Manager endpoints.

> **Note:** All three matter. Having an IAM role alone does not guarantee that the instance is manageable.
- A NAT gateway
- Appropriately configured VPC endpoints

Also, Systems Manager is not completely independent of the network or IAM configuration. If the agent cannot communicate with the service, or its permissions are missing, the operation may fail.
