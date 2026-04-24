---
layout: post
title: "VXLAN- EVPN Simplified"
date: 2025-01-03
categories: [networking, datacenter]
---
Q. What is Symmetric IRB?
Symmetric IRB (Integrated Routing and Bridging)

First — What is IRB?
IRB is the ability of a VTEP to perform both bridging (L2) and routing (L3) at the same device — the leaf switch acts as both a switch and a router simultaneously.
Without IRB:  L2 switching and L3 routing are separate devices
With IRB:     Leaf VTEP does both — it's the default gateway for VMs

Why "Symmetric"?
Because both the ingress and egress VTEPs perform routing:
Symmetric:
  VTEP-1  →  Route  →  L3 VNI  →  VTEP-2  →  Route  →  VM-C
  (ingress routes)                 (egress routes)
      ↑                                  ↑
  both sides route = symmetric
Compare to Asymmetric where only ingress routes:
Asymmetric:
  VTEP-1  →  Route + Bridge  →  L2 VNI 10002  →  VTEP-2  →  Bridge only
  (ingress routes AND bridges)                    (egress just bridges)
      ↑                                                ↑
  only one side routes = asymmetric

Symmetric IRB — Step by Step
Scenario
VM-A: 10.1.1.10, VLAN 100, L2 VNI 10001, on VTEP-1
VM-C: 10.1.2.10, VLAN 200, L2 VNI 10002, on VTEP-2
Same tenant → VRF-A → L3 VNI 50000
Step 1: VM-A sends packet to VM-C
VM-A → default gateway (anycast GW on VTEP-1)
Src: 10.1.1.10
Dst: 10.1.2.10
Step 2: VTEP-1 routes (first routing hop)
VTEP-1 receives frame on L2 VNI 10001
Looks up 10.1.2.10 in VRF-A routing table
Finds: 10.1.2.10 reachable via VTEP-2 (learned via EVPN Type 2/5)
Step 3: VTEP-1 encapsulates with L3 VNI
Strips L2 VNI 10001 context
Re-encapsulates with L3 VNI 50000  ← VRF identifier on the wire
Outer IP: VTEP-1 → VTEP-2
Inner frame: routed packet (TTL decremented)
Step 4: Packet traverses underlay
Spine sees outer IP header only
Routes VTEP-1 loopback → VTEP-2 loopback
VXLAN payload invisible to spine
Step 5: VTEP-2 routes (second routing hop)
VTEP-2 decapsulates
Sees L3 VNI 50000 → maps to VRF-A
Routes 10.1.2.10 → found locally in VRF-A
Forwards into L2 VNI 10002 → delivers to VM-C
Full Flow Diagram
VM-A                VTEP-1              VTEP-2              VM-C
 |                    |                    |                   |
 |── 10.1.1.10 ──────>|                    |                   |
 |   dst:10.1.2.10    |                    |                   |
 |                    |─ route in VRF-A ──>|                   |
 |                    |  L3 VNI 50000      |─ route in VRF-A ─>|
 |                    |                    |  L2 VNI 10002      |
 |                    |                    |                   |


Key Characteristics of Symmetric IRB
1. L3 VNI is Always Used
Inter-subnet traffic always crosses fabric on L3 VNI
L2 VNI of source or destination never used for inter-subnet
2. Both VTEPs Must Have L3 VNI
VTEP-1 must have L3 VNI 50000 configured for VRF-A
VTEP-2 must have L3 VNI 50000 configured for VRF-A
If VTEP-2 missing L3 VNI → traffic drops
3. Both VTEPs Must Have the Subnets in VRF
VTEP-1 VRF-A routing table:
  10.1.1.0/24 → local (L2 VNI 10001)
  10.1.2.0/24 → via VTEP-2 (learned EVPN Type 5)

VTEP-2 VRF-A routing table:
  10.1.1.0/24 → via VTEP-1 (learned EVPN Type 5)
  10.1.2.0/24 → local (L2 VNI 10002)

Symmetric vs Asymmetric Comparison


Symmetric IRB
Asymmetric IRB
Who routes
Both ingress + egress VTEP
Ingress VTEP only
VNI used on fabric
L3 VNI (always)
L2 VNI of destination
L3 VNI required
Yes — on every VTEP
No
All VTEPs need all subnets
No — only local subnets
Yes — every VTEP needs every subnet
Scale
Better — subnets local to VTEPs
Poor — subnet sprawl across all VTEPs
Industry adoption
Dominant standard
Legacy / small deployments


Why Symmetric Wins at Scale
Asymmetric requires every VTEP to have every subnet in its local table:
Asymmetric — 100 subnets across 20 VTEPs:
  Every VTEP needs all 100 subnets configured
  → 2000 subnet configurations total

Symmetric — 100 subnets across 20 VTEPs:
  Each VTEP only needs its LOCAL subnets
  + one L3 VNI per tenant
  → scales cleanly

The One-Line Summary
In Symmetric IRB, the ingress VTEP routes and hands off via L3 VNI, and the egress VTEP receives and routes again — both sides do equal work, hence symmetric.

Q. What is Ingress replication?
One-Line Summary
Ingress Replication means the sending VTEP makes one unicast copy per remote VTEP for every BUM frame — no multicast needed, but bandwidth cost scales with VTEP count. EVPN Type 3 automates building the replication list so it's always accurate.

"Ingress" = the sending VTEP (where traffic enters the VXLAN fabric) "Replication" = making copies of the BUM frame and sending one to each remote VTEP
So Ingress Replication means: the sending VTEP is responsible for replicating BUM traffic to all other VTEPs in the VNI.

Why Replication is Needed
BUM traffic by nature must reach everyone in the VNI:
VM-A sends an ARP broadcast (BUM):
  → must reach VM-B on VTEP-2
  → must reach VM-C on VTEP-3
  → must reach VM-D on VTEP-4


VXLAN underlay is pure unicast IP — no native broadcast
Someone must replicate the frame to all destinations
In ingress replication, that someone is VTEP-1 itself.

Step by Step — How IR Works
Setup: Replication List
Each VTEP maintains a replication list per VNI — the list of all remote VTEPs participating in that VNI:
VTEP-1, VNI 10001 replication list:
  ├── 10.0.0.2  (VTEP-2)
  ├── 10.0.0.3  (VTEP-3)
  └── 10.0.0.4  (VTEP-4)

With BGP EVPN, this list is built automatically from Type 3 IMET routes. Without EVPN it is configured statically.
Step 1: VM-A Sends BUM Frame
VM-A → ARP broadcast
Arrives at VTEP-1 on VLAN 100 (L2 VNI 10001)
Step 2: VTEP-1 Checks Forwarding Table
VTEP-1:
  Is the destination MAC known? → No (unknown unicast) or Broadcast
  → Must flood to all VTEPs in VNI 10001
  → Consult replication list
Step 3: VTEP-1 Makes N Copies
Original frame → copy 1 → encapsulate → unicast to VTEP-2 (10.0.0.2)
              → copy 2 → encapsulate → unicast to VTEP-3 (10.0.0.3)
              → copy 3 → encapsulate → unicast to VTEP-4 (10.0.0.4)
Each copy is a full VXLAN encapsulated packet:
Copy 1:
  Outer IP: src 10.0.0.1, dst 10.0.0.2
  UDP dst: 4789
  VNI: 10001
  Inner: original ARP broadcast frame

Copy 2:
  Outer IP: src 10.0.0.1, dst 10.0.0.3
  UDP dst: 4789
  VNI: 10001
  Inner: same original ARP broadcast frame

Copy 3:
  Outer IP: src 10.0.0.1, dst 10.0.0.4
  ...

Step 4: Remote VTEPs Receive and Deliver
VTEP-2 receives copy 1:
  Decapsulates → sees VNI 10001 → maps to VLAN 100
  Delivers ARP broadcast to local VMs in VLAN 100

VTEP-3 receives copy 2:
  Same process → delivers to local VMs

VTEP-4 receives copy 3:
  Same process → delivers to local VMs


Bandwidth Impact
This is the key cost of IR:
Without IR (multicast):
  VTEP-1 uplink carries:  1 copy
  Spine replicates via multicast tree

With IR:
  VTEP-1 uplink carries:  N copies (one per remote VTEP)
  


Example:
  10 VTEPs in VNI → VTEP-1 sends 9 copies per BUM frame
  100 VTEPs in VNI → VTEP-1 sends 99 copies per BUM frame

This is why ARP suppression matters so much with IR — it eliminates the most common BUM source (ARP) so the replication cost rarely hits.

How EVPN Automates the Replication List
Without EVPN — static, painful:
VTEP-1 config:
  nve1:
    vni 10001:
      ingress-replication 10.0.0.2
      ingress-replication 10.0.0.3
      ingress-replication 10.0.0.4
← must manually update every VTEP when topology changes

With EVPN Type 3 (IMET) — dynamic, automatic:
VTEP-2 comes online →
  advertises BGP EVPN Type 3:
  "I am 10.0.0.2, I participate in VNI 10001, use ingress replication"

VTEP-1 receives Type 3 →
  automatically adds 10.0.0.2 to VNI 10001 replication list

VTEP-2 goes offline →
  BGP withdrawal received →
  VTEP-1 automatically removes 10.0.0.2 from list


IR vs Multicast — Traffic Path Comparison
Ingress Replication:
VTEP-1 ──unicast──> VTEP-2
VTEP-1 ──unicast──> VTEP-3
VTEP-1 ──unicast──> VTEP-4
       ↑
  3 separate packets leave VTEP-1 uplink


Multicast:

VTEP-1 ──multicast 239.1.1.1──> Spine
                                  ├──> VTEP-2
                                  ├──> VTEP-3
                                  └──> VTEP-4
       ↑
  1 packet leaves VTEP-1 uplink
  Spine replicates


Where Replication Happens
Method
Replication Point
Requirement
Ingress Replication
Sending VTEP (software/hardware)
None — just unicast IP
Multicast
Spine/underlay switches
PIM on every device
Static IR
Sending VTEP
Manual config
EVPN IR
Sending VTEP
BGP EVPN



FQ. How is the same gateway MAC and IP possible for every leaf?

Q.  What is VNI, 
VNI is the VXLAN equivalent of a VLAN ID — it identifies which logical network segment a frame belongs to.
Traditional:  VLAN ID  →  12 bits  →  4094 segments max
VXLAN:        VNI      →  24 bits  →  16,777,216 segments max

Where It Lives
VNI sits in the 8-byte VXLAN header, occupying 24 bits:
+─────────────────────────────────────────+
| Flags (8 bits) | Reserved (24 bits)     |
+─────────────────────────────────────────+
| VNI (24 bits)  | Reserved (8 bits)      |
+─────────────────────────────────────────+
The I flag must be set to 1 for the VNI field to be considered valid.

What VNI Actually Does
VNI tells the receiving VTEP which logical network this frame belongs to after decapsulation:
VTEP-2 receives encapsulated packet
↓
Strips outer Ethernet + IP + UDP
↓
Reads VXLAN header → VNI = 10001
↓
"This frame belongs to logical network 10001"
↓
Forwards inner frame into the correct bridge domain / VRF

Without VNI, VTEP-2 would have no idea which of its hundreds of logical networks the inner frame belongs to.

Two Types of VNI
This is critical — VNI serves two distinct purposes:
L2 VNI (Bridge Domain Identifier)
Maps to:    A single VLAN / broadcast domain
Purpose:    Intra-subnet traffic (same L2 segment)
Scope:      One subnet per L2 VNI
Example:    VNI 10001 = VLAN 100 = 10.1.1.0/24
L3 VNI (VRF Identifier)
Maps to:    A VRF (routing instance / tenant)
Purpose:    Inter-subnet traffic within same tenant
Scope:      One per tenant (e.g: company A,  B in DC), covers all subnets in that VRF
Example:    VNI 50000 = VRF-A = Tenant A routing domain
Visual Mapping
Tenant A (VRF-A)
├── L3 VNI: 50000  ← identifies VRF-A on the wire
├── Subnet 10.1.1.0/24  →  L2 VNI 10001  →  VLAN 100
├── Subnet 10.1.2.0/24  →  L2 VNI 10002  →  VLAN 200
└── Subnet 10.1.3.0/24  →  L2 VNI 10003  →  VLAN 300

Tenant B (VRF-B)
├── L3 VNI: 50001  ← identifies VRF-B on the wire
├── Subnet 10.1.1.0/24  →  L2 VNI 10004  →  VLAN 100
└── Subnet 10.1.2.0/24  →  L2 VNI 10005  →  VLAN 200
Note: Tenant A and B can reuse the same IP space — VNI provides isolation.

VNI in Action — Traffic Scenarios
Scenario 1: Intra-Subnet (L2 VNI used)
VM-A (10.1.1.10) → VM-B (10.1.1.20)
Same subnet → same broadcast domain

VTEP-1 encapsulates with L2 VNI 10001
VTEP-2 decapsulates → sees VNI 10001 → bridges into VLAN 100

Scenario 2: Inter-Subnet (L3 VNI used)
VM-A (10.1.1.10) → VM-C (10.1.2.10)
Different subnets → same tenant

VTEP-1 routes → encapsulates with L3 VNI 50000
VTEP-2 decapsulates → sees VNI 50000 → maps to VRF-A → routes to VM-C

Scenario 3: Different Tenant (isolated)
VM-A (Tenant A) → VM-X (Tenant B)
Different VRF → different L3 VNI

No path exists unless explicit VRF leaking configured
VNI 50000 and VNI 50001 are completely isolated

VNI to VLAN Mapping
On the VTEP, VNI maps to a local VLAN:
VTEP config:
  VLAN 100  ←→  VNI 10001
  VLAN 200  ←→  VNI 10002
  VLAN 300  ←→  VNI 10003

Ingress (from VM):
  Frame arrives tagged VLAN 100
  VTEP maps VLAN 100 → VNI 10001
  Encapsulates with VNI 10001

Egress (to VM):
  Packet arrives with VNI 10001
  VTEP maps VNI 10001 → VLAN 100
  Delivers frame tagged VLAN 100 to VM
The VLAN is local to the VTEP — different VTEPs can map different VLANs to the same VNI:
VTEP-1:  VLAN 100  →  VNI 10001
VTEP-2:  VLAN 200  →  VNI 10001  ← different local VLAN, same logical network
This is another expression of decoupling logical from physical.

VNI Allocation Best Practice
Range           Purpose
─────────────────────────────
10000–19999     L2 VNIs (one per subnet)
50000–59999     L3 VNIs (one per tenant VRF)
Keeping L2 and L3 VNI ranges separate avoids operational confusion and simplifies troubleshooting.

One-Line Summary Per Concept
Concept
One Line
VNI
24-bit segment identifier in VXLAN header
L2 VNI
Identifies a broadcast domain — one per subnet
L3 VNI
Identifies a VRF — one per tenant
VNI + I flag
I flag validates the VNI field is populated
VLAN↔VNI
Local mapping on each VTEP — can differ per VTEP


Full Tenant Picture on a Single VTEP
Physical VTEP (Leaf-1)
│
├── VRF-A (Tenant A)
│   ├── L3 VNI: 50000
│   ├── VLAN 100 ↔ L2 VNI 10001  (10.1.1.0/24)
│   └── VLAN 200 ↔ L2 VNI 10002  (10.1.2.0/24)
│
├── VRF-B (Tenant B)
│   ├── L3 VNI: 50001
│   ├── VLAN 300 ↔ L2 VNI 10003  (10.1.1.0/24)  ← overlapping IP, isolated
│   └── VLAN 400 ↔ L2 VNI 10004  (10.1.2.0/24)
│
└── VRF-C (Tenant C)
    ├── L3 VNI: 50002
    └── VLAN 500 ↔ L2 VNI 10005  (192.168.0.0/24)

Q.Where VNI  lives?  In this case VNI 10010?
VNI is only in the VXLAN header between the two VTEPs. It does not exist on the host side, and it does not exist in the underlay. It lives exclusively in the overlay tunnel.

The Three Segments of the Path
Host-A          Leaf-1                    Leaf-2          Host-B
  |                |                        |                |
  |--- VLAN 10 --->|                        |<--- VLAN 10 ---|
                   |<----- VNI 10010 ------>|
                         (VXLAN tunnel)

Each segment speaks a different language:
Segment 1 — Host-A to Leaf-1:
Plain Ethernet frame
Tagged with VLAN 10 (or untagged)
No VNI, no VXLAN, nothing overlay-related
Host has zero awareness of VNI
Segment 2 — Leaf-1 to Leaf-2 (the fabric):
VXLAN encapsulated packet
VNI 10010 sits inside the VXLAN header
This is the only place VNI exists in the physical network
Underlay sees it as plain UDP/IP — doesn't read the VNI
Segment 3 — Leaf-2 to Host-B:
Plain Ethernet frame again
Tagged with VLAN 10
VNI stripped off by Leaf-2's NVE interface
Host-B receives exactly what Host-A sent

What Each Device Knows About VNI
Host-A      →  knows nothing about VNI, speaks VLAN 10
Leaf-1      →  maps VLAN 10 → VNI 10010, adds VXLAN header
Spine       →  never reads VNI, routes outer IP packet only
Leaf-2      →  reads VNI 10010 → maps back to VLAN 10, strips VXLAN header
Host-B      →  knows nothing about VNI, receives VLAN 10 frame

The VNI exists only in Leaf-1 and Leaf-2's awareness — it's created at ingress, consumed at egress, invisible to everything else.

The VLAN to VNI Stitching — Where It Happens
Leaf-1 ingress:
  Frame arrives tagged VLAN 10
  NVE lookup: VLAN 10 → VNI 10010
  Encapsulate with VNI 10010 in VXLAN header
  Send into underlay

Leaf-2 egress:
  VXLAN packet arrives
  NVE reads VNI 10010
  Lookup: VNI 10010 → VLAN 10
  Strip VXLAN header
  Deliver frame tagged VLAN 10 to Host-B

This stitching is the NVE interface's core job — VLAN is the local concept, VNI is the fabric-wide concept. The NVE translates between them at each end.

Why This Design Is Powerful
Because VLAN 10 on Leaf-1 and VLAN 10 on Leaf-2 don't have to match — only the VNI has to match:
Leaf-1: VLAN 10  →  VNI 10010
Leaf-2: VLAN 20  →  VNI 10010   ← different local VLAN, same VNI
Host-A on Leaf-1 VLAN 10 and Host-B on Leaf-2 VLAN 20 are in the same logical segment because they share VNI 10010. The local VLAN is just a local label. VNI is the global tenant identity across the fabric.
This is one of EVPN-VXLAN's biggest operational advantages over traditional stretched VLANs — local VLAN numbering is decoupled from fabric-wide tenant identity.
