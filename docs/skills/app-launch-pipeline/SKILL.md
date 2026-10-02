---
name: app-launch-pipeline
description: Use this skill whenever the user wants to take a new app idea from concept to a Base44-ready build, or any part of that path — prototyping a concept, researching whether a niche has real demand and low competition, writing a Base44 build prompt, or getting a built app through Google Play compliance. Trigger this proactively any time the user floats a new app idea, asks "is there a market for X", asks to turn something into an app, asks for a Base44 prompt, or pastes a Play Store compliance report — even if they only name one stage, since the stages are meant to run in sequence and each one informs the next. Also trigger when the user asks to validate, scope, or de-risk an app idea before building it.
---

# App Launch Pipeline (Daybreak Labs / Loomwork)

A five-stage process for taking an app idea from "is this worth building" to a working, Play-Store-ready app. Each stage produces an artifact that feeds the next one — don't skip straight to Stage 3 (the Base44 prompt) without at least a lightweight pass through Stages 1 and 2, or the resulting app risks being well-built but aimed at nothing.

**Full pipeline:**
1. Prototype the concept
2. Research the niche (demand vs. competition)
3. Write the Base44 build prompt
4. Build it (Base44 connector)
5. Play Store compliance pass

Jump to whichever stage the user is actually asking about, but mention where it sits in the sequence and offer the adjacent stages.

---

## Stage 1 — Prototype the concept

Before writing a single word of marketing copy or a Base44 prompt, make the idea tangible. The right prototype format depends on what's being validated:

- **A calculation, tracker, or personal tool** (budgets, trackers, planners) → build it as a real spreadsheet (xlsx skill) or, for something interactive, a React artifact with **persistent storage** (`window.storage`) so it behaves like a real app, not a mockup. Populate it with realistic sample data, not placeholder Lorem Ipsum — the person needs to see their actual use case working.
- **A UI/flow concept** → a React artifact is usually enough; persistent storage isn't required if the point is just to validate the interaction, not to actually use the tool day to day.

Why this stage matters: it's cheap (minutes, not a Base44 build) and it forces concrete decisions — what are the actual data fields, what's the actual core loop — that the Base44 prompt in Stage 3 needs to be specific rather than generic. A vague idea produces a vague prompt produces a generic app.

Apply the `frontend-design` skill's design-token discipline even at prototype stage: name a real palette, a real type pairing, a real signature element — never default to the generic AI palettes (cream+terracotta, dark+neon accent, broadsheet hairlines). This matters more than it seems: the visual identity chosen here is usually what carries through to the Base44 prompt's design direction section in Stage 3, so getting it distinctive now saves a rewrite later.

## Stage 2 — Research the niche

Goal: find a niche with **real, evidenced demand** and **genuinely thin competition** — not a guess, and not a generic "market size" statistic (those reports are mostly low-quality SEO content and disagree with each other by 10x; don't lean on them alone).

**Search technique that actually works** (in rough order):
1. Search the direct competitor landscape first: `best [category] apps [current year]`, `[dominant former product] alternatives [current year]` (e.g. "Mint alternatives 2026") — this surfaces who currently owns the space and how saturated it is.
2. Check raw competitor *density*, not just quality: sites like alternativeto.net list every small/indie/open-source competitor in a category, often 50-100+. A category with a huge alternativeto.net list is more crowded than it looks from a "best 10 apps" listicle alone.
3. Mine specific pain-point niches with targeted phrase searches: `reddit "wish there was an app"`, `reddit [pain point] frustration`, `[niche] app "wish"`. These rarely land on a single perfect thread, but the *pattern* across App Store listings, Gumroad templates, and blog posts that keep re-describing the same unmet need is itself the signal — if five different small products are all pitching the same narrow angle, that's evidence of real demand.
4. Where possible, ground demand in a hard number, not vibes: population/prevalence stats, workforce percentages, survey stats from the niche's own advocacy or research orgs (CDC, CHADD-style bodies, Statista, etc. — cite these, don't approximate).

**Scoring niches — demand vs. competition:**
- High demand + thin competition (a real number backing the pain point, and only one or two dedicated, poorly-executed competitors) → best target.
- High demand + many small competitors (long alternativeto.net-style tail, even if none dominate) → deprioritize; the niche is real but the ceiling for a new entrant is low because acquisition cost is high and differentiation is hard.
- High demand + one or two dominant, well-funded competitors → deprioritize; the niche is proven but owned.
- Low/unclear demand → deprioritize regardless of competition.

Present the ranked options plainly, cite every demand claim, and be explicit when market-size numbers conflict across sources (they usually do) rather than picking the one that sounds best.

## Stage 3 — Write the Base44 build prompt

Once a niche is chosen, write a structured prompt using the template in `references/base44_prompt_template.md`. The consistent shape across every prompt in this pipeline:

1. **One-paragraph pitch** — what the app is, who it's for, and the one thing it does that competitors don't.
2. **Data model** — every entity, per-user, with field types. Be specific about which fields are computed live vs. user-entered; Base44 apps should almost never store a manually-entered "total" that could instead be calculated from underlying records.
3. **Pages** — one subsection per screen, describing what's shown and how it behaves, not just a feature list.
4. **Design direction** — a real palette (named hex values), real type pairing, and one signature visual element that's specific to this app's subject, not a generic SaaS look. Each app in this pipeline should look and feel distinct from the others, even when they share a category (see the "Pause" vs. "Tidewater" vs. "The Ledger" prompts as reference — same category, three different visual languages, each justified by what the app actually does).
5. **Behavior notes** — cross-cutting rules: what's always live-calculated, mobile-first constraints, privacy/data-scope rules (per-user RLS), disclaimers where relevant (e.g. "not tax advice").
6. **Standard pages and content** — every app ships the same public trust and support pages, styled in the app's own design: About, Contact, Guide, FAQ, Privacy Policy, Terms and Delete account, linked from the footer and from Settings. It also ships the low-value-content defense: a public landing page, a resources hub of 5–8 original articles, honest comparison pages, and a sitemap and the other search basics. Follow `references/standard_pages.md` for what each page must contain. These aren't optional extras: store review needs the privacy, deletion and support URLs, and users judge a new app's trustworthiness partly from them. Never invent company facts or support addresses; leave clearly marked placeholders and list them for the owner.

Deliver this as a markdown file the user can review/edit before it goes anywhere near Base44 — don't call the Base44 connector unprompted. Only call `create_base44_app` or `edit_base44_app` directly when the user explicitly asks you to run it.

## Stage 4 — Build it

If asked to actually build the app (not just hand over the prompt), use the Base44 MCP connector: `create_base44_app` with the Stage 3 prompt as `appPrompt`, passed close to verbatim (Base44's own builder AI handles further expansion — don't over-elaborate what's already a complete spec). Report the editor URL back immediately, then poll `get_app_status` in the background.

## Stage 5 — Play Store compliance pass

Once the app is built and nearing submission, run it through the standing `playstore-compliance-fixes` skill (already part of this profile) — either proactively while reviewing the build, or when the user pastes a compliance report. That skill owns the actual fix patterns (Optimistic UI Updates, Unified Navigation & Back Stack, and the rest of the checklist); this pipeline just makes sure it's not forgotten as the last gate before shipping.

Also confirm, as part of this gate:
- All standard pages and the low-value-content items (public landing, resources hub, sitemap/robots, per-page meta) from `references/standard_pages.md` exist, are public (no login) and are linked from the footer and Settings.
- The Privacy Policy matches the app's real data flows.
- The in-app "Delete my account" flow and the public `/delete-account` URL both work.
- The support address is real and confirmed by the owner.

---

## Notes for this profile

- Products ship under **Daybreak Labs** (app publishing) or **Loomwork** (client AI/web work) — both DBAs of Northbound Digital LLC. Ask which this app is for if it's not obvious; it affects pricing/positioning framing but not the pipeline itself.
- Standing principle across all products: solve a real user problem, and make the experience seamless and frictionless — this is the filter for Stage 2 (does the niche have a *real* problem) and Stage 3 (is the core loop actually low-friction, not just feature-complete).
- Existing live apps (LunaCycle, TrueTrack, Twine, TrueNetPay) are useful reference points for what's already been tried, tone-of-voice precedent, and design-token precedent (e.g. TrueTrack's "no fake precision" positioning, the "ledger" ink/brass aesthetic used for personal-finance tools) — check `conversation_search` for prior audits/prompts on a given app before assuming a clean slate.
