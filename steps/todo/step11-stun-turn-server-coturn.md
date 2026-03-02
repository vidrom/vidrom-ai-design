# Step 11 — Add STUN/TURN Server (coturn) on Existing EC2

## Goal

Install and configure coturn on the existing EC2 signaling server to provide a self-hosted STUN fallback and a TURN relay. This guarantees WebRTC connectivity in all network conditions — including symmetric NAT, carrier-grade NAT, and corporate firewalls where direct peer-to-peer connections fail.

## Overview

- Install coturn on the existing `t3.micro` EC2 instance (IP: `52.203.117.37`)
- Configure it as both a STUN and TURN server on port 3478
- Open the required ports in the CDK security group
- Update ICE server configuration in both apps (home + intercom)
- Verify TURN relay works end-to-end

## Background

Currently both apps only use Google's free STUN server (`stun:stun.l.google.com:19302`). This works for most scenarios but fails when:
- Both peers are behind symmetric NAT
- A corporate firewall blocks direct UDP traffic
- Google's STUN server is temporarily unavailable

Adding coturn provides:
1. **STUN fallback** — our own STUN server as a backup if Google's is unreachable
2. **TURN relay** — media relay for the ~5–10% of connections where direct paths fail

See `webrtc.md` for detailed concepts on STUN, TURN, and ICE.

## Prerequisites

1. Steps 1–10 completed (both apps functional with WebRTC calls)
2. SSH access to the EC2 instance (`52.203.117.37`)
3. SSH key: `vidrom-useast1-v2.pem`

---

## Step 11.1 — Open Ports in CDK Security Group

Add STUN/TURN and relay port rules to `vidrom-cdk/lib/vidrom-cdk-stack.ts`.

After the existing ingress rules (WebSocket 8080, SSH 22), add:

```typescript
// STUN/TURN listener port
sg.addIngressRule(ec2.Peer.anyIpv4(), ec2.Port.tcp(3478), 'TURN TCP');
sg.addIngressRule(ec2.Peer.anyIpv4(), ec2.Port.udp(3478), 'TURN UDP');
// TURN relay ports (media flows through these)
sg.addIngressRule(ec2.Peer.anyIpv4(), ec2.Port.udpRange(49152, 65535), 'TURN relay');
```

Deploy the updated stack:

```bash
cd vidrom-cdk
npx cdk deploy --require-approval never
```

Verify in the AWS Console that the security group now has rules for UDP 3478, TCP 3478, and UDP 49152–65535.

---

## Step 11.2 — Install coturn on EC2

SSH into the instance:

```bash
ssh -i /path/to/vidrom-useast1-v2.pem ec2-user@52.203.117.37
```

Install coturn:

```bash
# Amazon Linux 2023
sudo dnf install -y coturn
```

> **Note:** If coturn is not available in the default repos, install from EPEL:
> ```bash
> sudo dnf install -y epel-release
> sudo dnf install -y coturn
> ```
>
> If neither works on Amazon Linux 2023, build from source:
> ```bash
> sudo dnf install -y gcc make openssl-devel libevent-devel
> cd /tmp
> git clone https://github.com/coturn/coturn.git
> cd coturn
> ./configure --prefix=/usr/local
> make && sudo make install
> ```

---

## Step 11.3 — Configure coturn

Create the coturn configuration file:

```bash
sudo tee /etc/turnserver.conf << 'EOF'
# ── Vidrom STUN/TURN Server ──────────────────────────────
# Runs on the same EC2 as the signaling server

# Listener
listening-port=3478
listening-ip=0.0.0.0

# External IP (Elastic IP) — required so coturn advertises the correct address
external-ip=52.203.117.37

# TURN relay port range (must match security group)
min-port=49152
max-port=65535

# Authentication (long-term credential mechanism, required for TURN)
lt-cred-mech
realm=vidrom.com
user=vidrom:VidromTurn2026!

# Fingerprint for STUN message integrity
fingerprint

# Disable CLI admin (not needed)
no-cli

# Logging
log-file=/var/log/turnserver.log
verbose

# Security: deny relay to private/internal networks
denied-peer-ip=10.0.0.0-10.255.255.255
denied-peer-ip=172.16.0.0-172.31.255.255
denied-peer-ip=192.168.0.0-192.168.255.255
EOF
```

### Configuration explained

| Setting | Purpose |
|---------|---------|
| `listening-port=3478` | Standard STUN/TURN port |
| `external-ip=52.203.117.37` | Elastic IP — coturn tells clients to reach it here |
| `lt-cred-mech` | Long-term credentials for TURN authentication |
| `user=vidrom:VidromTurn2026!` | Username and password for TURN clients |
| `realm=vidrom.com` | Authentication realm |
| `fingerprint` | Adds STUN fingerprint to messages (improved compatibility) |
| `min-port`/`max-port` | UDP port range for media relay allocations |
| `denied-peer-ip` | Prevents TURN relay to private networks (security best practice) |

---

## Step 11.4 — Start coturn as a systemd Service

```bash
# Create systemd service
sudo tee /etc/systemd/system/coturn.service << 'EOF'
[Unit]
Description=coturn STUN/TURN Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/turnserver -c /etc/turnserver.conf
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable coturn
sudo systemctl start coturn
```

Verify it's running:

```bash
sudo systemctl status coturn
# Should show: active (running)

# Check it's listening on port 3478
sudo ss -ulnp | grep 3478
# Should show: udp  UNCONN  0  0  0.0.0.0:3478  ...  turnserver
```

---

## Step 11.5 — Test STUN/TURN from Outside

### Test STUN

Use an online tool or `stunclient` to verify:

```bash
# From your local machine (not the EC2):
# Install stunclient (e.g., via apt: sudo apt install stuntman-client)
stunclient 52.203.117.37 3478
```

Expected output: your public IP and mapped port.

### Test TURN

Use the [Trickle ICE](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/) web tool:

1. Open https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/
2. Add a TURN server:
   - URI: `turn:52.203.117.37:3478`
   - Username: `vidrom`
   - Password: `VidromTurn2026!`
3. Click **Gather candidates**
4. Verify you see **relay** candidates in the output — this confirms TURN is working

---

## Step 11.6 — Update ICE Configuration in Home App

Edit `vidrom-ai-home/config.js`:

```javascript
// ---- Shared configuration constants ----

export const SERVER_IP = 'signaling.vidrom.com';
export const SERVER_PORT = '8080';
export const SERVER_HTTP = `http://${SERVER_IP}:${SERVER_PORT}`;
export const SERVER_WS = `ws://${SERVER_IP}:${SERVER_PORT}`;

export const ICE_SERVERS = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
    { urls: 'stun:52.203.117.37:3478' },
    {
      urls: 'turn:52.203.117.37:3478',
      username: 'vidrom',
      credential: 'VidromTurn2026!',
    },
  ],
};
```

---

## Step 11.7 — Update ICE Configuration in Intercom App

Edit `vidrom-ai-intercom/webrtcHtml.js` — update the `ICE_SERVERS_JSON` constant at the top:

```javascript
const ICE_SERVERS_JSON = JSON.stringify({
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
    { urls: 'stun:52.203.117.37:3478' },
    {
      urls: 'turn:52.203.117.37:3478',
      username: 'vidrom',
      credential: 'VidromTurn2026!',
    },
  ],
});
```

---

## Step 11.8 — Update CDK User Data (for Future Rebuilds)

Add coturn installation to the user data in `vidrom-cdk/lib/vidrom-cdk-stack.ts` so new instances are provisioned automatically. After the existing Node.js/systemd setup, add:

```typescript
// Install and configure coturn (STUN/TURN)
'dnf install -y coturn || (dnf install -y epel-release && dnf install -y coturn)',

'cat > /etc/turnserver.conf << \'TURNCONF\'',
'listening-port=3478',
'listening-ip=0.0.0.0',
'external-ip=52.203.117.37',
'min-port=49152',
'max-port=65535',
'lt-cred-mech',
'realm=vidrom.com',
'user=vidrom:VidromTurn2026!',
'fingerprint',
'no-cli',
'log-file=/var/log/turnserver.log',
'denied-peer-ip=10.0.0.0-10.255.255.255',
'denied-peer-ip=172.16.0.0-172.31.255.255',
'denied-peer-ip=192.168.0.0-192.168.255.255',
'TURNCONF',

'systemctl daemon-reload',
'systemctl enable coturn',
'systemctl start coturn',
```

---

## Step 11.9 — End-to-End Verification

### Test 1: Normal call (STUN path)

1. Run the intercom and home app on the same Wi-Fi network
2. Make a call from intercom → home
3. Verify audio and video work
4. Check logs — ICE should use **host** or **srflx** (server-reflexive) candidates

### Test 2: TURN relay path

To force TURN usage and verify the relay works:

1. Temporarily change the ICE config in both apps to **only** include the TURN server (remove STUN entries):
   ```javascript
   iceServers: [
     {
       urls: 'turn:52.203.117.37:3478',
       username: 'vidrom',
       credential: 'VidromTurn2026!',
     },
   ]
   ```
2. Make a call
3. Verify audio and video still work — all media is now relaying through coturn
4. Check the coturn log: `sudo tail -f /var/log/turnserver.log` — you should see relay allocations
5. **Restore** the full ICE config with all STUN + TURN servers after testing

### Test 3: Watch mode

1. From the home app, start a watch session
2. Verify the intercom camera video streams to the home app
3. Stop watching and confirm cleanup

---

## Step 11.10 — Monitoring (Optional)

Check coturn status and active sessions:

```bash
# Service status
sudo systemctl status coturn

# Active relay allocations
sudo ss -ulnp | grep turnserver

# Log monitoring
sudo tail -f /var/log/turnserver.log

# Log rotation (prevent disk from filling up)
sudo tee /etc/logrotate.d/coturn << 'EOF'
/var/log/turnserver.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    copytruncate
}
EOF
```

---

## Files Changed

| File | Change |
|------|--------|
| `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Add security group rules for UDP/TCP 3478 and UDP 49152–65535; add coturn install to user data |
| `vidrom-ai-home/config.js` | Add STUN fallbacks and TURN server to `ICE_SERVERS` |
| `vidrom-ai-intercom/webrtcHtml.js` | Add STUN fallbacks and TURN server to `ICE_SERVERS_JSON` |
| EC2: `/etc/turnserver.conf` | New coturn configuration file |
| EC2: `/etc/systemd/system/coturn.service` | New systemd service for coturn |

## Completion Criteria

- ✅ coturn installed and running on EC2 (`systemctl status coturn` → active)
- ✅ Security group allows UDP/TCP 3478 and UDP 49152–65535
- ✅ STUN test returns your public IP (via `stunclient` or Trickle ICE tool)
- ✅ TURN test shows relay candidates (via Trickle ICE tool)
- ✅ Both apps updated with new ICE configuration including STUN fallbacks and TURN
- ✅ Normal call works (STUN path)
- ✅ Forced TURN-only call works (relay path)
- ✅ Watch mode works with new ICE configuration
- ✅ CDK user data updated for future instance rebuilds
