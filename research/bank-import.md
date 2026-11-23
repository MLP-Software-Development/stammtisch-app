# Bank import for the cash ledger

Research for [#9](https://github.com/MLP-Software-Development/stammtisch-app/issues/9): which German bank statement
formats and automatic sync options can feed a group's **cash ledger**, and how incoming transfers can be matched to
members' **dues**. Pages were checked on 2026-11-23. Prices are net, as listed that day, and change often.

Confidence markers: **(verified)** means a primary source says so. **(secondary)** means a third-party write-up or an
open-source importer's code. **(unverified)** means we believe it but still need a real export to confirm it.

## TL;DR

- **MVP: an admin uploads a file.** Every bank on the list exports CSV. Most also offer camt.053 or MT940, though not
  every bank does. File import needs no licence, no contract and no running costs.
- **Use camt.053 where the bank offers it, and per-bank CSV everywhere else.** Treat MT940 as legacy. The German
  banking industry dropped MT940 from its standard in November 2025, but many online banking UIs still offer it as a
  download.
- **Automatic sync is harder than it looks for a small noncommercial app.** GoCardless Bank Account Data, the old free
  tier everyone used, has taken no new accounts since July 2025. finAPI starts at about €200–300 a month. Tink, Salt
  Edge and Enable Banking need a sales contract and company KYB. A server that reads other people's accounts is an
  account information service under § 1 (34) ZAG and needs BaFin registration, unless it runs under a licensed
  provider's umbrella.
- **Matching:** give each member a short **reference code** to put in the purpose text. Pre-fill it with a GiroCode
  (EPC QR). Remember the counterparty IBAN once an admin confirms a match, and use name matching only to make
  suggestions. Do not rely on the end-to-end reference.

## 1. Export formats by bank

### The formats

| Format | What it is | Notes for us |
| --- | --- | --- |
| **camt.053** | ISO 20022 XML end-of-day statement (booked entries) | The target format. Structured fields for counterparty name, IBAN, purpose and references. The German banking industry (DK) moved to **version 08** in November 2025 ([Sparkasse Nürnberg FAQ](https://www.sparkasse-nuernberg.de/content/dam/myif/spk-nuernberg/work/dokumente/pdf/ebics/FAQ%20EBICS%20camt%20Umstellung.pdf?n=true), [DATEV](https://www.datev.de/web/de/berufsgruppenuebergreifend/gesetzliche-themen/payment/kontoumsaetze-bereitstellen-neues-format)) **(verified)** |
| **camt.052** | ISO 20022 intraday report, which can include unbooked entries | Not for a ledger: entries can still change. Ignore for import. |
| **camt.054** | Single debit/credit notifications | Mostly business/EBICS. Ignore. |
| **MT940** | SWIFT text statement. German banks pack SEPA data into field `:86:` with `?20`–`?29` (purpose, in 27-character chunks, with prefixes such as `EREF+` and `SVWZ+`), `?31` (IBAN) and `?32`/`?33` (name) | DK removed it from its standard in November 2025 ([DATEV](https://www.datev.de/web/de/berufsgruppenuebergreifend/gesetzliche-themen/payment/kontoumsaetze-bereitstellen-neues-format)). Commerzbank still offers it next to camt over HBCI for a transition period ([Commerzbank](https://www.commerzbank.de/service/umstellung-von-mt-formaten-auf-camt-formate-in-hbci/)) **(verified)**. Support it only if a target bank has no camt and no usable CSV. |
| **CSV** | Each bank's own layout: semicolon-separated, German number and date formats, often ISO-8859-1, often with metadata lines above the header | Offered everywhere. It is what admins know. It needs one small adapter per bank and breaks when a bank redesigns its export. |

### Per bank (private and club accounts in online banking)

| Bank | Offered for download | Counterparty IBAN in CSV? | Source / confidence |
| --- | --- | --- | --- |
| **Sparkasse** | CSV in a camt-derived layout (labelled roughly "Excel (CSV-CAMT V2)"). MT940 and camt downloads may also be offered, and the menu differs between institutions (unverified) | Yes (unverified) | Vendor blog only ([imagetotable](https://imagetotable.ai/blog/extract-sparkasse-kontoauszug-excel)) **(secondary)**. Needs a real export. |
| **Volksbank / Raiffeisenbank** | PDF, CSV and MT940 by default. CAMT only if the bank has enabled it on the account. Export reaches back about two years | Unknown | Two cooperative banks' export guides ([Volksbank Backnang](https://www.volksbank-backnang.de/content/dam/f0012-0/cloud-transfer/dokumente/downloads---leitf%C3%A4den/Anleitung%20zum%20Export%20von%20Ums%C3%A4tzen%20im%20OnlineBanking.pdf), [VR Bank Kempen-Krefeld](https://www.vrbank-ke-oa.de/content/dam/f0283-0/cloud-transfer/Homepage/OnlineBanking/Services/Download-und-Demos/Anleitung%20Ums%C3%A4tze%20exportieren.pdf)) **(verified for those banks)**. Each of the several hundred cooperative banks configures this itself. |
| **Deutsche Bank** | CSV in online banking. camt mainly through EBICS for business customers ([camt fact sheet](https://www.deutsche-bank.de/dam/deutschebank/de/shared/pdf/ub/camt_FactSheet.pdf)) | Unknown | **(unverified)** for private accounts |
| **Commerzbank** | PDF and CSV ("Excel-compatible") in online banking. 24 months of history, 60 for premium accounts. camt and MT940 over HBCI/FinTS | Unknown | [Commerzbank export help](https://www.commerzbank.de/service/wie-kann-ich-meine-umsaetze-exportieren/) **(verified)** |
| **ING** | CSV via "Exportieren" in web banking. Columns: `Buchung; Valuta; Auftraggeber/Empfänger; Buchungstext; Kategorie; Verwendungszweck; Saldo; Währung; Betrag; Währung`, ISO-8859-1, with metadata lines (IBAN, period, balance) above the header. Only booked entries | **No** | [ING help](https://www.ing.de/hilfe/banking/), column list from the [beancount-ing](https://github.com/siddhantgoel/beancount-ing) test fixtures **(secondary, but from real files)** |
| **DKB** | CSV from the web portal (the app is less reliable), plus PDF. The 2023 layout has `Buchungsdatum; Wertstellung; Status; Zahlungspflichtige*r; Zahlungsempfänger*in; Verwendungszweck; Umsatztyp; IBAN; Betrag (€); Gläubiger-ID; Mandatsreferenz; Kundenreferenz`. Two-digit year in the date | Yes | Column list from the [beancount-dkb](https://github.com/siddhantgoel/beancount-dkb) source **(secondary)** |
| **N26** | CSV and PDF from the **web app** only. The mobile app gives monthly PDF statements. Current CSV: `Booking Date, Value Date, Partner Name, Partner Iban, Type, Payment Reference, Account Name, Amount (EUR), …`. German and English header variants exist | Yes | [N26 support](https://support.n26.com/en-de/account-and-personal-details/bank-statements-and-confirmations/how-to-get-bank-statement-n26) **(verified)**, columns from [beancount-n26](https://github.com/siddhantgoel/beancount-n26) **(secondary)** |
| **Postbank** | CSV in online banking. camt.052/053/054 through EBICS and SWIFT for business customers ([Postbank ISO 20022 doc](https://www.postbank.de/dam/postbank/pdf/geschaeftskunden/Postbank-ISO-20022-Kontoauszugsdokumente-0326.pdf)) | Unknown | **(unverified)** for private accounts |

### Which fields we can rely on

| Field | camt.053 | MT940 (German SEPA `:86:`) | Bank CSV |
| --- | --- | --- | --- |
| Booking date, amount, credit/debit | always | always | always |
| Purpose text (up to 140 chars) | `RmtInf/Ustrd`, sometimes split into several lines | `?20`–`?29`, split into 27-character chunks, with `SVWZ+` | always, sometimes with line breaks removed |
| Counterparty name | `RltdPties/Dbtr` (for credits) | `?32`/`?33` | almost always |
| Counterparty IBAN | `RltdPties/DbtrAcct` | `?31` | **missing in ING CSV**. Present at DKB and N26. Unknown elsewhere |
| End-to-end reference | `Refs/EndToEndId`, but usually `NOTPROVIDED` | `EREF+`, usually `NOTPROVIDED` | at best a "Kundenreferenz" column, usually empty |
| Stable transaction ID for de-duplication | `AcctSvcrRef`/`NtryRef` (optional) | sometimes in `:61:` | **none** |

Most private payers send transfers from online banking forms that have no field for an end-to-end reference, so the
purpose text is the only reference we reliably get. A CSV file has no stable transaction ID, and admins' exports will
overlap, so the importer needs a content-based de-duplication key: booking date, amount, counterparty, purpose, plus a
running number for identical rows on the same day.

## 2. Parsing libraries (TypeScript / npm)

Checked against the npm registry on 2026-11-23. Downloads are for the last month.

| Package | Purpose | License | Latest release | Downloads | Verdict |
| --- | --- | --- | --- | --- | --- |
| [fast-xml-parser](https://www.npmjs.com/package/fast-xml-parser) | XML parsing, pure JS | MIT | 5.12.1 (Nov 2026) | very high | **Base for our own camt.053 mapper** |
| [papaparse](https://www.npmjs.com/package/papaparse) | CSV, runs in browser and Node | MIT | 5.7.1 (Nov 2026) | very high | **Use for CSV.** Decode ISO-8859-1 with `TextDecoder` first |
| [@ekwo-ai/camt053](https://www.npmjs.com/package/@ekwo-ai/camt053) | camt.053 to plain objects, no dependencies | MIT | 0.27.0 (Nov 2026) | ~8.5k | Promising, but less than 3 months old. Evaluate it, or copy its approach |
| [@financica/camt053](https://www.npmjs.com/package/@financica/camt053) | camt.053, built on fast-xml-parser | MIT | 0.4.0 (Aug 2026) | ~190 | Young, low adoption |
| [camt-parser](https://www.npmjs.com/package/camt-parser) | camt.053 with XSD validation | MIT | 2.1.0 (May 2025) | ~650 | Depends on native `libxmljs2`: no browser, awkward on serverless. Avoid |
| [mt940js](https://www.npmjs.com/package/mt940js) | MT940 | Apache-2.0 | 1.3.5 (2020) | ~25k | Mature but unmaintained. Fine if we ever need MT940 |
| [mt940-js](https://www.npmjs.com/package/mt940-js) | MT940, isomorphic | MIT | 1.0.0 (2020) | ~25k | Same |
| [lib-fints](https://www.npmjs.com/package/lib-fints) | FinTS 3.0 PIN/TAN client: statements as MT940 or camt, decoupled TAN | **LGPL-2.1-or-later** | 1.5.3 (Oct 2026) | ~96k | The live FinTS option, tested against comdirect, DKB and ING ([README](https://github.com/robocode13/lib-fints)). LGPL is fine as an npm dependency on a server. Check it before bundling into the apps |
| [fints-lib](https://www.npmjs.com/package/fints-lib) | Fork of Prior99/fints: FinTS 3.0, plus an experimental read-only FinTS 4.1 | MIT | 0.12.0 (Apr 2026) | ~9.6k | Alternative to lib-fints with a permissive license |

All candidates are permissively licensed except lib-fints (LGPL). That matters because the project itself is under
the PolyForm Noncommercial license. We need only a small subset of camt.053 (entries, amount, sign, dates,
counterparty, purpose, references), so a ~150-line mapper on fast-xml-parser with fixture tests is a safe default and
avoids betting on a two-month-old package.

## 3. Automatic sync without file upload

### 3a. FinTS (HBCI) directly

- Most German banks offer FinTS 3.0 with PIN/TAN. **N26 does not**: it offers only its PSD2 interface
  ([Outbank help](https://help.outbankapp.com/en/kb/articles/wie-kann-ich-meinen-n26-zugangder-app-anlegen)
  **(secondary)**, [N26 PSD2 page](https://support.n26.com/en-de/security/open-banking-psd2/psd2-open-banking-for-third-party-providers)).
- Since August 2019, banks accept only **registered products**. The Deutsche Kreditwirtschaft (DK) registers a
  product by paper form, typically in 10–15 working days, and the product must then send its registration number in
  every dialog ([fints.org](https://www.fints.org/de/hersteller/produktregistrierung)). Library authors say
  registration is currently free ([lib-fints README](https://github.com/robocode13/lib-fints)) **(secondary)**.
- Every sync needs strong customer authentication. With decoupled TAN, the admin confirms in their banking app.
  Some banks let read access run without a new TAN for a while, often 90 or 180 days. This varies by bank (unverified).
- Bank FinTS servers send no CORS headers, so a FinTS client cannot run in a normal browser tab. It can run on our
  server, or inside the Capacitor apps through native HTTP (`CapacitorHttp`), which bypasses WebView CORS
  ([Capacitor HTTP docs](https://capacitorjs.com/docs/apis/http), community confirmation) **(secondary)**.
- **The legal line, our reading and not legal advice:** if **our server** logs in to a group's bank account with an
  admin's PIN and fetches transactions as an online service, that looks like an account information service
  (*Kontoinformationsdienst*, [§ 1 (34) ZAG](https://www.gesetze-im-internet.de/zag_2018/__1.html)). Doing that on a
  commercial basis needs BaFin registration under [§ 34 ZAG](https://www.gesetze-im-internet.de/zag_2018/__34.html),
  plus professional indemnity insurance under [§ 36 ZAG](https://www.gesetze-im-internet.de/zag_2018/__36.html). If
  the **admin's own device** runs the FinTS client, the way desktop banking software does, and the admin pushes the
  result to the ledger, it is much closer to "software the user runs" than to a service. It also never puts bank PINs
  on our server. Whether "noncommercial" escapes "gewerbsmäßig" is a question for a lawyer. Do not assume it.

### 3b. PSD2 aggregators

All of these hold the AIS licence themselves. That is what lets an unlicensed app use them.

| Provider | Status / price on 2026-11-23 | Fit for a small noncommercial app |
| --- | --- | --- |
| **GoCardless Bank Account Data** (formerly Nordigen) | **Has taken no new accounts since July 2025**. Existing accounts keep working ([Actual Budget docs](https://actualbudget.org/docs/advanced/bank-sync/gocardless/), [Firefly III docs](https://docs.firefly-iii.org/how-to/data-importer/import/gocardless/)). Mollie announced it would acquire GoCardless ([GoCardless FAQ](https://support.gocardless.com/hc/en-au/articles/24187025310620-Mollie-to-acquire-GoCardless-Customer-FAQs)) | **Not available** |
| **finAPI Access** (Germany) | Per month: B2C (private accounts) €60 for up to 200 users; B2X (includes business and club accounts) €100. Unlicensed customers must use the finAPI web form and pay the PSD2 licence fee on top (AIS €200 for up to 200 users). "Own use", at most 10 of your own accounts and no PSD2 licence needed, costs €200 per month. 30-day free trial ([finAPI prices](https://www.finapi.io/en/prices/)) **(verified)** | Works technically and covers every German bank. At **€260–300 a month** it is too expensive for a noncommercial app |
| **Tink** (Visa) | "Standard" plan runs on Tink's licence, price on request. "Enterprise" is custom ([Tink pricing](https://tink.com/pricing/)) | Sales-led. Likely too big for us |
| **Enable Banking** (Finland) | Free **restricted mode**: only accounts the app owner has linked, for evaluation or the personal use of private individuals ([Terms](https://enablebanking.com/terms), [Firefly III guide](https://docs.firefly-iii.org/tutorials/data-importer/eb/)). Production needs a contract, company KYB and a volume-based price with a monthly minimum ([FAQ](https://enablebanking.com/docs/faq)) | Restricted mode fits one self-hosted group but not a multi-group service. Production pricing is unpublished: ask for a quote |
| **Salt Edge**, **Yapily** | No public prices. Free sandbox, production sales-led (Salt Edge [AIS product](https://saltedge.com/products/spectre)) **(secondary)** | Ask for a quote only if we go this way |

PSD2 consent for account information lasts at most **180 days**, after which the admin must authenticate again
([N26 TPP page](https://support.n26.com/en-de/security/open-banking-psd2/psd2-open-banking-for-third-party-providers)).
Any automatic sync therefore still needs an admin re-consent flow about twice a year.

## 4. How other apps match transfers to members

Club and accounting software converges on the same pattern
([Lexware Office](https://help.lexware.de/de-form/articles/548900-zuordnungsvorschlage-fur-ihre-bankumsatze),
[sevdesk](https://intercom.help/sevdesk/de/articles/9382118-automatische-transaktionszuordnung),
[VereinOnline guide](https://www.VereinOnline.org/VereinOnline_Leitfaden_Buchhaltung.pdf),
[Kufer import sheet](https://www.kufer.de/fileadmin/Infoblatt_Import_von_Auszuegen_.pdf),
[BSSB member tool](https://www.BSSB.de/fileadmin/Service/ZMI-Client/Handbuecher/Import_Bankdaten.pdf)):

1. **A unique key in the purpose text** (invoice or member number) is the only signal any of them trusts for fully
   automatic assignment. Kufer and VereinOnline assign automatically *only* in that case.
2. **Name and amount** produce suggestions. sevdesk lets users rank purpose, amount and name.
3. **Account data** (IBAN, or old account number plus bank code) identifies a member in BSSB.
4. **Everything else goes to a manual "unassigned" pool.**

For us, in order of strength:

- **A reference code per member and group**, for example `ST-K7Q4`: short, upper case, without ambiguous characters
  (0/O, 1/I), ideally with a check character. Match it in the purpose text, ignoring spaces, hyphens and case. Show it
  in the app next to the dues, together with a **GiroCode** (EPC QR,
  [EPC069-12](https://europeanpaymentscouncil.eu/sites/default/files/kb/file/2022-09/EPC069-12%20v3.0%20Quick%20Response%20Code%20-%20Guidelines%20to%20Enable%20the%20Data%20Capture%20for%20the%20Initiation%20of%20an%20SCT_0.pdf))
  that pre-fills IBAN, amount and purpose in the member's banking app. Use the free-text purpose (max 140 chars), not
  the structured ISO 11649 reference: German banking apps pass the free text through, and it ends up in every export
  format.
- **IBAN memory:** when an admin confirms a match, remember counterparty IBAN → member. One member can have several
  IBANs. One IBAN can pay for several members, for example a couple, which must stay a manual split. This does not
  help with ING CSV, which has no counterparty IBAN.
- **Fuzzy name matching** of the counterparty name against member names, for suggestions only.
- **Amount logic:** a payment of N × dues covers N periods, oldest unpaid first. Anything else is flagged.
- **Never auto-book without a code or a known IBAN.** Every other match is a suggestion an admin confirms.

## Recommended MVP path

1. **The admin uploads a file in the web app.** Detect the format by sniffing: camt.053 XML, a known CSV header, or
   MT940 `:20:`/`:61:` lines.
2. **Support camt.053 (version 02 and 08) plus CSV adapters** for the banks real groups use. The first group's bank
   comes first, then the list above as demand appears. Each adapter is a tested mapping from columns to one internal
   `BankTransaction` shape (booking date, value date, amount in cents, counterparty name, counterparty IBAN if any,
   purpose, end-to-end reference if any, source hash).
3. **Parse on the client**, using fast-xml-parser and papaparse in the browser, and send only normalised rows to
   Hasura. The raw file never leaves the device. Use an idempotent upsert on a de-duplication key so that overlapping
   exports are harmless.
4. **Reconcile with the cash ledger** by comparing the opening and closing balance from the file (camt, MT940, ING CSV
   metadata) with the ledger and warning on gaps. DKB and Sparkasse CSV carry no balances, so the admin enters the
   current balance.
5. **Matching:** reference code plus GiroCode, then IBAN memory, then name suggestions, then a manual pool.
   Auto-confirm only on a code or a known IBAN.

Running cost: zero. Legal surface: GDPR only, because we store members' IBANs and names, but no ZAG.

## Later path: automatic sync

In order of preference:

1. **FinTS on the admin's device**, in the Capacitor app, using lib-fints or fints-lib and native HTTP. Register the
   product with the DK first, which is free and takes weeks. PIN and TAN stay on the phone. The app pushes normalised
   rows exactly as a file upload would, so the server side is unchanged. Covers Sparkasse, cooperative banks,
   Deutsche Bank, Commerzbank, ING, DKB and Postbank, but not N26. Get a short legal check of the "user's own
   software" reading before shipping.
2. **A licensed aggregator** if we want sync from the server, N26 support, or no app-side banking code. Get quotes
   from Enable Banking and finAPI. Today it costs at least about €260 a month for an unlicensed customer at finAPI.
   Viable only if the product gets a commercial model or a sponsor.
3. **Never run our own AIS server with stored PINs.** That would mean BaFin registration and insurance under § 34 and
   § 36 ZAG.

## Open questions for the cash-ledger grilling

- Does the ledger hold *only* imported bank transactions, or also cash entries (money collected at the table) that
  never touch the account? If both, how do they reconcile?
- Is the bank import the source of truth, with manual entries as exceptions, or does the admin book manually and the
  import only confirms?
- What is a dues period (monthly, yearly), and how do we treat overpayment, partial payment and paying ahead?
- Who may import: any admin, or a "treasurer" stewardship? Who sees counterparty IBANs and names: all admins, or
  only that person?
- Retention: how long do we keep counterparty IBANs and purpose texts? Do we keep non-member transactions (rent,
  drinks) in full or only the amount and a category?
- One couple paying for two members, or a friend paying for someone: how do we split one transfer across members?
- Which bank and export format does the first real group use? We need one anonymised sample file per bank as a test
  fixture. It must never be committed with real data.
- Is a monthly manual upload acceptable to the first group's admin, or is automatic sync a must-have that would pull
  the FinTS-on-device path into scope earlier?
- Do we want to generate GiroCodes and reference codes in the MVP even before import exists, so members already pay
  with the right purpose text?
