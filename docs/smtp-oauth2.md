# SMTP with OAuth2 (XOAUTH2)

Microsoft 365 is phasing out password (Basic) authentication for SMTP AUTH, and Google only allows
OAuth2 for most accounts. Vaultwarden can authenticate to the SMTP server with OAuth2 access tokens
that it requests and renews by itself.

- [How it works](#how-it-works)
- [Settings](#settings)
- [Microsoft 365](#microsoft-365)
- [Google (Gmail / Workspace)](#google-gmail--workspace)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)

## How it works

1. When an email has to be sent, Vaultwarden requests an access token from `SMTP_OAUTH2_TOKEN_URL`.
2. It logs in to the SMTP server with the `XOAUTH2` mechanism: `SMTP_USERNAME` is the mailbox,
   the access token is the credential.
3. The token is cached in memory and reused until 2 minutes before it expires, then a new one is
   requested. Nothing is written to disk.
4. The cached token is dropped when the SMTP server answers `535` (authentication failed), so the
   next attempt gets a fresh token, and when any of the OAuth2 settings is changed in the admin panel.

Two OAuth2 grants are supported:

| Grant | Used when | Typical provider | User interaction |
|---|---|---|---|
| Client credentials | `SMTP_OAUTH2_REFRESH_TOKEN` is **not** set | Microsoft 365 (`SMTP.SendAsApp`) | None, app-only access |
| Refresh token | `SMTP_OAUTH2_REFRESH_TOKEN` is set | Gmail, Google Workspace | Once, to obtain the refresh token |

## Settings

| Variable | Required | Description |
|---|---|---|
| `SMTP_HOST` | yes | e.g. `smtp.office365.com`, `smtp.gmail.com` |
| `SMTP_SECURITY` / `SMTP_PORT` | yes | `starttls` / `587` for both providers |
| `SMTP_FROM` | yes | Sender address, usually the same mailbox as `SMTP_USERNAME` |
| `SMTP_USERNAME` | yes | Mailbox to authenticate as |
| `SMTP_PASSWORD` | **must not be set** | Not used with OAuth2 |
| `SMTP_OAUTH2_TOKEN_URL` | yes | Token endpoint, must be `https://`. Setting it enables OAuth2 |
| `SMTP_OAUTH2_CLIENT_ID` | yes | Application (client) ID |
| `SMTP_OAUTH2_CLIENT_SECRET` | for client credentials | Client secret |
| `SMTP_OAUTH2_SCOPE` | provider dependent | e.g. `https://outlook.office365.com/.default` |
| `SMTP_OAUTH2_REFRESH_TOKEN` | for refresh token grant | Enables the refresh token grant |

`SMTP_AUTH_MECHANISM` is ignored when OAuth2 is enabled, `XOAUTH2` is always used.

All settings can also be edited in the admin panel (`/admin` → SMTP Email Settings). The secret and the
refresh token are masked there like `SMTP_PASSWORD`. Like every Vaultwarden setting, `*_FILE` variants
work too, e.g. `SMTP_OAUTH2_CLIENT_SECRET_FILE=/run/secrets/smtp_oauth2_client_secret`.

## Microsoft 365

Microsoft 365 uses the client credentials grant: an app registration in Entra ID with the
`SMTP.SendAsApp` permission, allowed to send from one specific mailbox. No user has to sign in, and
no refresh token has to be kept alive.

You need a Global Administrator (or Privileged Role Administrator) for the admin consent and an
Exchange administrator for the PowerShell part.

### 1. Mailbox

Use a dedicated mailbox to send from, e.g. `noreply@example.com`. A **shared mailbox** is enough and
needs no license: Exchange admin center → Recipients → Mailboxes → *Add a shared mailbox*.

### 2. App registration

[Entra admin center](https://entra.microsoft.com) → App registrations → *New registration*

- Name: e.g. `Vaultwarden SMTP`
- Supported account types: *Accounts in this organizational directory only*
- Redirect URI: none

Note the **Application (client) ID** and the **Directory (tenant) ID** from the overview page.

### 3. API permission

App registration → API permissions → *Add a permission*:

1. Switch to the **APIs my organization uses** tab. It is **not** listed under *Microsoft APIs*.
2. Search for **Office 365 Exchange Online**.
3. **Application permissions** → `SMTP.SendAsApp` → *Add permissions*.
4. Click **Grant admin consent for \<your organization\>**. The status must turn into
   *Granted for \<your organization\>*.

> Do **not** use Microsoft Graph `Mail.Send` instead. It is a different API (not SMTP) and allows
> sending as *any* user in the tenant.

The default `User.Read` (Microsoft Graph, delegated) permission is not needed and can be removed.

### 4. Client secret

App registration → Certificates & secrets → Client secrets → *New client secret*.

- Expiry: at most 24 months in the portal. **Put a reminder in your calendar**: when the secret
  expires, Vaultwarden can no longer send emails.
- Copy the **Value** column (not *Secret ID*). It is only shown once.

### 5. Service principal object ID

Exchange needs the Object ID of the **service principal**, which is different from the Object ID
shown on the app registration's overview page.

Entra admin center → **Enterprise applications** → `Vaultwarden SMTP` → Overview → **Object ID**.
(Or, on the app registration overview: *Managed application in local directory* → the link.)

Using the app registration's Object ID fails with `AADServicePrincipalNotFound` in the next step.

### 6. Exchange Online

```powershell
Install-Module ExchangeOnlineManagement -Scope CurrentUser
Connect-ExchangeOnline

# Register the service principal in Exchange
New-ServicePrincipal -AppId <client-id> -ObjectId <service-principal-object-id> -DisplayName "Vaultwarden SMTP"

# Allow it to send from the mailbox
Add-MailboxPermission -Identity noreply@example.com -User <service-principal-object-id> -AccessRights FullAccess

# Make sure SMTP AUTH is enabled for the mailbox
Set-CASMailbox -Identity noreply@example.com -SmtpClientAuthenticationDisabled $false
```

Permission changes can take a few minutes (sometimes up to an hour) to become effective.

### 7. Vaultwarden

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    env_file:
      - .env   # SMTP_OAUTH2_CLIENT_SECRET=...
    environment:
      DOMAIN: "https://vault.example.com"
      SMTP_HOST: "smtp.office365.com"
      SMTP_SECURITY: "starttls"
      SMTP_PORT: "587"
      SMTP_FROM: "noreply@example.com"
      SMTP_FROM_NAME: "Vaultwarden"
      SMTP_USERNAME: "noreply@example.com"
      SMTP_OAUTH2_TOKEN_URL: "https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token"
      SMTP_OAUTH2_CLIENT_ID: "<client-id>"
      SMTP_OAUTH2_SCOPE: "https://outlook.office365.com/.default"
```

Restart Vaultwarden and send a test email from the admin panel (`/admin` → SMTP Email Settings →
*Send test email*; requires `ADMIN_TOKEN`).

### 8. Deliverability

Independent of OAuth2, but needed for mail to land in the inbox and not in spam:

- **SPF**: `v=spf1 include:spf.protection.outlook.com -all` on the domain.
- **DKIM**: [Defender portal → DKIM](https://security.microsoft.com/dkimv2) → domain → create keys,
  publish the two `selector1._domainkey` / `selector2._domainkey` CNAME records (DNS only, no proxy
  when using Cloudflare), then enable signing.
- **DMARC**: TXT record `_dmarc` with e.g. `v=DMARC1; p=none; rua=mailto:postmaster@example.com`
  to start with.

## Google (Gmail / Workspace)

Google uses the refresh token grant: the sending account authorizes the app once, Vaultwarden keeps
exchanging the refresh token for access tokens.

1. [Google Cloud console](https://console.cloud.google.com) → create a project.
2. APIs & Services → OAuth consent screen: configure it. Use *Internal* for a Workspace domain.
   For *External*, **publish the app** (set it to *In production*): refresh tokens of apps in
   *Testing* status expire after 7 days.
3. Credentials → *Create credentials* → OAuth client ID → *Web application*, add
   `https://developers.google.com/oauthplayground` as authorized redirect URI. Note client ID and secret.
4. [OAuth 2.0 Playground](https://developers.google.com/oauthplayground) → gear icon →
   *Use your own OAuth credentials* → enter client ID and secret.
   Scope: `https://mail.google.com/` → *Authorize APIs* → sign in **as the sending account** →
   *Exchange authorization code for tokens* → copy the **refresh token**.
5. Vaultwarden:

```yaml
      SMTP_HOST: "smtp.gmail.com"
      SMTP_SECURITY: "starttls"
      SMTP_PORT: "587"
      SMTP_FROM: "vaultwarden@example.com"
      SMTP_USERNAME: "vaultwarden@example.com"
      SMTP_OAUTH2_TOKEN_URL: "https://oauth2.googleapis.com/token"
      SMTP_OAUTH2_CLIENT_ID: "<client-id>"
      # in .env: SMTP_OAUTH2_CLIENT_SECRET and SMTP_OAUTH2_REFRESH_TOKEN
```

The refresh token stops working when the account's password is changed, access is revoked in the
Google account security settings, or it is unused for 6 months. Repeat step 4 then.

## Troubleshooting

### `SMTP OAuth2 token request failed with HTTP 400/401`

The token endpoint rejected the request: wrong tenant ID in the URL, wrong client ID, wrong or expired
secret, or (Google) revoked refresh token. Start Vaultwarden with
`LOG_LEVEL=info,vaultwarden::mail=debug` to log the endpoint's error response.

### `535 5.7.3 Authentication unsuccessful` (Microsoft 365)

The token was issued, but Exchange refused it. In that order:

1. **Missing role in the token.** Check that `SMTP.SendAsApp` (Office 365 Exchange Online,
   *Application*) is added **and** admin consent is granted. You can inspect the token (PowerShell):

   ```powershell
   $r = Invoke-RestMethod -Method Post "https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token" `
     -Body @{ client_id='<client-id>'; client_secret='<secret>'; grant_type='client_credentials'; scope='https://outlook.office365.com/.default' }
   $p = $r.access_token.Split('.')[1].Replace('-','+').Replace('_','/'); while ($p.Length % 4) { $p += '=' }
   [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p)) | ConvertFrom-Json | Select-Object aud, appid, oid, roles
   ```

   Expected: `aud` = `https://outlook.office365.com`, `roles` = `SMTP.SendAsApp`,
   `oid` = the service principal Object ID used in Exchange.
2. **Mailbox permission**: `Add-MailboxPermission` must use the service principal Object ID (`oid`).
3. **SMTP AUTH disabled** for the mailbox: `Get-CASMailbox noreply@example.com | fl SmtpClientAuthenticationDisabled`.
4. **Propagation**: wait 10–30 minutes after changing permissions.

There is no need to restart Vaultwarden after fixing it: the cached token is dropped on `535`.

### `AADServicePrincipalNotFound` in `New-ServicePrincipal`

The Object ID of the app registration was used. Use the one of the Enterprise application
(see [step 5](#5-service-principal-object-id)).

### Changes in the environment have no effect

Settings saved in the admin panel are stored in `data/config.json` and **override** environment
variables (they are highlighted in the admin panel). Either change them in the admin panel, or remove
the overrides with *Reset defaults* in the admin panel so that only the environment is used.

### Email sent but not received

The SMTP server accepted it, delivery failed later. Check the recipient's spam folder, the provider's
message trace (Exchange admin center → Mail flow → Message trace) and quarantine, and
[deliverability](#8-deliverability).

## Security notes

- The client secret and refresh token have to be stored in plain text: Vaultwarden presents them to
  the token endpoint, so they can't be hashed (unlike `ADMIN_TOKEN`). Keep them out of the compose
  file (`.env` or `*_FILE` with Docker secrets) and restrict file permissions.
- Values entered in the admin panel end up in `data/config.json`, and with it in backups of `data`.
- With Microsoft 365 the app can only send from mailboxes it was explicitly given access to with
  `Add-MailboxPermission`. Use a dedicated mailbox.
- Rotate the client secret before it expires; a new secret can be created while the old one is
  still valid, so rotation causes no downtime.
