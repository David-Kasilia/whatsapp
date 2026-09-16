### WhatsApp

Official WhatsApp integration for Frappe Apps.

### Installation

You can install this app using the [bench](https://github.com/frappe/bench) CLI:

```bash
cd $PATH_TO_YOUR_BENCH
bench get-app $URL_OF_THIS_REPO --branch main
bench install-app whatsapp
```

### Getting Started

Six steps take a fresh site from nothing to a first message. Everything the app does along the
way is recorded in **WhatsApp Log**, so open it beside Desk while you go.

#### 1. Set up the Meta side

1. At [developers.facebook.com](https://developers.facebook.com/apps) create an app of type
   **Business** and add the **WhatsApp** product to it.
2. Open **WhatsApp > API Setup** and note three values: the **App ID**, the **WhatsApp Business
   Account ID** and the **Phone Number ID**.
3. Generate a permanent access token. The token shown on API Setup expires in 24 hours, so
   instead go to **Meta Business Settings > Users > System Users**, create a system user, assign
   it the WhatsApp app and the business account, and generate a token with the
   `whatsapp_business_messaging` and `whatsapp_business_management` permissions.

Until the phone number is live, Meta only delivers to numbers you add under **API Setup >
To**. Sends to anyone else fail with "Recipient phone number not in allowed list".

#### 2. Fill in WhatsApp Settings

Open **WhatsApp Settings** in Desk.

| Field | Value |
|---|---|
| Webhook Verify Token | Any string you choose. You will repeat it on Meta in the next step |
| Webhook Secret | The **App Secret** from the app's **App Settings > Basic** page. Every webhook delivery is checked against it with HMAC-SHA256; leave it blank and deliveries are accepted unverified |
| API Url | Defaults to `https://graph.facebook.com` |
| API Version | Defaults to `v23.0` |

Leave **Default Account** empty for now; the first account you create fills it in.

#### 3. Register the webhook on Meta

In the app's **WhatsApp > Configuration** page:

1. Set the **Callback URL** to `https://<your-site>/api/method/whatsapp.whatsapp.webhook.handler`.
2. Set the **Verify token** to the value from step 2 and click **Verify and save**. The app
   answers Meta's challenge and writes "Webhook verified successfully" to WhatsApp Log.
3. Under **Webhook fields**, subscribe to `messages` and `message_template_status_update`.
   Other fields are delivered but ignored.

The site must be reachable over HTTPS from the internet. For a local bench, put a tunnel such as
ngrok in front of it and use the tunnel's URL.

#### 4. Create a WhatsApp Account

Open **WhatsApp Account > New** and fill in:

| Field | Value |
|---|---|
| Account name | Any label |
| Status | Active |
| App ID | From step 1 |
| Business ID | The WhatsApp Business Account ID from step 1 |
| Phone ID | The Phone Number ID from step 1 |
| Access token | The system user token from step 1. Hidden after save |

The first account saved becomes the **Default Account** in WhatsApp Settings; later accounts
leave that choice alone. **Auto Send Read Receipts** marks incoming messages as read on
WhatsApp as they arrive. The **Append Actions** table is optional automation, covered under
Notifications & Automation below.

#### 5. Sync templates

With one active account, the scheduler pulls templates from Meta daily. With several, the daily
job only logs that it found more than one, so sync each account by hand: open the **WhatsApp
Template** list, click **Sync from Meta** and pick the account.
Each template arrives with its Meta status, and only templates with status **Approved** can be
sent. The list view's sync calls
`whatsapp.whatsapp.doctype.whatsapp_template.whatsapp_template.sync_all`, and
`sync_from_account(account_name)` syncs a single account.

#### 6. Send a first message

WhatsApp only delivers free-form text inside its **customer service window**: the 24 hours after
the contact last messaged you. Outside that window, which includes a contact you have never
heard from, only an approved template gets through. So the first message to a new contact is
a template.

- **From Desk:** open **WhatsApp Message > New**, pick the recipient in **To** (a WhatsApp
  Profile, created automatically for every sender that has messaged you, or created by hand
  with the phone number), tick **Is Template** and choose a template, then **Submit**. The
  send happens on submit and the record's **Status** moves from Pending to Sent, then to
  Delivered and Read as Meta's status webhooks arrive.
- **From code:** call `send_template` or `send_message` from the Client API section below.
  Both accept a raw phone number for `to` and create the profile if needed.

Reply from the phone and the message lands as an incoming **WhatsApp Message** within a few
seconds, the window opens, and plain text sends work for the next 24 hours.

#### Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Meta reports the callback URL could not be verified | Response was 403 "token mismatch" or "invalid request": the verify token on Meta differs from **Webhook Verify Token** in WhatsApp Settings, or the URL is wrong. Check the log entry of type Webhook |
| Log shows "HMAC signature verification failed" on every delivery | **Webhook Secret** does not match the app's App Secret. Copy it again from App Settings > Basic |
| Sends fail with "Recipient phone number not in allowed list" | The number is live only for test recipients. Add the recipient on **API Setup > To**, or complete Meta's business verification to go live |
| A text message fails but templates work | The customer service window is closed. Send a template and wait for a reply |
| Nothing arrives when the phone sends a message | The `messages` webhook field is not subscribed, or the site is not reachable from Meta. WhatsApp Log gets a "Webhook payload received" entry for every delivery that reaches the site |

Every log entry carries a **Level** (Info, Warning, Error, Debug) and an **Event Type** (Webhook,
Template, Message, API, System), and API entries keep the request and response payloads, so
filtering the list on Level = Error is usually the fastest way to the cause.

### Contributing

This app uses `pre-commit` for code formatting and linting. Please [install pre-commit](https://pre-commit.com/#installation) and enable it for this repository:

```bash
cd apps/whatsapp
pre-commit install
```

Pre-commit is configured to use the following tools for checking and formatting your code:

- ruff
- eslint
- prettier
- pyupgrade

### Features

#### Core Messaging

- Receive incoming messages via Meta webhook (text, buttons, interactive, reactions, images, audio, documents, video, stickers)
- Auto-create WhatsApp Profiles for new contacts
- Send outgoing template, text, media, reaction, and interactive (buttons/lists) messages
- Message status tracking (Sent, Delivered, Read, Failed)
- **Reply-to / context messages** — outgoing messages can reference a previous message ID for threaded conversations via `reply_to_message` Link field
- **Read receipts** — configurable auto-send of `read` status per account (`auto_read_receipts` checkbox on WhatsApp Account)
- **Reaction messages** — send and receive emoji reactions to messages
- **Media messages (non-template)** — attach files (images, documents, videos, audio) as standalone outgoing messages; file is uploaded to Meta at send time
- **Template header media upload** — lazy upload of template header media (images, videos, documents) to Meta on first send, cached for reuse
- **Interactive messages** — quick reply buttons (up to 3) and list messages (up to 10 items) for structured user responses

#### Template Management

- WhatsApp Template management (create, sync, push to Meta)
- Template variables (named and positional)
- Button support (Quick Reply, URL, Copy Code, Phone Number, Voice Call)
- Template status tracking (Pending, Approved, Rejected, Deleted)

#### Client API

Whitelisted endpoints so a host app can build a messaging UI without reimplementing WhatsApp
logic. All of them are host-agnostic — no host's DocTypes or roles appear in their signatures.

| Method | Purpose |
|---|---|
| `whatsapp.whatsapp.api.messages.get_messages(references)` | Messages for one or more reference documents, with reactions folded onto their targets, template bodies rendered, replies resolved, attachment metadata joined and failure payloads reduced to a sentence |
| `whatsapp.whatsapp.api.messages.send_message(to, message, attach, content_type, reply_to, reference_doctype, reference_docname)` | Send text or media, optionally as a reply. Returns the new message's name |
| `whatsapp.whatsapp.api.messages.react_to_message(message, emoji)` | React to a message. Returns the reaction message's name |
| `whatsapp.whatsapp.api.messages.send_template(template, to, reference_doctype, reference_docname)` | Send an approved template |
| `whatsapp.whatsapp.doctype.whatsapp_template.whatsapp_template.get_sendable_templates(reference_doctype)` | Approved templates whose variables can be resolved from that DocType, buttons included |
| `whatsapp.whatsapp.doctype.whatsapp_template.whatsapp_template.create_template_and_push(doc_data, account_name)` | Create a template and push it to Meta for approval |

`references` is a JSON list of `[doctype, docname]` pairs — the **host** decides the scope
(e.g. a CRM Deal that should also show its converted Lead's messages), and the endpoint
verifies `read` permission on every reference it is handed. The **first** pair is where a
send attaches; the rest only widen the read.

`to` is a `WhatsApp Profile` name or a raw phone number, resolved against the default
account. `WhatsApp Message.notify_change()` publishes a `whatsapp_message` realtime event
carrying the reference doctype/docname, so a conversation view can refresh itself. It fires on
insert, on delete, and on a status change from the webhook, and is emitted after commit.

The event goes to the **reference document's room**, as `Communication.notify_change()` does,
so a client receives it only after a `doc_subscribe` the socket server has permission-checked.
A message with no reference publishes nothing — there is no room to scope it to, and the
site-room fallback would reach every Desk user.

Sender display names are deliberately **not** returned: for a given conversation the name is a
single string the host already knows, so it is passed to the UI rather than resolved per
message.

Permissions guard on the reference document. The app has no role model of its own yet
(see Known Gaps below), so a host with its own role policy must keep that check in front.

#### Client UI — `@whatsapp/ui`

Shared Vue components for rendering WhatsApp conversations, in [`ui/`](ui/). Ships raw source
consumed by a host's bundler; `frappe-ui` and `vue` are peer dependencies. See
[`ui/README.md`](ui/README.md) to install and use it.

#### Account & Configuration

- Multiple WhatsApp Business Accounts
- Auto-detect account from phone_number_id on webhook
- Default account fallback

#### Notifications & Automation

- 6 built-in Frappe Notifications (message received/sent/failed, status updated, template approved/rejected)
- Append Actions — auto-create linked documents in other DocTypes on incoming/outgoing messages (configurable per account)
- Server Script hooks via standard Frappe lifecycle (after_insert, after_save, etc. on WhatsApp Message)

#### Observability

- Browsable audit log (WhatsApp Log) capturing all webhook events, API calls, template operations, and message sends
- Log levels: Info, Warning, Error, Debug
- HMAC-SHA256 webhook signature verification

### Deferred

These features are planned but not yet implemented:

- **Media download on webhook** — incoming media messages capture only metadata (`media_id`, `mime_type`, `media_url`); the file bytes are never fetched. Pulling them into a Frappe `File` needs an async job plus a realtime update so the form reflects the download.
- **Location messages** — send and receive geographic location data (`latitude`/`longitude`/`name`/`address` fields)
- **Order messages (catalog)** — support `order` webhook type for catalog-based purchases and sending product catalog messages

### Known Gaps (to be fixed)

These are operational issues in the current implementation that should be addressed before a stable public release:

- **[P1] Role-based permissions** — all operations require System Manager. Production deployments need per-role read/write control on `WhatsApp Message`, `WhatsApp Profile`, and `WhatsApp Template` so non-admin users (e.g. support agents) can use the app safely.
- **[P1] App screen / home page** — `add_to_apps_screen` in `hooks.py` is commented out. The app has no dedicated UI entry point in Frappe Desk.
- **[P2] No contact enrichment** — `WhatsApp Profile` only stores phone number and display name. No fetch of Meta profile photo, email, or other contact metadata.
- **[P2] No send scheduling** — messages are sent immediately on submit. No support for delayed or time-zone-aware scheduled sends.
- **[P2] Webhook retry/recovery** — if webhook processing throws mid-way (e.g. after profile created but before message inserted), there is no recovery path. Partial state can be left behind silently.

### Planned

- Tech Provider Based Login Flow
- Bulk Sending
- Catalog Upload + Catalog Based Templates
- Group management
- Calling features
- Auto-block profile after N consecutive failed messages
- WhatsApp Flows (Meta's native form/flow builder)
- CRM chat-style UI in Frappe Desk
- WhatsApp preview for templates in desk

### Design Decisions

See [DESIGN_DECISIONS.md](DESIGN_DECISIONS.md) for intentional product constraints (e.g. named-only template variables, reference-DocType-driven parameters).

### License

mit
