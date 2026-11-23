# Nhost: what it gives us and where it stops

Research for [#7](https://github.com/MLP-Software-Development/stammtisch-app/issues/7). Researched 2026-11-23.

**Sources.** The Nhost docs ([docs.nhost.io](https://docs.nhost.io), read through their generated
[llms-full.txt](https://docs.nhost.io/llms-full.txt) and the [Auth](https://docs.nhost.io/openapi/auth.json) and
[Storage](https://docs.nhost.io/openapi/storage.json) OpenAPI specs), the source in
[nhost/nhost](https://github.com/nhost/nhost) (MIT, last push 2026-11-23), [nhost.io/pricing](https://nhost.io/pricing),
the Hasura, Apple, Google and Capacitor docs linked below. Most doc pages carry no date, so treat "as of 2026-11-23"
as the date for every claim. Where a claim comes from source code rather than docs, it says so.

**Short answer.** Nhost covers sign-in (passkeys, Apple, Google, magic link, email code), file storage with row-level
access rules and thumbnails, and scheduling through Hasura cron triggers and one-off scheduled events. It does **not**
send push notifications, does **not** strip photo location data, has **no** account deletion endpoint, and its free
tier is not fit for production (pauses after a week idle, 10 sign-in emails per hour, English-only emails). Plan on
Pro (from $25/month) once real members use it.

## Auth

Docs: [Auth overview](https://docs.nhost.io/products/auth), [config reference](https://docs.nhost.io/reference/configuration).

| Need | What Nhost provides |
| --- | --- |
| Passkeys | [WebAuthn sign-in](https://docs.nhost.io/products/auth/webauthn), off by default. Sign-up needs an email address. Sign-in without an email (discoverable passkeys) works: the source falls back to a discoverable login when no email is sent ([`sign_in_webauthn.go`](https://github.com/nhost/nhost/blob/main/services/auth/go/controller/sign_in_webauthn.go)). Relying-party ID and allowed origins can be set in `auth.method.webauthn.relyingParty`. Users can add several keys. |
| Sign in with Apple | Two ways: the web OAuth redirect (needs an Apple Services ID, key, team ID) or [ID-token sign-in](https://docs.nhost.io/products/auth/providers/idtokens), where the app gets an Apple ID token from the native sheet and sends it to Nhost. ID tokens are accepted for `apple` and `google` only. |
| Google | Same two ways ([Google provider](https://docs.nhost.io/products/auth/providers/sign-in-google)); ID-token audience can list several client IDs, comma-separated. |
| Magic link | [Magic link](https://docs.nhost.io/products/auth/sign-in-magic-link) with PKCE: the verifier is stored on the device that asked, and the link redirects back with a code to exchange. First use creates the account. |
| Email code | [Email OTP](https://docs.nhost.io/products/auth/otp/email): a code valid 10 minutes, single use; first use creates the account. |
| Session length | Access token (JWT) 15 minutes, refresh token 30 days by default; both configurable (`auth.session.*.expiresIn`, in seconds). |
| Account deletion | **No endpoint.** The Auth API has sign-out, email change and so on, but nothing that deletes the user. We delete the `auth.users` row ourselves from a function with admin rights. |
| Restricting sign-up | Allow/block lists for emails and domains, and an option to block sign-up; useful if joining only works through an invite link. |

### Inside a Capacitor app

- **Passkeys, iOS.** WebAuthn works in a `WKWebView` only if the relying-party domain is configured as an
  [associated domain](https://developer.apple.com/documentation/xcode/supporting-associated-domains) with the
  `webcredentials` service; otherwise the request fails
  ([Apple: Supporting passkeys](https://developer.apple.com/documentation/authenticationservices/supporting-passkeys)).
  So the RP ID must be a domain we own and serve an `apple-app-site-association` file from, not the
  `*.nhost.run` domain.
- **Passkeys, Android.** The system WebView supports WebAuthn from `androidx.webkit` 1.12 when the app enables
  `setWebAuthenticationSupport` and links to its website through Digital Asset Links; passkey autofill
  (`mediation: "conditional"`) is not supported there
  ([Android: Credential Manager in WebView](https://developer.android.com/identity/sign-in/credential-manager-webview)).
  Capacitor does not enable this for us; it needs a small native change or a plugin.
- **Apple and Google sheets.** The native sheets need a Capacitor plugin (community plugins exist; none is official).
  The ID token they return goes to Nhost's ID-token sign-in, so no browser redirect is needed.
- **Magic link on a phone.** The PKCE verifier lives in the app's storage. If the email link opens in the browser,
  or on another device, the exchange fails unless the link is a universal link / app link that opens the app.
  Email codes avoid this entirely. Custom URL schemes work as redirect targets once added to the allowed redirect
  URLs (the Nhost React Native guide uses one).
- **App Store rules.** An app that lets people create an account must let them delete it in the app
  ([guideline 5.1.1(v)](https://developer.apple.com/app-store/review/guidelines/#5.1.1)). Offering Google sign-in
  requires an equivalent privacy-friendly option such as Sign in with Apple
  ([4.8](https://developer.apple.com/app-store/review/guidelines/#login-services)). Apps using Sign in with Apple
  should revoke the Apple tokens on deletion via Apple's REST API
  ([Apple: account deletion](https://developer.apple.com/support/offering-account-deletion-in-your-app/)); I found
  no such revocation in the Nhost source, so that is ours to build.

## Storage

Docs: [Storage overview](https://docs.nhost.io/products/storage), [buckets](https://docs.nhost.io/products/storage/buckets),
[permissions and relationships](https://docs.nhost.io/products/storage/guides/permissions-and-relationships).

- **Upload limits.** Set per bucket (min and max size in bytes). The default bucket allows up to 50 MB per file
  (from the [storage migration](https://github.com/nhost/nhost/blob/main/services/storage/migrations/postgres/000001_create-initial-tables.up.sql)).
  Function request and response bodies are capped at 6 MB, so uploads should go straight to Storage, not through a
  function.
- **Access rules per row.** File metadata sits in `storage.files` and uses the same Hasura permission rules as any
  table. Nhost documents a "community files" pattern: a junction table links files to a group and only members can
  read them. "Only attendees of this event may see the album" is the same pattern with an RSVP check. Upload,
  download and delete each get their own rule.
- **Thumbnails.** Done on the fly through URL parameters: width/height up to 8000 px, quality, format
  (`auto`, JPEG, WebP, PNG, AVIF), blur. Results are cached at the CDN
  ([image transformation](https://docs.nhost.io/products/storage/image-transformation)).
- **EXIF / location data.** **Not stripped.** The original is stored as uploaded. The transformer (libvips) saves
  with default options and only auto-rotates when the format changes
  ([`image.go`](https://github.com/nhost/nhost/blob/main/services/storage/image/image.go)); I read that as metadata
  being kept, which should be tested. Either way the original download keeps GPS data. We must strip it on the
  device before upload.
- **Signed URLs.** Pre-signed URLs are supported, allowed per bucket, with a per-bucket expiry (default 30 seconds).
  Storage checks the caller's permission before issuing one
  ([pre-signed URLs](https://docs.nhost.io/products/storage/guides/presigned-urls)).
- **CDN.** Public files are cached at the edge; files that need a login are cached but every request is re-checked
  against permissions ([CDN](https://docs.nhost.io/products/storage/cdn)).
- **Antivirus.** Optional ClamAV scan on upload.
- **Cost.** Starter: 1 GB storage, 5 GB egress. Pro: 50 GB storage then $0.05/GB, 50 GB egress then $0.10/GB
  ([pricing](https://nhost.io/pricing)).

## Push

- **Nhost sends no push.** Nothing in the docs, config or source covers APNs, FCM or Web Push. Its "notification"
  guides write rows to a table for in-app display. We build or buy a sender, triggered by Hasura events.
- **App side.** The official [`@capacitor/push-notifications`](https://capacitorjs.com/docs/apis/push-notifications)
  plugin returns an APNs token on iOS and an FCM token on Android (Android needs a Firebase project). It has no web
  support and no sending.
- **Sending.** FCM is free on all Firebase plans ([Firebase pricing](https://firebase.google.com/pricing)). APNs
  needs a key from the paid Apple Developer Program (which we need for the App Store anyway).
- **Web push on iOS.** Works from iOS/iPadOS 16.4, only for a web app the person has added to the Home Screen (a
  manifest with `display: standalone` or `fullscreen`), and permission has to be asked for after a tap. No Apple
  Developer membership is needed for web push itself; servers must be able to reach `*.push.apple.com`
  ([WebKit, 2023-02-16](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)).
- **Functions limits for a sender.** Node.js only, no native add-ons, timeout 10 s on Starter, 180 s on Pro
  ([limits](https://docs.nhost.io/products/functions/limits)). Native-free libraries for APNs/FCM/Web Push would be
  needed, or a container on Nhost Run.

## Scheduled work

Docs: [Events](https://docs.nhost.io/products/events), [cron triggers](https://docs.nhost.io/products/events/cron-triggers),
[one-off scheduled events](https://docs.nhost.io/products/events/one-off-scheduled-events).

- **Three kinds of trigger**, all calling a webhook (usually an Nhost function) with retries: event triggers (row
  inserted/updated/deleted), cron triggers, and one-off scheduled events.
- **Cron runs in UTC.** A daily cron that looks for events whose RSVP deadline is 14 or 2 days away is the simplest
  reminder design; it has to convert to the group's local time itself.
- **One-off events** fire once at a given time, within about 10 seconds
  ([Hasura](https://hasura.io/docs/2.0/scheduled-triggers/create-one-off-scheduled-event/)). They can be created and
  deleted through the Hasura metadata API (`create_scheduled_event`, `delete_scheduled_event`) with admin rights
  ([Hasura metadata API](https://hasura.io/docs/2.0/api-reference/metadata-api/scheduled-triggers/)). That fits
  "remind 2 days before the deadline", but every moved or cancelled event means deleting and re-creating its
  reminders.
- **pg_cron** is also loaded in Nhost Postgres by default, an option for pure database jobs.
- **Function quotas.** Starter: 10 functions, 1 GB-hour; Pro: 50 functions, 10 GB-hours, then $0.18/GB-hour.

## Email for sign-in

- **Without our own SMTP** Nhost's shared sender is limited to 10 emails per hour for the whole project
  ([rate limits](https://docs.nhost.io/platform/cloud/rate-limits)). Not usable beyond testing.
- **Own SMTP:** any SMTP host (host, port, user, password, sender), or Postmark with its own templates.
- **German emails:** the built-in templates exist in English, Bulgarian, Czech, Spanish and French only
  ([source](https://github.com/nhost/nhost/tree/main/services/auth/email-templates)), and custom templates are
  **Pro only** ([email templates](https://docs.nhost.io/products/auth/email-templates)). The template is chosen by
  the user's `locale` column. Postmark's template integration may be a way round this on Starter; unverified.

## Hosting

- **EU regions.** The CLI source lists `eu-central-1`, `eu-north-1` and `eu-west-2` among ten regions
  ([`configure.go`](https://github.com/nhost/nhost/blob/main/cli/cmd/dockercredentials/configure.go)); the names
  match AWS regions (Frankfurt, Stockholm, London). No docs page lists the regions, and I could not see which ones
  the free plan may use.
- **GDPR.** Nhost AB is a Swedish company ([privacy policy](https://nhost.io/legal/privacy-policy), updated
  2025-02-12). It offers a pre-signed DPA to download, sign and email back
  ([DPA](https://nhost.io/legal/data-processing-agreement)). I could not read the PDF's subprocessor list.
- **Plans** ([pricing](https://nhost.io/pricing), no date on the page):

  | | Starter | Pro |
  | --- | --- | --- |
  | Price | $0 | from $25/month, $15 compute credit included |
  | Projects | 1 active; pauses after 1 week without activity | several; each extra project pays its own compute |
  | Database | 1 GB | 10 GB, then $0.20/GB |
  | Backups | none | daily, 7 days |
  | Function timeout | 10 s | 180 s |
  | Auth users | unlimited | unlimited |
  | Custom email templates | no | yes |

  Usage is never stopped on paid plans; extra use is billed.
- **Self-hosting.** All services are open source (MIT) with a Docker Compose reference setup, community support only
  ([self-hosting](https://docs.nhost.io/platform/self-hosting)). It is a real fallback but means running Postgres,
  Hasura, Auth, Storage and an S3 store ourselves.

## Gaps we fill ourselves

1. **Push sender** for APNs, FCM and (if we want it) Web Push, plus a table of device tokens per member, and
   clean-up of dead tokens.
2. **Account deletion**: a function that deletes the user, their files and personal data, and revokes Sign in
   with Apple tokens.
3. **EXIF stripping** on the device before upload (and a test that the server copy has no GPS).
4. **Reminder scheduler**: cron or one-off events, local-time handling, re-planning when events move.
5. **Native glue in Capacitor**: associated domains / asset links for passkeys, enabling WebView WebAuthn on
   Android, plugins for the Apple and Google sheets, universal links if we use magic links.
6. **SMTP provider** with EU hosting, and German sign-in email templates.
7. **A domain of our own** for the passkey relying-party ID and app links.

## Open questions for the grilling tickets

**Notifications**
- Push only in the native apps, or also Web Push for people who use the web app? On iOS that means asking them to
  add it to the Home Screen first.
- Do we run the sender as an Nhost function, or pay for a push service? Who holds the APNs key?
- Reminders by daily cron (simple, coarse) or one-off events per event (precise, but must follow every change)?
- Which time zone does a group live in, and is it stored on the group?

**Files**
- Does the album keep originals at full size (cost, privacy), or only a resized copy made on the device?
- What file size limit for photos and documents?
- Are pre-signed links ever needed (e.g. a document shared outside the app), or does everything stay behind login?

**Membership and sign-in**
- Which sign-in methods at launch: passkeys plus email code, with Apple and Google as extras? Adding Google forces
  Apple on iOS.
- Magic link or email code? The code is simpler in a native app.
- Is sign-up open, or does an account only come into being through an invite link?
- What happens to a member's RSVPs, photos and ledger entries when they delete their account?

**Stack**
- Starter for development only, Pro (from $25/month) before the first real group? Who pays?
- Which EU region, and is it available on our plan?
- Do we sign the DPA now, and which SMTP provider?
- Is self-hosting a plan B we keep tested, or only a theoretical exit?
