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
