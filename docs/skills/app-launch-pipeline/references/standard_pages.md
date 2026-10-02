# Standard pages: every app in this pipeline ships these

Every Daybreak Labs / Loomwork app gets the same set of public trust and support pages, modelled on TrueTrack and Twine. They exist for three reasons:
- Google Play and App Store review need them: a privacy policy URL, an account-deletion URL and a support contact.
- Users decide whether to trust a new app partly from them.
- They're cheap SEO surface.

Write them in the app's own voice and visual language, never as generic boilerplate.

## Rules for all of them

- **Public.** No login is required, so store reviewers and search engines can reach them.
- **Linked** from the landing/marketing footer *and* from inside the app (Settings or an "About & help" section). Every page has a back link.
- **Own SEO metadata.** Each page gets its own title and meta description through the app's SEO helper (e.g. `useSeoMeta({ title, description, path })`). The FAQ also gets FAQPage JSON-LD.
- **Same design tokens** as the rest of the app (palette, type pairing, card style). They must not look like a bolted-on template.
- **No made-up facts.** Don't invent company history, team members, addresses or certifications. Use placeholders the owner fills in, and list each one in the hand-off.

## The pages

1. **About (`/about`)**
   - Eyebrow "About Us", then a headline stating the app's stance in one line.
   - Two or three paragraphs: the frustration that started it, what the app does differently, and the business model in plain words (free tier, paid tier, no data selling).
   - A "What we stand for" grid of four value cards, each tied to a real product decision, not a slogan.
   - An end-of-page call to action into the app's real entry flow.
2. **Contact (`/contact`)**
   - Eyebrow "Contact Us", then "We'd love to hear from you".
   - Optional: an instant AI support chat panel (a Base44 agent) with 3–4 suggested questions about the app's real features, billing and data.
   - A name/email/message form sent through `Core.SendEmail` to the app's support inbox, with a success state and an error fallback.
   - A "we reply within 48 hours" promise. The owner must confirm the support address; never ship a guessed address.
3. **Guide (`/guide`)**
   - How to use the app: getting started in 3–5 steps, then one section per core feature (what it does, how to use it, what the result means).
   - Then tips and limits, with honest caveats about what the app can't do.
   - For safety, health or finance apps, add a section on what to do next or where to get real help, with links to authoritative bodies.
4. **FAQ (`/faq`)**
   - 8–14 expandable questions: accuracy and limits, free vs paid, whether we sell data, cancelling, export and delete, platforms, payments.
   - A "Still have questions? Contact us" link at the end.
   - FAQPage JSON-LD.
5. **Privacy Policy (`/privacy`)**
   - "Last updated" date.
   - Sections: Your data, your control; What we collect; What stays on your device vs what's sent to our servers (and to which processors, e.g. an AI or detection API); How we use it; We never sell your data; Retention; Data deletion (in-app account deletion, plus the /delete-account page); Children; Contact.
   - It must match what the code actually does. Re-check it after any feature that changes data flows.
6. **Terms of Service (`/terms`)**
   - Acceptable use, subscription and trial terms, cancellation and refunds through the store, disclaimers (e.g. "not medical, financial or legal advice", or for scam tools "a signal, not a verdict"), limitation of liability, contact.
7. **Delete account (`/delete-account`)**
   - A public explanation of how to delete your account in the app, plus a request form for people who can't sign in.
   - Google Play requires this URL in the Data safety section.

## In the Base44 prompt

Add a "Standard pages" section to every Stage 3 prompt that lists these seven routes, says they're public, and points to this file's rules. Also add footer and Settings links to them.

## Low-value-content defense (required for every app)

Google flags thin, tool-only or login-only surfaces as "low value content". This hits AdSense and AdMob approval, Play review and search ranking alike. It's judged **screen by screen**, so a login wall in front of everything is the worst case. Every app ships these too:

8. **A public landing page at `/`** for logged-out visitors and crawlers; signed-in users go straight to the app.
   - A hero with a plain one-line value proposition, then a "How it works" section (one block per core feature, with a real explanation).
   - A "Why this matters" section with cited, real statistics from authoritative sources: link the source and never round up or invent numbers.
   - A free vs paid summary, a short FAQ teaser, trust signals (privacy stance, "who built this"), and a call to action into sign-up.
   - At least 800 words of original copy in total.
9. **A resources / guides hub (`/resources`) with 5–8 original articles of 800–1,500 words each**, on the real questions the Stage 2 research surfaced.
   - Each article is written for that topic, not spun from a template.
   - Each one has a byline or "Reviewed by" placeholder for the owner to fill in, a last-updated date, cited sources, and internal links to the relevant app feature and to related articles.
   - Never auto-generate near-duplicate variants (per-city, per-number and similar).
10. **One or two honest comparison pages** (`/compare/[competitor]`) where Stage 2 found a dominant competitor. Be fair about what the competitor does better.
11. **Search and crawl basics:**
    - A unique title and meta description on every public page, plus Open Graph tags.
    - `sitemap.xml` listing every public route, and a `robots.txt` that allows public pages and blocks app-only routes.
    - FAQPage or Article JSON-LD where it applies.
    - A footer with links to every public page.
12. **No thin screens.** Every in-app result screen carries a sentence or two explaining what the result means, alongside the number. Empty, loading, confirmation and error screens never carry ads. If the app ever adds ads, run the full checklist in the `subscription-trial-model` skill ("Ad-supported free tiers") against every ad-bearing screen, and verify `ads.txt` by fetching the live URL.
