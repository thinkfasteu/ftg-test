# FTG Frankfurt – Automated Social Media Content Generator (Kickoff Package)

> Version 1.0 · 26 Aug 2025 · Owner: FTG AI Modernization · Tool: Automated Social Media Content Generator

---

## 1) Executive Summary

A lightweight, compliance-first content system that drafts, routes for approval, and publishes consistent FTG-branded updates across channels. It automates **inputs** (e.g., class changes from the Live Plan), **outputs** (multichannel variants), and **governance** (role-based approvals, audit log), while learning from performance to suggest adjustments.

---

## 2) High-Level Technical Architecture

**Frontend**

* **Planner Dashboard (web)**: Next.js/React or Vue (SPA). Modules: Calendar, Queue, Draft Preview, Approvals, Media Library.
* **Draft Preview Pane**: Channel-specific renderers (IG Feed 1:1, Story 9:16, YT Shorts 9:16, Web Ticker, Newsletter block).
* **Role-based UI**: Surfaces only relevant queues/actions per role (Gym Manager, General Manager, Social Media Manager).

**Backend**

* **API Layer**: Node/Express or Python/FastAPI. Authentication via OAuth (Google/Microsoft) + RBAC.
* **Scheduler**: Cron (e.g., Supabase Edge Functions/Cloud Run) + manual triggers for campaigns.
* **Ingestion**: Live Plan watcher (Excel/PDF on website) → parser → normalized events.
* **Content Engine**: Template + rules → drafts (per channel variant).
* **Media Library**: Object storage (Supabase/AWS S3) with metadata and rights/consent.
* **Publish Integrations**: Instagram/Facebook (Meta Graph), YouTube Data API, Kurabu posting API or upload, Website ticker (simple endpoint/JSON), Newsletter (Mailchimp/Sendinblue or SMTP).
* **Analytics**: Collector for engagement metrics → warehouse (Postgres) → monthly report generator.
* **Audit & Compliance**: Immutable approval logs; data retention on drafts ≤ 90 days.

**Security & GDPR**

* No member personal data processed.
* Trainer name usage only when **opt-in** flag is true; else default to “unser Team”.
* Enforce asset rights/consent tags; block publish if missing/expired.

---

## 3) Data Model (ERD Overview)

**Core Entities**

* **User**(id, name, email, roles\[])
* **Role**(id, name) → e.g., `gym_manager`, `general_manager`, `social_media_manager`
* **Channel**(id, key, name, specs) → `instagram_feed`, `instagram_story`, `facebook`, `youtube_shorts`, `kurabu`, `website_ticker`, `newsletter`, `whatsapp_broadcast`
* **Asset**(id, url, type, aspect\_ratio, tags\[], rights\_status, consent\_ref, expires\_at)
* **Campaign**(id, name, type, window\_start, window\_end, partner\_logo?, status)
* **ClassChange**(id, date, time, location, class\_name, status, trainer\_optin, trainer\_first, trainer\_last\_initial, change\_type \[cancel, substitute, delay], source\_ref)
* **Post**(id, category, title, body\_md, campaign\_id?, class\_change\_id?, status \[draft, queued, needs\_review, approved, rejected, scheduled, published, expired], created\_by, created\_at)
* **PostVariant**(id, post\_id, channel\_id, payload\_json, media\_ids\[], scheduled\_for, published\_at, publish\_status)
* **Approval**(id, post\_id, role\_required, user\_id?, decision \[pending, approved, rejected], comment, decided\_at)
* **AuditLog**(id, actor\_id, action, entity, entity\_id, diff\_json, created\_at)
* **MetricSnapshot**(id, variant\_id, channel\_metrics\_json, captured\_at)

**Relational Notes**

* `Post` → many `PostVariant` (one per channel).
* `Post` → many `Approval` (rules by category/initiative).
* `Asset` can be linked via `PostVariant.media_ids[]` and tagged for search & compliance.
* `ClassChange` records feed automatic bulletins.

**JSON Schemas (concise)**

```json
// PostVariant.payload_json (example, IG Feed)
{
  "text": "⚠️ Kursänderung: Rückenfit fällt heute aus.",
  "hashtags": ["#FTG", "#Sportfabrik", "#Frankfurt"],
  "link": null,
  "overlay_template": "announcement-bar",
  "visual_tokens": {
    "badge": "CLASS_CHANGE",
    "brand_color": "ftg-blue"
  }
}
```

```json
// Asset example
{
  "id": "ast_123",
  "type": "image",
  "aspect_ratio": "1:1",
  "tags": ["campaign:ScheineFürVereine", "rights:ok", "consent:none"],
  "rights_status": "ok",
  "expires_at": null
}
```

**SQL Sketch (Postgres)**

```sql
create type post_status as enum ('draft','queued','needs_review','approved','rejected','scheduled','published','expired');
create type decision as enum ('pending','approved','rejected');

create table posts (
  id uuid primary key default gen_random_uuid(),
  category text not null, -- class_changes, initiatives, services, promotions, history, health
  title text,
  body_md text,
  campaign_id uuid null,
  class_change_id uuid null,
  status post_status not null default 'draft',
  created_by uuid not null,
  created_at timestamptz not null default now()
);

create table post_variants (
  id uuid primary key default gen_random_uuid(),
  post_id uuid references posts(id) on delete cascade,
  channel text not null,
  payload_json jsonb not null,
  media_ids text[] default '{}',
  scheduled_for timestamptz null,
  published_at timestamptz null,
  publish_status text default 'draft'
);

create table approvals (
  id uuid primary key default gen_random_uuid(),
  post_id uuid references posts(id) on delete cascade,
  role_required text not null,
  user_id uuid null,
  decision decision not null default 'pending',
  comment text null,
  decided_at timestamptz null
);

create table assets (
  id uuid primary key default gen_random_uuid(),
  url text not null,
  type text not null,
  aspect_ratio text,
  tags text[] default '{}',
  rights_status text not null default 'pending',
  consent_ref text null,
  expires_at date null
);
```

---

## 4) Rules Engine (Category Logic)

**Class Changes**

* Source: Live Plan ingestion. On `cancel` or `substitute` or `delay`, create `Post` → auto-generate `PostVariants` for: Website ticker, IG Story, Kurabu, WhatsApp broadcast.
* Text: short, factual bulletin. IG Story adds a community-friendly footer line.
* Trainer naming: if `trainer_optin=true` then `Vorname + NachnameInitiale`; else fallback `unser Team`.
* Always **requires human click** to publish.

**Initiatives & Campaigns**

* Generate kickoff + weekly reminders + closing thanks within campaign window.
* Provide 2–3 alternative drafts per slot.
* Use official campaign graphics + FTG overlay.
* Route approvals per policy (see §7).

**Service Offerings**

* Evergreen posts quarterly + “burst weeks” before seasonal peaks.
* Lifestyle/value tone; stock/in-house assets only.
* Multi-format: static, 10–30s clips, infographics.

**Membership Awards & Promotions**

* Template-driven recurring promos with small variations (colorway, copy angle, badge).
* Auto-schedule follow-up: “Don’t miss out next year”.
* Visuals are graphic/templated, not reportage.

**Historical Blurbs**

* Heritage stories with light visuals; cross-post IG static + Shorts/Reels.
* Keep to 60–90 words; 1–2 visuals or short b-roll.

**Health & Fitness Tips**

* Educational, motivational; keep clips ≤30s, carousels ≤6 cards.
* Cite source internally in metadata; no medical claims.

---

## 5) Branded Template Set (Specs)

**Color/Fonts**: Use FTG brand palette and type (to be loaded). Slogan: *“Fit. Fitter. Sportfabrik.”*

**A) Announcement Ticker (static)**

* Layout: Full-bleed brand bar top/bottom; headline centered.
* Token fields: `{CATEGORY_BADGE}`, `{HEADLINE}`, `{SUBLINE}`.
* Sizes: 1080×1080, 1080×1920, 1200×675.

**B) Campaign Overlay**

* Safe zones for partner logo + FTG logo.
* Token fields: `{CAMPAIGN_NAME}`, `{DATE_RANGE}`, `{CTA}`.

**C) Award Badge**

* Circular or rounded-square badge; variants for “Ehrung”, “Aktion”, “Seasonal”.
* Token fields: `{BADGE_TEXT}`, `{YEAR}`.

**D) Flyer Layout**

* A4 print-ready PDF. Grid: 12-col. Space for QR & contact box.
* Token fields: `{TITLE}`, `{VALUE_PROP}`, `{INCLUSIONS[]}`, `{PRICE}`, `{CTA_URL}`.

**E) Short Clip Lower-Third & Endcard**

* Lower-third: name strip + badge token.
* Endcard: logo lockup + CTA.

---

## 6) Initial Copy Templates (Tokenized)

**Class Change (Ticker/Story)**

* **Headline**: "Kursänderung heute: {CLASS\_NAME} {CHANGE\_TYPE}"
* **Subline**: "{TIME} • {LOCATION} • Trainer: {TRAINER\_OR\_TEAM}"
* **IG Story footer**: "Danke für euer Verständnis! 💙"

**Campaign Kickoff**

* "Startschuss: {CAMPAIGN\_NAME}! Unterstütze uns vom {DATE\_RANGE}. {CTA\_SHORT}"

**Weekly Reminder**

* "Woche {N}: {CAMPAIGN\_NAME} läuft! Schon mitgemacht? {CTA\_SHORT}"

**Closing Thanks**

* "Danke, FTG-Familie! {CAMPAIGN\_NAME} ist beendet – ihr seid großartig. 💙"

**Service Offering (Child-Sitting)**

* "Mehr Zeit für dich: Kinderbetreuung im Sportfabrik – trainiere entspannt, wir kümmern uns. {CTA\_URL}"

**Promotion (30 Tage für 30 €)**

* "Jetzt starten: 30 Tage für 30 €. Teste die Sportfabrik – ohne Risiko. {CTA\_URL}"

**Historical Blurb**

* "Seit {YEAR}: {MICRO\_STORY}. Heute trainieren hier {COMMUNITY\_TIE}. #FTGGeschichte"

**Health Tip**

* "Mini-Bewegungspause: 60 Sekunden {EXERCISE}. Spür den Unterschied – heute noch!"

---

## 7) Role-Based Approval Workflow

**Routing Rules by Category**

* **Class Changes** → *Gym Manager* must approve.
* **Initiatives/Campaigns** → *General Manager* approves; *Social Media Manager* can request edits.
* **Service Offerings** → *Social Media Manager* approves; *Gym Manager* notified.
* **Promotions/Awards** → *General Manager* approves; alt. versioning enabled.
* **History & Health Tips** → *Social Media Manager* approves.

**Decision Flow (example)**

1. Draft created (auto/manual) → status `needs_review`.
2. System assigns required approvals (can be parallel). SLA reminders at 4/24h.
3. Approver actions: **Approve**, **Request changes**, or **Reject** (must choose reason tag).
4. On **Reject**, system offers 2–3 alternative drafts (re-roll). Keeps audit trail.
5. On **Approve**, variants are scheduled/published per channel rules.

**Audit**

* Every step logged with user, timestamp, diff.

---

## 8) Scheduling & Publishing Rules

**Scheduler**

* **Class Changes**: immediate queue; default publish window `now–+60min` after click.
* **Campaigns**: kickoff at window\_start 10:00; weekly reminders Tue 10:00; closing at window\_end 18:00.
* **Evergreen services**: 1×/quarter + bursts (3 posts in the week before a known seasonal peak).

**Channel-Specifics**

* IG Feed: max 1 promo/day; space organic content between.
* Stories: allow multiple/day; prioritize class bulletins.
* YT Shorts: ≤30s MP4, portrait.
* Website ticker: overwrite with most recent; keep 5 latest entries.
* Newsletter blocks: queued to next send cycle.
* WhatsApp broadcast: only for urgent class changes or major announcements.

---

## 9) Example Draft Posts (Ready-to-use)

### A) Class Changes

**Website Ticker**

* "⚠️ Heute 18:00 Rückenfit entfällt. Danke für euer Verständnis."

**Instagram Story (3 frames)**

1. Frame: "Heute: Rückenfit entfällt" (announcement-bar)
2. Frame: "18:00 • Studio 2 • Trainer: unser Team"
3. Frame: "Danke fürs Verständnis 💙 #Sportfabrik"

**Kurabu**

* "Kursänderung heute: Rückenfit (18:00) entfällt."

**WhatsApp Broadcast**

* "⚠️ Kursänderung: Rückenfit 18:00 entfällt – Danke!"

### B) Initiatives & Campaigns (z. B. „Scheine für Vereine“)

**Kickoff IG Feed**

* "Startschuss: Scheine für Vereine! Unterstütze FTG vom 01.–30.09. Hol dir beim Einkauf deinen Vereinsschein & registriere ihn für uns. 💙 #FTG #Frankfurt #ScheineFürVereine"

**Reminder**

* "Woche 2 – schon dabei? Jeder Schein zählt! Danke, FTG-Familie. 💙"

**Closing**

* "Ihr seid spitze! Aktion beendet – tausend Dank für jeden Schein."

### C) Service Offerings (Kinderbetreuung)

**IG Feed**

* "Mehr Zeit für dich: Kinderbetreuung bei uns im Haus. Trainiere entspannt – wir kümmern uns. Öffnungszeiten & Infos im Link."

**IG Story (CTA)**

* "Kinderbetreuung → Heute geöffnet! Details im Profillink."

### D) Promotions (30 Tage für 30 €)

**IG Feed (graphic)**

* "30 Tage für 30 € – jetzt starten, stark fühlen. Nur für Neukund\:innen. Infos: Link im Profil."

**Reel (≤20s script)**

* Hook (3s): Badge + “30/30”
* VO/Text (10s): „Teste uns 30 Tage – volle Studio-Power.“
* Endcard (5s): Logo + CTA.

### E) Historical Blurbs

**IG Static**

* "Seit 18xx gehört die FTG zur Stadtgeschichte: Vom Turnverein zur modernen Sportfabrik – immer für Frankfurt da. #FTGGeschichte"

**Shorts/Reels (≤30s)**

* 3–4 Archivbilder, ruhige Musik, kurze Captions (10–15 Wörter je Bild).

### F) Health & Fitness Tips

**IG Carousel (4–6 cards)**

* Card 1: "60-Sekunden-Pause: Schultern mobilisieren"
* Cards 2–5: 1 Übung pro Karte, 1–2 Sätze Anleitung.
* Card 6: "Speichern & teilen – für später!"

**Reel (≤30s)**

* Demonstration + Lower-third + Endcard.

---

## 10) Media Library & Tagging

* Tags: `topic:*`, `initiative:*`, `rights:*`, `consent:*`, `format:*`, `orientation:*`, `brand:*`.
* Example: `topic:kids`, `initiative:ScheineFürVereine`, `rights:ok`, `format:mp4`, `orientation:9:16`.
* Hard blocks: `rights!=ok` or `consent=missing` → cannot publish.

---

## 11) Compliance Controls

* Drafts auto-delete at day 90.
* Name policy: Trainer names only with opt-in; else “unser Team”.
* No member personal data.
* Partner guidelines: Store campaign brand rules and validate template usage.

---

## 12) Analytics & Feedback Loop

* Ingest metrics per variant: impressions, reach, likes, comments, CTR, saves.
* Attribute class attendance impact where possible (e.g., spikes after announcements).
* Monthly report auto-generated: top formats, best time slots, category performance.
* Suggestion engine: e.g., “Historical blurbs engagement ↑ 30% → add 1/wk.”

---

## 13) API & Webhook Sketch

**Endpoints (selected)**

* `POST /ingest/live-plan` – push new/changed classes.
* `POST /posts` – create post with category & tokens.
* `POST /posts/{id}/variants` – create per-channel variants.
* `POST /posts/{id}/approve` – decision payload with role.
* `POST /publish/{variant_id}` – publish now.
* `GET /metrics/{variant_id}` – fetch latest metrics.

**Webhooks**

* `live_plan.changed` → generate class-change draft.
* `campaign.window.start|week|end` → schedule.
* `approval.reminder` → notify slack/email.

---

## 14) MVP Scope (Phase 1 – 4–6 weeks)

1. **Live Plan ingestion** (Excel/PDF watcher) and **Class Change drafts** with human-click publish to Ticker/IG Story/Kurabu/WhatsApp.
2. **Media Library** with rights tags and basic template set (Announcement, Campaign Overlay, Badge, Endcard, A4 Flyer).
3. **Approval UI** with role routing and audit log.
4. **Campaign scheduler** (kickoff/reminders/closing) with alt-draft re-roll.
5. **Basic analytics** ingestion + monthly PDF report.

---

## 15) Next Steps & Open Items

* Confirm brand assets (logos, palette, fonts) & partner graphic rules.
* Define Kurabu posting method (API vs manual upload endpoint).
* Set up Meta/YouTube app credentials and verify scopes.
* Finalize WhatsApp broadcast policy (urgent-only rubric).
* Collect 50–100 stock/in-house assets and tag them.
* Approver roster & SLAs per category.

---

## 16) Appendix: Sample Payloads

**Class Change → Post creation**

```json
{
  "category": "class_changes",
  "title": "Rückenfit entfällt",
  "body_md": "Heute 18:00 entfällt Rückenfit (Studio 2). Danke fürs Verständnis.",
  "class_change": {
    "date": "2025-08-26",
    "time": "18:00",
    "location": "Studio 2",
    "class_name": "Rückenfit",
    "change_type": "cancel",
    "trainer_optin": false,
    "trainer_first": null,
    "trainer_last_initial": null
  }
}
```

**Generated IG Story Variant**

```json
{
  "channel": "instagram_story",
  "payload_json": {
    "frames": [
      {"layout":"announcement-bar","text":"Heute: Rückenfit entfällt"},
      {"layout":"plain","text":"18:00 • Studio 2 • Trainer: unser Team"},
      {"layout":"footer","text":"Danke fürs Verständnis 💙"}
    ]
  },
  "media_ids": ["tpl_announcement_bar"],
  "scheduled_for": null
}
```

**Approval Decision**

```json
{
  "post_id": "...",
  "role_required": "gym_manager",
  "decision": "approved",
  "comment": "Ok."
}
```
