# SASE-ZeroTrust# SASE / Zero Trust

Secure Access Service Edge (SASE) implementation using Cloudflare Zero Trust, integrated with SD-WAN overlay.

---

## 🔬Overview

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

## 🏗Architecture Position
<img width="500" alt="image" src="https://github.com/user-attachments/assets/e91a20f1-3cf3-4629-a8a9-df8145b5517d" />

This SASE component operates at the security layer, above the SD-WAN overlay and independent of the MPLS underlay.

Traffic inspection and policy enforcement occur after SD-WAN path selection but before application access,
clearly separating transport decisions from security controls.

<br>　**【日本語サマリ】**  <br>
本コンポーネントはSD-WANの経路制御とは独立したセキュリティレイヤーとして動作し、経路選択後の通信にポリシー制御を行います。

---

## 🧩Components

### 🔧Secure Web Gateway (SWG)

| Function | Implementation |
|----------|----------------|
| Content Filtering | Category-based blocking (Gambling, Adult, etc.) |
| TLS Inspection | Decrypt-inspect-re-encrypt for HTTPS traffic |
| HTTP Logging | Full visibility into allowed/blocked requests |

### 🔧Zero Trust Network Access (ZTNA)

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

### 🔧DNS Locations

| Location | Device | DoH Endpoint |
|----------|--------|--------------|
| eve-lab | CF-POP1 | https://xx579sxsi4.cloudflare-gateway.com/dns-query |
| eve-lab-2 | CF-POP2 | (Separate endpoint) |

Each POP uses a dedicated DNS location for policy enforcement and logging separation.

<br>　**【日本語サマリ】**　<br>

SWGはカテゴリベースのコンテンツフィルタリングとTLS Inspection。ZTNAはService Token（ヘッドレスデバイス用）とIdP連携（Auth0/Entra ID）によるブラウザ認証をサポート。<br>
DNSフィルタリングは禁止ドメインへのNull応答（0.0.0.0/::）を実装。<br>
POP1とPOP2で別々のDNS Location（eve-lab, eve-lab-2）を設定し、それぞれ固有のDoHエンドポイントを使用。<br>
デバイス別のポリシー適用とログ分離が可能。

---

## 🔧Client Deployment (CF-POP1 / CF-POP2)

Both POPs are headless Linux (Debian 12) inside EVE-NG. All setup performed via CLI.

### 1. Package Installation

```bash
# GPG key and repository
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | \
  gpg --yes --dearmor -o /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] \
  https://pkg.cloudflareclient.com/ bookworm main" \
  > /etc/apt/sources.list.d/cloudflare-client.list

apt update && apt install -y cloudflare-warp cloudflared
```

### 2. TLS Certificate (for SWG TLS Inspection)

```bash
curl -o /usr/local/share/ca-certificates/cloudflare.crt \
  https://developers.cloudflare.com/cloudflare-one/static/Cloudflare_CA.crt
update-ca-certificates
```

### 3. Device Enrollment via Service Token

Headless devices cannot use browser-based IdP login. Service Token + mdm.xml was used instead.

![image](https://github.com/user-attachments/assets/2f267a3f-40bc-466d-9fd2-5afe7d41a6b6)
**📷 Cloudflare Dashboard - Service Tokens**

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

### 4. DNS Service Conflict Resolution

cloudflared, WARP, and systemd-resolved all compete for port 53.

| POP | Active DNS | Stopped | Reason |
|-----|-----------|---------|--------|
| CF-POP1 | WARP | cloudflared proxy-dns, systemd-resolved | WARP handles DNS + HTTP inspection |
| CF-POP2 | cloudflared proxy-dns | systemd-resolved | DoH required for eve-lab-2 identification |

### 5. Split Tunnel Configuration

WARP client routes all traffic through Cloudflare by default. This caused WARP to block direct communication to each other's WireGuard endpoints.

**Problem:**
- POP1's WARP blocks traffic to POP2's WireGuard endpoint (49.109.x.x)
- POP2's WARP blocks traffic to POP1's WireGuard endpoint (106.73.26.x)
- WireGuard tunnel cannot be established

```
POP1 ──► WARP ──✕ Blocked ──✕ POP2 WireGuard endpoint
POP2 ──► WARP ──✕ Blocked ──✕ POP1 WireGuard endpoint
```

**Solution:** Added WireGuard endpoint IPs to Split Tunnel exclusion list:

| Entry | Description |
|-------|-------------|
| 106.73.26.0/32 | POP1 WireGuard Endpoint |
| 49.109.0.0/16 | POP2 WireGuard Endpoint (docomo range) |

**Result:** WireGuard traffic bypasses WARP and connects directly.

```
POP1 ──► Direct Internet ──► POP2 WireGuard endpoint ✓
POP2 ──► Direct Internet ──► POP1 WireGuard endpoint ✓
```

　**【日本語サマリ】**<br>
 
ヘッドレスLinuxにCLIでWARP/cloudflaredをインストール、TLS証明書配置、Service Token認証でデバイス登録。<br>
ポート53競合はPOP別の役割分担で解決。<br>
WARPがデフォルトで全トラフィックをCloudflare経由にするため、WireGuard Endpoint IPをSplit Tunnelで除外し直接接続を確保。

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
WARPクライアントはUDP 443（QUIC/MASQUE）でCloudflare Gatewayに接続し、DNS/HTTP/TLSポリシーを適用。<br>
POP1-POP2間はWireGuard（UDP 4960）でサイト間接続し、その上でFG1-FG2間のSD-WAN IPsec（ESP）トラフィックを転送。<br>
本番環境では拠点間接続にBGP over IPsecを使用。<br>
ラボではLinux POPでBGP over IPsecが設定できないためWireGuardで代替。

---

## ✅Verification Results
### ESP Traffic Capture
📷 Wireshark ESP Capture - SASE Path Traffic
<img width="1760" alt="image" src="https://github.com/user-attachments/assets/ae77f17e-9b8c-47a2-bfac-db073ace4e93" />

ESP packets (Protocol 50) between POP1 (10.0.0.1) and POP2 (10.0.1.1).
I/O Graph shows traffic pattern during normal operation.


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
DNSブロックはbet365.com等に対し0.0.0.0/::を返却しTCP接続を阻止。<br>
HTTPポリシーでGambling/Adultカテゴリをブロック、Cloudflare内部通信はBYPASS。<br>
Gateway Logsでデバイス識別・ポリシー適用を確認。

---

## 🚀Key Learnings

### SASE Visibility Model

In encrypted environments (WARP/QUIC), traditional packet capture cannot see TCP handshakes or payload.

**Instead, use Cloudflare Gateway Logs for:**
- DNS query and response
- HTTP request and policy action
- Device identity and connection status

**【日本語サマリ】**

WARP/QUIC環境ではtcpdumpでTCPハンドシェークやペイロードが見えない。<br>
代わりにCloudflare Gateway LogsでDNS/HTTPポリシー適用状況を確認。

---

## 🔗Related Components

- [SD-WAN](../SD-WAN) - Path selection and failover
- [Brownout](../Brownout) - Quality degradation detection
- [Troubleshooting](../Troubleshooting) - Encrypted tunnel analysis

---

*Part of SASE × SD-WAN Verification Lab*
