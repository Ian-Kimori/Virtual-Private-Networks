# Virtual Private Networks — Complete Technical Reference

> A complete breakdown of VPN architecture, all major protocols, and how types and mechanisms relate — end to end.

---

## What a VPN actually is

A VPN (Virtual Private Network) creates an **encrypted tunnel** between two endpoints over an untrusted network (typically the internet). The core job is three things:

- **Encapsulation** — wrapping your traffic in a new packet
- **Encryption** — making the payload unreadable to outsiders
- **Authentication** — proving both sides are who they claim to be

The "virtual" part is key — you're building a logical private link over public infrastructure. The "private" part means confidentiality, integrity, and (usually) anonymity from the transport layer's perspective.

---

## The two axes: Types vs Protocols

Most people confuse these. Here is the distinction:

- **VPN type** = the *architecture* — what two things are being connected, and why
- **VPN protocol** = the *mechanism* — how the tunnel is built, secured, and maintained

The type answers *"what are we connecting?"*. The protocol answers *"how do we build the tunnel?"*. Any type can theoretically use multiple protocols.

---

## Landscape overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VPN TYPES (architecture)                         │
│                                                                     │
│  ┌───────────────┐  ┌───────────────┐  ┌──────────────────────┐   │
│  │ Remote access │  │ Site-to-site  │  │   MPLS / Cloud /     │   │
│  │ User→network  │  │  Net ↔ Net   │  │   Mesh / Zero Trust  │   │
│  └───────────────┘  └───────────────┘  └──────────────────────┘   │
│                                                                     │
└─────────────────┬───────────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│          Encrypted tunnel — the "virtual private" part              │
└─────────────────┬───────────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                VPN PROTOCOLS (mechanism)                            │
│                                                                     │
│  Layer 3 / network:                                                 │
│  ┌────────┐  ┌─────┐  ┌────────────┐  ┌───────────┐              │
│  │ IPsec  │  │ GRE │  │ L2TP/IPsec │  │ WireGuard │              │
│  │IKEv2+  │  │Enc. │  │L2 over L3  │  │ ChaCha20  │              │
│  │ESP/AH  │  │only │  │tunnel      │  │ kernel-   │              │
│  └────────┘  └─────┘  └────────────┘  │ native    │              │
│                                        └───────────┘              │
│  Layer 4–7 / application:                                          │
│  ┌──────────┐  ┌─────────────┐  ┌──────┐  ┌──────────────────┐  │
│  │ OpenVPN  │  │  SSL/TLS    │  │ SSTP │  │ PPTP             │  │
│  │ TLS/UDP  │  │  VPN        │  │ :443 │  │ ⚠ DEPRECATED    │  │
│  │ or TCP   │  │  (browser)  │  │      │  │ DO NOT USE       │  │
│  └──────────┘  └─────────────┘  └──────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## VPN Types — the architecture

### 1. Remote Access VPN

The most common consumer and enterprise use case. A **single user's device** connects to a central VPN gateway (server). All traffic is tunneled through that gateway before reaching the internet or corporate resources.

**Examples:** Employee working from home connecting to office systems; consumer using NordVPN/Mullvad.

**How it works:** Client software initiates a tunnel to a VPN concentrator. The user gets an IP address from the corporate/VPN subnet. Traffic is decrypted at the gateway and forwarded onward.

---

### 2. Site-to-Site VPN

Two entire **networks** are connected together — neither end user does anything. The routers/firewalls at each site handle the tunnel automatically.

**Examples:** Enterprise linking branch offices to HQ; connecting two datacentres.

**How it works:** Edge devices (firewalls, routers) at each site negotiate the tunnel between themselves. Traffic between the two networks is encrypted in transit but users experience it as a flat internal network.

---

### 3. Client-to-Site

Often used interchangeably with remote access, but more precisely describes a **device** (not a full network) connecting to a corporate **gateway**. The distinction is that it's one endpoint, not two networks, and it's usually policy-driven: split tunneling, MFA, device posture checks.

---

### 4. MPLS VPN (Provider-managed)

A **telecom carrier** provides the private WAN — you never touch the public internet. The provider's MPLS backbone separates your traffic from other customers using VRFs (Virtual Routing and Forwarding tables).

**Examples:** Financial services WAN, healthcare provider networks.

Expensive but very reliable and low-latency.

---

### 5. Cloud / Mesh / SD-WAN VPN

SD-WAN and cloud-native VPNs (AWS VPN Gateway, Azure VPN Gateway, Cloudflare WARP, Tailscale). The "VPN" here is often a mesh — each node has a tunnel to every other node, routing handled by software rather than dedicated hardware.

**Examples:** Tailscale builds a WireGuard mesh between all your devices automatically.

---

## Packet encapsulation — what actually happens

```
BEFORE VPN (original packet):
┌─────────────┬─────────────┬──────────────────────────┐
│  IP header  │  TCP / UDP  │        Payload           │
└─────────────┴─────────────┴──────────────────────────┘

                    ▼ IPsec tunnel mode wraps everything

AFTER ENCAPSULATION (what the internet sees):
┌────────────┬──────────┬─────────────────────────────────────────────────────┬──────────┐
│ New IP hdr │ ESP hdr  │          ████████ ENCRYPTED ████████                │ ESP trl  │
│ Gateway IPs│ SPI+seq  │  Orig IP │ TCP/UDP │     Payload (data)             │ +auth tag│
└────────────┴──────────┴──────────┴─────────┴────────────────────────────────┴──────────┘
      │                  └──────────────────────────────────────────────┘         │
      │                        Nobody can read anything in here                   │
      ▼                                                                           ▼
ISP sees only:                                                         Integrity-protected
Gateway A → Gateway B

At VPN gateway: outer header stripped, inner packet decrypted, forwarded normally
```

---

## VPN Protocols — the mechanism

### IPsec (Internet Protocol Security) ✅ Recommended

The gold standard for network-layer VPNs. IPsec is a **suite** of sub-protocols:

| Sub-protocol | Role |
|---|---|
| **IKE / IKEv2** | Authentication and key negotiation. Phase 1: secure channel via DH + certs/PSK. Phase 2: data channel parameters. |
| **ESP** (Encapsulating Security Payload) | Encryption, integrity, anti-replay. Uses AES-GCM. The workhorse. |
| **AH** (Authentication Header) | Integrity only, no encryption. Rarely used — breaks NAT, ESP covers everything AH does. |

**Two modes:**

- **Transport mode** — encrypts only the payload, keeps original IP header. Used for host-to-host.
- **Tunnel mode** — wraps the entire original packet in a new one. Used for gateway-to-gateway (the standard VPN scenario).

**Used in:** StrongSwan, Cisco ASA, Palo Alto, AWS VPN Gateway, Azure VPN.

---

### GRE (Generic Routing Encapsulation)

GRE alone provides **encapsulation with zero encryption** — it wraps packets in a tunnel header so you can route across incompatible networks or carry multicast/broadcast traffic (which IPsec can't carry natively).

In practice, GRE is almost always paired with IPsec: GRE handles routing flexibility, IPsec handles encryption. Written as GRE-over-IPsec or IPsec-over-GRE (order matters for how crypto is applied).

---

### L2TP/IPsec (Layer 2 Tunneling Protocol)

L2TP creates a **Layer 2 tunnel** — it encapsulates PPP frames, allowing it to carry non-IP traffic and assign IP addresses to remote clients exactly like a dial-up PPP session.

L2TP has zero encryption by itself, so it's always paired with IPsec.

This was the dominant Windows/mobile VPN protocol through the 2000s–2010s. It is being phased out because it double-encapsulates (L2TP inside IPsec), adding overhead with no performance advantage over IKEv2/IPsec alone.

---

### WireGuard ✅ Modern

The newest and architecturally cleanest protocol. Operates in the Linux kernel (since 5.6).

| Property | Detail |
|---|---|
| Encryption | ChaCha20-Poly1305 (fixed, no negotiation — eliminates downgrade attacks) |
| Key exchange | Curve25519 |
| Hashing | BLAKE2s |
| Codebase | ~4,000 lines (vs OpenVPN ~100,000) — dramatically smaller attack surface |
| Authentication | Public key only — peers pre-configure each other's public keys (no CA) |
| Interface | Creates a virtual NIC (e.g. `wg0`), routing via standard kernel tables |

**Used in:** Tailscale, Mullvad, ProtonVPN (all use WireGuard under the hood).

> **Caveat:** WireGuard doesn't hide that you're using WireGuard (recognizable UDP patterns). By design, the server logs your IP indefinitely until the session expires — Mullvad patches around this with rotating IPs.

---

### OpenVPN ✅ Battle-tested

A mature protocol built on **OpenSSL/TLS**. Runs over UDP (faster, preferred) or TCP (more firewall-friendly). Over TCP port 443 it can disguise itself as HTTPS traffic — very useful in censored environments.

**Authentication:** Certificate authority model — server has a cert, clients have certs, both verify each other. Also supports TLS-auth (HMAC pre-shared key on top of TLS — a DoS defense).

Slower than WireGuard due to userspace implementation, though DCO (Data Channel Offload) now moves part of it into the kernel.

---

### SSL/TLS VPN (clientless)

Not the same as OpenVPN. A clientless SSL VPN is typically a **web portal** — you browse to `https://vpn.company.com`, authenticate, and get access to specific internal web apps or RDP sessions through the browser. No client software needed.

**Examples:** Cisco AnyConnect, Pulse Secure, Citrix.

---

### SSTP (Secure Socket Tunneling Protocol)

Microsoft's SSL-based protocol. Tunnels PPP traffic inside HTTPS (port 443), making it very firewall-friendly. Windows-native but practically a Microsoft-only ecosystem — rare in non-Windows environments.

---

### PPTP (Point-to-Point Tunneling Protocol) ⛔ Deprecated

> **Do not use under any circumstances.**

MS-CHAPv2, its authentication protocol, is completely broken — offline cracking in hours with commodity hardware. NSA has been known to decrypt it passively at scale. The only reason it still exists is legacy compatibility in ancient equipment.

---

## Protocol × Type matrix

| Protocol | Remote access | Site-to-site | Cloud / mesh | Clientless |
|---|---|---|---|---|
| **IPsec IKEv2** | ✅ Best choice | ✅ Best choice | ✅ Common (AWS) | — |
| **WireGuard** | ✅ Modern clients | ✅ Growing | ✅ Tailscale/mesh | — |
| **OpenVPN** | ✅ Very common | ✅ Supported | ⚪ Less common | — |
| **L2TP/IPsec** | ⚠ Legacy only | ⚪ Rare | ✗ Not used | — |
| **SSL/TLS VPN** | ✅ Portal access | — | ✅ ZTNA / SASE | ✅ Primary use |
| **PPTP** | ⛔ Avoid | ⛔ Avoid | ✗ Not used | — |

---

## Key concepts that tie it all together

### Split tunneling vs full tunnel

**Full tunnel:** All traffic from the client goes through the VPN — including YouTube, Netflix, everything.

**Split tunnel:** Only traffic destined for specific subnets (e.g. `10.0.0.0/8`) goes through the VPN; everything else exits directly to the internet. Saves bandwidth on the gateway, but an attacker on your local network can still intercept non-tunneled traffic.

---

### Perfect Forward Secrecy (PFS)

Each session negotiates a fresh ephemeral Diffie-Hellman key pair. If the long-term private key is ever compromised, past sessions remain protected because those session keys were never stored.

**Support:** IKEv2, WireGuard, OpenVPN all support PFS. PPTP does not.

---

### MTU and fragmentation

Adding VPN headers reduces usable packet size. A typical Ethernet MTU is 1500 bytes; IPsec ESP adds ~50–70 bytes of overhead. If you don't configure the MSS (Maximum Segment Size) correctly on the VPN interface, you'll get silent packet drops and degraded TCP performance. WireGuard handles this better than most.

---

### NAT traversal

IPsec was designed before NAT was pervasive. AH breaks when a NAT device changes the IP header (because AH protects it). The fix is **NAT-T** (NAT Traversal) — wrapping IPsec ESP packets inside UDP port 4500, which NAT devices can handle. IKEv2 detects NAT automatically and switches to UDP 4500. WireGuard is UDP-native and traverses NAT cleanly by design.

---

### Zero Trust Network Access (ZTNA) — the successor

Traditional VPNs grant broad network access once authenticated: you're in, you can reach anything. ZTNA (Cloudflare Access, Zscaler, Palo Alto Prisma) inverts this — access is granted per-application, per-session, based on identity, device posture, and context. The "VPN" becomes a per-request proxy rather than a persistent tunnel.

---

## Decision framework

| Scenario | Recommendation |
|---|---|
| Enterprise site-to-site | IPsec IKEv2 — AES-256-GCM + SHA-384 |
| Modern remote access (endpoint VPN) | WireGuard or IKEv2/IPsec |
| Censorship circumvention / firewall bypass | OpenVPN over TCP 443 or obfuscated WireGuard |
| Legacy Windows-only environment | SSTP or IKEv2 |
| Personal device mesh (home lab, team) | Tailscale (WireGuard underneath) |
| Web app access without installing a client | SSL/TLS VPN portal |
| Someone recommends PPTP | Hard no. |

---

## Throughput order

Fastest → slowest on modern hardware at the same cipher strength:

```
WireGuard  >  IPsec (hw offload)  >  OpenVPN (DCO)  >  OpenVPN (userspace)  >  L2TP/IPsec
```

WireGuard saturates gigabit links on a Raspberry Pi. OpenVPN struggles on the same hardware.

---

*VPN Technical Reference — IPsec · WireGuard · OpenVPN · L2TP · SSL/TLS · PPTP*
