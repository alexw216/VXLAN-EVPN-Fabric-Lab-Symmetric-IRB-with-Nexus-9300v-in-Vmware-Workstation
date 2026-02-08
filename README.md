
# ## VXLAN EVPN Fabric Lab: Symmetric IRB with Nexus 9300v

A comprehensive 2-Spine, 2-Leaf VXLAN EVPN fabric design deployed in **VMware Workstation**. This lab demonstrates **L2 VNI extension**, **Anycast Gateway**, and **L3 VNI Symmetric IRB** routing.

## ### 1. Topology & Design

* **Spines (S1, S2):** BGP Route Reflectors & OSPF Underlay.
* **Leaves (L1, L2):** VTEPs (VXLAN Tunnel End Points).
* **Underlay:** OSPF Area 0.
* **Overlay:** MP-BGP EVPN.
* **Routing:** Symmetric IRB via Transit L3 VNI.

| Device | Loopback 0 | Role |
| --- | --- | --- |
| **S1** | 1.1.1.1/32 | Spine / RR 1 |
| **S2** | 1.1.1.2/32 | Spine / RR 2 |
| **L1** | 2.2.2.1/32 | VTEP 1 |
| **L2** | 2.2.2.2/32 | VTEP 2 |

---

## ### 2. Technical Theory

### #### Underlay (OSPF)

Provides IP reachability between Loopback 0 interfaces. The VTEPs use these loopbacks as the source/destination for the VXLAN UDP-encapsulated packets.

### #### L2 VNI & Anycast Gateway

Extends a Layer 2 broadcast domain across the IP fabric. **Anycast Gateway** allows both leaves to share the same Gateway IP and MAC (`0000.beef.cafe`), enabling seamless VM mobility.

### #### L3 VNI (Symmetric IRB)

Instead of mapping every VLAN to every leaf, we use a **Transit VNI (VNI 50000)**. Routing happens at both the Ingress and Egress leaves, which is highly scalable for multi-subnet environments.

---

## ### 3. Switch Configurations

### #### Spine 1 (S1)

```nxos
feature bgp
feature ospf
feature nv overlay
nv overlay evpn

hostname S1

interface loopback0
  ip address 1.1.1.1/32
  ip router ospf 1 area 0.0.0.0

interface Ethernet1/1
  no switchport
  ip address 10.1.1.1/30
  ip router ospf 1 area 0.0.0.0
  no shutdown

interface Ethernet1/2
  no switchport
  ip address 10.1.2.1/30
  ip router ospf 1 area 0.0.0.0
  no shutdown

router ospf 1
  router-id 1.1.1.1

router bgp 65001
  address-family l2vpn evpn
    retain route-target all
  template peer LEAF-LOGIC
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      route-reflector-client
      send-community extended
  neighbor 2.2.2.1 inherit peer LEAF-LOGIC
  neighbor 2.2.2.2 inherit peer LEAF-LOGIC

```

*(S2 configuration follows the same logic with IPs 1.1.1.2, 10.2.1.1, and 10.2.2.1)*

### #### Leaf 1 (L1)

> **Note:** Execute `hardware access-list tcam region racl 0` and `hardware access-list tcam region arp-ether 256 double-wide` followed by a `reload` before applying.

```nxos
feature bgp
feature ospf
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
feature fabric forwarding
nv overlay evpn

fabric forwarding anycast-gateway-mac 0000.beef.cafe

vlan 10
  vn-segment 10010
vlan 20
  vn-segment 10020
vlan 2000
  vn-segment 50000

vrf context TENANT-A
  vni 50000
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn

interface Vlan2000
  vrf member TENANT-A
  ip forward
  no shutdown

interface Vlan10
  vrf member TENANT-A
  ip address 10.1.10.1/24
  fabric-forwarding mode anycast-gateway
  no shutdown

interface nve1
  source-interface loopback0
  host-reachability protocol bgp
  member vni 50000 associate-vrf
  member vni 10010
    ingress-replication protocol bgp
  member vni 10020
    ingress-replication protocol bgp
  no shutdown

router bgp 65001
  address-family l2vpn evpn
  template peer SPINE-LOGIC
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
  neighbor 1.1.1.1 inherit peer SPINE-LOGIC
  neighbor 1.1.1.2 inherit peer SPINE-LOGIC

evpn
  vni 10010 l2
    rd auto
    route-target both auto
  vni 10020 l2
    rd auto
    route-target both auto

```

---

## ### 4. Verification & Test Scenarios

### #### Verification Commands

| Layer | Command |
| --- | --- |
| **Underlay** | `show ip ospf neighbors` |
| **Control Plane** | `show bgp l2vpn evpn summary` |
| **Data Plane** | `show nve vni` / `show nve peers` |
| **EVPN Routes** | `show bgp l2vpn evpn route-type 2` (MAC) / `type 5` (IP) |

### #### Test Scenarios

1. **L2 Extension:** Ping from VM1 (10.1.10.10) on L1 to VM2 (10.1.10.20) on L2.
2. **Anycast Gateway:** Ping 10.1.10.1 from any VM.
3. **Symmetric IRB:** Ping from VM1 (10.1.10.10) to VM3 (10.1.20.30).

---

This addition ensures that anyone cloning your repository knows exactly how to set up the virtual "cabling" and the specific VMX optimizations required for a Nexus 9300v lab in VMware Workstation.


## ## 5. Physical Topology & Cabling (VMware Mapping)

In VMware Workstation, the "Network Adapter" order directly maps to the Nexus Ethernet interfaces. Each connection below must be assigned to its own unique **LAN Segment**.

| Nexus Device | VMware Adapter | Interface ID | Connection To | LAN Segment Name |
| --- | --- | --- | --- | --- |
| **S1** | Adapter 2 | Eth 1/1 | L1 (Eth 1/1) | `S1-L1` |
| **S1** | Adapter 3 | Eth 1/2 | L2 (Eth 1/1) | `S1-L2` |
| **S2** | Adapter 2 | Eth 1/1 | L1 (Eth 1/2) | `S2-L1` |
| **S2** | Adapter 3 | Eth 1/2 | L2 (Eth 1/2) | `S2-L2` |
| **L1** | Adapter 4 | Eth 1/3 | Ubuntu VM 1 | `L1-HOSTS` |
| **L2** | Adapter 4 | Eth 1/3 | Ubuntu VM 2 | `L2-HOSTS` |

---

## ## 6. VMware Critical Configurations

### ### VMX File Optimization (Promiscuous Mode)

Because VXLAN generates virtual MAC addresses (Anycast Gateway) that differ from the VMware virtual NIC MAC, Workstation will drop packets by default. You **must** manually edit the `.vmx` file for all 4 Nexus nodes.

1. Shutdown all VMs.
2. Open each `.vmx` file in Notepad.
3. Add the following lines at the end of the file:

```text
ethernet1.noPromisc = "FALSE"
ethernet2.noPromisc = "FALSE"
ethernet3.noPromisc = "FALSE"

```

4. Restart the VMs.

### ### Test VM Setup (Ubuntu)

Ensure your Ubuntu VMs are connected to the `L1-HOSTS` and `L2-HOSTS` LAN Segments respectively. Configure the IP stack as follows:

**VM1 (on L1):**

```bash
sudo ip addr add 10.1.10.10/24 dev ens33
sudo ip route add default via 10.1.10.1

```

**VM2 (on L2):**

```bash
sudo ip addr add 10.1.20.20/24 dev ens33
sudo ip route add default via 10.1.20.1

```

---

## ## 7. Advanced Troubleshooting Checklist

If you see BGP Established but cannot ping:

1. **TCAM Verification:**
`show hardware access-list tcam region`
*Confirm `arp-ether` is 256 and `double-wide`.*
2. **ARP Suppression:**
On virtual labs, sometimes ARP suppression is flaky. If pings fail, disable it:
`interface nve1 -> member vni 10010 -> no suppress-arp`
3. **NVE Peer Discovery:**
`show nve peers`
*If this is empty, check `show bgp l2vpn evpn` to see if Route Type 3 (Inclusion Multicast) is being exchanged.*

---

