# Compliance: App Store, Google Play and German/EU law

Research for [#12](https://github.com/MLP-Software-Development/stammtisch-app/issues/12). Sources read live on
2026-11-23. **This is not legal advice.** It is a map of what the official texts say, written by a non-lawyer, to
decide what to build and what to look at with a lawyer or a data protection authority before launch.

Context: one operator (a private hobby developer, no revenue) runs a group app on the web, iOS and Android
(Capacitor). Backend is Nhost (Hasura, Postgres, auth, storage). Members hold accounts, RSVP to events, keep a cash
ledger, upload photos to albums and documents. Push notifications go through APNs and FCM.

## 1. Apple App Review Guidelines

Current version: [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/), "Last Updated:
October 16, 2026" (the October revision added 2.3.14 and 2.5.19 and tightened section 5; see the
[news post](https://developer.apple.com/news/?id=ey6d8onl)).

- **1.2 User-generated content.** Any app with UGC needs four things: a way to filter objectionable material, a way
  to report content with timely responses, a way to block abusive users, and published contact details. Photos,
  documents and RSVP notes count as UGC. Apple does not exempt closed groups.
- **1.2.1 Creator content.** Not relevant unless content is sold or age-restricted.
- **2.1(a) Review access.** Apps behind a login must give App Review a working demo account and a live backend. We
  need a seeded demo group with invented people and photos.
- **4.2 / 4.2.2 Minimum functionality.** An app must offer more than a repackaged website; web clippings and link
  collections are rejected. A Capacitor build that only wraps the web app is at risk. Native value that helps:
  push reminders, camera/photo picker upload, share sheet, calendar feed hand-off, offline view of the next event.
- **4.5.4 Push.** Push must not be required for the app to work, must not carry sensitive personal data, and
  marketing pushes need explicit opt-in plus an in-app opt-out. 5.1.1 also bans gating features behind enabling push.
- **4.8 Login services.** Only triggered if a third-party or social login (e.g. Google) is the primary sign-in. Then
  an equivalent option (Sign in with Apple qualifies) must also be offered. An app that uses only its own account
  system (email + password or magic link through Nhost) needs no Sign in with Apple.
- **5.1.1(i) Privacy policy.** Linked in App Store Connect and easy to reach inside the app. It must say what is
  collected and why, who receives it, how long it is kept, and how to withdraw consent or ask for deletion.
- **5.1.1(iii) Data minimisation.** Ask only for what the feature needs; prefer the system photo picker over full
  library access.
- **5.1.1(v) Account deletion.** If the app creates accounts, deletion must be startable inside the app. Apple's
  [deletion guidance](https://developer.apple.com/support/offering-account-deletion-in-your-app/) adds: deactivation
  is not enough; the account record and personal data go; content the user shared with others (posts, photos)
  should also be deleted unless law requires keeping it; a manual or delayed process is fine if the user is told the
  timeline and gets a confirmation; with Sign in with Apple, tokens must be revoked via the REST API.
- **5.1.2 Data use.** No reuse of data for another purpose without consent. Photo/camera data may not be used for
  marketing or data mining (5.1.2(vi)).
- **5.1.4 Kids.** Apps collecting data from minors must comply with children's privacy law (GDPR included).
- **Privacy details ("nutrition labels").** [App Privacy Details](https://developer.apple.com/app-store/app-privacy-details/)
  must be filled in App Store Connect, including data collected by SDKs. For this app, expect: Contact Info (name,
  email), User Content (Photos, Other User Content), Identifiers (User ID, Device ID for push), possibly Diagnostics.
  All linked to the user, purpose App Functionality, no tracking.
- **Seller identity.** [Enrolling](https://developer.apple.com/programs/enroll/) as an individual shows the
  person's legal name as seller. An organisation needs a legal entity, a D-U-N-S number and its own website/domain.
- **EU trader status (DSA).** [Apple's DSA page](https://developer.apple.com/help/app-store-connect/manage-compliance-information/manage-european-union-digital-services-act-trader-requirements/):
  hobbyists without commercial intent are unlikely to be traders. Non-traders publish no contact data on the
  product page; EU users see a notice that consumer-law rights do not apply. Traders publish address, phone, email.

## 2. Google Play

- **Data safety form** ([help](https://support.google.com/googleplay/android-developer/answer/10787469)). Required
  for every app on closed, open or production tracks. Covers data collected by SDKs and webviews. Processing by a
  service provider on our behalf (Nhost, FCM) is not "sharing". "Encrypted in transit" may only be declared if all
  traffic is TLS. A privacy policy link is mandatory even for apps that collect nothing.
- **Account deletion** ([help](https://support.google.com/googleplay/android-developer/answer/13327111)). If users can
  create accounts: an in-app way to start deletion **and** a public web page to request deletion without
  reinstalling (form, email or button), named after the app as on Play. Deleting the account means deleting its
  data; any retained data must be disclosed. Processors must be asked to delete too. The web link goes into the
  Data safety form.
- **User-generated content** ([help](https://support.google.com/googleplay/android-developer/answer/9876937)). Users
  accept terms that define and forbid objectionable content before posting; in-app reporting of content and users,
  also for closed communities; blocking wherever one member can reach another directly (mentions, messages);
  ongoing moderation and acting on reports.
- **Webview apps** ([Spam policy](https://support.google.com/googleplay/android-developer/answer/9899034)). An app
  whose main purpose is a webview of a website is disallowed without the site owner's permission (we own the site,
  so this is fine), and the [functionality policy](https://support.google.com/googleplay/android-developer/answer/9898783)
  rejects static apps with no app-specific features. Same lesson as Apple 4.2.
- **Target API level** ([requirements](https://developer.android.com/google/play/requirements/target-sdk)). Since
  31 August 2026, new apps and updates must target Android 16 (API 36); extension possible to 1 November 2026.
  Expect API 37 to follow in 2027. Capacitor's Android template must keep up.
- **Testing gate for new personal accounts** ([help](https://support.google.com/googleplay/android-developer/answer/14151465)).
  Personal developer accounts created after 13 Nov 2023 need a closed test with at least 12 testers opted in for
  14 consecutive days before production access. Our own group can be the testers.
- **Trader status** ([help](https://support.google.com/googleplay/android-developer/answer/17594752)). Every account
  must declare trader or non-trader (existing accounts by 1 Feb 2027; display changes from 1 Mar 2027). A trader
  with a personal account must publish a phone number and address. A false non-trader claim breaches EU consumer law.
  A personal account otherwise shows the chosen developer name and email
  ([contact info](https://support.google.com/googleplay/android-developer/answer/10840893)).
- **Families policy.** Only applies if the target audience includes children. Declaring an adult audience keeps us
  out of it.

## 3. GDPR (Regulation (EU) 2016/679)

Text: [EUR-Lex](https://eur-lex.europa.eu/eli/reg/2016/679/oj).

- **No hobby exemption for the operator.** The household exemption (Art. 2(2)(c)) covers members sharing among
  friends, but [Recital 18](https://gdpr-info.eu/recitals/no-18/) says the GDPR still applies to whoever provides the
  means for that private processing. The operator is the controller for the app, even unpaid.
- **Legal bases (Art. 6).** Account, membership, RSVP, ledger: performance of the service the member signed up for
  (Art. 6(1)(b)). Photos of people: consent (Art. 6(1)(a)) or legitimate interest (Art. 6(1)(f)) with a working
  objection path. Photos are not special-category data unless processed for biometric identification (Recital 51),
  so **no face recognition**.
- **Transparency (Art. 12–14).** A privacy policy in German, reachable before sign-up and inside the app.
- **Rights (Art. 15–21).** Access, correction, erasure (Art. 17), portability, objection. Erasure must reach photos a
  member uploaded and photos that show them. Export is cheap to build once and covers access + portability.
- **Privacy by design and default (Art. 25).** Albums default to the narrowest sensible visibility; no member data
  visible outside the group; invite links revocable (already in the glossary).
- **Storage limitation (Art. 5(1)(e)).** Decide retention: e.g. what happens to a member's ledger entries and photos
  when they leave a group or delete their account (anonymise vs delete), and to a whole group when it is abandoned.
- **Processors (Art. 28).** A written processor agreement with every provider that handles member data:
  - Nhost: DPA offered for GDPR customers, pre-signed by Nhost, returned by email
    ([DPA page](https://nhost.io/legal/data-processing-agreement), [privacy hints](https://nhost.io/legal/privacy-hints)).
    Sub-processor is AWS; the customer picks the region, Frankfurt (`eu-central-1`) exists.
  - Firebase Cloud Messaging: Google acts as processor under the Firebase Data Processing and Security Terms; Google
    LLC is certified under the EU-US Data Privacy Framework ([Firebase privacy](https://firebase.google.com/support/privacy)).
  - APNs, e-mail delivery, error tracking, any CDN: each needs the same check.
- **Third-country transfers (Art. 44 ff.).** US providers rely on the Data Privacy Framework or SCCs. Keep the
  database in the EU and push payloads free of personal content (also Apple 4.5.4).
- **Records of processing (Art. 30).** The under-250-staff exemption (Art. 30(5)) does not apply to regular
  processing, which an app is. A short record is needed.
- **Security and breaches (Art. 32–33).** Appropriate security (Hasura row-level permissions per group, private
  storage buckets, signed URLs). A breach must be reported to the authority within 72 hours unless unlikely to cause
  risk.
- **Data protection officer.** Not needed: [§ 38 BDSG](https://www.gesetze-im-internet.de/bdsg_2018/__38.html)
  starts at 20 people constantly processing data, or DPIA-level processing.
- **Minors (Art. 8).** Where consent is the basis, under-16s in Germany need parental consent. Simplest: a minimum
  age of 16 in the terms and the store rating, no child-directed features.

## 4. German law

- **TDDDG § 25** (the law the ticket calls TDDG; now officially *Telekommunikation-Digitale-Dienste-Datenschutz-Gesetz*,
  [text](https://www.gesetze-im-internet.de/ttdsg/__25.html)). Storing or reading anything on the device (cookies,
  local storage, push tokens, analytics IDs) needs consent unless strictly necessary for a service the user
  explicitly asked for. Login session and push token for reminders the member turned on: necessary. Analytics or
  crash SDKs: need consent or must be left out. **No analytics in the MVP avoids a consent banner.**
- **Legal notice (Impressum).** Two overlapping duties:
  - [§ 5 DDG](https://www.gesetze-im-internet.de/ddg/__5.html) covers "geschäftsmäßige, in der Regel gegen Entgelt
    angebotene digitale Dienste": name, address, fast electronic contact incl. e-mail, and register/VAT data if any.
    A free, noncommercial app is arguably outside it.
  - [§ 18(1) MStV](https://www.gesetze-bayern.de/Content/Document/MStV-18) catches every telemedium that does not
    serve "ausschließlich persönlichen oder familiären Zwecken": name and postal address. An app published in public
    stores for any group is not purely family use, so **a hobby operator still needs name and address** in the web
    app and inside the native apps (and an e-mail address for 1.2 / DSA contact anyway).
- **Right to one's own image (KUG §§ 22–23).** [§ 22](https://www.gesetze-im-internet.de/kunsturhg/__22.html):
  pictures of a person may be distributed or shown only with their consent.
  [§ 23](https://www.gesetze-im-internet.de/kunsturhg/__23.html) has exceptions (public figures, people as background,
  gatherings and processions, art), but they never apply against the depicted person's legitimate interest. A
  private Stammtisch evening is not a public gathering in that sense. How KUG and the GDPR interact for
  non-journalistic photos is unsettled in German case law (GDPR Art. 85 opening clause), so design for both: the
  uploader confirms the people shown agree, every member can flag "that's me, please remove", admins act on it, and
  album visibility stays inside the group.
- **Digital Services Act** (Regulation (EU) 2022/2065, [EUR-Lex](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32022R2065)).
  Storing member uploads makes the operator a hosting service. Points of contact, terms that state content rules,
  a notice-and-action mechanism (Art. 16) and a statement of reasons when content is removed (Art. 17) apply to
  hosting services of any size; the heavier online-platform duties are waived for micro enterprises (Art. 19). The
  same report button that Apple 1.2 and Google UGC require can serve Art. 16.
- **Cash ledger.** Manual bookkeeping of a group's money is not a payment service. Pulling bank transactions
  automatically through a bank API would be an account information service that needs BaFin registration under the
  ZAG. Keep the ledger manual.

## 5. What a noncommercial hobby operator still owes

Being unpaid removes the trader label in both stores and probably § 5 DDG, and nothing else of substance. Still
required: GDPR in full as controller (privacy policy, processor agreements, records, rights, deletion, breach
duty), § 25 TDDDG for anything on the device, § 18 MStV name and address, KUG consent for photos, DSA notice and
action, and every store rule above. The operator's name becomes public twice: the § 18 MStV notice and, with an
individual Apple account, the App Store seller name. Using a legal entity (e.g. a registered association or a small
company) would put that entity's name there instead, at the cost of a D-U-N-S number and a domain.

## 6. Checklist

| # | Item | Source | Kind |
|---|------|--------|------|
| 1 | Report content and members, block members, filter objectionable uploads, act on reports | Apple 1.2; Google UGC policy; DSA Art. 16–17 | Product |
| 2 | Terms of use accepted before first upload, defining forbidden content | Google UGC policy; DSA Art. 14 | Product + paperwork |
| 3 | Published contact e-mail for abuse and privacy | Apple 1.2; DSA Art. 11–12; § 18 MStV | Paperwork |
| 4 | In-app account deletion that removes the account, its data and its uploads; timeline and confirmation shown | Apple 5.1.1(v) + deletion guidance; Google account deletion; GDPR Art. 17 | Product |
| 5 | Public web page to request deletion without the app | Google account deletion | Product (small) |
| 6 | Decide retention: leaving a group, deleting an account, abandoned groups, ledger entries | GDPR Art. 5(1)(e), 17 | Product |
| 7 | Own account system only (email/magic link); adding Google login would force Sign in with Apple | Apple 4.8 | Product |
| 8 | Native value beyond the website: push reminders, photo picker/camera, share sheet, offline next event | Apple 4.2 / 4.2.2; Google functionality and spam policies | Product |
| 9 | Push optional, no personal content in payloads, no marketing pushes | Apple 4.5.4, 5.1.1; GDPR Art. 44 ff. | Product |
| 10 | Use system photo picker, not full library access | Apple 5.1.1(iii) | Product |
| 11 | Photo consent flow: uploader confirms, depicted member can request removal, album visibility inside the group | KUG §§ 22–23; GDPR Art. 6, 21, 25 | Product |
| 12 | No face recognition or biometric tagging | GDPR Art. 9, Recital 51 | Product |
| 13 | No analytics/ads SDKs in the MVP, so no consent banner | TDDDG § 25; Apple privacy details; Google Data safety | Product |
| 14 | Minimum age 16, adult store audience, no child-directed features | GDPR Art. 8; Apple 5.1.4; Google Families policy | Product + paperwork |
| 15 | Data export for a member | GDPR Art. 15, 20 | Product |
| 16 | Per-group data isolation (Hasura permissions, private storage) | GDPR Art. 25, 32 | Product |
| 17 | Cash ledger stays manual, no bank API | ZAG (account information service) | Product |
| 18 | Privacy policy in German, linked in both stores and in the app | Apple 5.1.1(i); Google Data safety; GDPR Art. 13 | Paperwork |
| 19 | Legal notice with name and postal address in web and native apps | § 18 MStV; maybe § 5 DDG | Paperwork |
| 20 | Processor agreements: Nhost DPA (EU region), Firebase terms, every other provider | GDPR Art. 28, 44 ff. | Paperwork |
| 21 | Records of processing | GDPR Art. 30 | Paperwork |
| 22 | Breach procedure (72 h notice) | GDPR Art. 33–34 | Paperwork |
| 23 | Apple privacy details and Google Data safety form filled to match the app | Apple App Privacy Details; Google Data safety | Paperwork |
| 24 | Demo account with invented data for review | Apple 2.1(a) | Paperwork |
| 25 | Declare non-trader in both stores (only while there is no revenue) | Apple DSA page; Google trader status | Paperwork |
| 26 | Choose individual vs organisation developer account (public name) | Apple enrolment; Google contact info | Decision |
| 27 | Closed test with 12 testers for 14 days before Play production | Google testing requirement | Paperwork |
| 28 | Android target API 36 now, follow yearly bumps | Google target API requirement | Paperwork (build) |

Open questions for a lawyer or the state data protection authority: whether the operator alone or also each group's
admins are controllers for group content; how far KUG still applies next to the GDPR for closed-group photos;
whether § 5 DDG applies to a free app.
