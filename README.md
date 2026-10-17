# Automated SMB Site & Outreach Pipeline — Architecture Writeup

**What it is:** An automated pipeline that finds local businesses with no website, builds one, and sells it.
**Why it existed:** To turn a cold-outreach small-business sales motion into a fully automated, self-funding acquisition pipeline.
**What I owned:** Full pipeline design, international localization, and the seven real production bugs documented below.
**Stack:** Python, Airtable, PostgreSQL, n8n, Stripe, domain-registrar APIs.
**Production status:** Live, multi-region. This is a writeup, not a source dump — the real app has live payment credentials and prospect data running through it continuously.

---

An automated small-business acquisition pipeline: discover local
businesses with no website, auto-build one for them, run outreach selling
it, and upsell an AI phone receptionist on top. Live, multi-region
(US/Canada/UK/Australia/New Zealand plus an "interested, no local payment
rail yet" mode for a dozen more countries), fully self-hosted.

This is an architecture writeup, not a source dump — the real
implementation has live payment keys and real prospect data running
through it continuously, so what's useful to show publicly is the design
and the real engineering problems solved, not the literal files.

## Pipeline

A daily cron sequence, each stage independent and individually
observable:

1. **Discover** — searches a rotating pool of (city × business category)
   combinations for businesses with no real website, filtering out
   results that merely have a social-media profile masquerading as one.
2. **Generate** — builds a real preview site for each new prospect, checks
   domain availability under a hard per-domain budget cap, and queues the
   first touches of an outreach sequence.
3. **Expire** — lets unclaimed previews age out on a fixed window.
4. **Report** — posts a daily funnel summary, broken down by country.

Prospect, outreach, and payment state live in a structured database
(fields for lifecycle status, discovery date, chosen domain, touch
history); the generated sites and the outreach copy that sells them are
produced from the same pipeline run, not maintained separately.

## Real engineering problems solved

**A locale-pricing bug that had been live for weeks, affecting hundreds of
real prospects.** Early locale detection mapped a wide swath of
Commonwealth and international markets (India, Pakistan, Bangladesh, most
of Africa, the Caribbean) to "UK" for pricing and copy purposes, purely
because they use British-style spelling conventions — meaning real
prospects in those markets were being quoted in GBP and told their site
served "your local area in the UK." Fixed by separating **dialect**
(spelling conventions — genuinely shared across Commonwealth English) from
**locale** (currency, legal text, payment rail — genuinely not shared) as
two independent axes instead of one conflated setting. 913 already-
generated offer pages were migrated to the corrected locale, safely
re-runnable, with preview pages and outreach history left untouched.

**An "interest mode" for markets with no local payment rail yet**, rather
than either blocking those markets entirely or silently charging them in
a currency/method that doesn't actually work for them: offer pages for
those locales show no price, no checkout — just an invitation to reply
and say how they'd prefer to pay (bank transfer, mobile money, etc.). The
page template itself is built from three independently-fillable blocks
(the pricing/CTA section, the checkout section, the terms section) so the
two modes render from the same template rather than maintaining two
separate page designs that could drift apart.

**A background process that looked crashed but wasn't.** The outreach
relay process, launched via the OS's service manager without explicit
unbuffered-output configuration, had its log output silently buffer in
memory for long stretches even while genuinely sending — indistinguishable
from a crash loop by anyone reading the log. Fixed with an explicit
unbuffered-output environment flag; confirmed via two real test sends
producing immediate real-time log output afterward (and worth knowing
generally: an environment-variable change to a long-running service's
config doesn't take effect on a simple reload — it needs a full
stop/start cycle).

**A monitoring gap that reported "all healthy" during a multi-week silent
stall.** The pipeline's own health check only ever verified queue
*fullness* (is the backlog over some cap), never *drainage* (is anything
actually being sent) — so it reported clean the entire time outreach
had, for unrelated reasons, stopped sending entirely. Fixed by adding a
second, independent check: if there's a real backlog due *and* nothing
has gone out in the last 48 hours, that's alertable on its own, regardless
of what the fullness check says.

**A silent Slack-delivery failure that looked identical to "no alerts
because everything's fine."** A private alert channel's bot had lost
membership in it at some point during an unrelated cleanup pass — and a
bot calling the channel-list API for a channel it's no longer a member of
gets back a result indistinguishable from "that channel doesn't exist."
This read as clean monitoring for a real stretch of time before being
caught by cross-referencing against what a human could actually see in
the Slack client. **General lesson, not specific to this one incident:
verify a bot's actual channel membership directly, don't infer it from
the absence of errors** — a channel a bot was removed from fails exactly
like a channel that was never created.

**A real fraud-prevention gate that masqueraded as an API bug for weeks.**
Automated domain registration kept failing with a generic error, which
triggered a long chase through reverse-proxy routing, User-Agent headers,
and possible IP-based blocking — none of which was the actual cause. The
real explanation had been sitting unread in a support reply for weeks: the
registrar requires exactly one domain to be purchased manually through
their own website before API-based registration unlocks at all, a
deliberate "ensure a human is actually behind this account" gate. Once
found, the fix was trivial; the lesson generalizes — when an API fails
in a way that resists every normal debugging angle, check for an
account-level gate before assuming the integration itself is broken.

**A 32x jump in usable contact data from one targeted scraping change.**
The core data source (a business-listing API) has no email field at all —
a hard limitation, not a bug — so outreach could only ever reach
businesses that happened to list a social-media profile as their "website."
The first attempt to extract an email from those profile pages used a
plain HTTP fetch, which barely worked (~1.1% hit rate) against modern,
JavaScript-rendered social profile pages. Replacing it with a real
headless-browser render of the same pages lifted the hit rate to **34.9%**
on a full production run — real contact data for hundreds of prospects
that were previously unreachable by this channel at all, with zero change
to the underlying data source.

## Why self-hosted rather than off-the-shelf

Every piece — site generation, payment processing, domain provisioning,
outreach sequencing — is wired together from primitives (a structured
database, a workflow engine, standard payment/registrar/DNS APIs) rather
than built on a single vertical SaaS platform, because the actual product
spans several things no single platform does all of: automated site
generation *and* automated domain purchase *and* locale-aware billing
*and* a long-running, state-tracked outreach sequence per prospect. The
tradeoff is real integration work instead of a subscription; the payoff is
a product shape no off-the-shelf tool was built to support.
