# Off-Market Outreach Routine — Cold Landowners (CLAUDE.md)

## Purpose

Instruction file for the **daily off-market outreach routine** — texting **cold landowners** (not listing agents) across all active outreach campaigns.

**Campaigns now live in Supabase, not Excel/Google Drive.** Each campaign is a table in the `Outreach app building` Supabase project. Ruben (or anyone with access) can log into the app and see the same live data the routine reads and writes — there is exactly one copy of each lead, no per-county Excel masters, no Drive uploads, no "which file is the latest modified copy" problem.

Runs once per day at **20:00 Amsterdam / 1:00 PM Dallas time**, two phases in this order, per campaign:

1. **PHASE 1 — Conversation Engine**: pull last-24h replies from Quo → classify → **actively reply**, working every lead toward that campaign's qualification goals → write back to Supabase immediately, row by row
2. **PHASE 2 — Send Outreach**: ALL eligible follow-ups (no cap) + up to **15 NEW** first-touch texts

**Campaigns are not interchangeable.** Positioning, the qualification goal set, and the conversation stage machine are all campaign-dependent — see each campaign's section below. Never reuse the Subdivide script on a Tax Delinquent lead or vice versa.

> **Lead sourcing / master-building is NOT part of this routine.** Building a new campaign table or importing a new raw export is a separate job (skills like `subdivide-master-builder` / `off-market-leads-builder`, or a direct Supabase import). This file only covers the day-to-day texting/reply routine against tables that already exist.

---

## Why Supabase (not Excel)

- Every campaign is a table in the **`public` schema** of the Supabase project, queried and updated live via the Supabase MCP tools (`execute_sql`, `apply_migration`) — no export/download/upload cycle, no duplicate files.
- Ruben's app reads the same tables directly, so anything this routine writes is visible to him immediately without a manual refresh step.
- **Do not touch `public.leads`.** It's a small, generic legacy/test table (2 rows, no `reach_method`/DNC columns) — it predates the campaign-table structure and is out of scope for this routine.
- Read column names from the table schema at runtime (`list_tables` with `verbose: true`, or `information_schema`) rather than hardcoding — schemas evolve, and the two live campaigns already use different column names for the same concept (see each campaign's mapping table below).

---

## ⚠️ CRITICAL RULES — READ FIRST (apply to every campaign)

### 1. Inbox rule

**NEVER send any message from phone number `(984) 368-4758`** (Sales inbox). Before sending anything, call `Quo:list-inboxes` and identify the Outreach inbox (any inbox that is NOT `(984) 368-4758`). If only the Sales inbox exists, **abort the entire run** and log:

```
ABORT: Only Sales inbox available. No messages sent. Manual review required.
```

This check is global — done once before touching any campaign.

### 2. DNC rule — the DNC list is NEVER contacted by this routine

Every campaign table has its own DNC mechanism (see that campaign's section), but the principle is universal:

- Only rows flagged as text-eligible **and** clear of DNC may EVER be messaged.
- Rows flagged email-only, social-only, manual-only, or DNC/litigator: **no automated outreach of any kind — no texts, no emails, no fallback.** Manual only, by Ruben.
- **If the 15/day cap is reached, or all mobile numbers have been messaged, the run simply sends fewer/no new messages.** NEVER dip into a DNC-flagged row to fill the quota.
- Prefer known-Mobile numbers when texting.

### 3. Opt-out rule

If ANY inbound reply contains "stop", "unsubscribe", "remove me", "don't text", "do not contact", "leave me alone", or similar:

- `outreach_status = "Do Not Contact"`, `quo_response_category = "No"`, `response_type = "Opt Out"`
- Never message this lead again, any inbox, any number, incl. all other rows for the same owner (search by owner name/phone across the campaign table).

### 4. No numbers from the agent

The agent never states a dollar offer, never negotiates. In the Subdivide campaign it asks for their price (never volunteers ours). In the Tax Delinquent campaign it does not discuss price *at all* — see that campaign's goals below. Numbers, when they happen, are Ruben's job, later, off-script.

### 5. Parcel link rule

Every lead's `parcel_link` (a LandInsight or LandPortal parcel detail URL, depending which platform the batch was sourced from) must be attached to its Quo contact — both for brand-new contacts and for existing ones being reached out to for the first time by this routine:

- **Read `parcel_link` verbatim from the row. Never reconstruct or derive it** — LandPortal links are an opaque encoded query string, not a predictable pattern from the APN like older LandInsight links were. Some campaigns mix both platforms row by row (e.g. the Ellis County Landlocked slice of the Subdivide table uses LandPortal while Kaufman/Van Zandt use LandInsight) — always take whatever string is stored, don't assume a format.
- ⚠️ **The live Quo MCP contact schema has no `notes` field** (`create-contact`/`update-contact` only accept `firstName`, `lastName`, `phoneNumber`, `email`, `company`, `role`) — verified against the actual tool schemas, not assumed. Use the **`company`** field as the parcel-link carrier instead: `APN {apn} | {acreage/lot_acres} ac | {address} | LandPortal: {parcel_link}` (or `LandInsight:` when the row is LandInsight-sourced). It's a free-text field and holds long strings fine (a full LandPortal URL + APN/acreage/address prefix has been confirmed to save and read back intact, no truncation).
- **New contact** (`Quo:create-contact`) → create with name/phone/email as usual, then immediately `Quo:update-contact` to set `company` to the string above (creation itself has no field for it).
- **Existing contact** (`contact_quality = "Existing in Quo"`) → before sending, `Quo:get-contact` and check its `company` field; if no parcel-link line is present, call `Quo:update-contact` to set it (don't overwrite an unrelated existing `company` value if one is ever present — append the parcel-link line to it instead).
- Never send a first-touch or follow-up message to a contact whose Quo record is missing this link. This is a hard gate, always, on every campaign.

---

## Connections & File References

| Resource | Details |
| :---- | :---- |
| **Data store** | Supabase project attached to this Claude Project — query/update via the Supabase MCP tools |
| **Campaign tables** | One table per campaign, see **Active Campaigns** below |
| **Quo MCP** | `https://mcp.quo.com/mcp` |
| **Google Calendar MCP** | For booking the call (list_events / create_event) |
| **Master/table builders** | `subdivide-master-builder` / `off-market-leads-builder` skills (on demand, not part of this routine) — now write into Supabase rather than an .xlsx |

### Active Campaigns

| Campaign | Table | Scope | Positioning | Reach gate |
| :---- | :---- | :---- | :---- | :---- |
| **Subdivide — cold landowners** | `public.subdivide_outreach_leads` | Multi-county: filter by `campaign` column (currently `Kaufman`, `Van Zandt`, `Ellis County Landlocked`) | Land investor/developer, can pay near/full market value | `reach_method` in (`TEXT`,`TEXT+SOCIAL`) AND `dnc`='No' AND `state_dnc`='No' |
| **Tax Delinquent (NC)** | `public.tax_delinquent_leads` | Single campaign (`campaign` = `NC Tax Delinquent`), spans multiple NC counties via `parcel_county` | Split by `deceased`: `Y` → curative title researcher; `N`/blank → direct land investor/developer offer, no market-value promise. Neither branch discusses price by text. | `reach_method`='TEXT' AND `dnc_status`='Clear' AND `source_sheet`='Main Outreach' |

> To add a new campaign: add a row here pointing at its Supabase table, define its positioning/stage machine in a new section below (copy the Tax Delinquent section as a template if it's a distressed/legal-angle campaign, or the Subdivide section if it's a straightforward buy pitch). No other change to this document is needed — every rule, cap, and phase applies identically per campaign. **The 15/day new-first-touch cap and the Sales-inbox check apply per campaign, per run** — and for Subdivide specifically, per `campaign` value (county), per run, since that table holds three counties side by side.

---

# CAMPAIGN 1 — Subdivide (cold landowners)

Table: `public.subdivide_outreach_leads`. Filter every query in this section by `campaign = '{county}'` and run the full routine once per county in the Active Campaigns table before moving to the next.

## 🎯 Qualification Goals

1. **WOULD SELL** — will they sell? (Yes / Maybe / No → `quo_response_category`)
2. **ASKING PRICE** — what number do they have in mind? (→ `counter_amount`). **We want THEIR number** — never volunteer ours, never offer to go first.
3. **CALL** — once 1 and 2 are captured, get a time for a quick call (→ Google Calendar event, `call_date`)

Only then does Ruben get on the sales call to negotiate. The agent NEVER negotiates price or sends offer numbers — that's Ruben's job on the call.

## 💬 Positioning

Ruben is a **land investor / developer**. We are **NOT lowball cash-offer buyers**. Because we develop and subdivide the land (and potentially build later), we can often pay **close to market value — or at market value — if our development numbers allow it**. Lead with this in the INTRO script. It's the core differentiator vs. every other "cash for land" texter they get.

This pitch is safe here because Subdivide rows are not, by definition, distressed/tax-delinquent sellers — the numbers can plausibly clear market value once subdivided. **Do not port this pitch to the Tax Delinquent campaign** — see that campaign's positioning for why.

**Landlocked exception:** any Subdivide `campaign` value whose name contains "Landlocked" (currently `Ellis County Landlocked`, and automatically any future landlocked-designated county slice added the same way — no doc change needed to pick up a new one) does **not** open with the market-value developer pitch above. Access is the seller's actual problem on a landlocked parcel, not price, so lead with that instead — see the **Landlocked opener** under First-touch templates below. Once they engage, the rest of the stage machine (would-sell → price → call) is unchanged; only the opening message differs.

## Table columns (verbatim, `public.subdivide_outreach_leads`)

```
id, reach_method, dnc_status, facebook, linkedin, parcel_link,
apn, acreage, market_value_estimate, mkt_value_per_acre_estimate, road_frontage_ft,
sub_lots, avg_lot_acres, total_acreage, parcel_count,
parcel_address, city, zip, latitude, longitude,
owner_first_name, owner_last_name, email,
phone_1, phone_2, phone_3, phone_4, phone_5, phone_6,
phone_1_type, phone_2_type, phone_3_type,
dnc, state_dnc, tag_subdivide, last_sale_date, last_sale_price,
active_phone, active_phone_index,
quo_message_sent, quo_message_date, quo_response_category, response_date, response_type,
follow_up_1_date, follow_up_2_date, follow_up_3_date,
outreach_status, contact_quality, skip_reason,
offer_made, offer_amount, offer_status, counter_amount,
call_completed, call_date, call_notes, notes,
created_at, campaign, pipeline_stage
```

### Tracking-column semantics

| Column | Written by | Meaning |
| :---- | :---- | :---- |
| `parcel_link` | Table builder | LandInsight or LandPortal parcel detail URL — read verbatim, never re-derive (see Critical Rule 5) |
| `campaign` | Table builder | Which county/sub-campaign this row belongs to (`Kaufman` / `Van Zandt` / `Ellis County Landlocked`) — filter every query by this |
| `pipeline_stage` | Both | Deal-level CRM stage, separate from `outreach_status`. One of: `Leads`, `Underwritten`, `Outreached`, `Offered`, `Follow-up`, `Accepted`, `Rejected`, `Long-term Follow-up` — advance opportunistically alongside `outreach_status` (e.g. first text sent → `Outreached`) |
| `active_phone` / `active_phone_index` | Phase 2 / rotation | Number currently in use + its slot (1–6) |
| `quo_message_sent` | Phase 2 | **Full verbatim text** of the last outbound message (incl. template tag, e.g. "V3: Hi {first}…") |
| `quo_message_date` | Phase 2 | Date of FIRST outbound message — the "already contacted" gate and the anchor for follow-up timing |
| `quo_response_category` | Phase 1 | **No / Yes / Maybe / Follow-up** — the master classification |
| `response_date` | Phase 1 | Date of most recent inbound reply |
| `response_type` | Phase 1 | Opt Out, Wrong Number, Price Named, Wants Our Offer, Call Me, Question, Identity Confirm, Already Sold, Other |
| `follow_up_1/2/3_date` | Phase 2 | Dates templates E / F / G were sent |
| `outreach_status` | Both | **Doubles as the conversation stage** — values below |
| `contact_quality` | Phase 2 | "New" / "Existing in Quo" |
| `skip_reason` | Both | Why a row was skipped (e.g. "All phones wrong/dead") |
| `counter_amount` | Phase 1 | **The owner's ASKING PRICE** (verbatim, e.g. "$450,000" or "$15k/acre" or "Wants our offer") |
| `call_date` | Phase 1 | Scheduled call datetime (Google Calendar event created) |
| `call_completed` / `call_notes` | Ruben | After the sales call |
| `offer_made` / `offer_amount` / `offer_status` | Ruben / Phase 1 detection | Only after Ruben sends a number; if an OUTBOUND thread message contains `$` → `offer_made`="Yes", `offer_amount`=extracted (never overwrite) |
| `notes` | Both | Running conversation log, newest first: "{date} IN: {text} / OUT: {reply}" (append, never overwrite) |

There is no separate verbatim-response column — the conversation log lives in `notes`. Always confirm the live column list at runtime; this section documents it as of the last schema check, not as a frozen contract.

### `outreach_status` values (= conversation stage)

```
(blank)                  not yet contacted
Messaged                 first-touch sent, no reply (stage 0)
Responded                reply received, intro not yet sent (stage 1)
Intro Sent               positioning delivered, awaiting sell answer (stage 2)
Price Asked              sell = Yes/Maybe, asked their price (stage 3)
Call Time Asked          price captured, asked when they can talk (stage 4)
Qualified - Call Booked  goals captured, calendar event created — Ruben takes over (stage 5)
Closed - No              would sell = No (polite close sent)
Sequence Complete        3 follow-ups sent, never replied
Retry Next Phone         wrong number — resend to next phone as new
Already Contacted        prior Quo history found — skipped
Contact Missing          no valid phone left
Do Not Contact           opt-out — permanent
Quo Error                send failed
```

## Conversation scripts

```
# INTRO — first reply after they initially answer (positioning, once per lead)
INTRO = (
 "Hi {first}, Ruben here — I'm a land investor and developer. I'm very interested in making "
 "you an offer on your {ac} acres on {road}. We're not the lowball cash-offer guys: we develop "
 "and subdivide, so if our numbers work we can pay close to market value — sometimes full "
 "market value. Would you consider selling?"
)

# PRICE ASK — after would-sell = Yes/Maybe. We want THEIR number. Never offer ours.
PRICE_ASK = (
 "Great. Do you have a price in mind for the property? If it fits our development numbers "
 "we can usually get close to it."
)

# PRICE NUDGE — one retry if they dodge the number
PRICE_NUDGE = "Even a rough number helps — what would it take for you to sell?"

# CALL ASK — after their price (or 'Wants our offer') is captured
CALL_ASK = "Thanks. When's a good time this week for a quick call to talk it through?"

# CLOSE — would sell = No (not opt-out)
CLOSE = (
 "All good, thanks for letting me know. If that ever changes, keep my number — "
 "we pay close to market value when the numbers work. Take care! — Ruben"
)
```

## Stage machine

```
Messaged → reply received (category ≠ No):
    → Send INTRO (positioning; delivered exactly once per lead)
    → outreach_status = "Intro Sent"
Intro Sent → they answered the sell question:
    - No         → send CLOSE → outreach_status = "Closed - No"
    - Yes/Maybe  → quo_response_category = Yes/Maybe → send PRICE_ASK
                   → outreach_status = "Price Asked"
Price Asked → price answered:
    - Names a number → counter_amount = amount verbatim, response_type = "Price Named"
    - Won't name it  → nudge once with PRICE_NUDGE; if still no number →
      counter_amount = "Wants our offer", response_type = "Wants Our Offer"
    → Send CALL_ASK → outreach_status = "Call Time Asked"
Call Time Asked → they give a time (or say "call me"):
    → Check Google Calendar availability → create 30-min event:
        title: "Land call — {Owner name} — {Acreage} ac {Road}, {County}"
        description: phone, APN, asking price, table row id, parcel link
      (If their time conflicts, propose the nearest free alternative in one short text.)
    → call_date = scheduled datetime
    → outreach_status = "Qualified - Call Booked"
    → No confirmation message. Ruben takes over from here.
```

Skip stages when the owner jumps ahead (e.g. first reply = "I'd take $500k" → write category, `counter_amount`; fold the INTRO positioning into the next message). Category No at ANY stage → CLOSE → "Closed - No" (unless Opt Out → "Do Not Contact").

## First-touch templates — 5 rotating versions, HARD MAX 160 CHARACTERS

> After filling placeholders, verify `len(msg) <= 160`. If over: use street name only for `{road}`; still over: drop the acreage; still over: fall back to V1. `{road}` = street portion of `parcel_address` (or "{city} area" if blank). `{ac}` = `acreage`.

```
V1 = "Hi {first}, I'm a land developer — would you be open to an offer on your land?"
V2 = "Hi {first}, I'm a developer in the area. Can I make you an offer on your land on {road}?"
V3 = "Hi {first}, this is Ruben. Would you sell your {ac} acres on {road}? I pay close to market value."
V4 = "Hi {first}, I'd like to buy your {ac} acres on {road}. Do you have a price in mind?"
V5 = "Hi {first}, Ruben here. I'm buying land near {city} to develop. Would you sell yours for the right price?"
```

V1–V5 all ask the sell/offer question directly, so a plain "yes" = would-sell Yes (and a number back to V4 = asking price captured immediately). A "who is this?" / identity-check reply to ANY opener → category Follow-up (`response_type = "Identity Confirm"`) and triggers the INTRO script before re-asking.

### Landlocked opener — replaces V1–V5 for any "…Landlocked" campaign value

For rows whose `campaign` value contains "Landlocked" (currently `Ellis County Landlocked`), send this in place of V1–V5 as the first-touch message — the pitch is access-focused, not market-value:

```
LL_V1 = "I'm a land investor specializing in landlocked properties. Looking at aerial/CAD maps, {road} appears landlocked. Have you considered selling? — Ruben"
```

Same `{road}` placeholder and fallback as above (street portion of `parcel_address`, else "your land near {city}" if blank). Verify `len(msg) <= 160` after filling; if over, use the city-only fallback: `"I'm a land investor specializing in landlocked properties. Aerial/CAD maps show your land near {city} appears landlocked. Considered selling? — Ruben"`.

This is the ONLY thing that changes for landlocked campaigns — follow-ups (E/F/G below), the stage machine, PRICE_ASK, CALL_ASK, and CLOSE are all unchanged and reused as-is once the lead replies. A "who is this?" reply to LL_V1 still triggers the standard INTRO script (not the market-value line — swap "we develop and subdivide, so if our numbers work we can pay close to market value" for a landlocked-appropriate reason to be interested, e.g. "we specialize in resolving access issues on landlocked parcels") before re-asking would-sell.

### Follow-ups (also ≤160 chars) — 3-touch, matches the 3 date columns

```
E = "Hi {first}, following up about your land on {road}. We're developers, not lowballers — we pay near market value. Open to a chat?"   # 5+ days after initial
F = "{first}, checking in once more on your {ac} acres on {road}. Even a rough idea of your price helps. Worth a quick chat? — Ruben"     # 10+ days after E
G = "{first}, last note about your land on {road}. If selling ever makes sense, we pay close to market value. Keep my number. — Ruben"    # 10+ days after F
```

---

# CAMPAIGN 2 — Tax Delinquent (NC)

Table: `public.tax_delinquent_leads`. One campaign value (`NC Tax Delinquent`), spanning multiple NC counties via `parcel_county` — the 15/day cap applies to the whole table per run, not per county, unless Ruben asks to split it later.

## Why this campaign is different

Every row here is, by definition, tax-delinquent. That changes the deal math: to make one of these work, the price usually has to land **below** market value — there isn't room to pay near/full market value the way the Subdivide pitch promises. Opening with "we pay close to market value" here would be an over-promise the numbers can't back up, on either branch below.

**The `deceased` column is now the fork that decides the whole opening positioning, not just phrasing** — the two branches use genuinely different scripts, not variants of the same one:

- **`deceased = 'Y'`** — assume you may be texting a relative, not the owner of record. Lead softly: **curative title researcher** — someone looking into ownership/title records on the parcel, not a buyer. The first goal is just to confirm who you're actually talking to (owner vs. heir/family vs. unrelated) before pitching anything. This framing exists because it's genuinely unclear who's on the other end of the phone in an estate situation, and "are you the owner" would be the wrong question to lead with.
- **`deceased = 'N'` (or blank/unconfirmed)** — the owner of record is presumed alive and reachable directly, so skip the identity-research framing entirely. Position Ruben plainly as a **land investor and developer** interested in making an offer, and ask directly if they've considered selling — same shape as the Subdivide opener, but **do not promise market value or near-market value here**: these are tax-delinquent sellers, the numbers usually only work at a discount, and over-promising on the opener creates a problem for the call later.

Both branches share the same hard rule: **never ask for a price and never mention a dollar amount in this campaign's texts.** There is no `PRICE_ASK` step on either branch — that's still the single biggest difference from the Subdivide stage machine, deceased or not. The goal of the *texting* routine, either branch, is only to **get a call booked**; price and any actual offer are Ruben's job later, off-script.

## 🎯 Qualification Goals

**`deceased = 'Y'` branch:**
1. **RELATIONSHIP TO PROPERTY** — are they the owner, an heir/relative, or unrelated (wrong number)? (→ `response_type`: "Identity Confirm", "Owner Confirmed", "Heir/Family", or "Wrong Number")
2. **CALL** — once relationship is confirmed and the curative-title framing has been delivered, get a time for a quick call (→ Google Calendar event, `call_date`)

**`deceased = 'N'`/blank branch:**
1. **WOULD SELL** — have they considered selling? (Yes / Maybe / No → `quo_response_category`) — no identity-confirmation step, the opener already addresses them directly as the owner
2. **CALL** — once willing (Yes/Maybe), get a time for a quick call (→ Google Calendar event, `call_date`)

Neither branch asks price, takes a counter, or makes an offer over text. `offer_made` / `offer_amount` / `offer_status` / `counter_amount` exist on this table for Ruben to fill in **after** the call, during the manual curative/offer phase — this routine never writes to them.

## Table columns (verbatim, `public.tax_delinquent_leads`)

```
id, campaign, source_sheet, reach_method, dnc_status, deceased, facebook, linkedin,
apn, lot_acres, total_acreage_owner, parcel_count_owner, primary_parcel, sub_lots, avg_lot_acres,
owner_1_full_name, owner_2_full_name, owner_1_first_name, owner_1_last_name,
mail_full_address, mail_city, mail_state, mail_zip,
parcel_full_address, parcel_city, parcel_state, parcel_county, parcel_zip,
land_use, road_frontage, total_market_value, total_assessed_value, tax_amt, tax_delinquent_year,
zoning, subdivision_name, age, land_locked, latitude, longitude, parcel_link,
active_phone, active_phone_index, email_1, email_2,
phone_1..phone_7, phone_1_type..phone_7_type,
skip_reason, quo_message_sent, response_date, response_type,
follow_up_1_date, follow_up_2_date,
outreach_status, contact_quality, notes,
offer_made, offer_amount, offer_status, counter_amount,
call_completed, call_date, call_notes, pipeline_stage, raw, created_at, property_id,
quo_response_category
```

> ⚠️ **Legacy columns — do not write to these:** `quo_message` and `quo_response` are older/duplicate columns left over from schema migration, both currently empty. Always write to `quo_message_sent` and `quo_response_category` instead (same names as the Subdivide table), so the two campaigns stay consistent.
>
> ⚠️ **Known schema gap:** unlike Subdivide, this table has **no `quo_message_date` column** and **no `follow_up_3_date` column**. Until `quo_message_date` is added, derive "days since first touch" by parsing the earliest dated `OUT` line out of `notes` (same log format used everywhere: "{date} OUT V{n}..."). The missing third follow-up slot is not a bug to route around — this campaign is deliberately a **2-touch** follow-up sequence (E → F only, no G), which fits the softer, relationship-first positioning better than Subdivide's 3-touch push. If Ruben wants true parity with Subdivide later, `quo_message_date timestamptz` and `follow_up_3_date timestamptz` can be added to this table with a simple `ALTER TABLE`.

### Tracking-column semantics

| Column | Written by | Meaning |
| :---- | :---- | :---- |
| `parcel_link` | Table builder | LandPortal parcel detail URL — opaque encoded string, read verbatim, never re-derive (see Critical Rule 5) |
| `source_sheet` | Table builder | `Main Outreach` (eligible) or `DNC - Manual Outreach` (never automated — Ruben only, no texts/emails/fallback) |
| `dnc_status` | Table builder | `Clear` (eligible), or `DNC` / `Litigator` / `No Data` (never automated) |
| `deceased` | Table builder / Phase 1 | `Y`/`N` — whether the owner of record is deceased; drives opener phrasing. Phase 1 may flip this to `Y` if a reply confirms it, as a data-quality improvement |
| `active_phone` / `active_phone_index` | Phase 2 / rotation | Number currently in use + its slot (1–7) |
| `quo_message_sent` | Phase 2 | Full verbatim text of the last outbound message (incl. template tag) |
| `quo_response_category` | Phase 1 | No / Yes / Maybe / Follow-up — same classification meaning as Subdivide, reinterpreted for this campaign's goals (see below) |
| `response_date` / `response_type` | Phase 1 | Date + finer-grain type of the most recent inbound reply |
| `follow_up_1/2_date` | Phase 2 | Dates templates E / F were sent (2-touch only, see gap note above) |
| `outreach_status` | Both | Conversation stage — values below |
| `contact_quality` | Phase 2 | "New" / "Existing in Quo" |
| `skip_reason` | Both | Why a row was skipped |
| `call_date` | Phase 1 | Scheduled call datetime |
| `call_completed` / `call_notes` | Ruben | After the call |
| `offer_made` / `offer_amount` / `offer_status` / `counter_amount` | Ruben, manual, post-call | Never written by this routine — the automated texting stage never discusses price |
| `pipeline_stage` | Both | Same enum as Subdivide: `Leads`, `Underwritten`, `Outreached`, `Offered`, `Follow-up`, `Accepted`, `Rejected`, `Long-term Follow-up` |
| `raw` | Table builder | Original source-record JSON backup — reference only, not written by this routine |
| `property_id` | Table builder | LandPortal's internal property ID (distinct from `apn`) — reference only |
| `notes` | Both | Running conversation log, newest first: "{date} IN: {text} / OUT: {reply}" (append, never overwrite) |

### `outreach_status` values (= conversation stage)

```
Not Contacted            not yet contacted (table default)
Messaged                 first-touch sent, no reply (stage 0)
Responded                reply received, relationship not yet confirmed (stage 1)
Identity Confirmed       [deceased='Y' only] owner or heir/relative confirmed, curative intro not yet sent (stage 2)
Intro Sent               [deceased='Y' only] curative-title positioning delivered, call not yet asked (stage 3)
Call Time Asked          asked when they can talk (stage 4)
Qualified - Call Booked  call scheduled, calendar event created — Ruben/curative team takes over (stage 5)
Not Owner / No Relation  confirmed no connection to the property — close politely, no further texts
Closed - No              declined to engage further (not an opt-out)
Sequence Complete        2 follow-ups sent, never replied
Retry Next Phone         wrong number — resend to next phone as new
Already Contacted        prior Quo history found — skipped
Contact Missing          no valid phone left
Do Not Contact           opt-out — permanent
Quo Error                send failed
```

## Conversation scripts

> **Always mention the specific property, every message, this campaign.** Define
> `{loc}` = `"on {street}, {county} Co"` using the street portion of `parcel_full_address`
> (e.g. "Mt Pleasant Rd S") when it's non-blank, else fall back to `"in {county} County"`
> alone (`{county}` = `parcel_county`). Never send a message that only says "a property" /
> "this property" with no location at all — the county-only fallback is the floor, not an
> opt-out. Every first-touch, follow-up, and TD_INTRO below uses `{loc}`.

```
# --- DECEASED = 'Y' BRANCH — curative title framing. ≤160 chars. ---
TD_V1_D = "Hi, I'm researching title records for a property {loc} listed under {owner_last}. Are you a family member or connected to the estate?"

# --- DECEASED = 'N' / blank BRANCH — direct land-investor pitch, no market-value promise. ≤160 chars. ---
TD_LIVE_V1 = "Hi {first}, this is Ruben — I'm a land investor and developer. I'm interested in making an offer on your land {loc}. Have you considered selling?"
```

> Only one variant per branch exists today — there's no rotation to manage, `Template: {tag}`
> is just whichever branch's single template applies to that row's `deceased` value. If more
> variants are added later to either branch, rotate within that branch only (never mix D and
> LIVE variants on the same row).
>
> After filling placeholders, verify `len(msg) <= 160`. If over (long street name): drop the
> ", {county} Co" suffix from `{loc}` and keep just "on {street}"; still over: fall back to the
> county-only form `"in {county} County"`; still over (TD_LIVE_V1 only, TD_V1_D is already
> shortest): drop "and developer" and "have you considered selling" → "Interested in selling?"
> This mirrors the Subdivide campaign's own truncation ladder (see V1–V5 above) — property
> mention is required, but it degrades gracefully rather than blowing the character cap.

```
# INTRO — deceased='Y' branch ONLY, after relationship is confirmed (owner or heir/family), once per lead
TD_INTRO = (
 "Thanks for confirming. I'm a curative title researcher — I help sort out ownership/title "
 "issues on the parcel {loc}. No offer, no pressure — just background. Got a few mins this "
 "week for a quick call?"
)

# CALL ASK — both branches, if they haven't given a time yet
TD_CALL_ASK = "When's a good time this week for a quick call?"

# CLOSE — both branches, they engaged but don't want to continue (not an opt-out keyword)
TD_CLOSE = (
 "No worries, thanks for confirming. If anything changes or you have questions about the "
 "property down the road, feel free to reach out. Take care!"
)

# NOT OWNER / NO RELATION — deceased='Y' branch ONLY, they say wrong number / no connection to the parcel
TD_NOT_OWNER = "Got it, sorry to bother you — thanks for letting me know, and take care."
```

## Stage machine

Two separate stage machines, selected once per row by `deceased` at first-touch time and followed through to close — a row doesn't switch branches mid-conversation unless a reply reveals the owner is deceased (see note below).

```
### deceased = 'Y' branch — identity confirmation first
Messaged → reply received (not opt-out, not wrong-number-final):
    - Confirms owner or heir/family        → response_type = "Owner Confirmed" / "Heir/Family"
                                              → send TD_INTRO → outreach_status = "Identity Confirmed"
    - Says no relation / wrong person       → send TD_NOT_OWNER → outreach_status = "Not Owner / No Relation"
    - Unclear (e.g. "who is this?")         → response_type = "Identity Confirm", ask again plainly
Identity Confirmed → after TD_INTRO delivered:
    → outreach_status = "Intro Sent"
Intro Sent → they respond:
    - Willing to talk    → send TD_CALL_ASK → outreach_status = "Call Time Asked"
    - Not interested      → send TD_CLOSE → outreach_status = "Closed - No"
Call Time Asked → they give a time (or say "call me"):
    → Check Google Calendar availability → create 30-min event:
        title: "Curative title call — {Owner name} — {County}, NC"
        description: phone, APN, relationship (owner/heir), table row id, parcel link
      (If their time conflicts, propose the nearest free alternative in one short text.)
    → call_date = scheduled datetime
    → outreach_status = "Qualified - Call Booked"
    → No confirmation message, no price talk. Ruben/curative team takes over from here.

### deceased = 'N' / blank branch — opener already asks would-sell directly, no identity step
Messaged → reply received (not opt-out, not wrong-number-final):
    - Willing/interested ("yes", "maybe", "depends", "how do I sell")
                                            → quo_response_category = Yes/Maybe, response_type = "Interested"
                                              → send TD_CALL_ASK → outreach_status = "Call Time Asked"
                                              (skip Identity Confirmed / Intro Sent — TD_LIVE_V1 already delivered the pitch)
    - Not interested / No                   → send TD_CLOSE → outreach_status = "Closed - No"
    - Unclear (e.g. "who is this?", "why?") → response_type = "Question", answer plainly
      (e.g. "I buy land in the area and came across your property") without asking price, then
      re-ask "Have you considered selling?"
Call Time Asked → they give a time (or say "call me"):
    → Check Google Calendar availability → create 30-min event:
        title: "Land call — {Owner name} — {County}, NC"
        description: phone, APN, table row id, parcel link
      (If their time conflicts, propose the nearest free alternative in one short text.)
    → call_date = scheduled datetime
    → outreach_status = "Qualified - Call Booked"
    → No confirmation message, no price talk. Ruben takes over from here.
```

If a reply on the `deceased = 'N'`/blank branch reveals the owner is actually deceased, set `deceased = 'Y'` and switch to the `deceased = 'Y'` branch from that point on (send TD_INTRO next, not TD_CALL_ASK) rather than restarting from a fresh first-touch. Category No at ANY stage, either branch → TD_CLOSE → "Closed - No" (unless Opt Out → "Do Not Contact"). "No relation" is *not* an opt-out — it just ends outreach on this specific row; it doesn't touch other rows for a different owner.

## Follow-ups (also ≤160 chars) — 2-touch, matches the 2 date columns

```
E = "Hi {first}, following up — still trying to confirm the right contact for the property record {loc}. Are you connected to it?"   # 5+ days after initial
F = "{first}, last note on this — if you're the owner or related to the property {loc}, I'd appreciate a quick reply either way."        # 10+ days after E
```

After F with no response: `outreach_status = "Sequence Complete"`.

---

# DAILY ROUTINE

## Startup Sequence

```
1. Quo:list-inboxes → OUTREACH_INBOX_ID (NOT (984) 368-4758; else ABORT — global check,
   done once before touching any campaign)
2. For EACH campaign in the Active Campaigns table, run steps 2a-4 through Phase 2 before
   moving to the next campaign (and for Subdivide, once per `campaign` column value):
   2a. Confirm the live column list for that table (list_tables verbose, or
       information_schema.columns) — don't assume this doc's column dump is still exact
3. Build phone→row index over phone_1..phone_N (all non-blank, N=6 for Subdivide, N=7 for
   Tax Delinquent), normalized E.164 (phones stored as text, normalize to "+1..." form)
```

## PHASE 1 — Conversation Engine (check + classify + reply, every day)

Pull `Quo:fetch-messages(inboxId=OUTREACH_INBOX_ID, limit=100)` → last 24h → INBOUND only. For each inbound: match phone → row (any of that campaign's phone columns), across **all** campaign tables — a phone number only belongs to one campaign's rows, but check both before giving up. Unknown number → log + skip.

### 1a. Classify — every reply gets exactly one `quo_response_category`

| Category | Meaning | Signals |
| :---- | :---- | :---- |
| **No** | Won't sell / not interested / won't engage | "not selling", "not interested", "not for sale", "keeping it", "family land", "never", "already sold" (Subdivide) / "not interested", "leave me alone" (Tax Delinquent, both branches) |
| **Yes** | Open to selling (Subdivide) / Tax Delinquent — meaning depends on branch: `deceased='Y'` → confirmed owner or heir and willing to talk; `deceased='N'`/blank → directly open to selling | "yes I'd sell", "interested", "make me an offer", names a price (Subdivide) / "yes that's me", "I'm his daughter", "yes I own it" (Tax Delinquent, `deceased='Y'`) / "yes", "sure", "I'd consider it", "how much" (Tax Delinquent, `deceased='N'`/blank) |
| **Maybe** | On the fence / conditional | "maybe", "depends", "possibly", "might consider", "what kind of offer" |
| **Follow-up** | Answered but not the current goal yet | "who is this?", questions, "call me later", unclear |

Safety overrides (checked FIRST, both campaigns):

- **Opt Out** keywords → category `No`, `response_type = "Opt Out"`, `outreach_status = "Do Not Contact"`, stop forever.
- **Wrong Number** ("wrong number", "not me", "don't own that") → `response_type = "Wrong Number"`, rotate to next phone (prefer Mobile) via `active_phone`/`active_phone_index`, `outreach_status = "Retry Next Phone"`. Exhausted → `Contact Missing` + `skip_reason = "All phones wrong/dead"`.
  - Tax Delinquent-specific: if they say they're simply *not* the owner or a relative (as opposed to a dead number), that's `outreach_status = "Not Owner / No Relation"`, not a phone-rotation case — there's no reason to try their other phones for someone else's property.

Write on every inbound: `quo_response_category`, `response_date`, `response_type`, append verbatim to `notes` log ("{date} IN: {text}"), `outreach_status` per that campaign's stage machine.

### 1b. Reply — follow each campaign's stage machine (see Campaign sections above)

After classifying, send the next script message via `Quo:send-message` on `OUTREACH_INBOX_ID` and append "OUT: {reply}" to `notes`. Conversation replies have **no daily cap** — always answer every reply the same day, in both campaigns.

### 1c. End of Phase 1

Write back to Supabase immediately, row by row, as each reply is handled — there is no "save the workbook" step; every `UPDATE` is already durable and already visible in Ruben's app the moment it commits.

## PHASE 2 — Send Outreach

### Eligibility (hard gate per row, per campaign)

**Subdivide** (`public.subdivide_outreach_leads`, filtered to one `campaign` value at a time):

```
reach_method in ('TEXT', 'TEXT+SOCIAL')
AND dnc = 'No' AND state_dnc = 'No'
AND outreach_status not in ('Do Not Contact', 'Contact Missing', 'Closed - No',
                            'Qualified - Call Booked', 'Sequence Complete')
AND quo_response_category is null
```

**Tax Delinquent** (`public.tax_delinquent_leads`):

```
reach_method = 'TEXT'
AND dnc_status = 'Clear'
AND source_sheet = 'Main Outreach'
AND outreach_status not in ('Do Not Contact', 'Contact Missing', 'Closed - No',
                            'Not Owner / No Relation', 'Qualified - Call Booked',
                            'Sequence Complete')
AND quo_response_category is null
```

When the 15/day cap is hit or every eligible mobile number has been messaged, the run is done — **DNC-flagged or non-text-eligible rows are never used as fallback** (Critical Rule 2).

### Caps

- **Follow-ups**: no cap — send ALL eligible (Subdivide: 3-touch E→F→G; Tax Delinquent: 2-touch E→F)
- **New first-touch texts**: exactly **15 per run**, per campaign (fewer if the eligible pool runs out) — for Subdivide, 15 per `campaign` value (county); for Tax Delinquent, 15 across the whole table
- **Conversation replies (Phase 1)**: no cap

### LOOP A — Follow-ups (first, no cap)

Candidates: `quo_message_date` filled (Subdivide) / earliest `OUT` date parsed from `notes` (Tax Delinquent, until `quo_message_date` exists) AND `quo_response_category` empty AND eligible.

Sort: oldest first-touch date first.

**Subdivide (3-touch):**
```
FU2 sent and ≥10 days ago            → Template G → follow_up_3_date = today
FU1 sent and ≥10 days ago (no FU2)   → Template F → follow_up_2_date = today
initial ≥5 days ago (no FU1)         → Template E → follow_up_1_date = today
else skip (window not open)
```
After G with no response: `outreach_status = "Sequence Complete"`.

**Tax Delinquent (2-touch):**
```
FU1 sent and ≥10 days ago (no FU2)   → Template F → follow_up_2_date = today
initial ≥5 days ago (no FU1)         → Template E → follow_up_1_date = today
else skip (window not open)
```
After F with no response: `outreach_status = "Sequence Complete"`.

### LOOP B — New first-touch (stop at 15)

Candidates: `quo_message_date` empty / `outreach_status` is `Not Contacted`, blank, or `"Retry Next Phone"` AND eligible.

**Pool-exhaustion check (before sending anything):** count the candidates. If the count is **0** — no eligible lead left unmessaged in this campaign (or this county, for Subdivide) — set `LEAD POOL EXHAUSTED = true` for this run:

- Do NOT run Loop B at all this run.
- Phase 1 (react to replies) and Loop A (follow-ups) still run normally — exhaustion only stops brand-new first-touch outreach.
- Surface it loudly in the Daily Run Summary so Ruben sees it every day the pool stays empty.
- Re-check every run: the moment new rows are added to the table, Loop B resumes automatically — no manual re-enable needed.

**Subdivide sort:** `PRIORITY` desc, then `sub_lots` desc, where PRIORITY = `+3` absentee owner (mailing city/state ≠ property city/state, if tracked) `+2` tax delinquent (if ever added to this table) `+2` `sub_lots ≥ 10` `+1` `last_sale_date > 10 years ago` `−5` landlocked.

**Tax Delinquent sort:** `PRIORITY` desc, where PRIORITY = `+3` absentee owner (`mail_state` ≠ `parcel_state` OR `mail_city` ≠ `parcel_city`) `+2` `tax_delinquent_year` ≥5 years ago `+1` deceased = 'Y' (heir situations often more motivated) `−5` `land_locked` = 'Yes'.

Per row:

```
a. Send phone: active_phone if set, else first phone_N_type = Mobile, else phone_1 → E.164;
   fail → outreach_status = "Contact Missing", skip (no count)
b. Quo:list-contacts by phone:
   - found + prior history in Outreach inbox → outreach_status = "Already Contacted", skip (no count)
   - found, no history → contact_quality = "Existing in Quo"; Quo:get-contact and check `company`;
     if it doesn't already contain a parcel-link line → Quo:update-contact to set/append it
   - not found → Quo:create-contact(name, phone, email?) then Quo:update-contact(company =
     "APN {apn} | {acreage/lot_acres} ac | {address} | LandInsight/LandPortal: {parcel_link}")
     → contact_quality = "New"
     (create-contact has no field for this — the follow-up update-contact call is required, not optional)
c. Template: Subdivide (and its Landlocked slice) — rotate the campaign's first-touch versions
   by send order (continue rotation across runs using count of already-sent rows mod number of
   versions). Tax Delinquent — no rotation, select by `deceased`: `deceased='Y'` → TD_V1_D,
   else → TD_LIVE_V1.
d. ⚠️ Final inbox check: NOT (984) 368-4758
e. Quo:send-message → write back IMMEDIATELY, row by row:
     quo_message_date = today (Subdivide only — Tax Delinquent: log the date in notes instead
       until the column exists) | quo_message_sent = "V{n}: {full body}"
     outreach_status = "Messaged" | active_phone = phone used | active_phone_index = slot
     notes += "{date} OUT V{n} to slot {i} via inbox {ID}"
f. new_counter += 1  (stop at 15)
```

### After both loops

No workbook to upload — Supabase writes are already durable. Print the run summary.

---

## Error Handling

| Situation | Action |
| :---- | :---- |
| Only Sales inbox `(984) 368-4758` available | **ABORT entire run** |
| Row is email-only / social-only / manual-only, or DNC-flagged (`dnc`/`state_dnc`='Yes', or `dnc_status` in ('DNC','Litigator','No Data'), or `source_sheet`='DNC - Manual Outreach') | **Never contacted by this routine** — manual only |
| 15-cap reached, pool not exhausted | End Loop B for today only — never fall back to DNC-flagged rows — try remaining pool next run |
| Eligible pool exhausted (0 leads left unmessaged) | `LEAD POOL EXHAUSTED = true` — skip Loop B entirely, flag prominently in the Daily Run Summary every run until new leads are added; Phase 1 and Loop A keep running |
| Inbound "stop"-type reply | `Do Not Contact` — permanent, all rows for that owner |
| Phone invalid / normalization fails | `Contact Missing`, try next phone next run |
| Wrong Number reply | Rotate next phone (prefer Mobile); exhausted → `Contact Missing` |
| Tax Delinquent: confirmed no relation to property | `outreach_status = "Not Owner / No Relation"` — stop texting this row only, not a DNC/opt-out |
| Reply can't be confidently classified | Category `Follow-up`, ask a clarifying question |
| Owner's proposed call time conflicts with calendar | Propose nearest free alternative in one short text |
| Calendar event creation fails | `notes` += "CAL ERROR — book manually", flag in summary |
| `quo_message_date` (or, for Tax Delinquent, the parsed first-OUT date) filled | Skip in Loop B (unless "Retry Next Phone") |
| `quo_response_category` filled | Phase 1 territory — never send Loop A/B messages |
| Prior Quo history for the number | `Already Contacted`, skip |
| Quo API error on send | `outreach_status = "Quo Error"`, error in `notes` |
| Land Locked = Yes | Deprioritized, note "landlocked — access diligence" if messaged |
| Supabase write fails | Retry once; else log locally and flag in summary — never silently drop a write |
| Unknown inbound number | Console log, skip |

---

## Daily Run Summary Output

```
=== OFF-MARKET OUTREACH — {date} — {Campaign} ({county/scope if applicable}) ===
INBOX USED: {outreach phone} (ID: {OUTREACH_INBOX_ID})
⚠️  Sales inbox (984) 368-4758 was NOT used.

--- PHASE 1: Conversations ---
Replies processed: {n}   (No: {n} | Yes: {n} | Maybe: {n} | Follow-up: {n})
Opt-outs: {n} | Wrong numbers rotated: {n}
Replies sent: {n}
🎯 Goals captured today:
  Subdivide      → Would-sell answers: {n} | Asking prices: {n} | Calls booked: {n}
  Tax Delinquent → Relationships confirmed: {n} | Calls booked: {n}
Qualified leads (call booked, ready for Ruben):
  {owner} — {campaign-specific detail} — call {appt datetime}

--- PHASE 2: Outreach ---
New texts sent:     {new_counter}/15  (template rotation)
[if LEAD POOL EXHAUSTED] 🚨 NEW LEAD POOL EXHAUSTED — 0 eligible leads left unmessaged in
{table}. No new first-touch texts sent today. Add more leads to resume. Still running:
Phase 1 replies + Loop A follow-ups.
Follow-ups sent:    E: {n} | F: {n} | [G: {n} — Subdivide only] (no cap)
Skipped: Already Contacted {n} | Contact Missing {n} | Waiting {n}
DNC/manual-only rows contacted: 0 (always)
Supabase: {table} — {n} rows updated
==========================================
```
