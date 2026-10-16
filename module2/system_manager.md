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

# Patch Manager

## 1. What is Patch Manager?

AWS Systems Manager Patch Manager helps you automate the process of applying operating system updates and security patches to managed instances.

Imagine a company has 100 EC2 instances running Ubuntu, Amazon Linux, and Windows Server. Manually checking and updating each instance would take a lot of time.

Patch Manager helps administrators define patching rules, scan instances for missing patches, and install applicable updates at scale.

---

## 2. Why do we need patch management?

Operating systems receive updates to fix vulnerabilities, bugs, and reliability issues.

For example, suppose a security vulnerability is discovered in an installed Linux package. If the company doesn't update affected instances, attackers might exploit that vulnerability.

Patch management helps organizations:

- Apply security updates consistently.
- Reduce exposure to known vulnerabilities.
- Automate routine maintenance.
- Track which instances are compliant with patching policies.
- Manage updates across multiple instances.

> **Important:** Installing patches is not a guarantee against every security vulnerability.

---

## 3. How does Patch Manager work?

```text
1. Define patch rules
   Specify which patches are approved
          |
          v
2. Scan instances
   Identify missing or applicable patches
          |
          v
3. Install patches
   Apply approved updates when installation is requested or scheduled
          |
          v
4. Check compliance
   Review patch status and outstanding updates
```

Patch scanning and patch installation are different operations. A scan identifies patch status; it does not itself install missing updates.

---

## 4. Important Patch Manager concepts

### A. Patch baseline

A patch baseline defines which patches are approved for installation.

For example, an administrator might approve security updates while excluding certain packages until they have been tested.

Patch baselines can use operating-system-specific rules, classifications, severities, and explicit approvals or rejections, depending on the operating system.

### B. Patch groups

A patch group lets you organize managed nodes for patching purposes, often using tags.

For example, a company could separate instances into:

- Development
- Testing
- Production

The team can test patches in development before applying them to production. A patch group can be associated with a patch baseline.

### C. Maintenance windows

Maintenance windows define when scheduled maintenance activities are allowed to run.

For example, an organization may schedule patching every Sunday at 2 AM to minimize business disruption.

A maintenance window is useful because patch installations can restart services or require an operating-system reboot.

### D. Compliance reporting

Patch compliance reporting helps you understand whether instances meet the organization's patching requirements.

An instance may be reported as noncompliant if it has required patches missing, depending on the configured baseline and compliance rules.

---

## 5. Patch Manager vs. Run Command

| Feature | Patch Manager | Run Command |
|---------|---------------|-------------|
| Main purpose | Manage operating-system patches | Execute commands or scripts |
| Typical task | Install approved security updates | Restart Nginx or check disk usage |
| Patch-specific baselines | Yes | No |
| Compliance reporting | Patch compliance | Command execution status and output |

Patch Manager uses Systems Manager capabilities behind the scenes, including SSM Agent and command execution mechanisms, to carry out patching operations.

# Parameter Store

## 1. What is Parameter Store?

AWS Systems Manager Parameter Store is a service for securely storing and retrieving configuration values and secrets used by applications.

Imagine you're building a FastAPI application and need to configure database settings, API keys, and environment-specific values. Instead of hardcoding these values into your Python code, you can store them in Parameter Store and retrieve them when needed.

For example, you might store:

| Parameter name | Example value | Type |
|----------------|---------------|------|
| `/myapp/dev/DB_HOST` | `dev-db.example.com` | String |
| `/myapp/prod/DB_HOST` | `prod-db.example.com` | String |
| `/myapp/prod/API_KEY` | A sensitive API key | SecureString |

> These are illustrative values, not real credentials.

---

## 2. Why do we need Parameter Store?

Consider this Python code:

```python
DATABASE_URL = "postgresql://user:password@host:5432/app"
```

Hardcoding credentials creates risks. Secrets can accidentally end up in source control, logs, or shared files.

With Parameter Store, your application retrieves configuration at runtime using the AWS SDK, such as `boto3`.

Benefits include:

- Centralized configuration management.
- Separation of development and production settings.
- Encryption for sensitive parameters.
- IAM-based access control.
- Parameter version history.
- Integration with other AWS services.

> **Important:** Parameter Store does not automatically make an application secure. You must configure permissions carefully and avoid exposing retrieved secrets in logs or responses.

---

## 3. The three parameter types

### String

Stores a single text value, such as a database hostname or application name.

Example: `/myapp/DB_HOST`

### StringList

Stores a comma-separated list of strings.

Example: `server1,server2,server3`

It isn't a general-purpose JSON or nested data format.

### SecureString

Stores sensitive text encrypted using AWS KMS.

Example: `/myapp/prod/DB_PASSWORD`

For SecureString, access permissions and KMS key permissions matter. Encryption does not mean every authorized reader is prevented from retrieving the plaintext value.

---

## 4. Parameter hierarchy

Parameter Store supports hierarchical names, which help organize configuration values by application and environment.

For example:

```text
/myapp/dev/DB_HOST
/myapp/dev/DB_PASSWORD
/myapp/prod/DB_HOST
/myapp/prod/DB_PASSWORD
```

An application can retrieve parameters by name or retrieve multiple parameters using a shared path.

This structure is useful when the same application runs in development, testing, and production environments.

---

## 5. Parameter Store vs. Secrets Manager

This is a common AWS interview question.

| Feature | Parameter Store | Secrets Manager |
|---------|-----------------|-----------------|
| Configuration values | Yes | Possible, but not its main focus |
| Encrypted secrets | Yes, using SecureString | Yes |
| Automatic secret rotation | Not built in as a general native rotation feature | Built-in rotation workflows and integrations |
| Typical use | Application settings and many simple secrets | Database credentials and secrets requiring rotation |
| Pricing | Standard parameters have a free tier, subject to current AWS terms | Generally paid per secret and API usage |

> **Rule of thumb:** Use Parameter Store for centralized configuration and many straightforward secrets. Consider Secrets Manager when automated secret rotation and dedicated secrets lifecycle management are important.

---

## 6. How would a FastAPI application use it?

At a high level, the workflow is:

1. Create a parameter in Parameter Store.
2. Give the application's IAM role permission to read that parameter.
3. Use `boto3` to retrieve it.
4. Pass the value into the application's configuration.

For example, the application might retrieve `/myapp/prod/DB_HOST` instead of hardcoding the production database hostname.

For sensitive SecureString values, the application must request decryption and have the necessary KMS permissions where applicable.

# Automation

## 1. What is Automation?

AWS Systems Manager Automation helps you automate common operational and administrative tasks across AWS resources.

Think of it as creating a repeatable workflow that performs several steps for you, instead of manually executing each step in the AWS Console.

For example, suppose an EC2 instance is experiencing an issue. You want to:

1. Stop the instance.
2. Create an EBS snapshot of its attached volume.
3. Restart the instance.
4. Verify its state.

You can design an Automation workflow to perform supported steps in sequence, with appropriate permissions and error handling.

---

## 2. What is a runbook?

A runbook is a document that defines the steps Automation should execute.

AWS provides predefined runbooks, and you can create custom runbooks for your own operational workflows.

A runbook is typically an SSM document written in YAML or JSON.

For example, AWS provides a runbook named:

```text
AWS-RestartEC2Instance
```

This runbook can restart a specified EC2 instance.

Other workflows can include steps to create snapshots, change resource configurations, or perform operational checks, depending on the runbook and its permissions.

---

## 3. How does Automation work?

```text
1. Trigger
   An administrator or supported automation trigger starts the workflow
          |
          v
2. Runbook
   Defines the actions and their order
          |
          v
3. Execute steps
   Performs actions against supported AWS resources or managed nodes
          |
          v
4. Review results
   Inspect step status, outputs, and failures
```

Depending on the workflow, Automation can call AWS APIs directly, run commands on managed instances, or invoke other supported actions. Not every Automation step requires the SSM Agent.

---

## 4. Real-world use cases

| Use case | Example |
|----------|---------|
| EC2 operations | Stop, start, or restart instances |
| Backup preparation | Create EBS snapshots before maintenance |
| Troubleshooting | Run diagnostic workflows |
| Resource remediation | Correct supported configuration problems |
| Multi-step operations | Execute several dependent actions in sequence |

Automation is especially useful when the same operational procedure needs to be performed consistently and repeatedly.

---

## 5. Automation vs. Run Command

This is an important interview distinction.

| Feature | Automation | Run Command |
|---------|------------|-------------|
| Main purpose | Automate workflows and operational procedures | Execute commands or scripts on managed nodes |
| Scope | Can coordinate multiple steps and AWS resource actions | Primarily runs commands on target managed nodes |
| Uses runbooks/documents | Yes | Uses command documents |
| Example | Snapshot a volume, then perform a maintenance workflow | Run `df -h` on 20 EC2 instances |

> **Easy way to remember:** Run Command executes a command; Automation orchestrates a workflow.

Automation can use Run Command as one step within a larger workflow.

---

## 6. Permissions and safety

Automation workflows can make changes to infrastructure, so permissions matter.

- The initiating user or role must be authorized to start the Automation execution.
- AWS resources must be accessible through the permissions used by the workflow.
- Some workflows use an IAM service role, also called an Automation assume role.
- Destructive operations should include suitable validation and safeguards.

> **Note:** Don't assume that every Automation runbook requires an EC2 instance role or that every runbook acts only on EC2 instances.

# Fleet Manager

## 1. What is Fleet Manager?

Fleet Manager is an AWS Systems Manager feature that provides a centralized interface to view and manage your fleet of servers, including supported EC2 instances and hybrid servers registered with Systems Manager.

Instead of navigating through multiple services and connecting to each instance individually, you can use Fleet Manager to inspect information about your managed nodes from one place.

Imagine a company manages 50 EC2 instances running different operating systems. Fleet Manager helps administrators view and inspect those servers through the AWS Console.

---

## 2. What can Fleet Manager do?

### A. View managed nodes

See registered servers and inspect details such as their instance information and management status.

### B. Inspect system information

Depending on the operating system and available permissions, inspect information about processes, users, filesystems, and other system details.

### C. Manage nodes remotely

Access supported remote management capabilities through the console, including connections using Session Manager where configured.

> **Note:** Fleet Manager's available capabilities depend on the node type, operating system, agent status, permissions, and configuration.

---

## 3. What is a managed node?

A managed node is a server that AWS Systems Manager has registered and can manage.

It may be:

- An EC2 instance.
- An on-premises server.
- A virtual machine in another environment, when configured for hybrid activation.

An EC2 instance existing in your AWS account does not automatically guarantee that it will appear as a managed node. It needs the appropriate Systems Manager setup, including agent and permissions, plus connectivity.

This is why we planned to check your EC2 instance's Ping status earlier.

---

## 4. Fleet Manager vs. EC2 Dashboard

| Feature | EC2 Dashboard | Fleet Manager |
|---------|---------------|---------------|
| Primary focus | AWS EC2 infrastructure | Systems Manager-managed nodes |
| View instance state | Yes | Managed-node status and details |
| Inspect operating-system details | Limited directly | Provides supported OS-level views |
| Manage on-premises servers | Not as EC2 instances | Supports registered hybrid nodes |
| Start a Session Manager session | Through supported integrations | Through supported integrations |

> **Key distinction:** EC2 focuses on the cloud infrastructure; Fleet Manager focuses on inspecting and managing servers registered with Systems Manager.

---

## 5. Interview scenario

Suppose a company has 30 EC2 instances and 10 on-premises Linux servers. Its administrator wants a centralized console to inspect the managed servers' operating-system information.

Fleet Manager is a suitable option, provided the on-premises servers have been registered and configured as Systems Manager managed nodes.

# Inventory

## 1. What is Inventory?

AWS Systems Manager Inventory collects metadata about your managed nodes so you can understand what is installed and configured on your servers.

For example, imagine you manage 100 EC2 instances and need to know:

- Which operating systems are installed?
- Which software packages are present?
- Which applications and versions are running?
- What hardware and network configuration information is available?

Instead of manually connecting to each instance, you can use Inventory to collect and review this information centrally.

---

## 2. What information can Inventory collect?

### A. Operating system information

OS name, version, and other supported system details.

### B. Installed applications and packages

Software names, versions, and package details supported by the operating system.

### C. Hardware and network details

Supported metadata about processors, network configuration, and other system properties.

### D. Custom inventory

You can configure custom inventory collection to report additional information that matters to your organization.

> **Note:** The exact inventory types available depend on the operating system and configuration.

---

## 3. How does Inventory work?

```text
1. Configure Inventory collection
   Choose the managed nodes and inventory types
          |
          v
2. SSM Agent collects metadata
   Gathers supported information from each node
          |
          v
3. Inventory data is reported
   Systems Manager stores the collected metadata for review
          |
          v
4. Analyze your fleet
   Review software, OS versions, and configuration details
```

Inventory collection must be configured; don't assume that every managed node automatically reports every possible inventory type.

---

## 4. Real-world example

Suppose your company has 60 EC2 instances. A security team wants to identify servers running an outdated version of a particular application.

Using Inventory, the team can collect installed software and version information, then identify affected instances.

The team can then use another appropriate tool or workflow to update those instances.

> **Important:** Inventory primarily collects and reports metadata. It does not itself patch software or automatically remove outdated applications.

---

## 5. Inventory vs. Fleet Manager

| Feature | Inventory | Fleet Manager |
|---------|-----------|---------------|
| Main purpose | Collect and query system metadata | Inspect and manage managed servers |
| Installed software details | Core use case | Some system information is available |
| OS and hardware metadata | Collects supported inventory data | Provides supported node details |
| Remediate an outdated package | Requires another action or workflow | Not its primary purpose |

An easy way to remember:

- **Fleet Manager:** inspect and manage your servers.
- **Inventory:** discover what's installed and configured on them.


  # State Manager

## 1. What is State Manager?

AWS Systems Manager State Manager helps you define and maintain a desired configuration for your managed nodes.

The key idea is configuration consistency. Instead of manually configuring every EC2 instance and hoping it stays that way, you define what the instance should have or how it should be configured, and State Manager can apply that configuration according to your association settings.

For example, suppose your company has 50 EC2 instances. You want a monitoring agent installed on every instance. State Manager can help apply the configuration across those instances and periodically reapply it.

---

## 2. How does it work?

``````text
1. Define the desired configuration
   Choose a document, targets, and schedule
          |
          v
2. Create an association
   Connect the document to your managed nodes
          |
          v
3. Apply the configuration
   SSM Agent executes the specified instructions
          |
          v
4. Maintain consistency
   Reapply instructions according to the configured schedule
``````

State Manager doesn't automatically detect and fix every configuration change. Its behavior depends on the document, association schedule, and instructions you configure.

---

## 3. What is an association?

An association connects an SSM document with target managed nodes and specifies how or when that document should run.

For example, you could create an association that:

- Targets instances tagged `Environment=Production`.
- Uses a document that checks or configures a required setting.
- Runs on a defined schedule, such as every 30 minutes.

If the document installs a required package, the package installation instructions must be designed appropriately. The association is what schedules and applies the document; it doesn't itself define the configuration logic.

---

## 4. Real-world use cases

| Requirement | How State Manager helps |
|-------------|-------------------------|
| Install required software | Apply a document across selected instances |
| Maintain a configuration | Reapply configuration instructions on a schedule |
| Run routine checks | Schedule a document to inspect system settings |
| Standardize a fleet | Apply the same configuration to many managed nodes |

---

## 5. State Manager vs. Run Command

Both can execute commands on managed nodes, but their purposes differ.

| Feature | State Manager | Run Command |
|---------|---------------|-------------|
| Primary purpose | Maintain a recurring configuration or operational state | Execute a command when requested |
| Scheduling | Associations can run on a schedule | Commands can be launched on demand or through automation |
| Repeated execution | Designed for ongoing associations | Not inherently a configuration-maintenance system |
| Example | Ensure a required agent remains configured | Check disk usage once across 20 instances |

> **Interview tip:** State Manager is useful when you want a configuration to be applied repeatedly, while Run Command is ideal for a specific command execution.
``````

# Maintenance Windows

## 1. What are Maintenance Windows?

AWS Systems Manager Maintenance Windows let you schedule when operational tasks should run on your managed nodes and other supported AWS resources.

For example, suppose your company runs a production application 24/7. You need to install security patches, but you don't want maintenance to occur during peak business hours.

You can schedule a maintenance window for Sunday at 2:00 AM and configure patching tasks to run during that period.

---

## 2. Why are Maintenance Windows useful?

They help organizations:

- Schedule maintenance during low-traffic periods.
- Coordinate patching and other operational tasks.
- Control which resources are targeted.
- Limit the time available for scheduled tasks.
- Reduce unexpected disruption to production workloads.

A maintenance window provides a controlled time period; it does not guarantee that every task will finish successfully within that period.

---

## 3. How do Maintenance Windows work?

``````text
1. Define the schedule
   For example, every Sunday at 2:00 AM
          |
          v
2. Select targets
   Choose managed nodes or other supported resources
          |
          v
3. Register tasks
   Specify the operations to run during the window
          |
          v
4. Execute and review
   Monitor task executions and their results
``````

---

## 4. Important concepts

- **Maintenance window schedule:** Defines when the window opens, using a supported scheduling expression and time zone configuration.
- **Duration:** The number of hours the maintenance window remains open.
- **Cutoff:** The point near the end of the window after which Systems Manager stops starting new tasks. Tasks already running may continue.
- **Targets:** The resources on which the maintenance tasks should operate.
- **Registered tasks:** The operations configured to run during the window.

Tasks may include supported Run Command operations, Automation workflows, patching, and other Systems Manager task types.

---

## 5. Real-world example

Imagine a company manages 80 EC2 instances and wants automatic patching every Sunday at 2:00 AM.

A possible configuration would be:

1. Create a maintenance window for Sunday at 2:00 AM.
2. Set an appropriate duration and cutoff.
3. Select the production instances using tags.
4. Register a patching task with an appropriate patch baseline and reboot policy.
5. Review execution results and patch compliance afterward.

> **Important:** You must configure the patching task separately; creating the maintenance window alone does not install patches.

---

## 6. Maintenance Windows vs. State Manager

| Feature | Maintenance Windows | State Manager |
|---------|---------------------|---------------|
| Primary purpose | Schedule maintenance tasks within defined windows | Apply and maintain configurations through associations |
| Typical use | Patch production servers at 2 AM | Run configuration instructions every 30 minutes |
| Scheduling focus | Controlled maintenance periods | Recurring configuration execution |
| Can work together? | Yes | Yes |

> **Interview tip:** If a question emphasizes a specific maintenance period, such as patching servers every Sunday at 2 AM, think of Maintenance Windows.
``````
