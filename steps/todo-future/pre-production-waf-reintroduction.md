# Future Task - Pre-Production WAF Reintroduction and Validation

## Problem

WAF can be removed as a cost-reduction measure while traffic is low, but that should be treated as a temporary development or low-scale operating state rather than the intended long-term production posture.

Before Vidrom is treated as a hardened production system, AWS WAF protections should be restored for the public application entry points that can receive unauthenticated or high-volume traffic.

This is a pre-production hardening and validation task, not an immediate development blocker.

## Why Bring WAF Back

- restore managed protection for common web attack patterns
- restore rate limiting before requests reach ALB, API Gateway, Lambda, and EC2
- reduce backend cost amplification from abusive or noisy traffic
- restore an application-layer control plane separate from app authentication
- re-establish production parity with the earlier hardened AWS design intent

## Scope

Reintroduce WAF for the public Vidrom application surfaces that benefit from edge or regional request filtering:

1. signaling ALB for `signaling.vidrom.com`
2. portal CloudFront distribution for `portal.vidrom.com`

Static marketing sites `vidrom.com` and `vidrom.co.il` should remain a separate decision. They are not part of this task unless their exposure or traffic profile changes.

## Required Changes

### 1. Restore CDK WAF resources

In `vidrom-cdk/lib/vidrom-cdk-stack.ts`:

- re-add the regional WAF for the signaling ALB
- re-add the ALB WAF association
- re-add the CloudFront WAF for the portal distribution
- reattach the portal distribution `webAclId`

### 2. Reapply baseline rules

At minimum, restore:

- AWS managed common rule protections
- rate-based protection for obvious abusive traffic

Review whether the production baseline should also include:

- bad-input or known-bad-input managed rules
- IP reputation managed rules
- tighter portal-specific rate limits
- path-specific protections for admin and management API routes

### 3. Validate production behavior after reintroduction

Confirm that the restored WAF does not break legitimate flows:

- `https://signaling.vidrom.com/healthz` still returns `200`
- signaling HTTP to HTTPS redirect behavior still works
- WebSocket upgrade and authenticated call flows still work
- `portal.vidrom.com` loads normally
- admin and management login still work
- authenticated portal API requests still succeed

## Test Plan

Before closing this task, run and record:

1. **Infrastructure verification**
   - confirm WAF ACLs exist in AWS
   - confirm the signaling ALB is associated with the intended regional ACL
   - confirm the portal CloudFront distribution is associated with the intended CloudFront ACL

2. **Positive smoke tests**
   - `GET /healthz` succeeds
   - unauthenticated portal page load succeeds
   - authenticated admin and management flows succeed
   - home and intercom clients still complete RTC config retrieval and call setup

3. **Negative / abuse tests**
   - high-rate repeated requests trigger rate limiting
   - obviously malformed requests are blocked when they should be
   - blocked requests are observable in WAF metrics or logs

4. **Observability checks**
   - WAF metrics are visible in CloudWatch
   - alerting or at least dashboard visibility exists for sudden blocked-request spikes

## Deployment Sequence

1. restore the WAF resources in CDK
2. run CDK diff and verify only intended WAF associations and ACL resources change
3. deploy the CDK stack
4. verify ALB and portal remain healthy
5. run positive and negative smoke tests
6. record any rule tuning needed to reduce false positives

## Exit Criteria

This task is complete when:

- WAF is reattached to the signaling ALB
- WAF is reattached to the portal CloudFront distribution
- baseline managed and rate-limit rules are active
- legitimate signaling and portal traffic is verified working
- basic abuse tests show requests are filtered before reaching the app tier
- any accepted gaps or deferred WAF tuning are documented