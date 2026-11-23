# Product name candidates: conflict check

Research for [#13](https://github.com/MLP-Software-Development/stammtisch-app/issues/13). As of **2026-11-23**.

**This is not a legal trademark clearance.** It is a first screen for obvious conflicts, so the founder can choose from a
short list. Before a name is used publicly or filed as a trademark, a trademark attorney should run a proper clearance
search (similarity and phonetic search, all relevant classes, company names and work titles).

The choice is not made here. It belongs to the grilling ticket this one unblocks.

## What a good name needs

From the README and the ticket: short, easy to say in German, suggests "we meet regularly" and "answer quickly", works for
a monthly Stammtisch first and for clubs later, and is free as a trademark, a store name and a domain.

Two legal points shape the screen:

- **Descriptive words are hard to protect.** § 8 (2) Nr. 1 and Nr. 2 MarkenG exclude marks without distinctiveness and
  marks made only of terms that describe the goods or services ([§ 8 MarkenG](https://www.gesetze-im-internet.de/markeng/__8.html)).
  "Stammtisch", "Treffpunkt" or "Dabei" on their own are therefore weak, whatever the register says.
- **Existing apps count even without a registered mark.** § 5 (1) and (3) MarkenG protect business designations and work
  titles ([§ 5 MarkenG](https://www.gesetze-im-internet.de/markeng/__5.html)), and the Federal Court of Justice held that
  app names can be work titles ([BGH, 28 Jan 2016, I ZR 202/14 "wetter.de"](https://rewis.io/urteile/urteil/j13-28-01-2016-i-zr-20214/)).
  So a store app with the same name in the same field is a real conflict, not only a marketing problem.

## Method and sources

| Check | Source | How | Limits |
| --- | --- | --- | --- |
| Trademarks DE, EU, international | [TMview](https://www.tmdn.org/tmview/) (EUIPO), offices DE (DPMA), EM (EUIPO), WO (WIPO) | Word search "contains" (substring), status filed or registered, mostly without class filter; broad stems ("dabei", "treff", "runde", "stamm", "zack", "jupp", "treffpunkt", "stammtisch") filtered to classes 9, 38, 42 | See below |
| `.de` domains | DENIC whois (`whois -h whois.denic.de "-T dn name.de"`) | `Status: free` or `connect` (registered) | Port-43 status only |
| `.app` domains | Google Registry RDAP (`https://pubapi.registry.google/rdap/domain/name.app`) | 404 = not registered, 200 = registered | `.app` has no port-43 whois |
| `.com` domains | Verisign whois (`whois -h whois.verisign-grs.com name.com`) | "No match" = not registered | |
| App Store | [iTunes Search API](https://performance-partners.apple.com/search-api), `country=de`, `entity=software` | Name and description of top results | German store only |
| Play Store | Google Play web search, `hl=de&gl=DE` | Top result titles | Fuzzy ranking; German store only |
| Social handles | Instagram profile page title; GitHub users API | A named page title = handle taken | TikTok and X cannot be checked without logging in; see below |
| Web | General web search for the shortlist | Existing products, clubs, companies | Spot check only |

**What the trademark search cannot show, and what a manual check must confirm:**

- The **DPMAregister** and **EUIPO eSearch plus** were not queried directly. DPMAregister has no open API, and both are
  interactive web apps. TMview takes in their data, but TMview says itself that it is not an official register and has
  no legal effect. TMview's bot protection also blocked scripted requests, and after about 40 searches through the
  browser it showed a "problem detected" page, so the stem searches for "klön" and "prost" could not be run.
- The search finds marks that **contain** the string. It does **not** find marks that look or sound similar
  ("Treffrunde" vs. "Trefrunde", "Jadabei" vs. "Ja, dabei!"), and it does not show figurative marks without a word part.
- For each name on the shortlist, a manual check must confirm:
  1. DPMAregister: an advanced search with similarity or phonetic matching in classes 9, 35, 38, 41, 42 and 45 (event
     organisation in 41 and social networking in 45 matter as much as software in 9 and 42).
  2. EUIPO eSearch plus: the same, for EU marks.
  3. WIPO Global Brand Database: international registrations designating DE or EU.
  4. Company names (Handelsregister, Unternehmensregister) and title announcements (Titelschutzanzeiger), since § 5
     MarkenG protects these without a registered mark.
  5. App Store and Play Store in Austria and Switzerland, if the DACH region is in scope.
  6. TikTok and X handles, by hand while logged in.

## Candidates

Domain key: **free** = not registered on 2026-11-23, **taken** = registered. TM = trademark. Instagram "taken" means a
profile page with a name exists. Holders are described, not named.

| # | Name | Rationale | TM conflicts found (TMview, active) | App / store conflicts | `.de` / `.app` / `.com` | Instagram |
| - | --- | --- | --- | --- | --- | --- |
| 1 | Stammtisch | The thing itself; instantly understood | Descriptive (§ 8 MarkenG). In 9/38/42: "MOG Stammtisch A46 Niederrhein" (DE 3020161021101, 38/39/41) | iOS "Stammtisch" (2026, organises recurring meetups) and iOS "DeinStammtisch" (Nov 2026, "the app for your regular group"): direct competitors. Also "Stammtisch/Sparverein Manager" (iOS, Play), Play "Stammtisch Manager", "Stammtischkasse" | taken / taken / taken | unclear |
| 2 | Stammi | Casual short form of Stammtisch | "Stammi Liquids" (DE 3020162096709, 16/26/34), unrelated goods | None by that name | taken / taken / taken | taken |
| 3 | Stammrunde | "Regular round": the fixed group that meets | Only an expired 1929 mark (DE 402363, class 34). Stem "stamm" in 9/38/42: "s stammgast" (EU 019133210), "SP StammPass" (DE, filed 2026) | None | free / free / free | taken |
| 4 | Stammtreff | "Regular meet-up" | None | None | taken / free / free | free |
| 5 | Stammzeit | "Our regular time" | "Stammzeit" (DE 3020222299627, EU 018856281; classes 8/11/20, unrelated goods) | None | taken / free / free | free |
| 6 | Tischrunde | "Round at the table" | None | iOS "Tischrunde" (games, Oct 2026): same name | taken / taken / taken | taken |
| 7 | Tafelrunde | Round Table; friends at one table | "tafelrunde" (DE 3020242053615, classes 16/41) | iOS and Play "Tafelrunde – Wochenplaner" (shared meal planning for groups, 2026) | taken / taken / taken | taken |
| 8 | Treffrunde | "Meeting round": a group that meets | None for the word. Stem "treff": 86 active marks in 9/38/42, e.g. "Treffpunkt" (DE 3020192220106, 35/38/45), "SingleTreff", "plusTreff" (dating operators), "TREFF ZEIT" | None | free / free / free | free |
| 9 | Rundtreff | "The round's meet-up" | None for the word. Same "treff" field as #8 | None | free / free / free | free |
| 10 | Monatsrunde | "The monthly round" | None | None | free / free / free | free |
| 11 | Klönrunde | Northern German "klönen" = to chat; a sociable round | None (stem search "klön" blocked, see limits) | None | free / free / free (also `klönrunde.de` free) | free |
| 12 | Prostrunde | "Cheers round"; Stammtisch mood | None (stem search "prost" blocked) | None | free / free / free | free |
| 13 | Jadabei | "Yes, I'm in": the one-tap Yes | None for the word. Stem "dabei": 27 active marks in 9/38/42, among them recent filings "dabei." (DE 3020262610463, filed Oct 2026, 9/42/45), "daDabei" (DE 3020262223990, 2026, 9/35/42/45), "weristdabei" (DE 3020262440886, filed Aug 2026, 9/42) | None | free / free / free | taken |
| 14 | Bindabei | "Count me in" | "ichbindabeitrag" (EU 013785878, 36/41/43/44); "my stella ich bin dabei!" (WO 965222, 9/38/42/45); "dabei" field as #13 | None by that name | taken / taken / taken | taken |
| 15 | Rundumdabei | "In on everything"; play on "rundum" | None for the word; "dabei" field as #13 | None | free / free / free | free |
| 16 | Kommste | Colloquial "are you coming?" | Only "… BEKOMMSTE SAURES!!!" (DE, 28/29/31), unrelated | iOS "Kommste" (social networking, Oct 2026, invites people to events): direct clash. iOS and Play "kommste mit?" (2026, social) | taken / free / taken | taken |
| 17 | Jupp | Rhineland "yep"; a quick yes | 8 active in 9/38/42, e.g. "JupptElectronics" (EU, 9), "Lernen mit Jupp" (DE, filed 2026, 9/41), "Sparklingjupp" (DE, 9/25/35/41) | Play "JuppIt", "JUP!" | taken / taken / taken | taken |
| 18 | Zackzu | "Zack, zugesagt": answered in a flash | None for the word. "Zack" registered in DE and EU by a games/software company in our classes; "Zack und Da", "zackig" (filed 2026) | None | free / free / free | taken |
| 19 | Treffly | Invented: "treffen" + "-ly" | None | Play "Treffly" (plan meetings with friends): direct clash | taken / free / taken | taken |
| 20 | Kränzchen | Old word for a circle that meets regularly | In 41 etc.: "Bierkränzchen" (DE, 25/32/41), "Das Krimikränzchen" (DE, filed 2026, 35/41/43) | None | taken (both spellings) / free / free | taken |
| 21 | Kumpanei | "Band of mates" | "KUMPANEI" (DE 3020232062107, incl. 35/43/45) | None | taken / free / taken | taken |
| 22 | Wiedermal | "Let's meet again" | None | None | taken / free / free | taken |
| 23 | Treffpunkt | "Meeting point" | Descriptive; 17 active in 9/38/42, incl. "Treffpunkt" (DE 3020192220106, 35/38/45) | iOS and Play: "Dein Treffpunkt", "Treffpunkt Leben …" and others | taken / taken / taken | taken |
| 24 | Dabei | "In" | Descriptive; 27 active in 9/38/42, incl. "dabei." (filed Oct 2026, 9/42/45) | iOS "Dabei-" (events-near-you map, 2026), "dabei!" (Vienna offers app) | taken / taken / taken | taken |

GitHub user or organisation names were also checked for #3, #8 to #13, #15 and #21: all are free except `kumpanei`,
which is a GitHub organisation.

**Ruled out by a direct clash:** Stammtisch, Tischrunde, Tafelrunde, Kommste, Treffly (same name, same field, in a store).
**Ruled out by domains:** Stammi, Jupp, Treffpunkt, Dabei, Bindabei (all three domains taken).
**High trademark risk:** Zackzu ("Zack" in class 9), Jadabei and Rundumdabei (fresh "dabei" filings in 9/42/45),
Kumpanei (registered in 45).

## Shortlist: the five with the least conflict

All five have `.de`, `.app` and `.com` free, no active trademark with the same word in DE, EU or WO, and no app by that
name in the German App Store or Play Store.

| Name | Why it is on the list | What to watch |
| --- | --- | --- |
| **Treffrunde** | Clear meaning ("a group that meets"), two syllables plus one, all domains and the Instagram handle free | "Treff" is a common, weak element with many marks in 38/45 (dating platforms); a similarity search must confirm there is no close mark |
| **Stammrunde** | Closest to "Stammtisch" in meaning, but not descriptive of software; no active mark (only one expired in 1929) | Instagram `@stammrunde` is taken; "stamm" marks in the field ("s stammgast", "StammPass"); may be confused with the "Stammtisch" apps |
| **Klönrunde** | Warm, sociable, memorable; `klönrunde.de` free too | Umlaut: spell the domain `kloenrunde`; regional (northern) flavour may feel less natural in the south; stem search for "klön" could not be run |
| **Monatsrunde** | Says exactly what the first group does | Descriptive (§ 8 MarkenG), so hard to protect; ties the brand to a monthly rhythm, which fits clubs that meet weekly badly |
| **Rundtreff** | Same field as Treffrunde, less common word, so more distinctive | Less natural German; same "treff" field as Treffrunde |

Runner-up: **Prostrunde** (everything free) is left off only because the alcohol connotation fits clubs and family groups
badly.

## Raw notes

- TMview search URLs used (one per name, offices DE, EM, WO, status filed and registered):
  `https://www.tmdn.org/tmview/#/tmview/results?page=1&pageSize=100&criteria=C&offices=DE,EM,WO&basicSearch=<name>&tmStatus=Filed,Registered`
  with `&niceClass=9,38,42` added for the broad stems.
- App Store apps named above:
  [Stammtisch](https://apps.apple.com/de/app/stammtisch/id6759274599),
  [DeinStammtisch](https://apps.apple.com/de/app/deinstammtisch/id6789122309),
  [Kommste](https://apps.apple.com/de/app/kommste/id6823226259),
  [kommste mit?](https://apps.apple.com/de/app/kommste-mit/id6777772127),
  [Tischrunde](https://apps.apple.com/de/app/tischrunde/id6809017731),
  [Tafelrunde – Wochenplaner](https://apps.apple.com/de/app/tafelrunde-wochenplaner/id6760330003),
  [Dabei-](https://apps.apple.com/de/app/dabei/id6787285279).
- Play Store: [Treffly](https://play.google.com/store/apps/details?id=com.bumblebee.meetly).
- Web searches for "Stammrunde", "Treffrunde", "Rundtreff", "Klönrunde", "Monatsrunde" and "Jadabei" as app names
  found no product with these names.
