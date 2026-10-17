# SimpleDock MCP Server

[![Listed on mcpservers.org](https://mcpservers.org/badge.svg)](https://mcpservers.org/servers/replenysh/simpledock-mcp)

<img src="assets/logo.png" width="120" alt="SimpleDock logo">

The official remote [Model Context Protocol](https://modelcontextprotocol.io) server for [SimpleDock](https://simpledock.ai): warehouse dock appointment scheduling. With it, AI assistants can find open dock slots, book, reschedule, approve and cancel appointments, check trucks in, and read schedules and analytics.

This repo has documentation only. The server is hosted by SimpleDock, so you don't need to install or run anything.

| | |
|---|---|
| Endpoint | `https://simpledock.ai/api/v1/mcp` |
| Transport | Streamable HTTP |
| Auth | OAuth 2.1 (Dynamic Client Registration + PKCE). Your client opens a SimpleDock sign-in page. |
| Registry name | `ai.simpledock/simpledock` (official MCP Registry) |
| Tools | 40 |
| Docs | https://simpledock.ai/developers/mcp |

You need a SimpleDock account; the free plan works. Sign up at https://simpledock.ai.

## Setup

### Claude (claude.ai / Claude Desktop)
Settings → Connectors → **Add custom connector** → URL `https://simpledock.ai/api/v1/mcp` → Connect, then sign in to SimpleDock.

Claude Code: `claude mcp add --transport http simpledock https://simpledock.ai/api/v1/mcp`

### ChatGPT
Settings → Apps & Connectors → Advanced → enable Developer mode → **Create** → MCP server URL `https://simpledock.ai/api/v1/mcp`, Authentication **OAuth** → sign in to SimpleDock.

### Cursor
`~/.cursor/mcp.json`:
```json
{
  "mcpServers": {
    "simpledock": { "url": "https://simpledock.ai/api/v1/mcp" }
  }
}
```

### VS Code (GitHub Copilot)
`.vscode/mcp.json`:
```json
{
  "servers": {
    "simpledock": { "type": "http", "url": "https://simpledock.ai/api/v1/mcp" }
  }
}
```

### Cline
Open MCP Servers → Remote Servers, or edit `cline_mcp_settings.json`:
```json
{
  "mcpServers": {
    "simpledock": {
      "url": "https://simpledock.ai/api/v1/mcp",
      "type": "streamableHttp"
    }
  }
}
```
Cline opens a browser window for the SimpleDock OAuth sign-in on first use.

### Other clients
Any client that supports remote Streamable HTTP servers with OAuth works. For clients that only support stdio, use `npx mcp-remote https://simpledock.ai/api/v1/mcp`.

## Suggested workflow
Start with `list_locations`, then `list_appointment_types`, then `get_availability` for a date, then `create_appointment`.

## Tools

| Tool | Description | Access |
|---|---|---|
| `list_locations` | List all dock locations in the organization. | read-only |
| `list_doors` | List the dock doors at a location. | read-only |
| `list_appointment_types` | List a location's appointment types with their custom field definitions. Call this first: create_appointment and get_availability both need an appointmentTypeId, and bookings must satisfy each type's required fields. | read-only |
| `get_availability` | List bookable time slots for a location, appointment type, and date (YYYY-MM-DD). | read-only |
| `list_appointments` | List appointments, optionally filtered by location, status, and start-time range. Returns up to `limit` rows plus a `nextCursor` for the following page. | read-only |
| `get_appointment` | Fetch a single appointment by id. | read-only |
| `get_analytics` | Per-location appointment summary (counts by status and type) over a date range. Defaults to the last 30 days; the range may not exceed 366 days. | read-only |
| `create_appointment` | Create (book) a confirmed appointment at a location. | write |
| `update_appointment_details` | Edit an appointment's carrier/driver/reference fields (no time change). Omitted fields are left unchanged. | write |
| `reschedule_appointment` | Move a confirmed appointment to a new start time and optional door. | write |
| `set_appointment_status` | Transition an appointment's status (one of: CONFIRMED, COMPLETED, NO_SHOW). COMPLETED and NO_SHOW are dock outcomes and need appointments:receive; CONFIRMED needs appointments:write. Use cancel_appointment to cancel. | write |
| `approve_appointment` | Approve a pending appointment, optionally moving it to a new start time. | write |
| `reject_appointment` | Reject a pending appointment with an optional reason. | write |
| `check_in_appointment` | Record the driver's own check-in update on a confirmed appointment: ON_TRACK, RUNNING_LATE (with estimatedDelay minutes), or CANCELLING. To record that the truck is at the dock, use mark_arrived. | write |
| `mark_arrived` | Record that a truck has physically arrived at the dock (e.g. "the Swift driver just pulled in"). Starts the dwell-time clock, which ends when the appointment is marked COMPLETED. Pass arrived: false to undo a mistaken arrival. Not the same as check_in_appointment, which records the driver's own on-track or running-late update before arrival. | write |
| `cancel_appointment` | Cancel an appointment. | write |
| `get_location_setup` | Everything about how a location is set up, in one call: its name, address, and timezone; scheduling settings (start-time spacing, approval, door mode); weekly hours; appointment types (including inactive ones); doors with their type limits and own hours; and upcoming closures. Read this before changing setup or when diagnosing missing times. | read-only |
| `set_up_location` | Set up a facility in one call: scheduling settings, weekly hours, appointment types, and doors, applied together (a bad entry changes nothing). Every part is optional. Types and doors are matched by name: a name that already exists at the location is updated, a new one is created, and anything you leave out is untouched, so repeating the call is safe. Doors refer to types by name, so you don't need ids. Returns the full setup and the booking-page checklist (bookingPage.problems is empty when carriers can book). | write |
| `update_location_settings` | Change a location's scheduling settings. Omitted fields are left unchanged. | write |
| `set_weekly_hours` | Replace a location's hours for the listed days (days you leave out are unchanged; an empty windows list closes that day). A booking must fit entirely inside one window, so to offer fixed slots, make each window exactly one appointment long: 08:00–10:00 and 10:30–12:30 with a 2-hour type offer only 8:00 and 10:30. | write |
| `create_appointment_type` | Add an appointment type (what a carrier picks when booking): its length, load, direction, minimum notice, and buffers. | write |
| `update_appointment_type` | Edit an appointment type. Omitted fields are left unchanged. Set isActive to false to stop offering it without deleting it or its history. | write |
| `add_closed_dates` | Close the whole location on specific dates (holidays, shutdowns), in its local timezone. Existing appointments on those dates are not cancelled; the response counts them so you can follow up. | write |
| `remove_closure` | Reopen a closure by id (ids are in get_location_setup's upcomingClosures). | write |
| `create_door` | Add a dock door. By default it takes every appointment type and follows the location's hours, so it can be created before any types exist. Optionally limit it to certain types or give it its own hours (see update_door). To add types and doors together by name, use set_up_location. | write |
| `update_door` | Edit a dock door. Omitted fields are left unchanged. appointmentTypeIds limits which types it takes ([] = every type). weeklyHours gives the door its own full week of hours, always clipped to the location's (days you leave out are closed for this door); pass null to go back to the location's hours. isActive: false takes it out of service. | write |
| `create_location` | Add a facility. It starts with no hours, appointment types, or doors: follow up with set_up_location, which returns the booking-page checklist. The Free plan allows one location; more need Pro (payment_required). | write |
| `update_location_details` | Edit a location's name, address, timezone, or booking-page URL. Omitted fields are left unchanged. slug sets the public link /book/{slug}: 3+ lowercase letters, numbers, and hyphens; a slug another facility uses is a conflict. Changing it breaks links already sent to carriers. | write |
| `update_notification_settings` | Change who hears about bookings at a location. notifyEmail is one extra address that gets booking alerts besides the team (empty string clears it). emailNotificationsEnabled turns carrier confirmation emails on or off; smsCheckInEnabled texts drivers a check-in link before their appointment. Omitted fields are left unchanged. | write |
| `get_booking_page` | The public booking link carriers use (/book/{slug}) and whether it can take bookings: acceptingBookings is false until the location has weekly hours, an active appointment type, and (when it assigns doors) an active door. problems says what to fix. | read-only |
| `list_booking_questions` | The extra questions carriers answer when booking, per appointment type, in form order. Also lists which built-in fields (driverName, trailerNumber, poReference, …) each type hides or requires. | read-only |
| `create_booking_question` | Add a question to an appointment type's booking form (Pro). DROPDOWN needs options. key is how the answer is stored (customFields.{key} on appointments); it defaults to the camelCased label and can't change later. isReference makes the answer a reference number, searchable in the appointments list. | write |
| `update_booking_question` | Edit a booking question (Pro). Omitted fields are left unchanged; the key never changes. isActive: false hides it from the form but keeps past answers. | write |
| `delete_booking_question` | Remove a booking question for good. Answers already given stay on past appointments but are no longer labeled; to keep them, set isActive to false with update_booking_question instead. | write |
| `create_webhook` | Send appointment events to a URL (Pro). SimpleDock posts a test event first and saves the webhook only if the URL answers 2xx. Non-Slack payloads are signed: verify the X-SimpleDock-Signature header with signingSecret, which is returned only now. ZAPIER and SLACK need hooks.zapier.com / hooks.slack.com URLs. eventTypes empty = every event; locationIds empty = every location. | write |
| `get_account_status` | Whether the account owner has confirmed their email, and what waits until they do (appointment emails and texts, directory listing, search indexing, teammate invites). An account an agent created and nobody confirmed is deleted at deletesAt. Also lists each location's booking page URL. | read-only |
| `verify_email` | Confirm the owner's email with the six-digit code from the SimpleDock email they received. If you can read their inbox, take it from the newest SimpleDock email (the subject starts with the code); otherwise ask them to read it to you. Never guess. Five wrong codes and the email stops accepting codes (use resend_verification_email). | write |
| `resend_verification_email` | Send the owner a new confirmation email (link and code). Earlier links and codes stop working. Limited to a few per hour. | write |
| `invite_member` | Email someone an invitation to join the organization. Roles: ADMIN (full access, including members and billing), SCHEDULER (book and manage appointments), DOCK_STAFF (mark trucks arrived and done). Needs the owner's email confirmed first. | write |
| `create_api_key` | Create a long-lived API key (sd_live_…) for scripts and clients that can't refresh OAuth tokens. The full key is returned only now; store it. It's listed under Settings → API, where it can be revoked. Needs the owner's email confirmed first. | write |

## Support
- Email: support@simpledock.ai
- Docs: https://simpledock.ai/developers/mcp
- Website: https://simpledock.ai

## License
The documentation in this repo is MIT licensed. The SimpleDock service is governed by its [terms](https://simpledock.ai) and [privacy policy](https://simpledock.ai/privacy-policy).
