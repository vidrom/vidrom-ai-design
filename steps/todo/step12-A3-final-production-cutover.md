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

## AWS-Unlocked Validation — 2026-05-28

Read-only checks after the account was unlocked found:

1. `npm run build` passes in `vidrom-cdk`.
2. CDK synth still contains the secure signaling ALB, certificate, Route 53 alias, and `https://signaling.vidrom.com` / `wss://signaling.vidrom.com` outputs.
3. `VidromSignalingStack` is currently `ROLLBACK_COMPLETE`, not an updateable state for the normal deploy path.
4. The only CloudFormation resource not marked `DELETE_COMPLETE` is the old retained portal bucket logical resource, but `head-bucket` for `vidromsignalingstack-portalbucketf34416c0-lx1oiuk6kwpg` returned `404 Not Found`.
5. The expected Elastic IP `eipalloc-034861387a9f4e324` / `52.203.117.37` exists and is currently unassociated.
6. Route 53 currently has no `signaling.vidrom.com.` or `portal.vidrom.com.` record.
7. `https://signaling.vidrom.com/healthz` does not resolve yet, so HTTPS/WSS production verification cannot pass until the stack is recreated/deployed.
8. There is an existing Vidrom RDS instance, `vidromsignalingstack-vidromdatabasef96bc19c-xeyojemby2p5`, that is `available` and still `PubliclyAccessible=true`; keep this visible for Step 12-B1 hardening and before deciding whether to preserve, migrate, or remove it.

Because the stack is `ROLLBACK_COMPLETE`, the next AWS-changing action is for the operator to delete that failed stack record, then rerun the canonical CDK deploy. Do not run `continue-update-rollback` for this state.

## Files Already Landed

- `vidrom-ai-home/config.js`
- `vidrom-ai-home/app.json`
- `vidrom-ai-intercom/config.js`
- `vidrom-cdk/lib/vidrom-cdk-stack.ts`
- `vidrom-cdk/start-instances.sh`
- related design and ops docs that reference signaling endpoints

## Do This When AWS Is Unlocked

Operational safety rule: the assistant may run local-only validation, but the operator runs every AWS-changing command. Keep command output attached to this step before moving it to `done/`.

### 1. Reconfirm local infra state before shipping

From `vidrom-cdk`:

```sh
npm run build
npx cdk diff
```

Only continue if the diff still matches the expected secure entry-point changes.

Use the same deployment parameters for the diff as the deploy script will use:

```sh
cd vidrom-cdk
export AWS_REGION=us-east-1
export RUNTIME_SECRET_ARN="$(aws secretsmanager describe-secret --secret-id vidrom/signaling/runtime --region us-east-1 --query ARN --output text)"
export FIREBASE_SERVICE_ACCOUNT_SECRET_ARN="$(aws secretsmanager describe-secret --secret-id vidrom/signaling/firebase-service-account --region us-east-1 --query ARN --output text)"
export APNS_AUTH_KEY_SECRET_ARN="$(aws secretsmanager describe-secret --secret-id vidrom/signaling/apns-auth-key --region us-east-1 --query ARN --output text)"
npx cdk diff \
  --parameters RuntimeSecretArn="$RUNTIME_SECRET_ARN" \
  --parameters FirebaseServiceAccountSecretArn="$FIREBASE_SERVICE_ACCOUNT_SECRET_ARN" \
  --parameters ApnsAuthKeySecretArn="$APNS_AUTH_KEY_SECRET_ARN"
```

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

If using raw CloudFormation status checks before deploying, this read-only command should show an updateable stack state such as `CREATE_COMPLETE`, `UPDATE_COMPLETE`, or `UPDATE_ROLLBACK_COMPLETE`:

```sh
aws cloudformation describe-stacks \
  --stack-name VidromSignalingStack \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

If this returns `ROLLBACK_COMPLETE`, do not run rollback recovery. A `ROLLBACK_COMPLETE` stack cannot be updated by the normal CDK deploy path. First verify orphaned resources, then have the operator delete the failed stack record:

```sh
aws cloudformation delete-stack \
  --stack-name VidromSignalingStack \
  --region us-east-1

aws cloudformation wait stack-delete-complete \
  --stack-name VidromSignalingStack \
  --region us-east-1
```

After the stack record is deleted, rerun the normal deploy path:

```sh
cd vidrom-cdk
./cdk-deploy.sh
```

If this returns `Stack: VidromSignalingStack does not exist`, do not run rollback recovery. First verify the active account/region and check for orphaned resources that could block a clean stack creation:

```sh
aws sts get-caller-identity --region us-east-1
aws cloudformation list-stacks \
  --region us-east-1 \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE UPDATE_ROLLBACK_COMPLETE ROLLBACK_COMPLETE DELETE_FAILED UPDATE_ROLLBACK_FAILED \
  --query 'StackSummaries[].{Name:StackName,Status:StackStatus,Updated:LastUpdatedTime}'
```

```sh
aws ec2 describe-addresses \
  --allocation-ids eipalloc-034861387a9f4e324 \
  --region us-east-1 \
  --query 'Addresses[0].{PublicIp:PublicIp,AllocationId:AllocationId,InstanceId:InstanceId,AssociationId:AssociationId}'
```

```sh
aws rds describe-db-instances \
  --region us-east-1 \
  --query "DBInstances[?contains(DBInstanceIdentifier, 'vidrom')].{DBInstanceIdentifier:DBInstanceIdentifier,Status:DBInstanceStatus,PubliclyAccessible:PubliclyAccessible,Endpoint:Endpoint.Address}"
```

```sh
aws wafv2 list-web-acls \
  --scope REGIONAL \
  --region us-east-1 \
  --query "WebACLs[?starts_with(Name, 'vidrom-')].{Name:Name,Id:Id,ARN:ARN}"

aws wafv2 list-web-acls \
  --scope CLOUDFRONT \
  --region us-east-1 \
  --query "WebACLs[?starts_with(Name, 'vidrom-')].{Name:Name,Id:Id,ARN:ARN}"
```

```sh
aws route53 list-resource-record-sets \
  --hosted-zone-id "$(aws route53 list-hosted-zones-by-name --dns-name vidrom.com --query 'HostedZones[0].Id' --output text | sed 's#/hostedzone/##')" \
  --query "ResourceRecordSets[?Name=='signaling.vidrom.com.' || Name=='portal.vidrom.com.']"
```

If there is no stack and no conflicting orphaned resource that must be preserved, create from scratch with the normal deploy path:

```sh
cd vidrom-cdk
./cdk-deploy.sh
```

### 3. Verify the live secure endpoints

Confirm:

1. `https://signaling.vidrom.com` is reachable with the expected certificate
2. `wss://signaling.vidrom.com` accepts secure WebSocket connections
3. the backend remains reachable only through the intended TLS edge path

Suggested verification commands after deploy:

```sh
curl -fsS https://signaling.vidrom.com/healthz
```

```sh
openssl s_client -connect signaling.vidrom.com:443 -servername signaling.vidrom.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

```sh
cd vidrom-signaling-server
node - <<'NODE'
const WebSocket = require('ws');
const ws = new WebSocket('wss://signaling.vidrom.com');
const timeout = setTimeout(() => {
  console.error('timeout waiting for WSS open');
  ws.terminate();
  process.exit(1);
}, 10000);
ws.on('open', () => {
  clearTimeout(timeout);
  console.log('wss ok');
  ws.close();
});
ws.on('error', (error) => {
  clearTimeout(timeout);
  console.error(error.message);
  process.exit(1);
});
NODE
```

```sh
aws cloudformation describe-stack-resources \
  --stack-name VidromSignalingStack \
  --region us-east-1 \
  --query "StackResources[?ResourceType=='AWS::EC2::SecurityGroup' && contains(LogicalResourceId, 'SignalingSg')].PhysicalResourceId | [0]" \
  --output text
```

Use the returned signaling security-group ID to confirm port `8080` ingress is only from the ALB security group, not `0.0.0.0/0`:

```sh
aws ec2 describe-security-groups \
  --group-ids <signaling-security-group-id> \
  --region us-east-1 \
  --query 'SecurityGroups[0].IpPermissions[?FromPort==`8080`]'
```

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

## AWS Production Verification — 2026-05-29

All infrastructure exit criteria verified:

### HTTPS and WSS endpoints

```
$ curl -s https://signaling.vidrom.com/healthz
{"ok":true}
```

```
$ openssl s_client -connect signaling.vidrom.com:443 -servername signaling.vidrom.com 2>/dev/null </dev/null | openssl x509 -noout -subject -issuer -dates
subject=CN=signaling.vidrom.com
issuer=C=US, O=Amazon, CN=Amazon RSA 2048 M04
notBefore=May 28 00:00:00 2026 GMT
notAfter=Dec 11 23:59:59 2026 GMT
```

```
$ node wss-test.js wss://signaling.vidrom.com
wss error (expected if auth required): Unexpected server response: 403
```
(403 confirms: TLS upgrade path works; server auth middleware rejects unauthenticated connections correctly.)

### Network isolation

Port 8080 on the EC2 security group (`sg-0a667159c03078fdc`) accepts ingress only from ALB security group `sg-0e4a551fe4e374075` — no direct public internet access.

### Database and service state

- All 12 SQL migrations (001–012) applied to RDS
- Service restart after migrations: `[RECOVERY] No active calls to recover` — no schema errors
- Service running as `vidrom` user, port 8080, `NODE_ENV=production`
- APNs VoIP push initialized (production) on startup

### Exit criteria status

| Criterion | Status |
|---|---|
| 1. Production signaling traffic uses HTTPS and WSS only | ✅ ALB terminates TLS; EC2:8080 blocked from internet |
| 2. Secure signaling hostname live and reachable | ✅ `healthz` returns `{"ok":true}` |
| 3. iOS no longer relies on ATS insecure exception | ✅ Removed in local Step 12-A3 work (app.json) |
| 4. Mobile apps complete real signaling flow over secure endpoints | ⏳ Requires physical device test |
| 5. Docs and scripts point only at secure production endpoints | ✅ config.js, app.json, CDK, ops docs all updated |

### Remaining: Mobile app verification (criterion 4)

The only outstanding exit criterion is a live end-to-end call test from the mobile apps against the production `https://` and `wss://` endpoints. This requires running the home app and intercom app against production and completing a real call flow.

## Completion Trigger

After the CDK deploy and live secure-endpoint verification succeed, move this final cutover step out of `todo` into the appropriate completed location and keep Step 12 focused on the remaining post-deploy work that is still outstanding.