# Project 01 (AWS) – Identity Foundation & Access Lifecycle

**Domain:** Identity and access management (AWS Organizations, IAM Identity Center, IAM)
**Company:** [Magnolia Bay Legal Group](../00-company-scenario/README.md)
**Companion project:** [Project 01 – Azure version](../project-01-identity/README.md)
**Business priorities addressed:** #1 Client confidentiality · #2 Know who has access to what
**Estimated cost:** $0 (AWS Organizations, IAM Identity Center, IAM, SCPs, and the IAM Access Analyzer external-access analyzer are free; the optional EC2 test is pennies if terminated the same day)

> **Why this version exists:** The ticket, company, and staff roster are the same as the Azure build. The point is to solve the *same business problem* on a second platform and document where the platforms differ. The **Translation Notes** section at the end is the most important part of this README.


## 1. The Ticket

> **From:** Diane Fairhope, Managing Partner
> **To:** IT
> **Subject:** Same request, different cloud
>
> Our new co-counsel firm and one of our insurance clients both run on AWS, and we may need to host some case work there. Before we put a single client file in AWS, I want the same controls we have in Azure.
>
> - Everyone needs their own account, set up by department.
> - Jordan is new in IT. He should be able to help people with passwords and restart servers, but I don't want him having the keys to everything yet.
> - Owen in billing needs to see what AWS is costing us. He should not be able to change anything.
> - Co-counsel's attorney needs to *view* the litigation resources, temporarily.
> - Kevin Semmes in Pensacola gave notice. His last day is Friday. Make sure he's locked out the moment he walks out the door.
>
> Send me a one-page summary of who has access to what when you're done.

---

## 2. Requirements & Success Criteria

| ID | Requirement | Done when... |
|---|---|---|
| R0 | Secure the account and build a multi-account structure | Root has MFA, a budget exists, Organizations and Identity Center are enabled, member accounts exist |
| R1 | Create user accounts for the staff roster with department, job title, and office filled in | All 10 roster users exist in Identity Center with correct attributes |
| R2 | Create groups, and solve "access follows attributes" without dynamic groups | Groups exist; at least one permission is driven by a user attribute (ABAC) |
| R3 | Assign access **to groups, not individuals**, at the narrowest scope (account) | Each group can reach only the accounts it needs, verified with the Policy Simulator |
| R4 | Give Jordan limited helpdesk powers and limited server powers | Jordan can reset a standard user's password and start/stop/reboot tagged servers, but cannot create, terminate, or resize anything |
| R5 | Give Owen read-only visibility into costs | Owen can view Billing and Cost Explorer but cannot create or change a budget |
| R6 | Give external co-counsel temporary read-only access to litigation resources only | Co-counsel's identity can assume a role into the litigation account only |
| R7 | Offboard Kevin securely, including sessions already in flight | Kevin's existing sessions stop working, he can't sign in, and his account is removed |
| R8 | Produce an access matrix for the managing partner | One table showing every group/identity and its access |

---

## 3. Environment

| Item | Value |
|---|---|
| Management account | Your existing AWS account (account alias: `mblg-management`) |
| Home region | US East (N. Virginia) `us-east-1` |
| Organization structure | Root → OU `Workloads` → member accounts below |
| Member accounts | `mblg-litigation-prod`, `mblg-realestate-prod`, `mblg-servers-prod` |
| Identity | IAM Identity Center (organization instance) in the management account |
| Tags (all member accounts) | `Owner=it-admin`, `Department=<dept>`, `Environment=prod`, `Project=P01-Identity-AWS` |

> **Where's `rg-identity-prod-01`?** In AWS, Identity Center lives in the management account, so there's no separate identity account in this project. Real organizations often move day-to-day identity admin to a **delegated administrator** member account so fewer people touch the management account. Mention that in your writeup.

> **Account email addresses:** Every AWS account needs a unique email. If your provider supports plus-addressing, use `you+litigation@domain.com`, `you+realestate@domain.com`, and so on. They all arrive in one inbox.

> **Think before creating accounts:** Closing an AWS account isn't instant. Closed accounts stay in a suspended state for a period before they're gone, and AWS limits how many you can close in a short window. Three accounts is fine; don't create and close accounts for fun.

---

## 4. Build Tasks

Each task explains **what** to do and **why**. Click-by-click steps go in your build notes with screenshots, written in your own words after you do them.

### Task 0 – Account structure (R0)

1. Complete **Section 0** above.
2. Enable **AWS Organizations** (all features). Create the `Workloads` OU and the three member accounts inside it. Tag each account.
3. Enable **Service Control Policies** in Organizations (Policies → Service control policies → Enable).
4. Enable **IAM Identity Center** in `us-east-1`, using the default Identity Center directory as the identity source.
5. As root, turn on **IAM user and role access to Billing information** (Account settings). Task 3 needs it.
6. Create an Identity Center user for yourself, put it in an admin group (Task 2), and from then on **stop using root** for daily work.

**Why it matters:**
- **The AWS account is the real security boundary.** An Azure resource group is a folder inside one subscription. An AWS account is a fully separate container with its own IAM, its own resources, and its own blast radius. Putting litigation data in its own account is the AWS way to say "nobody outside this boundary gets in by accident."
- **SCPs are guardrails, not grants.** They set the *maximum* permissions anything in an account can have. Even an administrator in a member account can't do what an SCP denies. You'll use one in Task 6.
- **Root is a break-glass account.** It can't be restricted by IAM policies or SCPs. Lock it down with MFA and don't use it day to day. This is the AWS version of keeping Owner on a break-glass account in Azure.

### Task 1 – Create the users (R1)

Create the 10 roster users from the [company scenario](../00-company-scenario/README.md) in **IAM Identity Center → Users**. Create 2–3 in the console, filling in **Job title**, **Department**, and the office/address fields. Then create the rest from the command line.

Identity Center has no CSV bulk-upload button like Entra ID, so scripting is the "bulk create" here.

**CLI practice (one user):**
```bash
# Find your identity store ID first
aws sso-admin list-instances

aws identitystore create-user \
  --identity-store-id d-xxxxxxxxxx \
  --user-name jordan.dauphin \
  --display-name "Jordan Dauphin" \
  --name GivenName=Jordan,FamilyName=Dauphin \
  --emails Value=jordan.dauphin@example.com,Type=work,Primary=true \
  --title "IT Support Specialist"
```

**Bulk practice:** write a short Bash or PowerShell script that reads a CSV (the same roster you used for Azure) and loops over `create-user`. Commit the script to this repo without any real emails.

> **Check your CLI version's options:** Run `aws identitystore create-user help` and see whether it accepts the enterprise attributes (such as Department). If it doesn't, create the users with the script and fill in Department in the console. Document what you found.

> **Users created by the API don't get a welcome email automatically.** You'll need to send email verification or set a one-time password from the console before they can sign in.

**Why it matters:**
- Same reason as Azure: **attributes drive access.** In Task 2, Department becomes a session tag that policies check, so a typo here breaks access later.
- **Identity Center users are not IAM users.** IAM users live inside one account and usually have long-lived passwords and access keys. Identity Center users sign in once to the AWS access portal and get **temporary credentials** into whichever accounts they're assigned. AWS recommends Identity Center for people and IAM roles for workloads. If an interviewer asks "IAM user or Identity Center?", this is the answer.

### Task 2 – Groups and attribute-based access (R2)

| Group | Members | Azure equivalent |
|---|---|---|
| `SG-IT-CloudAdmins` | You | `SG-IT-AzureAdmins` |
| `SG-IT-Support` | Jordan | Same |
| `SG-Finance-CostReaders` | Owen | Same |
| `SG-Paralegals-All` | All paralegals (assigned manually) | Was **dynamic** |
| `SG-Litigation-Team` | Litigation staff (assigned manually) | Was **dynamic** |

**The gap:** Identity Center groups have no membership rules. There is no `(user.department -eq "Paralegals")`.

**The AWS answer: ABAC (attribute-based access control).**
1. In Identity Center → **Settings → Attributes for access control**, enable it and add the key `Department` mapped to `${path:enterprise.department}`.
2. When a user signs in, Identity Center passes their Department as a **session tag**.
3. Policies can then compare the user's tag with a resource's tag, using `aws:PrincipalTag/Department` and `aws:ResourceTag/Department`.

**ABAC exercise:** In `mblg-litigation-prod`, create an S3 bucket tagged `Department=Paralegals` and another tagged `Department=Attorneys`. Build a permission set `Paralegals-TaggedS3` whose policy allows reading only objects in buckets whose `Department` tag matches the caller's `Department` session tag. Assign it to `SG-Paralegals-All`. Change one test user's Department and confirm their access changes **without touching any group or policy**.

**Why it matters:**
- Azure dynamic groups put the logic in **group membership**. AWS ABAC puts the logic in the **policy**. Both deliver "access follows attributes," but they get there differently. Explaining that difference is a strong interview answer.
- ABAC scales well: one policy covers every department, instead of one policy per department.
- **If you really want dynamic groups in AWS**, the real-world answer is to connect an external IdP (such as your Entra ID tenant) to Identity Center with SAML and SCIM, so Entra's dynamic groups sync over. That's a stretch goal below.

### Task 3 – Permission sets and account assignments (R3, R5)

In Identity Center, a **permission set** is a reusable bundle of policies. Assigning a permission set to a group *on a specific account* creates a role in that account that the group's members can use.

| Who | Permission set (AWS managed policy) | Account(s) | Why this policy and scope |
|---|---|---|---|
| `SG-IT-CloudAdmins` | `PowerUserAccess` | All member accounts | Can build and manage almost everything, but **cannot manage IAM** (users, roles, policies). That's the closest match to Azure Contributor: full build rights, no ability to grant access. Keep `AdministratorAccess` for break-glass only. |
| `SG-Litigation-Team` | `ViewOnlyAccess` | `mblg-litigation-prod` | Can see resources and configuration but **not read data** (for example, it can list S3 buckets but not download objects). See note below. |
| `SG-Finance-CostReaders` | `AWSBillingReadOnlyAccess` | Management account | Billing for the whole organization lives in the management account, so that's where cost visibility has to be granted. |

> **ViewOnlyAccess vs ReadOnlyAccess:** These sound the same but aren't. `ReadOnlyAccess` can *read data*, including downloading S3 objects. `ViewOnlyAccess` sees resource metadata only. Azure's **Reader** role is control-plane only and can't read blob contents, so `ViewOnlyAccess` is the closer match. At a law firm, "read-only" that can download client files is a confidentiality problem. Document this choice.

> **Owen's access is in the management account.** That account is the most sensitive one you have. Billing read-only is acceptable, but note in your writeup that real organizations minimize who can reach the management account at all.

**CLI practice:**
```bash
aws sso-admin create-account-assignment \
  --instance-arn <identity-center-instance-arn> \
  --target-id <litigation-account-id> \
  --target-type AWS_ACCOUNT \
  --permission-set-arn <viewonly-permission-set-arn> \
  --principal-type GROUP \
  --principal-id <SG-Litigation-Team-group-id>
```

**Why it matters:**
- **Scope:** AWS's hierarchy is Organization root → OU → account → resource. Granting access to one account is the AWS version of granting access to one resource group, except the isolation is much stronger.
- **AWS managed vs customer managed vs inline:** AWS managed policies (like `ViewOnlyAccess`) are maintained by AWS and change as services are added. Customer managed and inline policies are yours. You'll write one in Task 4.

### Task 4 – Limited powers for Jordan (R4)

**Part A – Directory side (password resets):**

AWS has no built-in "Helpdesk Administrator" role. You'll build one.

1. Create a permission set `IdentityCenter-Helpdesk`, assigned to `SG-IT-Support` on the management account.
2. Start it with only read access to the Identity Center directory.
3. **Discover the exact actions:** as your admin, reset a test user's password in the console, then open **CloudTrail → Event history** and find the events that action generated. Those event names tell you which API actions the reset needs. Add only those to Jordan's policy.
4. Test as Jordan. Iterate until a password reset works and nothing else does (Jordan should not be able to create users, delete users, or change group membership).

> **The gap:** Entra's Helpdesk Administrator is automatically blocked from resetting admin passwords. Identity Center has no equivalent built-in protection, so a helpdesk policy that can reset passwords may be able to reset *any* user's password. Your compensating controls: **MFA on every user** (a reset password alone doesn't get past MFA) and keeping your break-glass access outside Identity Center (root, with MFA). Write this up honestly. Finding and documenting a gap like this is more impressive than pretending it isn't there.

**Part B – Server side (custom "VM Operator" policy):**

Create a permission set `EC2-Operator` with this inline policy, and assign it to `SG-IT-Support` on `mblg-servers-prod` only:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SeeInstances",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus"
      ],
      "Resource": "*"
    },
    {
      "Sid": "OperateTaggedServersOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": { "aws:ResourceTag/Department": "IT" }
      }
    }
  ]
}
```

*Optional test:* launch a `t3.micro` Amazon Linux instance in `mblg-servers-prod` tagged `Department=IT`, with **no public IP** (AWS charges for public IPv4 addresses). Sign in as Jordan through the access portal. Confirm he can reboot it, can't terminate it, and can't change the instance type. Launch a second instance *without* the tag and confirm he can't touch it. **Terminate both the same day.**

**Why it matters:**
- **Describe actions don't support resource-level permissions**, so they need `"Resource": "*"`. Start/stop/reboot do, which is why those can be locked down by tag. Knowing which actions support resource-level control is a real IAM skill.
- **The tag condition is a second layer of scoping.** In Azure you scoped the custom role to one resource group. Here, the account is the first boundary and the tag narrows it to specific servers.
- **Using CloudTrail to build least-privilege policies** is how this is done in practice: perform the action, see what was called, grant exactly that. It's also how attackers and pentesters figure out what a stolen credential can do.

### Task 5 – External co-counsel (R6)

AWS has no B2B guest invite. The standard pattern for "someone from another organization needs access" is a **cross-account IAM role**: you create a role in your account, and the other organization's identity is allowed to assume it.

1. You need a second AWS account **outside your organization** to act as co-counsel's firm. Options:
   - Create a new standalone AWS account (most realistic), or
   - Use a member account in a separate OU named `External-Sim` (cheaper on effort, but note in your writeup that it's in-org and therefore not a true external test).
2. In co-counsel's account, create an IAM user or Identity Center user for their attorney, with MFA.
3. In `mblg-litigation-prod`, create a role `CoCounsel-ViewOnly` with the `ViewOnlyAccess` policy and this trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<cocounsel-account-id>:user/<attorney-user>" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": { "aws:MultiFactorAuthPresent": "true" }
      }
    }
  ]
}
```

4. Set the role's **maximum session duration** to 1 hour.
5. Sign in as the attorney and use **Switch role** in the console to reach the litigation account.
6. Turn on **IAM Access Analyzer** (external access analyzer, free) in the litigation account and confirm it flags this role as accessible from outside the organization.

**Why it matters:**
- **Trust the narrowest principal you can.** Trusting a whole account (`arn:aws:iam::<id>:root`) lets *any* identity in that account that's been granted `sts:AssumeRole` reach you. Trusting one specific user is tighter.
- **The MFA condition** means a stolen password alone can't be used to cross into your environment.
- **Like B2B guests, the attorney uses their own organization's identity.** If their firm disables their user, they lose the ability to assume your role.
- **Access Analyzer is the auditor's view:** "show me everything outside my organization that can get in." That's the AWS equivalent of having all external access in one `SG-External-CoCounsel` group.

### Task 6 – Offboarding Kevin (R7)

This is where AWS differs most from Azure. Read the reasons, because the **order matters**.

**Preparation (do this once, before Kevin's last day):**
Create an SCP named `Deny-Revoked-Users` and attach it to the `Workloads` OU:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRevokedIdentityCenterUsers",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": { "identitystore:userId": ["placeholder-not-a-real-id"] }
      }
    }
  ]
}
```

Deploying it with a placeholder ahead of time means that when you need it, revocation is a quick edit instead of building and attaching a policy under pressure.

**On Kevin's last day:**
1. Copy Kevin's **User ID** from his Identity Center user page and put it in the SCP in place of the placeholder. His in-flight role sessions in member accounts now fail immediately.
2. Remove Kevin from all groups and any direct permission set assignments.
3. **Delete his active sessions** (his user page → Active sessions → Delete session). Do this **before** disabling him; once he's disabled, you can no longer stop his sessions from the console.
4. **Disable** his user access.
5. Wait for any role sessions to expire (they can last up to the permission set's session duration, at most 12 hours), then remove his ID from the SCP so the policy stays clean.
6. Plot twist: Diane emails that Kevin's files are needed for a case. **Try to restore him.** You can't. Identity Center has no "Deleted users" recycle bin.

**What to do instead:** Don't delete departing users right away. Disable them and keep them for your retention period. In your writeup, explain that Kevin's *files* live in resources like S3 buckets, which aren't deleted when his user is deleted. Recreating "Kevin" creates a brand-new user with a new user ID, so any policy that referenced the old ID won't match him.

**Why it matters:**
- **Disabling a user doesn't kill credentials already issued.** A user with an active access portal session can keep assuming roles, and role credentials stay valid until they expire. The SCP with the user ID condition is what stops in-flight sessions.
- **SCPs do not apply to the management account.** If Kevin had access to the management account, this SCP wouldn't protect it. That's another reason to keep people out of the management account. (Alternative from AWS's guidance: put the same deny statement inside the permission sets themselves.)
- This is a great interview story: "In Azure I blocked sign-in and revoked sessions. In AWS, I learned the same goal needs an SCP, because disabling the user leaves existing role credentials working."

### Task 7 – The access matrix (R8)

Build the same one-page table for Diane: every group or external identity, its members, which accounts it can reach, and what it can do there.

Verify each row instead of writing it from memory:
- **Identity Center → AWS accounts** shows which groups and permission sets are assigned to each account.
- `aws sso-admin list-account-assignments` produces the same thing from the CLI (good for scripting the matrix).
- The **IAM Policy Simulator** tests whether a specific role can perform a specific action, for example "can the litigation ViewOnly role download an S3 object?"
- **IAM Access Analyzer** lists the cross-account access (co-counsel).

**Why it matters:** same as Azure. The technical work only counts if a non-technical managing partner can read the result.

---

## 5. Validation

Use separate InPrivate/Incognito windows for each test user. Sign in through your **AWS access portal URL** (shown in Identity Center settings), not the root sign-in page.

| Test | Expected result | Pass/Fail |
|---|---|---|
| Tasha (Litigation) signs in to the access portal | Sees only `mblg-litigation-prod` with ViewOnly | |
| Tasha tries to download an S3 object in litigation | Denied (ViewOnly, not ReadOnly) | |
| Paralegal's Department changed (ABAC test) | Access to tagged buckets changes with no group or policy edits | |
| Jordan resets Priya's password | Succeeds | |
| Jordan tries to create or delete a user | Denied | |
| Jordan reboots the tagged server | Succeeds | |
| Jordan tries to terminate a server or reboot the untagged one | Denied | |
| Owen opens Billing and Cost Explorer | Sees costs | |
| Owen tries to create a budget | Denied | |
| Co-counsel switches role with MFA | Reaches litigation account, view-only | |
| Co-counsel tries without MFA | Denied | |
| Kevin's in-flight session after SCP update | Actions fail immediately | |
| Kevin tries to sign in after offboarding | Blocked | |
| Restore Kevin | Not possible; document the lesson | |

> **Expect an MFA prompt** when test users first sign in to the access portal. Check **Identity Center → Settings → Authentication** to see your MFA settings and note them in your writeup. This is the AWS counterpart to Entra's security defaults.

---

## 6. Troubleshooting Tickets

Try to diagnose before opening the hints. Document your actual investigation in the Troubleshooting Log.

**T1 – "I'm a litigation paralegal. I signed in, but I don't see any AWS accounts."** (Tasha)
<details><summary>Hints</summary>

- Is she actually in `SG-Litigation-Team`? Check the group's members.
- Is the permission set assigned to that group *on the litigation account*, and did provisioning finish? (Check the permission set's provisioning status.)
- Is she signing in to the access portal, or trying the regular console sign-in page?
- If she signed in before you made the assignment, has she refreshed or signed in again?
</details>

**T2 – "Jordan says the EC2 console shows red error banners, and one server won't reboot."**
<details><summary>Hints</summary>

- The console calls many more Describe actions than the policy grants just to draw the page. Are the banners blocking his actual job, or just cosmetic? How would you decide whether to grant more?
- For the server that won't reboot: what's its `Department` tag, exactly? Tag values are case-sensitive. `it` is not `IT`.
- Use CloudTrail to see the exact action that was denied.
</details>

**T3 – "Co-counsel says Switch Role gives an error."**
<details><summary>Hints</summary>

- Did they enter the correct account ID and role name?
- Did they sign in **with MFA** in their own account? The trust policy requires it.
- Does the principal in your trust policy match their exact user ARN?
- Does *their* account give their user permission to call `sts:AssumeRole` on your role? Cross-account access needs both sides to allow it.
- Side lesson: you may see guides that use an **External ID** in trust policies. External IDs are designed for third-party *services* acting on your behalf (the "confused deputy" problem), and the console's Switch Role screen can't pass one. For a human, MFA plus a specific principal is the right control.
</details>

**T4 – "Owen gets Access Denied on the Billing page even though he has the billing read-only permission set."**
<details><summary>Hints</summary>

- Was **IAM user and role access to Billing information** activated by the root user (Task 0, step 5)?
- Is his permission set assigned on the **management** account, where billing lives, and not a member account?
</details>

---

## 7. Cleanup & Cost

- [ ] Terminate any test EC2 instances and confirm their EBS volumes are deleted
- [ ] **Keep** the organization, accounts, users, groups, and permission sets. Later projects reuse them
- [ ] Remove the co-counsel role (or its trust relationship) when the "case closes," and confirm Access Analyzer no longer flags it
- [ ] Leave the `Deny-Revoked-Users` SCP in place with the placeholder ID, ready for next time
- [ ] Record the actual cost from Billing: $____

---

## 8. Translation Notes (Azure → AWS)

Fill in the last column as you build. This table is the part that proves you understand identity, not just one portal.

| Requirement | Azure (Project 01) | AWS (this project) | What I learned from the difference |
|---|---|---|---|
| Directory | Entra ID | IAM Identity Center | |
| Resource boundary | Resource group | AWS account | |
| Users | Entra users, CSV bulk create | Identity Center users, scripted create | |
| Dynamic groups | Membership rules | Not available; ABAC with session tags | |
| Admin group | Contributor (no access management) | PowerUserAccess (no IAM management) | |
| Read-only | Reader (control plane only) | ViewOnlyAccess (not ReadOnlyAccess) | |
| Helpdesk | Built-in Helpdesk Administrator, protects admins | Custom permission set, no built-in admin protection | |
| VM operator | Custom role scoped to an RG | Custom policy scoped by account + tag | |
| Cost visibility | Cost Management Reader at subscription | Billing read-only in management account | |
| External user | B2B guest | Cross-account role with MFA condition | |
| Kill sessions | Revoke sessions | Delete portal sessions + SCP deny on user ID | |
| Restore user | 30-day soft delete | No restore; disable instead of delete | |
| Verify access | Check access | Policy Simulator, Access Analyzer | |

---

## 9. My Build Notes *(fill in as you go)*

### What I built
<!-- Screenshots go in ./images/. Redact account IDs, user IDs, emails, and the access portal URL. -->

### Why I made these choices
<!-- In your own words. This is the section interviewers read. -->

### Troubleshooting Log
| Issue | What I checked | Root cause | Fix |
|---|---|---|---|
| | | | |

### Lessons Learned

---

## Stretch Goals

- **Entra ID as the identity source:** connect your Azure tenant to Identity Center with SAML and SCIM so users and groups (including the dynamic ones) sync from Entra. This is how many Microsoft-centric firms actually run AWS. *Read AWS's guidance on changing the identity source first; it affects existing users and assignments, so plan it and screenshot your current setup before switching.*
- **Temporary elevated access:** research AWS's Temporary Elevated Access Management (TEAM) solution as an AWS counterpart to Entra PIM, so Jordan's helpdesk permission set is requested and time-limited instead of standing.
- **Region guardrail SCP:** deny actions outside `us-east-1` (with exceptions for global services). Compare it with the Azure Conditional Access goal of blocking sign-ins from outside the US, and explain why they aren't the same control.
- **Infrastructure as code:** rebuild Tasks 0–3 with Terraform or CloudFormation, so the whole identity setup can be recreated from a repo.

## Interview Talking Points

- Why the AWS account, not a resource group, is the real security boundary
- Identity Center users vs IAM users, and why people should get temporary credentials
- How I replaced Azure dynamic groups with ABAC, and the tradeoff
- Why ViewOnlyAccess instead of ReadOnlyAccess for a law firm
- How I used CloudTrail to build a least-privilege helpdesk policy
- Why disabling a user isn't enough in AWS, and how the SCP stops in-flight sessions
- What I'd do differently in AWS vs Azure for offboarding, given there's no restore
