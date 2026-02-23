# SASE / Zero Trust

Secure Access Service Edge (SASE) implementation using Cloudflare Zero Trust, integrated with SD-WAN overlay.

## TL;DR
- **Goal:** Verify SWG / DNS / ZTNA enforcement when traffic traverses encrypted tunnels
- **Design choice:** Same NAT, two POPs → POP1 uses **WARP** (full SWG), POP2 uses **cloudflared DoH** (DNS-only) to separate DNS Locations
- **Key finding:** Under QUIC/MASQUE, packet capture loses L7 visibility — Cloudflare Gateway Logs replace tcpdump as the primary verification tool

**【日本語サマリ】**<br>
暗号化トンネル環境でSWG/DNS/ZTNAの制御ポイントを検証しています。<br>
同一NAT配下の2拠点をWARP（フルSWG）とDoH（DNS専用）で分離し、拠点別ポリシーを実現しています。<br>
QUIC/MASQUE環境ではtcpdumpでL7が見えないため、Cloudflare Gateway Logsが主要な検証手段となることを確認しました。

---

## 🔬Overview

This component is part of an SD-WAN–integrated SASE lab.
It focuses on how traffic visibility and security enforcement points change
when traffic traverses encrypted tunnels.

The lab demonstrates Zero Trust security enforcement at the network edge, including:

- **Secure Web Gateway (SWG)** – Content filtering and TLS inspection
- **DNS Filtering** – Policy enforcement at DNS resolution
- **Zero Trust Network Access (ZTNA)** – Application-level access control

 <br>**【日本語サマリ】**  <br>
Cloudflare Zero Trustを使用したSASE実装です。
SWG（コンテンツフィルタリング、TLS検査）、DNSフィルタリング、ZTNA（アプリケーションアクセス制御）によるゼロトラストセキュリティを検証しています。

---

## 🏗Architecture Position
<img width="500" alt="image" src="https://github.com/user-attachments/assets/e91a20f1-3cf3-4629-a8a9-df8145b5517d" />

This SASE component operates at the security layer, above the SD-WAN overlay and independent of the MPLS underlay.

Traffic inspection and policy enforcement occur after SD-WAN path selection but before application access,
clearly separating transport decisions from security controls.

<br>**【日本語サマリ】**  <br>
本コンポーネントはSD-WANの経路制御とは独立したセキュリティレイヤーとして動作し、経路選択後の通信にポリシー制御を行います。

---

## 🧩Components

### 🔧Secure Web Gateway (SWG)

| Function | Implementation |
|----------|----------------|
| Content Filtering | Category-based blocking (Gambling, Adult, etc.) |
| TLS Inspection | Decrypt-inspect-re-encrypt for HTTPS traffic |
| HTTP Logging | Full visibility into allowed/blocked requests |

### 🔧DNS Filter

DNS policy is enforced per POP via separate DNS Locations (see [Gateway Connection Methods](#gateway-connection-methods) for details).

### 🔧Zero Trust Network Access (ZTNA)

| Function | Implementation |
|----------|----------------|
| Identity Integration | Service Token for headless devices, IdP for browser-based auth |
| Identity Providers | Auth0 (OpenID Connect), Entra ID (Azure AD) |
| Application Access | Per-app access policies |
| Device Posture | WARP client enrollment verification |

#### IdP Integration

![image](https://github.com/user-attachments/assets/e6e5c844-3b02-4318-bcd6-b99a4f5c9502)
**📷 Identity Provider integrations**

Configured IdPs:
- **Auth0** - OpenID Connect integration
- **Entra ID** - Azure AD integration
- **One-time PIN** - Fallback method

<br>**【日本語サマリ】**  <br>
SWGはカテゴリベースのコンテンツフィルタリングとTLS Inspectionを実装しています。
ZTNAはIdP連携（Auth0/Entra ID）によるブラウザ認証と、Service Token（ヘッドレスデバイス用）をサポートしています。

---

## 🔌Gateway Connection Methods

Cloudflare Gateway provides multiple connection methods — not just the WARP agent:

| Method | Protocol | Scope | Identification | Agent Required |
|--------|----------|-------|----------------|----------------|
| WARP Client | MASQUE/QUIC (UDP 443) | DNS + HTTP + All Traffic | Device enrollment | Yes |
| cloudflared DoH | HTTPS (TCP 443) | DNS only | DoH URL token | No (proxy process) |
| IPv4/IPv6 DNS | UDP 53 → 172.64.36.1 | DNS only | Source IP | No (agentless) |

### Why This Lab Uses Two Methods

Both POPs are behind the same NAT (106.73.26.0), so IPv4 DNS alone cannot distinguish them.
Different methods are used to assign separate DNS Locations:

| POP | Method | DNS Location | DoH Endpoint | Policy Scope |
|-----|--------|-------------|--------------|-------------|
| CF-POP1 | WARP | eve-lab | — (via WARP) | DNS + HTTP (full SWG) |
| CF-POP2 | cloudflared DoH | eve-lab-2 | `xx579sxsi4.cloudflare-gateway.com` | DNS only |

POP2 uses cloudflared as a local DoH proxy (port 53), forwarding queries to a dedicated Gateway endpoint.
This URL token identifies POP2 as eve-lab-2, enabling per-site policy separation under a single NAT.

<br>**【日本語サマリ】**  <br>
Cloudflare GatewayはWARP（フルSWG）、DoH（DNS専用）、IPv4 DNS（エージェントレス）の3方式を提供しています。
本ラボでは同一NAT配下の2拠点を区別するため、POP1はWARP、POP2はDoH URLトークンで別のDNS Locationに接続し、拠点別ポリシー適用とログ分離を実現しています。

---

## 📦Client Deployment & Device Enrollment

### Package Installation

Both POPs are headless Linux (Debian 12) in EVE-NG. Two packages installed via CLI:

```bash
# Add Cloudflare GPG key and repository
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | \
  gpg --yes --dearmor -o /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] \
  https://pkg.cloudflareclient.com/ bookworm main" \
  > /etc/apt/sources.list.d/cloudflare-client.list

apt update && apt install -y cloudflare-warp cloudflared
```

<br>**【日本語サマリ】**  <br>
ヘッドレスLinux（Debian 12）にCloudflareのGPGキーとリポジトリを追加し、WARPクライアント（全通信をGateway経由）とcloudflared（DNSのみGateway転送するDoHプロキシ）の2パッケージをインストールしています。

### TLS Certificate (for SWG TLS Inspection)

Cloudflare Gateway performs TLS decryption to inspect HTTPS traffic.
The Cloudflare Root CA must be trusted on the endpoint, or HTTPS connections will fail.

```bash
curl -o /usr/local/share/ca-certificates/cloudflare.crt \
  https://developers.cloudflare.com/cloudflare-one/static/Cloudflare_CA.crt
update-ca-certificates
```

<br>**【日本語サマリ】**  <br>
SWGのTLS Inspection（HTTPS復号検査）用にCloudflare Root CA証明書をインストールしています。
GatewayがHTTPS通信を復号→検査→再暗号化するため、エンドポイント側でこの証明書を信頼しないとHTTPS接続が証明書エラーで失敗します。

### Service Token Authentication

![image](https://github.com/user-attachments/assets/2f267a3f-40bc-466d-9fd2-5afe7d41a6b6)
**📷 Cloudflare Dashboard - Service Tokens**

For headless Linux devices (no browser-based auth):

| Parameter | Purpose |
|-----------|---------|
| auth_client_id | Service Token identifier |
| auth_client_secret | Authentication credential |

Configuration delivered via MDM file:

```xml
<!-- /var/lib/cloudflare-warp/mdm.xml -->
<dict>
  <key>organization</key>
  <string>eve-lab</string>
  <key>auth_client_id</key>
  <string>[Service Token Client ID].access</string>
  <key>auth_client_secret</key>
  <string>[Service Token Client Secret]</string>
</dict>
```

```bash
systemctl restart warp-svc
warp-cli connect
warp-cli status    # → "Connected" confirms enrollment
```

![image](https://github.com/user-attachments/assets/d73d655b-d77d-4078-b397-74cb01d06d18)
**📷 Cloudflare Dashboard - Enrolled Devices (CF-POP1 / CF-POP2)**

| Device | Registration | Organization |
|--------|--------------|--------------|
| CF-POP1 | Service Token | eve-lab |
| CF-POP2 | Service Token | eve-lab |

<br>**【日本語サマリ】**  <br>
ヘッドレスLinuxデバイスはService Token認証を使用しています。
MDMファイル（mdm.xml）経由でauth_client_id/secretを配布し、warp-cli connectでeve-lab組織にCF-POP1/POP2として登録しています。

### Port 53 Conflict Resolution

Three services compete for port 53 (DNS). Only one can bind at a time:

| Service | Role |
|---------|------|
| systemd-resolved | OS default DNS resolver |
| cloudflared proxy-dns | DoH forwarder to Gateway |
| WARP | SWG client (captures all DNS) |

Resolution applied:

| POP | Active (port 53) | Stopped | Reason |
|-----|-------------------|---------|--------|
| CF-POP1 | WARP | cloudflared, systemd-resolved | Full SWG (DNS + HTTP) |
| CF-POP2 | cloudflared DoH | systemd-resolved, WARP | DoH URL token for DNS Location identification |

```bash
# Stop systemd-resolved to free port 53
systemctl stop systemd-resolved
systemctl disable systemd-resolved
```

<br>**【日本語サマリ】**  <br>
systemd-resolved、cloudflared、WARPの3つがport 53を取り合い競合しました。
POP1はWARP（フルSWG）、POP2はcloudflared DoH（DNS Location識別用）がport 53を占有しています。
WARPとcloudflaredは同じポートを使用するため共存できません。

### WARP vs WireGuard Conflict (Split Tunnel)

WARP routes all traffic through Cloudflare by default.
This breaks the WireGuard tunnel between POP1 and POP2, because handshake UDP packets are also routed through Cloudflare instead of direct internet.

```
POP1 ──► WARP ──✕ Blocked ──✕ POP2 WireGuard endpoint
POP2 ──► WARP ──✕ Blocked ──✕ POP1 WireGuard endpoint
```

Split Tunnel (exclude mode) can bypass specific IPs from WARP.
However, POP2 uses a mobile carrier (docomo) whose global IP changes across wide ranges:

| Date | POP2 IP | Excluded Range |
|------|---------|----------------|
| 2026/01 | 49.109.x.x | 49.109.0.0/16 |
| 2026/02 | 1.73.x.x | 1.72.0.0/13 |

Excluding all possible mobile IP ranges is not practical.

**Lab policy:** WARP is kept OFF on POP2 permanently. SWG demonstration uses POP1 only (fixed IP: 106.73.x.x/32).

> **Note:** In production, this issue does not exist — firewalls connect to Cloudflare via Magic WAN directly, not through a WireGuard relay between POPs.

<br>**【日本語サマリ】**  <br>
WARPがデフォルトで全トラフィックをCloudflare経由にするため、POP間のWireGuard Endpointへの通信もWARP経由となりトンネル確立に失敗します。
Split TunnelでEndpoint IPを除外しましたが、POP2のモバイル回線（docomo）はグローバルIPが接続のたびに広範囲に変動するため、恒久対策は困難です。
ラボ運用方針としてPOP2のWARPは常時OFFとし、SWGデモはPOP1（固定IP）のみで実施しています。
本番環境ではMagic WANで直接Cloudflareに接続するため、この問題は発生しません。

---

## 🔀Traffic Flow

### WARP Client Connection
<img width="350" alt="image" src="https://github.com/user-attachments/assets/c0a5f931-05a6-452b-8ef0-c66ba1703697" />


### WireGuard POP-to-POP Connection

POP1 and POP2 establish a WireGuard tunnel over the internet for site-to-site connectivity:<BR>

<img width="520" alt="image" src="https://github.com/user-attachments/assets/1aa707cc-c8c7-4999-8b35-afb7f8accbac" />

<BR><BR>

This WireGuard tunnel carries SD-WAN IPsec (ESP) traffic between FortiGate devices:<BR>

<img width="600" alt="image" src="https://github.com/user-attachments/assets/b46679fc-15e4-4e9d-b7a7-94bbe4cf7415" />

<BR><BR>

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
WARPクライアントはUDP 443（QUIC/MASQUE）でCloudflare Gatewayに接続し、DNS/HTTP/TLSポリシーを適用しています。<br>
POP1-POP2間はWireGuard（UDP 4960）でサイト間接続し、その上でFG1-FG2間のSD-WAN IPsec（ESP）トラフィックを転送しています。<br>
本番環境では拠点間接続にBGP over IPsecを使用します。<br>
ラボではLinux POPでBGP over IPsecが設定できないためWireGuardで代替しています。

---

## ✅Verification Results

### ICMP Traffic Capture

📷 Wireshark ICMP Capture - SASE Path Traffic
<img width="1760" alt="image" src="https://github.com/user-attachments/assets/a32cc5c0-cd92-4a6d-9e53-716795002353" />

ICMP packets between POP1 (10.255.0.1) and POP2 (10.255.0.2) via WireGuard tunnel.<BR>
I/O Graph shows stable connectivity.

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

<img width="1724" height="1528" alt="image" src="https://github.com/user-attachments/assets/9ace8fa2-b4bd-483f-9b33-5d88b7f78668" />

**📷 Cloudflare Dashboard DNS/HTTP Logs**

Logs confirm:
- Device identity (non_identity@eve-lab.cloudflareaccess.com)
- Policy application (Block/Allow/Bypass)
- Timestamp and query details

 <br>**【日本語サマリ】**　<br>
WiresharkでSASEパス経由のICMPトラフィック（POP1-POP2間）とI/Oグラフを可視化しました。<br>
DNSブロックはbet365.com等に対し0.0.0.0/::を返却しTCP接続を阻止しています。<br>
HTTPポリシーでGambling/Adultカテゴリをブロック、Cloudflare内部通信はBYPASSしています。<br>
Gateway Logsでデバイス識別・ポリシー適用を確認しています。

---

## 🚀Key Learnings

### SASE Visibility Model

In encrypted environments (WARP/QUIC), traditional packet capture cannot see TCP handshakes or payload.

**Instead, use Cloudflare Gateway Logs for:**
- DNS query and response
- HTTP request and policy action
- Device identity and connection status

<br>**【日本語サマリ】**  <br>
WARP/QUIC環境ではtcpdumpでTCPハンドシェークやペイロードが見えません。<br>
代わりにCloudflare Gateway LogsでDNS/HTTPポリシー適用状況を確認します。

---

## 🔗Related Components

- [SD-WAN](https://github.com/mikio-abe/SD-WAN) - Path selection and failover
- [Troubleshooting](https://github.com/mikio-abe/Troubleshooting) - Encrypted tunnel analysis
- [Enterprise-SP](https://github.com/mikio-abe/Enterprise-SP) - Service provider configuration
- [Lab-vs-Production](https://github.com/mikio-abe/Lab-vs-Production) - Lab and production comparison

---

*Part of SASE × SD-WAN Verification Lab*
