---
name: propose
description: Show the user an interactive review page instead of a wall of text whenever they need to weigh in — a design proposal, a set of decisions/options, or an architecture to sign off on. Builds a page where each decision is a card they react to (👍 love / 🤔 tweak / 👎 nope + notes), with a "copy my feedback" button, then replies in chat with a SHORT blurb + the link. Two modes — "proposal" (quick, a few decisions) and "blueprint" (architecture sign-off, bigger, with an overall approve/rework gate). Trigger whenever you'd otherwise dump a design/spec/plan/options in chat for the user to approve.
---

# Propose — interactive review docs

Some users don't want walls of text in chat when they have to make a call. Instead:
show them a **visual, sectioned page on your own share domain** where each decision is a
card they tap a reaction on, then send a **short blurb + the link** in chat. They
react on the page, hit **Copy my feedback**, and paste it back to you.

This assumes you already have a small git-backed "publish a page to my domain" flow (see
the `share` command in this repo for a template) and a page-rendering script. Adapt the
publish step below to your own setup.

## When to use
- Presenting a **spec / design proposal** → mode `proposal`.
- Asking the user to choose between **options / approaches** → mode `proposal`.
- Getting **sign-off on an architecture / data model / big plan** → mode `blueprint`
  (bigger framing + an overall Approve / Approve-with-changes / Rework gate).

If you catch yourself about to write more than ~2 short paragraphs of design/options
in chat, stop and use this instead.

## How

1. **Compose the sections.** Each decision = one card: a short punchy proposal, a
   mockup/table where it helps, kept scannable. Lead with the single most important
   thing (the "why this matters"). Mark your own *suggested* additions with
   `"suggest": true` so the user knows they're your idea, not a requirement.

2. **Write a spec JSON** to your scratchpad:
   ```json
   {
     "title": "Finance Hub — v1 Spec",
     "subtitle": "Your call on each piece — react to anything off.",
     "project": "finance",
     "mode": "proposal",
     "blurb": "17 decisions — tap reactions, copy feedback back to me",
     "sections": [
       {"tag": "The core", "title": "Per-category budgets",
        "html": "<p>You set a budget per category…</p><div class='mock'>…</div>",
        "suggest": false}
     ]
   }
   ```
   `html` is arbitrary HTML for the card body. Useful mockup classes to bake into your
   template: `.mock` (monospace mockup box), `.hero.over`/`.hero.under` (big signed
   number), `.tabbar`+`<div class="on">` (tab bar), `.bar`+`<i style="width:%">`
   (progress bar), `table.tradeoff` (options/trade-off table — great for `blueprint`
   mode), `.rev`/`.grn`/`.amb` (red/green/amber inline text), `.k` (blue keyword),
   `.subline` (muted caption).

3. **Publish** via your own render-and-deploy script (writes the page, upserts a
   manifest as **unlisted**, commits, pushes, deploys). It should print the URL.

4. **Reply in chat, SHORT.** One or two lines of context + the link. Do NOT restate
   the whole design in chat — the page is the design.

## Modes
- **proposal** (default): quick. A handful of decision cards. No sign-off gate.
- **blueprint**: architecture/big-design sign-off. Same card mechanic PLUS an
  **overall verdict** gate at the bottom (Approve / Approve-with-changes / Rework).
  Use richer cards — trade-off tables, phased plans, the data model.

## After feedback comes back
Fold it in. For a `blueprint`, an "Approve" verdict is your green light to move to
an implementation plan. For a `proposal`, revise and either re-publish or proceed.

## Notes
- Verify the page is live after publishing (`curl -s -o /dev/null -w "%{http_code}"`
  the URL → expect 200).
- Keep card copy tight — this whole skill exists because walls of text don't get read.
