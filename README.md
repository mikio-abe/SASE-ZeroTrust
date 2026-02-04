# SASE-ZeroTrust# SASE / Zero Trust

Secure Access Service Edge (SASE) implementation using Cloudflare Zero Trust, integrated with SD-WAN overlay.

---

## Overview

This component is part of an SD-WAN–integrated SASE lab.
It focuses on how traffic visibility and security enforcement points change
when traffic traverses encrypted tunnels.

The lab demonstrates Zero Trust security enforcement at the network edge, including:

- **Secure Web Gateway (SWG)** – Content filtering and TLS inspection
- **Zero Trust Network Access (ZTNA)** – Application-level access control
- **DNS Filtering** – Policy enforcement at DNS resolution

**【日本語サマリ】**  
Cloudflare Zero Trustを使用したSASE実装。  
SWG（コンテンツフィルタリング、TLS検査）、ZTNA（アプリケーションアクセス制御）、  
DNSフィルタリングによるゼロトラストセキュリティを検証。

---

## Architecture Position

```
┌──────────────────────────────────────────────────────┐
│                    SASE Layer                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │     SWG     │  │    ZTNA     │  │ DNS Filter  │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  │
├──────────────────────────────────────────────────────┤
│                   SD-WAN Overlay                     │
├──────────────────────────────────────────────────────┤
│                   MPLS Underlay                      │
└──────────────────────────────────────────────────────┘
```

This SASE component operates at the security layer, above the SD-WAN overlay and independent of the MPLS underlay.

Traffic inspection and policy enforcement occur after SD-WAN path selection but before application access,
clearly separating transport decisions from security controls.

**【日本語サマリ】**  
本コンポーネントはSD-WANの経路制御とは独立したセキュリティレイヤーとして動作し、
経路選択後の通信に対してポリシー制御を行います。

---

## Components

### Secure Web Gateway (SWG)

| Function | Implementation |
|----------|----------------|
| Content Filtering | Category-based blocking (Gambling, Adult, etc.) |
| TLS Inspection | Decrypt-inspect-re-encrypt for HTTPS traffic |
| HTTP Logging | Full visibility into allowed/blocked requests |

### Zero Trust Network Access (ZTNA)

| Function | Implementation |
|----------|----------------|
| Identity Integration | Service Token authentication for headless devices |
| Application Access | Per-app access policies |
| Device Posture | WARP client enrollment verification |

### DNS Filtering

| Function | Implementation |
|----------|----------------|
| Policy Enforcement | Block resolution for prohibited domains |
| Null Response | Returns 0.0.0.0 / :: for blocked queries |
| Logging | Full DNS query visibility in Gateway logs |

**【日本語サマリ】**
SWGはカテゴリベースのコンテンツフィルタリングとTLS Inspection、
ZTNAはService Token認証とアプリケーション単位のアクセス制御、
DNSフィルタリングは禁止ドメインへのNull応答（0.0.0.0/::）を実装。

---

## Traffic Flow

### WARP Client Connection
```
Endpoint (WARP Client)
       │
       │ UDP 443 (QUIC/MASQUE)
       ▼
Cloudflare Gateway (NRT - Tokyo)
       │
       ├─── DNS Policy Check
       ├─── HTTP Policy Check
       └─── TLS Inspection (if enabled)
       │
       ▼
   Internet / Private Resources
```

### WireGuard POP-to-POP Connection

POP1 and POP2 establish a WireGuard tunnel over the internet for site-to-site connectivity:
```
POP1 (Site A)                                    POP2 (Site B)
10.255.0.1                                       10.255.0.2
     │                                                │
     │◄────────── WireGuard Tunnel ──────────────────►│
     │            UDP 4960                            │
     │            (over Internet via WARP)            │
     │                                                │
     ▼                                                ▼
   ens5 ──► WARP ──► Cloudflare ◄─── WARP ◄─── ens5
                      Gateway
```

This WireGuard tunnel carries SD-WAN IPsec (ESP) traffic between FortiGate devices:
```
FG1 ─── IPsec ESP ─── POP1 ═══ WireGuard ═══ POP2 ─── IPsec ESP ─── FG2
        (10.0.0.1)              (Internet)              (10.0.1.1)
```

In production, Cloudflare Magic WAN would replace WireGuard for site interconnection.

```

### Tunnel Encapsulation

| Tunnel Type | Protocol | Port | Visibility |
|-------------|----------|------|------------|
| WARP/MASQUE | QUIC | UDP 443 | Encrypted from start |
| WireGuard | UDP | 4960 | POP-to-POP site connectivity |
| IPsec (ESP) | Protocol 50 | - | SD-WAN overlay (FG1-FG2) |

Unlike traditional TLS, QUIC/MASQUE encrypts immediately
- no visible ClientHello/ServerHello handshake in packet captures.

**【日本語サマリ】**
WARPクライアントはUDP 443（QUIC/MASQUE）でCloudflare Gatewayに接続し、DNS/HTTP/TLSポリシーを適用。
POP1-POP2間はWireGuard（UDP 4960）でサイト間接続し、その上でFG1-FG2間のSD-WAN IPsec（ESP）トラフィックを転送。
本番環境ではWireGuardの代わりにMagic WANを使用。

### Tunnel Encapsulation

| Tunnel Type | Protocol | Port | Visibility |
|-------------|----------|------|------------|
| WARP/MASQUE | QUIC | UDP 443 | Encrypted from start |

Unlike traditional TLS, QUIC/MASQUE encrypts immediately
- no visible ClientHello/ServerHello handshake in packet captures.

---

## Verification Results

### DNS Blocking Verification

**Test:** `curl https://www.bet365.com` (Gambling category)

**Result:**
```
Query: www.bet365.com A
Response: 0.0.0.0

Query: www.bet365.com AAAA  
Response: ::

Connection: Failed (no route to host)
```

DNS-level blocking prevents TCP connection establishment entirely.

### HTTP Policy Verification

| Host | Action | Notes |
|------|--------|-------|
| www.bet365.com | BLOCK | Gambling category |
| www.pornhub.com | BLOCK | Adult category |
| www.google.com | ALLOW | Permitted traffic |
| *.cloudflare-gateway.com | BYPASS | Internal communication |

### Gateway Logs

**[ 📷 ここに画像添付：Cloudflare Dashboard DNS/HTTP Logs ]**

Logs confirm:
- Device identity (non_identity@eve-lab.cloudflareaccess.com)
- Policy application (Block/Allow/Bypass)
- Timestamp and query details

**【日本語サマリ】**
DNSブロックはbet365.com等に対し0.0.0.0/::を返却しTCP接続を阻止。
HTTPポリシーでGambling/Adultカテゴリをブロック、Cloudflare内部通信はBYPASS。
Gateway Logsでデバイス識別・ポリシー適用を確認。

---

## Device Enrollment

### Service Token Authentication

For headless Linux devices (no browser-based auth):

| Parameter | Purpose |
|-----------|---------|
| auth_client_id | Service Token identifier |
| auth_client_secret | Authentication credential |

Configuration delivered via MDM file (`/var/lib/cloudflare-warp/mdm.xml`)

### Enrolled Devices

| Device | Registration | Organization |
|--------|--------------|--------------|
| CF-POP1 | Service Token | eve-lab |
| CF-POP2 | Service Token | eve-lab |

**【日本語サマリ】**
ヘッドレスLinuxデバイスはService Token認証を使用。MDMファイル経由でauth_client_id/secretを配布し、
eve-lab組織にCF-POP1/POP2として登録。

---

## Split Tunnel Configuration

### Challenge

WARP client routes all traffic through Cloudflare by default, including WireGuard endpoint IPs. This created a routing loop:
```
WireGuard Endpoint (106.73.26.0)
       │
       └─── Routed via WARP ───┐
                               │
       ┌───────────────────────┘
       │
       ▼
  WARP Tunnel (broken - endpoint unreachable)
```

### Solution

Added WireGuard endpoints to Split Tunnel exclusion list:

| Entry | Description |
|-------|-------------|
| 106.73.26.0/32 | POP1 WireGuard Endpoint |
| 49.109.0.0/16 | POP2 WireGuard Endpoint (docomo range) |

This ensures WireGuard traffic bypasses WARP and uses direct internet path.

**【日本語サマリ】**
WARPがWireGuard EndpointをCloudflare経由でルーティングし、ループが発生。
Split TunnelにEndpoint IP（106.73.26.0/32, 49.109.0.0/16）を除外登録して解決。

---

## Key Learnings

### SASE Visibility Model

Traditional network troubleshooting relies on packet inspection. SASE changes this:

| Traditional | SASE |
|-------------|------|
| Wireshark/tcpdump | Gateway Logs |
| SYN/ACK analysis | DNS/HTTP policy logs |
| Packet payload | Encrypted - not visible |

### Policy-First Design

SASE enforces policy at multiple layers:
1. **DNS** - Block resolution before connection
2. **HTTP** - Inspect and filter after decryption
3. **Network** - Split Tunnel controls routing

### Integration with SD-WAN

SASE path is one of multiple SD-WAN paths:
- MPLS path for enterprise traffic
- SASE path for internet-bound traffic
- Health-check determines active path

**【日本語サマリ】**
SASEではパケットキャプチャよりGateway Logs重視。DNS→HTTP→Networkの多層ポリシー適用。
SD-WANの複数パス（MPLS/SASE）の一つとして統合。

---

## Related Components

- [SD-WAN](../SD-WAN) - Path selection and failover
- [Brownout](../Brownout) - Quality degradation detection
- [Troubleshooting](../Troubleshooting) - Encrypted tunnel analysis

---

*Part of SASE × SD-WAN Verification Lab*
