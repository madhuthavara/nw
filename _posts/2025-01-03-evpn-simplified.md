---
layout: post
title: "VXLAN- EVPN Simplified"
date: 2025-01-03
categories: [networking, datacenter]
---
# VXLAN-EVPN Simplified

A practical guide to common EVPN-VXLAN concepts explained in simple terms.

---

# What is Symmetric IRB?

**Symmetric IRB** (Integrated Routing and Bridging) means both the ingress and egress VTEPs perform routing for inter-subnet traffic.

## First: What is IRB?

IRB allows a leaf switch (VTEP) to do both:

- **Layer 2 bridging**
- **Layer 3 routing**

Without IRB:

- L2 switching and L3 routing are separate devices

With IRB:

- The leaf VTEP acts as both switch and router
- It becomes the default gateway for attached hosts

---

## Why is it called Symmetric?

Because both sides route traffic.

### Symmetric IRB

```text
VTEP-1 → Route → L3 VNI → VTEP-2 → Route → VM-C
Ingress VTEP routes
Egress VTEP routes
Asymmetric IRB
VTEP-1 → Route + Bridge → L2 VNI → VTEP-2 → Bridge only
Only ingress VTEP routes
Symmetric IRB Example
Scenario
Device	IP	VLAN	VNI
VM-A	10.1.1.10	VLAN 100	L2 VNI 10001
VM-C	10.1.2.10	VLAN 200	L2 VNI 10002

Tenant VRF:

VRF-A = L3 VNI 50000
Packet Flow
Step 1 – VM-A Sends Traffic
Src: 10.1.1.10
Dst: 10.1.2.10
Gateway: Anycast GW on VTEP-1
Step 2 – VTEP-1 Routes
Looks up destination in VRF-A
Learns VM-C reachable via VTEP-2
Step 3 – Encapsulates with L3 VNI
Outer IP: VTEP-1 → VTEP-2
VNI: 50000
Step 4 – Underlay Routes Packet

Spines only see outer IP header.

Step 5 – VTEP-2 Routes Again
Decapsulates
Maps L3 VNI 50000 to VRF-A
Forwards to VLAN 200 / VM-C
Symmetric vs Asymmetric IRB
Feature	Symmetric	Asymmetric
Who routes	Both VTEPs	Ingress only
Fabric VNI	L3 VNI	Destination L2 VNI
Scale	Better	Poorer
Subnet sprawl	No	Yes
Industry use	Standard	Legacy
Why Symmetric Wins

Asymmetric requires every VTEP to know every subnet.

Symmetric allows:

Local subnet ownership
Cleaner scaling
Simpler operations
What is Ingress Replication?

Ingress Replication means the sending VTEP creates one copy of BUM traffic for each remote VTEP.

BUM =

Broadcast
Unknown Unicast
Multicast
Why Needed?

VXLAN underlay is normal IP unicast.

It has no native broadcast domain.

So someone must copy ARP broadcasts etc.

That someone is the ingress VTEP.

Example
VM-A sends ARP broadcast

VTEP-1 sends copies to:

VTEP-2
VTEP-3
VTEP-4
Traffic Example
VTEP-1 → VTEP-2
VTEP-1 → VTEP-3
VTEP-1 → VTEP-4

Each packet is separate unicast VXLAN traffic.

Cost of Ingress Replication
VTEPs in VNI	Copies Sent
10	9
100	99

This is why ARP suppression is important.

EVPN Type-3 IMET Automates This

Remote VTEPs advertise:

I participate in VNI 10001
Use me in replication list

So replication membership becomes dynamic.

What is VNI?

VNI = VXLAN Network Identifier

Equivalent to VLAN ID in VXLAN.

Technology	Bits	Segments
VLAN	12	4094
VNI	24	16,777,216
Where VNI Lives

Inside VXLAN header only.

Host → Leaf → Fabric → Leaf → Host

Only between VTEPs.

Hosts never see VNI.

Spines do not inspect VNI.

Two Types of VNI
1. L2 VNI

Used for bridging within same subnet.

Example:

VNI 10001 = VLAN 100 = 10.1.1.0/24
2. L3 VNI

Used for routing within tenant VRF.

Example:

VNI 50000 = VRF-A
Example Tenant Mapping
Tenant A
├── L3 VNI 50000
├── Subnet 10.1.1.0/24 → L2 VNI 10001
├── Subnet 10.1.2.0/24 → L2 VNI 10002
Traffic Examples
Same Subnet
VM-A → VM-B
Uses L2 VNI 10001
Different Subnet
VM-A → VM-C
Uses L3 VNI 50000
Different Tenant

No path unless route leaking configured.

VLAN to VNI Mapping

Local on each VTEP:

VLAN 100 ↔ VNI 10001
VLAN 200 ↔ VNI 10002

Different switches can use different VLAN IDs.

Example:

Leaf-1 VLAN 10 → VNI 10010
Leaf-2 VLAN 20 → VNI 10010

Still same logical network.

Where Does VNI Exist?

Only in overlay tunnel.

Host-A -- VLAN10 --> Leaf-1 == VNI10010 == Leaf-2 -- VLAN10 --> Host-B
What Devices Know
Device	Knows VNI?
Host	No
Leaf	Yes
Spine	No
Remote Leaf	Yes
Why This is Powerful

VLAN numbers become local only.

VNI becomes global segment identity.

This decouples physical VLAN design from logical tenant networking.

One-Line Summary

EVPN-VXLAN lets you stretch L2/L3 networks across a fabric using scalable overlays, while EVPN provides the control plane intelligence.
