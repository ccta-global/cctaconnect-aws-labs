# AWS IAM Roles and Cross-Account Access

> Practical guide for cloud engineers, platform teams, security teams, and AWS administrators.  
> Examples use fictional account IDs and role names. Replace every placeholder before deployment.

## 1. What is AWS IAM?

AWS Identity and Access Management (IAM) is the AWS service used to authenticate principals and authorize their actions on AWS resources. IAM policies express permissions using **Effect**, **Action**, **Resource**, and optional **Condition** elements. Access is denied by default; an applicable explicit deny overrides an allow.

### Core IAM building blocks

| Component | What it represents | Typical use |
|---|---|---|
| User | IAM identity with long-term credentials (where configured) | Legacy or exceptional user-specific access; prefer federation for workforce |
| Group | Collection of IAM users | Organizing permissions for legacy IAM users |
| Role | Assumable identity with permissions and temporary credentials | EC2/Lambda workloads, federation, cross-account access, automation |
| Policy | JSON permissions document | Allow/deny actions on resources under conditions |
| Trust policy | Resource-based policy attached to a role | Defines who can assume that role |
| Permission boundary | Maximum permissions an identity-based policy can grant | Delegated administration guardrail |
| SCP | AWS Organizations maximum-permission guardrail for member accounts/OUs | Organization-wide restrictions; does not grant permissions |
| STS | AWS Security Token Service | Issues temporary credentials for role sessions |

## 2. What is an IAM role?

An IAM role is an identity that has a set of permissions but is not permanently associated with one human or application. A trusted principal assumes the role and receives **temporary security credentials** from AWS STS. The credentials include an access key ID, secret access key, and session token, and expire.

Roles are used for:
- **AWS service access:** EC2 instance profiles, Lambda execution roles, ECS task roles.
- **Human access:** workforce federation and IAM Identity Center permission sets/roles.
- **Cross-account delegation:** a principal in Account A assumes a role in Account B.
- **Third-party access:** a vendor assumes a narrowly scoped role, usually protected with an external ID.
- **CI/CD and automation:** deployment pipelines assume deployment roles without storing permanent AWS keys.
- **Break-glass operations:** tightly controlled, audited emergency access.

Prefer short-lived role credentials over embedded or long-lived access keys.

## 3. How an IAM role is evaluated

A role has two distinct policy concerns:

1. **Trust policy (who may assume it?)** — attached to the role. It names trusted principals and may impose conditions such as MFA, external ID, organization ID, source identity, or principal tags.
2. **Permissions policy (what can the assumed session do?)** — identity-based policies attached to the role. It grants the role session specific actions on specific resources.

A trust policy does not grant S3, EC2, or other service permissions. A permissions policy does not, by itself, let an external principal assume the role.

For cross-account access, the caller also needs permission in its own account to call `sts:AssumeRole`, and the target role must trust that caller/account. Both sides must be configured.

## 4. Cross-account role: detailed explanation

Assume:
- **Account A — Source / Shared Services:** `111122223333`
- **Account B — Target / Production:** `444455556666`
- Source role: `arn:aws:iam::111122223333:role/PlatformEngineer`
- Target role: `arn:aws:iam::444455556666:role/ReadOnlyProduction`

### Request flow

1. A user or workload authenticates in Account A (commonly through IAM Identity Center, an external IdP, or an existing workload role).
2. Its current identity policy allows `sts:AssumeRole` on the target role ARN.
3. Account B's target role trust policy trusts the source principal and any required conditions.
4. The caller invokes AWS STS `AssumeRole`.
5. STS evaluates the request and returns temporary credentials if allowed.
6. The caller uses those credentials to call AWS APIs in Account B.
7. The role session is constrained by the role's permissions and any applicable session policy, permissions boundary, SCP, resource policy, and explicit deny.
8. CloudTrail records the role assumption and subsequent API activity in the relevant account/region context.

![Cross-account role flow](aws_cross_account_role_diagram.png)

### Example A — Source-side identity policy

Attach this to `PlatformEngineer` (or the source identity that will assume the target role):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AssumeReadOnlyProduction",
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::444455556666:role/ReadOnlyProduction"
  }]
}
```

This grants the source principal permission to request the role. It does not grant the target role's read permissions directly.

### Example B — Target role trust policy

Attach this as the trust relationship on `ReadOnlyProduction` in Account B:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "TrustPlatformEngineerFromSharedServices",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111122223333:role/PlatformEngineer"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

For production, trust a specific source role rather than an entire account whenever practical. If you trust an account principal, remember that the source account's own policies still determine which identities can assume the role.

### Example C — Target role permissions policy

Attach a least-privilege policy to `ReadOnlyProduction`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DescribeInfrastructure",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeVolumes",
        "elasticloadbalancing:DescribeLoadBalancers",
        "cloudwatch:ListMetrics",
        "cloudwatch:GetMetricData"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ReadSpecificApplicationBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::example-prod-application-data"
    },
    {
      "Sid": "ReadObjectsFromApplicationBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::example-prod-application-data/*"
    }
  ]
}
```

Some AWS `List`/`Describe` actions do not support resource-level permissions and therefore require `"Resource": "*"`. Check each action's service authorization reference before narrowing resources.

### Example D — Assume the role with AWS CLI

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::444455556666:role/ReadOnlyProduction \
  --role-session-name platform-readonly-session
```

The response contains temporary credentials. For a safer local workflow, configure a named CLI profile with `role_arn` and `source_profile`, or use IAM Identity Center / an approved credential process. Avoid pasting temporary secrets into shell history, source control, tickets, or logs.

Example profile:

```ini
[profile production-readonly]
role_arn = arn:aws:iam::444455556666:role/ReadOnlyProduction
source_profile = shared-services
region = ap-south-1
```

Then:

```bash
aws sts get-caller-identity --profile production-readonly
aws ec2 describe-instances --profile production-readonly --region ap-south-1
```

## 5. External ID for third-party access

When a SaaS provider or managed service operates in its own AWS account and assumes a role in your account, use a unique external ID generated for your customer relationship. The target trust policy can require it:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "VendorAssumeRole",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::999900001111:root"},
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "customer-unique-value-7f2c"
      }
    }
  }]
}
```

The vendor's source identity must also be allowed to call `sts:AssumeRole`. External IDs mitigate the confused deputy problem; they are not passwords and should not be treated as a substitute for least privilege. Use a unique value per customer/tenant and follow the vendor's secure onboarding process.

## 6. Enterprise usage patterns

### A. Multi-account landing zone

A common organization separates workloads and duties into accounts, for example:
- Management / Organizations
- Security tooling and audit
- Log archive
- Network / shared services
- Development, test, staging, and production workloads

Central identity is commonly provided by an enterprise IdP integrated with **AWS IAM Identity Center**. Permission sets assign job-function access across accounts. For specialized automation or vendor delegation, purpose-built cross-account IAM roles can still be used.

### B. Central security and audit

A security account may assume read-only or security-audit roles in member accounts to inventory IAM, inspect configuration, collect findings, or investigate incidents. Keep remediation permissions separate from read-only investigation permissions. Log archive access should be tightly controlled and protected against deletion or alteration.

### C. CI/CD deployment

A pipeline identity in a tooling account assumes a deployment role in the target workload account. Use separate roles per environment and pipeline purpose, restrict trust to the pipeline's exact identity, and scope permissions to the resources the deployment manages. For external CI platforms, prefer OIDC federation with constrained `aud` and `sub` claims over static AWS keys.

### D. Vendor / partner access

Create one role per vendor integration or responsibility, not a shared administrator role. Require an external ID where appropriate, scope resources and actions, set a bounded session duration, and review access regularly.

### E. Emergency access

Use a separate, monitored break-glass process with strong authentication, limited membership, alerting, and post-use review. Do not use the everyday administrator role as the emergency path.

## 7. Cross-account role vs. resource-based policy vs. IAM Identity Center

| Option | Best fit | Important distinction |
|---|---|---|
| Cross-account IAM role | Delegated access to a target account; broad service/API access | Caller permission plus target trust relationship are both required |
| Resource-based policy (for example S3 bucket policy) | Share a specific supported resource | Policy is attached to the resource; cross-account caller generally also needs identity-based permission |
| AWS RAM | Share supported AWS resources across accounts/OUs/organization | Managed resource-sharing workflow for supported services |
| IAM Identity Center | Workforce access across many AWS accounts | Central user/group assignments and permission sets; AWS manages corresponding roles |

For human workforce access across an AWS Organization, evaluate IAM Identity Center first. Use direct IAM cross-account roles where custom trust, machine-to-machine delegation, vendor access, or specialized automation requires them.

## 8. Enterprise security checklist

- Use least privilege: narrow actions, resources, and conditions; avoid `Action: "*"` and `Resource: "*"` unless justified.
- Prefer IAM Identity Center/federation for people and IAM roles for workloads; avoid long-lived access keys.
- Trust exact role ARNs where practical; avoid broad account principals and wildcard principals.
- Add MFA conditions for supported human-assumption flows; test the actual federation path before enforcing.
- Use `sts:ExternalId` for third-party confused-deputy protection.
- Consider `aws:PrincipalOrgID`, source identity, principal tags, and session tags where they fit the trust model.
- Set a sensible maximum session duration; use shorter sessions for sensitive roles.
- Use session policies for additional per-session restriction when needed.
- Apply permissions boundaries for delegated role administration.
- Use SCPs as organization guardrails; remember SCPs do not grant permissions.
- Enable organization-wide CloudTrail and protect audit logs in a dedicated log archive account.
- Review IAM Access Analyzer findings and unused access; validate policies before deployment.
- Monitor `AssumeRole` activity, unexpected principals, unusual regions, and privilege changes.
- Separate read-only, deployment, security-remediation, and administrator roles.
- Test both allowed and denied paths in a non-production account.

## 9. Troubleshooting checklist

| Symptom | Check |
|---|---|
| `AccessDenied` on `AssumeRole` | Source identity policy allows exact target role ARN; target trust principal and conditions match |
| AssumeRole works, API call fails | Target role permission policy, resource policy, region, resource ARN, and explicit denies |
| Works in dev but not prod | Prod trust policy, SCPs, permissions boundaries, session policies, KMS key policy, resource policy |
| Vendor cannot assume role | External ID exact match, vendor source account/role, source-side `sts:AssumeRole` permission |
| Console switch role fails | Correct account ID/role name, caller authorization, trust policy, session restrictions |
| S3 access denied | Bucket policy, role identity policy, object ARN vs bucket ARN, KMS key policy, VPC endpoint policy |

Use IAM Policy Simulator and IAM Access Analyzer as supporting tools, but validate with a controlled test in the actual target account. Policy evaluation can involve service-specific resource policies and organization controls.

## 10. Suggested implementation runbook

1. Identify the source principal, target account, business purpose, and data classification.
2. Decide whether IAM Identity Center, a cross-account role, resource policy, or AWS RAM is the best fit.
3. Create a target role with a narrow trust policy.
4. Attach a least-privilege permissions policy to the target role.
5. Grant the source principal only `sts:AssumeRole` on the exact target role ARN.
6. Add external ID / MFA / organization / tag conditions as appropriate.
7. Test positive and negative cases; verify the caller identity after assumption.
8. Confirm CloudTrail visibility and alerting.
9. Document owner, purpose, expiry/review date, and break-glass path.
10. Review permissions and trust relationships periodically.

## 11. Example account layout

```text
AWS Organization
|
+-- Management account
|   +-- Organizations, billing, delegated administration
|
+-- Security account
|   +-- Security tooling, delegated audit roles
|
+-- Log Archive account
|   +-- Centralized, protected CloudTrail / service logs
|
+-- Shared Services account
|   +-- CI/CD, identity integrations, network services
|
+-- Workloads OU
    +-- Development account
    +-- Test account
    +-- Production account
```

## 12. Important notes and limitations

- Account IDs, role names, bucket names, and external IDs in this guide are fictional examples.
- IAM policy examples are templates, not a substitute for reviewing the exact AWS service actions and resource types.
- A role trust policy is not the same thing as a role permissions policy.
- Cross-account role assumption does not make the source principal an administrator in the target account; it grants only the target role session's effective permissions.
- Console screenshots are not included because no AWS account or console session was provided. The architecture diagram is conceptual and intentionally labeled as such. For production documentation, capture screenshots from a sandbox account after implementing and reviewing the exact policies.

## 13. Official AWS references

- [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [Common scenarios for IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_common-scenarios.html)
- [Cross-account resource access in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies-cross-account-resource-access.html)
- [Policies and permissions in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)
- [Temporary security credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html)
- [IAM security best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [IAM Identity Center permission sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html)
- [AWS CloudTrail and IAM](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/security_iam_service-with-iam.html)

---
**Document purpose:** learning and implementation reference. Review all policies with your AWS security team before production use.
