# Network Labs

I'm an ICT student at FIEK, University of Prishtina. I built these two labs to practise networking hands-on instead of only learning it in theory:

1. **Branch–HQ network (Cisco Packet Tracer)**: I built and configured a small two-site network, tested it, then broke it on purpose and fixed it.
2. **Traffic analysis (Wireshark)**: I captured real traffic on my own home network to see DHCP, DNS, the TCP handshake and tracert in action.

---

## 1. Branch–HQ network (Cisco Packet Tracer)

A branch office connected to headquarters. Staff and guests share one switch at the branch but are kept in separate VLANs. The branch router gives out IP addresses and routes traffic to HQ, where a server provides DNS and a web page.

### Devices

- 2 × Cisco 1941 routers: `R-BRANCH`, `R-HQ`
- 2 × Cisco 2960 switches: `SW-BRANCH`, `SW-HQ`
- 3 × PCs at the branch: `PC1`, `PC2` (staff), `PC3` (guest)
- 1 × server at HQ: `HQ-SERVER`

### IP plan

| Network | Subnet | Gateway | Addressing |
|---|---|---|---|
| Staff (VLAN 10) | 192.168.10.0/24 | 192.168.10.1 | DHCP from R-BRANCH |
| Guest (VLAN 20) | 192.168.20.0/24 | 192.168.20.1 | DHCP from R-BRANCH |
| Branch–HQ link | 10.0.0.0/30 | – | R-BRANCH 10.0.0.1, R-HQ 10.0.0.2 |
| HQ LAN | 192.168.100.0/24 | 192.168.100.1 | HQ-SERVER 192.168.100.10 (static) |

### What I configured

- **VLANs on SW-BRANCH:** VLAN 10 (STAFF) on Fa0/1–2, VLAN 20 (GUEST) on Fa0/3, and a trunk on Gig0/1 to the router.
- **Inter-VLAN routing (router-on-a-stick) on R-BRANCH:** subinterfaces Gig0/0.10 and Gig0/0.20 with 802.1Q tags, one gateway per VLAN.
- **DHCP on R-BRANCH:** one pool per VLAN, giving out the address, gateway and DNS server. Addresses .1–.9 are excluded.
- **Static routing:** R-BRANCH has a route to the HQ network; R-HQ has routes back to both branch VLANs.
- **HQ-SERVER:** static IP, DNS service with an A record for `portal.hq.local`, and HTTP.

Running configs: [R-BRANCH](packet-tracer-lab/R-BRANCH-config.txt) · [R-HQ](packet-tracer-lab/R-HQ-config.txt) · [SW-BRANCH](packet-tracer-lab/SW-BRANCH-config.txt) · Packet Tracer file: [branch-hq-network.pkt](packet-tracer-lab/branch-hq-network.pkt)

### Testing

The PCs got their addresses from DHCP (PC1: 192.168.10.10, PC3: 192.168.20.10). From PC1 I tested:

| Test | Result |
|---|---|
| `ping 192.168.10.1` (own gateway) | Replies, TTL=255 |
| `ping 192.168.20.10` (PC3, guest VLAN) | Replies, TTL=127, so inter-VLAN routing works |
| `ping 192.168.100.10` (HQ server) | Replies, TTL=126, so routing to HQ works both ways |
| `ping portal.hq.local` | Resolved to 192.168.100.10, so DNS works |
| `tracert 192.168.100.10` | 192.168.10.1 → 10.0.0.2 → 192.168.100.10 |
| Web browser: `http://portal.hq.local` | Page loads from HQ-SERVER |

The TTL values show how many routers the reply passed through: PCs start at 128, so 127 means one router and 126 means two.

### Troubleshooting exercises

I broke the network on purpose, then found and fixed each problem. Screenshots of every step are in the [packet-tracer-lab](packet-tracer-lab/) folder.

**1. Missing return route**
- **Problem:** I removed R-HQ's route back to the staff network (192.168.10.0/24).
- **Symptom:** PC1 could not ping the HQ server, and tracert stopped after the first hop.
- **Diagnosis:** `show ip route` on R-HQ showed a route to 192.168.20.0 but none to 192.168.10.0. The packets reached HQ, but the replies had no way back.
- **Fix:** added the static route again.
- **Lesson:** routing has to work in both directions.

**2. Port in the wrong VLAN**
- **Problem:** I moved PC1's switch port (Fa0/1) into VLAN 20.
- **Symptom:** PC1 got a guest address (192.168.20.11) even though it is a staff PC. DHCP still said "successful", so nothing looked broken to the user.
- **Diagnosis:** `show vlan brief` showed Fa0/1 under GUEST.
- **Fix:** moved Fa0/1 back to VLAN 10.
- **Lesson:** the VLAN on the switch port decides which network a device ends up in.

**Bonus: PC kept its old address**
- After fixing the VLAN, PC1 still could not reach anything.
- `ipconfig` showed it was still using its old guest address (192.168.20.11, gateway 192.168.20.1) while sitting in the staff VLAN, so it could not reach its gateway.
- **Fix:** `ipconfig /renew`. PC1 got 192.168.10.10 again and everything worked.

**3. DNS failure**
- **Problem:** I deleted the `portal.hq.local` record on the DNS server.
- **Symptom:** `ping 192.168.100.10` worked, but `ping portal.hq.local` failed with "could not find host".
- **Diagnosis:** working by IP but not by name means the network is fine and the problem is DNS. The DNS record table on the server was empty.
- **Fix:** added the A record again.
- **Lesson:** "works by IP, not by name" points to DNS.

---

## 2. Traffic analysis (Wireshark)

Captured on my home Wi-Fi (FRITZ!Box router at 192.168.178.1) using Wireshark and the Windows command prompt. Screenshots are in the [wireshark-lab](wireshark-lab/) folder.

### DHCP
Commands: `ipconfig /release`, `ipconfig /renew` · Filter: `dhcp`

My laptop got its address in four steps: Discover, Offer, Request, ACK. Because it had no IP yet, it sent from 0.0.0.0 to the broadcast address 255.255.255.255. The router replied (UDP port 67 to 68) with IP 192.168.178.138, mask 255.255.255.0, gateway and DNS 192.168.178.1, and a 10-day lease. A later Request/ACK pair sent directly to the router was a lease renewal.

### DNS
Commands: `ipconfig /flushdns`, `nslookup example.com` · Filter: `dns.qry.name == "example.com"`

I cleared the DNS cache so my laptop had to ask again. It sent an A query over UDP port 53 to the router and got two IPv4 addresses back in about 14 ms (104.20.23.154 and 172.66.147.243). The query and response had the same transaction ID (0x0004). A second query asked for the IPv6 (AAAA) addresses.

### TCP three-way handshake
Command: `curl -4 -I https://example.com` · Filter: `ip.addr == 104.20.23.154 || ip.addr == 172.66.147.243`

The connection opened with SYN, SYN-ACK, ACK between my laptop (random port 52038) and the server (port 443). In the first packet only the SYN flag is set. After the handshake, a TLS 1.3 Client Hello started the encryption, and the connection was closed with FIN/ACK at the end.

### tracert
Command: `tracert -d 8.8.8.8` · Filter: `icmp`

tracert reached 8.8.8.8 in 12 hops (about 26 ms). It sends packets with TTL 1, 2, 3 and so on; each router that drops one replies "Time-to-live exceeded", which shows its address. Hop 1 was my router, the next hops were my internet provider's network, and the last hops were Google's network. Hops shown as `* * *` did not reply. In Wireshark, the TTL=1 packets got their reply from 192.168.178.1 and the TTL=2 packets from 10.178.0.1, matching the tracert output.

---

## Tools

Cisco Packet Tracer · Cisco IOS CLI · Wireshark · Windows command prompt (ping, ipconfig, tracert, nslookup, curl)
