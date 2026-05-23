# Step 12-A3 Final — HTTPS/WSS Production Cutover After AWS Unlock

## Purpose

Step 12-A3 is implemented locally, but production is not complete until the TLS infrastructure is deployed in AWS and the mobile apps are re-verified against the live `https://` and `wss://` signaling endpoints.

Use this step only after the AWS account is unlocked again.

## Current Status

Already completed locally:

1. production client defaults point to `https://signaling.vidrom.com` and `wss://signaling.vidrom.com`
2. local and temporary endpoint overrides use `EXPO_PUBLIC_SIGNALING_HTTP_URL` and `EXPO_PUBLIC_SIGNALING_WS_URL`
3. the home app no longer carries the production ATS insecure transport exception
4. CDK defines the TLS entry point in front of the existing signaling backend
5. docs and ops scripts now reference secure endpoints

Not completed in AWS yet:

1. deploy the TLS entry point and secure routing changes with CDK
2. confirm the live production hostname serves HTTPS and WSS successfully
3. verify the mobile apps connect cleanly to the secure production endpoints

## Files Already Landed

- `vidrom-ai-home/config.js`
- `vidrom-ai-home/app.json`
- `vidrom-ai-intercom/config.js`
- `vidrom-cdk/lib/vidrom-cdk-stack.ts`
- `vidrom-cdk/start-instances.sh`
- related design and ops docs that reference signaling endpoints

## Do This When AWS Is Unlocked

### 1. Reconfirm local infra state before shipping

From `vidrom-cdk`:

```sh
npm run build
npx cdk diff
```

Only continue if the diff still matches the expected secure entry-point changes.

### 2. Deploy the secure signaling edge

Canonical deploy path:

```sh
cd vidrom-cdk
./cdk-deploy.sh
```

If CloudFormation is still stuck in `UPDATE_ROLLBACK_FAILED`, recover first:

```sh
aws cloudformation continue-update-rollback \
  --stack-name VidromSignalingStack \
  --resources-to-skip ApiLambda91D2282D \
  --region us-east-1
```

Then rerun:

```sh
cd vidrom-cdk
./cdk-deploy.sh
```

### 3. Verify the live secure endpoints

Confirm:

1. `https://signaling.vidrom.com` is reachable with the expected certificate
2. `wss://signaling.vidrom.com` accepts secure WebSocket connections
3. the backend remains reachable only through the intended TLS edge path

### 4. Run mobile verification against production endpoints

Verify these end-to-end behaviors:

1. the home app resolves the production HTTP base URL to `https://signaling.vidrom.com`
2. the intercom app resolves the production WebSocket base URL to `wss://signaling.vidrom.com`
3. both apps connect successfully without insecure transport fallbacks
4. iOS no longer needs the production ATS insecure-load exception
5. a real call flow still works over the secure endpoints

### 5. Re-check the exit criteria

Mark Step 12-A3 complete in production only when all are true:

1. production signaling traffic uses HTTPS and WSS only
2. the secure signaling hostname is live and reachable
3. iOS no longer relies on a production insecure transport exception
4. the mobile apps complete a real signaling flow over the secure endpoints
5. docs and scripts point only at the secure production endpoints

## Completion Trigger

After the CDK deploy and live secure-endpoint verification succeed, move this final cutover step out of `todo` into the appropriate completed location and keep Step 12 focused on the remaining post-deploy work that is still outstanding.