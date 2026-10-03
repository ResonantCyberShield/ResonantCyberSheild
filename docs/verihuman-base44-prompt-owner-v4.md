# VeriHuman — Base44 Build Prompt (Stage 3)

## One-paragraph pitch

VeriHuman is a mobile app that gives people a second opinion before they trust a stranger they've only met online — someone they're talking to on a dating app, social media, or messaging. It checks photos for AI-generation and camera-metadata signals, analyzes voice clips for the spectral signatures of cloned audio, and scans message text against known romance-scam script patterns. Unlike reverse-image/people-search tools that already crowd this space, VeriHuman's angle is detecting the media itself — the photo, the voice, the words — not looking up who the person claims to be. Free users get unlimited heuristic checks (the same ones running in the companion web tool); Pro users get a real trained deepfake classifier ("Verified Scan"), full-thread message analysis ("Deep Analyze"), and a running trust history on a given contact instead of one-off scans.

## Data model

**User** (per-account)
- `email`, `displayName`
- `subscriptionStatus` (enum: `none`, `trialing`, `active`, `expired`, `canceled`)
- `trialStartedAt` (datetime, set once, lazy-started on first gated-feature tap)
- `trialEndsAt` (computed from `trialStartedAt` + 14 days — never independently editable)
- `subscriptionPlatform` (enum: `ios`, `android`, `web`)
- `createdAt`

**DeviceTrialRecord** (keyed by device, not account — trial-abuse backstop)
- `deviceId` (opaque identifier)
- `trialUsed` (boolean)
- `firstSeenAt` (datetime)
- Stores nothing else; exists solely to prevent a second free trial on the same device.

**Contact** (per-user; the person they're vetting)
- `label` (user-chosen name/nickname — "Mike from the dating app")
- `createdAt`
- `confidenceScore` (computed live from its Scan records — never manually entered)
- `lastScanAt`

**Scan** (per-contact, one per check run)
- `contactId` (ref to Contact; nullable — a scan can be run standalone before being attached to a contact)
- `type` (enum: `photo`, `voice`, `message`)
- `tier` (enum: `free`, `pro` — which engine ran it)
- `score` (0–100 concern score, computed, never user-entered)
- `signals` (array of `{name, value}` — the specific flagged items, same shape as the web tool's signal list)
- `rawInputRef` (for Pro scans only: a pointer to the uploaded file/text actually sent to the server-side classifier; free scans never store the input itself, only the result, matching the web tool's "nothing is uploaded" promise for the free tier)
- `createdAt`

**Subscription events** should be written by the store webhook (StoreKit/Play Billing), not derived independently in app code — the store is the source of truth for `subscriptionStatus`.

## Pages

**1. Home / Dashboard**
Shows the user's Contacts, each as a card with its live-computed Confidence Score (color-coded teal/gold/coral the same way the web tool's verdict states work) and last-scan date. A prominent "New Scan" button starts a check without requiring a Contact first. Empty state (no contacts yet) shows the three check types front and center rather than a blank screen.

**2. New Scan**
Three entry points matching the web tool exactly: Photo, Voice clip, Message. Each explains in one line what it checks before the user commits a file/text, same honest framing as the web tool ("a signal, not a verdict"). Free tier runs the existing heuristic engines on-device where possible; Pro tier routes the same input to the server-side classifier and clearly marks the result as "Verified Scan" vs. the free "Quick Scan" so users always know which engine produced a given score.

**3. Scan Result**
The Confidence Ring is the signature visual element — a circular progress-ring readout (teal low-concern → gold mid → coral high-concern, same palette and thresholds as the web tool: <34 / 34–66 / 67+) with the numeric score at its center. Below it, the same kind of named-signal list as the web tool ("AI-generator markers found: 2", "Natural pauses detected: 1.2% of clip"), and the same disclaimer block ("This is a signal, not a verdict"). An "Attach to a Contact" action lets the user roll this scan into a running Confidence Score for that person instead of letting it be a one-off.

**4. Contact Detail**
Shows one Contact's full scan history as a timeline, with the Confidence Score's trend over time (does this person's score improve as more gets checked, or get worse?) rather than just the latest number. This is the concrete Pro-tier value-add over the free web tool: tracking a person over multiple scans instead of one-off checks.

**5. Upgrade / Pro**
Paywall screen triggered when a free user taps a gated feature (Verified Scan, Deep Analyze, or Contact trend history beyond the most recent scan). Reuses the app's existing type and color tokens rather than introducing a separate visual style. Shows the trial-status banner when `trialing` ("13 days left in your trial," no countdown animation, no red urgency styling). Shows both billing options from the Pricing section below ($9.99/mo and $79.99/yr, annual visually favored as "Best value") side by side, plus a plain feature comparison between free and Pro mirroring the website's pricing section.

**6. Settings / Account**
Subscription status and a "Manage subscription" deep link to the relevant store's subscription page, "Restore purchases," sign-out, and a data export/delete action — per the free-tier-rights rule, every user can export or delete their own Contact and Scan data regardless of subscription state, independent of the Pro/export-report distinction above.

## Pricing — Pro tier

Single Pro tier, two billing options for the same entitlement:

- **Monthly: $9.99/month**
- **Annual: $79.99/year** (works out to ~$6.67/month — show this as "save ~33%" next to the monthly price on the Upgrade/Pro screen, not as an unexplained second number)

This was benchmarked against the closest real competitor — Bitdefender's RealCheck, a consumer deepfake-detection app, prices its tiers at $4.99/mo and $12.99/mo. VeriHuman sits above RealCheck's entry tier because it does more (Deep Analyze full-thread scans, persistent per-contact trust history that RealCheck doesn't offer), but stays well under "people-search/background-check" apps like Social Catfish ($27–80/mo) and BeenVerified (~$33/mo) — those charge more because they're paying for data-broker lookups per search, a different cost structure than VeriHuman's on-device heuristics plus occasional server-side classifier calls. Don't price this like a background-check product; it isn't one.

Both billing options route through the same checkout to the same subscription entity/entitlement — don't model them as two different products with separate feature sets, only two prices for the same Pro access. The Upgrade/Pro screen should show both side by side with the annual option visually favored (e.g. a "Best value" badge). The 14-day free trial applies to whichever option the user picks first.

## Design direction

Same visual identity as the VeriHuman website and free web tool, carried over deliberately rather than reinvented — codename "Night Scanner, Bold." The intensity level matters: color should cover a meaningful amount of each screen (headers, card edges, primary buttons), not just appear as a small accent on an otherwise quiet dark screen.

- **Palette, dark mode (the primary/signature look)**: background `#0B0F1A`, card surface `#141A2C` (not darker-than-background — cards should read as distinctly lifted, colorful objects), primary text `#EAF6FF`, secondary text `#7C94AA`, safe/low-concern electric cyan `#00E5FF` (soft fill `#07312F`), caution/mid-concern amber `#FFC24B` (soft fill `#3A2A0A`), risk/high-concern magenta `#FF3D71` (soft fill `#3A0E1C`), plus a decorative bridge violet `#6C4CFF` used only in gradients, never as a status color.
- **Palette, light mode**: background `#F7F9FB`, surface `#EDF1F5`, primary text `#0E1420`, secondary text `#5B6578`, safe cyan `#008299` (soft fill `#E0F7FA`), caution amber `#B9770E` (soft fill `#FDF1DD`), risk magenta-red `#D81B60` (soft fill `#FCE4EC`).
- **The signature gradient**: `linear-gradient(135deg, #00E5FF 0%, #6C4CFF 55%, #FF3D71 100%)` (swap the magenta endpoint for `#D81B60` in light mode). Use it for: the Dashboard header band, every primary CTA button (filled, not outlined — text on top of the gradient should be near-black `#061018` for contrast, not white), and the Upgrade/Pro screen's hero area.
- **Card treatment**: every card (Contact cards, signal rows) gets a visible colored border tied to its own status color at ~20-30% opacity, not just a neutral border — a "safe" contact card's border tints cyan, a "risk" one tints magenta.
- **Type pairing**: Source Serif 4 for headlines/scores, Inter for body/UI, IBM Plex Mono for every numeric value.
- **Signature element**: the Confidence Ring. In dark mode, give the ring's active arc a strong two-layer glow (e.g. a tight `0 0 16px` plus a wider soft `0 0 32px`, both in the arc's own color) — this should be unmistakable, not subtle. In light mode, drop the glow (it doesn't read on a light background) but keep the colored border/accent treatment so cards still feel colorful rather than flattening to gray.
- Dark is the primary/hero presentation this app is designed around first; light mode keeps the gradient and color-coding but trades glow for crisp colored borders and shadows.

## Behavior notes

- Core actions (running a free Quick Scan, viewing Contacts, viewing past scans) always work regardless of subscription state — never gate the fundamental loop behind an entitlement check.
- Downgrading (trial or subscription lapsing) never deletes or hides historical Scan/Contact data — it only locks forward-looking gated features (Verified Scan, Deep Analyze, trend view beyond the latest scan).
- Trial eligibility is checked against the store's platform account first (StoreKit/Play Billing), with the `DeviceTrialRecord` as a lightweight backstop against delete-and-reinstall abuse — no IP correlation, ad-ID fingerprinting, or phone verification, consistent with VeriHuman's privacy-first positioning.
- Raw data export/deletion is a free-tier right on every account, independent of subscription status.
- Every result screen states plainly that this is a signal, not a verdict — the same honesty pattern already established on the website ("What's actually running, and what isn't") carries over to the app; no feature should imply certainty the underlying check can't actually back up.
- Free-tier scans never leave the device; only Pro-tier "Verified Scan" input is transmitted to a server, and that should be disclosed in-app at the moment it happens, not buried in a settings page.

## Audit checklist — run this before calling it done

Once the app is built, verify these directly rather than trusting that "it's built" means "it's correct":

- **Free-tier privacy promise holds in practice.** Pick one free Quick Scan and actually confirm no network call fires for it — not just that the copy says "nothing is uploaded." If a free scan silently round-trips through a server anyway, the single biggest honesty claim in this product is false.
- **Confidence Score is always computed, never stored as an editable field.** Try to find any path (admin panel, direct entity edit, API) where `confidenceScore` could be set directly instead of derived from Scan rows. If one exists, it's a bug.
- **Trial lazy-start actually waits for a gated-feature tap.** Create a fresh account and confirm `trialStartedAt` stays null through signup and idle use, and only gets set the moment a Pro feature is tapped — not at signup.
- **A lapsed trial/subscription doesn't touch historical data.** Let a trial expire (or simulate it) and confirm existing Contacts and Scans are still fully visible and exportable — only forward-looking Pro actions should be newly blocked.
- **Data export/delete actually works and isn't gated.** Test it on a free account specifically, since it's supposed to be a free-tier right regardless of subscription state.
- **Every result screen's disclaimer text is actually present**, not just in the design spec — check the Scan Result screen renders "this is a signal, not a verdict" every time, including for high-concern results, where it's most tempting to let the UI read as a confident diagnosis.
- **Dark mode isn't just inverted colors** — check a real screen in both modes for contrast and that the teal/gold/coral concern-level colors stay distinguishable from each other in dark mode too.
