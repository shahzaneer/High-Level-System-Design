# Chaos Engineering

## Introduction
Chaos Engineering is the discipline of experimenting on a system to build confidence in its ability to withstand turbulent conditions in production. Netflix pioneered the practice in 2010 with Chaos Monkey—a tool that randomly terminated production instances to verify that the system gracefully handled failures. The premise is counterintuitive: deliberately break things in production to discover weaknesses before they cause real outages.

Chaos Engineering has evolved from simple instance termination to sophisticated experiments: network latency injection, DNS failures, database failover verification, region-wide outages, and application-level fault injection. It is now a mainstream practice at Google, Amazon, Microsoft, financial institutions, and any organization serious about resilience. It transforms resilience from a theoretical design property to an empirically verified capability.

## Definition

**Chaos Engineering** is the practice of thoughtfully and safely injecting failures into systems to identify weaknesses before they manifest as outages. It follows the scientific method:

1. **Define Steady State**: Establish measurable normal behavior (error rate, latency, throughput)
2. **Hypothesize**: Predict that the system will maintain steady state during the experiment
3. **Introduce Variables**: Inject real-world failure events (instance termination, network latency, resource exhaustion)
4. **Observe**: Measure the system's behavior against the steady state
5. **Analyze & Improve**: If steady state is disrupted, fix the weakness before it causes an outage

**Key Principle**: Start small (single instance), expand blast radius gradually, never experiment without the ability to abort, and always have a rollback plan.

## Concept Explanation

### The Chaos Engineering Maturity Model

```
Level 1: Ad Hoc
  - Occasional manual instance termination
  - No systematic approach
  - "Let's see what happens if we kill this server"

Level 2: Planned Experiments
  - Scheduled GameDays
  - Documented hypotheses and observations
  - Pre-defined abort conditions

Level 3: Automated Experiments
  - Continuous chaos experiments in staging
  - Automated analysis of results
  - Regression: did this experiment pass last time?

Level 4: Production Chaos (Advanced)
  - Automated experiments in production (during business hours)
  - Canary experiments (small % of traffic first)
  - Integrated with CI/CD pipeline
  - Chaos experiment gates deployment
```

### Experiment Design

```python
class ChaosExperiment:
    def __init__(self, name, hypothesis, steady_state_metrics, blast_radius):
        self.name = name
        self.hypothesis = hypothesis
        self.metrics = steady_state_metrics
        self.blast_radius = blast_radius
        self.abort_conditions = []
    
    def run(self):
        # 1. Verify steady state
        if not self._verify_steady_state():
            raise ExperimentAborted("Steady state not established")
        
        # 2. Inject failure
        self._inject_failure()
        
        # 3. Observe for duration
        start_time = time.time()
        while time.time() - start_time < self.duration:
            if self._should_abort():
                self._rollback()
                raise ExperimentAborted(f"Abort condition triggered: {self.abort_reason}")
            
            if self._steady_state_deviated():
                self._rollback()
                self._analyze_failure()
                return ExperimentResult.FAILED
            
            time.sleep(self.observation_interval)
        
        # 4. Rollback
        self._rollback()
        
        # 5. Verify steady state returns
        time.sleep(30)  # Stabilization period
        if self._verify_steady_state():
            return ExperimentResult.PASSED
        else:
            return ExperimentResult.FAILED
```

### Common Chaos Experiments

#### 1. Infrastructure-Level Experiments

```python
# Terminate random instance
def chaos_monkey():
    instances = ec2.describe_instances(
        Filters=[{'Name': 'tag:Chaos', 'Values': ['enabled']}]
    )
    victim = random.choice(get_running_instances(instances))
    ec2.terminate_instances(InstanceIds=[victim.id])

# AZ failure simulation
def az_failure(az_name):
    # Deny all traffic to/from specific AZ via NACLs
    nacl_id = get_nacl_for_az(az_name)
    ec2.create_network_acl_entry(
        NetworkAclId=nacl_id,
        RuleNumber=900,
        Protocol='-1',
        RuleAction='deny',
        CidrBlock='0.0.0.0/0',
        Egress=False
    )
```

#### 2. Network-Level Experiments

```python
# Inject latency: simulate cross-region latency within a region
tc qdisc add dev eth0 root netem delay 100ms 20ms distribution normal

# Packet loss: simulate unreliable network
tc qdisc add dev eth0 root netem loss 5% 25%

# DNS failure: simulate DNS resolution failure
iptables -A OUTPUT -p udp --dport 53 -j DROP
```

#### 3. Application-Level Experiments

```python
# Circuit breaker testing: make dependency fail
@chaos_experiment("payment-service-circuit-breaker")
def test_payment_failure():
    with chaos.inject_fault('payment-service', fault_type='timeout', duration=60):
        # All calls to payment service will timeout for 60 seconds
        # Verify: 
        # - Order service circuit breaker opens
        # - Fallback behavior works (show "payment pending")
        # - No cascading failures
        # - Circuit closes after payment service recovers

# Resource exhaustion
@chaos_experiment("memory-pressure")
def test_memory_pressure():
    import sys
    data = []
    while True:
        data.append(' ' * 1024 * 1024)  # Allocate 1MB
        if len(data) > 500:  # 500MB
            break
    # Verify: OOM killer behavior, graceful degradation, monitoring alerts
```

### Chaos Mesh (Kubernetes-Native)

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: order-service
  delay:
    latency: "100ms"
    jitter: "20ms"
    correlation: "50"
  duration: "5m"
  scheduler:
    cron: "@every 15m"
```

### GameDay Design

```
GameDay: "Region Failure Drill"
  Date: 2024-06-15, 10:00 AM - 12:00 PM
  Participants: SRE, Engineering, DBAs, Incident Commander
  
  Scenario: us-east-1 region becomes completely unavailable
  Steady State: 
    - Error rate < 0.1%
    - p99 latency < 500ms
    - All API endpoints responsive
  
  Hypothesis:
    - Traffic will route to eu-west-1 via Route 53
    - Aurora Global Database will promote eu-west-1 to primary
    - Maximum RTO: 5 minutes
    - Maximum RPO: < 1 second (Aurora global replication lag)
  
  Experiment Steps:
    1. 10:00 - Announce GameDay start
    2. 10:05 - Verify steady state in both regions
    3. 10:10 - Deny all traffic to us-east-1 via NACL modification
    4. 10:11 - Observe Route 53 health checks detect failure
    5. 10:12 - Observe DNS failover to eu-west-1
    6. 10:13 - Promote Aurora replica in eu-west-1
    7. 10:15 - Verify all services healthy in eu-west-1
    8. 10:30 - Attempt failback to us-east-1
    9. 11:00 - Post-mortem discussion
  
  Abort Conditions:
    - Error rate exceeds 5% for > 2 minutes
    - User-visible data loss detected
    - Incident Commander calls ABORT
  
  Rollback:
    - Revert NACL modifications
    - Failback database
    - Verify steady state
```

### Observability During Chaos

```python
class ChaosObserver:
    def __init__(self):
        self.metrics = {
            'error_rate': CloudWatchMetric('4xxErrorRate'),
            'latency_p99': CloudWatchMetric('TargetResponseTime', 'p99'),
            'throughput': CloudWatchMetric('RequestCount'),
            'circuit_breaker_state': AppMetric('circuit_breaker.*.state'),
            'db_connections': RDSMetric('DatabaseConnections')
        }
    
    def compare_steady_state(self, before_experiment, during_experiment):
        deviations = []
        
        for metric_name, data in self.metrics.items():
            before_value = before_experiment[metric_name]
            during_value = during_experiment[metric_name]
            
            deviation_pct = abs(during_value - before_value) / before_value * 100
            
            if deviation_pct > data.allowed_deviation:
                deviations.append({
                    'metric': metric_name,
                    'before': before_value,
                    'during': during_value,
                    'deviation_pct': deviation_pct
                })
        
        return deviations
```

## Layman's Explanation

### The Fire Drill
Chaos Engineering is exactly like a fire drill for a building:

- **Steady State**: On a normal day, 500 employees work in the building. The elevators work. The power is on.
- **Hypothesis**: "If the fire alarm goes off, all 500 employees will evacuate within 5 minutes through the designated exits."
- **Experiment**: Pull the fire alarm (on a scheduled day, with everyone notified it's a drill).
- **Observe**: Did everyone get out in 5 minutes? Did anyone use the blocked east stairwell (unexpected behavior)? Did the fire doors work? Did the PA system function?
- **Findings**: Stairwell B got congested because 300 people tried to use it. The PA speaker on the 3rd floor didn't work. Two exits were blocked by renovation equipment.
- **Fix**: Add signage directing people to stairwell A. Fix the speaker. Clear the exits.
- **Next Drill**: Verify the fixes worked.

The alternative is waiting for a real fire to discover these problems—which is a terrible time to learn your exits are blocked.

### Why in Production?
Netflix's famous philosophy: "We inject failures in production during business hours because that's when our engineers are awake, monitoring systems are fully staffed, and we can learn the most. Waiting until 3 AM when a real failure happens means the on-call engineer is alone, tired, and facing an unknown scenario."

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Blast Radius Control**: Every chaos experiment must have a defined blast radius. Start with a single pod → single service → single AZ → single region. Never experiment on something without the ability to roll back.
- **Observability Prerequisites**: You cannot run chaos engineering without comprehensive monitoring. If you can't measure the steady state, you can't determine if the experiment succeeded or failed. Observability must come first.
- **Application Design for Chaos**: Chaos-ready applications implement: circuit breakers (fail fast), retries with backoff (handle transient failures), idempotency (safe to retry), timeouts (don't hang forever), and graceful degradation (serve partial results).
- **Progressive Delivery Integration**: Chaos experiments as deployment gates. Before promoting a canary deployment from 5% to 100%, inject chaos into the canary. If it survives, the new version is more resilient than the old.

### Business Impact
- **MTTD/MTTR Reduction**: Chaos-engineering-mature organizations detect and recover from real incidents 80% faster. Engineers recognize failure patterns because they've practiced them.
- **Confidence in DR**: Most organizations discover DR doesn't work during an actual disaster. Chaos engineering verifies DR through controlled experiments, giving leadership confidence in business continuity plans.
- **Cost of Failure Reduction**: Finding a single-point-of-failure through a planned chaos experiment costs a few engineer-hours. Finding it through a production outage costs revenue, reputation, and possibly regulatory fines.

## On-Premises Examples

### Chaos Toolkit (Open-Source)
```json
{
  "version": "1.0.0",
  "title": "What happens if we terminate a database node?",
  "description": "Verify application handles database failover gracefully",
  "steady-state-hypothesis": {
    "title": "Application responds normally",
    "probes": [
      {
        "type": "probe",
        "name": "app-responds",
        "tolerance": {"type": "regex", "pattern": "200"},
        "provider": {
          "type": "http",
          "url": "https://app.example.com/health",
          "timeout": 5
        }
      }
    ]
  },
  "method": [
    {
      "type": "action",
      "name": "terminate-db-instance",
      "provider": {
        "type": "process",
        "path": "aws",
        "arguments": ["rds", "reboot-db-instance", "--db-instance-identifier", "orders-db", "--force-failover"]
      }
    },
    {
      "type": "probe",
      "name": "db-replica-promoted",
      "tolerance": {"type": "regex", "pattern": "available"},
      "provider": {
        "type": "process",
        "path": "aws",
        "arguments": ["rds", "wait", "db-instance-available", "--db-instance-identifier", "orders-db"]
      }
    }
  ],
  "rollbacks": []
}
```

### LitmusChaos (Kubernetes)
```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: nginx-chaos
spec:
  appinfo:
    appns: 'production'
    applabel: 'app=order-service'
    appkind: 'deployment'
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '60'
            - name: CHAOS_INTERVAL
              value: '10'
            - name: FORCE
              value: 'false'
```

## AWS Examples

### AWS Fault Injection Simulator (FIS)
```hcl
resource "aws_fis_experiment_template" "az_failure" {
  description = "Simulate AZ failure for order service"
  role_arn    = aws_iam_role.fis.arn

  stop_condition {
    source = "none"
  }

  action {
    name      = "disrupt-network"
    action_id = "aws:network:disrupt-connectivity"

    target {
      key   = "Subnets"
      value = "az-a-subnet"
    }

    parameter {
      key   = "duration"
      value = "PT5M"  # 5 minutes (ISO 8601 duration)
    }
  }

  action {
    name       = "terminate-instances"
    action_id  = "aws:ec2:terminate-instances"

    target {
      key   = "Instances"
      value = "random-instance"
    }
  }

  target {
    name           = "az-a-subnet"
    resource_type  = "aws:ec2:subnet"
    resource_tags  = {
      AZ = "us-east-1a"
    }
    selection_mode = "ALL"
  }

  target {
    name           = "random-instance"
    resource_type  = "aws:ec2:instance"
    resource_tags  = {
      Chaos = "enabled"
    }
    selection_mode = "COUNT(1)"
  }
}
```

```bash
# Run experiment
aws fis start-experiment \
  --experiment-template-id "EXTxxxxxxxxx" \
  --tags Name=az-failure-drill,Date=2024-06-15
```

### AWS Resilience Hub
```bash
# Assess application resilience
aws resiliencehub create-app \
  --name "OrderProcessing" \
  --assessment-schedule "Daily"

# Resilience Hub analyzes:
# - Single AZ failure impact
# - Regional failure recovery
# - RTO/RPO estimates
# - Recommendation to improve resilience score
```

## GCP Examples

### GCP Fault Injection
```bash
# Network chaos: inject latency to Cloud SQL
gcloud compute instances add-metadata app-instance-1 \
  --metadata=startup-script='
    tc qdisc add dev eth0 root netem delay 100ms
  '

# Instance chaos: simulate preemption (Spot VM)
gcloud compute instances simulate-maintenance-event app-instance-1
```

### Cloud Monitoring SLO-Based Alerts
```bash
gcloud monitoring service-level-objectives create \
  --service=order-service \
  --slo-id=availability \
  --goal=99.9 \
  --calendar-period=month

# If chaos experiment drops availability below 99.9%, alert fires
# But within SLO—business as usual, no alert needed
```

## Azure Examples

### Azure Chaos Studio
```bash
# Create chaos experiment
az chaos experiment create \
  --name "az-failure" \
  --resource-group myResourceGroup \
  --location eastus \
  --selectors '[{"type":"List","id":"app-vms","targets":[{"id":"/subscriptions/.../virtualMachines/app-vm-1","type":"Microsoft-VirtualMachine"}]}]' \
  --steps '[{"name":"step1","branches":[{"name":"branch1","actions":[{"type":"continuous","name":"shutdown","duration":"PT5M","parameters":[{"key":"abruptShutdown","value":"true"}],"selectorId":"app-vms","urn":"urn:csci:microsoft:virtualMachine:shutdown/1.0"}]}]}]'
```

### Azure Load Testing
```bash
az load test create \
  --name order-service-load-test \
  --resource-group myResourceGroup \
  --location eastus

# Run load test during chaos experiment
az load test-run create \
  --load-test-resource order-service-load-test \
  --test-id chaos-baseline \
  --display-name "Chaos Experiment Baseline"
```

## Summary Decision Matrix

| Chaos Tool | Best For | Complexity | Production Ready |
|-----------|----------|------------|-----------------|
| AWS FIS | AWS-native experiments | Medium | Yes |
| Chaos Mesh | Kubernetes environments | Medium | Yes |
| LitmusChaos | Kubernetes, GitOps-native | Medium | Yes |
| Gremlin | Enterprise, multi-cloud | Medium | Yes |
| Chaos Toolkit | Declarative, cloud-agnostic | Low-Medium | Yes |
| Custom scripts + cron | Simple experiments | Low | With care |

Chaos engineering is the only way to empirically verify resilience. A system that hasn't been tested with real failures is a system whose resilience is assumed, not proven. Start small—a single pod termination in staging, observed carefully. Progress to automated experiments in staging. Graduate to controlled GameDays in production. The goal is not to cause outages but to discover the conditions that would cause outages, and fix them before they manifest. Every chaos experiment that passes increases confidence; every one that fails prevents a future outage.
