# Surf AI AWS Integration Templates

CloudFormation templates for connecting your AWS account or organization to Surf AI.

Deploying one of these grants Surf AI a read-only role in your account (or your entire AWS Organization, for the org-level template) so we can pull in the data our product analyzes. This repository is the exact source of what runs — review it before you deploy, and check back here to see what changed between versions.

## Templates

### `v1/surfai-integration-role-org.yaml`

Organization-level deployment via CloudFormation StackSet. Run it from your AWS Organizations management account (or a registered delegated administrator) and it will:

- Create a cross-account IAM role, `surfai-integration-role`, in every account across the Organizational Unit(s) you choose — including accounts added later, unless you exclude them.
- Also create the role directly in the management account, since StackSets can't deploy there.
- Enable S3 request metrics on your buckets, existing and future, by default (turn off with the `EnableS3RequestMetrics` parameter).

If some accounts in scope already have `surfai-integration-role` from a prior standalone connection, list them in `AlreadyConnectedAccountIds`. This template deploys two StackSets: `surfai-integration-role` creates the role fresh in every other account (fails if the role is already there), and `surfai-integration-role-adopted-accounts` adopts and validates the existing role in the accounts you listed, without touching the ones created later — that split, not a single shared setting, is what lets a mixed org (some accounts already connected, most not) work correctly. Both StackSets tolerate per-account failures so that one bad account doesn't block the rest, which means the top-level operation can report `SUCCEEDED` even when some individual accounts failed. Always check **CloudFormation → StackSets → surfai-integration-role → Stack instances** and **CloudFormation → StackSets → surfai-integration-role-adopted-accounts → Stack instances** for the real per-account status rather than trusting the top-level operation result.

### `v1/surfai-integration-role.yaml`

Single-account deployment — for connecting one AWS account at a time, with no AWS Organizations requirement. It contains the same role definition and behavior as the template the org-level StackSet deploys into each member account, published standalone for direct use. Run it in the account you want to connect (console upload or `aws cloudformation create-stack`, once per account, any single region) and it will:

- Create the same cross-account IAM role, `surfai-integration-role`, in that account.
- Optionally adopt and validate a pre-existing `surfai-integration-role` instead of creating one (`IfRoleAlreadyExists=use-existing`).
- Enable S3 request metrics on the account's buckets, existing and future, by default (turn off with the `EnableS3RequestMetrics` parameter).

The stack's **Outputs** tab shows the Role ARN and region to send to your Surf AI representative.

Unlike the org-level template, this one can be deployed by any IAM administrator with access to a single account — there's no requirement for organization-level or centrally-reviewed access. If your organization wants a central approval point before granting Surf AI access to any account, prefer the org-level template, or make sure your own internal process reviews standalone deployments.

## What the role can access

The role is read-only, but its scope is broader than basic inventory or config visibility — it can read resource *content*, not just metadata:

- `SecurityAudit` and `ReadOnlyAccess` (AWS managed policies): S3 object contents, DynamoDB items, CloudWatch Logs events, SQS message bodies (via `ReceiveMessage`), and more, across most AWS services.
- `CloudWatchReadOnlyAccess` and `CloudWatchLogsReadOnlyAccess`: a small amount of additional coverage `ReadOnlyAccess` doesn't include.

**This also means the role can reach several places secrets commonly end up, even though it has no direct access to a secrets manager:**

- Lambda and ECS environment variables, in plaintext (`lambda:GetFunctionConfiguration`, `ecs:DescribeTaskDefinition`).
- EC2 instance user-data (`ec2:DescribeInstanceAttribute`).
- CloudFormation stack parameters that aren't marked `NoEcho` (`NoEcho` parameters are masked in the API response).
- SSM Parameter Store `SecureString` values encrypted with the account's default `aws/ssm` key — that key's own resource policy grants decrypt to any principal in the account calling through SSM, regardless of that principal's own IAM permissions. Parameters encrypted with a customer-managed KMS key are not affected unless that key's own policy grants the same access.

**Not included:** `secretsmanager:GetSecretValue`. Secrets Manager values specifically are out of reach — that exclusion is narrower than "no secrets," so check the list above against how your organization actually stores credentials.

Trust requires a unique `ExternalId`, issued to you when you set up the integration, so that Surf AI's own systems can't be tricked into assuming the wrong customer's role. The External ID itself is not a secret — [AWS documents it as a confused-deputy safeguard, not an authentication factor](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html), and this role can read its own trust policy (and therefore its own External ID) via `iam:Get*`. Treat it like any other non-sensitive configuration value, not a password.

## S3 request metrics: not automatically cleaned up

Enabling S3 request metrics (`EnableS3RequestMetrics=true`, the default) publishes 16 CloudWatch custom metrics per bucket. At $0.30/metric/month (AWS's standard custom-metric rate), that's about **$4.80/month per bucket that actually receives requests** — buckets with no request activity in a given month are not billed for that month's metrics. Neither deleting the stack nor changing this parameter back to `false` removes the metrics configuration from your buckets — this is intentional, so that removing the integration never silently disables monitoring you may still be relying on. To turn metrics off, remove the `EntireBucket` request-metrics filter yourself, per bucket:

```
aws s3api delete-bucket-metrics-configuration --bucket <bucket-name> --id EntireBucket
```

## How trust is scoped, and why `sts:TagSession` carries a different condition than `sts:AssumeRole`

The trust policy has two statements, both trusting the Surf AI account root (`arn:aws:iam::861276082564:root`) rather than a specific role ARN, and both carrying a condition that narrows this down to one specific Surf AI role:

```json
"Condition": { "ArnEquals": { "aws:PrincipalArn": "arn:aws:iam::861276082564:role/surfai/integration/surfai-customer-integration" } }
```

This is deliberately not the same as naming that role directly as the `Principal` (which an earlier version of this template did). If an IAM role is named directly as `Principal` and that role is later deleted and recreated - even with the identical name and ARN - AWS resolves and freezes the *old* role's internal unique ID into every already-saved trust policy that named it, silently breaking every deployed customer's trust policy at once. A condition value is a plain string compared at request-evaluation time, not resolved or frozen when the policy is saved, so scoping this way avoids that fragility entirely while restricting access exactly as tightly.

The `AllowSurfAIAssumeRole` statement additionally requires your `ExternalId`, but `AllowSurfAITagSession` does not - and cannot. Surf AI attaches session tags when it assumes the role, so AWS authorizes both `sts:AssumeRole` and `sts:TagSession` as part of the same call. `aws:PrincipalArn` is a global condition key present in every request regardless of action, so it works safely on both statements - but AWS does not support the action-specific `sts:ExternalId` condition key for the `sts:TagSession` action; the key is simply absent from the request context for that action, so a `StringEquals` test on it can never match. An `ExternalId` condition on the `AllowSurfAITagSession` statement therefore denies every tagged assume-role call, and the integration stops ingesting data. Note that both AWS's IAM policy simulator and a plain `aws sts assume-role` call without `--tags` will report success against such a policy, so the failure only shows up in real traffic - this is why `sts:ExternalId` is absent from that statement while `aws:PrincipalArn` is present on both.

This costs you nothing in security: `sts:TagSession` grants no access on its own, and both statements are equally scoped to the same specific Surf AI role via `aws:PrincipalArn`. The two statements must also stay separate - merging them into one statement with both actions would reapply the `ExternalId` condition to `sts:TagSession` and break it again, because a `Condition` block applies to every action in its statement.

## Prerequisites (org-level template only)

- AWS Organizations trusted access for CloudFormation StackSets, enabled (Organizations console → Services → CloudFormation StackSets).
- Deploy from the organization's management account, or an account registered as a delegated administrator for StackSets (set the `CallAs` parameter accordingly). Under `CallAs=DELEGATED_ADMIN`, the management-account role is not created (a delegated administrator account is a normal member account, not the true management account) — if you need the management account's own role for account discovery, deploy the single-account template there directly, or run this template with `CallAs=SELF` from the management account itself.

The single-account template has no prerequisites beyond permissions to create a CloudFormation stack and IAM roles in the target account.

## Versions

See [CHANGELOG.md](./CHANGELOG.md).
