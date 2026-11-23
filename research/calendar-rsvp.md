# Research: answering an RSVP from the calendar app, and reminders in a calendar feed

Ticket: [#6](https://github.com/MLP-Software-Development/stammtisch-app/issues/6). Researched 2026-11-23. Terms as in
[GLOSSARY.md](../GLOSSARY.md): **calendar feed**, **RSVP**, **event**, **series**, **member**.

## Short answer

- A subscribed **calendar feed** (ICS over `webcal://`/`https://`) is **read-only in every client**. It cannot carry an
  RSVP back to the app. iTIP even forbids attendees on a published (one-way) calendar. It refreshes slowly (hours) and
  the publisher cannot speed it up in Google or Outlook.
- **Reminders (`VALARM`) in a feed are not reliable.** Apple Calendar drops them unless the member unticks "Remove
  alerts" when subscribing. Google and Outlook document nothing about honouring feed alarms. Plan to send reminders as
  app push notifications.
- **Answering from the calendar works only through email invitations** (iTIP over iMIP: `METHOD:REQUEST` out,
  `METHOD:REPLY` back). It needs our own mail domain, outbound and inbound mail, Gmail's "known sender" rules, and a
  plan for duplicates next to the feed. Doable, but it is a project of its own.
- **For the MVP:** a per-member feed with a one-tap RSVP link in each event's description, the member's current answer
  in the event title, and reminders handled by the app. Email invitations can come later as an opt-in per member.

## 1. What a subscribed calendar feed can and cannot do

The feed is a plain iCalendar file ([RFC 5545](https://www.rfc-editor.org/rfc/rfc5545)) that the client polls. Every
client stores it as a separate, read-only calendar.

| | Google Calendar (web, Android) | Apple Calendar (macOS, iOS) | Outlook (web, new, classic) |
|---|---|---|---|
| Subscribe | Web only ("From URL"). The Android/iOS apps cannot add one, but show it once added ([Google, answer 37100](https://support.google.com/calendar/answer/37100)). Google fetches the URL from its servers, so it must be public. | macOS: File > New Calendar Subscription; a `webcal://` link pre-fills it. Location "iCloud" syncs to all devices, "On My Mac" stays local ([Apple, Mac guide](https://support.apple.com/en-au/guide/calendar/icl1022/16.0/mac/26); [iCloud subscriptions, 2026-05-27](https://support.apple.com/en-gb/ht202361)). | Outlook.com / web: Add calendar > Subscribe from web ([Microsoft support](https://support.microsoft.com/en-us/office/import-or-subscribe-to-a-calendar-in-outlook-com-or-outlook-on-the-web-cff1429c-5af6-41ec-a5b4-74f2c278e98c)). New and classic Outlook desktop can fail to add feeds on some servers (an auth header issue); the workaround is the web ([Microsoft Learn, 2026-02-25](https://learn.microsoft.com/en-us/troubleshoot/outlook/calendaring/cannot-add-an-internet-calendar)). |
| Refresh | **Not documented by Google.** Community and vendor reports say many hours, up to about a day, and it can't be changed. | The member picks an Auto-refresh interval in the subscription dialog ([Apple](https://support.apple.com/en-au/guide/calendar/icl1022/16.0/mac/26)). | Outlook.com: "approximately every 3 hours", work accounts about 6 h, both "can take more than 24 hours" ([Microsoft support](https://support.microsoft.com/en-us/office/import-or-subscribe-to-a-calendar-in-outlook-com-or-outlook-on-the-web-cff1429c-5af6-41ec-a5b4-74f2c278e98c)). Exchange-managed web subscriptions: about every 4 h ([Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/outlook/calendaring/cannot-add-an-internet-calendar)). |
| Publisher hint for refresh | `REFRESH-INTERVAL` ([RFC 7986](https://www.rfc-editor.org/rfc/rfc7986), 2016) is only a SHOULD. In practice it is ignored. | Not documented. | Old Outlook 2007 honoured the publisher's interval (`X-PUBLISHED-TTL`). No current Microsoft statement exists. |
| `VALARM` reminders | Not documented. Reports say it is ignored; the member sets notifications per calendar. **Verify.** | Alerts are removed unless the member unticks "Remove: Alerts". "Ignore alerts" can turn them off as well ([Apple](https://support.apple.com/en-au/guide/calendar/icl1022/16.0/mac/26)). | Not documented. **Verify.** |
| `ATTENDEE`/`PARTSTAT` | Read-only calendar, no RSVP buttons. | "Subscribed calendars are read-only" ([Apple](https://support.apple.com/en-au/guide/calendar/icl1022/16.0/mac/26)). | Read-only, no RSVP. |
| Updates / cancellations | Changes show up on the next refresh. An event that is missing from the feed disappears. How each client handles `STATUS:CANCELLED` in a feed is **not documented. Verify.** | Same. | Same. Import, unlike subscribe, is a one-off snapshot. |

What the standards say:

- iTIP ([RFC 5546](https://www.rfc-editor.org/rfc/rfc5546), 2009): `PUBLISH` is a one-way posting and must not list
  attendees, so no replies are expected. RSVP needs `REQUEST`/`REPLY`, which a feed never carries.
- Identity of an event is `UID` (plus `RECURRENCE-ID` for one occurrence of a series). `SEQUENCE` and `DTSTAMP` order
  revisions. A feed must keep the `UID` stable, or clients show the event twice after an edit.
- **Per-member feeds are fine**: the client just fetches a URL. Each member gets a URL with their own secret token, so
  the feed can list only their groups and show their own answer, e.g. "Stammtisch (Zugesagt)" in the title.

## 2. Can an answer given in the calendar reach the app?

Only through **email invitations (iMIP)**.

- How it works ([RFC 6047](https://www.rfc-editor.org/rfc/rfc6047), 2010): the app emails each member a
  `text/calendar; method=REQUEST` part. The member's client shows Yes/No/Maybe and emails a `method=REPLY` to the
  `ORGANIZER` `mailto:` address. That reply holds one `ATTENDEE` with its `PARTSTAT` (ACCEPTED/DECLINED/TENTATIVE).
  The app needs an inbound mail endpoint that parses the reply, for example an inbound-mail webhook from a mail
  provider such as [Postmark Inbound](https://postmarkapp.com/developer/user-guide/inbound). Nhost does not offer
  inbound mail.
- **Google / Gmail**: members answer at the top of the invitation email or in Calendar
  ([answer 37135](https://support.google.com/calendar/answer/37135?hl=en&co=GENIE.Platform%3DDesktop)). With the
  setting "Only if the sender is known", an invitation from an unknown sender is **not added** to the calendar until the
  member clicks "Add to calendar" or answers
  ([answer 13159188](https://support.google.com/calendar/answer/13159188?hl=en&co=GENIE.Platform%3DDesktop)). For apps
  without access to the organiser's calendar, Google's own guidance is the ICS email. It also advises a unique, fixed
  sender address rather than a generic shared one, and asking members to add the sender to their contacts
  ([Calendar API: inviting attendees](https://developers.google.com/workspace/calendar/api/concepts/inviting-attendees-to-events)).
- **Apple**: an invitation that arrives in Mail is opened and then answered from Calendar's Notifications list
  ([Mac guide](https://support.apple.com/en-gb/guide/calendar/icl1016/mac)). iCloud lets each address receive
  invitations as in-app notifications or as email
  ([iCloud guide](https://support.apple.com/guide/icloud/send-receive-invitations-icloud-calendar-mm4a440a7b64/icloud)).
  Not documented: whether the reply to a non-iCloud organiser always goes out as iMIP email from iOS. **Verify.**
- **Outlook**: handles standard meeting requests from outside senders. We found no Microsoft page that documents it for
  outside ICS senders. **Verify in the prototype.**
- **Deliverability**: since 2024-02-01 Gmail requires SPF or DKIM, valid PTR, TLS, and a spam rate below 0.3% for all
  senders. Above 5,000 mails a day it also requires DMARC and alignment
  ([Gmail sender guidelines](https://support.google.com/a/answer/81126)). We need our own domain with SPF, DKIM and
  DMARC set up from day one.
- **Trust**: iMIP replies are only authentic with S/MIME, which no consumer client uses (RFC 6047 §5). A forged email
  could change someone's RSVP. Fix: use a per-member, per-event reply address, e.g. `rsvp+<token>@…`, and accept a
  reply only if the token and the `ATTENDEE` match.
- **Feed and invitations together**: they are two calendars in the client. The same event appears **twice**, once in
  the feed and once as the accepted invitation in the primary calendar. No client merges by `UID` across calendars.
  Each member therefore needs one channel per event: feed *or* invitations, never both.

## 3. Alternatives

| Option | Answer reaches app? | Reminders | Cost / effort | Privacy |
|---|---|---|---|---|
| **A. Feed only** | No | Unreliable (see §1) | Low: one Nhost function renders ICS per token | Feed URL is a secret (see §4) |
| **B. Feed + RSVP link in `DESCRIPTION`/`URL`** | Yes, one tap into app or web page | App push | Low | Link carries a token; keep the scope small |
| **C. Email invitations (iMIP)** | Yes, from the calendar's own buttons | Client default for invitations; `VALARM` in invites also unverified | Medium–high: mail domain, outbound + inbound provider, parser, bounce handling | Member's email goes to the mail provider; send one invite per member listing **only that member**, or all members' addresses leak to each other |
| **D. Google Calendar API / Microsoft Graph** | Only if *we* are the organiser in Google/Microsoft and watch attendee status. Writing into the member's own calendar gives them an event, not an RSVP. | API reminders work | High: OAuth per provider, consent screens, Google app verification for public apps ([scopes](https://developers.google.com/workspace/calendar/api/auth)); Graph needs `Calendars.ReadWrite`, also for personal accounts ([create event](https://learn.microsoft.com/en-us/graph/api/user-post-events?view=graph-rest-1.0)), and attendees always get invitations | We hold calendar write tokens: high risk |
| **E. CalDAV server (+ scheduling, [RFC 6638](https://www.rfc-editor.org/rfc/rfc6638))** | Yes on Apple: iOS/macOS can add a CalDAV account ([iPhone settings](https://support.apple.com/guide/iphone/change-calendar-settings-iphc37be2016/ios)). Google Calendar and Outlook web cannot add a CalDAV account. | Yes | High: run a CalDAV server next to Hasura, sync both ways | Member enters credentials in their OS |
| **F. Write to the device calendar from the Capacitor app** (Android [Calendar Provider](https://developer.android.com/identity/providers/calendar-provider), iOS EventKit) | No: the app writes the event, but nothing is sent back from the calendar | Yes: Android has a `Reminders` table; EventKit alarms (not checked in this pass) | Medium: native plugin, permission prompt, keeping events in sync | Device-local; needs a calendar permission |

Notes on B: a `GET` link must not change state, because mail scanners and link previews fetch links on their own
([RFC 9110 §9.2.1](https://www.rfc-editor.org/rfc/rfc9110#section-9.2.1)). The link opens a page or app screen
showing "Zusagen / Absagen", and the tap is a `POST`. On mobile it can be a universal or app link straight into the
Capacitor app.

## 4. Privacy of the feed URL

- A feed URL is a bearer secret, just like Google's own "secret address in iCal format". Google says only the owner
  should know it, and to reset it if it leaks
  ([answer 37648](https://support.google.com/calendar/answer/37648?hl=en)).
- Anyone holding the URL sees what the feed holds: events of all the member's groups, places, times, and their own
  answer. Third parties hold it too, because Google and Microsoft fetch it from their servers and iCloud stores it.
- Keep the feed lean: no other members' names, no headcount, no notes, no cash or document data. Use a long random
  token (no user id), one per member. "Link erneuern" in the app revokes it. Log the last fetch so the member can see
  that it is in use.
- RSVP links inside the feed must not be the feed token. Use a separate token per member and event that can only set
  that one RSVP. Even so, anyone holding the feed can answer for the member. If that is unacceptable, the link must
  require a login.

## Recommendation for the MVP

1. **One personal calendar feed per member** (`webcal://` link plus "In Google Kalender hinzufügen" instructions),
   covering all their groups. Stable `UID` per event, `SEQUENCE` bumped on change, cancelled events kept for a while
   with "Abgesagt" in the title.
2. **The member's own answer in the title** ("Zugesagt" / "Abgesagt" / "Offen"), plus an **RSVP link** in
   `DESCRIPTION` and `URL` that opens the app or web confirm page (POST to answer).
3. **Reminders come from the app** (push notifications, e.g. on the RSVP deadline and the day before). Include one
   `VALARM` in the feed as a bonus for Apple users who keep alerts, but promise nothing.
4. **Tell members the feed refreshes slowly** (Google: up to about a day), and that the app is the source of truth.
5. **Later, as an opt-in per member: email invitations (iMIP)** with a token reply address, for members who want to
   answer from the calendar's own buttons. Those members' feed then leaves out invited events to avoid duplicates.
   Skip the Google/Graph APIs and CalDAV.

## Open questions for the grilling ticket

- Is "answer in the app or via link" enough for the founding group, or is a Yes/No **button in the calendar** a must?
  If it is a must, iMIP moves into the MVP, with its own domain and mail work.
- Is the honest trade-off acceptable: anyone with the feed URL can see events and set the member's RSVP through its
  links? Or must the RSVP link always require a login?
- What goes in the feed: all groups in one feed, or one feed per group? Include events the member has declined?
- How should a cancelled event show: removed from the feed, or kept with "Abgesagt" for N days?
- Which reminders, at what times, and can a member switch them off per group? Push only, or email as well?
- Should a **binding event** after its RSVP deadline still show an RSVP link, or only a "cannot attend" link?
- To verify in a prototype on real accounts: Google and Outlook handling of `VALARM` and `STATUS:CANCELLED` in feeds,
  actual Google refresh times, and iMIP replies from iOS Mail and Outlook to an outside organiser.
