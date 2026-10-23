# Section 1: IAM — Identity and Access Management

## The idea

Every single thing that happens in AWS — launching a server, reading a file, deleting a database — starts with the same two questions: **who are you?** and **are you allowed to do that?** IAM (Identity and Access Management) is the service that answers both. It's the front door, the ID checker, and the rulebook, all in one — and it's **free** and **global** (not tied to any region).

Here's the analogy to carry through this whole section: **AWS is a giant office building.** IAM hands out the badges. A **user** is a long-lived identity (though AWS recommends workforce users use IAM Identity Center instead of IAM users). A **group** is a department — everyone in "Developers" gets whatever the department is allowed. A **role** is a **temporary costume**: a visitor's vest that a person, workload, another account, or identity provider can assume for a while, gain its powers, and then stop using. And **policies** are the written rules pinned to the wall saying which badges open which doors.

That "temporary costume" idea is the single most exam-tested concept in IAM, so let's start there.

## Users, groups, roles

| Identity | What it is | When to use |
|---|---|---|
| **User** | Long-lived identity with a password and/or long-lived access keys | Legacy or specific cases; for workforce access across accounts, prefer IAM Identity Center |
| **Group** | A bucket of users that share policies. Groups contain **only users** — never other groups, never roles | Manage permissions by team, not per person |
| **Role** | An identity with permissions but **no standard long-term password or access keys**. It's *assumed* — put on like a costume — and provides **temporary credentials** | Services, cross-account access, federated humans, and workloads outside AWS (for example, IAM Roles Anywhere) |

**The golden rule: never put long-lived access keys on compute.** If an EC2 instance needs to read from S3, you do NOT create a user, generate access keys, and paste them into a config file on the box. Keys on a server can leak, never rotate themselves, and show up in every "what's wrong with this architecture?" exam question. Instead:

- **EC2 → attach an instance role** (delivered via an *instance profile*). The instance fetches auto-rotating temporary credentials from its metadata — no keys ever touch the disk.
- **Lambda → execution role.** Same idea: the function assumes a role every time it runs.
- **ECS tasks → task role.** Same pattern again.

**THE trap:** any answer choice that says *"store access keys on the instance / in the AMI / in environment variables / in the code"* is wrong. For AWS workloads, use the appropriate role-based credentials; for non-AWS workloads, consider IAM Roles Anywhere or another supported federation method. If the exam says "credentials found hardcoded in an application on EC2" — the fix is *attach an IAM role to the instance*.

## Policy JSON anatomy

A policy is a JSON document. You don't need to write one on the exam, but you must be able to *read* one. Four load-bearing parts:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowReadReports",
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": "arn:aws:s3:::finance-reports/*",
    "Condition": {
      "Bool": { "aws:MultiFactorAuthPresent": "true" }
    }
  }]
}
```

- **Effect** — `Allow` or `Deny`. That's it, two values.
- **Action** — which API calls (`s3:GetObject`, `ec2:*`...). Service prefix + verb.
- **Resource** — which things, named by **ARN** (Amazon Resource Name — AWS's globally unique ID string for every object it manages).
- **Condition** — optional "only if" clauses: only with MFA, only from this IP range, only over HTTPS.

Resource-based policies (we'll meet them next) add a **Principal** field — *who* the statement applies to — because the policy isn't attached to an identity, so it has to name one.

## How AWS decides yes or no

Memorize this evaluation logic — it's worth several exam questions:

1. **Default is implicit deny.** Nothing is allowed until something allows it.
2. An **Allow** in an applicable policy opens the door.
3. An **explicit Deny in any applicable policy wins** over an Allow.

And for a request to succeed, it must pass through **every applicable gate**:

```
Request ──► Check all applicable policies and request context
            explicit Deny? ── yes ──► ✘ Denied
            otherwise, does an applicable Allow survive all applicable limits?
               (identity/resource policies, boundary/session policy,
                SCP/RCP, and possibly VPC endpoint policy) ──► ✔ / ✘
```

*Important nuance: in the same account, an identity-policy Allow **or** a resource-policy Allow can be sufficient in common cases; boundaries and other policy types have principal-specific evaluation rules. For cross-account access, both the caller's account and the resource-owning account generally must allow the request. SCPs constrain principals in member accounts, not the management account or service-linked roles. RCPs constrain supported resources in member accounts.*

**THE trap:** *"A user's policy clearly says Allow, but the action fails."* The answer is always one of: **an explicit Deny somewhere**, **an SCP on the account** (common when the question mentions AWS Organizations), or **a permission boundary** clipping the identity policy. Allow + Deny = Deny. Every time.

## The policy types table

| Policy type | Attached to | What it does |
|---|---|---|
| **Identity-based** | User, group, or role | The everyday "what can this identity do" |
| **Resource-based** | The resource itself (S3 bucket policy, SQS queue policy...) | "Who can touch *me*" — has a `Principal`, enables cross-account grants |
| **Permission boundary** | A user or role | A **maximum ceiling** for that one identity. Grants nothing itself — effective perms = identity policy ∩ boundary |
| **SCP** (Service Control Policy) | Organization root/OUs/accounts in **AWS Organizations** | A ceiling for IAM users and roles in **member accounts**, including their root users. Does **not** affect the management account or service-linked roles; grants nothing |
| **RCP** (Resource Control Policy) | Organization root/OUs/accounts in **AWS Organizations** | A ceiling on permissions that can be granted to supported resources in member accounts; grants nothing |
| **VPC endpoint policy** | A VPC endpoint | Filters which principals/actions/resources can use that endpoint; it does not replace identity or resource permissions |
| **Session policy** | Passed when assuming a role | Shrinks that one session's permissions even further |
| **ACL** (Access Control List) | S3 buckets/objects | **Legacy** S3-only mechanism. Can't use JSON conditions and **cannot grant to users in its own account** — AWS says avoid; use bucket policies. Don't confuse with **Network ACLs** (a VPC subnet firewall — totally unrelated) |

Ceilings vs grants, one line: **boundary = ceiling for one identity's identity-based permissions; SCP = ceiling for principals in member accounts; RCP = ceiling for supported resources; none grants permissions by itself.**

Exam scenario for boundaries: *"Developers may create their own IAM roles, but must never be able to create roles more powerful than X"* → **permission boundary** (require it on every role they create).

## STS and roles: how the costume gets put on

**STS (Security Token Service)** is the coat-check counter that hands out costumes. When a caller successfully assumes a role, STS returns **temporary credentials** — an access key, secret key, and session token — that expire automatically. For `AssumeRole`, the default session is **1 hour**; the minimum is 15 minutes and the maximum is set on the role (up to 12 hours). Role chaining is limited to **1 hour**. Other federation and service credential flows can have different duration rules. Temporary credentials still need to be protected while valid, but they avoid long-lived keys that must be rotated.

Every role has TWO policies, and the exam loves this:

- **Permissions policy** — what the costume lets you do.
- **Trust policy** — **who is allowed to put the costume on.** It's a resource-based policy on the role itself, naming the trusted principal (a service like `ec2.amazonaws.com`, another account, a SAML provider).

**THE trap:** *"Role has the right permissions but the user/service can't assume it"* → check the **trust policy**. Permissions say what the role can do; trust says who can wear it.

## Cross-account access

Company A's auditors need to read Company B's S3 bucket. The canonical recipe:

1. **Account B (the target)** creates a role with the needed permissions.
2. B sets the role's **trust policy** to trust Account A (and can require MFA or an `ExternalId` — an agreed secret string used with third parties to block the "confused deputy" problem).
3. **Account A's users** get permission to call `sts:AssumeRole` on that role's ARN.
4. They assume it, get temporary credentials, do the work, credentials expire.

No shared passwords, no duplicated users, no keys emailed around. *"Give another AWS account access"* → **cross-account role + trust policy**, essentially always.

## IAM Identity Center (the artist formerly known as AWS SSO)

One human, twelve AWS accounts, twelve passwords? No. **IAM Identity Center** gives your workforce **single sign-on (SSO): one login, a portal, and access to multiple AWS accounts** (plus business apps) — under the hood it just assumes roles in each account for you.

It also **federates**: connect an external identity provider such as **Okta or Microsoft Entra ID** using **SAML 2.0** for sign-in and typically **SCIM** (or manual provisioning) for users and groups. Active Directory can also be used through IAM Identity Center's supported directory options, including AWS Managed Microsoft AD. SAML authenticates users; it does not itself provision/query users and groups. Access is assigned centrally, and disabling a person in the IdP can stop future sign-ins; provisioned assignments and active sessions should also be managed according to your offboarding process.

**Signal decoding:** *"employees already have Active Directory / Okta credentials and shouldn't get separate IAM users"* or *"single sign-on across many AWS accounts"* → **IAM Identity Center**. (For federating *app customers* — sign in with Google/Facebook — that's **Amazon Cognito**, a different topic.)

## MFA and the root account

**MFA (multi-factor authentication)** = a password or other first factor *plus* a second factor (authenticator app, hardware security key/passkey). Policies can check **`aws:MultiFactorAuthPresent`**, but this key is present only for requests made with temporary credentials. A condition using it can behave unexpectedly with long-term IAM user access keys, where the key may be absent. For console/CLI/API MFA enforcement, prefer a carefully designed deny policy that handles missing keys (often `BoolIfExists`) and validate the credential flow; for role access, MFA can also be required in the role's trust policy. *"Require MFA for sensitive actions"* → use the appropriate MFA condition and account for the principal's credential type.

The **root user** (associated with the email used to create the account) has broad account-level privileges and can't have IAM identity policies or a permissions boundary attached. SCPs constrain root in **member accounts**, but do not affect the organization's management account. Best practice:

- **Enable MFA on root** immediately.
- **Delete root access keys** (never create them).
- **Don't use root for daily work** — create an admin IAM identity instead.
- Root only for the handful of root-only tasks (closing the account, changing support plans...).

Two quick tools worth a flashcard: **IAM Access Analyzer** identifies resource access that is external to a zone of trust and can help validate policies; **credential report** summarizes IAM user credentials, while **last accessed information** helps identify unused services/actions for least-privilege review. These tools surface evidence; review before removing access.

## Question patterns

> *"An application on EC2 needs to write to DynamoDB. What's the MOST secure way to provide credentials?"* → **IAM role attached to the instance (instance profile)** — never access keys on the box.

> *"A Lambda function must read from an S3 bucket. How should it authenticate?"* → **Lambda execution role** with S3 read permissions — same costume pattern, serverless flavor.

> *"Users in Account A need temporary access to resources in Account B."* → **Create a role in Account B with a trust policy trusting Account A; users AssumeRole via STS** — "temporary" + "another account" = cross-account role.

> *"A user's identity policy allows s3:PutObject, but uploads fail with Access Denied."* → **Look for an explicit Deny — bucket policy, SCP, or permission boundary** — explicit Deny always beats Allow.

> *"An action works in a standalone account but fails in an account inside AWS Organizations."* → **A Service Control Policy is blocking it** — "Organizations" is the tell.

> *"Developers may create IAM roles for their apps, but security wants a guarantee those roles can never exceed a defined permission set."* → **Permission boundaries** — "maximum permissions" / "cannot exceed" is the signal.

> *"5,000 employees with existing Active Directory logins need access to multiple AWS accounts without creating IAM users."* → **IAM Identity Center with SAML federation to AD** — "existing corporate directory" + "multiple accounts" + "no IAM users".

> *"An application needs short-lived credentials that expire automatically."* → **STS AssumeRole** — "temporary/short-lived credentials" is literally STS's product.

> *"Require that users can only terminate EC2 instances if they've signed in with MFA."* → **Policy Condition with `aws:MultiFactorAuthPresent`** — sensitive action + MFA = that condition key.

> *"A new AWS account was just created. What should be done FIRST to secure it?"* → **Enable MFA on the root user, delete/avoid root access keys, create an admin user for daily work.**

## Pocket card

| Keyword in the question | Answer |
|---|---|
| EC2/app needs AWS access, "most secure" | Instance role (instance profile), never keys |
| Lambda needs permissions | Execution role |
| Hardcoded / stored access keys | Wrong — replace with a role |
| Temporary / short-lived credentials | STS AssumeRole |
| Access across AWS accounts | Role in target account + trust policy |
| Third party assumes your role safely | ExternalId in trust policy |
| Role exists but "can't be assumed" | Fix the trust policy |
| Allow exists but action denied | Explicit Deny / SCP / permission boundary |
| Member account guardrail, including root | SCP (AWS Organizations; exceptions apply) |
| Cap max permissions of ONE user/role | Permission boundary |
| Policy on the resource, has a Principal | Resource-based policy (e.g., bucket policy) |
| Legacy S3 grant mechanism, avoid | S3 ACL (≠ Network ACL) |
| SSO, one login → many AWS accounts | IAM Identity Center |
| Federate Okta / Microsoft Entra ID | IAM Identity Center + SAML 2.0 (SCIM or manual provisioning) |
| Federate app customers (Google/Facebook login) | Cognito |
| Require MFA for an action | MFA condition (credential-flow caveats apply) |
| Root account | MFA on, no access keys, don't use daily |
| Who is my resource shared with externally? | IAM Access Analyzer |
| Find unused users / stale permissions | Credentials report / access advisor |

IAM is the "who may act" layer under everything else — next up, S3, where you'll see these same policies show up on the resource side as bucket policies.


## Current AWS references

- [IAM policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)
- [IAM policy types and permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)
- [AWS Organizations SCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)
- [IAM role session duration](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_manage-assume.html)
- [IAM Identity Center external identity providers](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html)
- [AWS global condition keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html)

