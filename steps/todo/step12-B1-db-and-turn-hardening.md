# Step 12-B1 — DB And TURN Hardening (12.4)

## Status

Implemented in code, pending deploy verification.

The repo changes for DB hardening and runtime TURN credentials have landed, but this step should stay in `todo/` until the deployed AWS environment is verified after the account/deploy blocker is cleared.

## Problem

The current infra model exposes more surface area than necessary:

- RDS is publicly accessible
- database ingress is broader than ideal
- TURN uses static credentials

## Dependencies

A2 should land first so secret storage and runtime TURN config are already in place.

## What To Do

### Database

1. Move RDS access behind a tighter boundary.
2. Remove broad public ingress for PostgreSQL.
3. Prefer application-layer access from trusted components only.
4. Revisit whether Lambda should reach DB directly or through a controlled private path.

### TURN

1. Replace static TURN credentials with time-limited credentials.
2. Generate credentials per session or short interval.
3. Expose TURN config from the server to clients instead of baking it into source.

## Files Touched

| File | Change |
|------|--------|
| `vidrom-cdk/lib/vidrom-cdk-stack.ts` | Tighten DB network exposure and security groups |
| signaling server config and ICE config endpoint | Generate or broker short-lived TURN credentials |
| mobile config consumers | Use runtime TURN config instead of static source constants |

## Verification

- [x] DB is no longer broadly reachable from the public internet
- [x] TURN credentials rotate or expire automatically
- [ ] Clients can still establish calls in restrictive network conditions

## AWS Production Verification — 2026-05-29

Read-only production checks after the Step 12-A3 AWS cutover verified the deployed hardening state:

### RDS network exposure

The CloudFormation-managed RDS instance is private:

- DB instance: `vidromsignalingstack-vidromdatabasef96bc19c-lltmtpczyz75`
- `PubliclyAccessible`: `false`
- DB security group: `sg-0ef53703f3f79b39e`
- PostgreSQL ingress on port 5432 is security-group based only:
	- signaling server SG `sg-0a667159c03078fdc`
	- portal API Lambda SG `sg-0169c87aa9fa7b72c`
	- no IPv4 or IPv6 CIDR ingress rules

### Runtime TURN config

Production `/api/rtc-config` was called through `https://signaling.vidrom.com` using a short-lived intercom JWT generated from the production runtime secret. Sensitive token and TURN credential values were not printed.

Redacted response summary:

```json
{
	"statusCode": 200,
	"ttlSeconds": 600,
	"hasTurn": true,
	"usernameShape": "EPOCH:intercom",
	"expiryIsEpoch": true,
	"credentialRedacted": true
}
```

This confirms the server is issuing expiring TURN credentials with an epoch-prefixed username instead of static source credentials.

### Deploy bucket IAM cleanup

The manual inline EC2 role policy `VidromDeployBucketAccess` was replaced with the CDK-managed `deployBucket.grantRead(role)` policy in `vidrom-cdk/lib/vidrom-cdk-stack.ts`, deployed successfully, and validated from EC2 with an S3 read probe. The redundant manual inline policy was removed; only the CloudFormation-managed default role policy remains.

### Remaining verification

The only Step 12-B1 item still requiring manual/device validation is media behavior from a restrictive network where TURN relay is actually used.

## Post-Deploy Verification Checklist

1. Deploy the CDK stack and confirm the update completes successfully.
2. Confirm the RDS instance is not publicly accessible and only trusted security groups can reach PostgreSQL.
3. Confirm the signaling service still serves `/api/rtc-config` successfully in production.
4. Inspect a production `/api/rtc-config` response and verify TURN credentials include an expiry and are not static.
5. Place a real intercom-to-home call and verify call setup still succeeds.
6. Re-test from a restrictive network path where TURN relay may be required and confirm media still connects.
7. If all checks pass, move this step to `done/` and mark the verification items complete.

## Operator-Run AWS Verification Commands

Operational safety rule: the assistant may run local-only validation, but the operator runs every AWS command that changes the account. The commands below are read-only unless explicitly noted.

Find the deployed RDS instance and confirm it is not publicly accessible:

```sh
DB_INSTANCE_ID="$(aws cloudformation describe-stack-resources \
	--stack-name VidromSignalingStack \
	--region us-east-1 \
	--query "StackResources[?ResourceType=='AWS::RDS::DBInstance'].PhysicalResourceId | [0]" \
	--output text)"

aws rds describe-db-instances \
	--db-instance-identifier "$DB_INSTANCE_ID" \
	--region us-east-1 \
	--query 'DBInstances[0].{DBInstanceIdentifier:DBInstanceIdentifier,PubliclyAccessible:PubliclyAccessible,DBSubnetGroup:DBSubnetGroup.DBSubnetGroupName,VpcSecurityGroups:VpcSecurityGroups[*].VpcSecurityGroupId}'
```

Find the database security group and confirm PostgreSQL ingress is security-group based only:

```sh
DB_SG_ID="$(aws cloudformation describe-stack-resources \
	--stack-name VidromSignalingStack \
	--region us-east-1 \
	--query "StackResources[?ResourceType=='AWS::EC2::SecurityGroup' && contains(LogicalResourceId, 'DatabaseSg')].PhysicalResourceId | [0]" \
	--output text)"

aws ec2 describe-security-groups \
	--group-ids "$DB_SG_ID" \
	--region us-east-1 \
	--query 'SecurityGroups[0].IpPermissions[?FromPort==`5432`]'
```

Verify runtime RTC config from production with a valid resident JWT:

```sh
VIDROM_AUTH_TOKEN='<resident-jwt>'
curl -fsS https://signaling.vidrom.com/api/rtc-config \
	-H "Authorization: Bearer $VIDROM_AUTH_TOKEN"
```

The response should include `ttlSeconds`, `expiresAt`, and a TURN `username` containing an epoch expiry prefix, for example `<epoch>:home`, instead of a static username.

If the runtime secret needs TURN shared-secret rotation later, the operator should run the Secrets Manager update and then restart/refresh the dependent services through SSM; do not rotate it from an assistant-run terminal.