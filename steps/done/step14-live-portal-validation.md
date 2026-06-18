# Step 14 — Live Portal Validation

## Purpose

This file records the final live validation pass used to close Step 14.

## Live State Verified

The following production facts were verified before closure:

1. the `google_subject` migration is present in production
2. the Lambda-backed portal stack was redeployed with Step 14 auth changes
3. unauthenticated admin API requests return `401`
4. unauthenticated management API requests return `401`
5. production operator row counts at closure are:
   - admins: `2`
   - admins with `google_subject`: `1`
   - managers: `1`
   - managers with `google_subject`: `1`
6. `vidromintercom@gmail.com` is a `manager`
7. `vidromintercom@gmail.com` is assigned to `BuildingOne`
8. `vidromintercom@gmail.com` successfully signed in to the management portal
9. management portal scope matches the `BuildingOne` assignment
10. the manager row backfilled `google_subject` after live sign-in

## Closure Decision

Step 14 is marked done.

The remaining legacy admin row without `google_subject` is accepted as a non-blocking follow-up because:

1. the deployed auth path already resolves operators by persisted subject first
2. verified email is used only as a one-time migration fallback for still-unbound legacy rows
3. that admin row will backfill `google_subject` automatically on its first successful sign-in
4. the security hardening objective for the deployed production path is already achieved

## Residual Follow-Up

If `ronenwes@gmail.com` signs in later, re-run the admin-row DB check and confirm the final backfill. This is no longer a blocker for Step 14 completion.

## Completion Note

This validation pass, together with the completed deploy record in [step14-final-production-cutover.md](step14-final-production-cutover.md), closes Step 14.