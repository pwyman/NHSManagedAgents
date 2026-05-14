# Guide 01: IMAP Email Setup

## Overview

The FOI email workflow polls the `foi@westshire.icb.nhs.uk` inbox every 2 minutes using the IMAP protocol. It reads unread messages, marks them as read, and passes them to the processing pipeline. This guide explains how to configure the IMAP connection for three common email providers: Gmail, Microsoft 365 / NHS.net, and any generic IMAP server.

**What the IMAP connection does in this system:**
- N8N connects to the email server as a mail client (like Outlook or Thunderbird)
- It checks the INBOX folder for messages with the UNSEEN flag
- On each poll, it fetches new messages and marks them as read to prevent duplicate processing
- Email content (sender, subject, body) is passed to the Claude API for analysis
- The connection is read-only from the mailbox's perspective — N8N reads emails but does not move or delete them unless you configure it to do so

**Security note:** N8N handles email credentials; ensure your N8N instance is secured with authentication before storing credentials. For NHS deployments, review whether shared mailbox access (rather than individual account credentials) is required.

---

## IMAP Settings Quick Reference

| Provider | IMAP Host | Port | SSL/TLS | Authentication |
|---|---|---|---|---|
| Gmail | imap.gmail.com | 993 | SSL/TLS | App Password (requires 2FA) |
| Microsoft 365 / NHS.net | outlook.office365.com | 993 | SSL/TLS | OAuth2 (recommended) or App Password |
| Generic IMAP (e.g. Postfix) | your.mail.server.com | 993 | SSL/TLS | Username + password |
| Generic IMAP (unencrypted — dev only) | your.mail.server.com | 143 | None | Username + password |

---

## Option A: Gmail

Gmail requires an **App Password** to allow N8N to connect via IMAP. Standard Gmail passwords cannot be used with IMAP once 2-Factor Authentication (2FA) is enabled (and 2FA is required for App Passwords).

### Step 1: Enable 2-Step Verification on your Google Account

1. Go to [myaccount.google.com](https://myaccount.google.com)
2. Click **Security** in the left navigation
3. Under "How you sign in to Google", click **2-Step Verification**
4. Follow the prompts to enable 2FA (phone, authenticator app, or security key)

### Step 2: Generate an App Password

1. Go to [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords) (you must have 2FA enabled)
2. Under "Select app", choose **Mail**
3. Under "Select device", choose **Other (custom name)**
4. Type a name for the app password, e.g. `N8N FOI Agent`
5. Click **Generate**
6. Google will display a 16-character app password. **Copy this immediately** — it will not be shown again.

### Step 3: Enable IMAP in Gmail

1. In Gmail, click the gear icon → **See all settings**
2. Click the **Forwarding and POP/IMAP** tab
3. Under "IMAP access", select **Enable IMAP**
4. Click **Save Changes**

### Step 4: Create the IMAP credential in N8N

1. In N8N, go to **Settings → Credentials → New Credential**
2. Search for and select **IMAP**
3. Fill in the fields:
   - **Credential Name:** `FOI IMAP Account`
   - **Host:** `imap.gmail.com`
   - **Port:** `993`
   - **SSL/TLS:** `SSL/TLS` (not STARTTLS)
   - **User:** `foi@westshireicb.nhs.uk` (your Gmail address)
   - **Password:** [the 16-character App Password from Step 2]
   - **Allow unauthorized certs:** `No`
4. Click **Test Credential** to verify the connection
5. Click **Save**

---

## Option B: Microsoft 365 / NHS.net

NHS.net email is hosted on Microsoft 365 (Office 365). There are two methods: **OAuth2** (recommended, more secure) or **Basic Auth with App Password** (simpler, requires legacy authentication to be enabled by your Microsoft 365 admin).

### Option B1: OAuth2 (Recommended for NHS)

OAuth2 allows N8N to access the mailbox without handling the account password. It requires a one-time setup in Azure Active Directory.

#### Step 1: Register an application in Azure AD

1. Sign in to the [Azure Portal](https://portal.azure.com) with your NHS.net admin account
2. Navigate to **Azure Active Directory → App registrations → New registration**
3. Fill in the registration:
   - **Name:** `N8N FOI Email Agent`
   - **Supported account types:** Accounts in this organizational directory only
   - **Redirect URI:** Web → `https://your-n8n-instance.url/rest/oauth2-credential/callback`
4. Click **Register**
5. Note the **Application (client) ID** — you will need this

#### Step 2: Create a client secret

1. In your app registration, click **Certificates & secrets → New client secret**
2. Description: `N8N FOI Agent Secret`
3. Expiry: 24 months (note the expiry date — you will need to renew before it expires)
4. Click **Add**
5. Copy the **Value** immediately — it is only shown once

#### Step 3: Grant API permissions

1. In your app registration, click **API permissions → Add a permission**
2. Select **Microsoft Graph** → **Delegated permissions**
3. Search for and add: `IMAP.AccessAsUser.All`
4. Also add: `SMTP.Send` (for the SMTP credential)
5. Click **Grant admin consent for [your organisation]**

Alternatively, for older Office 365 configurations using Exchange Online permissions:
- Use: `https://outlook.office365.com/IMAP.AccessAsUser.All`
- Use: `https://outlook.office365.com/SMTP.Send`

#### Step 4: Create the credential in N8N

1. In N8N, go to **Settings → Credentials → New Credential**
2. Search for and select **IMAP** (OAuth2 type if available, or use Microsoft OAuth2)
3. Fill in:
   - **Credential Name:** `FOI IMAP Account`
   - **Host:** `outlook.office365.com`
   - **Port:** `993`
   - **SSL/TLS:** `SSL/TLS`
   - **Client ID:** [from Step 1]
   - **Client Secret:** [from Step 2]
   - **Tenant ID:** [your Microsoft 365 tenant ID — found in Azure AD → Overview]
4. Click **Connect my account** and authorise

### Option B2: Basic Auth with App Password (NHS.net)

NHS.net supports per-user app passwords if Basic Authentication has not been disabled by your NHS.net administrator.

1. Sign in to the NHS.net portal (portal.office365.com)
2. Go to **My Account → Security info**
3. If App passwords are available: click **Add method → App password**
4. Name: `N8N FOI Agent`
5. Copy the generated app password
6. In N8N, create an IMAP credential using:
   - Host: `outlook.office365.com`
   - Port: `993`
   - SSL/TLS: `SSL/TLS`
   - User: `foi@westshire.icb.nhs.uk`
   - Password: [app password]

**Note:** Microsoft is progressively disabling Basic Authentication on all Microsoft 365 tenants. If this option is not available, use OAuth2 (Option B1).

#### Shared Mailbox Setup for NHS

For NHS deployments, it is preferable to use a **shared mailbox** (`foi@westshire.icb.nhs.uk`) rather than an individual user account:
1. Create the shared mailbox in Microsoft 365 admin centre
2. Grant the N8N service account (or your Microsoft app registration) access to the shared mailbox
3. In the IMAP credential, use the shared mailbox address as the username

---

## Option C: Generic IMAP

For any IMAP server not listed above:

1. Obtain your IMAP server details from your email administrator:
   - IMAP host (e.g. `mail.example.nhs.uk`)
   - Port (993 for SSL, 143 for STARTTLS)
   - SSL setting (SSL/TLS recommended; STARTTLS if 993 is not available)
   - Your email address and password (or app password)

2. In N8N, create an IMAP credential with these settings

3. If your server uses a self-signed certificate (common in NHS internal infrastructure), enable **Allow unauthorized certs** — but note this reduces security; consult your IG team.

---

## Testing the Connection

After saving the credential:

1. In N8N, navigate to your credential and click **Test Credential**
2. N8N will attempt to connect to the IMAP server and log in
3. A green tick indicates success; a red X with an error message indicates a problem

If the test fails:
- Verify the hostname, port, and SSL setting
- Check that the password or app password is correct (no spaces, correct case)
- Check that IMAP access is enabled for the account

---

## Troubleshooting

| Problem | Likely cause | Solution |
|---|---|---|
| `Connection refused` on port 993 | IMAP SSL/TLS not enabled, or wrong port | Try port 143 with STARTTLS; check with email admin |
| `Authentication failed` | Wrong password or 2FA enabled without app password | Generate an app password and use that |
| `IMAP is disabled` (Gmail) | IMAP not enabled in Gmail settings | Enable IMAP in Gmail settings (see Step 3 above) |
| `Certificate error` | Self-signed or invalid cert on IMAP server | Enable "Allow unauthorized certs" (only on trusted internal servers) |
| NHS.net: `Client not authenticated` | Basic Auth disabled by tenant admin | Use OAuth2 (Option B1) |
| `Too many connections` | IMAP polling too frequent | Increase schedule trigger interval to 5 minutes |
| Emails not being marked as read | `markSeen: false` in node settings | Set `markSeen: true` in the Read IMAP node parameters |

---

## Security Considerations

**App passwords vs account passwords:**  
App passwords are scoped to a single application and can be revoked without changing your main account password. Always use app passwords rather than your main account password for automation.

**Principle of least privilege:**  
Where possible, configure the IMAP account with read-only access to the FOI inbox only. For shared mailboxes in Microsoft 365, this can be configured per-folder.

**NHS.net shared mailbox guidance:**  
Using a dedicated shared mailbox (`foi@westshire.icb.nhs.uk`) rather than an individual user's mailbox is strongly recommended. This means:
- Access does not depend on any individual being employed
- Access can be controlled centrally by the Microsoft 365 admin
- The mailbox continues to function if staff change
- Audit logging applies to the shared mailbox account

**Credential storage in N8N:**  
N8N encrypts credentials at rest using the `N8N_ENCRYPTION_KEY` environment variable. Ensure this key is set and backed up securely. Do not store credentials in workflow JSON files or in code nodes.

**Data in transit:**  
All connections in this system use SSL/TLS (port 993 for IMAP, port 587/465 for SMTP). Ensure `Allow unauthorized certs` is set to `No` in production to prevent man-in-the-middle attacks.
