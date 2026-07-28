# Surf AI AWS Integration Templates

CloudFormation templates for connecting your AWS account or organization to Surf AI.

Deploying one of these grants Surf AI a read-only role in your account (or your entire AWS Organization, for the org-level template) so we can pull in the data our product analyzes. This repository is the exact source of what runs — review it before you deploy, and check back here to see what changed between versions.

## Templates

### `v1/surfai-integration-role-org.yaml`

Organization-level deployment via CloudFormation StackSet. Run it from your AWS Organizations management account (or a registered delegated administrator) and it will:

- Create a cross-account IAM role, `surfai-integration-role`, in every account across the Organizational Unit(s) you choose — including accounts added later, unless you exclude them.
- Also create the role directly in the management account, since StackSets can't deploy there.
- Enable S3 request metrics on your buckets, existing and future, by default (turn off with the `EnableS3RequestMetrics` parameter).

## What the role can access

The role is read-only, but its scope is broader than basic inventory or config visibility — it can read resource *content*, not just metadata:

- `SecurityAudit` and `ReadOnlyAccess` (AWS managed policies): S3 object contents, DynamoDB items, CloudWatch Logs events, SQS message bodies (via `ReceiveMessage`), and more, across most AWS services.
- `CloudWatchReadOnlyAccess` and `CloudWatchLogsReadOnlyAccess`: a small amount of additional coverage `ReadOnlyAccess` doesn't include.
- **Not included:** `secretsmanager:GetSecretValue`. This role can never read your secret values.

Trust is scoped to Surf AI's AWS account and requires a unique `ExternalId`, issued to you when you set up the integration — nobody else can assume this role, and neither can we without it.

## Why `sts:TagSession` carries no condition

The trust policy has two statements: `AllowSurfAIAssumeRole`, which requires your `ExternalId`, and `AllowSurfAITagSession`, which has no condition. That asymmetry is deliberate and required.

Surf AI attaches session tags when it assumes the role, so AWS authorizes both `sts:AssumeRole` and `sts:TagSession` as part of the same call. AWS does not support the `sts:ExternalId` condition key for the `sts:TagSession` action — the key is simply absent from the request context for that action, so a `StringEquals` test on it can never match. An `ExternalId` condition on the `AllowSurfAITagSession` statement therefore denies every tagged assume-role call, and the integration stops ingesting data. Note that both AWS's IAM policy simulator and a plain `aws sts assume-role` call without `--tags` will report success against such a policy, so the failure only shows up in real traffic.

This costs you nothing in security. `sts:TagSession` grants no access on its own; it only permits attaching tags to a session that `AllowSurfAIAssumeRole` has already authorized, and that statement still requires your `ExternalId`. The two statements must also stay separate — merging them into one statement with both actions would reapply the condition to `sts:TagSession` and break it again, because a `Condition` block applies to every action in its statement.

## Prerequisites

- AWS Organizations trusted access for CloudFormation StackSets, enabled (Organizations console → Services → CloudFormation StackSets).
- Deploy from the organization's management account, or an account registered as a delegated administrator for StackSets (set the `CallAs` parameter accordingly).

## Versions

See [CHANGELOG.md](./CHANGELOG.md).
