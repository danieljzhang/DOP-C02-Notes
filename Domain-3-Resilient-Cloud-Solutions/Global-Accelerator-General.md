# AWS Global Accelerator - DOP-C02 Exam Notes

## 1. Overview

**What it is:** A global networking service that routes user traffic through the AWS global network backbone to your application endpoints, improving availability and performance for global users.

**What problem it solves:** Public internet routing is unpredictable — packets take suboptimal paths, experience congestion, and suffer variable latency. Global Accelerator puts users onto the AWS backbone at the nearest edge location and keeps them there all the way to your endpoint, bypassing the public internet for most of the journey.

---

## 2. Core Concepts

### Anycast Static IP Addresses
Global Accelerator assigns **two static IPv4 addresses** (Anycast) to your accelerator. These IPs are advertised from all AWS edge locations simultaneously — a user's DNS resolves to the same two IPs regardless of their location, but their traffic is routed to the nearest edge location automatically.

**Exam angle:** "Static IPs that work globally without DNS changes" → Global Accelerator. This is the answer when a customer needs to whitelist IPs in a firewall and can't tolerate IP changes during failover.

### Listeners
A listener processes inbound connections on a port range and protocol (TCP or UDP). Each accelerator has one or more listeners.

### Endpoint Groups
One endpoint group per AWS Region. Each group has:
- A **traffic dial** (0–100%) — controls what percentage of traffic the listener sends to this region. Set to 0 to drain a region for maintenance without DNS changes.
- A **health check** configuration

### Endpoints
The actual targets within an endpoint group:
- Application Load Balancers (ALB)
- Network Load Balancers (NLB)
- EC2 instances
- Elastic IP addresses

Each endpoint has a **weight** (0–255) controlling its share of traffic within the group.

---

## 3. How Traffic Flows

```
User (anywhere in the world)
        ↓  DNS resolves to Anycast IP
AWS Edge Location (nearest to user)
        ↓  AWS global backbone (private, low-latency)
Endpoint Group (target AWS Region)
        ↓  traffic dial + health check
Endpoint (ALB / NLB / EC2 / EIP)
        ↓  weight
Application
```

The key: only the last hop (edge → endpoint) uses the public internet if the endpoint is public. The long-haul cross-continent leg stays on AWS's private network.

---

## 4. Global Accelerator vs Route 53 vs CloudFront

This comparison is heavily tested. Know when to use each:

| | Global Accelerator | Route 53 | CloudFront |
|---|---|---|---|
| **Primary use** | TCP/UDP performance + static IPs | DNS routing + health checks | Content delivery (caching) |
| **Caching** | No | No | Yes |
| **Static IPs** | Yes (2 Anycast IPs) | No (DNS names) | No |
| **Protocols** | TCP, UDP | DNS only | HTTP/HTTPS |
| **Failover speed** | ~30 seconds (no DNS TTL) | Depends on TTL (60s+) | N/A |
| **DDoS protection** | AWS Shield Standard (built-in) | Route 53 Resolver | AWS Shield Standard |
| **Use case** | Gaming, IoT, VoIP, APIs needing static IPs | Multi-region DNS failover | Static assets, video streaming |

**Key exam trap:** Route 53 failover depends on DNS TTL — clients may cache the old IP for minutes. Global Accelerator failover is near-instant because the Anycast IPs never change; only the routing behind them changes.

---

## 5. Traffic Dials and Weights — Deployment Patterns

### Blue/Green with Traffic Dials
```
Endpoint Group us-east-1 (blue):  traffic dial = 100%
Endpoint Group us-west-2 (green): traffic dial = 0%

→ Deploy new version to us-west-2
→ Set us-west-2 dial to 10%, validate
→ Gradually increase us-west-2, decrease us-east-1
→ Full cutover: us-east-1 = 0%, us-west-2 = 100%
```

### Canary with Endpoint Weights
Within a single region, send 5% of traffic to a new endpoint:
```
ALB-v1 weight = 95
ALB-v2 weight = 5
```

### Maintenance / Disaster Recovery
Set a region's traffic dial to 0 to drain it instantly — no DNS change, no TTL wait, no client impact beyond the in-flight connections.

---

## 6. Health Checks and Failover

- Global Accelerator performs its own health checks on endpoints (separate from ALB/NLB health checks)
- Unhealthy endpoints are automatically removed from routing
- If all endpoints in a region are unhealthy, traffic shifts to the next healthy region based on proximity
- Failover happens in ~30 seconds — much faster than DNS-based failover

---

## 7. Security

- **AWS Shield Standard** is included at no extra cost — protects the Anycast IPs from DDoS
- **AWS Shield Advanced** can be added for enhanced protection and cost protection
- Client IP preservation: for ALB and EC2 endpoints, the original client IP is preserved in the `X-Forwarded-For` header (ALB) or as the source IP (EC2)
- Supports VPC endpoints for private endpoint routing

---

## 8. Common Exam Scenarios

### Scenario 1: Gaming application needs low-latency global access with static IPs
**Solution:** Global Accelerator with UDP listener, EC2 or NLB endpoints in multiple regions. Static Anycast IPs let players whitelist IPs in their firewalls. AWS backbone reduces latency vs public internet routing.

### Scenario 2: Multi-region failover faster than Route 53
**Requirement:** RTO of under 1 minute for regional failover.
**Solution:** Global Accelerator — health checks detect failure in ~30s and reroute traffic without DNS TTL delays. Route 53 with a 60s TTL would take longer.

### Scenario 3: Zero-downtime regional maintenance
**Solution:** Set the maintenance region's traffic dial to 0%. All traffic shifts to other regions immediately. No DNS change, no client disruption. After maintenance, dial back up.

### Scenario 4: Canary deployment across regions
**Solution:** Deploy new version to a second region, set its traffic dial to 10%. Monitor error rates. Increase dial gradually. Full cutover by setting original region to 0%.

---

## 9. CLI Commands Reference

```bash
# Create accelerator
aws globalaccelerator create-accelerator \
  --name my-accelerator \
  --ip-address-type IPV4 \
  --enabled

# List accelerators (note: Global Accelerator is a global service, use us-west-2)
aws globalaccelerator list-accelerators \
  --region us-west-2

# Create listener
aws globalaccelerator create-listener \
  --accelerator-arn arn:aws:globalaccelerator::123456789012:accelerator/xxx \
  --protocol TCP \
  --port-ranges FromPort=80,ToPort=80 FromPort=443,ToPort=443

# Create endpoint group
aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::123456789012:accelerator/xxx/listener/xxx \
  --endpoint-group-region us-east-1 \
  --traffic-dial-percentage 100

# Update traffic dial (e.g., drain a region)
aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::123456789012:accelerator/xxx/listener/xxx/endpoint-group/xxx \
  --traffic-dial-percentage 0
```

> **Note:** Global Accelerator API calls must be made against the `us-west-2` region endpoint regardless of where your endpoints are.

---

## 10. Exam Tips

- **Two static Anycast IPs** — this is the unique differentiator vs Route 53 and CloudFront.
- **No caching** — Global Accelerator is a network-layer service, not a CDN. If the question mentions caching, the answer is CloudFront.
- **Faster failover than Route 53** — ~30 seconds vs DNS TTL-dependent.
- **Traffic dials** are on endpoint *groups* (per region); **weights** are on individual *endpoints* within a group.
- Global Accelerator API region is always `us-west-2` — a common CLI trap.
- Shield Standard is included — no extra cost for basic DDoS protection.
- Use case keywords: "static IPs", "global TCP/UDP", "sub-minute failover", "gaming", "IoT", "VoIP" → Global Accelerator.

---

*Added during the 2026-10 Amazon Q review. Facts cross-referenced against AWS documentation.*
