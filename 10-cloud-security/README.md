# Chapter 10: Cloud Security

**Cybersecurity Guide Book | Roadmap step 10 of 12** | Author: [ashihh07](https://github.com/ashihh07)

[Back to roadmap](../README.md) | [Previous: Chapter 9 Network Analysis](../09-network-analysis/README.md) | [Download the PDF version](Chapter-10-Cloud-Security.pdf)

> On-premises, a locked door protects a server. In the cloud, a single checkbox does. This chapter is about finding that checkbox before an attacker does.

---

## About this chapter

Everything so far assumed a network you could draw on paper: your router, your switch, your server rack. Most organisations now run a large share of that on **AWS, Azure or Google Cloud** instead, where the hardware belongs to someone else and almost every control is a setting, not a cable. The attacks do not change much, but where they live does: from exposed ports to exposed **permissions**, and from a stolen password to a stolen **access key**.

The roadmap lists four items:

- **AWS and Azure basics:** the services you will meet in almost every cloud environment.
- **IAM:** identity and access management, who can do what.
- **Security groups:** cloud firewalls, and how they differ from the ones in Chapter 4.
- **Cloud storage security and misconfigurations:** why "the cloud leaked data" is almost always a settings problem, not a hacking story.

### How each topic is built

- **Concept**: what it is and why it exists.
- **Attacker view**: how it gets abused, based on real, widely reported patterns.
- **Defender view**: how to configure it safely and what to monitor.
- **Labs**: run on free-tier accounts, with hard limits to avoid surprise bills.

### Contents

| # | Section | Roadmap item covered |
|---|---------|----------------------|
| 1 | [Cloud models and the shared responsibility](#1-cloud-models-and-the-shared-responsibility) | Orientation |
| 2 | [AWS basics](#2-aws-basics) | AWS Basics |
| 3 | [Azure basics](#3-azure-basics) | Azure Basics |
| 4 | [IAM: identity and access management](#4-iam-identity-and-access-management) | IAM |
| 5 | [Security groups and network controls](#5-security-groups-and-network-controls) | Security Groups |
| 6 | [Cloud storage security](#6-cloud-storage-security) | Cloud Storage Security |
| 7 | [Common misconfigurations](#7-common-misconfigurations) | Misconfigurations |
| 8 | [Hands-on labs](#8-hands-on-labs) | Practice for all of the above |
| 9 | [Quick revision sheet](#9-quick-revision-sheet) | Tables and commands |
| 10 | [Self-check quiz and answers](#10-self-check-quiz) | Test yourself |

> [!WARNING]
> **Ground rule for every lab in this book**
> - Use only your own personal free-tier AWS and Azure accounts for these labs. Never touch an employer's, school's or client's cloud account without explicit written permission, even "just to look around."
> - Cloud resources can **cost real money** if left running. Set a billing alert before you start (Lab 1), and delete every resource at the end of each lab. A free-tier account can still generate a bill from a forgotten instance or a large data transfer.
> - Never commit an access key, secret key or `.pem` file to a Git repository, public or private. Treat every credential in this chapter like a password you cannot see on screen.

---

## 1. Cloud models and the shared responsibility

### Concept

"The cloud" is just someone else's data centre, rented in pieces. Three models describe how much of the stack you manage versus the provider:

| Model | You manage | Provider manages | Example |
|-------|-----------|-------------------|---------|
| **IaaS** (Infrastructure as a Service) | OS, patches, applications, data, network rules | Physical hardware, hypervisor, data centre | AWS EC2, Azure Virtual Machines |
| **PaaS** (Platform as a Service) | Application code, data | OS, runtime, patching, scaling | AWS Elastic Beanstalk, Azure App Service |
| **SaaS** (Software as a Service) | Your data and user access | Everything else | Microsoft 365, Google Workspace |

### The shared responsibility model

Every provider states a version of this, and it is the single most important idea in this chapter:

```text
Security  OF  the cloud   ->  the provider's job   (data centres, hardware, hypervisor, physical access)
Security  IN  the cloud   ->  YOUR job              (identity, data, configuration, network rules, patching your own OS)
```

The provider will not misconfigure your S3 bucket, your IAM policy, or your database's network rules. **Almost every well-known "cloud breach" is a failure on the customer's side of this line**, not a flaw in AWS or Azure itself. Sections 6 and 7 come back to this repeatedly.

| Who is responsible for... | IaaS | PaaS | SaaS |
|----------------------------|------|------|------|
| Physical security | Provider | Provider | Provider |
| Network controls (perimeter) | Provider | Provider | Provider |
| Operating system patching | **You** | Provider | Provider |
| Application configuration | **You** | **You** | Provider (mostly) |
| Identity and access management | **You** | **You** | **You** |
| Data classification and encryption choices | **You** | **You** | **You** |

Notice that **identity and access management, and data, are always your responsibility**, in every model. That is why Sections 4 and 6 matter more than almost anything else in this chapter.

> [!TIP]
> **Defender view**
> - When something goes wrong in the cloud, ask "which side of the shared responsibility line was this?" before anything else. It almost always points straight to the fix.

---

## 2. AWS basics

### Concept

**Amazon Web Services (AWS)** is organised around **Regions** (geographic areas, such as `ap-south-1` for Mumbai) and **Availability Zones** (isolated data centres within a Region). Resources you create live in a specific Region unless the service is explicitly "global" (like IAM).

### Services you will meet constantly

| Service | What it is | Chapter parallel |
|---------|-----------|---------------------|
| **IAM** | Identity and Access Management: users, roles, permissions | Chapter 5's authentication/authorization, at account scale |
| **EC2** | Virtual machines ("instances") | A Linux box like your Chapter 2 VM, rented by the hour |
| **S3** | Object storage ("buckets" full of files) | A file server, but with its own permission model (section 6) |
| **VPC** | Virtual Private Cloud: your own isolated network inside AWS | Chapter 1's subnets and routing, software-defined |
| **Security Groups** | Instance-level firewall rules | Chapter 1's firewall concept, attached to a resource, not a network port |
| **RDS** | Managed relational databases (MySQL, PostgreSQL, and others) | The database behind your Chapter 5 web app, patched by AWS |
| **Lambda** | Run code without managing a server ("serverless") | PaaS taken further: you supply only the function |
| **CloudTrail** | Logs **every API call** made in the account: who did what, when, from where | The cloud's equivalent of Chapter 2's `auth.log` and Chapter 4's access log, combined |
| **CloudWatch** | Metrics, logs and alarms | Monitoring and basic SIEM-like alerting |
| **GuardDuty** | Managed threat detection service that watches CloudTrail, DNS and VPC traffic | A managed IDS (Chapter 9), built in |

### The AWS CLI

```bash
pip install awscli --break-system-packages   # or: sudo apt install awscli
aws configure                                # enter your access key, secret key, default region
aws sts get-caller-identity                  # "who am I?" -- always run this first in any account
aws s3 ls                                    # list your S3 buckets
aws ec2 describe-instances --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name}"
```

`aws sts get-caller-identity` is the cloud equivalent of Chapter 2's `whoami` and `id`. Run it immediately whenever you touch a new account or role, to confirm exactly who and what you are before doing anything else.

> [!CAUTION]
> **Attacker view**
> - A leaked AWS access key (committed to GitHub, left in a public Docker image, or pulled from a compromised server's environment variables) is often **immediately** more useful than any single exploit, because it may grant direct API access to the whole account.
> - Automated bots scan public code repositories for leaked keys within minutes of a push, and can spin up cryptomining instances on a stolen key before the owner notices.

> [!TIP]
> **Defender view**
> - Enable **CloudTrail** in every account, in every Region, from day one. Without it, you cannot investigate anything that happened before you turned it on.
> - Never put access keys in code or commit history. Use IAM roles for EC2 and Lambda instead (section 4), and `git-secrets` or similar tooling to block accidental commits.
> - Set a billing alarm immediately in any new account (Lab 1). An unexpected bill is often the first sign of compromise.

---

## 3. Azure basics

### Concept

**Microsoft Azure** uses similar ideas with different names. Resources live inside **Resource Groups**, inside a **Subscription**, inside a **Tenant** (your organisation's identity boundary, backed by **Microsoft Entra ID**, formerly Azure Active Directory).

### The AWS-to-Azure phrasebook

| AWS | Azure | Role |
|-----|-------|------|
| IAM | Microsoft Entra ID (Azure AD) + Azure RBAC | Identity and permissions |
| EC2 | Virtual Machines | Virtual servers |
| S3 | Blob Storage | Object storage |
| VPC | Virtual Network (VNet) | Isolated network |
| Security Groups | Network Security Groups (NSGs) | Instance/subnet-level firewall |
| RDS | Azure SQL Database / Azure Database for MySQL | Managed databases |
| Lambda | Azure Functions | Serverless compute |
| CloudTrail | Azure Activity Log + Microsoft Entra sign-in logs | Who did what, and who signed in |
| CloudWatch | Azure Monitor | Metrics, logs, alerts |
| GuardDuty | Microsoft Defender for Cloud | Managed threat detection |
| Account | Subscription | Billing and resource boundary |

### The Azure CLI

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash     # Debian/Ubuntu install
az login
az account show                                             # "who am I, which subscription?"
az storage account list --output table
az vm list --output table
```

`az account show` is Azure's version of `aws sts get-caller-identity`: run it first in any session.

> [!TIP]
> **Defender view**
> - Resource Groups are a genuine security boundary when used well: grouping a project's resources together makes it far easier to apply one consistent set of permissions and to delete everything cleanly when a project ends (Lab endings matter, see the warning at the top of this chapter).
> - Microsoft Entra sign-in logs show failed and successful logins, including **impossible travel** alerts (the same login appearing from two distant locations within an implausible time window), directly building on Chapter 6's "session used from two countries" signal.

---

## 4. IAM: identity and access management

### Concept

**IAM** controls who (or what) can do what, to which resource. This is the single most consequential topic in cloud security: a perfect network, a patched OS and an encrypted database are all undone by one over-broad permission.

### The building blocks (AWS terms; Azure RBAC uses the same ideas)

| Term | Meaning |
|------|---------|
| **Identity** | A **user** (a person), a **role** (a temporary identity assumed by a service or another account), or a **group** (a set of users that share permissions) |
| **Policy** | A JSON document stating what actions are allowed or denied on which resources |
| **Principle of least privilege** | Grant only the specific permissions needed, nothing more (the same rule from Chapter 2's sudo and file permissions, at account scale) |
| **Role vs user** | A user has long-lived credentials. A role is **assumed temporarily** and issues short-lived credentials, which is why EC2 instances and Lambda functions should use roles, never a user's access keys |
| **MFA** | Multi-factor authentication (Chapter 5). Mandatory on the account's root/owner identity, no exceptions |

### Reading an IAM policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::lab-bucket", "arn:aws:s3:::lab-bucket/*"]
    }
  ]
}
```

| Field | Meaning |
|-------|---------|
| `Effect` | `Allow` or `Deny`. An explicit `Deny` always wins over any `Allow`, anywhere |
| `Action` | Which API calls this applies to (`s3:GetObject` = read a file) |
| `Resource` | Which specific resource(s), by ARN (Amazon Resource Name), this applies to |

The **dangerous version** of this same policy, seen constantly in real misconfiguration reports:

```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

`"Action": "*"` on `"Resource": "*"` grants **every permission on every resource**: full administrative control. A policy this broad attached to an application's role means any flaw in that application (Chapter 5's SQL injection or remote code execution, for example) can potentially take over the entire cloud account, not just the one server.

### The root user and the account owner

Every AWS account has a **root user** created at sign-up, with unrestricted power over everything, including billing and the ability to close the account. The rule is simple and absolute:

- **Never use the root user for daily work.** Create an individual IAM user (or, better, use IAM Identity Center / SSO) for every real task.
- **Enable MFA on the root user immediately**, and store its credentials somewhere secure and rarely touched.
- Azure's equivalent concept is the subscription's **Owner** role; the same "do not use this for daily work" rule applies.

> [!CAUTION]
> **Attacker view**
> - Overly broad IAM policies (`"Action":"*"`, `"Resource":"*"`) are one of the most common findings in real cloud security assessments, because it is genuinely easier to write than a precise policy, and nothing stops it from working.
> - If a single compromised service's role can assume broader permissions than it needs, an attacker who gets code execution on that one service (Chapters 5 and 8) can pivot straight into the rest of the account: this is "privilege escalation" at cloud scale.
> - Stolen root credentials, or a root account with no MFA, is the single worst-case cloud compromise: the attacker owns the whole account, including billing and the ability to disable logging.

> [!TIP]
> **Defender view**
> - Write the **narrowest policy that works**, naming specific actions and specific resource ARNs, not wildcards. Tools like **AWS IAM Access Analyzer** and the **Azure Advisor** security recommendations flag overly broad policies automatically.
> - Give services **roles**, never long-lived user access keys. Rotate any access key that must exist, and delete ones that are unused.
> - Enforce MFA on every human identity, and absolutely on root/Owner accounts.
> - Review IAM policies the same disciplined way you review Linux file permissions in Chapter 2: a wildcard here deserves the same scrutiny as `chmod 777` there.

---

## 5. Security groups and network controls

### Concept

A **Security Group** (AWS) or **Network Security Group** (Azure) is a virtual firewall attached to a resource (an EC2 instance, a database), built on exactly the same ideas as Chapter 1's ports and Chapter 4's server hardening, just applied at the cloud layer instead of the OS layer.

### Key behaviour differences from a traditional firewall

| Property | Security Groups (AWS) | Traditional firewall (Chapter 4) |
|----------|------------------------|-----------------------------------|
| **Default stance** | Deny all inbound, allow all outbound, by default | Varies by vendor |
| **Statefulness** | Stateful: a reply to an allowed outbound or inbound request is automatically permitted | Also usually stateful |
| **Rule type** | Allow rules only (no explicit "deny" rule in a Security Group) | Can have explicit allow and deny rules |
| **Attached to** | The resource (instance/ENI) directly | Usually a network boundary |
| **Combines with** | **Network ACLs**, which operate at the subnet level and *can* have explicit deny rules | n/a |

### A sample rule set, and what it means

```text
Inbound rules for "web-server-sg":
  Type    Protocol   Port    Source
  SSH     TCP        22      203.0.113.7/32        <- only one specific admin IP, not 0.0.0.0/0
  HTTP    TCP        80      0.0.0.0/0              <- anyone on the internet
  HTTPS   TCP        443     0.0.0.0/0              <- anyone on the internet
```

`0.0.0.0/0` (Chapter 1's CIDR notation) means "any IPv4 address, the whole internet." It is correct and necessary for ports 80/443 on a public web server. It is a serious finding on port 22 (SSH), port 3389 (RDP), or any database port, exactly as Chapter 4 and Chapter 6 warned about exposing management and database ports directly to the internet.

```bash
# AWS CLI: list the inbound rules for a security group
aws ec2 describe-security-groups --group-ids sg-0123456789abcdef0 \
  --query "SecurityGroups[].IpPermissions"

# AWS CLI: a quick audit for anything open to the whole internet on a sensitive port
aws ec2 describe-security-groups --query \
  "SecurityGroups[].IpPermissions[?FromPort==\`22\` || FromPort==\`3389\`][?contains(IpRanges[].CidrIp, '0.0.0.0/0')]"
```

> [!CAUTION]
> **Attacker view**
> - Internet-wide scanning services (Chapter 6's Shodan/Censys mention) continuously index cloud IP ranges specifically looking for Security Groups that expose SSH, RDP or database ports to `0.0.0.0/0`. This is one of the fastest ways a freshly launched, misconfigured instance gets found and attacked.
> - A single overly broad Security Group rule undoes every bit of careful IAM and OS hardening elsewhere in the same deployment.

> [!TIP]
> **Defender view**
> - Restrict management ports (22, 3389) to a known admin IP range, a VPN, or a provider's own secure-access service (such as AWS Systems Manager Session Manager, which needs no open inbound port at all).
> - Treat `0.0.0.0/0` on anything other than 80/443 on a genuinely public web server as a finding requiring justification, the same way Chapter 7 would treat an open database port in a vulnerability scan.
> - Use tagging and automated tools (AWS Config rules, Azure Policy) to continuously check for and alert on newly created overly-permissive rules, since these are often created temporarily "just for testing" and then forgotten.

---

## 6. Cloud storage security

### Concept

Cloud object storage (**S3** on AWS, **Blob Storage** on Azure) is where "the cloud leaked millions of records" headlines come from, almost without exception because of **access settings**, not a flaw in the storage service itself.

### How S3 bucket access actually works

| Mechanism | What it controls |
|-----------|-------------------|
| **Bucket policy** | A resource-based IAM-style policy attached directly to the bucket |
| **IAM policy** | A policy attached to a user/role that grants them access to the bucket |
| **Access Control Lists (ACLs)** | An older, simpler, per-object/bucket permission mechanism (increasingly disabled by default) |
| **S3 Block Public Access** | An override that can forcibly block public access regardless of what any policy or ACL says |

### The default has changed, and old habits still catch people out

Since **April 2023**, Amazon automatically enables **S3 Block Public Access** and disables ACLs for every newly created bucket, in every Region, by default. New buckets are private unless someone deliberately changes that. This is genuinely good news, but two things still make this a top finding in real assessments:

1. **Buckets created before April 2023** did not get this default retroactively, and many organisations still run old buckets that were configured under the old, more permissive defaults.
2. **Someone can still deliberately (or carelessly) turn Block Public Access off**, or write a bucket policy that grants public access anyway, on any bucket, old or new.

```bash
# Check whether Block Public Access is actually enforced on a bucket you own
aws s3api get-public-access-block --bucket your-bucket-name

# Lock a bucket down completely (all four settings on)
aws s3api put-public-access-block --bucket your-bucket-name \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Check the current bucket policy
aws s3api get-bucket-policy --bucket your-bucket-name
```

### Encryption

| State | AWS option | Why it matters |
|-------|-----------|-----------------|
| **At rest** | SSE-S3 (Amazon-managed keys), SSE-KMS (your own managed keys, with audit logging of key use) | Protects data if the underlying storage media is ever compromised |
| **In transit** | HTTPS/TLS (Chapter 1) enforced via bucket policy (`aws:SecureTransport`) | Prevents interception of data moving to or from the bucket |

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::your-bucket-name/*",
  "Condition": { "Bool": { "aws:SecureTransport": "false" } }
}
```

This policy statement **denies** any request that does not use HTTPS, a direct application of Chapter 1's "TLS everywhere" rule to a bucket policy.

> [!CAUTION]
> **Attacker view**
> - Automated tools and researchers continuously scan for publicly readable buckets using predictable naming (company name, project name, `-backup`, `-prod`) and simple reachability checks. No login or exploit is needed once Block Public Access is off and a permissive policy exists.
> - A bucket left open even briefly, during testing, can be indexed and copied by automated scanners before anyone notices it was ever exposed.

> [!TIP]
> **Defender view**
> - Enable **Block Public Access at the account level**, not just per-bucket, unless you have a specific, reviewed reason for a particular bucket to be public (such as hosting public static website assets).
> - Enable **default encryption** on every bucket, and require HTTPS via a bucket policy like the one above.
> - Use **AWS Config** or **Azure Policy** rules to continuously detect public buckets and alert immediately, rather than relying on anyone remembering to check.
> - Turn on **S3 Server Access Logging** or **CloudTrail data events** for sensitive buckets, so "who read this file, and when" can actually be answered during an investigation (Chapter 6's reporting and evidence discipline, applied to cloud storage).

---

## 7. Common misconfigurations

### Concept

Real-world cloud incidents overwhelmingly trace back to a short list of repeated mistakes. This section is a checklist built from that pattern, tying every earlier chapter's thinking into the cloud context.

| # | Misconfiguration | Why it happens | Chapter parallel |
|---|--------------------|------------------|-------------------|
| 1 | **Public storage buckets** with sensitive data | Old buckets predating 2023 defaults, or deliberately disabled Block Public Access "to make something work" | Section 6 |
| 2 | **Overly broad IAM policies** (`"*":"*"`) | Easier to write than a precise policy; "I'll tighten it later" | Section 4, Chapter 2's `chmod 777` |
| 3 | **Security Groups open to `0.0.0.0/0`** on SSH/RDP/database ports | Left open "temporarily" during setup and never closed | Section 5, Chapter 6 |
| 4 | **Hardcoded credentials** in code, images, or environment variables | Convenience during development, then shipped by accident | Chapter 3's "never put secrets in code" |
| 5 | **Logging disabled or not reviewed** (no CloudTrail, no alerts) | Seen as a cost or a "set it up later" task | Chapter 7's "a scanner finding nobody looks at is as bad as no scan" |
| 6 | **No MFA on privileged accounts** | Treated as an inconvenience | Chapter 5's authentication failures |
| 7 | **Unpatched guest operating systems** on IaaS virtual machines | Forgetting that the OS inside an EC2/VM instance is still **your** job (section 1's shared responsibility line) | Chapter 7's vulnerability management |
| 8 | **Default or weak credentials** on managed services (databases, admin consoles) | Never changed after deployment | Chapter 5's authentication basics |
| 9 | **Shadow IT**: resources created outside of any inventory or approval process | Convenience, speed, lack of governance | Chapter 6's "you cannot protect assets you do not know exist" |
| 10 | **Excessive data retention** in storage no one is actively managing | "We might need it someday," forgotten, never reviewed | Increases the blast radius of any one of the mistakes above |

### A short case pattern seen repeatedly in real incident write-ups

```text
1. Developer commits code with a hardcoded AWS access key to a public GitHub repo (mistake #4)
2. An automated bot finds the key within minutes
3. The key's attached IAM policy is far broader than the one task it was meant for (mistake #2)
4. The attacker uses it to find and read a forgotten, publicly-misconfigured S3 bucket (mistake #1)
5. No CloudTrail alerting was in place to flag the unusual API activity (mistake #5)
6. The organisation finds out only when a security researcher or journalist reports the exposed data
```

Every step in that chain maps to a specific, well-known, preventable misconfiguration, and every single one of them is a **configuration decision**, not a flaw in AWS or Azure itself. This is the shared responsibility model from section 1, made concrete.

> [!IMPORTANT]
> **Automated scanning tools exist specifically for this checklist.** **ScoutSuite**, **Prowler** (AWS-focused) and the cloud providers' own **AWS Trusted Advisor** / **Azure Advisor** / **Microsoft Defender for Cloud** all check for these exact patterns automatically, continuously, across an entire account. Running one of these, and actually reading and acting on the output, is one of the highest-value things a small security team can do (Chapter 7's vulnerability assessment discipline, aimed at cloud configuration instead of hosts and services).

---

## 8. Hands-on labs

Use your own free-tier AWS and/or Azure account only. Set a billing alert (Lab 1) before anything else, and **delete every resource at the end of each lab**, even ones you think are free-tier, to avoid any surprise charge.

### Lab 1: Account setup and a billing alarm

Create a free-tier AWS account (or use an existing one). Immediately:

```bash
aws iam create-virtual-mfa-device --virtual-mfa-device-name root-mfa --outfile qr.png --bootstrap-method QRCodePNG
```

Enable MFA on the root user via the console using an authenticator app. Then, in the Billing console, create a **budget alert** (for example, "alert me if forecasted spend exceeds $5").

**Goal:** MFA confirmed on the root user, and a billing alert active, before you create a single other resource.

### Lab 2: `whoami` for the cloud

```bash
aws configure                           # use an IAM user's keys, never root's
aws sts get-caller-identity
aws iam list-users
aws iam list-attached-user-policies --user-name your-iam-username
```

**Goal:** identify exactly which identity you are using and exactly which policies are attached to it, in your own words.

### Lab 3: Launch, inspect and lock down an EC2 instance

Launch a free-tier-eligible EC2 instance (`t2.micro` or `t3.micro`, depending on your Region's free tier). During launch, create a Security Group that allows SSH only from your own current IP address (not `0.0.0.0/0`).

```bash
curl -s https://checkip.amazonaws.com                        # find your own current public IP
aws ec2 describe-security-groups --query "SecurityGroups[].IpPermissions"
ssh -i your-key.pem ubuntu@<instance-public-ip>
```

**Goal:** connect successfully from your own IP, then edit the rule to temporarily allow `0.0.0.0/0` on port 22, observe (by checking from a phone's mobile data, or just reasoning about it) that this would now accept connections from anywhere, and change it back.

### Lab 4: S3 bucket, public access, and locking it down

```bash
aws s3 mb s3://your-unique-lab-bucket-name-12345
aws s3api get-public-access-block --bucket your-unique-lab-bucket-name-12345
echo "lab file, not sensitive" > test.txt
aws s3 cp test.txt s3://your-unique-lab-bucket-name-12345/
```

**Goal:** confirm Block Public Access is enabled by default (section 6), then deliberately write a bucket policy that would make the object public, observe the API rejecting it because of Block Public Access, then explicitly disable Block Public Access, apply the same policy, and confirm the file really is now publicly reachable at its URL, before deleting the bucket entirely.

### Lab 5: Write and apply a least-privilege IAM policy

Create an IAM user with **no permissions at all**, then write the narrowest possible policy that allows it to only `s3:GetObject` and `s3:ListBucket` on the one bucket from Lab 4, and attach it.

**Goal:** confirm this user can read that one bucket but gets an explicit `AccessDenied` when trying `aws s3 ls` against your other buckets or `aws ec2 describe-instances`.

### Lab 6: Enable and read CloudTrail

```bash
aws cloudtrail describe-trails
aws cloudtrail lookup-events --max-results 10 --query "Events[].{Time:EventTime,User:Username,Event:EventName}"
```

**Goal:** find the exact CloudTrail event corresponding to one of your earlier labs (for example, creating the S3 bucket or launching the EC2 instance), and note which fields would matter most during a real investigation.

### Lab 7: Run a free cloud security scanner

```bash
pip install scoutsuite --break-system-packages
scout aws --report-dir scout-report
```

**Goal:** open the generated report and find at least three findings. For each, say which section of this chapter it maps to, and whether you consider it a real risk in your tiny lab account or a reasonable finding that would matter more in a production environment.

### Lab 8: Clean up everything

```bash
aws ec2 terminate-instances --instance-ids i-xxxxxxxxxxxxxxxxx
aws s3 rb s3://your-unique-lab-bucket-name-12345 --force
aws iam delete-user-policy --user-name lab-user --policy-name lab-s3-read
aws iam delete-user --user-name lab-user
```

**Goal:** confirm, by listing resources again, that nothing billable is still running, and write one paragraph on what you would check weekly if this were a real, ongoing account.

> [!NOTE]
> **Next step on TryHackMe:** work through an AWS fundamentals room and a cloud security room (several platforms offer "IAM privilege escalation" style rooms specifically). Write each up with the exact commands, evidence, and the defender fix, exactly as in every earlier chapter.

---

## 9. Quick revision sheet

### The phrasebook

| Concept | AWS | Azure |
|---------|-----|-------|
| Identity | IAM | Microsoft Entra ID + Azure RBAC |
| Virtual machine | EC2 | Virtual Machines |
| Object storage | S3 | Blob Storage |
| Virtual network | VPC | Virtual Network (VNet) |
| Instance firewall | Security Groups | Network Security Groups |
| Managed database | RDS | Azure SQL Database |
| Serverless | Lambda | Azure Functions |
| API/audit log | CloudTrail | Activity Log + sign-in logs |
| Monitoring | CloudWatch | Azure Monitor |
| Managed threat detection | GuardDuty | Microsoft Defender for Cloud |
| Billing/resource boundary | Account | Subscription |

### Commands

| Task | AWS CLI | Azure CLI |
|------|---------|-----------|
| **Who am I?** | `aws sts get-caller-identity` | `az account show` |
| **List storage** | `aws s3 ls` | `az storage account list -o table` |
| **List compute** | `aws ec2 describe-instances` | `az vm list -o table` |
| **Check public access on a bucket** | `aws s3api get-public-access-block --bucket name` | `az storage account show -n name --query allowBlobPublicAccess` |
| **View recent API activity** | `aws cloudtrail lookup-events` | `az monitor activity-log list` |
| **Run a free config scanner** | `scout aws` (ScoutSuite) | `scout azure` (ScoutSuite) |

### One-line summaries

- **Security OF the cloud is the provider's job. Security IN the cloud is yours**, always, in every model.
- **IAM is the new perimeter.** A wildcard policy is as dangerous as `chmod 777`.
- **Roles for services, MFA for humans, never long-lived keys where a role will do.**
- **New S3 buckets are private by default since April 2023, but old buckets and careless policies can still expose data.**
- **`0.0.0.0/0` is fine on 80/443 for a public web server, and a finding almost everywhere else.**
- **Almost every famous cloud breach is a misconfiguration, not a flaw in the cloud platform itself.**

---

## 10. Self-check quiz

Answer without looking back, then open the key.

1. In the shared responsibility model, who is always responsible for identity and access management, in every service model?
2. What is the key practical difference between an IAM user and an IAM role?
3. Why is a policy with `"Action":"*","Resource":"*"` dangerous, even if no attack is happening yet?
4. Since when have new S3 buckets had Block Public Access enabled by default, and why does that not mean every bucket is safe?
5. Why is `0.0.0.0/0` acceptable on port 443 but usually a finding on port 22?
6. Name the cloud equivalent of a Linux `auth.log`.
7. What is the first command you should run in any new AWS CLI session, and why?
8. Give two reasons hardcoded cloud credentials in a public repository are especially dangerous.
9. Name two free tools that automatically scan a cloud account for common misconfigurations.
10. A company's S3 bucket, created in 2021, is found publicly exposing customer data in 2026. Using this chapter's ideas, give the most likely explanation.

<details>
<summary>Show answer key</summary>

1. The customer (you), in every model — IaaS, PaaS and SaaS alike.
2. A user has long-lived credentials meant for a person. A role is assumed temporarily and issues short-lived credentials, and is the correct choice for services like EC2 instances or Lambda functions.
3. It grants every permission on every resource. Even with no active attack, it means any single compromise of whatever holds that policy (a server, an application, a leaked key) immediately grants full control of the account, with no additional work needed by an attacker.
4. Since April 2023, for newly created buckets. Buckets created before that date did not get the new default retroactively, and anyone can still deliberately disable Block Public Access or write a bucket policy that grants public access regardless of the default.
5. Port 443 (HTTPS) is the intended public entry point for a web server, so exposing it to the whole internet is the point. Port 22 (SSH) is an administrative/management port; exposing it to the whole internet massively increases the attack surface for brute force and credential attacks with no corresponding benefit.
6. AWS CloudTrail (or Azure's Activity Log plus Microsoft Entra sign-in logs): it records who did what, when, and from where, at the account/API level.
7. `aws sts get-caller-identity` (or `az account show` on Azure), to confirm exactly which identity and account you are operating as before taking any action.
8. Any two of: automated bots scan public repositories and find leaked keys within minutes; the key may carry far broader permissions than the one task it was meant for; a stolen key gives direct API access without needing to exploit any other vulnerability.
9. Any two of: ScoutSuite, Prowler, AWS Trusted Advisor, Azure Advisor, Microsoft Defender for Cloud.
10. The bucket predates the April 2023 default change, so it never automatically received Block Public Access, and nobody reviewed or manually enabled it in the years since, which is exactly the "old bucket under old defaults, never revisited" pattern described in section 6 and the case pattern in section 7.

</details>

---

## What comes next

**Chapter 11: Blue Team / SOC.** You have now covered systems, networks, applications and the cloud. Next you bring it all together from the defender's seat: SIEM tools such as Splunk and Wazuh, log analysis at scale, incident response, and threat hunting.

[Back to roadmap](../README.md) | [Previous: Chapter 9 Network Analysis](../09-network-analysis/README.md) | [Download the PDF version](Chapter-10-Cloud-Security.pdf)
