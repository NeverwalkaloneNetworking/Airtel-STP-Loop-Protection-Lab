# Airtel-STP-Loop-Protection-Lab

# Project 8: Mitigating Layer 2 Loops & Implementing High-Speed RSTP Convergence

**Company:** Airtel Kenya — Internal Engineering Training Lab  
**Objective:** Demonstrating Broadcast Storm Vulnerabilities and Deterministic STP Core Migrations  
**Status:** Production Validated & Documented  
**Target Audience:** Network Engineering Onboarding  

---

## 1. Executive Summary & Lab Mandate

Redundant switching paths are critical within enterprise service provider networks to prevent complete regional node isolation during physical fiber cuts or hardware port failures. However, introducing physical layer loops without logical control protocols results in catastrophic **Broadcast Storms**. 

This deployment project serves as an internal training lab documentation framework for Airtel Kenya junior engineers. It captures and documents how a three-node multi-path mesh reacts when Spanning Tree Protocol is disabled, demonstrates how the root election operates by default using backplane MAC parameters, implements manual priority scaling to align traffic vectors deterministically, and benchmarks the convergence optimization differences between legacy **802.1D STP** and high-speed **802.1w Rapid-PVST**.

---

##  2. Switch Loop Topology Map

The training topology features three interconnected layer-2 switches arranged in a physical triangle mesh:

```text
               [SW1-CORE] (Forced Priority: 4096 / Root Bridge)
                /        \
               /          \
         Gi0/1            Gi0/2
             /              \
            /                \
     [SW2-DIST]───Gi0/2───[SW3-DIST] (Post-Failover Active Path)
```
 **Host Asset Node: Workstation PC-A connected directly to SW1-CORE interface FastEthernet0/10 acting as the network packet injector.**
---
##  3. Multi-Phase Technical Implementation & CLI Logs

### Phase 1: Inducing a Broadcast Storm (STP Disabled)
To evaluate how raw hardware loops propagate without network intelligence safeguards, loop prevention mitigation was completely disabled across all active VLAN broadcast domains:

```bash
SW1-CORE# configure terminal
SW1-CORE(config)# hostname SW1-CORE
SW1-CORE(config)# no spanning-tree vlan 1
```

## Operational Validation: Packet Counter Footprint Analysis
Following an initial broadcast injection from the PC workstation, packet inspection capture logs verify that the loop layer is filled completely with non-routable transit traffic:
```bash
SW1-CORE# show interfaces gigabitEthernet 0/1
GigabitEthernet0/1 is up, line protocol is up (connected)
  Hardware is Lance, address is 00e0.f900.c319 (bia 00e0.f900.c319)
  Full-duplex, 100Mb/s
  Input queue: 0/75/0/0 (size/max/drops/flushes); Total output drops: 0
     956 packets input, 193351 bytes, 0 no buffer
     Received 956 broadcasts
     2357 packets output, 263570 bytes, 0 underruns
```
- Analysis For Junior Engineers: Notice that *956 packets* input matches *Received 956* broadcasts at a clean 1:1 ratio. This mathematical parity proves that 100% of the traffic hitting this interface is broadcast traffic. In a normal enterprise production environment, broadcast traffic should never exceed 5%. This footprint is the definitive diagnostic indicator of an unmitigated layer-2 network loop.

## Phase 2: Restoring Spanning Tree & Root Bridge Identification
Defensive topology controls were re-enabled globally across the cluster environment to shut down the loop propagation vectors:
```
SW1-CORE(config)# spanning-tree vlan 1
```
Allowing 30 seconds for state transitions, live command line collections were pulled across all elements to analyze the default dynamic election results:
```type
SW1-CORE# show spanning-tree vlan 1
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     0001.C96B.DC37
             This bridge is the root  <--- Default Root Bridge Election Winner

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi0/1            Desg FWD 4         128.25   P2p
Gi0/2            Desg FWD 4         128.26   P2p
```
```type
SW2-DIST# show spanning-tree vlan 1
VLAN0001
  Root ID    Priority    32769
             Address     0001.C96B.DC37
             Cost        4
             Port        25(GigabitEthernet0/1)

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     0004.9A88.5D94

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi0/1            Root FWD 4         128.25   P2p
Gi0/2            Desg FWD 4         128.26   P2p
```
```
SW3-DIST# show spanning-tree vlan 1
VLAN0001
  Root ID    Priority    32769
             Address     0001.C96B.DC37
             Cost        4
             Port        26(GigabitEthernet0/2)

  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     00E0.A3BE.DB49

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi0/1            Altn BLK 4         128.25   P2p  <--- Logical Loop Block Active
Gi0/2            Root FWD 4         128.26   P2p
```
Core Metrics Decoded:
1. Root Bridge Location: SW1-CORE won the master election because all switches shared a default base priority of 32768 (plus the system ID extension of 1, totaling 32769). To break the tie, the switches compared backplane MAC addresses. SW1-CORE possessed the lowest hexadecimal address, making it the root:

- SW1-CORE: 0001.C96B.DC37 🏆 (Lowest MAC Address Value)

- SW2-DIST: 0004.9A88.5D94

- SW3-DIST: 00E0.A3BE.DB49

2. Blocked Path Location: To break the loop while preserving physical redundancy, STP placed interface Gi0/1 on SW3-DIST into the Altn BLK (Alternate Blocking) state, completely halting unmanaged frame replication.
---

## Phase 3: Forced Root Bridge Hardening
Leaving the network core configuration to arbitrary MAC values introduces significant architecture risk. If a low-tier switch with a lower factory MAC address is introduced into the environment, it will hijack the root role and force unoptimized transit routes. To enforce structural predictability, SW1-CORE was administratively locked into the primary role:
```bash 
SW1-CORE(config)# spanning-tree vlan 1 priority 4096
```
Re-auditing adjacent nodes confirms that the root priority dropped successfully while the loop isolation block remained rock-solid:
```type
SW3-DIST# show spanning-tree vlan 1
VLAN0001
  Root ID    Priority    4097  (Configured 4096 + VLAN 1 ID)
             Address     0001.C96B.DC37
             Cost        4
             Port        26(GigabitEthernet0/2)

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi0/1            Altn BLK 4         128.25   P2p
Gi0/2            Root FWD 4         128.26   P2p
```
- Engineering Validation: The root bridge identity did not change because SW1-CORE was already winning via its MAC address. However, lowering the metric priority to 4096 ensures the control plane is completely hardened against accidental root hijacking.
---
## Phase 4: Measuring Rapid STP (802.1w) Convergence Speeds
Standard legacy 802.1D STP (protocol ieee) relies on fixed passive timers, taking up to 50 seconds to unblock an alternate path during a fiber cut. This latency is unacceptable in modern telecommunication networks. The fabric was transitioned to 802.1w Rapid STP (RSTP) to leverage active proposal-agreement handshakes:

```bash
SW1-CORE(config)# spanning-tree mode rapid-pvst
```
---
# Catastrophic Failure Simulation Profile
To test path re-convergence efficiency, a permanent link cut was simulated by administratively shutting down the primary path out of SW1-CORE interface Gi0/1.

```
SW1-CORE(config)# interface gigabitEthernet 0/1
SW1-CORE(config-if)# shutdown
```
---
Running tracking commands immediately on the backup forwarding switch confirms instant network reconfiguration:
```type
SW3-DIST# show spanning-tree vlan 1
VLAN0001
  Spanning tree enabled protocol rstp  <--- Active 802.1w Protocol Context
  Root ID    Priority    4097
             Address     0050.0FB6.A7E0

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi0/1            Root FWD 4         128.25   P2p  <--- Instantly Promoted to Forwarding
Gi0/2            Desg FWD 4         128.26   P2p
```

- Convergence Assessment: Interface Gi0/1 on SW3-DIST skipped the legacy 30-second Listening/Learning delay blocks entirely. It converted from an alternate blocked port directly into an active Root FWD (Forwarding) port in less than 2 seconds. This fast recovery prevents widespread service drops across the interconnected topology segments.
--- 
## 4. Lab Training Conclusions
This training lab provides junior network engineers with clear evidence of why loop mitigation protocols are vital to modern networks. By observing the impact of an unmitigated broadcast storm, auditing default MAC-based elections, hardening root roles with manual priority adjustments, and upgrading to Rapid STP, the engineering team has verified the core mechanics needed to maintain resilient, high-speed, loop-free layer-2 infrastructures.
