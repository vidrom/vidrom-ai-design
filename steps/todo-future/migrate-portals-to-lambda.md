# Future Task — Migrate REST APIs to Lambda

## Summary

Move all stateless REST APIs from the signaling server (EC2) to AWS Lambda + API Gateway. This includes portal APIs (`/api/admin/*`, `/api/management/*`) and app APIs (`/api/app/*`, `/api/intercom/*`). Static HTML files (`admin.html`, `management.html`) would be served from S3 + CloudFront.

See [backend-architecture.md](../../backend-architecture.md) for the full architecture overview.

## Why

- All REST APIs are stateless endpoints — classic Lambda candidates
- Portal traffic is bursty and low-frequency — paying for idle EC2 time is wasteful
- App REST traffic (building config, call history, etc.) is also low-frequency and stateless
- Lambda free tier covers 1M requests/month — expected volume will be well under that
- Auto-scales if the number of buildings/managers/residents grows
- Decouples REST concerns from the real-time signaling server
- Signaling server becomes a thin WebSocket relay — simpler, more stable

## Why Not Now

- The portals and app APIs are still being built — premature optimization
- Keeping everything in the signaling server is simpler during development
- The clean API prefix separation means nothing built today prevents migration later

## Target Architecture

```
┌─────────────────────────────┐
│  CloudFront + S3            │  ← serves admin.html / management.html
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  API Gateway + Lambda       │  ← /api/admin/*, /api/management/*,
│  (Node.js functions)        │     /api/app/*, /api/intercom/*
│  connects to RDS            │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  RDS PostgreSQL             │  ← shared database
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│  EC2 (signaling server)     │  ← WebSocket + real-time signaling only
│  direct DB for call/audit   │     (+ direct RDS writes for call events & audit logs)
└─────────────────────────────┘
```

## What Moves to Lambda

| API Prefix | Consumer | Auth |
|-----------|----------|------|
| `/api/admin/*` | Admin Portal | Google OAuth, `role = 'admin'` |
| `/api/management/*` | Management Portal | Google OAuth, `role = 'manager'` |
| `/api/app/*` | Home App | Google/Apple OAuth, `role = 'resident'` |
| `/api/intercom/*` | Intercom App | Device JWT |

## What Stays on EC2 (Signaling Server)

- WebSocket signaling (`wsHandler.js`)
- WebRTC offer/answer/ICE relay
- Call state management (real-time transitions)
- Watch camera session management
- Push notification triggering (FCM/APNs)
- Direct DB writes for call events and audit logs (same VPC as RDS)

## Migration Steps

### 1. Create Lambda functions

- Wrap existing route handlers from `httpRoutes.js` in Lambda-compatible exports
- Each handler is already stateless — just needs a small adapter:

```js
// Example: listBuildings Lambda handler
exports.handler = async (event) => {
  const user = await verifyToken(event.headers.Authorization);
  if (!user) return { statusCode: 401, body: '{"error":"Unauthorized"}' };

  const buildings = await db.query('SELECT * FROM buildings');
  return { statusCode: 200, body: JSON.stringify(buildings.rows) };
};
```

### 2. Add to CDK stack (`vidrom-cdk-stack.ts`)

- `new lambda.Function()` for portal API handler(s)
- `new apigateway.RestApi()` with routes mapped to Lambda
- `new s3.Bucket()` + `new cloudfront.Distribution()` for static HTML
- RDS security group updated to allow Lambda VPC access

### 3. Move static HTML to S3

- Upload `admin.html` and `management.html` to S3
- CloudFront distribution with custom domain (e.g., `admin.vidrom.com`)

### 4. Update API base URL

- Change `fetch('/api/...')` in HTML files to `fetch('https://api.vidrom.com/...')`
- Or use a relative path if API Gateway is behind the same CloudFront distribution

### 5. Remove REST routes from signaling server

- Remove `/admin`, `/management` static file serving
- Remove all `/api/admin/*`, `/api/management/*`, `/api/app/*`, `/api/intercom/*` route handlers
- Remove auth middleware (moved to Lambda)
- Keep only WebSocket handler + direct DB writes for call/audit events

## Estimated Effort

| Task | Time |
|------|------|
| Wrap route handlers in Lambda exports | ~2 hours |
| Add API Gateway + Lambda to CDK stack | ~1.5 hours |
| Move static HTML to S3 + CloudFront | ~30 min |
| Update API base URL in HTML and apps | ~15 min |
| Test and validate | ~1.5 hours |
| **Total** | **~5.5 hours** |
| **Total** | **~3.5 hours** |

## Cost Impact

| Component | Monthly cost |
|-----------|-------------|
| Lambda (portal APIs) | ~$0 (free tier) |
| API Gateway | ~$0 (free tier) |
| S3 + CloudFront (static HTML) | ~$0.50 |
| EC2 (signaling only) | Same as today |
| RDS | Same as today |

## When to Do This

The natural transition point is when PostgreSQL/RDS is added to the system. At that point, the portal APIs need a database connection anyway — wiring Lambda to RDS is straightforward with VPC configuration. Doing the Lambda migration at the same time avoids touching the code twice.
