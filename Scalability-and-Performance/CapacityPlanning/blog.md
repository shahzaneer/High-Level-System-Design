# Capacity Planning

## Introduction
Capacity Planning is the process of determining the resources needed to meet future demand. It answers the question: "Will our system handle next month's traffic?" before that question becomes an outage. In traditional IT, capacity planning meant ordering servers 3-6 months in advance. In the cloud era, capacity planning shifted to cost optimization—you CAN add resources instantly, but SHOULD you reserve capacity at a discount, or pay on-demand at a premium?

The discipline combines historical trend analysis, growth forecasting, load testing, and cost modeling. Netflix's capacity planning ensures global streaming during peak hours (evenings, weekends, new season releases). E-commerce platforms plan for Black Friday 10x traffic spikes. Without capacity planning, systems fail under predictable load—predictable because the growth trend was visible months in advance.

## Definition

**Capacity Planning** is the process of forecasting future resource requirements and ensuring sufficient capacity is available to meet those requirements cost-effectively. It encompasses:

- **Demand Forecasting**: Predicting future workload based on historical trends, business growth, and known future events
- **Resource Modeling**: Translating workload predictions into infrastructure requirements (CPU cores, memory GB, storage TB, network bandwidth)
- **Provisioning Strategy**: Deciding how to acquire capacity (reserved instances, on-demand, spot, self-managed hardware)
- **Headroom Planning**: Maintaining buffer capacity for unexpected spikes (typically 20-30% above predicted peak)

## Concept Explanation

### The Capacity Planning Process

```python
class CapacityPlanner:
    def __init__(self, historical_data):
        self.data = historical_data
        self.growth_rates = self._calculate_growth_rates()
    
    def forecast_demand(self, months_ahead=3):
        """Predict demand based on historical trends"""
        forecasts = {}
        
        for metric in ['requests_per_second', 'storage_gb', 'daily_active_users']:
            # Calculate compound monthly growth rate
            cmgr = self.growth_rates[metric]
            
            current = self.data[metric][-1]
            forecast = current * (1 + cmgr) ** months_ahead
            
            # Add seasonal adjustment (e.g., December = 2x average)
            seasonal_factor = self._seasonal_factor(metric, months_ahead)
            forecast *= seasonal_factor
            
            # Add known events (product launch, marketing campaign)
            event_factor = self._event_factor(months_ahead)
            forecast *= event_factor
            
            forecasts[metric] = forecast
        
        return forecasts
    
    def translate_to_resources(self, demand_forecast):
        """Convert demand into infrastructure requirements"""
        resources = {
            'app_servers': demand_forecast['requests_per_second'] / RPS_PER_SERVER,
            'database_cpu': demand_forecast['requests_per_second'] * DB_CPU_PER_REQUEST,
            'cache_memory_gb': demand_forecast['daily_active_users'] * CACHE_PER_USER_MB / 1024,
            'storage_tb': demand_forecast['storage_gb'] / 1024,
            'network_bandwidth_gbps': demand_forecast['requests_per_second'] * AVG_RESPONSE_SIZE_BYTES * 8 / 1e9
        }
        return resources
    
    def recommend_provisioning(self, required_resources, headroom_pct=0.25):
        """Recommend provisioning with headroom"""
        return {
            'app_instances': ceil(required_resources['app_servers'] * (1 + headroom_pct)),
            'database_instance_class': self._recommend_db_instance(
                required_resources['database_cpu'] * (1 + headroom_pct)
            ),
            'cache_node_type': self._recommend_cache_node(
                required_resources['cache_memory_gb'] * (1 + headroom_pct)
            ),
            'storage_provisioned_tb': ceil(required_resources['storage_tb'] * (1 + headroom_pct))
        }
```

### Cost Model: Reserved vs On-Demand vs Spot

```python
class CostOptimizer:
    """
    Match workload predictability to pricing model.
    """
    
    def optimize(self, workload_profile):
        recommendations = {}
        
        # Baseline: always-on, predictable workload → Reserved Instances
        # 1-year commitment → 40% savings
        # 3-year commitment → 60% savings
        baseline = workload_profile.p99_load * 0.5  # 50% of peak
        recommendations['reserved'] = {
            'instances': ceil(baseline / INSTANCE_CAPACITY),
            'term': '3_year',
            'savings': '60%'
        }
        
        # Scaling layer: variable workload → On-Demand
        # Pay as you go for the difference between baseline and peak
        variable = workload_profile.p99_load - baseline
        recommendations['on_demand'] = {
            'instances': ceil(variable / INSTANCE_CAPACITY),
            'cost': 'full_price'
        }
        
        # Burst layer: non-critical, interruptible → Spot
        # 90% discount but can be terminated with 2-minute notice
        recommendations['spot'] = {
            'instances': 2,  # Small buffer for unexpected spikes
            'savings': '90% (but interruptible)'
        }
        
        return recommendations
```

### Load Testing: Verifying Capacity

```python
# k6 load test: verify system handles predicted load
# test.js
import http from 'k6/http';
import { sleep, check } from 'k6';

export const options = {
    stages: [
        { duration: '5m', target: 100 },   // Ramp up to 100 users
        { duration: '10m', target: 1000 },  // Ramp up to 1000
        { duration: '30m', target: 1000 },  // Stay at peak for 30 minutes
        { duration: '5m', target: 0 },      // Ramp down
    ],
    thresholds: {
        'http_req_duration': ['p(95)<500'],  // 95% of requests < 500ms
        'http_req_failed': ['rate<0.01'],    // Error rate < 1%
    },
};

export default function () {
    const response = http.get('https://api.example.com/orders');
    check(response, {
        'status is 200': (r) => r.status === 200,
    });
    sleep(1);
}
```

### Monitoring for Capacity

```python
# Key capacity metrics to monitor
class CapacityMetrics:
    
    @staticmethod
    def tracking_indicators():
        return {
            'cpu_headroom': 'Percentage of CPU remaining at peak',
            'memory_headroom': 'Available memory at peak',
            'db_connections_used': 'Database connections used vs max',
            'disk_io_utilization': 'Disk I/O % utilization',
            'network_throughput': 'Network bytes vs interface limit',
            'api_rate_limit_hits': 'How often rate limits are triggered',
            'request_queue_depth': 'Pending requests in load balancer queue',
        }
    
    @staticmethod
    def alert_thresholds():
        return {
            'cpu_headroom': {'warning': '< 30%', 'critical': '< 10%'},
            'db_connections': {'warning': '> 70%', 'critical': '> 90%'},
            'disk_space': {'warning': '< 100GB free', 'critical': '< 20GB free'},
        }
    
    @staticmethod
    def weekly_growth_rate(metric_name):
        """Is the growth rate accelerating?"""
        weekly_rates = [get_growth_rate(metric_name, week) for week in range(12)]
        # If growth is accelerating, capacity planning horizon shortens
        if weekly_rates[-1] > weekly_rates[-4] * 1.5:
            return "Growth accelerating—review capacity plan"
        return "Growth stable"
```

## Layman's Explanation

### The Restaurant Reservation System
A restaurant (your system) has 50 tables (server capacity). On a typical Tuesday, 30 tables are occupied. On Friday nights, 45 are occupied. The restaurant manager (capacity planner) needs to know:

- **Trend**: Table occupancy is growing 5% month-over-month. At this rate, the restaurant will be full on Fridays in 3 months.
- **Seasonal**: December holiday parties double demand. Need to handle 90 tables on Fridays in December (impossible with 50 tables).
- **Solution**: Reserve a private room (reserved instances) for weekends. Partner with a nearby restaurant for overflow (cloud bursting). Add a "waiting list" system (queue-based load leveling) for peak hours.

### The Water Tank
Your system is like a town's water supply. The reservoir (capacity) must handle:
- Average daily usage (baseline)
- Peak morning shower usage (daily peak)
- Summer lawn-watering season (seasonal peak)
- Fire hydrant usage (unexpected spike)

You build the reservoir to handle the expected peak + 25% buffer. You don't build it for "what if every resident fills an Olympic pool simultaneously"—that's infinite capacity at infinite cost.

## Why Solution Architects Must Acquire This

### Critical Design Decisions
- **Reserved Capacity vs Elastic**: Committing to 3-year Reserved Instances saves 60% but locks you into a capacity decision. If traffic drops (product decline), you're paying for unused capacity. If traffic soars (viral growth), Reserved capacity is insufficient. The split between Reserved (baseline) and On-Demand (elastic) is the key capacity planning decision.
- **Overhead Ratios**: Translate business metrics (daily active users) to technical metrics (requests per second, database connections). These ratios (requests per user per day, CPU per request) must be empirically measured—they change as features are added.
- **Lead Time for Capacity**: Cloud: minutes (on-demand), days (Reserved Instance purchases), months (new AWS region capacity, Direct Connect circuits). On-premises: months (procurement, shipping, racking). Consider lead time in the forecast horizon.

### Business Impact
- **Cost Avoidance**: Reserve 70% of baseline capacity at 60% discount. Over-provision by 30% on-demand. This balanced approach saves 40-50% vs 100% on-demand.
- **Outage Prevention**: Most outages are capacity-related—too many users, not enough resources. Capacity planning prevents the most predictable type of outage.
- **Budget Forecasting**: Finance needs cloud spend forecasts 6-12 months out. Capacity planning provides the resource model that drives cost forecasting.

## Summary

| Scenario | Strategy |
|----------|----------|
| Steady, predictable growth | Reserved Instances (1-3 year) for baseline |
| Daily/weekly patterns | Auto-scaling on schedule + metrics |
| Seasonal spikes (Black Friday) | Pre-warm capacity 24-48 hours before |
| Viral/unpredictable growth | Aggressive auto-scaling + pre-approved max limits |
| Multi-year planning | Reserved + Savings Plans for baseline, spot for burst |

Capacity planning ensures you have enough capacity before users notice you don't. It combines forecasting (what will demand be?), resource modeling (what does demand translate to?), provisioning (how do we acquire it?), and monitoring (are we tracking to plan?). In the cloud era, capacity planning is fundamentally cost optimization—the capacity is always available; the question is how much to reserve at a discount vs. pay for on-demand.
