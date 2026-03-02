# Future Task — Production STUN/TURN Infrastructure for Scale

## Problem

The current STUN/TURN setup (Step 11) runs a single coturn instance on the same `t3.micro` EC2 as the signaling server. This is fine for development and a handful of intercom systems, but will not hold up when going live with many buildings, intercom devices, and residents. Specific limitations:

- **Shared resources** — coturn competes for CPU, memory, and bandwidth with the signaling server on one tiny instance
- **Static credentials** — a single hardcoded username/password (`vidrom:VidromTurn2026!`) is baked into every app build; if leaked, anyone can abuse the TURN server
- **No redundancy** — if the EC2 instance goes down, all calls fail (no STUN fallback, no TURN relay)
- **No horizontal scaling** — a single coturn instance can only relay so many concurrent media streams before saturating its bandwidth or CPU
- **Single region** — all media relays through `us-east-1`, adding latency for users in other geographies
- **No TLS** — TURN over plain UDP/TCP; corporate firewalls that only allow port 443/TLS will still block TURN traffic
- **No monitoring** — no visibility into relay usage, bandwidth consumption, or error rates

## What Needs to Change

### 1. Separate TURN Server(s) from Signaling Server

Move coturn to dedicated EC2 instance(s) sized for media relay workloads.

| Concern | Current | Target |
|---------|---------|--------|
| Instance | `t3.micro` (shared) | `c5.xlarge` or `c5n.large` (dedicated, network-optimised) |
| Bandwidth | ~5 Gbps burst shared | Dedicated enhanced networking |
| CPU | 2 vCPU shared | 4+ vCPU dedicated to relay |

**Actions:**
- Add a new EC2 instance (or Auto Scaling Group) in `vidrom-cdk-stack.ts` dedicated to coturn
- Assign its own Elastic IP
- Keep the signaling server on its own instance without coturn

### 2. Dynamic (Short-Lived) TURN Credentials

Replace the static `user=vidrom:VidromTurn2026!` with time-limited credentials generated per-session via the signaling server. coturn supports this natively with its `use-auth-secret` mode.

**How it works:**
1. Configure coturn with a shared secret instead of static users:
   ```
   use-auth-secret
   static-auth-secret=<LONG_RANDOM_SECRET>
   realm=vidrom.com
   ```
2. The signaling server generates temporary credentials on each call setup:
   ```javascript
   const crypto = require('crypto');
   function getTurnCredentials(name, secret) {
     const unixTimeStamp = Math.floor(Date.now() / 1000) + 24 * 3600; // 24h TTL
     const username = `${unixTimeStamp}:${name}`;
     const hmac = crypto.createHmac('sha1', secret);
     hmac.update(username);
     const credential = hmac.digest('base64');
     return { username, credential };
   }
   ```
3. The signaling server sends ICE server config (including temporary TURN credentials) to both peers at call setup time via WebSocket, instead of the apps having hardcoded credentials.

**Benefits:**
- Credentials expire after 24 hours — leaked credentials become useless
- Each session gets unique credentials — abuse is traceable
- No secrets in the client app bundles

**Files to modify:**
| File | Change |
|------|--------|
| EC2: `/etc/turnserver.conf` | Replace `lt-cred-mech` + `user=` with `use-auth-secret` + `static-auth-secret=` |
| `vidrom-signaling-server/src/wsHandler.js` | Generate temp credentials and include ICE config in `call-request` / `call-accepted` messages |
| `vidrom-ai-home/config.js` | Remove hardcoded `ICE_SERVERS`; receive ICE config from signaling server |
| `vidrom-ai-home/useCallManager.js` | Use dynamic ICE config from signaling messages to create `RTCPeerConnection` |
| `vidrom-ai-intercom/webrtcHtml.js` | Use dynamic ICE config received from signaling messages |
| `vidrom-ai-intercom/config.js` | Remove hardcoded TURN credentials |

### 3. TURNS (TURN over TLS on Port 443)

Corporate and hotel Wi-Fi networks often block all UDP and only allow TCP 443. Without TURNS, these users cannot make calls.

**Actions:**
- Obtain a TLS certificate for the TURN server domain (e.g., `turn.vidrom.com`) via Let's Encrypt or ACM
- Configure coturn to listen on port 443 with TLS:
  ```
  listening-port=3478
  tls-listening-port=443
  cert=/etc/letsencrypt/live/turn.vidrom.com/fullchain.pem
  pkey=/etc/letsencrypt/live/turn.vidrom.com/privkey.pem
  ```
- Add `turns:turn.vidrom.com:443?transport=tcp` to the ICE server list
- Set up automatic certificate renewal (certbot cron)

### 4. Domain Name for TURN Server

Replace the raw IP (`52.203.117.37`) with a DNS record so the TURN server IP can change without re-deploying apps.

- Create a Route 53 A record: `turn.vidrom.com → <TURN Elastic IP>`
- Update ICE config to use `turn:turn.vidrom.com:3478` and `turns:turn.vidrom.com:443`
- When scaling to multiple servers, this becomes a mechanism for routing (see §5)

### 5. High Availability & Horizontal Scaling

A single TURN server is a single point of failure and a bottleneck. For production:

**Option A — Multiple coturn instances behind DNS round-robin**
- Deploy 2+ coturn instances across availability zones
- Use Route 53 with health checks and weighted/round-robin routing
- Each instance has its own Elastic IP and runs independently
- Simple, no shared state needed (TURN allocations are per-server)

**Option B — Network Load Balancer (NLB) in front of coturn fleet**
- Deploy an NLB (UDP + TCP) in front of an Auto Scaling Group of coturn instances
- Scaling policy based on network bandwidth or CPU
- NLB natively supports UDP (required for TURN)
- More operational overhead but true auto-scaling

**Option C — Managed TURN service (see §6 for detailed comparison)**
- Offload TURN infrastructure entirely to a managed provider
- Can be used as the primary TURN solution, or as a fallback alongside self-hosted coturn
- See section 6 below for a full breakdown of providers, pricing, and pros/cons

**Recommendation:** Start with Option A (2 instances, DNS health checks) for cost control. Consider a managed provider (Option C / §6) as the primary solution if operational simplicity is more important than saving on relay costs, or as a global fallback alongside self-hosted.

### 6. Managed TURN Providers — Full Comparison

Instead of (or in addition to) running self-hosted coturn, you can use a managed TURN service. This section compares the major providers and the self-hosted approach.

#### Provider Overview

| Provider | Pricing Model | Global PoPs | TURNS (TLS 443) | Dynamic Credentials | Free Tier |
|----------|--------------|-------------|-----------------|--------------------|-----------|
| **Twilio NTS** | $0.0004/min per relay (~$0.024/hr) | 6 regions (US, EU, Asia, AU, SA, IN) | Yes | Yes (REST API) | None (pay-as-you-go) |
| **Cloudflare Calls TURN** | $0.05/GB relay traffic | 300+ edge cities | Yes | Yes (REST API) | 1 GB/month free |
| **Metered TURN** | $0.04/GB (Standard) / $0.08/GB (Premium) | 20+ global PoPs | Yes | Yes (REST API) | 500 GB/month free |
| **Xirsys** | $49–$499/month plans, overage per GB | 12 global PoPs | Yes | Yes (REST API) | Free plan (500 MB/month) |
| **Google STUN (free)** | Free (STUN only, no TURN) | Global | N/A | N/A | Unlimited STUN |
| **Self-hosted coturn** | EC2 cost only (~$62/mo for c5.large) | You choose regions | You configure | You implement | N/A |

#### How Each Provider Works

**Twilio Network Traversal Service (NTS)**
- REST API call from the signaling server → returns temporary ICE server list with short-lived credentials
- Credentials are valid for ~24 hours, generated per session
- Integration: ~10 lines of code in the signaling server
```javascript
// Example: signaling server generates Twilio TURN credentials
const twilio = require('twilio')(ACCOUNT_SID, AUTH_TOKEN);
const token = await twilio.tokens.create({ ttl: 86400 });
// token.iceServers → array of STUN + TURN servers with temp credentials
// Send to both peers via WebSocket
```

**Cloudflare Calls TURN**
- Similar REST API pattern; returns TURN credentials scoped to your account
- Leverages Cloudflare's 300+ PoP edge network — extremely low latency globally
- Relatively new service (launched 2024), rapidly evolving
```javascript
// Example: Cloudflare TURN credential generation
const resp = await fetch('https://rtc.live.cloudflare.com/v1/turn/keys/<KEY_ID>/credentials/generate', {
  method: 'POST',
  headers: { Authorization: `Bearer ${API_TOKEN}` },
  body: JSON.stringify({ ttl: 86400 }),
});
const { iceServers } = await resp.json();
```

**Metered TURN**
- Dashboard + REST API for credential generation
- Offers both Standard (shared infrastructure) and Premium (dedicated) tiers
- Generous free tier (500 GB/month) — may cover early production at low scale entirely for free

**Xirsys**
- Established WebRTC infrastructure provider
- Dashboard-based management with REST API
- Fixed monthly plans with overage fees — more predictable budgeting

#### Pros & Cons: Self-Hosted coturn vs. Managed TURN

**Self-Hosted coturn:**

| Pros | Cons |
|------|------|
| Full control over configuration and performance tuning | You own all operational burden (updates, security patches, monitoring) |
| Predictable cost at scale — EC2 instance cost is flat regardless of relay minutes | Requires infrastructure expertise to set up HA, TLS, monitoring |
| No third-party dependency — no vendor lock-in, no API changes | Single-region unless you deploy and manage multi-region yourself |
| Data stays on your infrastructure (compliance/privacy) | Scaling requires manual intervention or complex ASG setup |
| Can customize relay policies, logging, and rate limits | Building from source on Amazon Linux — no official package, updates are manual |
| No per-minute or per-GB fees | You handle DDoS protection, cert renewal, log rotation, etc. |

**Managed TURN (Twilio, Cloudflare, Metered, Xirsys):**

| Pros | Cons |
|------|------|
| Zero infrastructure to manage — no servers, no patching, no monitoring | Per-minute or per-GB cost adds up at high volume |
| Global coverage out of the box — low latency worldwide | Vendor dependency — API changes, pricing changes, or outages affect you |
| Built-in HA and redundancy — provider handles failover | Less control over relay behavior and performance tuning |
| Dynamic credentials via simple REST API — easy integration | Data relay goes through third-party servers (potential compliance concern) |
| TURNS (TLS on 443) included — works through restrictive firewalls | Debugging relay issues is harder when you don't control the infrastructure |
| Scales automatically — no capacity planning needed | No free tier (Twilio) or limited free tiers that may not last |
| Can be integrated in hours, not days | Ongoing recurring cost even during low-usage periods |

#### Cost Comparison at Different Scales

Assumptions: 720p video calls (~3 Mbps relay), 10–15% of calls need TURN relay, average call duration 2 minutes.

| Scale | Intercoms | Relayed Calls/Month | Self-Hosted Cost/mo | Twilio Cost/mo | Cloudflare Cost/mo | Metered Cost/mo |
|-------|-----------|--------------------|--------------------|---------------|-------------------|----------------|
| Pilot | 50 | ~450 | ~$62 (c5.large) | ~$0.36 | ~$0.08 (free tier) | Free (< 500 GB) |
| Small | 500 | ~4,500 | ~$62 (c5.large) | ~$3.60 | ~$0.81 | Free (< 500 GB) |
| Medium | 5,000 | ~45,000 | ~$124 (c5.xlarge ×2) | ~$36 | ~$8.10 | ~$7.29 |
| Large | 50,000 | ~450,000 | ~$310 (c5.xlarge ×5) | ~$360 | ~$81 | ~$72.90 |
| Very Large | 200,000 | ~1,800,000 | ~$620 (c5.2xl ×5) | ~$1,440 | ~$324 | ~$291.60 |

*Self-hosted EC2 costs are compute-only (no data transfer egress, which at high relay volumes could add $50–200+/mo).*

**Key takeaway:** Managed services are dramatically cheaper at small scale (especially with free tiers), roughly break-even at medium scale, and become more expensive than self-hosted at very large scale. Self-hosted has a higher fixed baseline but flatter cost curve.

#### Hybrid Approach (Recommended)

The best production setup often combines both:

1. **Self-hosted coturn as primary** — handles the majority of TURN relays at predictable cost
2. **Managed TURN as fallback** — the signaling server includes both self-hosted and managed TURN servers in the ICE config; if the self-hosted server is down or unreachable, clients automatically fall back to the managed provider
3. **Managed TURN for global reach** — if users are outside your self-hosted regions, the managed provider's global PoPs provide low-latency relay without deploying coturn everywhere

```javascript
// Example: hybrid ICE config sent from signaling server
const iceServers = [
  // Free STUN
  { urls: 'stun:stun.l.google.com:19302' },
  // Self-hosted TURN (primary)
  { urls: ['turn:turn.vidrom.com:3478', 'turns:turn.vidrom.com:443'],
    username: dynamicUsername, credential: dynamicCredential },
  // Managed TURN (fallback) — e.g., Twilio or Cloudflare
  ...twilioIceServers,
];
```

#### Provider Selection Guide

| If you need... | Best choice |
|----------------|------------|
| Fastest integration, proven reliability | Twilio NTS |
| Lowest cost at any scale, largest global footprint | Cloudflare Calls TURN |
| Generous free tier to start, good global coverage | Metered TURN |
| Predictable monthly billing, fixed plans | Xirsys |
| Full control, lowest cost at very high scale | Self-hosted coturn |
| Best of both worlds | Hybrid: self-hosted + managed fallback |

### 7. Multi-Region Deployment (Geographic Distribution)

For users across different countries/continents, relay through a single `us-east-1` server adds significant latency.

**Actions:**
- Deploy coturn instances in key regions (e.g., `us-east-1`, `eu-west-1`, `ap-southeast-1`)
- Use Route 53 latency-based routing so clients connect to the nearest TURN server
- Each regional instance runs the same coturn config with its own `external-ip`
- Alternatively, a managed TURN provider (Twilio, Cloudflare) handles this automatically with global PoPs

### 8. Capacity Planning & Instance Sizing

Rough estimates for coturn relay bandwidth:

| Metric | Value |
|--------|-------|
| Audio-only call (Opus) | ~50 kbps per direction → ~100 kbps total relay |
| Video call (VP8/H264 480p) | ~500 kbps per direction → ~1 Mbps total relay |
| Video call (720p) | ~1.5 Mbps per direction → ~3 Mbps total relay |
| TURN relay % of all calls | ~10–15% (rest are direct P2P via STUN) |

**Scaling math example:**
- 1000 intercoms, peak 5% concurrent calls = 50 simultaneous calls
- 15% need TURN relay = ~8 relayed calls
- 8 calls × 3 Mbps (720p) = ~24 Mbps sustained bandwidth
- A `c5.xlarge` with enhanced networking handles this easily
- At 10,000 intercoms: ~80 relayed calls, ~240 Mbps → still a single large instance or 2–3 medium ones

**Instance recommendations by scale:**

| Scale | Intercoms | Recommended TURN Setup |
|-------|-----------|----------------------|
| Small | < 500 | 1× `c5.large` or managed TURN service |
| Medium | 500–5,000 | 2× `c5.xlarge` with DNS failover |
| Large | 5,000–50,000 | NLB + ASG (3–5 instances) or managed TURN |
| Global | 50,000+ | Multi-region deployment + managed TURN fallback |

### 9. Monitoring & Alerting

Production TURN infrastructure needs observability.

**Metrics to track:**
- Active relay allocations (concurrent sessions)
- Bandwidth in/out per instance
- Failed authentication attempts (abuse detection)
- CPU and memory utilization
- Allocation failures (port exhaustion)

**Implementation:**
- Enable coturn's Prometheus exporter (`prometheus` option in `turnserver.conf`) or use `redis-statsdb`
- Push metrics to CloudWatch via the CloudWatch agent
- Create CloudWatch alarms:
  - Bandwidth > 80% of instance capacity → scale out
  - CPU > 70% sustained → scale out
  - Failed auth attempts > threshold → alert (possible abuse)
  - Instance health check failure → alert + DNS failover

**Log aggregation:**
- Ship coturn logs to CloudWatch Logs or an ELK stack
- Set up log-based alerts for connection errors and relay failures

### 10. Security Hardening

- **Rate limiting** — configure coturn's `max-bps` and `total-quota` to prevent a single user from consuming all bandwidth:
  ```
  max-bps=3000000          # 3 Mbps per session max
  total-quota=100          # Max 100 sessions per user
  user-quota=10            # Max 10 sessions per username
  ```
- **Denied peer IPs** — already configured, ensure private ranges stay blocked
- **Firewall rules** — restrict relay port range if possible; keep `min-port`/`max-port` as narrow as the expected concurrent session count allows
- **DDoS protection** — consider AWS Shield Standard (free) or Advanced for the TURN server's Elastic IP
- **Audit logging** — log all TURN allocations with timestamps and source IPs for forensic analysis

### 11. CDK Infrastructure Changes Summary

New resources to add to `vidrom-cdk-stack.ts`:

| Resource | Purpose |
|----------|---------|
| New EC2 instance(s) or ASG | Dedicated coturn server(s) |
| Elastic IP(s) | Stable TURN server addresses |
| Security Group | TURN-specific ports (3478 TCP/UDP, 443 TCP, relay range) |
| Route 53 records | `turn.vidrom.com` with health checks |
| CloudWatch Alarms | Bandwidth, CPU, health monitoring |
| IAM Role | CloudWatch Logs/Metrics permissions for coturn instances |
| (Optional) NLB | Load balancing across coturn fleet |

## Migration Path

1. **Phase 1** — Deploy a dedicated coturn instance separate from the signaling server; switch apps to use domain name (`turn.vidrom.com`) instead of raw IP; implement dynamic credentials
2. **Phase 2** — Add TURNS (TLS on 443); add a second coturn instance with DNS failover; set up monitoring and alerting
3. **Phase 3** — Integrate a managed TURN provider (Cloudflare or Metered for free-tier start) as a fallback alongside self-hosted; evaluate whether to keep self-hosted as primary or switch entirely to managed based on cost and usage data
4. **Phase 4** — Add multi-region self-hosted coturn if user base requires it, or rely on managed provider's global PoPs; implement auto-scaling if concurrent relay count justifies it

## Files to Create / Modify

| File | Change |
|------|--------|
| `vidrom-cdk/lib/vidrom-cdk-stack.ts` | New EC2/ASG for TURN, new SG, Route 53 records, CloudWatch alarms |
| `vidrom-signaling-server/src/wsHandler.js` | Generate and send dynamic TURN credentials per session |
| `vidrom-signaling-server/src/server.js` | Load TURN shared secret from environment variable |
| `vidrom-ai-home/useCallManager.js` | Accept dynamic ICE config from signaling server |
| `vidrom-ai-home/config.js` | Remove hardcoded TURN credentials; keep STUN-only defaults |
| `vidrom-ai-intercom/webrtcHtml.js` | Accept dynamic ICE config from signaling server |
| `vidrom-ai-intercom/config.js` | Remove hardcoded TURN credentials |
| EC2: `/etc/turnserver.conf` | `use-auth-secret`, TLS certs, Prometheus, rate limits |
