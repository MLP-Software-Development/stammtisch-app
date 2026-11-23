# Competitors and SpielerPlus import

Research for [#10](https://github.com/MLP-Software-Development/stammtisch-app/issues/10). All sources were read live on
2026-11-23. Prices and features change often, so check them again before relying on them. Domain terms follow
[GLOSSARY.md](../GLOSSARY.md). "Fines" is not a glossary term yet; it means what SpielerPlus calls *Strafen*.

## TL;DR

- **SpielerPlus has no self-serve export** for members, RSVP history, the team cash box or fines, and no public API.
  Premium adds a calendar subscription and a "PDF export/print". The only other route is a GDPR data-portability
  request, and that covers one person's own data, not the group's books.
- **Do not build a SpielerPlus importer.** Members join with our invite link (posted into the existing WhatsApp
  group), the admin creates the series again, and the cash ledger starts from an opening balance.
- **The market's recurring complaints are ads, paywalled basics (above all calendar sync), notification spam, and
  fragile web-shell apps that break after updates.** That last one matters for us because we plan a Capacitor app.
- **The Stammtisch-specific apps are all brand new** (released 2026, 0–4 ratings). They confirm demand for fines,
  dues and a transparent cash ledger. None of them is established.

## 1. SpielerPlus: export, import and API

SpielerPlus (and its international brand TeamPlus) is run by Sportplatz Media GmbH in Hamburg
([llms.txt](https://www.spielerplus.de/llms.txt)). Its help centre is a Zendesk instance. I listed all 124 German and
118 English articles through Zendesk's public Help Center API and searched their full text for export, Excel, CSV,
PDF, download, API and *Schnittstelle*.

| Data | Export? | Evidence |
|---|---|---|
| Members | **None documented.** Members join by invite link or code. | [Mitglieder einladen](https://support.spielerplus.de/hc/de/articles/36349554650257-Wie-kann-ich-neue-Mitglieder-ins-Team-einladen) |
| Events | **ICS calendar subscription, Premium only.** It stops working when Premium lapses. | [Kalender synchronisieren](https://support.spielerplus.de/hc/de/articles/6532216018833-Wie-kann-ich-den-SpielerPlus-Kalender-mit-meinem-eigenen-Kalender-synchronisieren), [Premium abgelaufen](https://support.spielerplus.de/hc/de/articles/34465976732817-Mein-Premium-ist-abgelaufen-Was-passiert-mit-den-Premium-Features-und-meinen-Daten) |
| RSVPs / attendance | **None documented.** In-app statistics only. | full-text search, no hits |
| Team cash box (*Teamkasse*: dues, payments) | **None documented.** The only bulk action is "reset", which deletes immediately. | [Teamkasse zurücksetzen](https://support.spielerplus.de/hc/de/articles/4402736317713-Wie-kann-ich-die-Teamkasse-zur%C3%BCcksetzen) |
| Fines and fine catalogue | **None documented.** Included in the reset above. | [Strafe vergeben](https://support.spielerplus.de/hc/de/articles/4402732638481-Wie-kann-ich-einem-Spieler-eine-Strafe-vergeben) |
| Poll results | PDF download, Premium only | [Abstimmungen](https://support.spielerplus.de/hc/de/articles/4415733785361) |
| "PDF-Export/Druck" | Premium only. The pricing page does not say what it covers. | [Premium page](https://www.spielerplus.de/lp/premium) |
| API | **None.** The privacy notice mentions APIs only for third parties such as Google Maps. No developer docs exist. | [Datenschutzhinweise](https://www.spielerplus.de/site/privacy) |
| GDPR portability | A right to receive *your own* data in a machine-readable format, on request | [Datenschutzhinweise](https://www.spielerplus.de/site/privacy), section "Recht auf Datenübertragbarkeit" |

Import goes the other way only. SpielerPlus takes events from an ICS URL or an `.xls` sheet whose columns are built
for matches (opponent, home game, reminder time, RSVP deadline), plus federation fixture feeds
([Excel-Import](https://support.spielerplus.de/hc/de/articles/37242392057873-Termine-per-Excel-Datei-xls-importieren),
[ICS-Import](https://support.spielerplus.de/hc/de/articles/37058387865489)).

**WhatsApp**, where the group actually came from, can export a chat as plain text (a ZIP on iPhone), with or without
media. WhatsApp itself says the export cannot be imported back
([WhatsApp Help](https://faq.whatsapp.com/1180414079177245/?cms_platform=android)). That export has no structured
member, event or money data.

### Import recommendation

1. **No SpielerPlus or WhatsApp importer in v1.** There is nothing structured to read. A scraper would be brittle and
   would handle members' personal data without their consent.
2. **Members:** an admin posts our invite link into the existing WhatsApp group, and everyone joins themselves. This
   is also how SpielerPlus and Spond onboard people.
3. **Events:** the admin creates one series (e.g. every 2nd Friday). For a monthly Stammtisch that is faster than any
   import. A generic "import events from ICS URL" could come later, and it would also cover SpielerPlus Premium
   feeds and Google Calendar.
4. **Cash ledger:** start with one opening-balance entry that matches the group's bank account on day one. Old
   history stays in SpielerPlus, a spreadsheet or a screenshot. If people ask for more, add a CSV import of
   transactions later, matching the bank CSV column style, not SpielerPlus.
5. **Exit path:** offer a free CSV export of members, RSVPs and the cash ledger from day one. No competitor except
   Spond does this for free, and it builds trust.

## 2. Feature comparison

A dash (—) means not offered or not found in official sources. "Paid" means it is behind a paid tier.

| | SpielerPlus | Spond | TeamSnap | Heja | Kadermanager |
|---|---|---|---|---|---|
| Focus | DACH amateur sports teams | Teams, clubs, groups (NO/EU) | US youth sports | Youth teams, parents | Web-first sports team site |
| Events + RSVP | Yes. Deadline, forced explicit decline; capacity, guests, cancel-and-notify are paid | Yes. Recurring events, season planner | Schedule free; **availability/RSVP tracking paid** | Yes, free | Yes. Deadline and capacity are paid |
| Reminders | Automatic: one reminder plus 4 pushes before the deadline, to everyone ([source](https://support.spielerplus.de/hc/de/articles/4402731729553)) | Yes | Yes | Automatic free; one-tap manual reminders paid | Email, plus SMS on paid tiers |
| Calendar sync | ICS subscription, **paid** | Writes into the device calendar; iOS needs **full calendar access** ([source](https://help.spond.com/app/en/articles/115529-calendar-synchronisation)) | Listed; tier not stated | **Paid** for non-admins (Plus / Pro Max) ([source](https://help.heja.io/en/articles/8061610-what-is-the-difference-between-heja-plus-team-pro-team-pro-max)) | ICS subscription, **free** |
| Cash box / dues | Team cash box with dues | Payment requests through Stripe, 2.5 % + €0.20 in the eurozone ([source](https://help.spond.com/club/en/articles/58192-what-is-the-transaction-fee-in-spond-club)) | Payment collection | Payment tracking paid | — |
| Fines | Fine catalogue, free | — | — | — | — |
| Photos | — (not on feature list) | In posts (one file at a time, per reviews) | Listed | — | Photo albums, free |
| Documents | TeamCloud, 1 GB paid | — | Listed | File sharing | Files, 1–20 GB by tier |
| Roles | Custom roles and permissions paid | Admins, guardians | Owner/manager | 1 admin free, unlimited paid | Custom player roles |
| Export | PDF paid; nothing else | **Excel:** members, attendance history, participant list; Excel/paste import of members ([source](https://help.spond.com/app/en/articles/203929-member-overview-status-colour-codes-import-export-and-member-management)) | — found | — found | **Excel:** events and attendance, paid |
| Ads | Banners in Basic | "Brand partnerships" / branded content ([source](https://help.spond.com/app/en/articles/102971-branded-content-on-spond)) | Heavy in Free | Yes in Free | — found |

Feature sources: [SpielerPlus Premium](https://www.spielerplus.de/lp/premium),
[Spond help](https://help.spond.com/app/en/), [TeamSnap teams](https://www.teamsnap.com/teams),
[Heja pricing](https://heja.io/pricing), [Kadermanager Preise](https://www.kadermanager.de/preise).

### Pricing (as shown on 2026-11-23)

| App | Free tier | Paid |
|---|---|---|
| SpielerPlus | Basic, with ads, **max 30 members** | Premium per team: €5.99/month or €53.99/year; cheaper per team for clubs |
| Spond | Everything | No subscription. Earns from payment fees and brand partnerships |
| TeamSnap | Free, roster up to 15, with ads | Premium $10.83/mo (annual) or $18.99/mo; Ultra $13.75 or $25.99; TeamSnap+ ad-free add-ons |
| Heja | Free, with ads | Team Pro from $8.33 (period not stated on the page); Pro Max; Heja Plus for parents |
| Kadermanager | Basis: 1 GB, 13-month archive | Pro €8/mo (annual) or €32/mo; Business €32 or €128 |

### Stammtisch-specific apps

All three appeared in the German App Store in 2026 and have almost no ratings yet.

- **DeinStammtisch** (iOS, Nov 2026). Built for a fixed evening group. RSVP by tap with reminders to people who have
  not answered. A "live evening" mode tracks who came late, fines and in-game counters (e.g. dice or card games
  that trigger fines automatically). Also a cash box with shared bills, annual leaderboards, documents (rules,
  minutes), offline last state, and no ads or tracking. Weak point: you join with a shared group name and password
  instead of personal accounts.
  [App Store](https://apps.apple.com/de/app/deinstammtisch/id6789122309)
- **Stammtisch/Sparverein Manager** (iOS, Sep 2026). Cash-ledger first. Monthly dues booked automatically on a rule
  (e.g. 1st Thursday), a free fine catalogue, expected vs actual balance shown separately, entries reversed instead of
  deleted, members without own accounts, many groups per account, and configurable permissions. Price per group: a
  2-month trial, then €0.99/month or €25 once.
  [App Store](https://apps.apple.com/de/app/stammtisch-sparverein-manager/id6809026484)
- **Stammtisch** (iOS, Feb 2026). A generic English meetup planner with date and place voting, RSVPs, recurring
  events, chat, calendar integration and public group discovery. Despite the name, nothing specific to German
  Stammtisch culture. [App Store](https://apps.apple.com/de/app/stammtisch/id6759274599)

Adjacent: cash-box apps for sports teams such as *TeamCash – Mannschaftskasse* (39 ratings) show that fines and dues
alone are enough for a product.

## 3. Store reviews: recurring complaints

Method: the public App Store review feed (newest reviews, up to 500 per storefront) for the German store, plus the US
store for Spond, Heja and TeamSnap. I tagged 1–2★ reviews since 2025-01-01 by theme with keywords, so the counts
are rough. Google Play was not sampled. Overall ratings on the German App Store: SpielerPlus 4.8 (12.3k), Spond 4.7
(5.9k), Heja 4.7 (109). TeamSnap is 4.8 (60k) in the US, but 237 of its 338 written reviews since 2025 are 1–2★.

- **SpielerPlus** (36 low out of 154 recent). The largest cluster: since a spring-2026 redesign the app opens to a
  white screen and needs several restarts. Several reviewers say it is just the website in a shell, not a native
  app. A team cash box update removed per-player fine totals and the members' own fine view. Other complaints: a
  notification-settings banner that keeps coming back, cookie consent asked again every few days, ads on members
  even when the club pays, paying for calendar sync, endless "load more" scrolling, and clumsy handling of several
  teams or children.
- **Spond** (DE 11 and US 10 low). Too much mandatory personal data at signup (phone, birth date, children's
  address). Full calendar access needed for sync. New ads and tracking. Font size cannot be changed. No message
  search. Recurring events failing in the app. Push notifications not arriving.
- **TeamSnap** (237 low). Dominated by **ads**: full-screen, hard-to-close ads that send you to the App Store, even
  while checking a last-minute field change. Also basic features behind a paywall, freezes, and no reachable
  support.
- **Heja** (US, 16 low). **Paying for calendar sync** is the top theme. Also pop-up and audio ads, and duplicate
  notifications for one post.
- **Kadermanager** (DE). Almost every written review is from April–May 2024, when an iOS update left the app stuck
  on its loading screen for weeks. One reviewer says support only suggested adding the website to the home screen.

## 4. Copy / avoid for a simple first version

**Copy**

- One-tap Yes/No RSVP, Pending shown clearly, and an optional RSVP deadline per event (SpielerPlus, Kadermanager).
- Reminders **only to Pending members**, one or two of them (DeinStammtisch), not SpielerPlus's five pushes to
  everyone.
- A calendar feed (ICS subscription URL) **free for every member** (Kadermanager). It needs no device calendar
  permission, unlike Spond.
- An invite link the admin can replace (SpielerPlus, Spond), plus joining several groups with one account
  (Sparverein Manager).
- Cash ledger rules from Sparverein Manager: dues booked automatically from a rule, entries reversed rather than
  deleted, expected vs actual balance. Every member can always see their own open amount (SpielerPlus lost this
  and was punished for it in reviews).
- Capacity, a guest (+1) and "cancel event and notify everyone" as plain free features. SpielerPlus charges for them.
- Free CSV export of members, RSVPs and the ledger (Spond does Excel for free).
- Show the last loaded state offline (DeinStammtisch).
- Later: a shareable recap image for the WhatsApp group after an event (DeinStammtisch), and a fine catalogue once
  "fine" enters the glossary.

**Avoid**

- Ads, sponsor content and consent prompts that come back again and again (TeamSnap, Heja, SpielerPlus, now Spond).
- Paywalling the basics: calendar sync, RSVP tracking, member count limits, export.
- Collecting data we do not need at signup (Spond). Ask for a name and nothing else that is mandatory.
- Notification spam, duplicate pushes and settings banners that keep coming back. Give per-type notification
  settings from the start.
- **A fragile web shell.** White screens after updates and iOS releases sank both SpielerPlus and Kadermanager. For
  Capacitor: bundle the app shell locally (no remote URL), cache data for offline, and test on iOS betas before each
  release.
- Redesigns that drop features people use (the SpielerPlus team cash box).
- Sports clutter: lineups, opponents, match reports, player stats, federation fixtures.
- A shared group password instead of personal accounts (DeinStammtisch).
- Fixed small fonts and no Dynamic Type (Spond). Switching between groups that is slow or interrupted by ads
  (TeamSnap, SpielerPlus).
