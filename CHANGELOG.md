# Changelog

## v1 — 2026-07-28

Initial public release. `surfai-integration-role-org.yaml`:

- Organization-level CloudFormation StackSet, deploys `surfai-integration-role` to every account in a chosen OU (or the org root), plus the management account directly.
- Trust scoped to Surf AI's AWS account with a required `ExternalId` (minimum 20 characters, matching AWS's own external-ID character set) on `sts:AssumeRole`. See README, "Why `sts:TagSession` carries no condition", for why the session-tagging statement is unconditioned.
- Attached policies: `SecurityAudit`, `ReadOnlyAccess`, `CloudWatchReadOnlyAccess`, `CloudWatchLogsReadOnlyAccess`.
- S3 request-metrics enablement on all buckets, including future ones (on by default; opt-out via `EnableS3RequestMetrics`).
- Supports adopting an existing `surfai-integration-role` instead of creating a new one, with live validation of its trust policy and attached policies.

If you deployed a copy of this template published before 2026-07-28, update your stack to pick up the current version.
