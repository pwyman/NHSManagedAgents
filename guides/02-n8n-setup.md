# Guide 02: N8N Setup

## Overview

N8N is the workflow automation platform that orchestrates the FOI Agent. It connects the email inbox, the Anthropic Claude API, and the Google Sheets case register. This guide covers installing N8N, importing the FOI workflows, and configuring it for an NHS production environment.

---

## 1. Prerequisites

Before installing N8N, ensure you have:

- **Docker** (recommended) — Docker Desktop for Mac/Windows, or Docker Engine + Docker Compose on Linux/server. Version 20.10 or later.
- **OR Node.js 18+** — for bare metal installation (not recommended for production)
- At least **2 GB RAM** and **10 GB disk space** available
- A domain name or subdomain for your N8N instance (e.g. `foi-agent.westshireicb.nhs.uk`) with HTTPS enabled
- An SSL certificate (Let's Encrypt / Certbot, or NHS PKI certificate)
- Your SMTP, IMAP, Anthropic API, and Google Sheets credentials ready (see other guides)

---

## 2. Docker Installation (Recommended)

Docker is the recommended way to run N8N. It is simpler to manage, update, and back up.

### Step 1: Create a project directory

```bash
mkdir ~/n8n-foi-agent
cd ~/n8n-foi-agent
```

### Step 2: Create the docker-compose.yml file

Create a file named `docker-compose.yml` with the following content:

```yaml
version: '3.8'
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${N8N_HOST}
      - GENERIC_TIMEZONE=Europe/London
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - FOI_EMAIL_ADDRESS=${FOI_EMAIL_ADDRESS}
      - FOI_REVIEWER_EMAIL=${FOI_REVIEWER_EMAIL}
      - GOOGLE_SHEETS_ID=${GOOGLE_SHEETS_ID}
    volumes:
      - n8n_data:/home/node/.n8n
volumes:
  n8n_data:
```

### Step 3: Create the environment file

Create a file named `.env` in the same directory (this file should NOT be committed to version control):

```bash
# N8N Authentication
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=change-this-to-a-strong-password

# N8N Host Configuration
N8N_HOST=foi-agent.westshireicb.nhs.uk

# N8N Encryption (generate a random 32-character string)
N8N_ENCRYPTION_KEY=your-32-character-random-encryption-key

# Anthropic API
ANTHROPIC_API_KEY=sk-ant-your-key-here

# FOI Email Configuration
FOI_EMAIL_ADDRESS=foi@westshire.icb.nhs.uk
FOI_REVIEWER_EMAIL=diane.okafor@westshire.icb.nhs.uk

# Google Sheets
GOOGLE_SHEETS_ID=your-google-sheet-id-here
```

**Generate a secure encryption key:**
```bash
openssl rand -hex 16
```

### Step 4: Start N8N

```bash
docker-compose up -d
```

N8N will start and be available at `http://localhost:5678` (or at your configured domain once you set up the reverse proxy).

**Check it is running:**
```bash
docker-compose logs -f
```

You should see lines like:
```
n8n_1  | Editor is now accessible via:
n8n_1  | http://localhost:5678
```

### Step 5: Set up HTTPS reverse proxy (production)

For production, you need HTTPS. The recommended approach is nginx + Let's Encrypt:

```bash
# Install nginx and certbot
sudo apt-get install nginx certbot python3-certbot-nginx

# Get SSL certificate (replace with your actual domain)
sudo certbot --nginx -d foi-agent.westshireicb.nhs.uk
```

Nginx configuration (`/etc/nginx/sites-available/n8n`):

```nginx
server {
    server_name foi-agent.westshireicb.nhs.uk;
    
    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_read_timeout 91s;
    }

    listen 443 ssl;
    # SSL config managed by certbot
}
```

---

## 3. Bare Metal Installation (Development/Testing Only)

Not recommended for production. Use this for local testing only.

### Step 1: Install Node.js 18+

Download from [nodejs.org](https://nodejs.org) and follow the installer.

Verify:
```bash
node --version  # should show v18.x.x or higher
npm --version
```

### Step 2: Install N8N globally

```bash
npm install -g n8n
```

### Step 3: Set environment variables

On macOS/Linux:
```bash
export N8N_BASIC_AUTH_ACTIVE=true
export N8N_BASIC_AUTH_USER=admin
export N8N_BASIC_AUTH_PASSWORD=your-password
export N8N_HOST=localhost
export GENERIC_TIMEZONE=Europe/London
export ANTHROPIC_API_KEY=sk-ant-your-key-here
export FOI_EMAIL_ADDRESS=foi@westshire.icb.nhs.uk
export FOI_REVIEWER_EMAIL=diane.okafor@westshire.icb.nhs.uk
export GOOGLE_SHEETS_ID=your-sheet-id
```

### Step 4: Start N8N

```bash
n8n start
```

N8N will be available at `http://localhost:5678`.

---

## 4. First Launch and Account Setup

On first launch:

1. Open a browser and go to `http://localhost:5678` (or your production URL)
2. N8N will show the Setup page. Create the **owner account**:
   - Email address (use a real address — you will receive workflow notifications here)
   - First name, last name
   - Password (use a strong password, different from the Basic Auth password)
3. Click **Get started**
4. N8N will ask about telemetry — choose your preference. For NHS, select **No** unless you have confirmed this with your IG team.
5. You will arrive at the N8N dashboard.

---

## 5. Import the FOI Workflows

### Step 1: Import the email workflow

1. In the N8N left sidebar, click **Workflows**
2. Click **+ New Workflow** → then click the **Import from file** option (the three dots menu or the import button)
3. Navigate to `NHSManagedAgents/n8n-workflows/foi-email-workflow.json`
4. Click **Open** or **Import**
5. The workflow will open in the editor. You will see warnings about missing credentials — this is expected; we configure them next.
6. Click **Save** (Ctrl+S / Cmd+S)

### Step 2: Import the form workflow

Repeat the same process for `foi-form-workflow.json`.

---

## 6. Configure Credentials

### Step 1: Create the IMAP credential

1. Go to **Settings → Credentials → Add Credential**
2. Search for **IMAP**
3. Name it exactly `FOI IMAP Account` (this matches the credential name in the workflow)
4. Fill in your IMAP settings (see Guide 01 for details)
5. Click **Test** then **Save**

### Step 2: Create the SMTP credential (NHS.net)

1. Go to **Settings → Credentials → Add Credential**
2. Search for **SMTP**
3. Name it exactly `NHS.net SMTP`
4. Fill in:
   - **Host:** `smtp.office365.com`
   - **Port:** `587`
   - **Secure:** `STARTTLS`
   - **User:** `foi@westshire.icb.nhs.uk`
   - **Password:** [your app password or OAuth token]
5. Click **Test** then **Save**

### Step 3: Create the Anthropic API Key credential

1. Go to **Settings → Credentials → Add Credential**
2. Search for **HTTP Header Auth**
3. Name it exactly `Anthropic API Key`
4. Fill in:
   - **Name:** `x-api-key`
   - **Value:** `sk-ant-your-api-key-here`
5. Click **Save**

### Step 4: Create the Google Sheets credential

1. Go to **Settings → Credentials → Add Credential**
2. Search for **Google Sheets OAuth2**
3. Name it exactly `Google Sheets`
4. Click **Connect my account** and follow the OAuth flow to authorise N8N to access your Google Sheets

---

## 7. Activate the Workflows

Once all credentials are configured:

1. Open the **foi-email-workflow** in the editor
2. Click the toggle in the top right corner from **Inactive** to **Active**
3. Confirm the activation
4. Repeat for **foi-form-workflow**

The email workflow will now poll the FOI inbox every 2 minutes. The form workflow will activate its webhook immediately — copy the webhook URL from the Form Trigger node to embed in the ICB website.

---

## 8. N8N on NHS Infrastructure

### Azure Deployment Notes

For NHS deployments on Azure (common for NHS ICBs using Azure NHS subscriptions):

**Recommended architecture:**
```
Azure App Service (or Azure Container Instances)
  └─ N8N Docker container
       ├─ Azure File Share (for /home/node/.n8n persistent data)
       └─ Azure Key Vault (for secrets — use environment variables referencing KV secrets)
```

**Azure environment variables:**  
In Azure App Service → Configuration → Application settings, add each environment variable from your `.env` file. Azure will inject these into the container at runtime.

**Data residency:**  
Ensure your Azure subscription uses a UK region (UK South or UK West) for data residency compliance. The N8N container itself does not store email content long-term, but execution logs may contain fragments of email content — review your retention policy.

**Networking:**  
For maximum security, place N8N inside a Virtual Network (VNet) with:
- Inbound: only from the ICB's IP ranges + the FOI request form webhook endpoint
- Outbound: allow HTTPS to `api.anthropic.com`, IMAP/SMTP to Office 365, HTTPS to Google APIs

**Persistent storage:**  
Always use a persistent volume. Without it, workflow configurations and credentials will be lost on container restart.

```bash
# Docker with persistent storage (Azure File Share example)
volumes:
  n8n_data:
    driver: azure-file
    driver_opts:
      share_name: n8n-foi-data
      storage_account_name: ${STORAGE_ACCOUNT}
```

### Production Environment Variables

Set these additional variables for production:

```bash
# Security
N8N_BASIC_AUTH_ACTIVE=true
N8N_SECURE_COOKIE=true

# Performance  
EXECUTIONS_PROCESS=main
N8N_CONCURRENCY_PRODUCTION_LIMIT=5

# Logging
N8N_LOG_LEVEL=info
N8N_LOG_OUTPUT=console

# Data retention (executions)
EXECUTIONS_DATA_SAVE_ON_ERROR=all
EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
EXECUTIONS_DATA_MAX_AGE=168  # hours (7 days)
```

### Updating N8N

To update N8N to the latest version (Docker):

```bash
cd ~/n8n-foi-agent
docker-compose pull
docker-compose up -d
```

Always test workflow imports after major N8N version upgrades.
