# Renderbase Make.com Integration - Deployment Guide

This guide covers the complete process for deploying the Renderbase integration to Make.com (formerly Integromat).

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Make.com Developer Account Setup](#makecom-developer-account-setup)
3. [Creating the App in Make.com](#creating-the-app-in-makecom)
4. [OAuth 2.0 Configuration](#oauth-20-configuration)
5. [Uploading Integration Files](#uploading-integration-files)
6. [Testing the Integration](#testing-the-integration)
7. [Publishing the Integration](#publishing-the-integration)
8. [Maintenance and Updates](#maintenance-and-updates)

---

## Prerequisites

### Required Access

- Make.com account with app development access
- Renderbase admin access for OAuth client creation
- Renderbase backend with OAuth module deployed

### Backend Requirements

The Renderbase backend must have:

- OAuth 2.0 module deployed (`/api/oauth/*` endpoints)
- Webhook subscriptions module deployed (`/api/v1/webhook-subscriptions/*`)
- SSL certificate (HTTPS required)
- CORS configured for Make.com domains

---

## Make.com Developer Account Setup

### 1. Access Make.com Developer Portal

1. Log in to your Make.com account
2. Navigate to **My Apps** in the left sidebar
3. Click **Create a new app**

### 2. Register as Developer (if needed)

If this is your first app:
1. Accept the developer terms
2. Complete developer profile
3. Verify email if required

---

## Creating the App in Make.com

### 1. Create New App

1. Click **Create a new app**
2. Fill in the basic information:

| Field | Value |
|-------|-------|
| **App Name** | Renderbase |
| **App Slug** | renderbase |
| **Description** | Send transactional emails with dynamically generated PDF and Excel attachments |
| **Category** | Email |
| **Version** | 1.0.0 |

### 2. Configure App Settings

**Branding:**
- Upload Renderbase logo (512x512px PNG)
- Set brand color: `#your-brand-color`

**URLs:**
- Homepage: `https://renderbase.dev`
- Documentation: `https://docs.renderbase.dev`
- Support Email: `support@renderbase.dev`

**Countries:**
- Select all countries where Renderbase operates
- Common selections: US, IN, GB, DE, FR, AU, CA

---

## OAuth 2.0 Configuration

### 1. Create OAuth Client in Renderbase

First, create an OAuth client in Renderbase for Make.com:

```bash
curl -X POST https://api.renderbase.dev/api/admin/oauth/clients \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Make.com Integration",
    "redirectUris": [
      "https://www.integromat.com/oauth/cb/app",
      "https://www.make.com/oauth/cb/app"
    ],
    "scopes": [
      "emails:send",
      "emails:read",
      "templates:read",
      "webhooks:read",
      "webhooks:write",
      "profile:read"
    ],
    "grantTypes": ["authorization_code", "refresh_token"],
    "tokenEndpointAuthMethod": "client_secret_post"
  }'
```

**Save the returned `client_id` and `client_secret`.**

### 2. Configure OAuth in Make.com

1. In your Make.com app, go to **Base** section
2. Set the base configuration:

```json
{
  "baseUrl": "https://api.renderbase.dev/api/v1"
}
```

3. Go to **Connections** section
4. Create a new connection of type **OAuth 2.0**
5. Configure the OAuth settings:

**Authorization URL:**
```
https://app.renderbase.dev/oauth/authorize
```

**Token URL:**
```
https://api.renderbase.dev/api/oauth/token
```

**Scopes:**
```
emails:send emails:read templates:read webhooks:read webhooks:write profile:read
```

**Refresh Token Settings:**
- Enable refresh token
- Set refresh URL to token URL

### 3. Configure Connection Test

Set up the info endpoint to verify connections:

**URL:** `https://api.renderbase.dev/api/v1/me`

**Response Mapping:**
```json
{
  "uid": "{{body.data.id}}",
  "label": "{{body.data.email}}"
}
```

---

## Uploading Integration Files

### 1. Upload Base Configuration

In the **Base** section, paste the contents of `base.json`:

```json
{
  "baseUrl": "https://api.renderbase.dev/api/v1",
  "headers": {
    "Content-Type": "application/json",
    "User-Agent": "Renderbase-Make/1.0"
  }
}
```

### 2. Upload Connection Configuration

In **Connections**, create OAuth connection using `src/connections/oauth2.json`.

### 3. Upload Webhooks

In **Webhooks** section, upload `src/webhooks/email-events.json`.

### 4. Upload Modules

For each module file in `src/modules/`:

1. Go to **Modules** section
2. Click **Create a new module**
3. Select appropriate type (Trigger, Action, or Search)
4. Paste the module JSON configuration
5. Save and test

**Trigger Modules:**
- `watch-email-sent.json`
- `watch-email-delivered.json`
- `watch-email-opened.json`
- `watch-email-clicked.json`
- `watch-email-bounced.json`
- `watch-unsubscribe.json`

**Action Modules:**
- `send-email.json`
- `send-email-pdf.json`
- `send-email-excel.json`
- `send-bulk-email.json`

**Search Modules:**
- `get-email.json`

### 5. Upload RPCs

In **Remote Procedures** section, upload:
- `src/rpcs/list-templates.json`
- `src/rpcs/list-pdf-templates.json`
- `src/rpcs/list-excel-templates.json`

### 6. Configure Groups

In **Groups** section, configure module organization using `src/groups.json`.

---

## Testing the Integration

### 1. Test Connection

1. Go to **Connections** in your Make.com workspace
2. Click **Add** to create new Renderbase connection
3. Enter the Client ID and Client Secret
4. Click **Authorize**
5. Complete OAuth flow in popup
6. Verify connection shows as "Connected"

### 2. Test Triggers

1. Create a new scenario
2. Add a Renderbase trigger module
3. Choose "Watch Email Sent"
4. Select your connection
5. Click **Run once**
6. Send a test email from Renderbase
7. Verify the trigger receives the event

### 3. Test Actions

1. Add a Renderbase action module
2. Choose "Send Email"
3. Select template from dropdown
4. Fill in recipient details
5. Run the scenario
6. Verify email is sent

### 4. Test Search

1. Add "Find Email" module
2. Search by email ID or recipient
3. Verify results returned correctly

---

## Publishing the Integration

### 1. Pre-Publication Checklist

- [ ] All modules have been tested
- [ ] OAuth flow works reliably
- [ ] All error messages are user-friendly
- [ ] Sample data is accurate for all modules
- [ ] Help text is clear and complete
- [ ] Logo and branding are configured
- [ ] Documentation links work

### 2. Submit for Review

1. Go to app settings
2. Click **Submit for review**
3. Fill in the review request form:
   - Describe the integration
   - List key use cases
   - Provide test credentials if needed
4. Submit

### 3. Review Process

**Timeline:** Typically 1-2 weeks

**Common Feedback:**
- Improve help text clarity
- Add more sample scenarios
- Fix edge case handling
- Improve error messages

### 4. After Approval

Once approved:
1. App becomes publicly available
2. Users can install from Make.com marketplace
3. Monitor usage in analytics dashboard

---

## Maintenance and Updates

### Version Management

1. Create new version in Make.com
2. Upload updated module files
3. Test thoroughly
4. Set migration path for existing users
5. Publish new version

### Common Updates

**Adding a new module:**
1. Create module JSON file
2. Upload to Make.com
3. Add to appropriate group
4. Test and document

**Updating OAuth scopes:**
1. Update Renderbase OAuth client
2. Update connection configuration
3. Users may need to re-authorize

### Monitoring

- Check Make.com analytics dashboard
- Monitor error rates
- Track user adoption
- Collect user feedback

---

## Troubleshooting

### Common Issues

**OAuth Authorization Fails:**
- Verify Client ID and Secret are correct
- Check redirect URIs match exactly
- Ensure SSL certificates are valid
- Check Renderbase OAuth module is running

**Webhook Not Receiving Events:**
- Verify webhook URL is accessible from internet
- Check webhook subscription was created
- Confirm user has webhooks:write scope
- Check Renderbase webhook delivery logs

**Template Dropdown Empty:**
- Verify templates:read scope is granted
- Check RPC configuration
- Ensure user has templates in Renderbase

**Rate Limiting:**
- Implement backoff in scenarios
- Use bulk operations when possible
- Contact support for limit increases

### Getting Help

- **Make.com Documentation:** https://www.make.com/en/api-documentation
- **Make.com Developer Forum:** https://community.make.com
- **Renderbase Support:** support@renderbase.dev

---

## Quick Reference

### Module Types in Make.com

| Type | Purpose | Example |
|------|---------|---------|
| Trigger | Start scenario on event | Watch Email Sent |
| Action | Perform an operation | Send Email |
| Search | Find and return data | Find Email |
| Instant Trigger | Real-time webhook trigger | All our triggers |

### File Structure

```
make-renderbase/
├── app.json              # App metadata
├── base.json             # Base configuration
├── package.json          # Package info
├── README.md             # Documentation
├── DEPLOYMENT.md         # This guide
└── src/
    ├── common.json       # Shared settings
    ├── groups.json       # Module grouping
    ├── connections/
    │   └── oauth2.json   # OAuth configuration
    ├── webhooks/
    │   └── email-events.json
    ├── modules/
    │   ├── watch-*.json  # Trigger modules
    │   ├── send-*.json   # Action modules
    │   └── get-*.json    # Search modules
    └── rpcs/
        └── list-*.json   # Dynamic dropdown data
```

### Important URLs

| Resource | URL |
|----------|-----|
| Make.com Developer Portal | https://www.make.com/en/integrations |
| Make.com API Docs | https://www.make.com/en/api-documentation |
| Renderbase API Docs | https://docs.renderbase.dev/api |
| Renderbase OAuth Docs | https://docs.renderbase.dev/oauth |
