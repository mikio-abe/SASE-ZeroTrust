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

 <br>**【日本語サマリ】**  <br>
Cloudflare Zero Trustを使用したSASE実装。 
SWG（コンテンツフィルタリング、TLS検査）、ZTNA（アプリケーションアクセス制御）、DNSフィルタリングによるゼロトラストセキュリティを検証。

---

## Architecture Position

```
┌──────────────────────────────────────────────────────┐
│                    SASE Layer                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │     SWG     │  │    ZTNA     │  │ DNS Filter  │   │
│  └─────────────┘  └─────────────┘  └─────────────┘   │
├──────────────────────────────────────────────────────┤
│                   SD-WAN Overlay                     │
├──────────────────────────────────────────────────────┤
│                   MPLS Underlay                      │
└──────────────────────────────────────────────────────┘
```

This SASE component operates at the security layer, above the SD-WAN overlay and independent of the MPLS underlay.

Traffic inspection and policy enforcement occur after SD-WAN path selection but before application access,
clearly separating transport decisions from security controls.

<br>　**【日本語サマリ】**  <br>
本コンポーネントはSD-WANの経路制御とは独立したセキュリティレイヤーとして動作し、経路選択後の通信にポリシー制御を行います。

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
| Identity Integration | Service Token for headless devices, IdP for browser-based auth |
| Identity Providers | Auth0 (OpenID Connect), Entra ID (Azure AD) |
| Application Access | Per-app access policies |
| Device Posture | WARP client enrollment verification |

**<img width="1716" height="669" alt="image" src="https://github.com/user-attachments/assets/e6e5c844-3b02-4318-bcd6-b99a4f5c9502" />
📷 Identity Provider integrations**

Configured IdPs:
- **Auth0** - OpenID Connect integration
- **Entra ID** - Azure AD integration
- **One-time PIN** - Fallback method

### DNS Locations

| Location | Device | DoH Endpoint |
|----------|--------|--------------|
| eve-lab | CF-POP1 | https://xx579sxsi4.cloudflare-gateway.com/dns-query |
| eve-lab-2 | CF-POP2 | (別エンドポイント) |

Each POP uses a dedicated DNS location for policy enforcement and logging separation.

<br>　**【日本語サマリ】**　<br>

SWGはカテゴリベースのコンテンツフィルタリングとTLS Inspection。ZTNAはService Token（ヘッドレスデバイス用）とIdP連携（Auth0/Entra ID）によるブラウザ認証をサポート。<br>
DNSフィルタリングは禁止ドメインへのNull応答（0.0.0.0/::）を実装。<br>
POP1とPOP2で別々のDNS Location（eve-lab, eve-lab-2）を設定し、それぞれ固有のDoHエンドポイントを使用。デバイス別のポリシー適用とログ分離が可能。

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

In production, site-to-site connectivity would use BGP over IPsec configured directly on edge routers. 
WireGuard is used in this lab as an alternative because the POP devices (Linux) cannot run BGP over IPsec natively.

### Tunnel Encapsulation

| Tunnel Type | Protocol | Port | Visibility |
|-------------|----------|------|------------|
| WARP/MASQUE | QUIC | UDP 443 | Encrypted from start |
| WireGuard | UDP | 4960 | POP-to-POP site connectivity |
| IPsec (ESP) | Protocol 50 | - | SD-WAN overlay (FG1-FG2) |

Unlike traditional TLS, QUIC/MASQUE encrypts immediately - no visible ClientHello/ServerHello handshake in packet captures.

 <br>**【日本語サマリ】**　<br>
WARPクライアントはUDP 443（QUIC/MASQUE）でCloudflare Gatewayに接続し、DNS/HTTP/TLSポリシーを適用。<br>
POP1-POP2間はWireGuard（UDP 4960）でサイト間接続し、その上でFG1-FG2間のSD-WAN IPsec（ESP）トラフィックを転送。<br>
本番環境では拠点間接続にBGP over IPsecを使用。ラボではLinux POPでBGP over IPsecが設定できないためWireGuardで代替。

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

**<img width="1724" height="1528" alt="image" src="https://github.com/user-attachments/assets/9ace8fa2-b4bd-483f-9b33-5d88b7f78668" />
📷Cloudflare Dashboard DNS/HTTP Logs ]**

Logs confirm:
- Device identity (non_identity@eve-lab.cloudflareaccess.com)
- Policy application (Block/Allow/Bypass)
- Timestamp and query details

 <br>**【日本語サマリ】**　<br>
DNSブロックはbet365.com等に対し0.0.0.0/::を返却しTCP接続を阻止。
HTTPポリシーでGambling/Adultカテゴリをブロック、Cloudflare内部通信はBYPASS。
Gateway Logsでデバイス識別・ポリシー適用を確認。

---

## Device Enrollment

### Service Token Authentication

### Service Token Authentication

**<img width="1708" height="627" alt="image" src="https://github.com/user-attachments/assets/2f267a3f-40bc-466d-9fd2-5afe7d41a6b6" />
📷 Cloudflare Dashboard - Service Tokens**

For headless Linux devices (no browser-based auth):

| Parameter | Purpose |
|-----------|---------|
| auth_client_id | Service Token identifier |
| auth_client_secret | Authentication credential |

Configuration delivered via MDM file (`/var/lib/cloudflare-warp/mdm.xml`)

### Enrolled Devices

**<img width="1568" height="662" alt="image" src="https://github.com/user-attachments/assets/d73d655b-d77d-4078-b397-74cb01d06d18" />
📷 Cloudflare Dashboard - Devices**

| Device | Registration | Organization |
|--------|--------------|--------------|
| CF-POP1 | Service Token | eve-lab |
| CF-POP2 | Service Token | eve-lab |

**【日本語サマリ】**

ヘッドレスLinuxデバイスはService Token認証を使用。MDMファイル経由でauth_client_id/secretを配布し、eve-lab組織にCF-POP1/POP2として登録。

---

## Split Tunnel Configuration

### Challenge

WARP client routes all traffic through Cloudflare by default. This caused WARP to block direct communication to each other's WireGuard endpoints.

**Problem:**
- POP1's WARP blocks traffic to POP2's WireGuard endpoint (49.109.x.x)
- POP2's WARP blocks traffic to POP1's WireGuard endpoint (106.73.26.x)
- WireGuard tunnel cannot be established
```
POP1 ──► WARP ──✕ Blocked ──✕ POP2 WireGuard endpoint
POP2 ──► WARP ──✕ Blocked ──✕ POP1 WireGuard endpoint
```

### Solution

Added WireGuard endpoint IPs to Split Tunnel exclusion list:

| Entry | Description |
|-------|-------------|
| 106.73.26.0/32 | POP1 WireGuard Endpoint |
| 49.109.0.0/16 | POP2 WireGuard Endpoint (docomo range) |

**Result:** WireGuard traffic bypasses WARP and connects directly.
```
POP1 ──► Direct Internet ──► POP2 WireGuard endpoint ✓
POP2 ──► Direct Internet ──► POP1 WireGuard endpoint ✓
```

**【日本語サマリ】**
WARPがデフォルトで全トラフィックをCloudflare経由にするため、お互いのWireGuard Endpointへの通信がブロックされトンネル確立に失敗。<br>
Split TunnelでEndpoint IP（106.73.26.0/32, 49.109.0.0/16）を除外し、直接インターネット経由で接続することで解決。

---

## Key Learnings

### SASE Visibility Model

In encrypted environments (WARP/QUIC), traditional packet capture cannot see TCP handshakes or payload.

**Instead, use Cloudflare Gateway Logs for:**
- DNS query and response
- HTTP request and policy action
- Device identity and connection status

**【日本語サマリ】**

WARP/QUIC環境ではtcpdumpでTCPハンドシェークやペイロードが見えない。
代わりにCloudflare Gateway LogsでDNS/HTTPポリシー適用状況を確認。

---

## Related Components

- [SD-WAN](../SD-WAN) - Path selection and failover
- [Brownout](../Brownout) - Quality degradation detection
- [Troubleshooting](../Troubleshooting) - Encrypted tunnel analysis

---

*Part of SASE × SD-WAN Verification Lab*
