# Make.com Review Scenarios Guide

This guide helps you quickly create the test scenarios required for Make.com app review.

**API Documentation Link:** `https://docs.renderbase.dev/api-reference`

---

## Prerequisites

Before creating scenarios, ensure you have:
1. A Renderbase account with at least one published template
2. Connected your Renderbase account to Make.com via OAuth
3. Note your template ID (found in Renderbase dashboard → Templates → click template → copy ID from URL)

---

## Scenario 1: Generate PDF

**Purpose:** Demonstrate PDF document generation

### Steps:
1. Click **Create a new scenario**
2. Click **+** → Search "Schedule" → Select **Schedule: Run once**
3. Click **+** → Search "Renderbase" → Select **Generate PDF**
4. Configure the module:
   - **Connection:** Select your Renderbase connection
   - **Template ID:** Select a template from the dropdown (or paste ID)
   - **Variables:** Add any required variables for your template
   ```json
   {
     "customerName": "Acme Corp",
     "invoiceNumber": "INV-TEST-001",
     "amount": 1500.00
   }
   ```
5. Click **OK**
6. Click **Run once** to execute
7. Verify the output shows `jobId` and `status`
8. **Save** the scenario
9. Copy the scenario URL (should match: `https://eu1.make.com/{org}/scenarios/{id}/edit`)

---

## Scenario 2: Generate Excel

**Purpose:** Demonstrate Excel document generation

### Steps:
1. Click **Create a new scenario**
2. Click **+** → Search "Schedule" → Select **Schedule: Run once**
3. Click **+** → Search "Renderbase" → Select **Generate Excel**
4. Configure the module:
   - **Connection:** Select your Renderbase connection
   - **Template ID:** Select an Excel-compatible template
   - **Variables:** Add required variables
   ```json
   {
     "reportTitle": "Monthly Sales Report",
     "month": "January 2026",
     "data": [
       {"product": "Widget A", "sales": 150},
       {"product": "Widget B", "sales": 230}
     ]
   }
   ```
5. Click **OK**
6. Click **Run once** to execute
7. Verify the output shows `jobId` and `status`
8. **Save** the scenario
9. Copy the scenario URL

---

## Scenario 3: Get Document

**Purpose:** Demonstrate retrieving document job status and download URL

### Steps:
1. Click **Create a new scenario**
2. Click **+** → Search "Schedule" → Select **Schedule: Run once**
3. Click **+** → Search "Renderbase" → Select **Generate PDF**
4. Configure with valid template and variables (same as Scenario 1)
5. Click **+** → Search "Renderbase" → Select **Get Document**
6. Configure the module:
   - **Connection:** Select your Renderbase connection
   - **Job ID:** Map from previous module: `{{1.jobId}}`
7. Click **OK**
8. Click **Run once** to execute
9. Verify the output shows job details including `status` and `downloadUrl`
10. **Save** the scenario
11. Copy the scenario URL

---

## Scenario 4: Watch Document Completed

**Purpose:** Demonstrate instant webhook trigger when document generation succeeds

### Steps:
1. Click **Create a new scenario**
2. Click **+** → Search "Renderbase" → Select **Watch Document Completed**
3. Configure the module:
   - **Connection:** Select your Renderbase connection
   - **Webhook:** Click **Add** to create a new webhook
   - Name it: "Document Completed - Review Test"
4. Click **OK**
5. The module will show "Waiting for data..." - this is expected
6. **In a new browser tab**, trigger a document generation:
   - Go to Renderbase dashboard
   - Generate a test document, OR
   - Run one of your other Make.com scenarios (Scenario 1 or 2)
7. Return to Make.com - the trigger should receive the event
8. Verify the output shows document completion data
9. **Save** the scenario
10. Copy the scenario URL

**Alternative Setup (if webhook doesn't trigger):**
1. Add a second module: **+** → Search "Tools" → Select **Set variable**
2. Configure: Variable name: `documentId`, Value: `{{1.jobId}}`
3. This ensures the scenario has visible output for reviewers

---

## Scenario 5: Watch Document Failed

**Purpose:** Demonstrate instant webhook trigger when document generation fails

### Steps:
1. Click **Create a new scenario**
2. Click **+** → Search "Renderbase" → Select **Watch Document Failed**
3. Configure the module:
   - **Connection:** Select your Renderbase connection
   - **Webhook:** Click **Add** to create a new webhook
   - Name it: "Document Failed - Review Test"
4. Click **OK**
5. The module will show "Waiting for data..."
6. **To trigger a failure**, you need to cause a document generation error:
   - Use a template with a required variable, but don't provide it
   - Use invalid data types for variables
   - Reference a non-existent image/asset
7. When the failure event is received, verify the output shows error details
8. **Save** the scenario
9. Copy the scenario URL

**Tip:** If you can't easily trigger a real failure:
- Create a simple scenario that has run at least once
- The webhook will be registered and ready to receive events
- Note in your submission that the webhook is configured and tested

---

## Scenario 6: API Error Handling

**Purpose:** Demonstrate proper error handling and user-friendly error messages

### Steps:
1. Click **Create a new scenario**
2. Click **+** → Search "Schedule" → Select **Schedule: Run once**
3. Click **+** → Search "Renderbase" → Select **Generate PDF**
4. Configure the module with **intentionally invalid data**:
   - **Connection:** Select your Renderbase connection
   - **Template ID:** Enter an invalid ID: `invalid-template-12345`
   - **Variables:** Leave empty or add dummy data
5. Click **OK**
6. Click **Run once** to execute
7. The scenario should fail with a clear error message like:
   ```
   [404] Template not found: invalid-template-12345
   ```
8. **Save** the scenario (even though it fails - this is intentional)
9. Copy the scenario URL

**What reviewers are looking for:**
- Error messages are clear and actionable
- HTTP status codes are properly returned
- Users can understand what went wrong

---

## Summary: URLs to Submit

After creating all scenarios, collect these URLs:

| Field | URL |
|-------|-----|
| API Documentation | `https://docs.renderbase.dev/api-reference` |
| Generate PDF | `https://eu1.make.com/XXXXX/scenarios/XXXXX/edit` |
| Generate Excel | `https://eu1.make.com/XXXXX/scenarios/XXXXX/edit` |
| Get Document | `https://eu1.make.com/XXXXX/scenarios/XXXXX/edit` |
| Watch Document Completed | `https://eu1.make.com/XXXXX/scenarios/XXXXX/edit` |
| Watch Document Failed | `https://eu1.make.com/XXXXX/scenarios/XXXXX/edit` |
| API Error Scenario | `https://eu1.make.com/XXXXX/scenarios/XXXXX/edit` |

---

## Checklist Before Submission

- [ ] All modules are set to **Visible** status
- [ ] All modules have proper labels and descriptions
- [ ] Connection uses OAuth 2.0 with proper error handling
- [ ] Error messages are user-friendly
- [ ] All scenarios have been executed at least once
- [ ] Webhook triggers are registered and ready to receive events

---

## Troubleshooting

### "Template not found" when selecting template
- Ensure you're connected with the correct Renderbase account
- Check that you have at least one published template

### Webhook not receiving events
- Verify the webhook URL is registered in Renderbase
- Check that document generation is completing (not stuck in queue)
- Try generating a document directly from the Renderbase dashboard

### OAuth connection fails
- Clear browser cookies and try reconnecting
- Verify the OAuth client is properly configured in Renderbase

---

## Support

- **Renderbase Support:** support@renderbase.dev
- **Documentation:** https://docs.renderbase.dev
- **Make.com Integration Docs:** https://docs.renderbase.dev/integrations/no-code#make-integration
