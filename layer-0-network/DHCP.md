# Layer 0 — DHCP

> DHCP configuration, static mappings, and log interpretation guide.

---

## Current DHCP Service
- **Service:** Kea DHCP
- **Status:** Active
- **Subnet:** 192.168.x.0/24 (current LAN)
- **Leases:** Dynamic by default with exceptions
- **Interface:** LAN

## Static Mappings
- Static DHCP mapping configured for:
  - Vintage Story server
- Verified lease transition from dynamic → static

## Domain Name
The host name + domain name helps to better identify devices and where they are located in the logs. You can also resolve to these names making it easier when connecting to devices or services. For the domain name, it is not wrong to use random names however, it is currently best practice to use `home.arpa` as it is designed for non-unique use in residential home networks. The domain is not resolvable across the internet and is intended only for use in small networks.

---

## Migration from DNSmasq to Kea

- **Target DHCP Service:** Kea DHCP
- **Reason for Migration:**
  - Better logging
  - Better multi-subnet / VLAN support
  - Cleaner separation of DNS and DHCP roles
  - Future automation friendliness

> Migration to Kea was planned **before VLAN rollout**
> to reduce complexity during segmentation.

---

## Kea DHCP Log Cheat Sheet (DHCPv4)

Kea DHCP logs describe how IP addresses are assigned, renewed, and released.
Understanding these logs allows you to verify correctness, detect conflicts,
and debug DHCP issues without guessing.

### Core DHCP Message Types (The DHCP Handshake)

| Message      | Name                    | Explanation                                                |
| ------------ | ----------------------- | ---------------------------------------------------------- |
| DHCPDISCOVER | Client Discovery        | Client broadcasts "Is there any DHCP server?"              |
| DHCPOFFER    | Server Offer            | DHCP server proposes an IP address                         |
| DHCPREQUEST  | Client Request          | Client requests the offered IP                             |
| DHCPACK      | Server Acknowledgment   | Server confirms and assigns the IP                         |
| DHCPRELEASE  | Client Release          | Client voluntarily releases its lease                      |
| DHCPNAK      | Negative Acknowledgment | Server rejects the request (misconfig, wrong subnet, etc.) |

This four-step exchange is known as **DORA**:
DISCOVER → OFFER → REQUEST → ACK

### Common Log Fields

| Field       | Meaning                                    |
| ----------- | ------------------------------------------ |
| hwtype      | Hardware type (1 = Ethernet)               |
| MAC address | Physical address of the client             |
| cid         | Client Identifier (may differ from MAC)    |
| tid         | Transaction ID (unique per DHCP exchange)  |
| interface   | Interface handling the request (e.g. igc1) |
| lease       | IP address being offered or assigned       |

### Lease Lifecycle Events

| Log Entry         | Meaning                        |
| ----------------- | ------------------------------ |
| DHCP4_LEASE_OFFER | IP address proposed to client  |
| DHCP4_LEASE_ALLOC | IP address officially assigned |
| DHCPRELEASE       | Client gave back its IP        |
| valid-lifetime    | Lease duration (in seconds)    |

### Static Reservations (Very Important)
When a client has a reservation, Kea logs will show:
- The reserved IP being offered
- Assignment **outside** the dynamic pool
- The reservation overriding pool logic

This proves the reservation system is working.

### Example: Dynamic Client (Pool Assignment)
```
DHCPDISCOVER  
DHCPOFFER lease 192.168.0.x
DHCPREQUEST  
DHCPACK lease allocated
```
Meaning:
- Client had no reservation
- IP came from the pool
- Lease assigned normally

### Example: Reserved Host (Static Mapping)
```
DHCPDISCOVER  
DHCPOFFER lease 192.168.x.20  
DHCPREQUEST  
DHCPACK lease allocated
```
Meaning:
- MAC matched a reservation
- IP assigned outside pool
- Reservation took priority

### DHCPRELEASE Explained
```
DHCPRELEASE received from 192.168.x.20
```
Meaning:
- Client shut down cleanly
- Lease returned early
- IP becomes reusable (unless reserved)

This is normal behavior for servers and VMs.

### What Normal Looks Like
Normal DHCP noise includes:
- Frequent DISCOVER / REQUEST messages
- Clients renewing leases
- Infrastructure devices asking occasionally
- DHCPRELEASE on shutdown or reboot

A quiet DHCP log is **not** expected.

### Red Flags to Watch For

| Pattern                         | Meaning                                     |
| ------------------------------- | ------------------------------------------- |
| Repeated DISCOVER with no OFFER | DHCP server unreachable                     |
| Frequent DHCPNAK                | Wrong subnet or conflicting servers         |
| Same IP offered repeatedly      | Pool exhaustion or conflict                 |
| No DHCPRELEASE ever             | Client crashes or power loss (not critical) |
