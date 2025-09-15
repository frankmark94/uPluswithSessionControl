## Private Session Control Enablement

This document summarizes the edits required to enable Pega Digital Messaging Private Session Control in this project and how to verify the flow end‑to‑end.

### What was implemented
- Added controlled session initialization callback:
  - Implemented `window.PegaUnifiedChatWidget.onCreateSessionRequest` in `src/global.js` to request session creation from the Digital Messaging Service.
  - Extracts `widgetId` from `settings.pega_chat.DMMURL` and signs a JWT with `iss = widgetId` using `settings.pega_chat.DMMSecret`.
  - POSTs to `https://widget.<region>.chat.pega.digital/<widgetId>/create` with `{ customerId }` and returns the `sessionId` to the widget.
- Persisted session ID:
  - On `onSessionInitialized`, we persist `sessionId` to `localStorage.sessionId` for subsequent API calls (private-data, custom event ack).
- Private Data on start:
  - When `UsePrivateSessionControl` is enabled, after session initialization we POST private data to the configured private endpoint using `iss = sessionId`.
- Custom Event Acknowledgements:
  - Reuse `localStorage.sessionId` to sign custom event acks (`iss = sessionId`).

### Files touched
- `src/global.js`
  - Added `onCreateSessionRequest` implementation
  - Updated `onSessionInitialized` to persist `sessionId`

### Configuration required
Update the active site configuration file under `public/<site>/js/config-settings.js`:
- `settings.pega_chat.UsePrivateSessionControl = true`
- `settings.pega_chat.DMMURL = "https://widget.<region>.chat.pega.digital/<widgetId>/widget.js"`
- `settings.pega_chat.DMMSecret = "<your JWT secret>"`
- Optional (if proxying private APIs via your gateway):
  - `settings.pega_chat.DMMPrivateURL = "https://<your-api-gateway-domain>/<stage>"`

Notes:
- Session create (`/create`) requires JWT with `iss = widgetId`.
- Private Data (`/private-data`) and custom-event ack require `iss = sessionId`.
- Per Pega guidance, only pass `customerId` in the `/create` call; do not resend `customerId` in subsequent Private Data posts.

### How to verify
1) Start dev server:
   - `npm run dev`
2) Open the page using the configured site and open the DM widget.
3) In the browser Network tab, confirm this order:
   - `/<widgetId>/create` POST (200, returns `sessionId`)
   - `onSessionInitialized` fires
   - Private data POST to your configured private endpoint
4) Confirm `localStorage.sessionId` is populated.
5) Trigger any custom event configured and verify the custom-event ack request is signed and delivered.

### Screenshot placeholder
Add a screenshot of the Security tab showing "Control chat session initialization" checked in Digital Messaging Manager:

![Configuration Screenshot Placeholder](docs/img/private-session-config.png)

Place your screenshot at `docs/img/private-session-config.png` or update the path above.

### Troubleshooting
- If `/create` does not fire:
  - Ensure `UsePrivateSessionControl = true` and `DMMURL`, `DMMSecret` are set for the active site config.
  - Confirm `DMMURL` contains the correct `<widgetId>` and region.
- If private data or acks fail:
  - Verify `localStorage.sessionId` exists and `DMMSecret` is correct.
  - If using a proxy (`DMMPrivateURL`), confirm the base URL and stage path (e.g., `/Prod`) match your gateway setup.

### Reference
- Pega documentation: Controlling Web Messaging session initialization


