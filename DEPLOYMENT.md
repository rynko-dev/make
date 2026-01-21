# Renderbase Make.com Integration - Deployment Guide

This guide covers the complete process for deploying the Renderbase integration to Make.com (formerly Integromat).

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Creating the App in Make.com](#creating-the-app-in-makecom)
3. [Configuring the Base](#configuring-the-base)
4. [Setting Up the Connection](#setting-up-the-connection)
5. [Creating Modules](#creating-modules)
6. [Adding Remote Procedures (RPCs)](#adding-remote-procedures-rpcs)
7. [Configuring Webhooks](#configuring-webhooks)
8. [Testing the Integration](#testing-the-integration)
9. [Publishing the Integration](#publishing-the-integration)
10. [Maintenance and Updates](#maintenance-and-updates)

---

## Prerequisites

### Required Access

- Make.com account (any paid plan with Custom Apps access)
- Renderbase admin access for OAuth client creation
- Renderbase backend with OAuth module deployed

### Backend Requirements

The Renderbase backend must have:

- OAuth 2.0 module deployed (`/api/oauth/*` endpoints)
- Webhook subscriptions module deployed (`/api/v1/webhook-subscriptions/*`)
- SSL certificate (HTTPS required)
- CORS configured for Make.com domains

### Create OAuth Client in Renderbase

Before starting in Make.com, create an OAuth client in Renderbase:

```bash
curl -X POST https://api.renderbase.dev/api/admin/oauth/clients \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Make.com Integration",
    "redirectUris": [
      "https://www.make.com/oauth/cb/app",
      "https://www.integromat.com/oauth/cb/app"
    ],
    "scopes": [
      "documents:generate",
      "documents:read",
      "templates:read",
      "webhooks:read",
      "webhooks:write",
      "profile:read"
    ],
    "grantTypes": ["authorization_code", "refresh_token"],
    "tokenEndpointAuthMethod": "client_secret_post"
  }'
```

**Save the returned `client_id` and `client_secret` - you'll need them later.**

---

## Creating the App in Make.com

### Step 1: Access Custom Apps

1. Log in to your Make.com account
2. In the left sidebar, scroll down and click **Custom Apps**
3. Click the **+ Create a new app** button in the upper-right corner

### Step 2: Fill in App Details

In the pop-up form, enter the following:

| Field | Value |
|-------|-------|
| **Name** | `renderbase` (internal identifier, must be lowercase with hyphens only) |
| **Label** | `Renderbase` (display name users will see) |
| **Description** | Generate PDF and Excel documents from templates with dynamic data |
| **Theme** | `#8B5CF6` (Renderbase violet) |
| **Language** | English |
| **Audience** | Global |

### Step 3: Upload App Logo

Upload the Renderbase logo (512x512px PNG recommended) from `assets/icon-512.png`.

### Step 4: Save the App

Click **Save** to create your app. You'll be taken to the app editor where you can configure the remaining components.

---

## Configuring the Base

The Base contains settings shared across all modules in your app.

### Step 1: Open Base Configuration

1. In your app editor, click **Base** in the left sidebar
2. You'll see the Base JSON editor

### Step 2: Enter Base Configuration

Copy and paste the contents of `base.json`:

```json
{
  "baseUrl": "https://api.renderbase.dev/api/v1",
  "authorizationUrl": "https://app.renderbase.dev/oauth/authorize",
  "tokenUrl": "https://api.renderbase.dev/api/oauth/token",
  "headers": {
    "Content-Type": "application/json",
    "User-Agent": "Renderbase-Make/1.0"
  },
  "log": {
    "sanitize": ["authorization", "access_token", "refresh_token"]
  },
  "response": {
    "error": {
      "message": "{{body.message}}",
      "type": "{{body.error}}"
    }
  }
}
```

### Step 3: Save Base

Click **Save** to store the base configuration.

---

## Setting Up the Connection

Connections handle authentication between Make.com and Renderbase.

### Step 1: Create New Connection

1. Click **Connections** in the left sidebar
2. Click **+ Create a new connection**

### Step 2: Configure Connection

1. Set the **Name** to `oauth2`
2. Set the **Label** to `Renderbase OAuth`
3. Set the **Type** to `OAuth 2.0`

### Step 3: Enter Connection JSON

Copy the contents of `src/connections/oauth2.json` into the connection editor:

```json
{
  "name": "oauth2",
  "label": "Renderbase OAuth",
  "type": "oauth2",
  "oauth2": {
    "authorize": {
      "url": "{{base.authorizationUrl}}",
      "qs": {
        "client_id": "{{common.clientId}}",
        "redirect_uri": "{{oauth.redirectUri}}",
        "response_type": "code",
        "scope": "documents:generate documents:read templates:read webhooks:read webhooks:write profile:read",
        "state": "{{oauth.state}}"
      }
    },
    "token": {
      "url": "{{base.tokenUrl}}",
      "method": "POST",
      "body": {
        "grant_type": "authorization_code",
        "code": "{{oauth.code}}",
        "client_id": "{{common.clientId}}",
        "client_secret": "{{common.clientSecret}}",
        "redirect_uri": "{{oauth.redirectUri}}"
      },
      "response": {
        "data": {
          "accessToken": "{{body.access_token}}",
          "refreshToken": "{{body.refresh_token}}",
          "expiresIn": "{{body.expires_in}}"
        }
      }
    },
    "refresh": {
      "url": "{{base.tokenUrl}}",
      "method": "POST",
      "body": {
        "grant_type": "refresh_token",
        "refresh_token": "{{connection.refreshToken}}",
        "client_id": "{{common.clientId}}",
        "client_secret": "{{common.clientSecret}}"
      },
      "response": {
        "data": {
          "accessToken": "{{body.access_token}}",
          "refreshToken": "{{body.refresh_token}}",
          "expiresIn": "{{body.expires_in}}"
        }
      }
    },
    "info": {
      "url": "{{base.baseUrl}}/auth/verify",
      "headers": {
        "Authorization": "Bearer {{connection.accessToken}}"
      },
      "response": {
        "data": {
          "uid": "{{body.data.id}}",
          "label": "{{body.data.email}}"
        }
      }
    }
  },
  "common": [
    {
      "name": "clientId",
      "label": "Client ID",
      "type": "text",
      "required": true
    },
    {
      "name": "clientSecret",
      "label": "Client Secret",
      "type": "password",
      "required": true
    }
  ]
}
```

### Step 4: Save Connection

Click **Save** to store the connection configuration.

### Understanding the OAuth Flow

1. **authorize**: Redirects user to Renderbase login page with required parameters
2. **token**: Exchanges the authorization code for access and refresh tokens
3. **refresh**: Automatically refreshes expired tokens (Make handles this automatically)
4. **info**: Validates the connection and displays the connected user's email

---

## Creating Modules

Modules are the building blocks users add to their scenarios. Create each module type as follows:

### Step 1: Access Modules Section

Click **Modules** in the left sidebar.

### Step 2: Create Action Modules

For each action module, click **+ Create a new module** and select **Action** as the type.

**Generate PDF Module:**
1. Click **+ Create a new module**
2. Select type: **Action**
3. Enter name: `generatePdf`
4. Enter label: `Generate PDF`
5. Copy contents from `src/modules/generate-pdf.json`
6. Click **Save**

**Generate Excel Module:**
1. Click **+ Create a new module**
2. Select type: **Action**
3. Enter name: `generateExcel`
4. Enter label: `Generate Excel`
5. Copy contents from `src/modules/generate-excel.json`
6. Click **Save**

**Generate Batch Module:**
1. Click **+ Create a new module**
2. Select type: **Action**
3. Enter name: `generateBatch`
4. Enter label: `Generate Batch Documents`
5. Copy contents from `src/modules/generate-batch.json`
6. Click **Save**

### Step 3: Create Search Modules

**Get Document Module:**
1. Click **+ Create a new module**
2. Select type: **Search**
3. Enter name: `getDocument`
4. Enter label: `Get Document`
5. Copy contents from `src/modules/get-document.json`
6. Click **Save**

### Step 4: Create Instant Trigger Modules

For triggers that use webhooks, select **Instant Trigger** type.

**Watch Document Completed:**
1. Click **+ Create a new module**
2. Select type: **Instant Trigger**
3. Enter name: `watchDocumentCompleted`
4. Enter label: `Watch Document Completed`
5. Copy contents from `src/modules/watch-document-completed.json`
6. Click **Save**

**Watch Document Failed:**
1. Click **+ Create a new module**
2. Select type: **Instant Trigger**
3. Enter name: `watchDocumentFailed`
4. Enter label: `Watch Document Failed`
5. Copy contents from `src/modules/watch-document-failed.json`
6. Click **Save**

**Watch Batch Completed:**
1. Click **+ Create a new module**
2. Select type: **Instant Trigger**
3. Enter name: `watchBatchCompleted`
4. Enter label: `Watch Batch Completed`
5. Copy contents from `src/modules/watch-batch-completed.json`
6. Click **Save**

---

## Adding Remote Procedures (RPCs)

RPCs provide dynamic data for module dropdowns, like template lists.

### Step 1: Access RPCs Section

Click **Remote Procedures** in the left sidebar.

### Step 2: Create RPCs

**List Templates RPC:**
1. Click **+ Create a new RPC**
2. Enter name: `listTemplates`
3. Enter label: `List Templates`
4. Copy contents from `src/rpcs/list-templates.json`
5. Click **Save**

**List PDF Templates RPC:**
1. Click **+ Create a new RPC**
2. Enter name: `listPdfTemplates`
3. Enter label: `List PDF Templates`
4. Copy contents from `src/rpcs/list-pdf-templates.json`
5. Click **Save**

**List Excel Templates RPC:**
1. Click **+ Create a new RPC**
2. Enter name: `listExcelTemplates`
3. Enter label: `List Excel Templates`
4. Copy contents from `src/rpcs/list-excel-templates.json`
5. Click **Save**

---

## Configuring Webhooks

Webhooks enable instant triggers to receive real-time events.

### Step 1: Access Webhooks Section

Click **Webhooks** in the left sidebar.

### Step 2: Create Webhook Configuration

1. Click **+ Create a new webhook**
2. Enter name: `documentEvents`
3. Enter label: `Document Events`
4. Copy contents from `src/webhooks/document-events.json`
5. Click **Save**

---

## Testing the Integration

### Step 1: Test Connection

1. Open a new browser tab and go to Make.com
2. Go to **Scenarios** and create a new scenario
3. Click the empty module placeholder
4. Search for your app name ("Renderbase")
5. Select any module (e.g., "Generate PDF")
6. Click **Create a connection**
7. Enter the OAuth Client ID and Client Secret from the prerequisites
8. Click **Save** and complete the OAuth authorization flow
9. Verify the connection shows the user's email

### Step 2: Test Action Modules

1. Add the "Generate PDF" module to your scenario
2. Select a template from the dropdown (should load via RPC)
3. Fill in the required variables
4. Click **Run once**
5. Verify the document is generated and the output contains the download URL

### Step 3: Test Trigger Modules

1. Create a new scenario with "Watch Document Completed" as the trigger
2. Configure the trigger and turn on the scenario
3. Generate a document from Renderbase (via API or dashboard)
4. Verify the scenario is triggered and receives the event data

### Step 4: Debug with Make DevTool

If you encounter issues:

1. Install the Make DevTool browser extension
2. Open DevTool while testing your app
3. Check the request/response logs for errors
4. Verify IML expressions are evaluated correctly

---

## Publishing the Integration

### Pre-Publication Checklist

- [ ] All modules tested and working
- [ ] OAuth flow works reliably
- [ ] Token refresh works correctly
- [ ] All RPCs return valid data
- [ ] Webhooks receive events properly
- [ ] Error messages are user-friendly
- [ ] Help text is clear and complete
- [ ] Logo and branding configured
- [ ] Documentation links are valid

### Submit for Review

1. In your app editor, click **App info** in the left sidebar
2. Ensure all required fields are complete
3. Click **Submit for review**
4. Fill in the review request form:
   - Describe the integration purpose
   - List key use cases
   - Provide test credentials if needed
5. Submit and wait for review (typically 1-2 weeks)

### After Approval

Once approved:
1. Your app becomes publicly available in the Make.com marketplace
2. Users can find and install it from the Apps directory
3. Monitor usage through Make.com analytics

---

## Maintenance and Updates

### Updating Modules

1. Open your app in the Custom Apps editor
2. Navigate to the module you want to update
3. Make your changes in the JSON editor
4. Click **Save**
5. Test the updated module in a scenario

### Version Management

For significant changes:
1. Create a new version in the app settings
2. Migrate modules to the new version
3. Test thoroughly
4. Set up migration path for existing users

### Monitoring

- Check Make.com app analytics dashboard
- Monitor error rates in your scenarios
- Review webhook delivery logs in Renderbase
- Collect user feedback from Make.com community

---

## Troubleshooting

### Connection Issues

**OAuth Authorization Fails:**
- Verify Client ID and Secret are correct
- Check redirect URI matches exactly: `https://www.make.com/oauth/cb/app`
- Ensure Renderbase OAuth module is running
- Check CORS allows Make.com domains

**Token Refresh Fails:**
- Verify refresh token is being stored correctly
- Check `refresh` block configuration
- Ensure refresh endpoint returns new tokens

### Module Issues

**Template Dropdown Empty:**
- Verify `templates:read` scope is granted
- Check RPC configuration and endpoint
- Test the API endpoint directly

**Document Generation Fails:**
- Check `documents:generate` scope is granted
- Verify template ID is valid
- Check variable data format matches template requirements

### Webhook Issues

**Triggers Not Firing:**
- Verify webhook was created in Renderbase
- Check webhook URL is accessible
- Verify `webhooks:write` scope is granted
- Check Renderbase webhook delivery logs

---

## Reference

### Module Types

| Type | Purpose | Example |
|------|---------|---------|
| Action | Perform a single operation | Generate PDF |
| Search | Return multiple results | List Documents |
| Instant Trigger | Real-time webhook events | Watch Document Completed |
| Universal | Custom API calls | Advanced users |

### IML Variables

Make uses IML (Integromat Markup Language) for dynamic values:

| Variable | Description |
|----------|-------------|
| `{{base.baseUrl}}` | Base URL from configuration |
| `{{common.clientId}}` | OAuth client ID |
| `{{connection.accessToken}}` | Current access token |
| `{{oauth.redirectUri}}` | Make's OAuth callback URL |
| `{{parameters.fieldName}}` | Module input parameters |
| `{{body.fieldName}}` | API response body fields |

### File Structure

```
make-renderbase/
├── app.json              # App metadata
├── base.json             # Base configuration
├── package.json          # Package info
├── README.md             # User documentation
├── DEPLOYMENT.md         # This guide
└── src/
    ├── common.json       # Shared secrets config
    ├── groups.json       # Module grouping
    ├── connections/
    │   └── oauth2.json   # OAuth configuration
    ├── webhooks/
    │   └── document-events.json
    ├── modules/
    │   ├── generate-pdf.json
    │   ├── generate-excel.json
    │   ├── generate-batch.json
    │   ├── get-document.json
    │   ├── watch-document-completed.json
    │   ├── watch-document-failed.json
    │   └── watch-batch-completed.json
    └── rpcs/
        ├── list-templates.json
        ├── list-pdf-templates.json
        └── list-excel-templates.json
```

### Useful Links

- [Make Developer Hub](https://developers.make.com/custom-apps-documentation)
- [Make Custom Apps Documentation](https://developers.make.com/custom-apps-documentation/basics/create-your-app)
- [OAuth 2.0 Connection Guide](https://developers.make.com/custom-apps-documentation/app-components/connections/oauth2)
- [Module Documentation](https://developers.make.com/custom-apps-documentation/app-structure/modules)
- [Make Community Forum](https://community.make.com/c/custom-apps/52)
- [Renderbase API Docs](https://docs.renderbase.dev/api-reference)
