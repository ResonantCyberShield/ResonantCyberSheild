# Base44 Build Prompt Template

Use this shape for every Stage 3 prompt. Copy the structure, not the specific content — every section below needs to be re-derived from the actual app being built, never filled in generically.

---

Build a [type of app] called "[Name]" for [specific audience]. [One or two sentences on the exact problem it solves and what makes it different from the obvious competitors — name the difference concretely, e.g. "unlike X, which requires linking a bank account, this app is manual-entry by design."]

Require user accounts (email login) if the app stores personal data. All data is private per user unless the app is explicitly collaborative (state the sharing model explicitly if so — who can see what).

## Data model

One entity per bullet, in the pattern: **EntityName** — per user: field (type), field (type, enum values if applicable), field (computed — describe the calculation inline rather than storing it).

Rules:
- Never spec a field as user-entered if it can be derived from other stored records (totals, percentages, running balances). Say explicitly "computed, not stored" for these.
- Every dropdown/enum field should either list its exact allowed values or say "user-editable list" if it's meant to grow with use.
- If the app needs seed/sample data so it isn't empty on first login, say so explicitly and describe roughly what the sample data should look like (a realistic, working example — not zeros).

## Core calculation (if the app has one signature number)

Some apps have one number that's the whole point (a "safe to spend" figure, a financial score, a completion estimate). If so, give it its own section: define the exact formula, in plain language, before describing any page — every page-level description should be able to reference this section rather than re-deriving the formula.

## Pages

One numbered subsection per screen. For each:
- What's shown (be specific about which fields/computed values appear, not just "a dashboard")
- What the user can do here (CRUD operations, filters, sort order)
- Any state/behavior rules specific to this page (e.g. "the current period is always calculated live and visually distinguished from editable history")

## Design direction

This section should never read as interchangeable with another app's. Include:
- A named palette: 4-6 hex values with what each is *for* (background, primary text/accent, a "good" color, a "warning/attention" color — name it "attention" not "error" if the app's tone should avoid alarm language).
- A named type pairing: a display/headline face, a body face, and a data/mono face if the app shows tabular numbers — each chosen for a reason tied to the subject, not "clean and modern."
- One signature visual element: the single thing this app will be remembered by (a specific chart treatment, an animation, a metaphor carried through the UI — e.g. a "smoothing line" motif for an income-volatility app, a "jar filling up" animation for a savings-goal app). Justify it in one sentence.
- Explicit avoid-list: name the generic AI-default look this app should NOT resemble (e.g. "not a clinical SaaS dashboard," "not a cream+terracotta template look").

## Behavior notes

Cross-cutting rules that apply everywhere, typically:
- What must always be a live calculation vs. what's legitimately user-entered.
- Data privacy/scope (per-user RLS; explicit if anything is ever shared).
- Any required disclaimers (financial/medical/legal — state plainly, e.g. "not tax advice, consult a professional").
- Mobile-first constraints if the primary use context is on-the-go (say what must be visible without scrolling, what needs to be reachable in N taps).
- Autosave/feedback expectations (a subtle save indicator, optimistic updates for low-stakes actions — see the `playstore-compliance-fixes` skill for the concrete pattern once this reaches Base44).

---

## Quality bar before handing a prompt to the user

- Could this prompt be mistaken for a different app in the same category if you swapped the name? If yes, the design direction and core-loop description aren't specific enough yet.
- Does every page description reference *this app's* data model, or could it be pasted into a generic CRUD app unchanged?
- Is there at least one sentence explaining why the chosen visual metaphor fits *this* subject specifically?

## Standard pages

Every prompt ends with this section. Keep the wording, but adapt the copy to the app:

All of the following routes are public (no login), use the app's own design tokens, have their own SEO title and description, and are linked from the footer and from Settings. Follow references/standard_pages.md for what each one contains:
- /about
- /contact (form to [SUPPORT EMAIL, owner to confirm], 48h reply promise)
- /guide (how to use each core feature, plus honest limits)
- /faq (8–14 expandable questions + FAQPage JSON-LD)
- /privacy (must match the real data flows)
- /terms
- /delete-account (public deletion instructions and request form)

## Monetization

Default, from references/monetization_default.md:
- Free forever, with ads in compliant AdSlot placements; Pro is ad-free.
- A 14-day free trial that starts on the first Pro tap, one per person.
- Pro features: [list].
- Ad placements: [this app's content screens].
- Blocked ad categories: [categories that conflict with the app's subject].
- Entitlement is set only by the server, from store/Stripe events.
