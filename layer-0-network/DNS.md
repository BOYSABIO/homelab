# Layer 0 — DNS

> DNS configuration, rationale, and log interpretation guide.

---

## Service
- **DNS Resolver:** Unbound
- **Mode:** Recursive
- **DNSSEC:** Enabled
- **Query Logging:** Enabled

## Behavior
- Clients use OPNsense as DNS server
- Unbound performs full recursive resolution:
  - Root → TLD → Authoritative
  - Recursion essentially means OPNsense will not only resolve DNS lookups locally but will also store all the logs to maintain control and future data that can be used for ML / AI analysis. **DNS is a crucial area for SOC**
- No third-party DNS forwarders configured

## Rationale
- Maximum privacy (no central DNS observer)
- Full visibility into DNS telemetry
- Better foundation for future detection & ML work
- Avoids trust dependency on public resolvers

## Observability
- Queries visible in Unbound logs
- Client IPs visible
- Domain patterns observable

---

## Log Cheat Sheet

Unbound DNS logs show how name resolution requests are processed by the firewall.
Understanding these entries helps distinguish normal behavior from misconfiguration or attacks.

### Common Log Fields

| Field       | Meaning                                                               |
| ----------- | --------------------------------------------------------------------- |
| Client IP   | The device that sent the DNS query (e.g. 192.168.0.x)                 |
| Domain name | The fully qualified domain name being queried (often ends with a dot) |
| Record Type | The type of DNS record requested (A, AAAA, PTR, etc.)                 |
| Class (IN)  | The DNS class — almost always `IN` (Internet)                         |

### DNS Record Types You Will Commonly See

| Record | Name                            | Explanation                                                               |
| ------ | ------------------------------- | ------------------------------------------------------------------------- |
| A      | IPv4 Address Record             | Maps a hostname to an IPv4 address (e.g. www.example.com → 93.184.216.34) |
| AAAA   | IPv6 Address Record             | Maps a hostname to an IPv6 address                                        |
| PTR    | Pointer Record (Reverse Lookup) | Maps an IP address back to a hostname (used for reverse DNS)              |
| NS     | Name Server Record              | Indicates which DNS servers are authoritative for a domain                |
| SOA    | Start of Authority              | Contains administrative information about a DNS zone                      |
| TXT    | Text Record                     | Carries arbitrary text (used for SPF, DKIM, verification, etc.)           |
| SRV    | Service Record                  | Used to locate services (e.g. LDAP, SIP, Kerberos)                        |

### DNS Class: IN

| Value | Meaning                                       |
| ----- | --------------------------------------------- |
| IN    | Internet (default and standard class for DNS) |

Other classes exist historically but are rarely used today.
In modern networks, DNS logs will almost always show `IN`.

### Fully Qualified Domain Names (FQDN)
Domains in logs often end with a dot:

Example: www.bing.com.

The trailing dot means:
> "This name is absolute and fully qualified, starting at the DNS root."

It is normal and expected in DNS logs.

### What Indicates Normal DNS Recursion
Recursion is working correctly when:
- Clients query the firewall
- The firewall resolves domains it does not host locally
- Responses are returned without SERVFAIL errors

Seeing queries for public domains (bing.com, cloudflare.com, omada, etc.)
indicates recursive resolution is functioning.

### Red Flags to Watch For

| Pattern                         | Meaning                                |
| ------------------------------- | -------------------------------------- |
| SERVFAIL spikes                 | Upstream resolution problems           |
| Large volumes of random domains | Possible malware or misbehaving client |
| External DNS servers contacted  | Potential DNS bypass or leak           |
| Rebind protection blocks        | Possible DNS rebinding attack          |
