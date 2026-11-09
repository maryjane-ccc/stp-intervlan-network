# stp-intervlan-network
# Week 4: STP, VLANs and Inter-VLAN Communication

**Project Series:** 10 Weeks of Networking: From NetAcad Student to Network Builder  
**Simulation Tool:** Cisco Packet Tracer  
**Status:** Completed

## Project Overview

This project involved designing and configuring a redundant enterprise network using VLAN segmentation, trunk links, Spanning Tree Protocol (STP), DHCP and inter-VLAN routing.

The primary objective was to understand how switches manage redundant connections, prevent Layer 2 switching loops and maintain network connectivity when network links fail.

The project also provided practical experience with router-on-a-stick configuration and automatic IP address allocation across multiple VLANs.

https://drive.google.com/file/d/1bVAAuM89ZNhC74xjuuoD3EvVwrisNf7h/view?usp=drive_link

## Network Topology

The network consists of:

- 4 Cisco 2960 switches (S1, S2, S3 and S4).
- 1 Cisco 2911 router (R1).
- 8 PCs, with two connected to each switch.
- 6 inter-switch connections forming a redundant topology.
- 1 router-to-switch connection for inter-VLAN routing.

### Topology Design

The four switches were connected in a square topology with two diagonal links:

```text
          S1 -------- S2
          |  \      / |
          |   \    /  |
          |    \  /   |
          |    /  \   |
          |   /    \  |
          |  /      \ |
          S4 -------- S3

          |
          R1
```

The six inter-switch connections provide multiple paths between switches. This introduces redundancy but also creates potential Layer 2 loops, making STP necessary.

*The diagram is a simplified representation of the topology.*

## VLAN and IP Addressing Plan

Two VLANs were implemented across all four switches.

| VLAN ID | VLAN Name | Network | Default Gateway |
|---|---|---|---|
| 10 | STAFF | 192.168.10.0/24 | 192.168.10.1 |
| 20 | GUEST | 192.168.20.0/24 | 192.168.20.1 |

VLANs logically separate staff and guest devices into different broadcast domains, even though they share the same physical switching infrastructure.

## Technologies Implemented

### 1. VLAN Configuration

VLAN 10 (STAFF) and VLAN 20 (GUEST) were created on all four switches.

Example configuration:

```cisco
enable
configure terminal

vlan 10
 name STAFF

vlan 20
 name GUEST

end
```

PC-facing interfaces were configured as access ports and assigned to their respective VLANs.

```cisco
configure terminal

interface range fa0/1 - 4
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast

interface range fa0/5 - 8
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast

end
```

PortFast was enabled on end-device ports to allow them to transition quickly to the forwarding state. It was not enabled on inter-switch links.

### 2. Trunk Configuration

Trunk links were configured between switches to carry traffic from multiple VLANs over a single physical connection.

Only VLANs 10 and 20 were permitted.

```cisco
configure terminal

interface range fa0/23 - 24
 switchport mode trunk
 switchport trunk allowed vlan 10,20

end
```

The same principle was applied to the relevant GigabitEthernet inter-switch links.

The router-facing interface on S1 was also configured as a trunk:

```cisco
interface gi0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
```

**Verification:**

```cisco
show interfaces trunk
show vlan brief
```

An important observation was that a VLAN could be allowed on a trunk but not forwarding because STP had placed that path in a blocking state.

### 3. Spanning Tree Protocol (STP)

The network used Cisco Per-VLAN Spanning Tree Plus (PVST+).

STP prevents Layer 2 switching loops by identifying redundant paths and placing selected ports into a blocking state.

```cisco
spanning-tree mode pvst
```

#### Root Bridge Configuration

Different switches were configured as primary and secondary root bridges for the two VLANs.

| Switch | VLAN | Configured Role |
|---|---|---|
| S1 | 10 | Primary Root |
| S2 | 10 | Secondary Root |
| S3 | 20 | Primary Root |
| S4 | 20 | Secondary Root |

**S1:**

```cisco
spanning-tree vlan 10 root primary
```

**S2:**

```cisco
spanning-tree vlan 10 root secondary
```

**S3:**

```cisco
spanning-tree vlan 20 root primary
```

**S4:**

```cisco
spanning-tree vlan 20 root secondary
```

#### STP Verification

```cisco
show spanning-tree summary
show spanning-tree vlan 10
show spanning-tree vlan 20
```

These commands were used to inspect:

- Root bridge information.
- Root, designated and alternate port roles.
- Forwarding and blocking states.
- STP mode and configuration.

#### Failover Testing

A failure test was performed by disabling a relevant connection on the primary root switch.

STP recalculated the topology, and the configured secondary switch eventually became the root bridge for VLAN 10.

This demonstrated how STP responds to topology changes and maintains a loop-free network.

**Related Huawei Concept:** VBST (VLAN-Based Spanning Tree) provides a similar per-VLAN spanning-tree approach in Huawei networking environments. However, the actual implementation in this project was Cisco PVST+, not Huawei VBST.

### 4. Router-on-a-Stick and Inter-VLAN Routing

VLANs create separate broadcast domains. Consequently, devices in different VLANs require Layer 3 routing to communicate.

Router-on-a-stick was implemented using one physical router interface with multiple logical subinterfaces.

Each subinterface was associated with a VLAN using IEEE 802.1Q encapsulation.

```cisco
enable
configure terminal

interface gigabitEthernet 0/0
 no shutdown

interface gigabitEthernet 0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

interface gigabitEthernet 0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0

end
```

| Subinterface | VLAN | Default Gateway |
|---|---|---|
| G0/0.10 | 10 | 192.168.10.1 |
| G0/0.20 | 20 | 192.168.20.1 |

The router serves as the default gateway for both VLANs and handles communication between them.

### 5. DHCP Configuration

DHCP was configured on R1 to automatically assign IP addresses to devices in both VLANs.

First, reserved addresses were excluded from the DHCP allocation range.

```cisco
configure terminal

ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10

end
```

**STAFF DHCP Pool:**

```cisco
ip dhcp pool STAFF
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
```

**GUEST DHCP Pool:**

```cisco
ip dhcp pool GUEST
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
```

The DNS address was configured as an example public DNS resolver.

PCs were configured to obtain their IP addresses automatically using DHCP.

**Verification:**

```cisco
show ip dhcp pool
show ip dhcp binding
show ip interface brief
```

## Testing and Verification

The following commands were used to verify the network configuration and observe its behaviour.

| Command | Purpose |
|---|---|
| `show vlan brief` | Displays VLANs and access port assignments |
| `show interfaces trunk` | Verifies trunk interfaces and allowed VLANs |
| `show spanning-tree summary` | Displays STP mode and summary |
| `show spanning-tree vlan 10` | Displays STP information for VLAN 10 |
| `show spanning-tree vlan 20` | Displays STP information for VLAN 20 |
| `show ip interface brief` | Checks router interface status and IP addresses |
| `show ip dhcp pool` | Displays DHCP pool information |
| `show ip dhcp binding` | Displays assigned DHCP addresses |

Connectivity was also tested using ping between devices and their respective default gateways.

## Key Concepts and Lessons Learned

Through this project, I developed a better understanding of the following:

**1. VLAN Segmentation**

VLANs allow devices to be logically separated into different broadcast domains without requiring separate physical switches.

**2. Access vs. Trunk Ports**

Access ports carry traffic for a single VLAN, while trunk ports carry traffic from multiple VLANs using IEEE 802.1Q tagging.

**3. Layer 2 Switching Loops**

Redundant switch connections can introduce loops, resulting in broadcast storms, duplicate frames and MAC address table instability.

**4. STP Root Bridge Election**

The root bridge is elected based on the lowest Bridge ID. PVST+ performs this election independently for each VLAN.

**5. STP Failover**

When the active root bridge becomes unavailable, STP can recalculate the topology and select another root bridge. A configured secondary root is intended to provide a preferred backup.

**6. Per-VLAN Spanning Tree**

Different VLANs can have different root bridges and forwarding paths, allowing more flexible network design.

**7. Inter-VLAN Routing**

Devices in separate VLANs require Layer 3 routing to communicate. Router-on-a-stick achieves this using multiple subinterfaces on one physical router interface.

## Skills Developed

- VLAN creation and management.
- Access and trunk port configuration.
- IEEE 802.1Q tagging.
- Cisco PVST+ configuration.
- Root bridge election and management.
- STP port roles and states.
- STP failover testing.
- Router-on-a-stick configuration.
- Inter-VLAN routing.
- DHCP configuration and verification.
- Network connectivity testing and troubleshooting.

## Project Takeaway

This project connected several networking concepts that I had previously studied independently.

One of my biggest takeaways was understanding the relationship between redundancy and STP. Redundant links improve network resilience, but without loop prevention, those same connections can destabilise the network.

I also learned that VLAN segmentation alone does not provide communication between different network segments. Layer 3 routing is necessary to enable that communication.

Building and testing this network in Cisco Packet Tracer helped me move beyond theoretical networking concepts and understand how they work together in a practical environment.


**Part of my 10 Weeks of Networking: From NetAcad Student to Network Builder series.**
