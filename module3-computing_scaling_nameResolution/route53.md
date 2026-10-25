# AWS Route 53: Complete Guide

Amazon Route 53 is an AWS service that provides DNS (Domain Name System), domain registration, traffic routing, and health checking. It helps users reach your application using a domain name instead of remembering an IP address.

---

## 1. What is DNS, and why do we need Route 53?

Imagine you deploy a web application on an EC2 instance. Your server has a public IP address, such as `54.12.34.56`. Users could visit that IP directly, but remembering IP addresses is inconvenient. Instead, you want users to visit:

```text
www.mywebsite.com
```

DNS connects that human-readable name to the destination your application uses.

```text
User's browser
(Enters www.mywebsite.com)
          |
          v
DNS query
          |
          v
Amazon Route 53
(Looks up the matching DNS record and returns the destination)
          |
          v
Web server / Load balancer
(Receives the browser's HTTP or HTTPS request)
```

> **Important distinction:** Route 53 resolves the domain name; it does not normally serve the website's content itself. The browser connects to the destination returned through DNS.

For example, if Route 53 directs `www.mywebsite.com` to an Application Load Balancer, the ALB receives the HTTP/HTTPS request and forwards it to the appropriate application targets.

---

## 2. How does DNS resolution work?

Suppose you type `www.example.com` into your browser.

1. **Browser checks its cache.** It may already know the IP address from a previous lookup.
2. **DNS resolver checks its cache.** If the answer is cached and its TTL has not expired, it can return the answer immediately.
3. **The resolver finds the authoritative DNS servers.** It follows the DNS delegation chain as needed, starting from the DNS root and top-level domain infrastructure.
4. **Route 53's authoritative name servers respond.** They return the DNS answer configured in the hosted zone, such as an IP address or an alias destination.
5. **The browser connects to the destination.** It sends the actual HTTP or HTTPS request to the server or service.

Route 53 is typically the authoritative DNS service in this example, while the recursive resolver is often operated by an ISP, public DNS provider, or organization.

---

## 3. The four main concepts of Route 53

- **Domain registration:** Register or manage domain names, such as `mywebsite.com`. You can also keep your domain registered with another registrar and use Route 53 only for DNS hosting.
- **DNS service:** Create records that determine where domain names should direct users, for example to an EC2 public IP address or an AWS load balancer.
- **Health checks:** Monitor supported endpoints and use health information to help route users away from unhealthy resources.
- **Route 53 Resolver:** Handles DNS resolution for VPCs and supports DNS forwarding between AWS networks and on-premises networks.

---

## 4. Hosted zones: public vs. private

A hosted zone is a container for DNS records associated with a domain. For example, a hosted zone named `example.com` can contain records for:

- `example.com`
- `www.example.com`
- `api.example.com`
- `dev.example.com`

Route 53 has two main types of hosted zones.

### Public hosted zone

Used to route DNS queries for resources accessible through public DNS.

**Example:** You host a website on an internet-facing Application Load Balancer and want people on the internet to access it using `www.example.com`.

### Private hosted zone

Used to resolve DNS names within the Amazon VPCs associated with the hosted zone.

**Example:** Your internal application is available at `api.internal.example.com`, and only resources with access to the relevant VPC DNS environment should resolve that private name.

### Real-world example

Suppose your architecture has three components:

| Component | DNS name | Intended audience |
|-----------|----------|-------------------|
| Public website | `www.example.com` | Internet users |
| Public API | `api.example.com` | Internet users, subject to application security |
| Internal database | `db.internal.example.com` | Authorized internal resources |

You could use a public hosted zone for the website and public API, and a private hosted zone for the internal database.

> **Remember:** A private hosted zone is not a replacement for security groups or other access controls. DNS resolution does not itself grant network access.

---

## 5. DNS records in Route 53

DNS records tell resolvers what information to return for a domain name. These are the important record types to know for interviews and practical AWS work.

| Record | Purpose | Example |
|--------|---------|---------|
| `A` | Maps a name to an IPv4 address | `example.com` → `203.0.113.10` |
| `AAAA` | Maps a name to an IPv6 address | `example.com` → an IPv6 address |
| `CNAME` | Points a name to another domain name | `www.example.com` → `app.example.net` |
| `MX` | Specifies mail servers | Routes email for a domain |
| `TXT` | Stores text data | Domain verification, SPF-related data |
| `NS` | Identifies authoritative name servers | Delegates DNS authority |
| `SRV` | Specifies service endpoints and ports | Some messaging and directory services |
| `CAA` | Specifies which certificate authorities may issue certificates for a domain | Restricts certificate issuance |

### What is the difference between A and CNAME?

Suppose your application has an IP address of `203.0.113.10`.

An **A record** could be:

| Field | Value |
|-------|-------|
| Record name | `app.example.com` |
| Record type | `A` |
| Value | `203.0.113.10` |

A **CNAME record** could instead be:

| Field | Value |
|-------|-------|
| Record name | `www.example.com` |
| Record type | `CNAME` |
| Value | `app.example.net` |

The A record returns an IPv4 address. The CNAME says that the queried name is an alias for another DNS name, which must then be resolved.

> **Important rule:** A standard CNAME cannot coexist with other record types at the same DNS name and cannot be used at the zone apex, such as `example.com`.

### What is an Alias record?

Route 53 supports a special Alias record that can point a DNS name to supported AWS resources, such as:

- Application and Network Load Balancers
- CloudFront distributions
- Certain Amazon S3 website endpoints
- Other supported AWS resources

An Alias record is not a separate DNS record type. It is a Route 53 feature used with supported record types, commonly `A` or `AAAA`.

```text
example.com → Alias A record → Application Load Balancer
```

Unlike a standard CNAME, an Alias record can be used at the zone apex. Route 53 also does not charge for Alias queries to supported AWS resources, although other Route 53 charges may apply.

> **Interview tip:** If an interviewer asks how to point a root domain to an ALB, an Alias A record is generally the expected answer.

---

## 6. Route 53 routing policies

This is one of the most important Route 53 topics. A routing policy determines which DNS answer Route 53 returns when users query a domain. Different policies are useful for traffic distribution, testing, geographic requirements, performance, and failover.

### 6.1 Simple routing
<img width="700" height="400" alt="image" src="https://github.com/user-attachments/assets/c7aec6c0-e1df-4031-a7e6-8ea5317915d6" />

Use this when a domain should direct traffic to one resource or when you want a straightforward DNS configuration.

```text
www.example.com → one ALB
```

You can also configure multiple values in a simple record set, but simple routing does not provide the weighted distribution or health-based routing behavior of the specialized policies.

**Use case:** A small website with a single application entry point.

### 6.2 Weighted routing
<img width="1858" height="1358" alt="image" src="https://github.com/user-attachments/assets/83a5e8f3-c23b-4db8-bbd3-0fab957977ac" />

Weighted routing distributes DNS responses according to configured weights. Imagine you have two application versions:

| Resource | Weight | Intended share |
|----------|--------|----------------|
| Version A | 80 | 80% |
| Version B | 20 | 20% |

The weights total 100, so the intended proportions are 80:20.

This is useful for:

- Canary deployments
- Gradually introducing a new version
- Testing a new application endpoint
- Distributing DNS responses among resources

> The percentages are approximate over a sufficiently large number of queries, not a guarantee for each individual user or HTTP request. DNS caching affects the actual traffic reaching each endpoint.

### 6.3 Latency-based routing

Suppose your application runs in two AWS Regions:

- US East (N. Virginia)
- Asia Pacific (Singapore)

Latency-based routing returns the endpoint associated with the AWS Region expected to provide lower latency for the DNS query's origin.

**Use case:** An application serving users globally where response time matters.

> This is based on AWS's latency measurements and DNS resolution context. It does not guarantee the physically closest server or the fastest response for every individual request.

### 6.4 Failover routing

Failover routing is used for an active-passive architecture.

```text
Primary endpoint
(Normally receives traffic)
          |
          v
Health check
(Evaluates endpoint health)
     |                |
     v                v
  Healthy          Unhealthy
  Continue         Use
  primary          secondary
```

**Example:**

- Primary: ALB in `us-east-1`
- Secondary: ALB in `ap-southeast-1`

When the primary is considered unhealthy, Route 53 can return the secondary endpoint.

**Use case:** Disaster recovery and high availability.

> **Critical detail:** DNS failover does not automatically repair or restart your application. It changes the DNS answer based on health information and configuration. Cached DNS answers can delay the effect for some users.

### 6.5 Geolocation routing

Geolocation routing directs users according to the geographic location of the DNS query source, such as a country or continent.

**Example:**

- Users in Nepal → Asia endpoint
- Users in the United States → US endpoint
- Users in Europe → Europe endpoint

You can also configure a default record for locations that do not match a more specific rule.

**Use case:** Delivering region-specific content, complying with certain geographic requirements, or directing users to region-specific experiences.

> Geolocation is based on location, not necessarily network latency.

### 6.6 Geoproximity routing

Geoproximity routing sends traffic to resources based on the geographic positions of the resources and the users. You can use a bias value to expand or shrink the area from which a resource attracts traffic.

For example, if you have servers in Singapore and Tokyo, you can adjust the bias to shift more traffic toward one location.

**Use case:** Fine-tuning geographic traffic distribution across several regions.

> Unlike geolocation routing, geoproximity routing is designed around resource proximity and adjustable bias.

### 6.7 Multivalue answer routing

Multivalue answer routing can return up to eight healthy record values selected at random for a query.

```text
api.example.com → multiple server IP addresses
```

Route 53 can exclude unhealthy resources when the records are configured with health checks.

**Use case:** Returning multiple healthy endpoints for a simple application.

> **Important:** Multivalue answer routing is not a replacement for a full load balancer. It does not guarantee that each individual request will be evenly distributed.

### 6.8 IP-based routing

IP-based routing directs DNS queries according to the IP address range of the resolver or query source, using configured CIDR collections and corresponding routing records.

**Use case:** Directing known network ranges, such as customers using particular corporate networks, to a designated endpoint.

It requires more specific IP-range configuration than geolocation routing.

### Routing policies at a glance

| Policy | Main decision | Typical use |
|--------|---------------|-------------|
| Simple | One basic destination configuration | Standard website |
| Weighted | Configured relative weights | Canary releases |
| Latency | Expected lower latency | Global applications |
| Failover | Primary or secondary health status | Disaster recovery |
| Geolocation | User's geographic location | Region-specific content |
| Geoproximity | Resource proximity and bias | Geographic traffic tuning |
| Multivalue answer | Multiple healthy answers | Multiple endpoints |
| IP-based | Source IP range mapping | Network-specific routing |

---

## 7. Health checks and DNS failover

A Route 53 health check monitors whether a configured endpoint is healthy. Depending on the configuration, Route 53 can monitor an endpoint, monitor another health check, or use a CloudWatch alarm's state.

### How health checks work

Suppose you have two web servers:

- Server A: primary
- Server B: backup

Route 53 health checks are configured for the primary endpoint. If the primary becomes unhealthy, a failover routing policy can return the backup endpoint instead.

Things to understand:

- **Endpoint monitoring:** Health checks can monitor supported IP addresses, domain names, and HTTP/HTTPS endpoints.
- **Health status:** Route 53 evaluates the health-check results against its configured criteria.
- **DNS response:** A failover policy uses health information when deciding which endpoint to return.
- **Recovery:** When the primary becomes healthy again, the DNS answer can return to the primary according to the failover configuration.

### Does a health check guarantee that the application works?

No. A server might respond successfully to an HTTP request while its database or another critical dependency is broken.

For better monitoring, configure a health-check endpoint that reflects the application's actual readiness, where appropriate. Also consider application monitoring with Amazon CloudWatch.

> **Interview question:** What is the difference between a health check and failover?
>
> A health check monitors an endpoint or another health signal. Failover is the routing behavior that directs DNS queries to an alternative endpoint when the configured conditions are met.

---

## 8. What is TTL in Route 53?

TTL stands for Time to Live. It specifies how long a DNS resolver can cache a record before it should query again for an updated answer.

For example, suppose this record exists:

| Setting | Value |
|---------|-------|
| Name | `www.example.com` |
| Type | `A` |
| Value | `203.0.113.10` |
| TTL | 300 seconds |

If a resolver caches this answer, it can reuse it for up to 300 seconds.

Now suppose you change the record to `203.0.113.20`. Resolvers that still have the old answer cached may continue using it until their cached TTL expires.

### TTL timeline

```text
0 sec            300 sec           New query
Old IP cached -> Cache expires -> Updated answer
```

### Should you use a high or low TTL?

- **Low TTL:** DNS changes can be reflected sooner, but resolvers may send more DNS queries.
- **High TTL:** Resolvers can reuse cached answers longer, reducing repeat queries, but changes may take longer to reach users.

> A low TTL does not mean every user instantly receives a new answer. DNS caching and application connections can affect the transition.

---

## 9. What is Route 53 Resolver?

Route 53 Resolver handles DNS resolution for Amazon VPCs and can connect DNS resolution between AWS and on-premises networks. It is different from the public authoritative DNS service used to answer queries for a public hosted zone.

Consider a company with:

- An AWS VPC running EC2 instances.
- An on-premises data center.
- Internal DNS names in both environments.

The company wants EC2 instances to resolve internal company names and on-premises systems to resolve private AWS names. Route 53 Resolver supports this through resolver rules and endpoints.

```text
On-premises network
(Corporate DNS servers)
          |
          v   DNS forwarding
Route 53 Resolver endpoints
(Inbound and outbound resolution)
          |
          v
Amazon VPC
(EC2 instances and private hosted zones)
```

### Inbound vs. outbound endpoints

| Endpoint | Direction | Purpose |
|----------|-----------|---------|
| Inbound | On-premises → AWS | On-premises DNS servers send queries into AWS |
| Outbound | AWS → on-premises | AWS forwards selected DNS queries to on-premises DNS servers |

For example, if an EC2 instance needs to resolve `server.corp.example`, an outbound forwarding rule can send the query to the corporate DNS server.

An inbound endpoint lets a corporate machine query private DNS names that AWS can resolve, provided the network connectivity and DNS configuration are in place.

---

## 10. Route 53 vs. ELB vs. CloudFront

These services often work together, but they do different jobs.

| Service | Primary responsibility |
|---------|------------------------|
| Route 53 | Resolves domain names and selects DNS destinations |
| Elastic Load Balancing | Distributes incoming application traffic across registered targets |
| CloudFront | Delivers cached content and application traffic through a global edge network |
| EC2 | Runs virtual machines and applications |
| S3 | Stores objects such as files, images, and static website assets |

### Example architecture

```text
User
(Visits www.example.com)
          |
          v
Route 53
(Resolves the domain to the configured destination)
          |
          v
Application Load Balancer
(Receives and distributes HTTP/HTTPS traffic)
     |                |
     v                v
   EC2 A            EC2 B
```

Here is the flow:

1. Route 53 resolves `www.example.com` to the ALB.
2. The browser establishes a connection to the ALB.
3. The ALB distributes the request to a healthy registered target.
4. The application processes the request and returns a response.

> Route 53 doesn't replace the ALB. It directs the user to the ALB, while the ALB handles application traffic distribution.

---

## 11. How to configure Route 53 for a website

Suppose you have an EC2 application or an ALB and own the domain `example.com`.

1. **Register or choose a domain.** You can register it through Route 53 or keep it at another registrar.
2. **Create a public hosted zone.** Name it `example.com`.
3. **Configure domain delegation.** If your domain is registered elsewhere, update its authoritative name-server settings at the registrar to match the Route 53 hosted zone.
4. **Create the DNS record.** For an ALB, create an Alias A record for `example.com` or `www.example.com` and select the ALB as the destination.
5. **Configure HTTPS separately.** Set up a certificate and HTTPS listener, commonly using AWS Certificate Manager with an ALB or CloudFront, as appropriate.
6. **Test the domain.** Verify DNS resolution, network access, listener configuration, and application health.

> **Common mistake:** Creating a hosted zone but forgetting to update the domain's delegation at the registrar. The records may exist in Route 53, but public DNS may still be using another provider's name servers.

---

## 12. Domain registration vs. DNS hosting

These two concepts are often confused in interviews.

- **Domain registrar:** The service through which you register ownership of a domain, such as `example.com`.
- **DNS hosting provider:** The service that stores and answers DNS records for that domain.

You can register a domain with one company and host its DNS records with Route 53. You do not have to transfer your domain registration to AWS.

For example:

1. Register `example.com` with another registrar.
2. Create a Route 53 public hosted zone for `example.com`.
3. Update the registrar's name-server delegation to Route 53.
4. Manage the domain's DNS records in Route 53.

---

## 13. Security and reliability features

- **DNSSEC:** Helps protect DNS integrity against certain DNS spoofing and manipulation attacks. Signing and validation must be configured appropriately.
- **Private hosted zones:** Support internal DNS naming without publishing those records through public authoritative DNS.
- **Resolver DNS Firewall:** Filters DNS queries made through Route 53 Resolver according to configured rules.
- **Health checks and failover:** Help direct DNS queries away from unhealthy endpoints.
- **Query logging:** Helps you analyze DNS queries for supported Route 53 DNS and Resolver configurations.

> These features solve different problems. For example, DNSSEC does not encrypt website traffic; HTTPS/TLS is used to protect the connection between a client and a website.

---

## 14. Route 53 pricing basics

Route 53 pricing can include several separate charges:

| Feature | Potential charge |
|---------|------------------|
| Hosted zones | Monthly charge per hosted zone |
| DNS queries | Charges depend on query type and volume |
| Domain registration | Registration and renewal fees vary by domain extension |
| Health checks | Charges depend on the type and configuration |
| Traffic Flow | Additional charges may apply |

Alias queries to supported AWS resources do not incur Route 53 query charges, but the destination service and other resources can still incur charges. Always check the official Route 53 pricing page for current rates before deploying.

---

## 15. AWS Route 53 interview questions

**1. What is Amazon Route 53?**
A highly available and scalable AWS DNS service that supports domain registration, DNS routing, health checking, and DNS resolution.

**2. What is a hosted zone?**
A container for DNS records belonging to a domain. Public hosted zones serve public DNS; private hosted zones resolve names within associated VPCs.

**3. What is the difference between a public and private hosted zone?**
A public hosted zone is used for public DNS resolution. A private hosted zone is used for private DNS resolution in associated VPC environments.

**4. What is the difference between A, CNAME, and Alias records?**
A maps a name to IPv4; CNAME maps a name to another DNS name; Alias is a Route 53 feature that points to supported AWS resources and can work at the zone apex.

**5. What is TTL?**
Time to Live: how long a DNS resolver may cache a record before it should query again for an updated answer.

**6. What is weighted routing used for?**
Distributing DNS responses according to configured weights, such as gradually introducing a new application version.

**7. What is latency-based routing?**
Selecting an endpoint based on the AWS Region expected to provide lower latency for the query's origin.

**8. What is failover routing?**
A routing policy for active-passive configurations that uses health information to select a primary or secondary endpoint.

**9. Does Route 53 replace an ALB?**
No. Route 53 resolves the DNS name to a destination; the ALB distributes application traffic across registered targets.

**10. What is Route 53 Resolver?**
A DNS resolution service for VPCs that can also forward DNS queries between AWS and on-premises networks through configured endpoints and rules.

---

## Final takeaway

If you remember just five things about Route 53, remember these:

1. **DNS:** Translates domain names into DNS answers that help clients locate destinations.
2. **Hosted zones:** Store DNS records for public or private DNS.
3. **Alias records:** Commonly point root domains to supported AWS resources such as ALBs.
4. **Routing policies:** Control DNS responses based on weights, latency, health, geography, proximity, or IP ranges.
5. **Health checks and failover:** Help route users away from unhealthy endpoints, but do not automatically repair the application.
