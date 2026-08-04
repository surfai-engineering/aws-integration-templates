# Changelog

## v1 — 2026-08-04 (3)

Added `poc/v2/`, a revision of the minimal-permission templates whose inline `SurfAiIntegrationReadOnly` policy is 9 statements and 19 actions. Relative to `poc/v1/` it drops `RDSReadOnly` (`rds:DescribeDBInstances`), `S3ReadOnly` (`s3:ListAllMyBuckets`), the whole `OrganizationsReadOnly` statement (`organizations:ListAccounts`, `organizations:DescribeOrganization`) and `cloudwatch:ListMetrics`. `poc/v1/` is left in place and unchanged, so anything already deployed from it keeps resolving the template it was deployed with.

Two consequences worth stating rather than discovering later. Without `organizations:ListAccounts`, the management-account role cannot enumerate the organization's accounts, so that account list has to be supplied to Surf AI by hand. And `cloudwatch:GetMetricData` without `cloudwatch:ListMetrics` can read a metric whose name is already known but cannot discover which metrics exist, which is the usual prerequisite for reading the S3 request metrics this template enables by default.

The role's effective access to RDS and S3 listing does not change either way: the AWS managed `SecurityAudit` policy, which the role still carries, grants `rds:Describe*` and `s3:ListAllMyBuckets` in its own right. Removing them from the inline policy narrows what Surf AI documents as calling, not what the role is able to call.

`poc/v2/surfai-integration-role-org.yaml` points its StackSets at `poc/v2/surfai-integration-role.yaml`, and the publish workflow syncs `poc/v2/` alongside the existing prefixes.

## v1 — 2026-08-04 (2)

Added `poc/v1/`: a variant of both templates granting a minimal permission set, for evaluations that do not need the full integration scope. The role there carries the AWS managed `SecurityAudit` policy and an inline `SurfAiIntegrationReadOnly` policy enumerating 24 read-only actions, instead of `ReadOnlyAccess`, `CloudWatchReadOnlyAccess` and `CloudWatchLogsReadOnlyAccess`. It has no access to resource contents — S3 objects, DynamoDB items, CloudWatch Logs events, SQS message bodies — nor to SSM Parameter Store values. The `ManagedPolicyArns` parameter does not exist there; the grant is fixed by the template.

`poc/v1/surfai-integration-role-org.yaml` points its StackSets at `poc/v1/surfai-integration-role.yaml`, so an organization deployment gets the same minimal role in every member account.

In that variant the `use-existing` adoption validator requires a pre-existing role to be equivalent to one the stack would create: exactly the required managed policies, and inline policies granting exactly the documented actions — no missing and no extra. Wildcard actions, `Deny` statements, `NotAction`/`NotResource`, `Condition` blocks, a `Resource` narrower than `*`, and a permissions boundary on the role are each rejected rather than interpreted, since a check that guessed at their effect could pass a role granting more or less than it appears to.

`v1/` is unchanged except for two fixes that apply regardless of the permission set:

- The adoption validator paginates `list_attached_role_policies` and `list_role_policies`, and fails if the role carries a permissions boundary. Truncated results previously produced a misleading "missing required managed policies" reason, and a boundary could silently remove access the role appeared to grant.
- README: called out explicitly that `ReadOnlyAccess` reaches end-user personal data where it is stored in AWS, notably Cognito user records with email addresses, phone numbers and custom attributes — a privacy consideration the previous wording left inside "and more, across most AWS services".

## v1 — 2026-08-04

Added a `ManagedPolicyArns` parameter (both templates) controlling which IAM managed policies the role gets, instead of a fixed set. Defaults to the same four policies this template already used (`SecurityAudit`, `ReadOnlyAccess`, `CloudWatchReadOnlyAccess`, `CloudWatchLogsReadOnlyAccess`), so an existing stack updated to this revision with default parameters is unaffected. Lets Surf AI send each customer a CloudFormation console link with a permission set pre-filled to their needs instead of a fixed take-it-or-leave-it list.

- The `use-existing` adoption validator's required-policy check is no longer a module-level Python constant - it now reads the list from the custom resource's `RequiredPolicyArns` property (itself sourced from `ManagedPolicyArns`), so adopting a role is validated against whatever this stack's parameter says, not a fixed set baked into the Lambda.
- The org template forwards `ManagedPolicyArns` to both StackSets (`surfai-integration-role`, `surfai-integration-role-adopted-accounts`), so one org-level deploy applies the same customization to the management-account role and every member-account role consistently.

## v1 — 2026-08-03 (2)

Org template:

- Replaced `IfRoleAlreadyExists` (a per-account fact that can't be a single org-wide setting) with `AlreadyConnectedAccountIds`, a list of accounts already connected individually. The org template now deploys two StackSets: `surfai-integration-role` creates the role fresh everywhere else (`fail`), and a new `surfai-integration-role-adopted-accounts` StackSet adopts and validates the existing role only in the listed accounts (`use-existing`, with `AutoDeployment` explicitly disabled — an account added later has no pre-existing role to adopt, so it gets a fresh one from the first StackSet instead of being swept into adoption). Both StackSets reference the published standalone template via `TemplateURL` instead of an embedded copy, which also keeps the org template under CloudFormation's 51,200-byte inline-template limit.

Reliability:

- Both Lambdas' `urllib.request.urlopen()` calls now pass `timeout=3` — previously unset, which defaults to no timeout and could block a retry attempt indefinitely, defeating the timeout watchdog. The S3 metrics sweep's watchdog reserve was raised from 15 to 20 seconds to leave more headroom for a slow response delivery.

Docs:

- Corrected the S3 request metrics cost estimate: 16 CloudWatch custom metrics per bucket at $0.30/metric/month, ~$4.80/month per bucket that receives requests in a given month (not "a small ongoing cost").

## v1 — 2026-08-03

Added `surfai-integration-role.yaml`: a single-account variant, for connecting one account at a time without AWS Organizations. Contains the same role definition and behavior as the template the org-level StackSet deploys into each member account, with the same parameters (`ExternalId`, `IfRoleAlreadyExists`, `EnableS3RequestMetrics`, `AutoEnableNewBuckets`).

Trust and validation:

- **Trust principal is the account root, scoped by an `aws:PrincipalArn` condition rather than naming a specific role ARN directly**, on both the `AllowSurfAIAssumeRole` and `AllowSurfAITagSession` statements, in both the member-account role and the management-account role. Naming a role directly as `Principal` makes IAM freeze that role's internal unique ID into the trust policy at save time; deleting and recreating that role would silently break every already-deployed customer's trust policy, with a misleading validator error blaming "an unexpected principal." `aws:PrincipalArn` is a global condition key, evaluated at request time as a plain string comparison, so it isn't subject to this. Live-verified: a real two-hop `AssumeRole` call carrying explicit session tags (the production shape) succeeds against the `TagSession` statement's condition.
- The `use-existing` adoption validator: matches managed policies by ARN, not name, so a customer-managed policy merely *named* the same as a required AWS-managed policy no longer passes as if it were the real one; rejects unrecognized trust-statement shapes (e.g. `Principal: "*"`, a `Principal` dict with a non-`AWS` key present, a single `Statement` object instead of a list) with a clear reason instead of crashing; still accepts a bare account-root principal with no condition, since that is what the manual-setup guide documents and existing manually-created roles use.

Reliability:

- `send()` (in both Lambdas) retries the CloudFormation response delivery up to 3 times with backoff, and raises a dedicated `ResponseDeliveryError` - not a generic exception - if all attempts fail, so it escapes the handler uncaught and Lambda's own async-invocation retry (2 further attempts) still fires. The handler explicitly re-raises `ResponseDeliveryError` before its generic exception handler, so a pure delivery hiccup is never mistakenly reported as a failed validation or a failed sweep - which would otherwise roll back the stack and delete the IAM role over a network blip, not an actual problem.
- The S3 metrics sweep checks `context.get_remaining_time_in_millis()` and stops early with a partial-progress result if time is running low, instead of risking a hard Lambda timeout with no response sent at all. The resulting message only promises a future daily sweep will cover the rest when `AutoEnableNewBuckets` is actually `true`.
- The S3 metrics sweep paginates `ListBuckets` (the previous unpaginated call is rejected outright by AWS above the 10,000-bucket quota) and reuses one S3 client per region instead of constructing a new one per bucket.
- A single inaccessible bucket (e.g. SCP-restricted) no longer fails the whole custom resource and rolls back the stack - per-bucket failures are reported in the resource's reason text instead.
- Both Lambdas log to a stack-unique custom CloudWatch Log Group (`/surfai/integration/<stack-name>-role-validation` / `-s3-metrics`) with 30-day retention, instead of the Lambda-default group, which is auto-created with no retention on first invocation and would fail with "already exists" if declared as a resource at that same default name on a redeploy.

Org template:

- No longer creates a management-account role under `CallAs=DELEGATED_ADMIN` - that role would have been created in the delegated administrator account (a normal member account for deployment purposes, not the true management account), colliding with the StackSet's own instance there while never reaching the real management account. Documented the existing `CreateManagementAccountRole=false` escape hatch for a management account already connected to Surf AI individually before deploying this template.
- `MaxConcurrentPercentage` reduced from 100 to 25.

Docs:

- README corrected: the role can read several places secrets commonly end up (Lambda/ECS environment variables, EC2 user-data, non-`NoEcho` CloudFormation parameters, SSM `SecureString` under the default KMS key) - only Secrets Manager values are actually out of reach. The External ID is documented as not a secret, per AWS's own guidance. Added notes that S3 request metrics are never automatically disabled (with the manual command to remove them), and that the org-level StackSet's top-level operation status can report success despite individual account failures.

Known, intentional gaps:

- Partition-hardcoding on Surf AI's own side (`861276082564`) is left as-is: AWS does not support cross-partition role trust, so a GovCloud or China-partition customer could not complete this integration regardless of how partition-aware the rest of the template is.

## v1 — 2026-07-28

Initial public release. `surfai-integration-role-org.yaml`:

- Organization-level CloudFormation StackSet, deploys `surfai-integration-role` to every account in a chosen OU (or the org root), plus the management account directly.
- Trust scoped to Surf AI's AWS account with a required `ExternalId` (minimum 20 characters, matching AWS's own external-ID character set) on `sts:AssumeRole`. See README, "Why `sts:TagSession` carries no condition", for why the session-tagging statement is unconditioned.
- Attached policies: `SecurityAudit`, `ReadOnlyAccess`, `CloudWatchReadOnlyAccess`, `CloudWatchLogsReadOnlyAccess`.
- S3 request-metrics enablement on all buckets, including future ones (on by default; opt-out via `EnableS3RequestMetrics`).
- Supports adopting an existing `surfai-integration-role` instead of creating a new one, with live validation of its trust policy and attached policies.

If you deployed a copy of this template published before 2026-07-28, update your stack to pick up the current version.
