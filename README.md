# DNS Zone Transfer (AXFR) Lab

## Overview
Demonstrated a DNS Zone Transfer (AXFR) attack against a deliberately misconfigured BIND9 server. Zone transfers are a legitimate DNS replication mechanism between primary and secondary servers, but when left open, they allow anyone to dump the entire DNS zone.

## Scenario
Simulated an internal engagement where a DNS server was misconfigured to allow AXFR requests from any source. The goal was to extract the full zone file - including hidden records that would not appear in normal forward or reverse enumeration.

## Lab Environment
- Scanner: Kali Linux
- Target DNS server: 127.0.0.1 (local BIND9 instance simulating 192.168.130.53)
- Target zone: dev.insecurecorp.org

### The Misconfiguration
The BIND9 zone configuration included:
This is the vulnerability. In a properly hardened DNS server, this should be restricted to known secondary DNS server IPs only.

## Command Used
dig dev.insecurecorp.org @127.0.0.1 axfr > zone_transfer.txt

## Results - Full Zone Dump
dev.insecurecorp.org.   IN  SOA  ns1.dev.insecurecorp.org. admin.dev.insecurecorp.org.
dev.insecurecorp.org.   IN  NS   ns1.dev.insecurecorp.org.
db.dev.insecurecorp.org.            IN  A  192.168.130.30
dev.dev.insecurecorp.org.           IN  A  192.168.130.20
ns1.dev.insecurecorp.org.           IN  A  192.168.130.53
old-db.dev.insecurecorp.org.        IN  A  192.168.130.60
superhiddenrecord.dev.insecurecorp.org. IN  A  192.168.130.99
vpn.dev.insecurecorp.org.           IN  A  192.168.130.50
www.dev.insecurecorp.org.           IN  A  192.168.130.10

XFR size: 10 records (messages 1, bytes 325)

## Key Findings

### Hidden Host Discovered
`superhiddenrecord.dev.insecurecorp.org -> 192.168.130.99`

This record would not appear in:
- Forward DNS brute-forcing with common wordlists (no dictionary contains "superhiddenrecord")
- Reverse DNS sweeps (the PTR isn't published)
- Standard DNS queries (nobody would guess the name)

**Zone transfers bypass all of these limitations** because they dump the entire zone in one shot.

### Other Sensitive Records Exposed
- `old-db.dev.insecurecorp.org` - legacy/backup database (potential vulnerable target)
- `vpn.dev.insecurecorp.org` - VPN endpoint (credential attack surface)
- `db.dev.insecurecorp.org` - primary database
- `ns1.dev.insecurecorp.org` - name server (reveals network structure)

### Naming Convention Leaked
The environment structure is fully mapped: prod hosts use simple names (www, db, vpn), legacy hosts use prefixes (old-db), and internal hosts use explicit names (superhiddenrecord). This informs downstream attacks.

## Why Zone Transfers Are Dangerous
1. **Complete infrastructure map** in one query - no need to brute-force.
2. **Exposes legacy and forgotten hosts** that aren't actively maintained.
3. **Reveals internal naming conventions**, which enables targeted guessing for other zones.
4. **Reveals the domain's DNS infrastructure** (SOA, NS records), which maps out the network.

## Why It Still Happens
- Admins assume "internal DNS is only reachable internally" and don't harden it.
- Legacy deployments still using default `allow-transfer { any; }`.
- Migration oversights when DNS is moved to cloud/new infrastructure.
- Missing ACLs on multi-tenant or shared DNS hosts.

## Key Takeaways
1. Open zone transfers are still occasionally found in production - usually in internal or legacy environments.
2. A single AXFR request reveals more than days of brute-forcing.
3. If you find one during an engagement, check the surrounding IP ranges - misconfigured DNS servers often cluster.
4. **Defensive**: Restrict `allow-transfer` to specific secondary server IPs, or disable it entirely if not using secondaries.

## Evidence
- zone_transfer.txt - Raw AXFR output

## Tools Used
- dig
- BIND9 (test DNS server)
