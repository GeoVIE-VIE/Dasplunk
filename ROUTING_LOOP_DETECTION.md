# Routing Loop Detection Guide

## Overview

This document explains the routing loop detection capabilities added to your Splunk dashboard. Routing loops are critical network issues that can cause service degradation, packet loss, and complete network outages.

## What's Been Added

The dashboard now includes a comprehensive **Routing Loop Detection** section with the following panels:

### 1. KPI Dashboard (4 Single-Value Metrics)

#### **Routing Loop Risk Score (0-100)**
- **Purpose**: Aggregated risk metric based on flapping adjacencies and affected devices
- **Formula**: `(high_risk_flaps × 5) + (affected_devices × 10)`, capped at 100
- **Color Coding**:
  - 🟢 Green (0-20): Healthy routing infrastructure
  - 🟡 Yellow (21-40): Minor routing instability
  - 🟠 Orange (41-60): Moderate risk, investigate
  - 🔴 Red (61-100): **CRITICAL** - Potential routing loop

#### **Routing Adjacency Flaps**
- **Purpose**: Count of neighbor relationships experiencing rapid state changes
- **Detection Logic**: Identifies device-neighbor pairs with more than 3 state changes in a 10-event window
- **Threshold**: >5 flapping adjacencies = Critical

#### **Loop Error Detections**
- **Purpose**: Count of explicit loop-related error messages
- **Detects**: Keywords like "routing loop", "TTL exceeded", "redistribution loop", "SIA", etc.
- **Color Coding**:
  - 🟢 Green (0): No loop errors
  - 🟠 Orange (1-9): Minor loop detection
  - 🔴 Red (10+): Multiple loop errors detected

#### **Routing Event Velocity**
- **Purpose**: Rate of routing protocol events per minute
- **High Values Indicate**: Routing instability, possible convergence issues
- **Thresholds**:
  - 🟢 0-5 events/min: Normal
  - 🟡 5-10 events/min: Elevated
  - 🟠 10-20 events/min: High
  - 🔴 20+ events/min: **CRITICAL**

---

## 2. Detailed Analysis Panels

### **Routing Adjacency Flapping Detection**
**Table showing potential loop indicators with:**

| Column | Description |
|--------|-------------|
| Protocol | BGP, OSPF, EIGRP, IS-IS, or RIP |
| Device | Router/switch experiencing flaps |
| Neighbor | Peer IP address |
| Interface | Local interface to neighbor |
| Flaps | Total number of state changes |
| Duration (min) | Time span of flapping |
| Flaps/Min | Flap rate (higher = worse) |
| Down Count | Number of times adjacency went down |
| State Changes | Distinct states observed |
| Risk Level | CRITICAL / HIGH / MEDIUM / LOW |
| First Detected | Timestamp of first flap |
| Last Detected | Timestamp of last flap |

**Risk Levels:**
- **CRITICAL**: ≥20 flaps AND ≥2 flaps/min
- **HIGH**: ≥10 flaps AND ≥1 flap/min
- **MEDIUM**: ≥5 flaps
- **LOW**: <5 flaps

### **Routing Protocol Event Rate Timeline**
- **Chart Type**: Line chart (5-minute buckets)
- **Purpose**: Visual anomaly detection - spikes indicate routing instability
- **Usage**: Look for sudden increases or oscillating patterns

### **Devices with Highest Routing Instability**
- **Chart Type**: Bar chart
- **Metric**: Instability Score = (Down Events / Total Events) × 100
- **Interpretation**: Higher percentages indicate devices with frequent routing failures

### **Explicit Loop Error Messages**
**Table capturing vendor-specific loop detection messages:**

| Loop Type | Cause |
|-----------|-------|
| Redistribution Loop | Routes redistributed between protocols causing feedback |
| EIGRP SIA | Stuck-in-Active - query responses failing |
| TTL Exceeded | Packets circulating until TTL reaches zero |
| BGP Path Loop | AS path contains loops |
| Hop/Metric Limit | Route exceeded maximum hop count |
| Martian Address | Invalid/unreachable addresses in routing table |
| Recursive/Forwarding Loop | Next-hop resolution loops |

### **Correlated Routing and Interface Instability**
- **Purpose**: Find routing events happening simultaneously with interface flaps
- **High Correlation**: Strong indicator of physical or Layer 2 issues causing routing loops
- **Risk Levels**:
  - **High Risk**: >5 correlated events in 5-minute window
  - **Medium Risk**: 3-5 correlated events
  - **Low Risk**: 2 correlated events

---

## How Routing Loops Manifest in Logs

### 1. **BGP Routing Loops**
```
%BGP-5-ADJCHANGE: neighbor 10.1.1.1 Down
%BGP-5-ADJCHANGE: neighbor 10.1.1.1 Up
%BGP-5-ADJCHANGE: neighbor 10.1.1.1 Down
```
**Cause**: Session flapping due to route oscillation, misconfigured route-maps, or path selection issues.

### 2. **OSPF Routing Loops**
```
%OSPF-5-ADJCHG: Process 100, Nbr 10.2.2.2 on GigabitEthernet0/1 from FULL to DOWN
%OSPF-5-ADJCHG: Process 100, Nbr 10.2.2.2 on GigabitEthernet0/1 from DOWN to FULL
```
**Cause**: Area misconfigurations, incorrect network types, or summarization issues.

### 3. **EIGRP Stuck-in-Active (SIA)**
```
%DUAL-3-SIA: Route 192.168.1.0/24 stuck-in-active state in IP-EIGRP 100
```
**Cause**: Routing loops preventing query responses from reaching the querying router.

### 4. **Redistribution Loops**
```
%ROUTING-4-LOOP: Redistribution loop detected between OSPF 1 and BGP 65000
```
**Cause**: Routes redistributed between protocols without proper filtering.

### 5. **TTL Exceeded**
```
%IP-3-TTL_EXCEEDED: TTL exceeded for packet to 10.5.5.5
```
**Cause**: Packet circulating through routers in a loop until TTL reaches zero.

---

## Detection Strategies

### 1. **Flapping Detection (streamstats)**
The queries use Splunk's `streamstats` with a sliding window to detect rapid state changes:
```spl
| streamstats window=20 count as state_changes by host neighbor_ip routing_int
| where state_changes > 5
```
This identifies adjacencies that change state more than 5 times in a 20-event window.

### 2. **Pattern Matching (regex)**
Extracts routing protocol details:
- Neighbor IPs
- Interface names
- State transitions (Up/Down/Full/Established/Idle)
- Protocol type (BGP/OSPF/EIGRP/IS-IS/RIP)

### 3. **Correlation Analysis**
Identifies events that are BOTH routing protocol events AND interface events, indicating physical layer involvement.

### 4. **Explicit Error Matching**
Searches for vendor-specific loop detection messages using keyword patterns.

---

## Tuning Thresholds

### Adjust Flap Detection Sensitivity

**Default**: 5 flaps in a 20-event window
```spl
| streamstats window=20 count as state_changes by host neighbor_ip routing_int
| where state_changes > 5
```

**More Sensitive** (lower threshold):
```spl
| streamstats window=10 count as state_changes by host neighbor_ip routing_int
| where state_changes > 3
```

**Less Sensitive** (higher threshold):
```spl
| streamstats window=30 count as state_changes by host neighbor_ip routing_int
| where state_changes > 10
```

### Adjust Risk Score Calculation

**Default**: `(high_risk_flaps × 5) + (affected_devices × 10)`

To weight device count more heavily:
```spl
| eval loop_risk_score=round(min(100, (high_risk_flaps * 3) + (affected_devices * 15)), 0)
```

### Adjust Time Buckets

**Event Rate Timeline** (default: 5-minute):
```spl
| timechart span=5m count by routing_protocol
```

For higher resolution (1-minute):
```spl
| timechart span=1m count by routing_protocol
```

For longer trends (30-minute):
```spl
| timechart span=30m count by routing_protocol
```

---

## Alerting Recommendations

### Critical Alert
```spl
index=network_syslog
| eval routing_protocol=case(match(_raw,"(?i)BGP"),"BGP", ...)
| search routing_protocol=*
| streamstats window=10 count as state_changes by host neighbor_ip routing_int
| where state_changes > 10
| stats count as critical_flaps
| where critical_flaps > 5
```
**Trigger**: When more than 5 adjacencies have >10 flaps in 10-event window
**Action**: Page NOC immediately

### Warning Alert
```spl
| search ("routing loop" OR "loop detected" OR "redistribution loop")
| stats count as loop_errors
| where loop_errors > 0
```
**Trigger**: Any explicit loop error message
**Action**: Email network team

### Anomaly Alert
```spl
| eval routing_protocol=case(match(_raw,"(?i)BGP"),"BGP", ...)
| search routing_protocol=*
| stats count as total_routing_events earliest(_time) as earliest_time latest(_time) as latest_time
| eval time_span_min = (latest_time - earliest_time) / 60
| eval events_per_min = if(time_span_min > 0, round(total_routing_events / time_span_min, 2), total_routing_events)
| where events_per_min > 20
```
**Trigger**: Routing event rate exceeds 20/minute
**Action**: Notify network team

---

## Troubleshooting Workflow

### When Loop Risk Score is High:

1. **Check Flapping Adjacencies Table**
   - Identify which device and neighbor are affected
   - Note the protocol (BGP/OSPF/EIGRP)
   - Check flap rate

2. **Review Event Rate Timeline**
   - Look for event spikes
   - Correlate with change windows

3. **Examine Explicit Loop Errors**
   - Check for redistribution loops
   - Look for SIA conditions (EIGRP)
   - Review TTL exceeded messages

4. **Check Correlated Instability**
   - If high correlation with interface events: Physical layer issue
   - If only routing events: Configuration issue

5. **Review Live Syslog Stream**
   - Filter by affected device: `host=<device_name>`
   - Look for recent configuration changes
   - Check for authentication failures

---

## Common Root Causes

### Configuration Issues
- **Incorrect route redistribution**: Missing route-maps or distribute-lists
- **Area misconfigurations**: OSPF area boundaries incorrect
- **AS path manipulation**: BGP prepending causing suboptimal paths
- **Summarization issues**: Discard routes not configured

### Physical Layer Issues
- **Interface flapping**: Bad cables, transceivers, or port issues
- **Duplex mismatches**: Half/full duplex misconfigurations
- **CRC errors**: Layer 1 problems causing packet loss

### Design Issues
- **Insufficient redundancy**: Single points of failure
- **Topology loops**: Layer 2 loops affecting Layer 3
- **Inadequate metrics**: Equal-cost paths causing route flapping

### Software Issues
- **IOS bugs**: Known issues in specific software versions
- **Resource exhaustion**: CPU/memory issues preventing convergence
- **Process crashes**: Routing protocol daemons restarting

---

## Best Practices

1. **Baseline Your Network**
   - Establish normal routing event rates
   - Document expected adjacency counts
   - Record typical convergence times

2. **Use Change Control**
   - Correlate routing events with change windows
   - Test routing changes in lab first
   - Have rollback procedures ready

3. **Implement Route Filtering**
   - Use prefix-lists and route-maps
   - Implement route dampening for BGP
   - Configure summarization at area boundaries

4. **Monitor Proactively**
   - Set up alerts for flapping adjacencies
   - Track routing event velocity trends
   - Review loop risk scores daily

5. **Document Topology**
   - Maintain network diagrams
   - Document routing policies
   - Keep vendor contact info handy

---

## Query Performance Tips

The routing loop detection queries can be resource-intensive. Optimize performance by:

1. **Use Time Restrictions**
   ```spl
   earliest=-1h latest=now
   ```

2. **Pre-Filter at Search Time**
   ```spl
   index=network_syslog (BGP OR OSPF OR EIGRP)
   ```

3. **Use Summary Indexing**
   - Schedule searches to run every 5 minutes
   - Store results in summary index
   - Dashboard references summary data

4. **Limit Result Sets**
   ```spl
   | head 25
   ```

---

## Additional Keywords for Detection

You can extend the loop detection by adding these patterns:

### Cisco-Specific
- `ROUTING_LOOP_DETECT`
- `INVALID_NEXT_HOP`
- `RECURSIVE_ROUTE_LIMIT`
- `MAX_REDISTRIBUTION_DEPTH`

### BGP-Specific
- `AS_PATH_LOOP`
- `CLUSTER_LIST_LOOP`
- `ORIGINATOR_ID_LOOP`

### EIGRP-Specific
- `DUAL-3-SIA`
- `DUAL-5-NBRCHANGE`

### OSPF-Specific
- `OSPF-4-ERRRCV`
- `OSPF-4-INVALID_METRIC`

---

## Support

For questions or issues with routing loop detection:
1. Review this documentation
2. Check the Live Syslog Stream for recent events
3. Adjust thresholds based on your network baseline
4. Consider engaging network engineering for analysis

---

## Version History

- **v1.0** (2025-12-04): Initial routing loop detection implementation
  - Added 4 KPI metrics
  - Added flapping detection table
  - Added event rate timeline
  - Added explicit error message detection
  - Added correlated instability analysis
