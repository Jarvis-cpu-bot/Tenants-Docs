# Tenant Setup Guide -- The Complete 13-Step Flow Explained

> **Audience:** Non-developers, operations staff, support engineers, and anyone who needs to understand what the tenant setup process does and how to troubleshoot it.
>
> **Last updated:** April 2026

---

## Table of Contents

1. [What is a Tenant?](#what-is-a-tenant)
2. [Key Concepts Before We Begin](#key-concepts-before-we-begin)
3. [The 13-Step Setup Flow](#the-13-step-setup-flow)
4. [Common Failures and Troubleshooting](#common-failures--troubleshooting)
5. [Manual Verification Checklist](#manual-verification-checklist)
6. [Glossary](#glossary)

---

## What is a Tenant?

### The Simple Explanation

Think of a **Microsoft 365 tenant** as a company's private apartment in a huge building called Microsoft Cloud. When a company signs up for Microsoft 365, they get their own "apartment" (tenant) with its own mailboxes, users, settings, and admin controls. No other company can see inside it.

For example, if "Acme Corp" signs up for Microsoft 365, they get a tenant. All of their employees' emails (john@acme.com, jane@acme.com) live inside that tenant. The tenant has one or more **admin accounts** that control everything -- like the building super who holds the master keys.

### What Our System Does

Our platform manages many tenants at once. We automatically configure each tenant so it can:

- Send and receive emails through our system
- Be monitored for health and deliverability
- Connect to third-party tools like Instantly (a cold email platform)

To do this, we need to "set up" each tenant -- which means logging in as the admin, creating an application inside that tenant, giving it the right permissions, and saving the credentials so we can access the tenant's resources programmatically (without a human logging in every time).

### The Three Credentials We Need to Start

When someone adds a new tenant to our system, they provide three things:

| Credential | What It Is | Analogy |
|---|---|---|
| **admin_email** | The Global Administrator email for the tenant (e.g., admin@acme.onmicrosoft.com) | The front door key to the apartment |
| **admin_password** | The password for that admin account | The combination to the deadbolt |
| **mfa_secret** | A secret key for Multi-Factor Authentication (if MFA is already set up) | The second lock -- a security code that changes every 30 seconds |

These are stored **encrypted** in our database (using Fernet encryption). We never store passwords in plain text.

---

## Key Concepts Before We Begin

Before diving into the 13 steps, here are some terms you will see repeatedly:

### Graph API

Microsoft's Graph API is like a phone line to Microsoft's cloud. Instead of a human logging into the Microsoft admin portal and clicking buttons, our code makes HTTP requests (basically, sends digital messages) to the Graph API to do the same things. For example:

- "Hey Graph API, what is this tenant's ID?" -- `GET /organization`
- "Hey Graph API, create a new app" -- `POST /applications`
- "Hey Graph API, give this app permission to send mail" -- `POST /appRoleAssignments`

### OAuth2 Token

To talk to the Graph API, you need a **token** -- think of it like a temporary visitor badge. You show your credentials, get a badge (token), and then that badge lets you in for a limited time (usually 1 hour). There are two kinds of tokens in our flow:

- **Delegated token** (Steps 1-9): Acts on behalf of the admin user. Like borrowing the admin's badge.
- **App token** (Steps 11-13): The application's own badge. It does not need a human to log in.

### Service Principal

When you create an "app registration" in Azure AD, it is like registering a robot employee. The **service principal** is that robot's identity inside the tenant. The app registration defines *what* the robot is; the service principal is *who* it is inside a specific tenant.

### Idempotent

Many steps in the setup flow are designed to be **idempotent** -- a fancy word that means "running it twice produces the same result as running it once." If a step fails halfway, you can re-run it without breaking anything. The code checks "does this thing already exist?" before trying to create it.

---

## The 13-Step Setup Flow

The setup is a pipeline -- each step must complete before the next one begins. If any step fails, the pipeline stops and records exactly which step failed and why. You can fix the problem and re-run the setup; it will skip all previously successful steps and resume from the failure point.

Here is a high-level picture:

```
[Step 1: Login] --> [Steps 2-9: Configure via Delegated Token]
    --> [Step 10: Save & Close Browser]
    --> [Steps 11-13: Verify & Clean Up via App Token]
    --> DONE
```

---

### Step 1: Login (Selenium Device Code Flow)

**What it does (simple):** Opens an invisible web browser, logs into the tenant's Microsoft account, handles MFA setup if needed, and obtains a temporary access pass (token) so the remaining steps can talk to Microsoft's APIs.

**What it does (technical):**

1. The system initiates an **OAuth2 Device Code Flow** by sending a POST request to `https://login.microsoftonline.com/{domain}/oauth2/v2.0/devicecode`. This returns a `device_code` and a `user_code`.
2. A headless Chrome browser (controlled by Selenium) navigates to `https://microsoft.com/devicelogin` and enters the `user_code`.
3. The browser then goes through the full Microsoft login sequence:
   - Enters the admin email address
   - Enters the admin password
   - If Microsoft forces a password change (common for new accounts), it enters the new password
   - If MFA is not yet set up, it goes through first-time MFA enrollment -- it clicks "I want to use a different authenticator app," then "Can't scan the QR code?" to extract the TOTP secret key
   - If MFA is already set up, it generates a 6-digit TOTP code from the saved secret and enters it
4. Meanwhile, a background thread polls `https://login.microsoftonline.com/{domain}/oauth2/v2.0/token` every 5 seconds, waiting for the user to complete authentication. Once the browser flow finishes, this polling succeeds and returns an `access_token`.

**Why we need this:** This is the "bootstrapping" step. Before we have an app registration with its own credentials, the only way to access the tenant is through the admin's personal login. The device code flow is used because it is the most reliable way to authenticate programmatically through Microsoft's various security checks (MFA, Conditional Access, etc.).

**What gets saved to database:**

| Field | Value |
|---|---|
| `mfa_secret_enc` | The TOTP secret key (encrypted), if it was extracted during first-time MFA enrollment |
| `admin_password_enc` | Updated to the new password, if a forced password change occurred |
| `new_password_enc` | Cleared after a successful password change |

**What can go wrong:**

- **"LoginError: Password input not found"** -- The Microsoft login page changed its HTML structure, or the page did not load in time. Check internet connectivity and whether the Chrome browser is working. Look at the screenshot saved in the `screenshots/` directory.
- **"MFASetupError: Could not extract TOTP secret"** -- During first-time MFA enrollment, the system could not find the "Can't scan the QR code?" button or the secret key text on the page. Microsoft may have changed their MFA enrollment UI. Manual MFA enrollment is required.
- **"LoginError: Both passwords were rejected"** -- Neither the original password nor the new password worked. The password may have been changed externally, or the account may be locked.
- **"DeviceCodeError: Token polling timed out"** -- The login flow in the browser took longer than 10 minutes. This can happen if MFA enrollment gets stuck or there is an unexpected pop-up.
- **"Account locked"** -- Microsoft locked the account after too many failed login attempts. The code intentionally does NOT auto-retry login failures to prevent this.

**How to fix manually if it fails:**

1. Check the screenshot in `backend/screenshots/` -- it shows what the browser saw when it failed.
2. Try logging into `https://portal.azure.com` manually with the admin credentials to verify they work.
3. If the password was changed externally, update the `admin_password_enc` in the database (or update it through the frontend).
4. If MFA enrollment failed, manually set up the Microsoft Authenticator app for the account, note the TOTP secret key, and save it to the `mfa_secret_enc` field (encrypted).
5. Re-trigger the setup -- it will resume from Step 1.

**Edge cases:**

- If the tenant account requires a forced password change, the system needs a `new_password` value to be provided. If none is set, the step will fail.
- Some tenants have Conditional Access policies that block device code flow authentication. In that case, the admin must create an exception for the Azure CLI client ID (`04b07795-8ddb-461a-bbee-02f9e1bf7b46`).
- The browser runs in non-headless mode (`headless=False`) because some Microsoft security checks block headless browsers.

---

### Step 2: Get Tenant Info

**What it does (simple):** Asks Microsoft "What is this tenant's unique ID?" and saves the answer.

**What it does (technical):**

Sends a `GET` request to `https://graph.microsoft.com/v1.0/organization` using the delegated token from Step 1. The response includes a list of organizations -- we take the first one and extract its `id` field, which is the **Azure AD Tenant ID** (a GUID like `a1b2c3d4-e5f6-...`).

**Why we need this:** The tenant ID is used in nearly every API call going forward. It is how Microsoft knows which "apartment" we are talking to. Without it, we cannot request app tokens, make API calls, or do anything else.

**What gets saved to database:**

| Field | Value |
|---|---|
| `tenant_id_ms` | The Azure AD tenant ID (GUID string) |

**What can go wrong:**

- **"No Graph API token available"** -- Step 1 did not produce a token, or the token expired. Re-run from Step 1.
- **"Graph API returned no organizations"** -- Extremely rare. The token is valid but the tenant has no organization object. This could mean the tenant was deleted or is in a broken state.

**How to fix manually if it fails:**

1. Log into `https://portal.azure.com` with the admin account.
2. Go to **Azure Active Directory** > **Overview**.
3. Copy the **Tenant ID** shown on the page.
4. Manually update the `tenant_id_ms` field in the database.
5. Re-trigger setup -- it will skip this step since `tenant_id_ms` is now set.

**Edge cases:**

- This step is idempotent: if `tenant_id_ms` is already set, it skips entirely.
- If resuming from a failed run where Step 1 already succeeded, the system recovers a token from saved credentials (if Steps 7+ also completed previously).

---

### Step 3: App Registration

**What it does (simple):** Creates a new "robot employee" (application) inside the tenant that our system will use to manage things going forward, instead of logging in as the admin every time.

**What it does (technical):**

1. First, checks if an application named `TenantDashboard-AutoApp` already exists by calling `GET /applications?$filter=displayName eq 'TenantDashboard-AutoApp'`.
2. If not found, creates one via `POST /applications` with the display name `TenantDashboard-AutoApp`.
3. Then checks if a **service principal** exists for this app. If not, creates one via `POST /servicePrincipals` with the app's `appId`.

**Why we need this:** The admin's personal login is not suitable for long-term automated access. An app registration gives us a permanent, programmatic identity that can authenticate with its own credentials (client secret or certificate) without needing a human to log in.

Think of it like this: instead of having the building super (admin) come unlock the door every time, we create a robot with its own key card that can do specific tasks on its own.

**What gets saved to database:**

| Field | Value |
|---|---|
| `client_id` | The application's `appId` (Client ID) -- this is the "username" of the robot |

**What can go wrong:**

- **"No Graph API token available"** -- Token expired or missing. Re-run from Step 1.
- **"Insufficient privileges to complete the operation"** -- The admin account does not have Global Administrator role. Only Global Admins can create app registrations via the API.

**How to fix manually if it fails:**

1. Log into `https://portal.azure.com`.
2. Go to **Azure Active Directory** > **App registrations** > **New registration**.
3. Name it `TenantDashboard-AutoApp`, select "Accounts in this organizational directory only."
4. Copy the **Application (client) ID** and update the `client_id` field in the database.
5. Under **Enterprise applications**, verify the service principal was created automatically.

**Edge cases:**

- This step is idempotent: if the app already exists, it reuses it.
- The service principal is created separately from the app registration. The app registration defines the app; the service principal is its identity within the tenant.

---

### Step 4: Disable Security Defaults

**What it does (simple):** Turns off Microsoft's built-in security features that would block our automated access, such as mandatory MFA for all users and blocking of legacy authentication methods.

**What it does (technical):**

1. Sends `PATCH /policies/identitySecurityDefaultsEnforcementPolicy` with `{"isEnabled": false}` to disable Azure AD Security Defaults.
2. Sends `PATCH /policies/authenticationMethodsPolicy` with `{"registrationCampaign": {"state": "disabled"}}` to stop Microsoft from nagging users to register for MFA.
3. Sends another `PATCH /policies/authenticationMethodsPolicy` with `{"systemCredentialPreferences": {"state": "disabled"}}` to turn off system-preferred MFA (where Microsoft auto-selects the MFA method).

**Why we need this:** Security Defaults enforce MFA for all users and block certain authentication patterns. Since our system needs to create and manage mailboxes with specific MFA configurations, we need to disable the blanket policy and manage security on a per-user basis instead. Additionally, the MFA registration campaign would prompt every new mailbox user to set up MFA, which interferes with our automated mailbox setup.

**What gets saved to database:** Nothing -- this is a configuration change on Microsoft's side.

**What can go wrong:**

- **"403 Forbidden"** -- The delegated token from Step 1 may not have sufficient permissions to modify security policies. This is treated as **non-blocking** -- the step logs a warning and continues. The security defaults will be retried later in Step 11 with app credentials.
- **"Authorization_RequestDenied"** -- The admin account might not be a Global Administrator, or Privileged Identity Management (PIM) requires the role to be activated first.

**How to fix manually if it fails:**

1. Log into `https://portal.azure.com`.
2. Go to **Azure Active Directory** > **Properties** > **Manage Security Defaults**.
3. Set "Security defaults" to **Disabled**.
4. Go to **Security** > **Authentication methods** > **Settings**.
5. Disable the "Registration campaign" and "System-preferred MFA."

**Edge cases:**

- This step is **non-blocking for 403 errors**. If the delegated token cannot disable security defaults, the pipeline continues. Step 11 retries with the app's own token (which has stronger permissions after admin consent).
- Some tenants have Conditional Access policies instead of Security Defaults. In that case, Security Defaults are already disabled and this step is a no-op.

---

### Step 5: Assign Permissions

**What it does (simple):** Gives the robot (app registration) a list of specific abilities -- like permission to read emails, manage users, handle domains, and connect to Exchange Online.

**What it does (technical):**

1. Builds a `requiredResourceAccess` payload containing **13 Microsoft Graph permissions** and **2 Exchange Online permissions**:

   **Microsoft Graph permissions (all Application type -- meaning the app acts on its own, not on behalf of a user):**
   - `Application.ReadWrite.All` -- manage app registrations
   - `Directory.ReadWrite.All` -- read and write directory data
   - `Mail.Send` -- send emails
   - `Mail.ReadWrite` -- read and write mailbox contents
   - `User.ReadWrite.All` -- manage user accounts
   - `Organization.Read.All` -- read organization info
   - `Organization.ReadWrite.All` -- modify organization settings
   - `Domain.ReadWrite.All` -- manage custom domains
   - `Policy.ReadWrite.ConditionalAccess` -- manage Conditional Access policies
   - `Policy.ReadWrite.AuthenticationMethod` -- manage MFA and auth methods
   - `Policy.Read.All` -- read all policies
   - `UserAuthenticationMethod.ReadWrite.All` -- manage users' MFA methods
   - `RoleManagement.ReadWrite.Directory` -- assign directory roles

   **Exchange Online permissions:**
   - `Exchange.ManageAsApp` -- connect to Exchange Online PowerShell as the app
   - `SMTP.SendAsApp` -- send emails via SMTP OAuth2 (required since Microsoft deprecated basic auth)

2. Sends `PATCH /applications/{app_object_id}` with the permissions payload.

3. Assigns the **Exchange Administrator** directory role to the service principal using `POST /roleManagement/directory/roleAssignments`. This gives the app admin-level access to Exchange Online.

**Why we need this:** Permissions define what the robot is *allowed* to do. Without them, every API call would return "Access Denied." Think of it like giving your robot employee access to specific rooms in the building -- one key card for the mail room, another for the user directory, another for the domain registry.

**What gets saved to database:** Nothing directly -- the permissions are stored on the app registration in Azure AD.

**What can go wrong:**

- **"Insufficient privileges"** -- The token does not have permission to modify the app registration. This usually means the delegated token has expired.
- **"Role assignment already exists"** -- The Exchange Admin role was already assigned. This is handled gracefully (logged and skipped).
- **"Exchange Online service principal not found"** -- In brand-new tenants, the Exchange Online service principal may take a few minutes to appear. The code retries up to 5 times with 10-second intervals.

**How to fix manually if it fails:**

1. Log into `https://portal.azure.com`.
2. Go to **Azure Active Directory** > **App registrations** > **TenantDashboard-AutoApp** > **API permissions**.
3. Click **Add a permission** and add each of the 13 Graph permissions listed above (select "Application permissions," not "Delegated").
4. Also add Exchange Online (`Office 365 Exchange Online`) > Application permissions > `Exchange.ManageAsApp` and `SMTP.SendAsApp`.
5. Go to **Roles and administrators** > **Exchange Administrator** > **Add assignments** > add the `TenantDashboard-AutoApp` service principal.

**Edge cases:**

- If Exchange Online is not yet provisioned in the tenant (e.g., no Exchange license), the Exchange SP may not exist at all. The retry logic handles temporary delays, but if Exchange was never provisioned, this step will fail after 5 attempts.

---

### Step 6: Admin Consent

**What it does (simple):** Officially approves all the permissions from Step 5. Step 5 *declared* the permissions; Step 6 *grants* them. It is like the difference between requesting access to a room and actually getting the key.

**What it does (technical):**

1. Finds the **Microsoft Graph service principal** in the tenant by looking up the well-known app ID `00000003-0000-0000-c000-000000000000`.
2. For each of the 13 Graph permissions, sends `POST /servicePrincipals/{sp_id}/appRoleAssignments` to grant the permission. This creates an **app role assignment** linking the app's service principal to the Graph service principal for that specific permission.
3. Finds the **Exchange Online service principal** (app ID `00000002-0000-0ff1-ce00-000000000000`) -- retries up to 5 times with 10-second backoff if it is not immediately available.
4. Grants `Exchange.ManageAsApp` and `SMTP.SendAsApp` the same way.

**Why we need this:** In Microsoft's security model, declaring permissions (Step 5) only puts them on the app's "wishlist." An admin must explicitly **consent** to those permissions. In a typical scenario, an admin would click a "Grant admin consent" button in the Azure portal. Our code does this programmatically via the API instead.

**What gets saved to database:** Nothing -- the consent is stored in Azure AD.

**What can go wrong:**

- **"Permission entry already exists"** -- The permission was already granted (from a previous run). This is handled gracefully and skipped.
- **"Microsoft Graph service principal not found"** -- Should never happen, but if it does, the tenant is in a severely broken state.
- **"Exchange Online SP not found after 5 attempts"** -- The Exchange Online service principal does not exist in the tenant. This means Exchange is not provisioned. You need to assign an Exchange Online license to at least one user first.

**How to fix manually if it fails:**

1. Log into `https://portal.azure.com`.
2. Go to **Azure Active Directory** > **App registrations** > **TenantDashboard-AutoApp** > **API permissions**.
3. Click **Grant admin consent for [Tenant Name]**.
4. Confirm the consent dialog.
5. Verify the "Status" column shows green checkmarks for all permissions.

**Edge cases:**

- Even after granting consent, it can take up to 60 seconds for the permissions to propagate through Microsoft's systems. This is why Step 11 has a retry loop with backoff.
- In rare cases, some permissions may fail to grant while others succeed. The code counts how many were granted vs. skipped and logs the results.

---

### Step 7: Create Client Secret

**What it does (simple):** Generates a secret password for the robot (app) so it can log in on its own without needing the admin to authenticate.

**What it does (technical):**

Sends `POST /applications/{app_object_id}/addPassword` with a label `auto-dashboard-secret`. Microsoft responds with a `secretText` (the actual secret value) plus metadata like `keyId` and `endDateTime`.

The `secretText` is then encrypted with Fernet and saved to the database immediately.

**Why we need this:** Up until now, every API call used the admin's personal token (obtained through the browser login in Step 1). That token expires in about 1 hour and cannot be refreshed without another browser login. The **client secret** is like giving the robot its own permanent password -- combined with the `client_id` and `tenant_id_ms`, it can obtain tokens independently, forever (until the secret expires, which is typically 2 years).

**What gets saved to database:**

| Field | Value |
|---|---|
| `client_secret_enc` | The client secret value (Fernet-encrypted) |

**What can go wrong:**

- **"Insufficient privileges"** -- The delegated token does not have permission to add credentials to the app. This usually means admin consent (Step 6) has not propagated yet.
- **Network timeout** -- The Graph API did not respond in time. Retry the step.

**How to fix manually if it fails:**

1. Log into `https://portal.azure.com`.
2. Go to **Azure Active Directory** > **App registrations** > **TenantDashboard-AutoApp** > **Certificates & secrets**.
3. Click **New client secret**, give it a description, set the expiry.
4. **Copy the secret value immediately** (it is only shown once).
5. Encrypt the value with Fernet and update `client_secret_enc` in the database.

**Edge cases:**

- The secret value is **only returned once** -- in the response to the `addPassword` call. If it is lost (e.g., the process crashes right after the API call but before saving to DB), a new secret must be generated.
- The code saves the secret to the database **immediately** after creation, before proceeding to Step 8. This minimizes the risk of losing the secret.
- The plain-text secret is also kept in the in-memory `context` dictionary for use in Step 11.

---

### Step 8: Generate Certificate

**What it does (simple):** Creates a digital security certificate for the robot -- like a more secure version of the client secret. This certificate is used specifically for connecting to Exchange Online (Microsoft's email server management system).

**What it does (technical):**

1. Generates an **RSA 2048-bit private key** using the Python `cryptography` library.
2. Creates a **self-signed X.509 certificate** with:
   - Common Name (CN): `TenantDashboard-{tenant_name}`
   - Validity: 2 years from now
   - Signed with SHA-256
3. Exports the private key + certificate as a **PKCS#12 (PFX) file** encrypted with a random 32-character password.
4. Computes the **SHA-1 thumbprint** of the certificate (used as an identifier).
5. Encrypts the PFX file, its password, and the thumbprint, then saves everything to the database.
6. Stashes the base64-encoded public certificate in memory for Step 9.

**Why we need this:** Exchange Online PowerShell requires **certificate-based authentication** for app-only access. A client secret alone is not enough for Exchange management commands. Think of the client secret as a regular key, and the certificate as a biometric scan -- it is a stronger form of authentication that Exchange demands.

**What gets saved to database:**

| Field | Value |
|---|---|
| `cert_pfx_enc` | The PKCS#12 (PFX) file containing the private key and certificate (Fernet-encrypted, base64-encoded) |
| `cert_password_enc` | The password protecting the PFX file (Fernet-encrypted) |
| `cert_thumbprint` | The SHA-1 fingerprint of the certificate (plain text -- not a secret) |
| `cert_expires_at` | When the certificate expires (2 years from creation) |

**What can go wrong:**

- This step is entirely local (no API calls), so failures are extremely rare.
- **Out of memory** -- Generating the RSA key requires some memory. Should not be an issue on any modern system.
- **Cryptography library not installed** -- The `cryptography` Python package must be installed.

**How to fix manually if it fails:**

1. Use OpenSSL on the command line:
   ```
   openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 730 -nodes -subj "/CN=TenantDashboard-YourTenantName"
   openssl pkcs12 -export -out cert.pfx -inkey key.pem -in cert.pem
   ```
2. Base64-encode the PFX, encrypt it, and save to the `cert_pfx_enc` field.
3. Save the PFX password (encrypted) to `cert_password_enc`.
4. Get the thumbprint: `openssl x509 -in cert.pem -fingerprint -noout`

**Edge cases:**

- The certificate is self-signed, which is fine for Azure AD app credentials (they do not need a CA-signed certificate).
- The 2-year expiry means someone needs to regenerate and re-upload the certificate before it expires, or Exchange Online authentication will start failing.

---

### Step 9: Upload Certificate

**What it does (simple):** Uploads the public part of the certificate (from Step 8) to the app registration in Azure AD, plus any local EXO certificate, so Microsoft knows to trust it when the app authenticates.

**What it does (technical):**

1. Calls `upload_certificate()` which:
   - First reads the app's existing `keyCredentials` via `GET /applications/{app_object_id}` to avoid overwriting any previously uploaded certificates.
   - Strips read-only fields (`displayName`, `customKeyIdentifier`) that Microsoft rejects on PATCH.
   - Appends the new certificate as type `AsymmetricX509Cert` with usage `Verify`.
   - Sends `PATCH /applications/{app_object_id}` with the updated `keyCredentials` array.

2. If an `EXO_CERT_THUMBPRINT` environment variable is set (pointing to a local certificate used by the PowerShell service), it also:
   - Uses PowerShell (`pwsh`) to export the certificate's public key from the Windows certificate store.
   - Appends that certificate to the app's `keyCredentials` as well.

3. Verifies the upload by re-reading the app and checking that `keyCredentials` is not empty.

**Why we need this:** Think of this like registering the robot's fingerprint at the security desk. Step 8 created the fingerprint (certificate); Step 9 tells Microsoft "this is a valid fingerprint for this robot." Without this, the robot will show its certificate and Microsoft will say "I don't recognize this."

The EXO cert upload is needed because our PowerShell service uses a specific local certificate to connect to Exchange Online. That cert also needs to be registered on the app.

**What gets saved to database:** Nothing new -- the certificate is stored on the app registration in Azure AD.

**What can go wrong:**

- **"KeyNotUpdatable"** -- Azure AD rejected the PATCH because the `keyCredentials` payload included read-only fields. The code sanitizes these, but if Microsoft adds new read-only fields, this could break.
- **"Invalid key"** -- The base64-encoded certificate is malformed or empty. Re-run from Step 8.
- **EXO cert not found** -- The local EXO certificate is not in any Windows certificate store. This is logged as a warning but does not block the step.
- **Propagation delay** -- After uploading, the re-read may show an empty `keyCredentials` list. This is normal and usually resolves within a few seconds.

**How to fix manually if it fails:**

1. Log into `https://portal.azure.com`.
2. Go to **Azure Active Directory** > **App registrations** > **TenantDashboard-AutoApp** > **Certificates & secrets** > **Certificates** tab.
3. Click **Upload certificate** and upload the `.cer` file (public key only).
4. Verify the thumbprint matches what is in the database.

**Edge cases:**

- The code preserves existing certificates on the app. If there is already a certificate from a previous run, it will not be overwritten.
- The EXO cert upload step uses `pwsh` (PowerShell Core), not `powershell` (Windows PowerShell). If `pwsh` is not installed, this sub-step will fail silently.

---

### Step 10: Save Credentials (Validate and Close Browser)

**What it does (simple):** Double-checks that all important credentials have been saved to the database, then closes the web browser that was opened in Step 1.

**What it does (technical):**

1. Validates that `client_id` exists (set in Step 3).
2. Validates that `client_secret_enc` exists (set in Step 7).
3. Sets the tenant status to `running`.
4. Closes the Selenium Chrome browser and removes it from the in-memory context.

**Why we need this:** This is a checkpoint step. After Steps 1 through 9, we have everything we need to operate without a browser. The delegated token (which was obtained through the browser login) will expire soon, but we now have a client secret and certificate that can get new tokens independently. Closing the browser frees resources and ensures no stale browser sessions remain.

Think of it like a relay race -- the browser carried the baton for Steps 1-9, and now it hands off to the API for Steps 11-13.

**What gets saved to database:**

| Field | Value |
|---|---|
| `status` | Updated to `running` |

**What can go wrong:**

- **"client_id missing after Step 3"** -- Step 3 failed or its result was not saved. The pipeline must be re-run from Step 3.
- **"client_secret_enc missing after Step 7"** -- Step 7 failed or its result was not saved. The pipeline must be re-run from Step 7.

**How to fix manually if it fails:**

1. Check the database for the `client_id` and `client_secret_enc` fields.
2. If missing, re-run the setup from the appropriate step (3 or 7).
3. If the browser is stuck, kill the Chrome processes manually.

**Edge cases:**

- If the browser was already closed (e.g., due to a crash in a previous step), the close operation is a no-op.

---

### Step 11: Verify API Mode (Switch from Delegated to App Token)

**What it does (simple):** Tests that the robot can log in on its own (without the admin), registers it with Exchange Online, and retries disabling security defaults with the robot's stronger permissions.

**What it does (technical):**

This is the most complex step. It does several things:

1. **Obtain an app token:** Calls the OAuth2 token endpoint with `client_credentials` grant using the `tenant_id_ms`, `client_id`, and `client_secret`. This proves the app can authenticate independently.

2. **Verify the token works:** Calls `GET /organization` with the new app token. If this succeeds, it means the client credentials are valid and the permissions have propagated.

3. **Retry with backoff:** Azure AD can take up to 60 seconds (sometimes longer) to propagate a new client secret and admin consent. The code retries 6 times with increasing delays: 10s, 20s, 30s, 40s, 50s, 60s. That is a total of 210 seconds (3.5 minutes) of waiting.

4. **Verify service principal:** Checks that the service principal still exists (it should, from Step 3). If it somehow disappeared, re-creates it.

5. **Register in Exchange Online:** Runs a PowerShell command via the PowerShell service:
   ```
   New-ServicePrincipal -AppId '<client_id>' -ObjectId '<sp_object_id>'
   ```
   This registers the app in Exchange Online so it can use certificate-based auth with `Connect-ExchangeOnline`.

6. **Retry security settings:** Now that we have an app token with full permissions (instead of the delegated token from Step 1), retries disabling security defaults and MFA policies. The app token usually has broader permissions because it directly holds the `Policy.ReadWrite.AuthenticationMethod` permission.

**Why we need this:** This is the critical "handoff" step. Everything before this used the admin's personal token (temporary, expires in 1 hour). Everything after this uses the app's own token (renewable, permanent). This step verifies the handoff worked. The Exchange Online registration is required because Exchange has its own internal directory of allowed service principals -- even though the app has the right permissions in Azure AD, Exchange will not recognize it until it is explicitly registered.

**What gets saved to database:**

| Field | Value |
|---|---|
| `tenant_id_ms` | Backfilled if it was not set in Step 2 (discovered from the email domain) |

**What can go wrong:**

- **"Token request failed -- HTTP 401"** -- The client secret is invalid or has not propagated yet. Usually resolves with the retry loop. If it fails after all 6 attempts, the secret may be wrong.
- **"Graph API GET /organization failed -- HTTP 403"** -- The admin consent from Step 6 has not propagated. Wait and retry.
- **"Could not register SP in Exchange Online"** -- The PowerShell service failed to connect to Exchange Online. This is **non-blocking** -- the SP may already be registered, or EXO may not be ready yet.
- **"API mode verification failed after 6 attempts"** -- Something is fundamentally wrong with the credentials or permissions. Check all previous steps.

**How to fix manually if it fails:**

1. Test the credentials manually:
   ```
   curl -X POST https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token \
     -d "grant_type=client_credentials&client_id={client_id}&client_secret={secret}&scope=https://graph.microsoft.com/.default"
   ```
2. If the token request fails, the client secret may be wrong. Generate a new one in Azure Portal.
3. If the token works but `/organization` returns 403, admin consent is not granted. Go to App registrations > API permissions > Grant admin consent.
4. For Exchange SP registration, use Exchange Online PowerShell manually:
   ```powershell
   Connect-ExchangeOnline -AppId <client_id> -CertificateThumbprint <thumbprint> -Organization <domain>
   New-ServicePrincipal -AppId <client_id> -ObjectId <sp_object_id>
   ```

**Edge cases:**

- If `tenant_id_ms` was not set in Step 2 (e.g., Step 2 was skipped due to an error), this step can discover it from the email domain. It uses the domain part of `admin_email` as the tenant hint.
- The Exchange SP registration may fail with "service principal already exists" -- this is handled gracefully.
- Security defaults retry may fail again with 403 -- this is non-blocking and just logged.

---

### Step 12: Grant Instantly Consent

**What it does (simple):** Gives a third-party email tool called Instantly permission to read and send emails on behalf of users in this tenant.

**What it does (technical):**

1. Finds or creates the **Instantly service principal** in the tenant. Instantly's well-known app ID is `65ad96b6-fbeb-40b5-b404-2a415d074c97`. If its service principal does not exist in the tenant, the code creates it via `POST /servicePrincipals`.

2. Finds the **Microsoft Graph service principal** (app ID `00000003-0000-0000-c000-000000000000`).

3. Grants OAuth consent via `POST /oauth2PermissionGrants`:
   - Client: Instantly service principal
   - Consent type: `AllPrincipals` (applies to all users in the tenant)
   - Scopes: `offline_access Mail.ReadWrite Mail.Send`

**Why we need this:** Instantly is a cold email platform that connects to Microsoft 365 mailboxes to send and manage emails. For Instantly to access a tenant's mailboxes, the tenant admin must grant it OAuth consent. Instead of requiring a manual consent flow (where someone clicks through a browser dialog), our system grants this consent programmatically using the app's token.

**What gets saved to database:** Nothing -- the consent is stored in Azure AD.

**What can go wrong:**

- **"Microsoft Graph service principal not found"** -- Should never happen. The Graph SP exists in every tenant.
- **"Consent grant already exists"** -- If Instantly consent was already granted (e.g., from a previous run or manual setup). The Graph API may return a conflict error. This is **non-blocking**.
- **"Insufficient privileges"** -- The app token does not have permission to grant consent. This means the admin consent from Step 6 did not include the right permissions.

**How to fix manually if it fails:**

1. Open a browser and navigate to:
   ```
   https://login.microsoftonline.com/{tenant_id}/adminconsent?client_id=65ad96b6-fbeb-40b5-b404-2a415d074c97
   ```
2. Log in as the tenant admin and click "Accept."
3. Alternatively, go to **Azure AD** > **Enterprise applications** > search for "Instantly" > **Permissions** > **Grant admin consent**.

**Edge cases:**

- This is a **non-blocking step** -- if it fails, the pipeline continues to Step 13. The Instantly consent can be granted later manually.
- The consent is for `AllPrincipals`, meaning Instantly can access any mailbox in the tenant. This is by design for our use case, but it is a broad permission.

---

### Step 13: Delete MFA Authenticator

**What it does (simple):** Removes the MFA authenticator app that was set up for the admin account during Step 1. Now that the robot has its own credentials, we do not need the admin's MFA anymore.

**What it does (technical):**

1. Calls `GET /users/{admin_email}/authentication/microsoftAuthenticatorMethods` to list all registered Microsoft Authenticator methods for the admin user.
2. For each method found, calls `DELETE /users/{admin_email}/authentication/microsoftAuthenticatorMethods/{method_id}` to remove it.
3. Logs how many were deleted and how many failed.

**Why we need this:** During Step 1, the system enrolled a Microsoft Authenticator app (by extracting the TOTP secret key) to get past MFA challenges. Now that setup is complete and the app has its own client credentials, the authenticator is no longer needed. Leaving it enrolled would mean:

- The TOTP secret is stored in our database -- an unnecessary security risk if we do not need it.
- If someone else tries to log into the admin account, the MFA enrollment could interfere.

Cleaning it up keeps the account in a known state.

**What gets saved to database:** Nothing directly. However, after this step, the `mfa_secret_enc` stored in the database is no longer usable (since the authenticator method was deleted).

**What can go wrong:**

- **"Failed to delete authenticator method"** -- The Graph API returned an error. This could be a permissions issue (`UserAuthenticationMethod.ReadWrite.All` not granted) or a transient error.
- **"No Graph token available"** -- The token from Step 11 is missing. Since this is the last step, this should not happen unless Step 11 failed and was manually marked as successful.

**How to fix manually if it fails:**

1. Log into `https://mysignins.microsoft.com/security-info` with the admin account.
2. Find the "Microsoft Authenticator" entry.
3. Click **Delete** to remove it.
4. Alternatively, use the Graph API Explorer at `https://developer.microsoft.com/en-us/graph/graph-explorer` to call the DELETE endpoint.

**Edge cases:**

- This is a **non-blocking step** -- if it fails, the pipeline still completes successfully. The authenticator is a cleanup concern, not a functional requirement.
- If the admin has multiple authenticator methods (e.g., from manual setup), all of them are deleted.
- If no authenticator methods exist (e.g., MFA was not enrolled during Step 1), the step finishes silently.

---

## Common Failures & Troubleshooting

### "403 AccessDenied" on Step 4 (Disable Security Defaults)

**Why it happens:** The delegated token from Step 1 does not include the `Policy.ReadWrite.ConditionalAccess` scope. This is common because the device code flow uses the Azure CLI's client ID, which may not have consent for all policy scopes.

**What to do:**
1. Do nothing -- this is non-blocking. Step 11 will retry with the app's own token.
2. If Step 11 also fails to disable security defaults, do it manually in the Azure portal (see Step 4 manual fix).

---

### "Permission not found" on Step 6 (Admin Consent)

**Why it happens:** The permission ID in the code does not match what Microsoft expects. This can happen if Microsoft changes or deprecates a permission. It can also occur if the Graph or Exchange service principal is not yet available in a brand-new tenant.

**What to do:**
1. Check the Azure portal -- go to App registrations > TenantDashboard-AutoApp > API permissions to see which permissions are listed.
2. Manually grant admin consent in the Azure portal.
3. Verify the permission GUIDs are still correct by checking [Microsoft's permission reference](https://learn.microsoft.com/en-us/graph/permissions-reference).

---

### Certificate Upload "KeyNotUpdatable" on Step 9

**Why it happens:** The `PATCH /applications/{id}` call included read-only fields in the `keyCredentials` array. Microsoft is strict about which fields can be sent back. The code strips `displayName` and `customKeyIdentifier`, but if Microsoft adds new read-only fields, this error will appear.

**What to do:**
1. Upload the certificate manually through the Azure portal (see Step 9 manual fix).
2. Report the issue so the code can be updated to strip the new read-only field.

---

### Device Code Flow Blocked (Step 1)

**Why it happens:** The tenant has a **Conditional Access policy** that blocks the device code flow. Some organizations do this for security reasons.

**What to do:**
1. Log into the Azure portal as the tenant admin.
2. Go to **Security** > **Conditional Access** > **Policies**.
3. Create an exception that allows the Azure CLI client ID (`04b07795-8ddb-461a-bbee-02f9e1bf7b46`) to authenticate via device code flow.
4. Alternatively, temporarily disable the blocking Conditional Access policy, run the setup, and re-enable it afterward.

---

### MFA Enrollment Stuck (Step 1)

**Why it happens:** Microsoft changed the MFA enrollment UI, and the Selenium selectors no longer match the page elements. Or the page is loading too slowly.

**What to do:**
1. Check the screenshot in `backend/screenshots/` for visual context.
2. Manually log into the admin account at `https://portal.azure.com`.
3. Complete MFA enrollment manually.
4. Record the TOTP secret key and save it (encrypted) to the `mfa_secret_enc` database field.
5. Re-trigger the setup.

---

### Setup Succeeds but Exchange Online Commands Fail Later

**Why it happens:** Step 11 registered the service principal in Exchange Online, but Exchange can take up to 24 hours to fully propagate. Alternatively, the EXO certificate was not uploaded in Step 9.

**What to do:**
1. Wait 24 hours and try again.
2. Verify the certificate is on the app registration in Azure portal.
3. Manually run `New-ServicePrincipal` in Exchange Online PowerShell if needed.

---

### Token Expired Mid-Pipeline

**Why it happens:** The delegated token from Step 1 typically lasts 1 hour. If the pipeline takes longer than that (due to retries, backoff, or slow network), later steps may fail with 401 errors.

**What to do:**
1. Simply re-trigger the setup. It will resume from the failed step.
2. If Steps 1-7 completed, the app has its own credentials, and the recovery logic (`_recover_context`) will obtain a fresh app token automatically.

---

## Manual Verification Checklist

After the setup pipeline completes (status = `complete`), verify everything works by checking these items:

### 1. Azure Portal -- App Registration

- [ ] Log into `https://portal.azure.com`
- [ ] Navigate to **Azure Active Directory** > **App registrations**
- [ ] Confirm `TenantDashboard-AutoApp` exists
- [ ] Check **API permissions** -- all 13 Graph + 2 Exchange permissions should have green checkmarks (admin consent granted)
- [ ] Check **Certificates & secrets** -- at least one client secret and one certificate should be listed

### 2. Azure Portal -- Enterprise Application

- [ ] Navigate to **Enterprise applications**
- [ ] Search for `TenantDashboard-AutoApp`
- [ ] Confirm a service principal exists
- [ ] Check **Permissions** -- should show all granted permissions

### 3. Azure Portal -- Roles

- [ ] Navigate to **Azure Active Directory** > **Roles and administrators**
- [ ] Search for **Exchange Administrator**
- [ ] Confirm `TenantDashboard-AutoApp` is listed as an assignee

### 4. Graph API Token Test

Test that the app can obtain a token independently:

```
POST https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={client_id_from_database}
&client_secret={decrypted_client_secret}
&scope=https://graph.microsoft.com/.default
```

Expected: a JSON response with `access_token`.

### 5. Graph API Organization Call

Using the token from step 4:

```
GET https://graph.microsoft.com/v1.0/organization
Authorization: Bearer {access_token}
```

Expected: a JSON response with the organization details.

### 6. Exchange Admin Center

- [ ] Navigate to `https://admin.exchange.microsoft.com`
- [ ] Log in as the tenant admin
- [ ] Confirm no errors or warnings about service principals

### 7. Security Defaults

- [ ] In Azure portal, go to **Azure Active Directory** > **Properties** > **Manage Security Defaults**
- [ ] Confirm Security Defaults are **Disabled**

### 8. Instantly Consent (if applicable)

- [ ] In Azure portal, go to **Enterprise applications** > search for "Instantly"
- [ ] Confirm the Instantly service principal exists
- [ ] Check **Permissions** -- should show `offline_access`, `Mail.ReadWrite`, `Mail.Send`

### 9. Database Verification

Check that all expected fields are populated in the Tenant record:

| Field | Should Be |
|---|---|
| `status` | `complete` |
| `tenant_id_ms` | A GUID (e.g., `a1b2c3d4-...`) |
| `client_id` | A GUID |
| `client_secret_enc` | Non-null binary data |
| `cert_pfx_enc` | Non-null binary data |
| `cert_password_enc` | Non-null binary data |
| `cert_thumbprint` | A 40-character hex string |
| `cert_expires_at` | A date ~2 years in the future |
| `mfa_secret_enc` | Non-null binary data (if MFA was enrolled during setup) |
| `step_results` | All 13 steps showing `{"status": "success"}` |
| `current_step` | `13` |

---

## Glossary

| Term | Meaning |
|---|---|
| **Azure AD / Entra ID** | Microsoft's cloud identity service. Manages users, apps, and permissions for Microsoft 365. Microsoft recently rebranded Azure AD to "Entra ID" but the terms are used interchangeably. |
| **App Registration** | A record in Azure AD that defines an application -- its name, permissions, credentials, etc. |
| **Service Principal** | The identity of an app within a specific tenant. Created automatically or manually when an app is used in a tenant. |
| **Client ID** | The unique identifier (GUID) of an app registration. Think of it as the app's username. |
| **Client Secret** | A password for the app. Used with the Client ID to obtain tokens. |
| **Certificate (X.509)** | A cryptographic credential, stronger than a client secret. Required by Exchange Online for app-only access. |
| **OAuth2** | The authorization protocol used by Microsoft. Defines how apps request and receive tokens. |
| **Delegated Token** | A token that acts on behalf of a specific user. Requires the user to log in. |
| **App Token** | A token for the app itself (client_credentials grant). Does not require a user to log in. |
| **Device Code Flow** | An OAuth2 flow where the user enters a code on a separate device (or browser) to authenticate. Useful when the app cannot show a browser directly. |
| **MFA** | Multi-Factor Authentication. Requires a second form of verification (e.g., a code from an authenticator app) in addition to the password. |
| **TOTP** | Time-based One-Time Password. A 6-digit code that changes every 30 seconds, generated from a secret key. Used for MFA. |
| **Fernet Encryption** | A symmetric encryption scheme used in our codebase to encrypt sensitive data before storing it in the database. |
| **Graph API** | Microsoft's REST API for interacting with Microsoft 365 services (users, mail, directory, policies, etc.). |
| **Exchange Online (EXO)** | Microsoft's cloud email service. Part of Microsoft 365. Managed through Exchange Admin Center or PowerShell. |
| **Security Defaults** | A set of pre-configured security settings in Azure AD that enforce MFA and block legacy auth. Must be disabled for our per-user configuration approach. |
| **Admin Consent** | The act of a Global Administrator approving an app's requested permissions for the entire tenant. |
| **PFX / PKCS#12** | A file format that bundles a private key and certificate together, protected by a password. |
| **Thumbprint** | A SHA-1 hash of a certificate, used as a short identifier. |
| **Celery** | A Python task queue used to run the setup steps asynchronously (in the background). |
| **Selenium** | A browser automation tool. Used in Step 1 to drive a Chrome browser through the Microsoft login flow. |
| **Instantly** | A third-party cold email platform that connects to Microsoft 365 mailboxes. |
| **Idempotent** | A property where running an operation multiple times produces the same result as running it once. Important for retry safety. |
| **Non-blocking step** | A step that logs a warning and continues the pipeline if it fails, rather than stopping the entire setup. |
