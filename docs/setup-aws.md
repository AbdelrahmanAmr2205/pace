# AWS account setup — from a bare root account to a locked-down instance

Starting point: one brand-new AWS account, a root email and password, nothing else.
End state: an arm64 EC2 instance running with **zero inbound firewall rules**, an
identity you use day to day that is not root, and alarms that shout before a bill does.

This is the AWS half of step 0. The Tailscale/Docker half is
[`setup-server.md`](setup-server.md). Do this one first.

> **Console layouts drift.** Every menu path here was accurate when written; if a
> button has moved, the concept is still right — search the console for the noun
> (*"Budgets"*, *"Permission sets"*) rather than hunting the exact path.

---

## 0. What "least privilege" actually buys a one-person account

Worth being honest up front, because it changes where you spend effort.

You are the account owner. You need to be able to do everything, eventually. So
hand-writing a minimal IAM policy for *yourself* mostly buys an afternoon of
`AccessDenied`, followed by you widening it until it is admin again. That is not
security, it is ceremony.

The controls that genuinely reduce risk here, in order of how much they pay:

1. **Nobody logs in as root.** Root cannot be restricted by any policy, ever. The only
   defence is not using it and locking it behind MFA.
2. **No long-lived access keys exist.** Almost every "AWS account drained overnight"
   story is a static `AKIA...` key in a repo, a Dockerfile, or a laptop backup. If no
   such key exists, that entire class of incident cannot happen to you.
3. **The instance's own credentials are near-worthless.** This is the one place
   least-privilege-by-allow-list is easy and real. Your app needs *nothing* from the
   AWS API, so the role it runs under should grant nothing beyond a shell fallback.
4. **The blast radius is bounded.** A region lock and a spend alarm turn "someone got
   in" from an unbounded bill into a contained, visible one.
5. **A narrow identity for the boring 95%.** Day to day you look at an instance and
   maybe reboot it. Doing that under a permission set that *can only do that* is real
   least privilege, because it never fights you — the admin set is one dropdown away
   when you actually need it.

Sections 1–5 below are the essentials. Section 6 is worth the twenty minutes.
Section 7 is optional hardening you can skip and add later.

---

## 1. Root account: harden it, then stop using it

Log in as root. You will do all of this once and then, with a couple of exceptions in
§9, never sign in this way again.

**a. MFA — at least two devices.** IAM → Security credentials → Multi-factor
authentication. AWS now requires MFA on the root user of a standalone account, so you
may be prompted anyway.

Root supports up to eight MFA devices. **Register two.** A TOTP app on your phone, and
either a second app on the laptop or a hardware key. Losing your only MFA device means
an identity-verification process with AWS support involving your phone and billing
details; a second device turns a bad week into a non-event.

**b. Confirm there are no root access keys.** Same page. If any exist, delete them. A
root access key is a permanent, unrestrictable credential — there is no legitimate
reason to hold one.

**c. Password into a password manager.** Long, random, never typed again.

**d. Alternate contacts.** Account → Alternate contacts. Fill in Billing, Operations
and Security. These are where AWS writes when something is wrong with your account; if
they bounce to an address you do not read, you find out from the bill.

**e. Let non-root identities see billing.** Account → IAM user and role access to
billing information → **Activate**. Without this, your day-to-day identity gets a
blank Billing page no matter what its IAM policy says. Newer accounts often ship with
this on — check rather than assume.

**f. Confirm which free tier you are on.** Billing and Cost Management → Free tier.
AWS changed the model in mid-2025: older accounts get 12 months of specific free
allowances; newer ones get a credit balance that everything draws down, with the free
plan ending when credits run out or after a fixed window. You said "at least three
months" — this page is what actually decides that. Read your own number here and put
the end date in your calendar today, with a reminder two weeks earlier.

---

## 2. Money guardrails — before you launch anything

Do this before the instance exists, so it is watching from minute one.

Billing and Cost Management → **Budgets** → Create budget. The first couple of budgets
are free.

- **Budget 1 — zero spend.** Use the "Zero spend budget" template. It alerts the moment
  actual charges exceed $0.01. Under a credits-based free plan this fires when you
  fall off the free plan, which is exactly the signal you want.
- **Budget 2 — a real ceiling.** A monthly cost budget of, say, $10, with alerts at
  80% **actual** and 100% **forecast**. The forecast alert is the useful one: it warns
  you mid-month, while there is still time to turn something off.

Also enable Billing → Preferences → **free tier usage alerts** and **PDF invoices by
email**.

**Why both:** the zero-spend budget catches "I am no longer free". The ceiling catches
"something is very wrong" — a runaway resource, or someone else using your account.
Neither costs anything, and a surprise AWS bill is the single most common way a hobby
project ends.

---

## 3. Pick a region, and only use that one

Everything for this project lives in one region. Pick it now; moving later means
rebuilding the instance.

From Egypt, the sensible candidates are Frankfurt (`eu-central-1`), Milan
(`eu-south-1`), Bahrain (`me-south-1`) and UAE (`me-central-1`). Round-trip latency
lands somewhere in the 40–90 ms band from any of them — for server-rendered HTML with
no client-side chatter, that difference is invisible.

**Default to `eu-central-1` (Frankfurt).** It is one of the cheapest and best-stocked
regions, gets new instance families early, and is very unlikely to tell you `t4g` is
unavailable. The Middle East regions are geographically closer but typically price
higher per hour and carry a thinner instance catalogue — you would be paying a
premium for latency you cannot perceive.

Tailscale does not change this calculation: traffic is peer-to-peer, so you get the
raw network path either way.

---

## 4. Account-wide toggles worth flipping once

All free, all one-time. In the **EC2 console**, with your chosen region selected:

- **EC2 → Account attributes → EBS encryption → Enable encryption by default.** Every
  volume you create from now on is encrypted with the AWS-managed key. No cost, no
  performance difference, and it covers the disk your SQLite database will sit on.
- **EC2 → Account attributes → IMDS defaults → IMDSv2 required, hop limit 1.** See
  §6c for why the hop limit is the interesting half.

In the **S3 console**: Block Public Access (account settings) → all four on. You are
not using S3, but a future you might, and this is the toggle people forget.

**CloudTrail:** leave it alone. Event history is on by default, free, and keeps 90 days
of management events — enough to answer "what happened to my account" for a project
this size. Creating a *trail* means an S3 bucket and a running cost, for retention you
do not need.

**GuardDuty:** skip it. It is genuinely good and genuinely metered — after the trial it
bills continuously on log volume. For one instance with no inbound ports, it would be
watching a door that does not exist.

---

## 5. A daily-driver identity that is not root

Two workable paths. Take the first.

### Recommended: IAM Identity Center

Identity Center (the thing formerly called AWS SSO) is free, and it gives you a login
portal that issues **short-lived** credentials. No `AKIA...` key ever exists on your
laptop, which deletes risk #2 from §0 outright. It is also what current AWS practice
looks like, so it is the version worth learning.

One thing to know before you click: enabling Identity Center also enables **AWS
Organizations**, turning your single account into a one-account organisation. That is
normal, free, reversible, and it unlocks the optional guardrails in §7.

1. IAM Identity Center → Enable. Note the region it lands in — make it your chosen
   region from §3.
2. **Users** → Add user. Yourself, your real email.
3. Register MFA for that user too. Short-lived credentials still start with a login.
4. **Permission sets** → create two (policies in §6a and §6b):
   - `PaceAdmin` — full admin, region-scoped. For setup and changes.
   - `PaceOperator` — look at the instance, start/stop it, open a shell. For every
     other day.
5. **AWS accounts** → select your account → Assign users → assign yourself **both**
   permission sets.
6. Bookmark the portal URL (`https://d-xxxxxxxxxx.awsapps.com/start`). You can rename
   that subdomain once, in Identity Center settings.

Day to day you pick `PaceOperator` in the portal. The point is not that it stops an
attacker who already has your session — it is that routine work happens in a context
where a fat-fingered click *cannot* delete the instance or open a firewall.

If you ever want CLI access: `aws configure sso`, then `aws sso login`. Credentials
expire on their own, which is the entire benefit.

### Fallback: a plain IAM user

If Identity Center feels like too much machinery right now: IAM → Users → create one
user, attach `AdministratorAccess`, enable MFA, **create no access keys**, and use the
console only. This is still a large improvement over root.

It is strictly worse in one way that matters: the moment you want CLI access you will
be tempted to mint a long-lived access key, and that key will outlive your memory of
creating it. If you take this path, promise yourself console-only.

---

## 6. The policies

### a. `PaceAdmin` — admin, fenced into one region

Full administrative power, with everything outside your chosen region denied. An
explicit `Deny` beats any `Allow`, so this holds even under `AdministratorAccess`.

Attach the AWS managed policy `AdministratorAccess`, **plus** this as an inline policy
on the permission set:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyEverythingOutsideMyRegion",
      "Effect": "Deny",
      "NotAction": [
        "iam:*", "sts:*", "organizations:*", "account:*", "sso:*",
        "sso-directory:*", "identitystore:*", "budgets:*", "ce:*", "cur:*",
        "billing:*", "payments:*", "tax:*", "freetier:*", "consolidatedbilling:*",
        "invoicing:*", "support:*", "health:*", "trustedadvisor:*",
        "notifications:*", "route53:*", "cloudfront:*", "waf:*", "shield:*",
        "s3:ListAllMyBuckets"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": "eu-central-1"
        }
      }
    }
  ]
}
```

Replace `eu-central-1` with your §3 choice.

**Why `NotAction` and not a tidy allow-list:** the services excluded above are *global*
— IAM, billing, Organizations, Route 53 and friends have no meaningful region, and AWS
reports their calls against `us-east-1`. Deny those and you lock yourself out of your
own billing console. The shape "deny everything except the global services, unless it
is in my region" is the standard idiom for this, and it is worth recognising when you
see it elsewhere.

**What it buys you:** a stolen session, or your own misclick, cannot quietly spin up
resources in Oregon that you never look at and that bill for months. Region sprawl is
the most common way small accounts leak money.

Expect the console to show errors if you browse to another region. That is the policy
working.

### b. `PaceOperator` — the one you actually live in

No managed policy. Just this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOnlyLooking",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "ce:GetCostAndUsage",
        "ce:GetCostForecast",
        "budgets:ViewBudget",
        "freetier:Get*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PowerCycleOnlyThePaceInstance",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": { "ec2:ResourceTag/Project": "pace" }
      }
    },
    {
      "Sid": "ShellIntoThePaceInstance",
      "Effect": "Allow",
      "Action": "ssm:StartSession",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": { "ssm:resourceTag/Project": "pace" }
      }
    },
    {
      "Sid": "SessionManagerPlumbing",
      "Effect": "Allow",
      "Action": [
        "ssm:DescribeSessions",
        "ssm:GetConnectionStatus",
        "ssm:DescribeInstanceInformation",
        "ssm:DescribeInstanceProperties",
        "ssm:TerminateSession",
        "ssm:ResumeSession"
      ],
      "Resource": "*"
    }
  ]
}
```

Note the tag conditions. This permission set can reboot **an instance tagged
`Project=pace`** and nothing else — not a future instance you forget to tag, not
anything an attacker creates. Tag-based conditions are the cheapest scoping mechanism
AWS offers, which is why §6d insists on tagging at launch.

This is the part of the setup where least privilege is real rather than ceremonial: the
permissions are narrow *and* they cover essentially everything you do after launch day.

### c. The instance's own role — deliberately almost empty

IAM → Roles → Create role → AWS service → EC2. Attach **only** the managed policy
`AmazonSSMManagedInstanceCore`. Name it `pace-instance`.

That is the whole thing. No S3, no Secrets Manager, no CloudWatch agent.

**Why this matters more than any of the human policies:** your app is a static binary
and a SQLite file. It has no reason to call AWS at all. Any process on that box can
read the instance's credentials out of the instance metadata service — so the question
"what can code running on this machine do to my AWS account?" is answered entirely by
this role. The answer here is: register with Systems Manager. Nothing else. There is
no bucket to exfiltrate, no secret to read, no permission to escalate.

`AmazonSSMManagedInstanceCore` earns its place by giving you a browser shell that needs
**no open port**, which is what makes §7 of the Tailscale runbook safe to do on day one
rather than day two.

**The IMDS hop limit from §4 is the other half of this.** With a hop limit of 1, the
metadata service is unreachable from inside a Docker container — the bridge network
adds a hop. You are about to run containers on this box. That single setting means a
compromised container cannot reach the instance credentials at all.

### d. Cost allocation tag

Billing → Cost allocation tags → activate `Project`. It takes up to 24 hours to
appear, after which Cost Explorer can group spend by project. Do it now so it is ready
when you need it.

---

## 7. Optional: guardrails you cannot disable by accident

Skip this section if you want to get to the instance. It is easy to add later.

Enabling Identity Center gave you an Organization, which means **Service Control
Policies** — restrictions that apply to *everything* in the account including the admin
permission set. You must enable the SCP policy type in Organizations → Policies first.

The one worth having wraps the §6a region lock at the organisation level, so it applies
even if you later attach `AdministratorAccess` somewhere and forget the inline deny.
Same JSON, attached as an SCP to the root.

A second, more interesting one: deny tampering with your own safety rails.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProtectTheGuardrails",
      "Effect": "Deny",
      "Action": [
        "budgets:DeleteBudget",
        "budgets:ModifyBudget",
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "account:PutAlternateContact",
        "account:DeleteAlternateContact"
      ],
      "Resource": "*"
    }
  ]
}
```

**Read the trade-off before applying this.** SCPs do not apply to the root user, so
root remains your escape hatch — which is exactly the design: changing a budget becomes
a deliberate act requiring the credential you keep in a password manager behind two MFA
devices. If that sounds like friction you will resent in three months, leave it off.
The region lock is the one that pays for itself; this one is a matter of taste.

---

## 8. Launching the instance

Sign in through the Identity Center portal as `PaceAdmin`. Confirm the region.

**Name and tags.** Name `pace`. Then add a second tag: **`Project` = `pace`**. Not
optional — §6b's entire scoping depends on it.

**AMI.** Amazon Linux 2023, **arm64**. AL2023 ships the SSM agent preinstalled and
running, which is what makes the no-open-ports plan work out of the box.

**Instance type.** `t4g.small` or `t4g.micro` — arm64 Graviton. Match Oracle's Always
Free tier, which is ARM, so the later migration is a copy rather than a rebuild. Trust
the **"Free tier eligible"** label in the type picker over any document, including this
one; the specific eligible types have changed more than once. If only x86 is free, take
it and note it in the migration plan — it is one word in the `docker build` command.

**Key pair.** Create one, ed25519, named `pace`. Download it, put it somewhere you will
still have it in a year. You will very probably never use it — it is the break-glass
behind the break-glass, for the day both Session Manager and Tailscale are unhappy at
once. Launching with no key pair at all removes an escape route for no benefit.

**Network settings** → Edit:

- VPC: default. Subnet: any. **Auto-assign public IP: Enable.**
- Firewall: **Create security group**, name `pace-server`.
- **Inbound rules: delete every one. Zero rules.** Not SSH-from-my-IP. Zero.
- Outbound: allow all (the default).

> **Why zero inbound is not a lockout risk, and why it is better than "SSH from my
> IP, temporarily":**
>
> Security group rules are editable at any time, from the console, in about fifteen
> seconds. Starting closed costs you nothing, because you can always open a door later.
> Starting open costs you the temporary rule you forget to remove — and "temporary"
> firewall rules are famously permanent.
>
> Session Manager does not need an inbound rule. The agent dials *out* over 443 and the
> shell rides back down that connection, exactly like Tailscale. So the break-glass
> access you need during setup already works with the door shut.
>
> The runbook's original "SSH from your IP only, temporarily" was inherited thinking. It
> is not wrong, but it is strictly worse than this, and it creates a cleanup step that
> is easy to skip.

**Storage.** 20–30 GiB `gp3`, encrypted (already default from §4). Your database lives
here. Snapshots are not a backup strategy on their own — see the note in §10.

**Advanced details:**

- **IAM instance profile: `pace-instance`** (from §6c). Easy to miss; without it the
  instance never appears in Session Manager.
- **Termination protection: Enable.** Free, and it turns an irreversible misclick into
  an error message.
- **Shutdown behaviour: Stop** (the default).
- **Metadata version: V2 only (token required). Metadata response hop limit: 1.**
- **Detailed CloudWatch monitoring: off.** It costs money and tells you nothing you
  need at this size.

Launch.

**Note that you do not need an Elastic IP.** Tailscale gives the box a stable identity
independent of its public address, so the dynamic IP changing across a stop/start is
something you will genuinely never notice. Skip it — and know that an Elastic IP that
is *allocated but not attached* bills by the hour, which is a classic way to pay AWS
for nothing.

Be aware that public IPv4 addresses are billed hourly (roughly $3–4/month) once free
allowances stop applying. It is the main standing cost of this instance after the free
window. Going IPv6-only avoids it but needs NAT64 to reach IPv4-only endpoints, which
means a NAT gateway, which costs several times more than the address. Pay the $3.

---

## 9. First connection, with no ports open

EC2 → Instances → select `pace` → **Connect** → **Session Manager** → Connect.

Give it two or three minutes after launch for the agent to register. You should land in
a browser shell as `ssm-user`. `sudo -i` works.

From here, go to [`setup-server.md`](setup-server.md) §4 and install Docker and
Tailscale. You never open a port.

**If Session Manager says the instance is not available:**

1. Confirm the instance is `running` and has finished its status checks.
2. Confirm the `pace-instance` role is actually attached — Actions → Security → Modify
   IAM role. Attaching it after launch works; the agent picks it up within a few
   minutes, or after a reboot.
3. Confirm outbound 443 is allowed (it is, unless you edited the outbound rule).
4. Still stuck: add a temporary inbound SSH rule from your own IP, connect with the
   `pace` key pair, and debug with `sudo systemctl status amazon-ssm-agent`. Then
   remove the rule. This is what the key pair is for.

### What still requires root

Sign in as root only for these, then sign back out:

- Closing the account
- Changing the account name, root email or root password
- Changing tax/payment settings
- Restoring access if you lock every other identity out
- Undoing anything in §7, since SCPs do not apply to root

Everything else is `PaceAdmin`.

---

## 10. Leaving (you are on a clock)

You are moving to Oracle Cloud in roughly three months. Two things to do now so that is
a copy rather than a rescue:

**Keep backups provider-neutral.** The tempting AWS answer is EBS snapshots via Data
Lifecycle Manager. Do not build on it: it costs storage, and it is exactly the kind of
provider-specific dependency `decisions.md` rules out. The portable answer is a nightly
`sqlite3 .backup` into a file plus a pull to your laptop over Tailscale — which works
identically on Oracle, and which you can actually restore from without AWS. Details
belong in roadmap step 5.

**Rehearse the migration while AWS is still free.** Standing up the Oracle box before
the AWS one expires means the migration is a rehearsal with a working fallback, instead
of a deadline with none. Two caveats to plan around, both documented in
`decisions.md` §6: Ampere A1 capacity is frequently unavailable in popular regions, so
choose the region by what you can actually launch; and Always Free tenancies have
historically reclaimed idle resources — a personal to-do app is idle by those metrics.

**When you do leave:** terminate the instance (disable termination protection first),
delete its volume and any snapshots, release any Elastic IPs, and check Cost Explorer a
few days later for anything still ticking. Closing the account entirely is root-only.

---

## 11. Checklist

Root and money:

- [ ] Root MFA enabled, **two devices** registered
- [ ] No root access keys exist
- [ ] Alternate contacts filled in (billing, operations, security)
- [ ] IAM access to billing information activated
- [ ] Free tier model identified and **end date in the calendar** with a reminder
- [ ] Zero-spend budget created
- [ ] Monthly ceiling budget with actual **and forecast** alerts
- [ ] Free tier usage alerts on

Account settings:

- [ ] Region chosen and written down
- [ ] EBS encryption by default on
- [ ] IMDSv2 required, hop limit 1
- [ ] S3 Block Public Access on (all four)
- [ ] `Project` activated as a cost allocation tag

Identity:

- [ ] Identity Center enabled, user created, MFA registered
- [ ] `PaceAdmin` permission set — admin + region-deny inline policy
- [ ] `PaceOperator` permission set — tag-scoped, no managed policy
- [ ] Both assigned; portal URL bookmarked
- [ ] No long-lived access keys anywhere
- [ ] Root credentials in a password manager, not in use

Instance:

- [ ] `pace-instance` role: `AmazonSSMManagedInstanceCore` **only**
- [ ] AL2023, arm64, free-tier-eligible type
- [ ] Tagged `Name=pace` **and `Project=pace`**
- [ ] Security group `pace-server` with **zero inbound rules**
- [ ] Instance profile attached at launch
- [ ] Termination protection on
- [ ] IMDSv2 required, hop limit 1
- [ ] No Elastic IP allocated
- [ ] Key pair downloaded and stored, expected to go unused
- [ ] Session Manager shell reached with no port open

Then continue in [`setup-server.md`](setup-server.md) §4.
