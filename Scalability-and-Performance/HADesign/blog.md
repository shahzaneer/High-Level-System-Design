# High Availability Design

## Introduction
High Availability (HA) is the discipline of designing systems that remain operational despite component failures. The goal is expressed in "nines": 99.9% = 8.76h downtime/year, 99.99% = 52min/year, 99.999% = 5.26min/year. Each additional nine roughly doubles infrastructure cost—the HA target must be a business decision, not a vanity metric.

HA is achieved through redundancy, fault isolation, automated failover, and eliminating Single Points of Failure (SPOFs). It differs from DR: HA handles component failures transparently; DR handles site-level catastrophes.

## Definition

**High Availability** aims to ensure an agreed level of operational performance. Key metrics: Availability = Uptime/(Uptime+Downtime) = MTBF/(MTBF+MTTR). Improving MTTR (faster recovery) improves availability as much as improving MTBF (fewer failures).

## Concept Explanation

### Eliminating SPOFs

Walk through every component in the request path. If its failure brings down the entire system, it's a SPOF that needs redundancy:

| Component | SPOF? | Mitigation |
|-----------|-------|------------|
| Load Balancer | Yes (single LB) | LB pair active/passive or active/active |
| App Server | No (if multiple) | Auto-scaling group, multi-AZ |
| Database | Yes (single DB) | Multi-AZ, read replicas |
| DNS | Yes | Multiple DNS providers |
| Region | Yes | DR region |

### Redundancy Patterns

**Active/Passive**: One serves, one standby. Failover via heartbeat. Recovery: 10-60s. Cost: 2x.

**Active/Active**: All serve simultaneously. No failover needed—traffic redistributes around failed node. Recovery: instantaneous. Requires N+1 capacity.

**N+1 Rule**: N nodes needed + 1 spare. 3 nodes tolerate 1 failure. 5 nodes tolerate 2 failures.

### Quorum Systems

Why 3 nodes minimum? Two nodes during network partition = split-brain (both think they're leader). Three nodes need 2/3 majority for decisions. One isolated node cannot act alone (1/3 < majority). Always use odd numbers: 3, 5, 7.

### Multi-AZ Architecture

```hcl
resource "aws_subnet" "private" {
  count = 3
  availability_zone = data.aws_availability_zones.available.names[count.index]
}

resource "aws_db_instance" "main" {
  multi_az = true  # Synchronous standby, auto-failover 60-120s
}

resource "aws_lb" "app" {
  subnets = aws_subnet.public[*].id  # All AZs
}
```

### Health Checks

```yaml
livenessProbe:          # Is the process alive?
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 3   # Restart after 3 failures

readinessProbe:         # Is it ready for traffic?
  httpGet:
    path: /ready
    port: 8080
  failureThreshold: 2   # Remove from service after 2 failures
```

### Availability Math

```python
# Series: A * B * C (each dependency reduces availability)
# ALB(99.99%) * EC2(99.99%) * RDS(99.95%) = 99.93% total

# Parallel redundancy improves it
# 2 instances in parallel: 1 - (1-0.9999)^2 = 99.999999%
```

## Layman's Explanation

**Commercial Airline**: Two engines (N+1), three hydraulic systems, two pilots. No single failure brings down the plane. Result: 99.99998% availability (1 fatal accident per ~5M flights).

**The Nines Reality**: 99% = internal tools. 99.9% = B2B SaaS. 99.99% = e-commerce/customer-facing. 99.999% = telecom/critical infrastructure. Each nine costs 2-3x more.

## Why Solution Architects Must Acquire This

- SPOFs are easiest to eliminate during design, hardest after deployment
- Multi-AZ adds ~40% cost; multi-region adds ~100%. Choose based on cost of downtime
- Stateless design enables HA. Stateful services (databases) need complex HA mechanisms
- Automatic failover must be tested—false positives cause cascading failures

## On-Prem + Cloud Examples

```bash
# Keepalived: Virtual IP failover on-prem
# HAProxy: Load balancer pair

# AWS: Multi-AZ RDS, ALB across AZs, Auto Scaling Groups across AZs

# GCP: Regional MIGs, Cloud SQL HA, Regional GKE

# Azure: Availability Sets/Zones, Azure SQL failover groups, Traffic Manager
```

## Summary

| HA Level | Downtime/Year | Architecture | Cost |
|----------|--------------|-------------|------|
| 99.9% | 8.76 hours | Multi-AZ, single region | $$ |
| 99.99% | 52 minutes | Multi-AZ + DR region | $$$ |
| 99.999% | 5.26 minutes | Active/Active multi-region | $$$$ |

HA is about eliminating SPOFs through redundancy at every layer. The target availability should be set by the business cost of downtime—not engineering pride. Each additional nine doubles cost; the ROI must justify it.
