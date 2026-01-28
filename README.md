# Rynko Make.com Integration

Official Make.com (formerly Integromat) integration for Rynko - the document generation platform with unified template design for PDF and Excel documents.

## Features

### Triggers (Instant)
- **Watch Document Completed** - Triggers when a document is successfully generated
- **Watch Document Failed** - Triggers when document generation fails
- **Watch Batch Completed** - Triggers when a batch generation completes

### Actions
- **Generate PDF** - Generate a PDF document from a template
- **Generate Excel** - Generate an Excel document from a template
- **Generate Batch Documents** - Generate multiple documents from a single template with different variables

### Searches
- **Get Document Job** - Get document job details by ID
- **List Document Jobs** - List document jobs with filters

## Project Structure

```
make-rynko/
├── app.json                    # App metadata
├── base.json                   # Base URL and headers configuration
├── package.json                # Package information
├── README.md                   # This file
├── DEPLOYMENT.md               # Deployment guide
└── src/
    ├── common.json             # Shared request/response settings
    ├── groups.json             # Module grouping configuration
    ├── connections/
    │   └── oauth2.json         # OAuth 2.0 connection
    ├── webhooks/
    │   └── document-events.json # Webhook configuration
    ├── modules/
    │   ├── watch-document-completed.json
    │   ├── watch-document-failed.json
    │   ├── watch-batch-completed.json
    │   ├── generate-document.json
    │   ├── generate-batch.json
    │   └── get-document-job.json
    └── rpcs/
        ├── list-templates.json
        ├── list-pdf-templates.json
        └── list-excel-templates.json
```

## Authentication

This integration uses OAuth 2.0 with the following scopes:
- `documents:write` - Generate documents
- `documents:read` - Read document job status
- `templates:read` - Access templates
- `webhooks:read` - View webhook subscriptions
- `webhooks:write` - Create/delete webhook subscriptions
- `profile:read` - Read user profile

## API Endpoints Used

- `POST /api/v1/documents/generate` - Generate single document
- `POST /api/v1/documents/generate/batch` - Generate batch documents
- `GET /api/v1/documents/jobs` - List/search document jobs
- `GET /api/v1/documents/jobs/:id` - Get document job details
- `GET /api/v1/templates` - List templates
- `POST /api/v1/webhook-subscriptions` - Subscribe to webhooks
- `DELETE /api/v1/webhook-subscriptions/:id` - Unsubscribe

## Workspace Support

When generating documents, you can optionally specify a `workspaceId` to generate documents in a specific workspace. If not provided, documents are generated in the user's current workspace.

## Development

### Make.com App Development

1. Log in to Make.com
2. Go to **My Apps** → **Create a new app**
3. Upload configuration files from this directory
4. Configure OAuth credentials
5. Test all modules

### Testing

1. Create a test scenario
2. Add Rynko modules
3. Configure with test connection
4. Run scenarios to verify

## Deployment

See [DEPLOYMENT.md](./DEPLOYMENT.md) for detailed deployment instructions.

## Documentation

- [Deployment Guide](./DEPLOYMENT.md)
- [Make.com Developer Docs](https://www.make.com/en/api-documentation)
- [Rynko API Documentation](https://docs.rynko.dev/api)

## Support

- Email: support@rynko.dev
- Documentation: https://docs.rynko.dev
