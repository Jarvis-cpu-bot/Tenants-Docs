# Mailbox Pipeline Guide

A complete, beginner-friendly walkthrough of how this system creates room mailboxes
in Microsoft 365 Exchange Online -- from license assignment all the way to disabling
calendars. Written for someone who is NOT a developer.

---

## Table of Contents

1. [What is the Mailbox Pipeline?](#what-is-the-mailbox-pipeline)
2. [Prerequisites](#prerequisites)
3. [The 8-Step Pipeline](#the-8-step-pipeline)
   - [Step 1: Assign Exchange License](#step-1-assign-exchange-license)
   - [Step 2: Add Domain + DNS](#step-2-add-domain--dns)
   - [Step 3: Verify Domain](#step-3-verify-domain)
   - [Step 4: Setup DMARC](#step-4-setup-dmarc)
   - [Step 5: Create Mailboxes](#step-5-create-mailboxes)
   - [Step 6: Enable Org SMTP Auth](#step-6-enable-org-smtp-auth)
   - [Step 7: Enable SMTP per Mailbox](#step-7-enable-smtp-per-mailbox)
   - [Step 8: Disable Calendar](#step-8-disable-calendar)
4. [DNS Records Deep Dive](#dns-records-deep-dive)
5. [SMTP OAuth Explained](#smtp-oauth-explained)
6. [Name Variations System](#name-variations-system)
7. [Common Failures and Troubleshooting](#common-failures-and-troubleshooting)
8. [Manual Verification Checklist](#manual-verification-checklist)

---

## What is the Mailbox Pipeline?

### What are we creating?

Imagine you need to set up dozens (or even hundreds) of email addresses under a
custom domain -- like `john.smith@yourcompany.com`. Doing that by hand in
Microsoft 365 would take forever. The Mailbox Pipeline is an automated system
that does all of that work for you in one go.

Specifically, we are creating **room mailboxes** in Microsoft Exchange Online.

### Why room mailboxes, not regular user mailboxes?

A "regular" user mailbox in Microsoft 365 costs money -- each one requires its
own paid license. A **room mailbox** is a special type of mailbox that
Microsoft designed for conference rooms and shared resources. Here is why we
use them:

- **No per-mailbox license required.** Room mailboxes are included in your
  existing Microsoft 365 subscription. You can create dozens of them without
  buying more licenses.
- **They can still send and receive email.** Even though they are "room"
  mailboxes, they work as fully functional email addresses. You can send
  from them, receive email on them, and use them programmatically via SMTP.
- **Lower cost at scale.** If you need 50 or 100 email addresses, room
  mailboxes save significant money compared to user mailboxes.

Think of it like this: Microsoft built "room mailboxes" for booking conference
rooms, but the underlying technology is the same as a regular mailbox. We take
advantage of that.

### What does the pipeline automate?

The pipeline handles everything you would otherwise have to do manually:

1. Making sure your Microsoft 365 account has the right license for Exchange.
2. Registering your custom domain (like `yourcompany.com`) with Microsoft 365.
3. Setting up all the DNS records (the "phone book of the internet") so that
   email for your domain routes through Microsoft.
4. Verifying that you actually own the domain.
5. Configuring email security (SPF, DKIM, DMARC) so your emails are not
   rejected as spam.
6. Creating all the mailboxes with realistic-looking names.
7. Enabling SMTP (the protocol that actually sends emails) on each mailbox.
8. Disabling calendar features (because these are not real conference rooms).

All 8 steps run automatically in sequence. If a non-critical step fails, the
pipeline keeps going and marks that step with a warning. If a critical step
fails, the pipeline stops and reports the error.

---

## Prerequisites

Before you can run the Mailbox Pipeline, the following things must already exist:

### 1. A completed Tenant Setup

The "tenant" is your Microsoft 365 account. Before creating mailboxes, the
tenant setup pipeline (a separate 13-step process) must have already run
successfully. This gives us:

- **Client ID** -- an identifier for our application registered in Microsoft's
  system (Azure Active Directory).
- **Client Secret** -- a password for that application (encrypted in our
  database).
- **Tenant ID** -- Microsoft's unique identifier for your organization.
- **Admin Email** -- the administrator account (e.g.,
  `admin@yourcompany.onmicrosoft.com`).
- **Exchange Admin Role** -- our application must have been granted the
  "Exchange Administrator" role during tenant setup. This role takes
  15-30 minutes to propagate for brand-new tenants.
- **Certificate (optional but preferred)** -- a PFX certificate for
  authenticating with Exchange Online PowerShell without using passwords.

### 2. A Cloudflare Zone for the Domain

The domain you want to create mailboxes on (e.g., `yourcompany.com`) must
already be managed by Cloudflare. Cloudflare is the DNS provider we use to
create and manage DNS records. Specifically:

- A **Cloudflare account** with API credentials (email + Global API key)
  must be stored in our system.
- The domain must have an active **Cloudflare zone** -- meaning Cloudflare
  is the authoritative DNS server for that domain.

### 3. A Domain Name

You need a domain name (like `yourcompany.com`) that:

- You own and control.
- Is already added to Cloudflare.
- Is NOT already registered in another Microsoft 365 tenant (you cannot use
  the same domain in two tenants simultaneously).

### 4. A Running Celery Worker

The pipeline runs as a background task using Celery (a task queue system).
The Celery worker must be running and listening on the `mailbox_queue`:

```
celery -A config worker -l info --pool=solo -Q celery,mailbox_queue
```

Without the `-Q mailbox_queue` flag, the worker will not pick up mailbox
pipeline tasks, and they will sit in the queue forever.

---

## The 8-Step Pipeline

Here is each step explained in detail.

---

### Step 1: Assign Exchange License

**What it does (simple):**
Checks if your Microsoft 365 admin account has an Exchange license and
assigns one if it does not.

**What it does (technical):**

1. Calls the Microsoft Graph API endpoint `GET /subscribedSkus` to list
   all licenses available in the tenant.
2. Searches through those licenses looking for one that includes Exchange
   Online. It checks in order of preference:
   - `EXCHANGESTANDARD` (standalone Exchange license)
   - `EXCHANGEENTERPRISE` (higher-tier Exchange license)
   - `EXCHANGE` (any Exchange-related SKU)
   - `ENTERPRISEPACK` (Office 365 E3, which includes Exchange)
   - `BUSINESS_PREMIUM` (Microsoft 365 Business Premium)
   - `SPB`, `SMB_BUSINESS`, `O365_BUSINESS` (other bundles)
3. If a license is found with available units (not all consumed), it checks
   if the admin user already has it assigned via
   `GET /users/{email}/licenseDetails`.
4. If the admin does not have the license, it assigns one via
   `POST /users/{email}/assignLicense`.

**Why we need this:**
Exchange Online features (creating mailboxes, managing mail flow) require
at least one licensed user in the tenant. The admin account is the natural
choice. Without this license, later steps that talk to Exchange would fail.

**What gets saved:**
- `tenant.license_sku_id` -- the ID of the assigned license.
- `tenant.license_sku_name` -- the human-readable name (e.g., `ENTERPRISEPACK`).
- `tenant.license_assigned_at` -- timestamp of when the license was assigned.
- `tenant.license_units_available` -- total license seats available.
- `tenant.license_units_consumed` -- how many seats are in use.

**What can go wrong:**

- **"No Exchange Online license SKU found"** -- The tenant does not have
  any license that includes Exchange. This is non-blocking: the pipeline
  continues because licenses might be pre-assigned by another admin. If
  mailbox creation fails later, this is likely the root cause. **Fix:**
  Purchase and assign an Exchange Online license in the Microsoft 365
  admin center.
- **Graph API 403 Forbidden** -- The application does not have permission
  to read or assign licenses. **Fix:** Ensure the app has
  `Organization.Read.All` and `User.ReadWrite.All` Graph API permissions.

**How to check manually:**

- Go to the Microsoft 365 Admin Center -> Billing -> Licenses.
- Verify the admin account has an Exchange-capable license assigned.
- Or use Graph API: `GET https://graph.microsoft.com/v1.0/subscribedSkus`

**Edge cases:**

- If all license seats are consumed (available - consumed = 0), the step
  quietly moves on without assigning. It does not purchase new licenses.
- This step is NON-BLOCKING -- if it fails, the pipeline continues.

---

### Step 2: Add Domain + DNS

**What it does (simple):**
Registers your custom domain with Microsoft 365 and creates all the DNS
records needed for email to work.

**What it does (technical):**

This is the most complex step. It does the following in order:

1. **Add domain to Microsoft 365** -- calls the Graph API
   `POST /domains` with the domain name. If the domain already exists
   in M365, it skips this.

2. **Fetch verification records** -- calls
   `GET /domains/{domain}/verificationDnsRecords` to get the TXT record
   Microsoft wants you to add to prove you own the domain. This is
   retried up to 10 times with 10-second delays because Microsoft can
   take 60-90 seconds to generate the verification record.

3. **Look up the Cloudflare zone** -- calls the Cloudflare API
   `GET /zones?name=yourdomain.com` to find the zone ID.

4. **Create 6 DNS records in Cloudflare:**

   | # | Type  | Name                              | Value                                                  |
   |---|-------|-----------------------------------|--------------------------------------------------------|
   | 1 | TXT   | yourdomain.com                    | MS=ms12345678 (Microsoft verification code)            |
   | 2 | MX    | yourdomain.com                    | yourdomain-com.mail.protection.outlook.com (priority 0)|
   | 3 | TXT   | yourdomain.com                    | v=spf1 include:spf.protection.outlook.com ~all         |
   | 4 | CNAME | autodiscover.yourdomain.com       | autodiscover.outlook.com                               |
   | 5 | CNAME | selector1._domainkey.yourdomain.com| selector1-yourdomain-com._domainkey.tenant.onmicrosoft.com |
   | 6 | CNAME | selector2._domainkey.yourdomain.com| selector2-yourdomain-com._domainkey.tenant.onmicrosoft.com |

5. **Verify all records were created** -- reads back all DNS records from
   Cloudflare and checks that all 6 are present.

6. **Clean duplicate SPF records** -- if multiple SPF TXT records exist
   (which breaks email authentication), the extras are deleted
   automatically, keeping only the first one.

**Why we need this:**
Microsoft needs to know that your domain exists and that you own it.
Email on the internet relies on DNS records to route messages to the
right server. Without MX records, email for your domain would not reach
Microsoft's servers. Without SPF/DKIM records, your sent emails would
be flagged as spam or rejected.

**What gets saved:**
- 6 DNS records in Cloudflare (visible in the Cloudflare dashboard).
- No database changes at this step -- the domain model is updated in
  Step 3 after verification succeeds.

**What can go wrong:**

- **"No Cloudflare zone found for domain"** -- The domain is not in the
  Cloudflare account whose credentials we have. **Fix:** Add the domain
  to Cloudflare, or update the Cloudflare config with the correct
  account.
- **"No TXT verification record returned by Microsoft"** -- Microsoft
  has not generated the verification code yet, even after 10 retries.
  **Fix:** Wait a few minutes and retry the pipeline.
- **Cloudflare API 429 (rate limited)** -- Too many API calls in a short
  time. The system retries automatically with exponential backoff (wait
  5s, 10s, 20s, up to 60s).
- **Duplicate SPF records** -- If you or someone else already added SPF
  records manually, duplicates will cause SPF authentication to fail.
  The pipeline detects and auto-cleans these.

**How to check manually:**

- Log into Cloudflare -> select the domain -> DNS -> Records.
- Verify you see all 6 records listed in the table above.
- Use an online tool like `mxtoolbox.com` to look up your domain's
  MX, SPF, and DKIM records.

**Edge cases:**

- The `upsert_dns_record` function is idempotent -- calling it twice
  will update the existing record rather than creating a duplicate.
  This means rerunning Step 2 is safe.
- DKIM selector CNAMEs are created now but DKIM signing is NOT enabled
  yet. That requires DNS propagation (24-48 hours) and is triggered
  separately via a "Setup DKIM" button.
- This step is BLOCKING -- if it fails, the pipeline stops.

---

### Step 3: Verify Domain

**What it does (simple):**
Tells Microsoft to check the DNS records you just created and confirm
that you own the domain.

**What it does (technical):**

This step has three phases:

**Phase 1: Graph API Domain Verification**

1. Calls `POST /domains/{domain}/verify` via the Microsoft Graph API.
2. Uses exponential backoff retry: waits 10s, then 20s, then 40s, then
   60s, then 60s repeatedly -- up to approximately 10 minutes total.
3. Each attempt checks if the response contains `isVerified: true`.
4. If the domain is already verified in M365 (a "VerifiedDomainExists"
   error), it treats this as a success.

**Phase 2: Short Propagation Wait**

After verification succeeds, the system waits 30 seconds for Exchange
Online to recognize the newly verified domain.

**Phase 3: Domain Readiness Probe**

This is a clever check: the system tries to create a temporary test
mailbox (`probe-test@yourdomain.com`) via PowerShell. If Exchange
accepts the command, the domain is truly ready for mailbox creation.
The probe mailbox is immediately deleted after creation.

- Retries up to 5 times with delays of 15s, 30s, 60s, 60s, 60s.
- If the probe fails after all attempts, it logs a warning but does
  NOT stop the pipeline. The re-queue mechanism handles failures later.

**Why we need this:**
Microsoft will not let you create mailboxes on a domain it does not
recognize as yours. The verification proves ownership. The readiness
probe ensures Exchange Online has actually provisioned the domain
internally -- there is a gap between "verified in Graph API" and
"ready in Exchange Online."

**What gets saved:**
- `Domain.is_verified = True` in the database.
- The Domain model record is created if it did not exist.

**What can go wrong:**

- **"Domain is already registered in another Microsoft 365 tenant"** --
  Someone else has already claimed this domain. This is a fatal error
  with no automatic fix. **Fix:** Remove the domain from the other
  tenant first, or use a different domain.
- **Verification times out after 10 minutes** -- DNS has not propagated
  yet. DNS changes can take anywhere from 1 minute to 48 hours to
  propagate globally, though Cloudflare is usually very fast (under 5
  minutes). **Fix:** Wait 15-30 minutes and retry.
- **Domain readiness probe fails** -- Exchange Online has not finished
  provisioning the domain internally. This is common with brand-new
  tenants. **Fix:** The system auto-retries. If it continues failing,
  wait 30 minutes and rerun the pipeline.

**How to check manually:**

- Go to Microsoft 365 Admin Center -> Settings -> Domains. Your domain
  should show as "Verified" or "Healthy."
- Or use Graph API: `GET https://graph.microsoft.com/v1.0/domains/{domain}`
  and check `isVerified`.

**Edge cases:**

- The pipeline updates the job's heartbeat timestamp during each retry
  attempt. This prevents the monitoring system from thinking the job
  is stuck.
- If a blocking step fails with an authentication/propagation error
  (e.g., "unauthorized", "access denied"), the entire pipeline is
  re-queued to run again in 10 minutes automatically.
- This step is BLOCKING -- if it fails, the pipeline stops.

---

### Step 4: Setup DMARC

**What it does (simple):**
Creates a special DNS record that tells email servers what to do when
they receive an email claiming to be from your domain but failing
authentication checks.

**What it does (technical):**

1. Constructs the DMARC record value:
   ```
   v=DMARC1; p=reject; sp=reject; aspf=r; adkim=r; rua=mailto:dmarc@yourdomain.com
   ```
2. Calls `upsert_dns_record` to create (or update) a TXT record at
   `_dmarc.yourdomain.com` in Cloudflare.
3. Queries Cloudflare for all TXT records at `_dmarc.yourdomain.com`
   and deletes any duplicates. Having multiple DMARC records causes
   DMARC to fail entirely.
4. Updates the Domain model: sets `dmarc_created = True`.

**Why we need this:**
DMARC (Domain-based Message Authentication, Reporting, and Conformance)
is the "policy layer" of email security. Think of it this way:

- SPF says "these servers are allowed to send email for my domain."
- DKIM says "I digitally sign my emails so you can verify they have
  not been tampered with."
- DMARC says "if an email fails both SPF and DKIM checks, here is
  what you should do with it."

Our DMARC policy is `p=reject`, which means: "If an email claiming
to be from our domain fails authentication, reject it completely."
This is the strictest setting and protects against email spoofing.

**What gets saved:**
- A TXT record at `_dmarc.yourdomain.com` in Cloudflare.
- `Domain.dmarc_created = True` in the database.

**What can go wrong:**

- **Duplicate DMARC records** -- If a DMARC record was created manually
  before the pipeline ran, you could end up with duplicates. The
  pipeline detects and auto-cleans these. **How to verify:** Check
  Cloudflare DNS for the `_dmarc` subdomain -- there should be
  exactly one TXT record.
- **Cloudflare API failure** -- Network issues or rate limiting.
  The Cloudflare service has built-in retry logic.

**How to check manually:**

- Cloudflare Dashboard -> DNS -> look for `_dmarc.yourdomain.com`
  TXT record.
- Use command line: `nslookup -type=TXT _dmarc.yourdomain.com`
- Use `mxtoolbox.com` -> DMARC Lookup.

**Edge cases:**

- This step is NON-BLOCKING. If it fails, the pipeline continues.
  Emails will still work without DMARC, but they are more
  vulnerable to spoofing.
- The `upsert_dns_record` function intelligently detects existing
  DMARC records and updates them instead of creating duplicates.

---

### Step 5: Create Mailboxes

**What it does (simple):**
Generates all the email addresses with realistic-looking names and
creates them as room mailboxes in Exchange Online.

**What it does (technical):**

1. **Generate names** -- uses the Name Variations System (see
   [that section below](#name-variations-system)) to create unique
   display names and email prefixes. If custom names were provided,
   it generates variations of those names. Otherwise, it picks
   random names from a pool of 100 first names and 100 last names.

2. **Build mailbox specs** -- for each generated name, creates a
   spec with `name`, `email`, and `password`. The email is
   `{prefix}@yourdomain.com`. The password is either a provided
   default or `AtoZ@123`.

3. **Skip existing mailboxes** -- checks the database for any
   mailboxes that already exist for this domain and skips those
   (makes the step safe to rerun).

4. **Batch creation** -- sends mailboxes to Exchange Online in
   batches of 25 (configurable). For each batch, it runs a single
   PowerShell session that:

   a. Connects to Exchange Online (using certificate or OAuth token).

   b. For each mailbox, runs:
      ```powershell
      New-Mailbox -Room -Name 'John Smith' -DisplayName 'John Smith'
        -Alias 'johnsmith' -PrimarySmtpAddress 'johnsmith@domain.com'
        -EnableRoomMailboxAccount $true
        -MicrosoftOnlineServicesID 'johnsmith@domain.com'
        -RoomMailboxPassword $securePassword
      ```

   c. If a "proxy conflict" error occurs (the alias is already taken),
      it automatically retries with a random suffix:
      `johnsmith` becomes `johnsmith42`.

   d. After creating each mailbox, it immediately:
      - Hides it from the Global Address List
        (`Set-Mailbox -HiddenFromAddressListsEnabled $true`)
      - Disables calendar auto-processing
        (`Set-CalendarProcessing -AutomateProcessing None`)
      - Enables SMTP authentication
        (`Set-CASMailbox -SmtpClientAuthenticationDisabled $false`)

   e. Each result is output as a parseable line:
      `MBRESULT:{"email":"...", "status":"success", "message":"..."}`

5. **Parse results** -- reads each `MBRESULT` line from the
   PowerShell output and creates database records for successful
   mailboxes.

6. **Grant SMTP OAuth2 permissions** -- for all successfully created
   mailboxes, grants `FullAccess` and `SendAs` permissions to the
   application's service principal. This is needed for OAuth2 SMTP
   sending (see [SMTP OAuth Explained](#smtp-oauth-explained)).

**Why we need this:**
This is the core of the pipeline -- actually creating the email
accounts. Everything before this was setup; everything after is
configuration.

**What gets saved:**
- A `Mailbox` record in the database for each successfully created
  mailbox, containing:
  - `email` -- the full email address.
  - `display_name` -- the human-readable name.
  - `password_enc` -- the password, encrypted with Fernet.
  - `smtp_enabled = True` -- flag indicating SMTP is turned on.
  - Links to the `tenant` and `domain`.

**What can go wrong:**

- **"PROXY conflict"** -- The email alias collides with an existing
  object in Exchange. The pipeline auto-resolves this by appending
  a random 2-digit number. **Example:** `johnsmith@domain.com`
  becomes `johnsmith42@domain.com`.
- **Batch timeout** -- Each mailbox takes about 15-18 seconds to
  create. A batch of 25 takes about 7-8 minutes. The system
  calculates a dynamic timeout: `max(300, count * 18 + 60)` seconds.
  Very large batches could still time out. **Fix:** Reduce the batch
  size by setting `batch_size` in the job config.
- **No mailboxes created** -- If every single mailbox in the batch
  fails, the step raises an error with details. **Common cause:**
  The domain is not yet ready in Exchange Online (see Step 3's
  readiness probe).
- **PowerShell authentication failure** -- The app's credentials or
  certificate are not working. If this is a new tenant, Exchange
  Online may need more time to propagate the admin role. The
  pipeline auto-re-queues on these errors.

**How to check manually:**

- Go to Exchange Admin Center
  (`https://admin.exchange.microsoft.com`) -> Recipients ->
  Resources. Your room mailboxes should appear there.
- Or use PowerShell:
  ```powershell
  Connect-ExchangeOnline
  Get-Mailbox -RecipientTypeDetails RoomMailbox -ResultSize Unlimited |
    Where-Object { $_.PrimarySmtpAddress -like '*@yourdomain.com' }
  ```

**Edge cases:**

- The timeout is dynamically calculated based on the number of
  mailboxes. A batch of 100 gets `100 * 18 + 60 = 1860 seconds`
  (31 minutes).
- If the pipeline is rerun, already-created mailboxes are skipped
  (idempotent).
- Display names must be unique in Exchange. The variations system
  ensures each mailbox gets a distinct display name.
- This step is BLOCKING -- if zero mailboxes are created, the
  pipeline stops.

---

### Step 6: Enable Org SMTP Auth

**What it does (simple):**
Turns on the tenant-wide setting that allows mailboxes to send email
using the SMTP protocol.

**What it does (technical):**

1. **Set-TransportConfig** -- runs the PowerShell command:
   ```powershell
   Set-TransportConfig -SmtpClientAuthenticationDisabled $false
   ```
   This is the organization-level switch. If this is `$true` (disabled),
   no mailbox in the entire tenant can use SMTP to send email,
   regardless of individual mailbox settings.

2. **Create AllowBasicSMTP Authentication Policy** -- runs:
   ```powershell
   New-AuthenticationPolicy -Name 'AllowBasicSMTP' -AllowBasicAuthSmtp
   Set-OrganizationConfig -DefaultAuthenticationPolicy 'AllowBasicSMTP'
   ```
   New Microsoft 365 tenants block basic SMTP authentication by default
   (even with Security Defaults disabled). This creates a policy that
   explicitly allows it and sets it as the default for the organization.

**Why we need this:**
SMTP is the protocol our system uses to actually send emails through
these mailboxes. Microsoft has been progressively disabling basic
SMTP authentication for security reasons. We need to ensure the
organization-level SMTP switch is turned on. (We also use OAuth2
SMTP -- see [SMTP OAuth Explained](#smtp-oauth-explained) -- but the
transport config must still allow SMTP client connections.)

**What gets saved:**
- No database changes.
- The Exchange Online transport configuration is modified.
- An authentication policy named "AllowBasicSMTP" is created in the
  tenant.

**What can go wrong:**

- **PowerShell authentication failure** -- Same as other steps. If
  the Exchange Admin role has not propagated, this will fail.
  **Fix:** Wait for propagation (auto-handled by the re-queue
  mechanism).
- **"AllowBasicSMTP already exists"** -- This is handled gracefully.
  The script catches this error and continues to set the policy as
  the default.

**How to check manually:**

- In PowerShell:
  ```powershell
  Connect-ExchangeOnline
  Get-TransportConfig | Select-Object SmtpClientAuthenticationDisabled
  # Should show: False
  Get-AuthenticationPolicy | Select-Object Name
  # Should include: AllowBasicSMTP
  ```

**Edge cases:**

- This step is NON-BLOCKING. If it fails, individual mailbox SMTP
  might still work if the tenant previously had SMTP enabled.
- The auth policy creation is idempotent -- "already exists" errors
  are caught and ignored.

---

### Step 7: Enable SMTP per Mailbox

**What it does (simple):**
Turns on SMTP sending ability for each individual mailbox that
does not already have it enabled.

**What it does (technical):**

1. Queries the database for all mailboxes on this domain where
   `smtp_enabled = False`.
2. For each such mailbox, runs the PowerShell command:
   ```powershell
   Set-CASMailbox -Identity 'user@domain.com'
     -SmtpClientAuthenticationDisabled $false
     -ActiveSyncEnabled $false
     -OWAEnabled $false
     -PopEnabled $false
     -ImapEnabled $false
   ```
3. On success, updates the mailbox record: `smtp_enabled = True`.

**Why we need this:**
Step 6 enabled SMTP at the organization level (the "master switch").
This step enables SMTP on each individual mailbox. Think of it like
a building's master power switch (Step 6) vs. the light switch in
each room (Step 7). Both must be on for the lights to work.

The command also disables other protocols (ActiveSync, OWA, POP,
IMAP) that we do not need. This follows the security principle of
least privilege -- only enable what is actually required.

**What gets saved:**
- `Mailbox.smtp_enabled = True` for each successfully configured
  mailbox.

**What can go wrong:**

- **"5.7.139 SMTP blocked"** -- Microsoft is still propagating the
  SMTP permissions. This can take up to 24 hours for new tenants.
  **Fix:** Wait and rerun the pipeline. The step only retries
  mailboxes where `smtp_enabled = False`, so it picks up where it
  left off.
- **Partial failure** -- Some mailboxes succeed, others fail. The
  step reports both counts. **Fix:** Rerun the pipeline; it will
  retry only the failed ones.

**How to check manually:**

- In PowerShell:
  ```powershell
  Connect-ExchangeOnline
  Get-CASMailbox -Identity 'user@domain.com' |
    Select-Object SmtpClientAuthenticationDisabled
  # Should show: False
  ```

**Edge cases:**

- If all mailboxes already have SMTP enabled (e.g., because Step 5
  enabled it during creation), this step does nothing and
  finishes instantly.
- This step is NON-BLOCKING for partial failures but raises an
  error if any mailboxes fail, so the pipeline marks it with a
  warning.

---

### Step 8: Disable Calendar

**What it does (simple):**
Turns off the automatic calendar/meeting features on all mailboxes
since they are not real conference rooms.

**What it does (technical):**

1. Queries the database for all mailbox emails on this domain.
2. Opens a single PowerShell session and runs, for each mailbox:
   ```powershell
   Set-CalendarProcessing -Identity 'user@domain.com'
     -AutomateProcessing None
   ```
3. Counts successes (`CALOK:` lines) and failures (`CALFAIL:` lines).

**Why we need this:**
Room mailboxes in Exchange Online have a "resource booking assistant"
that automatically accepts or declines meeting invitations. Since
these are not real rooms, we do not want them auto-responding to
calendar invites. Setting `AutomateProcessing` to `None` disables
this behavior entirely.

**What gets saved:**
- No database changes. The setting is applied directly in Exchange.

**What can go wrong:**

- **Partial failure** -- Some mailboxes might fail if Exchange is
  still provisioning them. **Fix:** Rerun the pipeline.
- **PowerShell timeout** -- With many mailboxes, the session could
  time out. The system calculates a dynamic timeout based on
  mailbox count.

**How to check manually:**

- In PowerShell:
  ```powershell
  Connect-ExchangeOnline
  Get-CalendarProcessing -Identity 'user@domain.com' |
    Select-Object AutomateProcessing
  # Should show: None
  ```

**Edge cases:**

- This is the final step. If it partially fails, the overall
  pipeline status is "warning" (not "failed") because the
  mailboxes are still functional for sending email.
- The batch approach (one session for all mailboxes) is much
  faster than opening a separate session per mailbox.

---

## DNS Records Deep Dive

DNS (Domain Name System) is like the phone book of the internet. When
someone sends an email to `user@yourdomain.com`, their email server
looks up DNS records to find out where to deliver it. Here is every
DNS record we create and why.

### MX Record (Mail Exchanger)

```
Type:     MX
Name:     yourdomain.com
Value:    yourdomain-com.mail.protection.outlook.com
Priority: 0
```

**What it is:** The MX record tells the internet "email for this domain
should be delivered to THIS server." It is the most fundamental email
DNS record.

**What the value means:** `yourdomain-com.mail.protection.outlook.com`
is Microsoft's email server for your domain. The dots in your domain
name are replaced with dashes. Priority 0 means "this is the
highest-priority (most preferred) mail server."

**Without this:** Email sent to your domain would bounce with "no mail
server found" errors. Nobody could email you.

### SPF Record (Sender Policy Framework)

```
Type:    TXT
Name:    yourdomain.com
Value:   v=spf1 include:spf.protection.outlook.com ~all
```

**What it is:** SPF tells receiving email servers which servers are
authorized to send email on behalf of your domain. It is an
anti-spoofing measure.

**Breaking down the value:**

- `v=spf1` -- "This is an SPF version 1 record."
- `include:spf.protection.outlook.com` -- "Microsoft's servers are
  authorized to send email for this domain." This references
  Microsoft's own SPF record, which lists all their mail server IPs.
- `~all` -- "For all other servers, apply a SOFTFAIL." This means
  emails from unauthorized servers are suspicious but not immediately
  rejected.

**Why `~all` (softfail) and not `-all` (hardfail)?**

We use `~all` because it is a "transitioning" posture. During the
initial setup period, some emails might legitimately come from
unexpected sources (forwarding, mailing lists, etc.). A softfail
marks them as suspicious without outright rejecting them. Once you
are confident all email flows through Microsoft, you can tighten
this to `-all`.

**What "transitioning" means:** The period after setting up SPF where
you monitor for legitimate senders that might not yet be included in
your SPF record. Checking DMARC reports (sent to the `rua` address)
helps identify these.

**Without this:** Your sent emails would likely be flagged as spam
because receiving servers cannot verify you authorized Microsoft
to send on your behalf.

### DKIM Records (DomainKeys Identified Mail)

```
Type:    CNAME
Name:    selector1._domainkey.yourdomain.com
Value:   selector1-yourdomain-com._domainkey.tenant.onmicrosoft.com

Type:    CNAME
Name:    selector2._domainkey.yourdomain.com
Value:   selector2-yourdomain-com._domainkey.tenant.onmicrosoft.com
```

**What it is:** DKIM adds a digital signature to every email you send.
The receiving server can use this signature to verify that the email
really came from your domain and was not modified in transit.

**What selector CNAMEs are:** Microsoft manages the actual DKIM
signing keys, but they need your DNS to point to where those keys
are published. The CNAME records say "to find my DKIM public key,
look at this Microsoft address." Think of it as a forwarding
address.

**Why 2 selectors?** Microsoft uses two selectors for key rotation.
When they periodically rotate (change) the DKIM signing key for
security reasons, they switch from selector1 to selector2 (or vice
versa). This ensures there is zero downtime -- the old key remains
valid while the new one starts being used.

**How signing works:**

1. When you send an email, Microsoft's server creates a hash (digital
   fingerprint) of the email content.
2. It encrypts this hash with a private key (that only Microsoft has).
3. It adds this encrypted hash as a `DKIM-Signature` header to the
   email.
4. The receiving server looks up the DKIM public key via your DNS
   CNAME record, decrypts the hash, and compares it to the email
   content.
5. If they match, the email is verified as authentic and unmodified.

**Without this:** Your emails would lack DKIM signatures, which is a
major red flag for spam filters. Many email providers (Gmail, Yahoo)
increasingly require DKIM for delivery.

### DMARC Record

```
Type:    TXT
Name:    _dmarc.yourdomain.com
Value:   v=DMARC1; p=reject; sp=reject; aspf=r; adkim=r; rua=mailto:dmarc@yourdomain.com
```

**What it is:** DMARC is the "policy" record that ties SPF and DKIM
together. It tells receiving servers what to do when an email fails
authentication checks.

**Breaking down the value:**

- `v=DMARC1` -- "This is a DMARC version 1 record."
- `p=reject` -- "If an email fails both SPF and DKIM checks, reject
  it completely." This is the strictest policy. Other options are
  `quarantine` (put it in spam) or `none` (just report, do not act).
- `sp=reject` -- Same policy for subdomains (e.g.,
  `mail.yourdomain.com`).
- `aspf=r` -- "SPF alignment mode is relaxed." This means the domain
  in the SPF check does not need to exactly match the From address --
  a subdomain match is acceptable.
- `adkim=r` -- "DKIM alignment mode is relaxed." Same idea as `aspf`
  but for DKIM signatures.
- `rua=mailto:dmarc@yourdomain.com` -- "Send aggregate DMARC reports
  to this email address." These reports tell you who is sending email
  on behalf of your domain and whether it passes or fails
  authentication.

**Without this:** Receiving servers would not know how to handle
emails that fail SPF/DKIM checks. Different servers would apply
different policies, leading to inconsistent delivery.

### Autodiscover CNAME

```
Type:    CNAME
Name:    autodiscover.yourdomain.com
Value:   autodiscover.outlook.com
```

**What it is:** Autodiscover is a Microsoft protocol that allows email
clients (Outlook, mobile mail apps) to automatically configure
themselves. When a user enters their email address, the email client
looks up `autodiscover.yourdomain.com` to find the mail server
settings.

**Without this:** Users would need to manually enter server names,
ports, and security settings when configuring their email client.
With this record, they just enter their email and password and
everything configures automatically.

### Microsoft Verification TXT

```
Type:    TXT
Name:    yourdomain.com
Value:   MS=ms12345678  (unique code from Microsoft)
```

**What it is:** A one-time verification record that proves you own the
domain. Microsoft gives you a unique code, and you add it to your DNS.
Microsoft then checks for this code to confirm ownership.

**After verification:** This record can technically be removed after
the domain is verified, but it is harmless to leave it in place and
provides a reference for which Microsoft tenant owns the domain.

---

## SMTP OAuth Explained

### Why not basic auth?

"Basic authentication" for email means sending a username and password
in (essentially) plain text. Microsoft has been deprecating basic auth
since 2020 because it is a major security risk:

- Passwords can be intercepted if the connection is compromised.
- There is no way to enforce multi-factor authentication (MFA).
- Credential stuffing attacks (trying leaked passwords) are trivially
  easy.

Microsoft's timeline: basic auth was default-disabled in December 2026,
with final removal planned for 2027 (postponed from the original March
2026 deadline).

### What is XOAUTH2?

XOAUTH2 is a way to authenticate to an SMTP server using an OAuth2
access token instead of a password. Here is the difference:

- **Basic auth:** "Hi server, my username is X and my password is Y."
- **XOAUTH2:** "Hi server, here is a temporary token that proves I
  have permission to send as this user."

The token is short-lived (usually 1 hour), so even if it is
intercepted, the window of exploitation is very small.

### How the token flow works

Our system uses the **client credentials flow**, which is designed for
server-to-server authentication (no human user involved):

```
1. Our server sends a request to Microsoft:
   POST https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token

   Body:
   - client_id = {our app's ID}
   - client_secret = {our app's password}
   - scope = https://outlook.office365.com/.default
   - grant_type = client_credentials

2. Microsoft validates the credentials and returns:
   { "access_token": "eyJ0eXAiOiJK..." }

3. Our server connects to smtp.office365.com:587, starts TLS, then:
   AUTH XOAUTH2 {base64-encoded auth string}

   The auth string format is:
   base64("user={email}\x01auth=Bearer {token}\x01\x01")

4. If Microsoft accepts the token, we get a 235 response (authenticated)
   and can now send emails.
```

### What SMTP.SendAsApp permission does

This is a Microsoft Graph application permission that allows your app
to send email as any user in the tenant. It is the permission that
makes the OAuth2 token work for SMTP sending.

Without this permission, the OAuth2 token would be valid but SMTP
would reject it with a "5.7.3 SMTP client was not authenticated"
error.

### What FullAccess + SendAs permissions do

These are Exchange Online mailbox-level permissions granted via
PowerShell:

- **FullAccess** -- allows the application's service principal to
  open and read the mailbox. This is needed for some SMTP
  authentication flows.
- **SendAs** -- allows the application to send email appearing to
  come from that mailbox's address.

Both permissions are granted to the Exchange Online service principal
(registered via `New-ServicePrincipal`), not directly to the Azure
AD app. This is because Exchange Online has its own permission
system separate from Azure AD.

The pipeline grants these permissions in Step 5 (Create Mailboxes),
immediately after the mailboxes are created.

---

## Name Variations System

The pipeline needs to create many mailboxes with unique, realistic-
looking names. Here is how the name generation works.

### When you provide custom names

If you provide custom names (e.g., "John Smith" and "Jane Doe"), the
system distributes the total mailbox count evenly across those names
and generates unique variations of each.

**Example:** You provide 2 names and request 50 mailboxes.
Each name gets 25 variations.

For "John Smith", the system generates variations in this order:

| # | Email Prefix | Display Name     | Pattern          |
|---|-------------|------------------|------------------|
| 1 | johnsmith   | John Smith       | firstlast        |
| 2 | jsmith      | J Smith          | flast            |
| 3 | smithjohn   | Smith John       | lastfirst        |
| 4 | john.smith  | John Smith       | first.last       |
| 5 | smithj      | Smith J          | lastf            |
| 6 | john.s      | John S.          | first.l          |
| 7 | j.smith     | J. Smith         | f.last           |
| 8 | jonsmith    | Jon Smith        | truncated first  |
| 9 | johnsmit    | John Smit        | truncated last   |
| 10| johsmith    | Joh Smith        | short first (3)  |
| 11| johnsmi     | John Smi         | short last (3)   |
| 12| smith.john  | Smith John       | last.first (dot) |
| 13| jons        | Jon S            | trunc first + last initial |
| 14| jsmit       | J Smit           | first initial + trunc last |
| 15| johnnsmith  | Johnn Smith      | doubled last char of first |
| 16| johnsmithh  | John Smithh      | doubled last char of last  |
| 17| smith.j     | Smith J.         | last.f           |
| 18| johnjsmith  | JohnJ Smith      | firstfirst       |
| 19+| johnsmith1 | John S1          | numbered         |
| 20+| johnsmith2 | John S2          | numbered         |
| ...| ...        | ...              | continues...     |

Each email prefix is sanitized: lowercased, only letters and numbers,
truncated to 20 characters maximum.

### When you do NOT provide custom names

The system picks random combinations from a pool of 100 common first
names (James, Mary, Robert, Patricia, etc.) and 100 common last names
(Smith, Johnson, Williams, Brown, etc.). Each combination is checked
against a "used prefixes" set to ensure uniqueness.

If a collision occurs (e.g., two random picks both produce
"johnsmith"), a numeric suffix is added: `johnsmith02`, `johnsmith03`,
etc.

### Important rules

- Every display name must be unique in Exchange Online. The variations
  system ensures this.
- Email prefixes are alias-safe: only lowercase letters and digits,
  maximum 20 characters.
- The system never creates middle names -- only rearrangements and
  truncations of the provided first and last name.

---

## Common Failures and Troubleshooting

### "No Cloudflare zone found"

**What it means:** The Cloudflare API returned no zones matching your
domain name.

**Why it happens:**
- The domain is not added to the Cloudflare account whose credentials
  are stored in our system.
- The Cloudflare credentials (email + API key) are for a different
  account.
- The domain's nameservers are not pointing to Cloudflare yet.

**How to fix:**
1. Log into the correct Cloudflare account.
2. Verify the domain is listed under "Websites."
3. Check that the stored Cloudflare credentials match the account
   where the domain lives.
4. If using multiple Cloudflare accounts, ensure the correct
   `cf_config_id` is specified in the job config.

---

### "No Exchange license SKU"

**What it means:** No available license with Exchange Online
capabilities was found in the tenant.

**Why it happens:**
- The tenant only has a free or trial plan without Exchange.
- All license seats are consumed.
- The license name does not match any of the expected keywords.

**Is it a problem?** Usually NOT. This step is non-blocking, and
the license may already be pre-assigned to the admin account by
another administrator. Mailbox creation will still work if any
Exchange-capable license is active in the tenant.

**How to fix (if mailbox creation later fails):**
1. Go to Microsoft 365 Admin Center -> Billing -> Purchase services.
2. Buy an Exchange Online Plan 1 or Microsoft 365 Business Basic license.
3. Assign it to the admin account.
4. Rerun the pipeline.

---

### "SPF SOFTFAIL"

**What it means:** An email from your domain was checked against your
SPF record and the sending server was not explicitly authorized, but
you have not set a strict policy (`~all` instead of `-all`).

**Why it happens:**
- SPF is in "transitioning" mode (using `~all` as we configured).
- The email may have been forwarded through a server not listed in
  your SPF record.
- DNS propagation is still in progress (new SPF records can take
  up to 24 hours to propagate globally).

**How to fix:**
- If you just set up SPF, wait 24 hours for full DNS propagation.
- Check email headers: look for the `Received-SPF` header to see
  which server was flagged.
- Once confident, you can tighten SPF to `-all` (hardfail) by
  updating the DNS record.

---

### "DMARC FAIL"

**What it means:** An email from your domain failed the DMARC check.

**Why it happens:**
- **Duplicate DMARC records** -- having more than one DMARC TXT
  record at `_dmarc.yourdomain.com` causes DMARC to fail entirely.
  The pipeline auto-cleans duplicates, but they could be recreated
  manually.
- **DKIM not signed yet** -- DKIM signing requires DNS propagation
  (24-48 hours). If DKIM is not active and SPF also fails, DMARC
  rejects the email.
- **SPF misalignment** -- the From address domain does not match
  the envelope sender domain.

**How to fix:**
1. Check for duplicate records: look up `_dmarc.yourdomain.com` in
   Cloudflare DNS. There should be exactly ONE TXT record.
2. Check if DKIM is enabled: in the Microsoft 365 admin center, go
   to Security -> Email authentication -> DKIM.
3. Wait 24-48 hours after setting up DKIM before expecting DMARC
   to pass.

---

### "PROXY conflict"

**What it means:** The email alias you are trying to use is already
taken by another object in Exchange Online (a mailbox, group,
distribution list, etc.).

**Why it happens:** Either a previous pipeline run created a mailbox
with that alias, or someone manually created a conflicting object.

**How it is resolved:** The pipeline automatically handles this. When
a proxy conflict occurs, it retries with a random 2-digit number
appended to the alias. For example, `johnsmith@domain.com` becomes
`johnsmith42@domain.com`. The original requested email and the
actual email used are both logged so you can track what happened.

---

### Mailbox creation timeout

**What it means:** The PowerShell session timed out before all
mailboxes in the batch were created.

**Why it happens:**
- The batch is too large (each mailbox takes ~15-18 seconds).
- Exchange Online is slow due to high load.
- Network issues between our server and Microsoft's servers.

**How to fix:**
1. Set a smaller `batch_size` in the job config (default is 25).
2. Rerun the pipeline -- it will skip already-created mailboxes
   and continue from where it left off.

---

### "5.7.139 SMTP blocked"

**What it means:** Microsoft is rejecting SMTP connections for this
mailbox.

**Why it happens:**
- The SMTP permissions have not propagated yet. After enabling SMTP
  via `Set-CASMailbox`, it can take up to 24 hours for Microsoft's
  systems to fully activate it.
- The organization-level SMTP switch is still disabled.
- The authentication policy has not taken effect yet.

**How to fix:**
1. Wait 1-4 hours and try again.
2. Verify org-level SMTP is enabled:
   ```powershell
   Get-TransportConfig | Select SmtpClientAuthenticationDisabled
   # Should be: False
   ```
3. Verify mailbox-level SMTP is enabled:
   ```powershell
   Get-CASMailbox -Identity user@domain.com |
     Select SmtpClientAuthenticationDisabled
   # Should be: False
   ```
4. Check the authentication policy:
   ```powershell
   Get-AuthenticationPolicy | Select Name, AllowBasicAuthSmtp
   ```

---

### Domain readiness probe fails

**What it means:** After the domain was verified in Microsoft 365,
Exchange Online has not yet provisioned it for mailbox operations.

**Why it happens:** There is a delay between the Graph API confirming
domain verification and Exchange Online actually being ready to
create mailboxes on that domain. This delay is typically 1-5 minutes
but can be longer for new tenants.

**How it is handled:** The pipeline automatically retries the probe
up to 5 times with increasing delays (15s, 30s, 60s, 60s, 60s).
If all probes fail, it logs a warning but continues. If mailbox
creation then fails, the entire pipeline is re-queued to try again
in 10 minutes.

---

## Manual Verification Checklist

After the pipeline completes (status: "complete" or "warning"), run
through this checklist to confirm everything is working.

### 1. Check Cloudflare DNS Records

Log into Cloudflare and navigate to your domain's DNS settings. Verify:

- [ ] **MX record** exists pointing to
      `yourdomain-com.mail.protection.outlook.com`
- [ ] **SPF TXT record** exists with value
      `v=spf1 include:spf.protection.outlook.com ~all`
      (and there is only ONE SPF record)
- [ ] **DMARC TXT record** exists at `_dmarc.yourdomain.com`
      with `v=DMARC1; p=reject...`
      (and there is only ONE DMARC record)
- [ ] **Autodiscover CNAME** exists pointing to
      `autodiscover.outlook.com`
- [ ] **DKIM selector1 CNAME** exists at
      `selector1._domainkey.yourdomain.com`
- [ ] **DKIM selector2 CNAME** exists at
      `selector2._domainkey.yourdomain.com`
- [ ] **MS verification TXT** exists (starts with `MS=`)

### 2. Check Exchange Admin Center for Mailboxes

Go to `https://admin.exchange.microsoft.com`:

- [ ] Navigate to Recipients -> Resources.
- [ ] Your room mailboxes should be listed.
- [ ] Click on a mailbox and verify the display name and email
      address are correct.
- [ ] Check that the mailbox type is "Room."

### 3. Send a Test Email

- [ ] From an external email account (like Gmail), send an email
      to one of the created mailbox addresses.
- [ ] Verify the email arrives (check in Exchange Admin Center or
      use the system's "Send Test Email" feature).
- [ ] If the email bounces, check the bounce message for SPF/DKIM/
      DMARC failure details.

### 4. Check Email Headers for SPF/DKIM/DMARC

When you receive a test email FROM one of the created mailboxes:

- [ ] Open the email headers (in Gmail: three dots -> Show original).
- [ ] Look for `Received-SPF: Pass` -- confirms SPF is working.
- [ ] Look for `DKIM-Signature:` header -- confirms DKIM is signing.
- [ ] Look for `Authentication-Results:` header and verify:
  - `spf=pass`
  - `dkim=pass` (may show `dkim=none` if DKIM signing is not yet
    enabled -- this is expected in the first 24-48 hours)
  - `dmarc=pass`

### 5. Verify SMTP Connectivity (Advanced)

If you have access to PowerShell or a programming environment:

```python
# Using the system's built-in SMTP health check:
from services.smtp_oauth_service import smtp_oauth_check

result = smtp_oauth_check(
    email="mailbox@yourdomain.com",
    tenant_id="your-tenant-id",
    client_id="your-client-id",
    client_secret="your-client-secret",
)
print(result)
# Expected: {"status": "healthy", "message": "SMTP OAuth2 login OK"}
```

Or manually via telnet/openssl:

```bash
openssl s_client -starttls smtp -connect smtp.office365.com:587
EHLO test
# Should see: 250-AUTH LOGIN XOAUTH2
```

---

## Appendix: Pipeline Flow Summary

```
  START
    |
    v
  [Step 1] Assign Exchange License  (non-blocking)
    |
    v
  [Step 2] Add Domain + DNS         (BLOCKING)
    |        - TXT verification
    |        - MX record
    |        - SPF record
    |        - Autodiscover CNAME
    |        - DKIM selector1 CNAME
    |        - DKIM selector2 CNAME
    |        - Auto-clean duplicate SPF
    v
  [Step 3] Verify Domain            (BLOCKING)
    |        - Graph API verification with retry
    |        - 30s propagation wait
    |        - Domain readiness probe
    v
  [Step 4] Setup DMARC              (non-blocking)
    |        - _dmarc TXT record
    |        - Auto-clean duplicate DMARC
    v
  [Step 5] Create Mailboxes         (BLOCKING)
    |        - Generate name variations
    |        - Batch PowerShell creation (25 per batch)
    |        - Auto-resolve proxy conflicts
    |        - Grant FullAccess + SendAs permissions
    v
  [Step 6] Enable Org SMTP Auth     (non-blocking)
    |        - Set-TransportConfig
    |        - AllowBasicSMTP auth policy
    v
  [Step 7] Enable SMTP per Mailbox  (non-blocking)
    |        - Set-CASMailbox per mailbox
    |        - Disable other protocols
    v
  [Step 8] Disable Calendar         (non-blocking)
    |        - Set-CalendarProcessing None
    v
  COMPLETE
    |
    +-- All steps passed?     --> status: "complete"
    +-- Non-blocking failed?  --> status: "warning"
    +-- Blocking step failed? --> status: "failed"
```

**Blocking vs Non-blocking:**

- **Blocking steps** (2, 3, 5): If these fail, the pipeline stops
  immediately because later steps depend on them. For example, you
  cannot create mailboxes (Step 5) without a verified domain (Step 3).
- **Non-blocking steps** (1, 4, 6, 7, 8): If these fail, the pipeline
  logs a warning and continues. The mailboxes will still be created;
  you just might need to fix the failed step manually later.

**Smart Re-queue:**

If a blocking step fails with an authentication or propagation error
(common with new tenants), the pipeline does not give up. Instead, it
re-queues itself to run again in 10 minutes, giving Exchange Online
time to propagate permissions. This re-queue only happens if the tenant
was created within the last 2 hours.

**Idempotency:**

The pipeline is designed to be safely re-run. Steps that already
succeeded are skipped. Mailboxes that already exist are skipped. DNS
records that already exist are updated (not duplicated). This means
you can rerun the pipeline after a partial failure and it will pick
up where it left off.
