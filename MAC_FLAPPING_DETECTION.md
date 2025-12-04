# MAC Address Flapping Detection Guide

## Overview

MAC address flapping occurs when a MAC address rapidly moves between different switch ports, typically indicating **Layer 2 loops** or network topology issues. This dashboard section provides comprehensive detection and analysis of MAC flapping events.

---

## What's Been Added

### **New Section: "MAC Address Flapping Detection"**
Located between Spanning Tree Events and Routing Protocol Events, featuring orange-themed styling to distinguish it from routing loop detection (red) and general events (blue/purple).

---

## Dashboard Components

### **1. KPI Dashboard (4 Metrics)**

#### **MAC Flap Risk Score (0-100)**
Aggregated risk metric based on:
- `(total_flaps × 3) + (unique_macs × 8) + (affected_devices × 10)`

**Color Coding:**
- 🟢 Green (0-20): Normal
- 🟡 Yellow (21-40): Minor flapping
- 🟠 Orange (41-60): Moderate risk - investigate
- 🔴 Red (61-100): **CRITICAL** - Layer 2 loop likely

#### **Flapping MAC Addresses**
Count of unique MAC addresses experiencing flapping.

**Thresholds:**
- 🟢 0-2 MACs: Normal
- 🟡 3-9 MACs: Elevated
- 🔴 10+ MACs: **CRITICAL**

#### **Affected VLANs**
Number of VLANs experiencing MAC flapping.

**Thresholds:**
- 🟢 0-1 VLAN: Isolated issue
- 🟡 2-4 VLANs: Multiple broadcast domains affected
- 🔴 5+ VLANs: **WIDESPREAD** loop or misconfiguration

#### **Loop Detection Events**
Count of explicit loop detection triggers (loop guard, err-disable, etc.).

**Thresholds:**
- 🟢 0: No loop detection
- 🟠 1-4: Some loops detected
- 🔴 5+: **MULTIPLE** loop detection events

---

## 2. Detailed Analysis Panels

### **MAC Address Flapping Table**

Shows individual MAC addresses experiencing flapping:

| Column | Description |
|--------|-------------|
| Device | Switch reporting the flapping |
| MAC Address | MAC experiencing instability |
| VLAN | VLAN ID where flapping occurs |
| Flaps | Total number of flap events |
| Duration (min) | Time span of flapping |
| Flaps/Min | Flap rate (higher = worse) |
| Source Ports | Ports MAC moved from |
| Destination Ports | Ports MAC moved to |
| Port Count | Number of unique ports involved |
| Risk Level | CRITICAL / HIGH / MEDIUM / LOW |
| First Detected | Timestamp of first flap |
| Last Detected | Timestamp of most recent flap |

**Risk Levels:**
- **CRITICAL**: ≥20 flaps AND ≥2 flaps/min
- **HIGH**: ≥10 flaps AND ≥1 flap/min
- **MEDIUM**: ≥5 flaps
- **LOW**: <5 flaps

---

### **MAC Flapping by VLAN**

Bar chart showing:
- **Unique MACs**: Number of different MAC addresses flapping per VLAN
- **Total Events**: Total flap event count per VLAN

**Usage:** Identifies which VLANs are most affected - helps scope the problem.

---

### **Loop Detection Timeline**

Line chart (30-minute buckets) showing:
- **MAC Flapping** events (orange line)
- **Loop Detection** events (red line)

**Usage:** Correlate MAC flapping with loop guard triggers to confirm loops.

---

### **Port Pair Analysis (Loop Candidates)**

Table showing port pairs with MAC movement:

| Column | Description |
|--------|-------------|
| Device | Switch with the port pair |
| Port Pair | Ports experiencing MAC movement (e.g., "Gi0/1 <-> Gi0/2") |
| Flap Count | Number of times MACs moved between these ports |
| VLANs Affected | Number of VLANs seeing flapping on this port pair |
| VLAN List | Specific VLANs affected |
| Loop Risk | VERY HIGH / HIGH / MEDIUM / LOW |
| Last Detected | Most recent flap timestamp |

**Loop Likelihood:**
- **VERY HIGH**: ≥20 flaps between port pair
- **HIGH**: ≥10 flaps between port pair
- **MEDIUM**: ≥5 flaps between port pair
- **LOW**: <5 flaps between port pair

**Usage:** Identifies exactly which physical port pairs are forming loops. Disconnect one port to break the loop.

---

## Vendor-Specific Log Formats

### **Cisco MAC Flapping Messages**

#### **Catalyst Switches (SW_MATM)**
```
%SW_MATM-4-MACFLAP_NOTIF: Host 0012.3456.abcd in vlan 10 is flapping between port Gi0/1 and port Gi0/2
```

#### **Catalyst 4000/4500 (C4K_L2MAN)**
```
%C4K_L2MAN-4-MACFLAP: Host 0012.3456.abcd in vlan 10 is flapping between port Gi0/1 and port Gi0/2
```

#### **Platform FEA**
```
%PLATFORM_FEA-3-MAC_FLAP_DETECTED: MAC flap detected on vlan 10
```

#### **Loop Guard**
```
%SPANTREE-2-LOOPGUARD_BLOCK: Loop guard blocking port Gi0/1 on VLAN0010
%PM-4-ERR_DISABLE: loopdetect error detected on Gi0/1, putting Gi0/1 in err-disable state
```

#### **Storm Control**
```
%STORM_CONTROL-3-SHUTDOWN: Packet storm detected on Gi0/1, interface shutdown
```

---

### **Brocade MAC Flapping Messages**

#### **MAC Movement**
```
MAC address 0012.3456.abcd moved from port 1/1 to port 1/2, VLAN 10
MAC moved: 0012:3456:abcd from 1/1 to 1/2
```

#### **Loop Detection**
```
Loop detected on port 1/1, port disabled
MAC flapping detected on VLAN 10
```

---

## MAC Address Format Support

The queries detect all common MAC address formats:

| Format | Example | Vendor |
|--------|---------|--------|
| **Dotted Quad** | `0012.3456.abcd` | Cisco |
| **Colon-Separated** | `00:12:34:56:ab:cd` | Brocade, Standard |
| **Hyphen-Separated** | `00-12-34-56-ab-cd` | Windows, Some vendors |

**Regex patterns:**
```spl
[0-9a-fA-F]{4}\.[0-9a-fA-F]{4}\.[0-9a-fA-F]{4}  # Cisco
[0-9a-fA-F]{2}:[0-9a-fA-F]{2}:...                # Colon
[0-9a-fA-F]{2}-[0-9a-fA-F]{2}-...                # Hyphen
```

---

## Port Format Support

Detects both full names and abbreviations:

| Full Name | Abbreviation | Example |
|-----------|--------------|---------|
| GigabitEthernet | Gi | Gi0/1 |
| TenGigabitEthernet | Te | Te0/1 |
| FastEthernet | Fa | Fa0/1 |
| Ethernet | Et | Et0/1 |
| Port-channel | Po | Po1 |

**Also supports:**
- Brocade format: `1/1`, `1/1/1`
- Modular format: `Gi1/0/1`
- Sub-interfaces: `Gi0/1.100`

---

## Common Causes of MAC Flapping

### **1. Layer 2 Loops (Most Common)**
**Cause:** Two or more paths exist between switches without proper spanning tree.

**Symptoms:**
- MAC flapping between port pairs
- High broadcast/multicast traffic
- CPU spikes on switches
- Network slowness or complete outage

**Solution:**
- Check spanning tree topology
- Verify all switches have spanning tree enabled
- Look for manually configured trunk ports bypassing STP
- Check for unmanaged switches in the network

---

### **2. Spanning Tree Misconfiguration**
**Cause:** Incorrect STP priority, disabled STP, or topology changes.

**Symptoms:**
- MAC flapping during topology changes
- Multiple root bridge elections
- Ports rapidly transitioning between states

**Solution:**
- Verify spanning tree mode (PVST+, RSTP, MST)
- Check root bridge priorities
- Enable portfast on access ports
- Enable BPDU guard on access ports

---

### **3. Port Channel (LAG) Misconfiguration**
**Cause:** Mismatch in port channel configuration between switches.

**Symptoms:**
- MACs flapping between member ports of the channel
- Intermittent connectivity
- Asymmetric routing

**Solution:**
- Verify matching port channel protocols (LACP vs PAgP)
- Check all member ports are in same VLAN
- Verify load balancing method

---

### **4. Virtualization Issues**
**Cause:** VM mobility, vSwitch misconfigurations, or nested virtualization.

**Symptoms:**
- MAC flapping between hypervisor uplinks
- Occurs during vMotion/Live Migration
- MACs from same vendor (VMware OUI)

**Solution:**
- Check vSwitch configuration
- Verify VLAN tags match physical switches
- Disable spanning tree on hypervisor uplinks (use access mode)

---

### **5. Rogue Devices**
**Cause:** Unauthorized switch, hub, or misconfigured device creating loops.

**Symptoms:**
- New MAC addresses appearing
- Flapping starts suddenly
- Isolated to specific VLAN or location

**Solution:**
- Use port security
- Enable DHCP snooping
- Implement 802.1X authentication
- Physical port audit

---

## Troubleshooting Workflow

### **When MAC Flap Risk Score is High:**

#### **Step 1: Identify the Loop Location**
1. Check "Port Pair Analysis" table
2. Note which port pairs have "VERY HIGH" or "HIGH" risk
3. Identify the device and physical ports involved

**Example:**
```
Device: SW-CORE-01
Port Pair: Gi0/1 <-> Gi0/2
Flap Count: 47
Loop Risk: VERY HIGH
```

This indicates a loop between Gi0/1 and Gi0/2 on SW-CORE-01.

---

#### **Step 2: Check VLAN Scope**
1. Look at "MAC Flapping by VLAN" chart
2. Note which VLANs are affected
3. If all VLANs affected: **Native VLAN loop** or **trunk link loop**
4. If single VLAN: **Access port loop** or **VLAN-specific issue**

---

#### **Step 3: Review MAC Address Table**
1. Check "MAC Address Flapping Table"
2. Identify specific MAC addresses
3. Lookup MAC vendor (OUI):
   - `00:50:56` = VMware
   - `00:0C:29` = VMware
   - Vendor MAC = Likely real device
   - Unknown OUI = Rogue device?

---

#### **Step 4: Check Spanning Tree**
1. Look at "Spanning Tree Events" panel
2. Check for topology changes
3. Verify root bridge is stable
4. Look for "Loop Detection" timeline correlation

---

#### **Step 5: Immediate Mitigation**
```
! Cisco - Shutdown one port in the loop
SW-CORE-01(config)# interface GigabitEthernet0/1
SW-CORE-01(config-if)# shutdown
SW-CORE-01(config-if)# end

! Verify flapping stops
SW-CORE-01# show mac address-table dynamic | include 0012.3456.abcd

! If fixed, re-enable with proper protection
SW-CORE-01(config)# interface GigabitEthernet0/1
SW-CORE-01(config-if)# spanning-tree portfast
SW-CORE-01(config-if)# spanning-tree bpduguard enable
SW-CORE-01(config-if)# no shutdown
```

---

#### **Step 6: Root Cause Analysis**
**Check for:**
- Unmanaged switches
- Patch panel loops (cable plugged into wrong port)
- User plugging both ports of laptop dock
- Misconfigured VLANs on trunk links
- STP not enabled on some switches

---

## Prevention Best Practices

### **1. Enable STP Protection Features**
```cisco
! Enable portfast on access ports
interface range GigabitEthernet1/0/1-48
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable

! Enable loop guard on uplinks
interface range GigabitEthernet1/0/49-52
 spanning-tree guard loop

! Enable root guard on distribution uplinks
interface range TenGigabitEthernet1/1/1-4
 spanning-tree guard root
```

---

### **2. Implement Port Security**
```cisco
interface GigabitEthernet1/0/1
 switchport port-security
 switchport port-security maximum 3
 switchport port-security violation restrict
 switchport port-security aging time 2
```

---

### **3. Use UDLD (Unidirectional Link Detection)**
```cisco
! Enable globally
udld enable

! Enable aggressively on fiber links
interface TenGigabitEthernet1/1/1
 udld port aggressive
```

---

### **4. Configure Storm Control**
```cisco
interface GigabitEthernet1/0/1
 storm-control broadcast level 10.00
 storm-control multicast level 10.00
 storm-control action shutdown
```

---

### **5. Regular Audits**
- Review MAC address tables weekly
- Check spanning tree topology monthly
- Validate VLAN configurations quarterly
- Physical port audit (patch cables) quarterly

---

## Alerting Recommendations

### **Critical Alert**
```spl
MAC Flap Risk Score > 60 OR Flapping MAC Addresses > 10
```
**Action:** Page network team immediately

### **High Alert**
```spl
Loop Detection Events > 5 OR Port Pair with Loop Risk = "VERY HIGH"
```
**Action:** Email network team within 15 minutes

### **Warning Alert**
```spl
Affected VLANs > 3 OR Flapping MAC Addresses > 5
```
**Action:** Email network team for investigation

---

## Dashboard Comparison

| Feature | MAC Flapping | Routing Loop Detection |
|---------|--------------|------------------------|
| **Layer** | Layer 2 | Layer 3 |
| **Detects** | MAC movement | Routing adjacency flapping |
| **Scope** | Switch ports, VLANs | Routers, routing protocols |
| **Typical Cause** | STP loops, physical loops | Redistribution, config errors |
| **Fix Location** | Access/distribution switches | Core routers |
| **Impact** | Broadcast storms, CAM thrashing | Packet loops, routing instability |

**Both can occur simultaneously** when a Layer 2 loop affects Layer 3 adjacencies!

---

## Performance Considerations

### **Query Optimization**
These queries can be resource-intensive. Optimize by:

1. **Limit time range**:
   ```spl
   earliest=-1h latest=now
   ```

2. **Pre-filter with index constraints**:
   ```spl
   index=network_syslog ("MACFLAP" OR "MAC_FLAP")
   ```

3. **Use summary indexing** for historical analysis

---

## File Locations

- **Dashboard**: `/home/user/Dasplunk/spl` (lines 651-865)
- **This Documentation**: `/home/user/Dasplunk/MAC_FLAPPING_DETECTION.md`

---

## Version History

- **v1.0** (2025-12-04): Initial MAC flapping detection implementation
  - 4 KPI metrics
  - MAC flapping table with risk scoring
  - VLAN analysis
  - Port pair analysis
  - Loop detection timeline
  - Vendor-specific pattern support (Cisco, Brocade)
  - Multiple MAC address format support

---

## Additional Resources

### **Cisco Documentation**
- Spanning Tree Protocol Configuration Guide
- MAC Address Table Management
- Loop Guard Configuration

### **Brocade Documentation**
- FastIron Switching Configuration Guide
- Layer 2 Loop Detection

### **Related Dashboard Sections**
- Spanning Tree Events (shows STP topology changes)
- Interface Link State Changes (shows physical port flapping)
- Routing Loop Detection (Layer 3 loops)

---

Your dashboard now provides **comprehensive Layer 2 loop detection** to complement the existing Layer 3 routing loop detection! 🎯
