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
- Integration API module deployed (`/api/v1/integration-api/*` endpoints)
- Webhook subscriptions module deployed (`/api/v1/webhook-subscriptions/*`)
- SSL certificate (HTTPS required)
- CORS configured for Make.com domains

> **Note:** The Integration API module provides user-scoped access to teams, workspaces, and templates for OAuth-authenticated users. This enables the cascading Team → Workspace → Template selection in Make.com modules.

### Create OAuth Client in Renderbase

Before starting in Make.com, you need to create an OAuth client in Renderbase. This is a **one-time setup** that provides the credentials used by all Make.com users.

#### Option 1: Via Renderbase Admin Dashboard (Recommended)

1. Log in to the Renderbase Admin Dashboard at `https://admin.renderbase.dev`
2. Navigate to **OAuth Clients** in the sidebar (under "System" section)
3. Click **Create OAuth Client**
4. Fill in the details:
   - **Name**: `Make.com Integration`
   - **Redirect URIs**:
     - `https://www.make.com/oauth/cb/app`
     - `https://www.integromat.com/oauth/cb/app`
   - **Scopes**: Select all required scopes:
     - `documents:generate`
     - `documents:read`
     - `templates:read`
     - `webhooks:read`
     - `webhooks:write`
     - `profile:read`

   > **Note**: Grant types are automatically set to `Authorization Code` + `Refresh Token` by the backend, which is the correct configuration for Make.com.

5. **Disable PKCE requirement** for this client (Make.com doesn't support the `base64url` function needed for PKCE)
6. Click **Create Client**
6. **Copy and securely store** the generated `Client ID` and `Client Secret`

> ⚠️ **Important**: The Client Secret is only shown once. Store it securely!

#### Option 2: Via Renderbase API

If you prefer using the API or need to automate the setup:

**Step 1: Get a JWT Token**

Log in to Renderbase as a platform admin and use the JWT access token from your session. You can find this in your browser's developer tools under Application → Cookies → `access_token`.

**Step 2: Create the OAuth Client**

```bash
curl -X POST https://api.renderbase.dev/api/oauth/clients \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Make.com Integration",
    "redirectUris": [
      "https://www.make.com/oauth/cb/app",
      "https://www.integromat.com/oauth/cb/app"
    ],
    "allowedScopes": [
      "documents:generate",
      "documents:read",
      "templates:read",
      "webhooks:read",
      "webhooks:write",
      "profile:read"
    ]
  }'
```

**Step 3: Save the Response**

The API returns the OAuth client details:

```json
{
  "success": true,
  "data": {
    "id": "oauth_client_abc123",
    "clientId": "rb_xxxxxxxxxxxxxxxx",
    "clientSecret": "rb_secret_xxxxxxxxxxxxxxxx",
    "name": "Make.com Integration",
    "redirectUris": ["https://www.make.com/oauth/cb/app", "..."],
    "scopes": ["documents:generate", "..."],
    "createdAt": "2026-01-21T10:00:00Z"
  }
}
```

**Copy and securely store the `clientId` and `clientSecret` values.**

#### What to Do with These Credentials

These credentials will be entered in the Make.com Custom Apps interface:

1. `clientId` → Goes in the **Common Data** tab
2. `clientSecret` → Goes in the **Common Data** tab

End users of your Make.com integration will **never see these credentials**. They simply click "Connect" and authenticate with their own Renderbase account via OAuth.

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

Upload the Renderbase logo (512x512px PNG recommended).

### Step 4: Save the App

Click **Save** to create your app. You'll be taken to the app editor where you can configure the remaining components.

---

## Configuring the Base

The Base contains settings shared across all modules in your app.

### Step 1: Open Base Configuration

1. In your app editor, click **Base** in the left sidebar
2. You'll see the Base JSON editor

### Step 2: Enter Base Configuration

Copy and paste the contents of [`base.json`](./base.json):

```json
{
  "baseUrl": "https://api.renderbase.dev/api/v1",
  "authorizationUrl": "https://app.renderbase.dev/oauth/authorize",
  "tokenUrl": "https://api.renderbase.dev/api/oauth/token",
  "userInfoUrl": "https://api.renderbase.dev/api/oauth/userinfo",
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

**Important URLs:**
- `baseUrl`: Base URL for API v1 endpoints (used by modules)
- `authorizationUrl`: OAuth authorization page (webapp, not API)
- `tokenUrl`: OAuth token exchange endpoint
- `userInfoUrl`: OAuth user info endpoint (used to verify connection)

### Step 3: Save Base

Click **Save** to store the base configuration.

---

## Setting Up the Connection

Connections handle authentication between Make.com and Renderbase. The Make.com interface has **separate tabs** for each configuration section.

### Step 1: Create New Connection

1. Click **Connections** in the left sidebar
2. Click **+ Create a new connection**
3. Set **Name** to `oauth2`
4. Set **Label** to `Renderbase OAuth`
5. Set **Type** to `OAuth 2.0`
6. Select **OAuth 2.0 Type**: `Authorization Code + Refresh Token`

> **OAuth 2.0 Types in Make.com:**
> - **Authorization Code** - Basic OAuth without refresh tokens
> - **Authorization Code + Refresh Token** - ✅ Use this one (supports automatic token refresh)
> - **Resource Owner Credentials** - For username/password authentication
> - **Client Credentials** - For server-to-server authentication

### Step 2: Configure Communication Tab

Click the **Communication** tab and paste the contents of [`src/connections/oauth2/communication.json`](./src/connections/oauth2/communication.json):

```json
{
  "authorize": {
    "url": "https://app.renderbase.dev/oauth/authorize",
    "qs": {
      "client_id": "{{common.clientId}}",
      "redirect_uri": "{{oauth.redirectUri}}",
      "response_type": "code",
      "scope": "documents:generate documents:read templates:read webhooks:read webhooks:write profile:read",
      "state": "{{oauth.state}}"
    },
    "response": {
      "temp": {
        "code": "{{query.code}}"
      }
    }
  },
  "token": {
    "url": "https://api.renderbase.dev/api/oauth/token",
    "method": "POST",
    "type": "urlencoded",
    "body": {
      "grant_type": "authorization_code",
      "code": "{{temp.code}}",
      "client_id": "{{common.clientId}}",
      "client_secret": "{{common.clientSecret}}",
      "redirect_uri": "{{oauth.redirectUri}}"
    },
    "response": {
      "data": {
        "accessToken": "{{body.access_token}}",
        "refreshToken": "{{body.refresh_token}}",
        "expires": "{{addSeconds(now, body.expires_in)}}"
      }
    }
  },
  "refresh": {
    "url": "https://api.renderbase.dev/api/oauth/token",
    "method": "POST",
    "type": "urlencoded",
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
        "expires": "{{addSeconds(now, body.expires_in)}}"
      }
    }
  },
  "info": {
    "url": "https://api.renderbase.dev/api/oauth/userinfo",
    "method": "GET",
    "headers": {
      "Authorization": "Bearer {{connection.accessToken}}"
    },
    "response": {
      "uid": "{{body.id}}",
      "metadata": {
        "type": "email",
        "value": "{{body.email}}"
      }
    }
  },
  "invalidate": {
    "url": "https://api.renderbase.dev/api/oauth/revoke",
    "method": "POST",
    "type": "urlencoded",
    "headers": {
      "Authorization": "Bearer {{connection.accessToken}}"
    },
    "body": {
      "token": "{{connection.accessToken}}"
    }
  }
}
```

> **Key Configuration Notes:**
> - The `authorize.response.temp` stores the authorization code from the callback
> - The `token` directive uses `{{temp.code}}` to access the stored code
> - The `type: "urlencoded"` sends the body as `application/x-www-form-urlencoded` (OAuth 2.0 standard)
> - OAuth scopes are hardcoded in `authorize.qs.scope` (Make.com's `{{oauth.scope}}` variable doesn't populate correctly)
> - Reference: [Make.com OAuth 2.0 Documentation](https://developers.make.com/custom-apps-documentation/app-structure/connections/oauth2)

> **Note:** Connection communication uses full URLs instead of `{{base.*}}` references because the base context is not available in connection configuration.

> **Important:** The Make.com OAuth client in Renderbase must have PKCE **disabled**. Make.com's IML doesn't support the `base64url` function required for PKCE. Since this is a confidential client (has client_secret), PKCE is not strictly required for security.

### Step 3: Configure Common Data Tab

Click the **Common Data** tab. This contains the OAuth client credentials that are **the same for all users** of your integration.

Enter the actual Client ID and Client Secret you obtained from Renderbase in the Prerequisites step:

```json
{
  "clientId": "YOUR_ACTUAL_RENDERBASE_CLIENT_ID",
  "clientSecret": "YOUR_ACTUAL_RENDERBASE_CLIENT_SECRET"
}
```

> **Important:** Replace the placeholder values with your actual OAuth client credentials from Renderbase. These values are encrypted and stored securely by Make.com. All users of your integration will use these same credentials - individual users authenticate via OAuth, not by entering these values.

### Step 4: Save Connection

Click **Save** to store the connection configuration.

---

## Creating Modules

Modules are the building blocks users add to their scenarios. Each module has **separate tabs** for different configuration sections.

### Action Module: Generate PDF

1. Click **Modules** in the left sidebar
2. Click **+ Create a new module**
3. Select type: **Action**
4. Set **Name** to `generatePdf`
5. Set **Label** to `Generate PDF`
6. Set **Description** to `Generate a PDF document from a template`
7. Select **Connection**: `oauth2`

**Configure each tab:**

**Communication Tab** - paste [`src/modules/generate-pdf/communication.json`](./src/modules/generate-pdf/communication.json):
```json
{
  "url": "/documents/generate",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer {{connection.accessToken}}",
    "Content-Type": "application/json"
  },
  "body": {
    "templateId": "{{parameters.templateId}}",
    "format": "pdf",
    "teamId": "{{parameters.teamId}}",
    "workspaceId": "{{parameters.workspaceId}}",
    "variables": "{{parameters.variables}}",
    "fileName": "{{parameters.fileName}}",
    "waitForCompletion": "{{ifempty(parameters.waitForCompletion, true)}}"
  },
  "response": {
    "output": "{{body.data}}"
  }
}
```

**Mappable Parameters Tab** - paste [`src/modules/generate-pdf/parameters.json`](./src/modules/generate-pdf/parameters.json):

This defines the cascading Team → Workspace → Template selection:
```json
[
  {
    "name": "teamId",
    "label": "Team",
    "type": "select",
    "required": true,
    "options": {
      "store": "rpc://listTeams"
    },
    "help": "Select a team from your Renderbase account"
  },
  {
    "name": "workspaceId",
    "label": "Workspace",
    "type": "select",
    "required": true,
    "options": {
      "store": "rpc://listWorkspaces?teamId={{parameters.teamId}}",
      "nested": "teamId"
    },
    "help": "Select a workspace within the selected team"
  },
  {
    "name": "templateId",
    "label": "PDF Template",
    "type": "select",
    "required": true,
    "options": {
      "store": "rpc://listPdfTemplates?workspaceId={{parameters.workspaceId}}",
      "nested": "workspaceId"
    },
    "help": "Select a PDF template from the selected workspace"
  },
  // ... variables, fileName, waitForCompletion fields
]
```

> **Note:** The `nested` property creates the cascading behavior - when Team changes, Workspace reloads; when Workspace changes, Template reloads.

**Interface Tab** - paste [`src/modules/generate-pdf/interface.json`](./src/modules/generate-pdf/interface.json)

**Samples Tab** - paste [`src/modules/generate-pdf/samples.json`](./src/modules/generate-pdf/samples.json)

**Scope Tab** - paste [`src/modules/generate-pdf/scope.json`](./src/modules/generate-pdf/scope.json)

Click **Save**.

### Action Module: Generate Excel

Repeat the same process using files from [`src/modules/generate-excel/`](./src/modules/generate-excel/):
- **Name**: `generateExcel`
- **Label**: `Generate Excel`
- **Description**: `Generate an Excel spreadsheet from a template`

### Action Module: Generate Batch

Repeat using files from [`src/modules/generate-batch/`](./src/modules/generate-batch/):
- **Name**: `generateBatch`
- **Label**: `Generate Batch Documents`
- **Description**: `Generate multiple documents from a single template`

### Search Module: Get Document

1. Click **+ Create a new module**
2. Select type: **Search** (or Action if retrieving single item)
3. Set **Name** to `getDocument`
4. Set **Label** to `Get Document`
5. Configure tabs using files from [`src/modules/get-document/`](./src/modules/get-document/)

### Instant Trigger: Watch Document Completed

1. Click **+ Create a new module**
2. Select type: **Instant Trigger**
3. Set **Name** to `watchDocumentCompleted`
4. Set **Label** to `Watch Document Completed`
5. Set **Description** to `Triggers when a document is successfully generated`
6. Select **Webhook**: `documentEvents`

**Configure each tab:**

**Static Parameters Tab** - paste [`src/modules/watch-document-completed/static.json`](./src/modules/watch-document-completed/static.json):
```json
{
  "events": ["document.completed"]
}
```

**Interface Tab** - paste [`src/modules/watch-document-completed/interface.json`](./src/modules/watch-document-completed/interface.json)

**Samples Tab** - paste [`src/modules/watch-document-completed/samples.json`](./src/modules/watch-document-completed/samples.json)

### Instant Trigger: Watch Document Failed

Repeat using files from [`src/modules/watch-document-failed/`](./src/modules/watch-document-failed/):
- **Name**: `watchDocumentFailed`
- **Label**: `Watch Document Failed`
- **Static events**: `["document.failed"]`

### Instant Trigger: Watch Batch Completed

Repeat using files from [`src/modules/watch-batch-completed/`](./src/modules/watch-batch-completed/):
- **Name**: `watchBatchCompleted`
- **Label**: `Watch Batch Completed`
- **Static events**: `["batch.completed"]`

---

## Adding Remote Procedures (RPCs)

RPCs provide dynamic data for module dropdowns. The integration uses a **cascading selection pattern**: Team → Workspace → Template.

### RPC: List Teams

1. Click **Remote Procedures** in the left sidebar
2. Click **+ Create a new RPC**
3. Set **Name** to `listTeams`
4. Set **Label** to `List Teams`
5. Select **Connection**: `oauth2`

**Communication Tab** - paste [`src/rpcs/list-teams/communication.json`](./src/rpcs/list-teams/communication.json):
```json
{
  "url": "/integration-api/teams",
  "method": "GET",
  "headers": {
    "Authorization": "Bearer {{connection.accessToken}}",
    "Content-Type": "application/json"
  },
  "response": {
    "iterate": "{{body}}",
    "output": {
      "value": "{{item.id}}",
      "label": "{{item.name}}"
    }
  }
}
```

### RPC: List Workspaces

1. Click **+ Create a new RPC**
2. Set **Name** to `listWorkspaces`
3. Set **Label** to `List Workspaces`
4. Select **Connection**: `oauth2`

**Communication Tab** - paste [`src/rpcs/list-workspaces/communication.json`](./src/rpcs/list-workspaces/communication.json):
```json
{
  "url": "/integration-api/workspaces",
  "method": "GET",
  "headers": {
    "Authorization": "Bearer {{connection.accessToken}}",
    "Content-Type": "application/json"
  },
  "qs": {
    "teamId": "{{parameters.teamId}}"
  },
  "response": {
    "iterate": "{{body}}",
    "output": {
      "value": "{{item.id}}",
      "label": "{{item.name}}"
    }
  }
}
```

**Parameters Tab** - paste [`src/rpcs/list-workspaces/parameters.json`](./src/rpcs/list-workspaces/parameters.json):
```json
[
  {
    "name": "teamId",
    "type": "text",
    "label": "Team ID",
    "required": true
  }
]
```

### RPC: List Templates

1. Click **+ Create a new RPC**
2. Set **Name** to `listTemplates`
3. Set **Label** to `List Templates`
4. Select **Connection**: `oauth2`

**Communication Tab** - paste [`src/rpcs/list-templates/communication.json`](./src/rpcs/list-templates/communication.json):
```json
{
  "url": "/integration-api/templates",
  "method": "GET",
  "headers": {
    "Authorization": "Bearer {{connection.accessToken}}",
    "Content-Type": "application/json"
  },
  "qs": {
    "workspaceId": "{{ifempty(parameters.workspaceId, '')}}",
    "teamId": "{{ifempty(parameters.teamId, '')}}"
  },
  "response": {
    "iterate": "{{body}}",
    "output": {
      "value": "{{item.shortId}}",
      "label": "{{item.name}} ({{item.workspaceName}})"
    }
  }
}
```

**Parameters Tab** - paste [`src/rpcs/list-templates/parameters.json`](./src/rpcs/list-templates/parameters.json):
```json
[
  {
    "name": "workspaceId",
    "type": "text",
    "label": "Workspace ID",
    "required": false
  },
  {
    "name": "teamId",
    "type": "text",
    "label": "Team ID",
    "required": false
  }
]
```

### RPC: List PDF Templates

Repeat using files from [`src/rpcs/list-pdf-templates/`](./src/rpcs/list-pdf-templates/):
- **Name**: `listPdfTemplates`
- Uses same endpoint, filters results client-side by `outputFormats` containing "pdf"

### RPC: List Excel Templates

Repeat using files from [`src/rpcs/list-excel-templates/`](./src/rpcs/list-excel-templates/):
- **Name**: `listExcelTemplates`
- Uses same endpoint, filters results client-side by `outputFormats` containing "excel"

---

## Configuring Webhooks

Webhooks enable instant triggers to receive real-time events from Renderbase.

### Webhook: Document Events

1. Click **Webhooks** in the left sidebar
2. Click **+ Create a new webhook**
3. Set **Name** to `documentEvents`
4. Set **Label** to `Document Events`
5. Set **Type** to `Dedicated` (each user gets their own webhook URL)
6. Check **Attached** (Make.com will automatically register/unregister webhooks via API)
7. Select **Connection**: `oauth2`

**Configure each tab:**

**Communication Tab** - paste [`src/webhooks/document-events/communication.json`](./src/webhooks/document-events/communication.json):

This handles incoming webhook data and challenge verification:
```json
{
  "verification": {
    "condition": "{{if(body.challenge, true, false)}}",
    "respond": {
      "status": 200,
      "type": "json",
      "body": {
        "challenge": "{{body.challenge}}"
      }
    }
  },
  "output": {
    "id": "{{body.id}}",
    "type": "{{body.type}}",
    "timestamp": "{{body.timestamp}}",
    "jobId": "{{body.data.jobId}}",
    "templateId": "{{body.data.templateId}}",
    "status": "{{body.data.status}}",
    "downloadUrl": "{{body.data.downloadUrl}}"
  }
}
```

**Attach Tab** - paste [`src/webhooks/document-events/attach.json`](./src/webhooks/document-events/attach.json):

This registers the webhook URL with Renderbase when a user enables a trigger:
```json
{
  "url": "/webhook-subscriptions",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer {{connection.accessToken}}",
    "Content-Type": "application/json"
  },
  "body": {
    "url": "{{webhook.url}}",
    "events": "{{parameters.events}}",
    "name": "Make.com - {{parameters.events}}"
  },
  "response": {
    "data": {
      "webhookId": "{{body.data.id}}"
    }
  }
}
```

**Detach Tab** - paste [`src/webhooks/document-events/detach.json`](./src/webhooks/document-events/detach.json):

This unregisters the webhook when a user disables the trigger:
```json
{
  "url": "/webhook-subscriptions/{{webhook.webhookId}}",
  "method": "DELETE",
  "headers": {
    "Authorization": "Bearer {{connection.accessToken}}",
    "Content-Type": "application/json"
  }
}
```

**Parameters Tab** - paste [`src/webhooks/document-events/parameters.json`](./src/webhooks/document-events/parameters.json):

This defines the event selection UI shown to users:
```json
[
  {
    "name": "events",
    "label": "Event Types",
    "type": "select",
    "required": true,
    "multiple": true,
    "options": [
      { "label": "Document Completed", "value": "document.completed" },
      { "label": "Document Failed", "value": "document.failed" },
      { "label": "Batch Completed", "value": "batch.completed" }
    ],
    "help": "Select which events should trigger this webhook"
  }
]
```

---

## Testing the Integration

### Step 1: Test Connection

1. Open a new browser tab and go to Make.com
2. Go to **Scenarios** and create a new scenario
3. Click the empty module placeholder
4. Search for your app name ("Renderbase")
5. Select any module (e.g., "Generate PDF")
6. Click **Create a connection**
7. You'll be redirected to Renderbase to authorize the connection
8. Log in to your Renderbase account and approve the permissions
9. Verify the connection shows your email in Make.com

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
3. Select the appropriate tab (Communication, Parameters, etc.)
4. Make your changes
5. Click **Save**
6. Test the updated module in a scenario

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
- Check `refresh` block in Communication tab
- Ensure refresh endpoint returns new tokens

### Module Issues

**Team Dropdown Empty:**
- Verify the user has access to at least one team
- Check `listTeams` RPC configuration
- Test the `/api/v1/integration-api/teams` endpoint directly

**Workspace Dropdown Empty:**
- Verify a team is selected first (cascading dependency)
- Check the user has access to workspaces in the selected team
- Verify `listWorkspaces` RPC includes `teamId` parameter

**Template Dropdown Empty:**
- Verify a workspace is selected first (cascading dependency)
- Verify `templates:read` scope is granted
- Check that templates exist in the selected workspace
- For PDF/Excel dropdowns, verify templates have the correct `outputFormats`
- Test the `/api/v1/integration-api/templates?workspaceId=...` endpoint directly

**Cascading Selection Not Working:**
- Verify the `nested` property is set correctly in parameters.json
- Check that RPC URLs include the parameter reference (e.g., `?teamId={{parameters.teamId}}`)
- Clear the module cache and try again

**Document Generation Fails:**
- Check `documents:generate` scope is granted
- Verify template ID is valid (should be `shortId` format)
- Verify `teamId` and `workspaceId` are included in the request
- Check variable data format matches template requirements

### Webhook Issues

**Triggers Not Firing:**
- Verify webhook was created in Renderbase
- Check webhook URL is accessible
- Verify `webhooks:write` scope is granted
- Check Renderbase webhook delivery logs

---

## File Structure

The integration files are organized by component, with each component having separate files for each Make.com tab:

```
make-renderbase/
├── app.json                    # App metadata
├── base.json                   # Base configuration
├── package.json                # Package info
├── README.md                   # User documentation
├── DEPLOYMENT.md               # This guide
└── src/
    ├── common.json             # Shared configuration
    ├── groups.json             # Module grouping
    │
    ├── connections/
    │   └── oauth2/
    │       ├── communication.json   # OAuth flow (authorize, token, refresh, info)
    │       └── common.json          # Client ID/Secret values
    │
    ├── webhooks/
    │   └── document-events/
    │       ├── communication.json   # Incoming data processing & verification
    │       ├── attach.json          # Webhook registration API call
    │       ├── detach.json          # Webhook unregistration API call
    │       └── parameters.json      # Event type selection UI
    │
    ├── modules/
    │   ├── generate-pdf/
    │   │   ├── communication.json   # API call configuration
    │   │   ├── parameters.json      # Mappable input fields
    │   │   ├── interface.json       # Output field definitions
    │   │   ├── samples.json         # Example output data
    │   │   └── scope.json           # Required scopes
    │   │
    │   ├── generate-excel/
    │   │   └── ... (same structure)
    │   │
    │   ├── generate-batch/
    │   │   └── ... (same structure)
    │   │
    │   ├── get-document/
    │   │   └── ... (same structure)
    │   │
    │   ├── watch-document-completed/
    │   │   ├── static.json          # Static parameters (event filter)
    │   │   ├── interface.json       # Output field definitions
    │   │   └── samples.json         # Example webhook payload
    │   │
    │   ├── watch-document-failed/
    │   │   └── ... (same structure)
    │   │
    │   └── watch-batch-completed/
    │       └── ... (same structure)
    │
    └── rpcs/
        ├── list-teams.json          # RPC definition for teams
        ├── list-teams/
        │   └── communication.json   # API call for user's teams
        │
        ├── list-workspaces.json     # RPC definition for workspaces
        ├── list-workspaces/
        │   ├── communication.json   # API call filtered by teamId
        │   └── parameters.json      # teamId parameter
        │
        ├── list-templates.json      # RPC definition for templates
        ├── list-templates/
        │   ├── communication.json   # API call filtered by workspaceId
        │   └── parameters.json      # workspaceId, teamId parameters
        │
        ├── list-pdf-templates.json  # RPC definition for PDF templates
        ├── list-pdf-templates/
        │   ├── communication.json   # Filters by outputFormats containing "pdf"
        │   └── parameters.json
        │
        ├── list-excel-templates.json # RPC definition for Excel templates
        └── list-excel-templates/
            ├── communication.json   # Filters by outputFormats containing "excel"
            └── parameters.json
```

---

## Reference

### IML Variables

Make uses IML (Integromat Markup Language) for dynamic values:

| Variable | Description |
|----------|-------------|
| `{{base.baseUrl}}` | Base URL from Base configuration |
| `{{common.clientId}}` | OAuth client ID from Common Data |
| `{{connection.accessToken}}` | Current access token |
| `{{connection.refreshToken}}` | Current refresh token |
| `{{oauth.redirectUri}}` | Make's OAuth callback URL |
| `{{oauth.code}}` | Authorization code from OAuth flow |
| `{{oauth.state}}` | State parameter for CSRF protection |
| `{{oauth.scope}}` | Requested OAuth scopes |
| `{{parameters.fieldName}}` | Module input parameters |
| `{{body.fieldName}}` | API response body fields |
| `{{item.fieldName}}` | Current item in iteration |
| `{{webhook.url}}` | Make-generated webhook URL |
| `{{webhook.webhookId}}` | Stored webhook ID |

### Useful Links

- [Make Developer Hub](https://developers.make.com/custom-apps-documentation)
- [Creating Custom Apps](https://developers.make.com/custom-apps-documentation/basics/create-your-app)
- [OAuth 2.0 Connections](https://developers.make.com/custom-apps-documentation/app-components/connections/oauth2)
- [Module Types](https://developers.make.com/custom-apps-documentation/app-structure/modules)
- [IML Functions](https://developers.make.com/custom-apps-documentation/iml-functions)
- [Make Community Forum](https://community.make.com/c/custom-apps/52)
- [Renderbase API Docs](https://docs.renderbase.dev/api-reference)
