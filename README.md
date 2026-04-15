# SDN-Mininet-ARP-Handler (Orange Problem)

| | |
|---|---|
| **Name** | Sri Pranav Gautam Buduguru |
| **Section** | 4H |
| **SRN** | PES1UG24CS466 |


## Introduction
This project presents the implementation for the "Orange Problem" assignment. The core goal is to design a Software-Defined Networking (SDN) controller module that replaces the conventional switch-level ARP flooding mechanism with a centralized, controller-driven approach.

The controller actively intercepts ARP queries, maintains a dynamic MAC-to-IP mapping table, and pushes proactive flow entries into the OpenFlow switch — thereby minimizing unnecessary broadcast traffic and dramatically improving forwarding latency.

---

## Network Setup & Tools
* **Operating System:** Ubuntu Linux (Virtual Machine)
* **Emulator:** Mininet
* **Topology Used:** `single,3` — a single OpenFlow switch with three end-hosts
* **Controller Framework:** POX (supporting OpenFlow 1.0)
* **Programming Language:** Python 3

**Rationale for Topology Choice:**
The `single,3` topology was chosen as it provides the simplest setup to clearly observe ARP broadcast behavior, controller-based interception, and MAC address learning — without adding multi-hop routing overhead.

---

## How the Controller Works (`arp_handler.py`)
The custom POX controller module operates through three primary mechanisms:

1. **MAC Address Learning:** On every `PacketIn` event triggered by an ARP packet, the controller extracts the source IP and MAC address pair and records it in an in-memory lookup table (`self.arp_table`).
2. **ARP Reply Fabrication:** When the controller receives an ARP request for a host whose MAC is already known, it constructs an ARP Reply on behalf of that host and sends it back through the ingress port (`of.OFPP_IN_PORT`), thus avoiding broadcast flooding.
3. **Flow Rule Injection:** After MAC resolution is complete, any subsequent traffic (e.g., ICMP Echo) causes the controller to install explicit `ofp_flow_mod` entries on the switch. These entries (with `idle_timeout=60` and `hard_timeout=120`) enable direct packet forwarding at hardware speed, eliminating further controller involvement.

---

## How to Run

### Step 1 — Launch the POX Controller
Open a terminal, navigate to the POX installation directory, and start the ARP handler module:
```bash
cd ~/pox
./pox.py arp_handler
```

### Step 2 — Create the Mininet Network
In a separate terminal window, clean up any residual state and spin up the topology:
```bash
sudo mn -c
sudo mn --topo single,3 --controller remote
```

### Step 3 — Trigger ARP Resolution via Ping
From the Mininet CLI, run a connectivity test between two hosts:
```
mininet> h1 ping -c 5 h2
```

### Step 4 — Inspect Installed Flow Rules
After pinging, verify that flow entries were pushed to the switch's flow table:
```
mininet> dpctl dump-flows
```

---

## Results & Analysis
The ICMP ping output clearly demonstrates the effectiveness of the SDN controller's ARP handling logic:

* **First-Packet Delay:** The initial ping exhibits high latency (approximately ~1086 ms) because the controller must handle the `PacketIn` event, perform ARP resolution, construct the reply, and compute the forwarding path before installing any rules.

* **Subsequent Packet Performance:** From the second ping onward, latency drops to sub-millisecond levels (~0.113 ms), confirming that the `ofp_flow_mod` entries were successfully installed and the switch is now forwarding packets autonomously at line-rate.

---

## Screenshots & Evidence

### 1. Controller Terminal Output (MAC Learning & ARP Handling)
The following capture shows the controller's log output — initial ARP flooding for unknown hosts, MAC address learning, direct ARP reply generation for known hosts, and OpenFlow rule installation.

<img width="1000" height="641" alt="image" src="https://github.com/user-attachments/assets/074e22a5-67f7-4d87-91ab-b4051caf9689" />

---

### 2. Ping Statistics (Latency Comparison)
This output illustrates the significant latency reduction between the first controller-handled ping and the subsequent hardware-accelerated pings.

<img width="997" height="214" alt="image" src="https://github.com/user-attachments/assets/bc2410e4-d81d-4e2a-bfab-7f40a6eadc31" />

---

### 3. Flow Table Verification (`dpctl dump-flows`)
This confirms that explicit forwarding rules with configured timeouts were successfully pushed to switch `s1`.

<img width="1958" height="108" alt="image" src="https://github.com/user-attachments/assets/2dc05450-6c79-4147-a2c1-f7ffc6c41cc4" />

---

### 4. Wireshark Capture
The packet-level trace validates the end-to-end behavior: the initial ARP Broadcast, the controller-generated ARP Reply, followed by a clean ICMP Echo Request/Reply sequence.

<img width="980" height="823" alt="image" src="https://github.com/user-attachments/assets/ade86e2c-8477-4ada-95fe-8b46a45c1d05" />
