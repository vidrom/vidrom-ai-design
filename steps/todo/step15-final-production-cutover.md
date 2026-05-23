# Step 15 Final — Security Hardening Production Cutover After AWS Unlock

## Purpose

Step 15 is implemented locally, but production is not complete until the CDK-backed portal, Lambda, and EC2 runtime changes are deployed and verified in AWS.

Use this step only after the AWS account is unlocked again.

## Current Status

Already completed locally:

1. portal API CORS is restricted and security headers are emitted
2. portal API throttling is in place in API Gateway and Lambda
3. EC2 runtime privileges and secret file placement are hardened
4. server and Lambda logs redact stable identifiers more aggressively
5. portal pages now rely on external JavaScript and page-level CSP instead of inline handlers/scripts
6. EC2 secret refresh automation is defined through systemd service and timer units

Not completed in AWS yet:

1. deploy the updated CDK stack and portal assets
2. verify the live portal still loads and authenticates correctly
3. verify the EC2 signaling service runs under the hardened systemd profile in production
4. confirm the secret refresh timer is installed and enabled on the live host

## Files Already Landed

- `vidrom-signaling-server/lambda/handler.js`
- `vidrom-signaling-server/lambda/rateLimit.js`
- `vidrom-signaling-server/lambda/logging.js`
- `vidrom-signaling-server/src/logging.js`
- `vidrom-signaling-server/run.sh`
- `vidrom-signaling-server/portals/admin.html`
- `vidrom-signaling-server/portals/admin.js`
- `vidrom-signaling-server/portals/management.html`
- `vidrom-signaling-server/portals/management.js`
- `vidrom-cdk/lib/vidrom-cdk-stack.ts`

## Do This When AWS Is Unlocked

### 1. Reconfirm local state

From `vidrom-signaling-server`:

```sh
npm test
```

From `vidrom-cdk`:

```sh
npm run build
npx cdk diff
```

Only continue if the diff still matches the expected Step 15 security changes.

### 2. Deploy the stack and portal assets

Canonical deploy path:

```sh
cd vidrom-cdk
./cdk-deploy.sh
```

### 3. Verify the portal edge and auth flows

Confirm from the live environment:

1. `https://portal.vidrom.com/admin` loads successfully
2. `https://portal.vidrom.com/management` loads successfully
3. Google Sign-In still works on both pages
4. portal API responses still include the Step 15 security headers
5. repeated portal abuse bursts are throttled as expected

### 4. Verify EC2 runtime hardening

On the signaling instance, confirm:

1. `vidrom-signaling.service` runs as the dedicated `vidrom` user
2. runtime secret files are created only under `/run/vidrom-signaling`
3. `vidrom-secret-refresh.timer` is enabled and active
4. a manual refresh run restarts `vidrom-signaling` and `coturn` cleanly

### 5. Re-check the exit criteria

Mark Step 15 complete in production only when all are true:

1. portal pages load without inline-script regressions
2. portal API CORS/security headers are live
3. live throttling behavior is in place for portal endpoints
4. EC2 runtime privilege reduction and ephemeral secret handling are active
5. secret refresh automation is installed and functioning

## Completion Trigger

After the deploy and live verification succeed, move this final cutover step out of `todo` and keep the design tracker aligned with the production state.