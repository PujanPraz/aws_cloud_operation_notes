# AWS Elastic Load Balancing (ELB)

Elastic Load Balancing (ELB) is an AWS service that automatically distributes incoming traffic across multiple targets, such as EC2 instances, containers, and IP addresses, to help your application remain available, scalable, and responsive.

---

## 1. Why do we need a load balancer?

Imagine you build a web application and deploy it on a single EC2 instance.

```text
1000 users
(Sending requests to your website)
          |
          v
One EC2 instance
(Handles every request)
          |
          v
Possible overload or single point of failure
```

You may encounter two problems:

- **Performance problem:** Too many requests can overwhelm the instance.
- **Availability problem:** If the instance fails, users may no longer be able to access your application.

You could launch multiple EC2 instances, but then you need a way to distribute incoming traffic among them. That's where ELB comes in.

---

## 2. How ELB works

Suppose you have three EC2 instances running the same application. ELB receives incoming requests and forwards them to appropriate healthy targets.

```text
Users / Clients
(Browser, mobile app, API client)
          |
          v
Elastic Load Balancer
(Receives traffic and selects healthy targets)
     |           |           |
     v           v           v
  EC2 A       EC2 B       EC2 C
 (Healthy)   (Healthy)   (Healthy)
```

> Conceptual illustration: the actual distribution algorithm depends on the load balancer type and its configuration.

Here's the basic flow:

1. A user sends an HTTP or HTTPS request to your application.
2. The request reaches the load balancer's DNS name.
3. ELB evaluates the listener rules and chooses a target according to the configured routing behavior.
4. ELB forwards the request to an eligible, healthy target.
5. The target processes the request and returns a response through the load balancer.

If an instance fails its configured health checks, ELB can stop sending it new requests and route traffic to other healthy targets.

> **Important:** ELB does not automatically create more EC2 instances. It distributes traffic across targets you provide. You can combine it with EC2 Auto Scaling to add or remove instances based on demand.

---

## 3. The four types of AWS load balancers

AWS ELB supports four types of load balancers: Application Load Balancer, Network Load Balancer, Gateway Load Balancer, and Classic Load Balancer. The first three are current-generation options; Classic Load Balancer is the older generation.

| Type | Layer | Best for |
|------|-------|----------|
| **Application Load Balancer (ALB)** | Layer 7 | Websites, REST APIs, and web applications using HTTP or HTTPS. Can route based on URL paths, hostnames, and HTTP headers. |
| **Network Load Balancer (NLB)** | Layer 4 | High-performance, low-latency network traffic using protocols such as TCP, UDP, and TLS. |
| **Gateway Load Balancer (GWLB)** | Layer 3/4 traffic handling | Deploying and scaling virtual network appliances such as firewalls and intrusion detection systems. |
| **Classic Load Balancer (CLB)** | Previous generation | Older EC2 load-balancing configurations. For new applications, AWS generally recommends ALB or NLB, depending on your needs. |

---

## 4. Application Load Balancer (ALB) in detail

ALB operates at Layer 7 (the application layer) of the OSI model. This means it understands HTTP and HTTPS request information, including the URL path and hostname, and can make routing decisions using that information.

### Example: one domain, multiple applications

Imagine you host an e-commerce application with these endpoints:

- `shop.example.com/products`
- `shop.example.com/orders`
- `admin.example.com`

You could use one ALB to route each request to a different target group.

```text
Application Load Balancer
(Examines HTTP request information)
     |             |              |
     v             v              v
 /products      /orders       admin host
 Product        Order         Admin
 service        service       service
```

ALB supports several important routing methods:

- **Path-based routing:** `/api/users` goes to the API service; `/images/*` goes to the image service.
- **Host-based routing:** `api.example.com` goes to one service; `admin.example.com` goes to another.
- **HTTP-header-based routing:** Requests with a specified header can be sent to a particular target group.
- **TLS termination:** ALB can decrypt HTTPS traffic at the load balancer, subject to listener configuration, so the application targets don't necessarily need to handle client-side TLS themselves.

### When would you choose ALB?

Choose ALB for web applications, websites, microservices, and HTTP/HTTPS APIs that need application-aware routing.

For example, if you're deploying a FastAPI backend with several EC2 instances, an ALB can distribute incoming HTTP requests across healthy instances.

---

## 5. Network Load Balancer (NLB) in detail

NLB operates at Layer 4 (the transport layer). It routes network connections using information such as IP addresses, ports, and transport protocols rather than HTTP URL paths. AWS designs it for high-performance workloads.

### Example: a TCP-based application

Suppose you operate a service that accepts TCP connections on port 9000. An NLB can distribute those connections across several EC2 instances running the service.

```text
Clients
(TCP connections to port 9000)
          |
          v
Network Load Balancer
(Selects a target using network-flow information)
     |                |
     v                v
  EC2 A            EC2 B
 TCP :9000        TCP :9000
```

Important NLB features include:

- Handles TCP, UDP, and TLS traffic, with additional supported protocol combinations depending on the configuration.
- Supports static IP addresses for the load balancer.
- Can preserve client source IP information in supported configurations.
- Supports targets such as EC2 instances and IP addresses.

### When would you choose NLB?

Choose NLB when you need high-throughput, low-latency network traffic handling, static IP addresses, or support for protocols that aren't HTTP-based.

> **Key difference from ALB:** ALB can route based on `/products` versus `/orders`. NLB doesn't inspect HTTP URL paths to make routing decisions.

---

## 6. Gateway Load Balancer (GWLB) in detail

GWLB serves a different purpose. Rather than primarily distributing ordinary web requests among application servers, it helps you deploy and scale virtual network appliances, such as security firewalls, intrusion detection systems, and traffic inspection appliances.

### Example: inspecting network traffic

Imagine your company wants traffic entering its environment to pass through a security appliance before reaching its application.

```text
Incoming network traffic
          |
          v
Gateway Load Balancer
(Distributes traffic to virtual appliances)
          |
          v
Security appliance fleet
(Firewall / intrusion detection / traffic inspection)
          |
          v
Destination application
```

GWLB uses the **GENEVE** protocol to encapsulate traffic between the load balancer and its appliances. The appliances inspect the traffic and process it according to their own security policies.

### When would you choose GWLB?

Choose GWLB when you need to scale and manage a fleet of network security or traffic-inspection appliances. It isn't the usual choice for routing website requests directly to application servers.

---

## 7. Classic Load Balancer (CLB) in detail

Classic Load Balancer is the older generation of AWS load balancer. It supports basic load balancing for EC2 instances at both the transport layer and application layer, depending on its listener configuration. AWS recommends migrating to current-generation load balancers for new features.

For example, an older application might use a CLB to distribute HTTP requests across multiple EC2 instances.

If you're starting a new project today, you would usually choose:

- **ALB** for HTTP/HTTPS web applications.
- **NLB** for high-performance TCP/UDP/TLS traffic.
- **GWLB** for virtual network appliances.

CLB is mainly relevant when maintaining or migrating legacy systems.

---

## 8. Quick comparison cheat sheet

| If the requirement is... | Choose |
|--------------------------|--------|
| Route by URL path, hostname, or HTTP header | ALB |
| Web app, REST API, or microservices over HTTP/HTTPS | ALB |
| Non-HTTP protocols (TCP/UDP/TLS), very low latency | NLB |
| Static IP addresses for the load balancer | NLB |
| Scale firewalls or traffic-inspection appliances | GWLB |
| Maintain or migrate a legacy setup | CLB |
