# Waba-Connect Backend API Documentation

Generated from the backend source code in `Backend/` on 2026-07-09.

## Backend Overview

- Framework: ASP.NET Core Web API targeting `.NET 10.0`.
- Default local URL: `http://localhost:5155`.
- Main API base URL: `http://localhost:5155/api`.
- Webhook URL implemented in code: `http://localhost:5155/api/webhook/whatsapp`.
- Sample weather route: `http://localhost:5155/WeatherForecast`.
- Database: PostgreSQL through Entity Framework Core.
- Swagger is enabled only in Development.
- CORS policy currently allows any origin, method, and header.
- All controller request DTO string properties are passed through a sanitization filter that strips HTML and normalizes whitespace.
- A background hosted service checks scheduled campaigns every 1 minute and sends campaigns whose `ScheduledAt` time has arrived.

## Standard Response Shape

Most business APIs return this wrapper:

```json
{
  "success": true,
  "message": "Optional message",
  "data": {},
  "errors": null
}
```

Some WABA setup, webhook, lookup-list, and weather endpoints return raw objects, arrays, plain strings, or empty `200 OK` responses. Those differences are documented per endpoint.

## Error Handling

`ExceptionHandlingMiddleware` converts unhandled backend exceptions into JSON responses:

| Condition | HTTP status | Response behavior |
| --- | ---: | --- |
| FluentValidation validation exception | 400 | `success=false`, `message="Validation failed"`, validation messages in `errors` |
| `ArgumentException` | 400 | `success=false`, exception message |
| `UnauthorizedAccessException` | 401 | `success=false`, `message="Unauthorized"` |
| `KeyNotFoundException` | 404 | `success=false`, exception message |
| `InvalidOperationException` | 409 | `success=false`, exception message |
| Any other exception | 500 | `success=false`, generic message; development also includes exception details |

ASP.NET automatic model validation can also return normal framework `400 Bad Request` responses for invalid `[Required]` DTOs before controller logic runs.

## Common Pagination Shape

Paginated endpoints use:

```json
{
  "items": [],
  "totalCount": 50,
  "page": 1,
  "pageSize": 20,
  "totalPages": 3
}
```

## Enum Values Used by the API

Most enum parsing is case-insensitive, but responses use C# enum names.

| Field | Accepted values |
| --- | --- |
| `ContactType` / contact `type` / campaign `relationType` | `Lead`, `Customer`, `Vendor` |
| `ContactStatus` | `New`, `Active`, `Inactive`, `InProgress`, `Contacted`, `Qualified`, `Closed` |
| `ContactSource` | `WhatsApp`, `Web`, `Import`, `Manual`, `Facebook`, `Saas` |
| `TemplateCategory` | `Marketing`, `Utility`, `Authentication` |
| `TemplateType` | `Text`, `Image`, `Video`, `Document` |
| `HeaderType` | `None`, `Text`, `Image`, `Video`, `Document` |
| `TemplateStatus` | `Approved`, `Rejected`, `Pending` |
| `ScheduleType` | `Immediate`, `Scheduled` |
| `CampaignStatus` | `Draft`, `Sending`, `Sent`, `Scheduled`, `Failed`, `Cancelled` |
| `MessageStatus` | `Pending`, `Sent`, `Delivered`, `Read`, `Failed` |

Important: the older supplied documentation says campaign `relationType` can be `All`, but the current backend validator does not accept `All`. It accepts only `Lead`, `Customer`, or `Vendor`.

## Important Implementation Notes

- `POST /api/Waba/connect-app` generates a webhook URL ending in `/api/webhook/whatsapp`, which matches the implemented Meta webhook controller route.
- Delivery, read, inbound, and later failed-message events require a public HTTPS webhook URL configured in Meta. A `localhost` webhook can be called by the app itself, but Meta cannot reach it from the internet.
- `GET /api/Templates` returns a paginated wrapper, not a raw array.
- `GET /api/Contacts` returns active contacts by default because the `Contact` entity has a global query filter on `IsActive`.
- `DELETE /api/Contacts/{id}` soft-deletes a contact by setting `IsActive=false`.
- `PATCH /api/Contacts/{id}/toggle-active` currently toggles the database value, then reloads the contact using the active-only query. When toggling an active contact to inactive, the database update can succeed but the response can become a `404` because the inactive contact is hidden by the query filter.
- Campaign creation stores `relationType`, but recipient selection is based on `contactIds` and `groupIds`; the service does not currently filter selected recipients by `relationType`.
- WABA dashboard responses include the saved access token. Avoid exposing that endpoint to untrusted clients without authentication.
- No authentication or authorization middleware is configured beyond `UseAuthorization`; there are no `[Authorize]` attributes in the current controllers.

---

# Dashboard API

## Get Dashboard Summary

`GET /api/Dashboard/summary`

Returns aggregate campaign and messaging statistics for the dashboard.

### Response

Wrapped in `ApiResponse<object>`.

```json
{
  "success": true,
  "data": {
    "totalContacts": 150,
    "totalCampaigns": 12,
    "messagesSent": 5000,
    "messagesDelivered": 4900,
    "messagesRead": 4500,
    "messagesFailed": 100,
    "recentCampaigns": [
      {
        "id": 1,
        "name": "Summer Promo",
        "status": "Sent",
        "totalRecipients": 1000,
        "deliveredCount": 990
      }
    ],
    "hourlyChartData": [
      {
        "name": "00:00",
        "sent": 0,
        "errors": 0
      }
    ],
    "deliveryTrend": [
      {
        "name": "00:00",
        "value": 0
      }
    ],
    "readTrend": [
      {
        "name": "00:00",
        "value": 0
      }
    ],
    "topReadRateCampaigns": [],
    "topDeliveryRateCampaigns": [],
    "overallDeliveryRate": 98,
    "overallReadRate": 91.84
  }
}
```

### How It Works

- Counts active contacts through the EF global `IsActive` filter.
- Counts all campaigns.
- Sums campaign aggregate fields for sent, delivered, read, and failed counts.
- Builds `recentCampaigns` from the latest 5 campaigns by descending ID.
- Looks at `CampaignContacts.SentAt` values to build hourly data for the latest sending date in the database. If nothing has ever been sent, the current UTC date is used and all hourly values are zero.
- Calculates top read-rate and delivery-rate campaigns from campaigns where `TotalRecipients > 0`.
- Calculates overall delivery rate as `messagesDelivered / messagesSent * 100`.
- Calculates overall read rate as `messagesRead / messagesDelivered * 100`.

---

# Contacts API

## Contact Response Shape

```json
{
  "id": 1,
  "name": "John Doe",
  "phone": "+919876543210",
  "type": "Customer",
  "status": "New",
  "source": "Manual",
  "assignedTo": "Jane Smith",
  "isActive": true,
  "createdAt": "2026-07-09T06:30:00Z",
  "updatedAt": "2026-07-09T06:30:00Z",
  "groups": [
    {
      "id": 1,
      "name": "VIP Customers"
    }
  ]
}
```

## Get All Contacts

`GET /api/Contacts?page=1&pageSize=20&type=Lead&status=New&isActive=true&search=john&sortBy=name&sortDescending=false`

### Query Parameters

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `page` | No | `1` | Page number |
| `pageSize` | No | `20` | Number of contacts per page |
| `type` | No | None | Filters by `ContactType` if the value parses successfully |
| `status` | No | None | Filters by `ContactStatus` if the value parses successfully |
| `isActive` | No | Active only | Passing `false` returns inactive contacts using `IgnoreQueryFilters()` |
| `search` | No | None | Searches contact name and phone |
| `sortBy` | No | `id` | Supports `name` and `createdAt`; anything else sorts by ID |
| `sortDescending` | No | `false` | Controls sort direction |

### Response

Wrapped paginated contact response.

### How It Works

- Loads contacts with their group memberships.
- Active contacts are returned by default because of the EF global query filter.
- If `isActive=false`, the service explicitly ignores the global query filter and returns inactive contacts only.
- Valid `type` and `status` filters are applied case-insensitively.
- Invalid enum filter values are ignored rather than rejected.
- Search checks lowercase `Name` and raw `Phone`.
- Results are sorted, skipped, taken, and mapped to `ContactResponse`.

## Get Contact By ID

`GET /api/Contacts/{id}`

### Response

Wrapped single contact response.

### How It Works

- Loads one active contact by ID with group memberships.
- Returns `404` if the active contact does not exist.
- Inactive contacts are hidden by the global query filter and also return `404`.

## Create Contact

`POST /api/Contacts`

### Request Body

```json
{
  "name": "John Doe",
  "phone": "+919876543210",
  "type": "Customer",
  "source": "Manual",
  "assignedTo": "Jane Smith",
  "groupIds": [1, 2]
}
```

### Validation

- `name`: required, 2 to 100 characters.
- `phone`: required, valid E.164 format such as `+919499373415`.
- `type`: required, valid `ContactType`.
- `source`: required, valid `ContactSource`.
- `assignedTo`: optional, maximum 100 characters.
- `groupIds`: optional, every ID must be greater than 0.

### Response

`201 Created`, wrapped created contact response.

### How It Works

- Sanitizes DTO string fields.
- Normalizes the phone before storage by removing spaces, hyphens, and parentheses.
- Checks for an existing active contact with the same normalized phone.
- Creates the contact with default `Status=New` and `IsActive=true`.
- Adds memberships for any supplied group IDs that exist. Missing groups are silently skipped.
- Saves the contact and reloads it with groups for the response.

## Update Contact

`PUT /api/Contacts/{id}`

### Request Body

Same shape as create contact.

### Response

Wrapped updated contact response.

### How It Works

- Finds the active contact by ID.
- Returns `404` if not found.
- Normalizes the new phone.
- If the phone changed, checks that no other active contact uses that phone.
- Updates name, phone, type, source, and assigned user.
- If `groupIds` is supplied, all existing group memberships are removed and replaced with memberships for existing group IDs.
- If `groupIds` is omitted or `null`, existing group memberships stay unchanged.

## Delete Contact

`DELETE /api/Contacts/{id}`

### Response

```json
{
  "success": true,
  "message": "Contact deleted successfully."
}
```

### How It Works

- Finds the active contact by ID.
- Sets `IsActive=false`.
- Saves changes.
- The record remains in the database for history and campaign references.

## Toggle Contact Active Status

`PATCH /api/Contacts/{id}/toggle-active`

### Response

Wrapped contact response when the final state is active.

### How It Works

- Looks up the contact with `IgnoreQueryFilters()`, so active and inactive contacts can be found.
- Flips `IsActive`.
- Saves changes.
- Reloads through `GetByIdAsync`.

Implementation warning: if an active contact is toggled to inactive, reload uses the active-only query filter and may return `404` after the database update succeeds.

## Get Contact Status Lookup

`GET /api/Contacts/statuses`

Returns a raw array, not the standard wrapper.

```json
[
  { "id": "New", "name": "New" },
  { "id": "InProgress", "name": "In Progress" },
  { "id": "Contacted", "name": "Contacted" },
  { "id": "Qualified", "name": "Qualified" },
  { "id": "Closed", "name": "Closed" }
]
```

### How It Works

Returns a hard-coded list for UI dropdowns. It does not include every value from the `ContactStatus` enum.

## Get Contact Source Lookup

`GET /api/Contacts/sources`

Returns a raw array:

```json
[
  { "id": "Facebook", "name": "facebook" },
  { "id": "WhatsApp", "name": "WhatsApp" },
  { "id": "Saas", "name": "saas" }
]
```

### How It Works

Returns a hard-coded list for UI dropdowns. The backend enum also supports `Web`, `Import`, and `Manual`.

## Get Assigned Users Lookup

`GET /api/Contacts/assigned-users`

Returns a raw array:

```json
[
  { "id": "superAdmin", "name": "superAdmin" },
  { "id": "johnMicheal", "name": "John Micheal" },
  { "id": "gunaratnam", "name": "Gunaratnam" }
]
```

### How It Works

Returns a hard-coded list. There is no user table lookup in the current backend.

## Get Contact Type Lookup

`GET /api/Contacts/types`

Returns a raw array:

```json
[
  { "id": "Lead", "name": "lead" },
  { "id": "Customer", "name": "customer" }
]
```

### How It Works

Returns a hard-coded dropdown list. The backend enum also contains `Vendor`.

## Get Language Lookup

`GET /api/Contacts/languages`

Returns a raw array:

```json
[
  { "id": "en", "name": "English" },
  { "id": "ms", "name": "Malay" },
  { "id": "zh", "name": "Chinese" }
]
```

### How It Works

Returns static language options for the UI.

## Get Country Lookup

`GET /api/Contacts/countries`

Returns a raw array:

```json
[
  { "id": "India", "name": "India (+91)", "code": "IN", "dialCode": "+91" },
  { "id": "Malaysia", "name": "Malaysia (+60)", "code": "MY", "dialCode": "+60" },
  { "id": "Singapore", "name": "Singapore (+65)", "code": "SG", "dialCode": "+65" },
  { "id": "UnitedStates", "name": "United States (+1)", "code": "US", "dialCode": "+1" },
  { "id": "UnitedKingdom", "name": "United Kingdom (+44)", "code": "GB", "dialCode": "+44" }
]
```

### How It Works

Returns static country options for the UI.

---

# Contact Groups API

## Group Response Shape

```json
{
  "id": 1,
  "name": "VIP Customers",
  "description": "High value clients",
  "memberCount": 42,
  "createdAt": "2026-07-09T06:30:00Z"
}
```

## Get All Groups

`GET /api/ContactGroups?page=1&pageSize=20&search=vip`

### Query Parameters

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `page` | No | `1` | Page number |
| `pageSize` | No | `20` | Number of groups per page |
| `search` | No | None | Searches group name |

### Response

Wrapped paginated group response.

### How It Works

- Loads groups with members.
- Applies a case-insensitive name search when supplied.
- Orders by descending ID.
- Calculates `memberCount` from loaded group members.

## Get Group By ID

`GET /api/ContactGroups/{id}`

### Response

Wrapped group response.

### How It Works

- Loads the group and its members.
- Returns `404` if the group does not exist.

## Create Group

`POST /api/ContactGroups`

### Request Body

```json
{
  "name": "VIP Customers",
  "description": "High value clients"
}
```

### Validation

- `name`: required, 2 to 100 characters.
- `description`: optional, maximum 500 characters.

### Response

`201 Created`, wrapped created group response.

### How It Works

- Sanitizes request strings.
- Creates the group.
- Saves and returns the group.

## Update Group

`PUT /api/ContactGroups/{id}`

### Request Body

Same as create group.

### Response

Wrapped updated group response.

### How It Works

- Finds the group by ID.
- Returns `404` if not found.
- Updates name and description.

## Delete Group

`DELETE /api/ContactGroups/{id}`

### Response

```json
{
  "success": true,
  "message": "Group deleted successfully."
}
```

### How It Works

- Finds the group by ID.
- Removes it from the database.
- Group memberships are removed through cascade behavior.

## Add Members To Group

`POST /api/ContactGroups/{id}/members`

### Request Body

```json
{
  "contactIds": [1, 2, 3]
}
```

### Validation

- `contactIds`: required and not empty.
- Every contact ID must be greater than 0.

### Response

```json
{
  "success": true,
  "message": "Members added successfully."
}
```

### How It Works

- Confirms the group exists.
- For each supplied contact ID:
  - skips it if that contact is already in the group;
  - checks the active contact exists;
  - adds a `ContactGroupMember` row if valid.
- Invalid or inactive contact IDs are skipped.

## Remove Members From Group

`DELETE /api/ContactGroups/{id}/members`

### Request Body

```json
{
  "contactIds": [1, 2]
}
```

### Response

```json
{
  "success": true,
  "message": "Members removed successfully."
}
```

### How It Works

- Finds membership rows matching the group ID and supplied contact IDs.
- Removes matching rows.
- Does not separately validate that the group exists.

---

# Templates API

## Template Response Shape

```json
{
  "id": 1,
  "name": "camp_platinum_credit_card_1",
  "language": "en",
  "category": "Marketing",
  "templateType": "Text",
  "status": "Approved",
  "bodyText": "Hello {{1}}, apply today.",
  "headerType": "None",
  "headerContent": null,
  "footerText": null,
  "whatsAppTemplateId": "12345",
  "createdAt": "2026-07-09T06:30:00Z",
  "updatedAt": "2026-07-09T06:30:00Z",
  "variables": [
    {
      "position": 1,
      "sampleValue": "John",
      "description": "Customer name"
    }
  ]
}
```

## Get All Templates

`GET /api/Templates?page=1&pageSize=20&status=Approved&category=Marketing&search=credit`

### Query Parameters

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `page` | No | `1` | Page number |
| `pageSize` | No | `20` | Number of templates per page |
| `status` | No | None | Filters by `TemplateStatus` if parseable |
| `category` | No | None | Filters by `TemplateCategory` if parseable |
| `search` | No | None | Searches template name |

### Response

Wrapped paginated template response.

### How It Works

- Loads templates with their variables.
- Applies parseable status and category filters.
- Applies name search.
- Orders by descending ID.
- Maps each template to `TemplateResponse`.

## Get Template By ID

`GET /api/Templates/{id}`

### Response

Wrapped template response.

### How It Works

- Loads one template with variables.
- Returns `404` if not found.

## Create Template

`POST /api/Templates`

### Request Body

```json
{
  "name": "camp_platinum_credit_card_1",
  "language": "en",
  "category": "Marketing",
  "templateType": "Text",
  "bodyText": "Hello {{1}}, apply today.",
  "headerType": "None",
  "headerContent": null,
  "footerText": null,
  "variables": [
    {
      "position": 1,
      "sampleValue": "John",
      "description": "Customer name"
    }
  ]
}
```

### Validation

- `name`: required, 2 to 100 characters, lowercase letters/numbers/underscores only, must start with a letter.
- `language`: required, 2 to 10 characters.
- `category`: required, valid `TemplateCategory`.
- `templateType`: valid `TemplateType`.
- `bodyText`: required, maximum 1024 characters.
- `headerContent`: optional, maximum 500 characters.
- `footerText`: optional, maximum 60 characters.
- Each variable `position` must be greater than 0.
- Each variable `sampleValue` must be at most 200 characters.

### Response

`201 Created`, wrapped created template response.

### How It Works

- Ensures no active template with the same name already exists.
- Creates a local template.
- Local templates default to `Pending` unless later updated by sync.
- Adds template variable rows if supplied.

## Update Template

`PUT /api/Templates/{id}`

### Request Body

Same as create template.

### Response

Wrapped updated template response.

### How It Works

- Finds the template by ID.
- Checks name uniqueness if the name changed.
- Updates template fields.
- Removes all existing template variables.
- Adds the supplied variable list as the new full set.

## Delete Template

`DELETE /api/Templates/{id}`

### Response

```json
{
  "success": true,
  "message": "Template deleted successfully."
}
```

### How It Works

- Finds the template by ID.
- Checks whether any campaign references this template.
- If in use, returns `409 Conflict`.
- If not in use, removes the template.

## Sync Templates From WhatsApp

`POST /api/Templates/sync`

### Response

```json
{
  "success": true,
  "message": "Successfully synced templates. Added/Updated: 3"
}
```

### How It Works

- Calls `WhatsAppCloudApiService.GetTemplatesAsync()`.
- That service reads the active WABA config, first phone number, and business account ID.
- It calls Meta Graph API `/{businessAccountId}/message_templates?limit=100`.
- For each returned template:
  - maps Meta statuses to `Approved`, `Rejected`, or `Pending`;
  - maps category to the local enum, defaulting to `Marketing` if parsing fails;
  - inserts by template name if not found;
  - otherwise updates WhatsApp template ID, status, and body text.
- The count in the message is the number of newly inserted templates, not every updated template.
- If the WhatsApp API call fails, the service logs and returns an empty list.

## Preview Template

`POST /api/Templates/{id}/preview`

### Request Body

```json
{
  "1": "John",
  "2": "Platinum Card"
}
```

You can also send `null` or omit values to use saved sample values.

### Response

```json
{
  "success": true,
  "data": {
    "id": 1,
    "name": "camp_platinum_credit_card_1",
    "previewText": "Hello John, apply for Platinum Card today."
  }
}
```

### How It Works

- Finds the template by ID.
- Starts with `BodyText`.
- If request variables are supplied, replaces `{{key}}` with the supplied value.
- If no variables are supplied, loads saved `TemplateVariable` rows and replaces `{{position}}` with `sampleValue` or `[position]`.

---

# Campaigns API

## Campaign Response Shape

```json
{
  "id": 1,
  "name": "Summer Promo",
  "templateName": "camp_platinum_credit_card_1",
  "relationType": "Lead",
  "scheduleType": "Immediate",
  "scheduledAt": null,
  "status": "Sent",
  "totalRecipients": 500,
  "deliveredCount": 490,
  "readCount": 400,
  "failedCount": 10,
  "createdBy": null,
  "createdAt": "2026-07-09T06:30:00Z",
  "updatedAt": "2026-07-09T06:30:00Z"
}
```

## Get All Campaigns

`GET /api/Campaigns?page=1&pageSize=20&status=Sent&search=summer`

### Query Parameters

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `page` | No | `1` | Page number |
| `pageSize` | No | `20` | Number of campaigns per page |
| `status` | No | None | Filters by `CampaignStatus` if parseable |
| `search` | No | None | Searches campaign name |

### Response

Wrapped paginated campaign response.

### How It Works

- Loads campaigns with their template.
- Applies a valid status filter when supplied.
- Applies name search.
- Orders by descending ID.

## Get Campaign By ID

`GET /api/Campaigns/{id}`

### Response

Wrapped campaign detail response:

```json
{
  "success": true,
  "data": {
    "id": 1,
    "name": "Summer Promo",
    "templateName": "camp_platinum_credit_card_1",
    "relationType": "Lead",
    "scheduleType": "Immediate",
    "scheduledAt": null,
    "status": "Sent",
    "totalRecipients": 2,
    "deliveredCount": 1,
    "readCount": 1,
    "failedCount": 0,
    "createdBy": null,
    "createdAt": "2026-07-09T06:30:00Z",
    "updatedAt": "2026-07-09T06:30:00Z",
    "recipients": [
      {
        "contactId": 1,
        "contactName": "John Doe",
        "phone": "+919876543210",
        "status": "Read",
        "sentAt": "2026-07-09T06:30:00Z",
        "deliveredAt": "2026-07-09T06:30:05Z",
        "readAt": "2026-07-09T06:35:00Z",
        "errorMessage": null
      }
    ]
  }
}
```

### How It Works

- Loads the campaign with template, campaign contacts, and recipient contact records.
- Returns full recipient status list without pagination.
- Returns `404` if campaign is missing.

## Create And Send Or Schedule Campaign

`POST /api/Campaigns`

### Request Body

```json
{
  "name": "My First Campaign",
  "templateId": 1,
  "relationType": "Lead",
  "scheduleType": "Immediate",
  "scheduledAt": null,
  "contactIds": [1, 2],
  "groupIds": [1],
  "variables": [
    {
      "variableName": "1",
      "variableValue": "John",
      "mergeField": null
    }
  ]
}
```

For merge fields:

```json
{
  "variableName": "1",
  "variableValue": null,
  "mergeField": "@name"
}
```

Supported merge fields in sending logic are `@name` and `@phone`.

### Validation

- `name`: required, 2 to 200 characters.
- `templateId`: required and greater than 0.
- `relationType`: required and must be a `ContactType`: `Lead`, `Customer`, or `Vendor`.
- `scheduleType`: required and must be `Immediate` or `Scheduled`.
- `scheduledAt`: required and must be in the future when `scheduleType` is `Scheduled`.
- At least one contact ID or group ID must be selected.
- Each variable must include `variableName`, maximum 50 characters.

### Response

`201 Created`, wrapped created campaign response.

### How It Works

- Validates the template exists.
- Requires the template status to be `Approved`.
- Combines direct `contactIds` and contacts from all supplied `groupIds` into a distinct recipient set.
- Creates a `Campaign` with:
  - `Status=Sending` for immediate campaigns;
  - `Status=Scheduled` for scheduled campaigns.
- Stores campaign variables.
- Creates one `CampaignContact` row per selected recipient with `Status=Pending`.
- Saves the campaign.
- If `scheduleType=Immediate`, starts background sending with `Task.Run`.
- Returns the campaign immediately; message sending continues in the background.

### Message Sending Flow

For each campaign recipient:

- Loads campaign template, variables, and contact data.
- Builds template variables.
- If `mergeField` is `@name`, uses contact name.
- If `mergeField` is `@phone`, uses contact phone.
- Calls WhatsApp Cloud API through `WhatsAppCloudApiService.SendTemplateMessageAsync`.
- On success, stores the returned WhatsApp message ID.
- On failure, marks that recipient `Failed` and increments campaign `FailedCount`.
- Waits 100 ms between recipients as a simple rate-limit delay.
- After looping through recipients, sets campaign status to `Sent`.

Delivery, read, and final failed status updates are expected to come later through the webhook.

## Delete Campaign

`DELETE /api/Campaigns/{id}`

### Response

```json
{
  "success": true,
  "message": "Campaign deleted successfully."
}
```

### How It Works

- Finds the campaign by ID.
- Allows deletion only for campaigns with `Draft`, `Failed`, or `Cancelled` status.
- Returns `409 Conflict` for `Sending`, `Sent`, or `Scheduled` campaigns.

## Cancel Scheduled Campaign

`POST /api/Campaigns/{id}/cancel`

### Response

Wrapped campaign response with message:

```json
{
  "success": true,
  "message": "Campaign cancelled successfully.",
  "data": {
    "id": 1,
    "status": "Cancelled"
  }
}
```

### How It Works

- Finds the campaign by ID.
- Allows cancellation only when status is `Scheduled`.
- Sets status to `Cancelled`.

## Get Campaign Recipients

`GET /api/Campaigns/{id}/recipients?page=1&pageSize=20`

### Query Parameters

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `page` | No | `1` | Page number |
| `pageSize` | No | `20` | Recipients per page |

### Response

Wrapped paginated recipient response.

### How It Works

- Queries `CampaignContacts` for the supplied campaign ID.
- Includes the related contact.
- Orders by `CampaignContact.Id`.
- Does not separately check that the campaign exists; a missing campaign with no rows returns an empty page.

## Scheduled Campaign Processing

No public route starts this directly. It is run by `CampaignSchedulerService`.

### How It Works

- Every 1 minute, the background service creates a scope and calls `ProcessScheduledCampaignsAsync`.
- It finds campaigns where `Status=Scheduled` and `ScheduledAt <= DateTime.UtcNow`.
- Each due campaign is marked `Sending`.
- Message sending is launched in the background using the same sending flow as immediate campaigns.

---

# WABA Integration API

These endpoints mostly return raw response objects, not the standard `ApiResponse` wrapper.

## Connect Facebook App

`POST /api/Waba/connect-app`

### Request Body

```json
{
  "facebookAppId": "123456789",
  "facebookAppSecret": "app-secret"
}
```

### Validation

Both fields are `[Required]`.

### Success Response

```json
{
  "message": "Facebook App connected successfully.",
  "webhookUrl": "http://localhost:5155/api/webhook/whatsapp",
  "verifyToken": "waba_verify_token_abc123..."
}
```

### How It Works

- Validates the Facebook app credentials through `MetaGraphService.ValidateAppAsync`.
- Mock, test, demo, or very short values are accepted for local/demo usage.
- Real credentials are checked with Meta Graph API client credentials flow.
- Generates a random verify token.
- Builds a webhook URL from request scheme and host using `/api/webhook/whatsapp`.
- Saves a single WABA configuration row with `Connected=false`.
- Returns the generated webhook URL and verify token for Meta setup.

## Configure WhatsApp Business Account

`POST /api/Waba/configure`

### Request Body

```json
{
  "wabaId": "1234567890",
  "accessToken": "EAAB..."
}
```

### Validation

Both fields are `[Required]`.

### Success Response

```json
{
  "message": "WhatsApp Business Account configured successfully."
}
```

### How It Works

- Requires `connect-app` to have already saved a config.
- Fetches business details with the WABA ID and access token.
- Saves WABA ID, access token, and `Connected=true`.
- Saves business details.
- Fetches WABA phone numbers and replaces locally saved phone number rows.
- Syncs message templates from WhatsApp into local templates.
- Runs an initial health check and stores a `HealthLog`.

## Get WABA Dashboard

`GET /api/Waba/dashboard`

### Response

If nothing is connected:

```json
{
  "isConnected": false
}
```

If app is connected but WABA is not configured:

```json
{
  "isConnected": false,
  "facebookAppId": "123456789",
  "webhookUrl": "http://localhost:5155/webhook",
  "verifyToken": "waba_verify_token_abc123..."
}
```

If fully connected:

```json
{
  "isConnected": true,
  "facebookAppId": "123456789",
  "wabaId": "987654321",
  "accessToken": "EAAB...",
  "webhookUrl": "http://localhost:5155/webhook",
  "verifyToken": "waba_verify_token_abc123...",
  "business": {
    "id": 1,
    "businessId": "biz_123",
    "businessName": "Business Name",
    "timezone": "UTC",
    "status": "APPROVED"
  },
  "phoneNumbers": [],
  "templates": [],
  "latestHealthLog": null,
  "healthLogs": [],
  "tokenInfo": {
    "appId": "123456789",
    "application": "WABA Facebook App",
    "type": "SYSTEM_USER",
    "isValid": true,
    "expiresAt": null,
    "issuedAt": "2026-07-09T06:30:00Z",
    "scopes": [
      "whatsapp_business_management",
      "whatsapp_business_messaging"
    ]
  }
}
```

### How It Works

- Reads the single saved WABA configuration.
- If not configured, returns connection setup details.
- If connected:
  - loads business row;
  - loads phone numbers;
  - loads local templates;
  - loads latest health log and recent 10 health logs;
  - calls Meta debug token API to collect token validity, app, type, issue/expiry times, and scopes;
  - falls back to default token info if debug parsing fails.

Security note: this endpoint returns the saved access token in the current backend.

## Send Test Template Message

`POST /api/Waba/send-message`

### Request Body

```json
{
  "recipientNumber": "919876543210",
  "templateName": "hello_world",
  "languageCode": "en_US"
}
```

`templateName` defaults to `hello_world`. `languageCode` defaults to `en_US`.

### Success Response

```json
{
  "message": "Test message sent successfully to 919876543210 using template 'hello_world'."
}
```

### How It Works

- Requires an active connected WABA configuration.
- Loads the first saved WABA phone number.
- Calls Meta Graph API `/{phoneNumberId}/messages` with a template-message payload.
- Uses the configured access token.
- Returns success when Meta returns a successful status code.
- Does not check the local templates table before sending.

## Verify Webhook

`POST /api/Waba/verify-webhook`

### Request Body

```json
{
  "verifyToken": "waba_verify_token_abc123..."
}
```

### Success Response

```json
{
  "message": "Webhook verified successfully.",
  "verified": true
}
```

### How It Works

- Loads saved WABA config.
- Builds a simulated Meta verification GET request using `hub.mode=subscribe`, the supplied token, and a generated challenge.
- Sends the request to the stored `WebhookUrl`.
- If the HTTP call returns success and the response body equals the challenge, verification succeeds.
- If the HTTP request fails or is blocked, it falls back to checking whether the supplied token equals the saved verify token.

## Disconnect WABA

`POST /api/Waba/disconnect`

### Success Response

```json
{
  "message": "WhatsApp Business Account disconnected successfully. Configuration wiped."
}
```

### How It Works

- Deletes the saved WABA configuration row.
- Clears saved business details.
- Clears saved WABA phone numbers.
- Clears health logs.
- Does not delete contacts, contact groups, templates, campaigns, or campaign delivery records.

## Refresh WABA Data

`POST /api/Waba/refresh`

### Response

Raw dashboard object, same shape as `GET /api/Waba/dashboard`.

### How It Works

- Requires an active connected WABA configuration.
- Attempts to refresh:
  - business details;
  - phone numbers;
  - templates.
- Logs refresh exceptions and continues.
- Runs a fresh health check.
- Returns current dashboard data.

---

# Chat API

Chat is database-backed. Conversations are based on contacts from the `Contacts` table, messages are stored in `ChatMessages`, campaign sends create outgoing chat messages, and WhatsApp webhooks update message status or create incoming messages.

## Get WABA Sender Accounts

`GET /api/Chat/accounts`

Returns WABA phone numbers available for sending chat messages.

### Response

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "phoneNumber": "+60108052877",
      "phoneNumberId": "1098234857203",
      "displayName": "RMA Support",
      "verifiedName": "RMA Global Enterprise Inc",
      "quality": "GREEN",
      "status": "APPROVED"
    }
  ]
}
```

## Get Conversations

`GET /api/Chat/conversations?search=tushar&filter=Unread%20Chats`

### Query Parameters

| Name | Required | Description |
| --- | --- | --- |
| `search` | No | Searches contact name and phone |
| `filter` | No | Use `Unread Chats` to return only conversations with unread incoming messages |

### How It Works

- Ensures every active contact has a `ChatConversation`.
- Loads conversations with contact and WABA phone number data.
- Sorts by latest message time, then contact name.
- Returns only database-backed contacts and message previews.

## Get Conversation By ID

`GET /api/Chat/conversations/{id}`

Returns one conversation and its contact metadata.

## Get Conversation Messages

`GET /api/Chat/conversations/{id}/messages`

Returns ordered chat messages. Also resets the conversation unread count to `0`.

## Send Manual Chat Message

`POST /api/Chat/conversations/{id}/messages`

### Request Body

```json
{
  "text": "Hi, how can I help?",
  "fromPhoneNumberId": "1098234857203"
}
```

### How It Works

- Creates an outgoing `ChatMessage` with `Pending` status.
- Sends a WhatsApp text message through Meta Cloud API using the selected WABA phone number.
- Stores the returned WhatsApp message ID when Meta accepts the message.
- Leaves the message `Pending` until Meta sends a webhook status update.
- Marks the message as `Failed` and stores Meta's error message when the immediate send is rejected or a failed-status webhook arrives later.

Important: Meta only allows free-form text replies inside an allowed WhatsApp conversation window. Outside that window, a template message is required and Meta can reject manual text sends.
Important: if the saved webhook URL is `localhost`, Meta can accept a message and return a `wamid`, but the app will not receive delivery, read, incoming reply, or late failed-message callbacks until the webhook is exposed through a public HTTPS URL and configured in Meta.

---

# WhatsApp Webhook API

## Meta Webhook Verification

`GET /api/webhook/whatsapp?hub.mode=subscribe&hub.challenge=12345&hub.verify_token=waba_verify_token_abc123`

### Success Response

Plain text body:

```text
12345
```

### Failure Response

`403 Forbidden`.

### How It Works

- Reads the saved WABA configuration from the database.
- Checks `hub.mode == "subscribe"`.
- Compares `hub.verify_token` with the saved verify token.
- If valid, returns the raw `hub.challenge` string exactly as Meta expects.
- If invalid, returns forbidden.

## Receive WhatsApp Status Webhook

`POST /api/webhook/whatsapp`

### Request Body

The body follows Meta's WhatsApp webhook payload. Status updates and incoming text messages are processed.

```json
{
  "object": "whatsapp_business_account",
  "entry": [
    {
      "id": "123",
      "changes": [
        {
          "field": "messages",
          "value": {
            "messaging_product": "whatsapp",
            "metadata": {
              "display_phone_number": "15550192834",
              "phone_number_id": "1098234857203"
            },
            "statuses": [
              {
                "id": "wamid.xxx",
                "status": "delivered",
                "timestamp": "1720000000",
                "recipient_id": "919876543210",
                "errors": null
              }
            ]
          }
        }
      ]
    }
  ]
}
```

### Response

Empty `200 OK`.

### How It Works

- Immediately starts background processing and returns `200 OK`.
- Loops through entries, changes, and statuses.
- For each status, finds `CampaignContact` by `WhatsAppMessageId`.
- Unknown message IDs are logged and ignored.
- Status mapping:
  - `sent`: if currently `Pending`, set `Sent` and `SentAt`.
  - `delivered`: if `Pending` or `Sent`, set `Delivered`, `SentAt`, and `DeliveredAt`.
  - `read`: if `Pending`, `Sent`, or `Delivered`, set `Read`, `SentAt`, `DeliveredAt`, and `ReadAt`.
  - `failed`: set `Failed` and store the first error title or a default error.
- Recalculates campaign aggregate totals after each status update.

---

# Weather Sample API

## Get Weather Forecast

`GET /WeatherForecast`

This route is outside the `/api` prefix.

### Response

Raw array:

```json
[
  {
    "date": "2026-07-10",
    "temperatureC": 26,
    "temperatureF": 78,
    "summary": "Warm"
  }
]
```

### How It Works

- Returns 5 random forecast objects.
- Uses the default ASP.NET sample controller.
- Not used by the WABA campaign workflow.

---

# Data Model Summary

## Contacts

- `Contact` has unique `Phone`.
- `Contact` has a global query filter where `IsActive=true`.
- Contacts can belong to many groups through `ContactGroupMember`.
- Contacts can appear in many campaigns through `CampaignContact`.

## Contact Groups

- `ContactGroup` stores name, description, and created timestamp.
- `ContactGroupMember` has a unique index on `(ContactId, GroupId)`.
- Deleting a group cascades to group memberships.

## Templates

- `Template.Name` is unique.
- Template enum values are stored as strings.
- Template variables are deleted when the template is deleted.

## Campaigns

- `Campaign` stores aggregate counts and scheduling state.
- `CampaignContact` stores one row per recipient and tracks WhatsApp message ID, status, timestamps, and errors.
- `CampaignContact` has a unique index on `(CampaignId, ContactId)`.
- Campaign variables can use static values or merge fields.

## WABA Setup

- Only one `WabaConfiguration` is actively used.
- Phone numbers are replaced on each save/sync.
- Health checks create `HealthLog` rows with `AVAILABLE`, `PARTIAL`, or `UNAVAILABLE`.

---

# Typical Frontend Flow

1. Connect the Facebook app with `POST /api/Waba/connect-app`.
2. Configure WABA with `POST /api/Waba/configure`.
3. Confirm setup with `GET /api/Waba/dashboard`.
4. Sync or view templates with `POST /api/Templates/sync` and `GET /api/Templates`.
5. Create contacts and groups with `POST /api/Contacts` and `POST /api/ContactGroups`.
6. Create a campaign with `POST /api/Campaigns`.
7. Poll campaign details or recipients with `GET /api/Campaigns/{id}` or `GET /api/Campaigns/{id}/recipients`.
8. Meta posts delivery/read updates to `POST /api/webhook/whatsapp`.
9. Dashboard stats update from campaign and campaign-recipient records.
