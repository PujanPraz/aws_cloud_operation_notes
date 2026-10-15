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


# IAM Role for EC2 and Systems Manager

## 1. Why does an EC2 instance need an IAM role?

Imagine you have an EC2 instance and want Systems Manager to execute a command on it.

The SSM Agent needs permission to communicate with the Systems Manager service. But how does AWS know that the instance is authorized?

Through an IAM role attached to the EC2 instance.

Think of it like an employee ID card:

- **EC2** is your employee.
- **IAM role** is the employee's authorization.
- **IAM policies** define what the employee is allowed to do.

Without the appropriate permissions, the agent may be unable to register the instance or communicate with Systems Manager.

---

## 2. How does it work?

```text
EC2 instance
(Runs the SSM Agent)
          |
          v
IAM instance role
(Provides temporary AWS credentials and permissions)
          |
          v
AWS Systems Manager
(Accepts authorized requests from the agent)
```

The instance role is not your personal IAM user. It is an identity assigned to the EC2 instance itself.

---

## 3. The important policy: `AmazonSSMManagedInstanceCore`

AWS provides a managed IAM policy named `AmazonSSMManagedInstanceCore`.

This policy grants the core permissions an EC2 instance typically needs for SSM Agent communication with Systems Manager.

You generally attach this policy to an IAM role that EC2 can assume.

There are two separate concepts:

- **IAM role:** The identity assigned to the instance.
- **IAM policy:** The permissions associated with that identity.

For example, a role might be named `EC2-SSM-Role`, and the role might have the `AmazonSSMManagedInstanceCore` policy attached.

> **Note:** The role name is your choice; the policy name is the AWS-provided policy.


# Session Manager

## 1. What is Session Manager?

Session Manager lets you connect to an EC2 instance through your browser or AWS CLI and interact with its terminal without opening an SSH port or managing SSH keys.

For example, suppose you have an EC2 instance running Ubuntu. You want to execute Linux commands on it, but you haven't configured SSH access.

With Session Manager, you can open a terminal session and run commands such as:

```bash
ls
pwd
top
```

---

## 2. How does it work?

```text
You
(Open Session Manager in the AWS Console)
          |
          v
AWS Systems Manager
(Authenticates and authorizes the session)
          |
          v
SSM Agent on your EC2 instance
(Opens the shell session on the instance)
```

Here's the important part: the SSM Agent initiates outbound communication with Systems Manager. You generally don't need to open inbound port `22` for SSH or port `3389` for RDP to use Session Manager.

---

## 3. What must be configured?

Your EC2 instance needs these prerequisites:

- **SSM Agent:** installed and running.
- **IAM permissions:** an attached instance role with appropriate Systems Manager permissions, commonly `AmazonSSMManagedInstanceCore`.
- **Network access:** connectivity to the required Systems Manager endpoints, typically over HTTPS on port `443`.
- **Session permissions:** your IAM user or role must be allowed to start a session.

> **Note:** The instance's IAM role and your own IAM permissions are different: one lets the instance communicate with Systems Manager; the other lets you initiate a session.

---

## 4. Why is Session Manager useful?

- Access instances without distributing SSH private keys.
- Manage instances in private subnets when the required network connectivity is configured.
- Control access using IAM permissions.
- Optionally log session activity to CloudWatch Logs or Amazon S3, depending on configuration and logging limitations.

> **Caveat:** Session Manager doesn't automatically make every EC2 instance accessible. The prerequisites must still be satisfied.

# Run Command

## 1. What is Run Command?

Run Command is an AWS Systems Manager feature that lets you execute commands or scripts on one or multiple managed EC2 instances remotely, without logging into each instance manually.

For example, suppose you manage 20 EC2 instances and need to check their disk usage. Instead of connecting to each instance through SSH, you can use Run Command to execute the same Linux command across all 20 instances.

```bash
df -h
```

This command displays disk space usage.

---

## 2. How does Run Command work?

```text
1. Administrator
   Chooses a command and target instances
          |
          v
2. Systems Manager
   Sends the command using an SSM document
          |
          v
3. SSM Agent
   Executes the command on each target instance
          |
          v
4. Results
   View execution status and command output
```

The SSM Agent is essential because it receives and executes commands on the instance.

---

## 3. What is an SSM document?

An SSM document defines the actions Systems Manager should perform. Think of it as a reusable instruction template.

AWS provides predefined documents, and you can also create your own.

Examples:

| Document | Purpose |
|----------|---------|
| `AWS-RunShellScript` | Execute shell commands on Linux instances |
| `AWS-RunPowerShellScript` | Execute PowerShell commands on Windows instances |
| `AWS-ConfigureAWSPackage` | Install or manage supported packages |

For example, using `AWS-RunShellScript`, you could run:

```bash
uptime
free -h
df -h
```

This checks system uptime, memory usage, and disk usage.

---

## 4. Run Command vs. Session Manager

Both let you manage instances remotely, but they serve different purposes.

| Feature | Run Command | Session Manager |
|---------|-------------|-----------------|
| Main purpose | Execute commands or scripts | Interactive terminal access |
| Typical use | Automate a task across many instances | Troubleshoot one instance interactively |
| Requires SSH port 22? | No | No |
| Output | Execution status and command output | Live interactive terminal |

> **Practical example:** If you want to restart a service on 50 EC2 instances, Run Command is a good choice. If you want to investigate why one instance is behaving strangely, Session Manager is often more convenient.
