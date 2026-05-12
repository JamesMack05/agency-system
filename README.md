# Diana's Team — Five-Specialist Workflow System

**A set of instructions you give to Claude (the AI) so it can run the back-office work of a small real estate team. You — the licensed realtor — review and send everything that goes to a client.**

This is not software. There is nothing to install. The whole system is the folders in this repo, dropped into a Claude Project. You can set it up in five minutes and try it on a real inbound lead in fifteen.

---

## What this actually is

When a lead comes in, or a property research request goes out, or a deal moves into contract, the work usually flows through five stages: someone decides who handles it, someone qualifies the lead, someone does the research, someone drafts the communication, and someone tracks the deal through to closing. On a small team those stages live in someone's head or scattered across three Google Docs.

This system turns each of those five stages into a **role the AI plays**. Not a new piece of software. Not a chatbot. A set of written instructions — five folders, four files each — that tells Claude how to behave at each stage. You hand work to Claude, Claude plays the right role, you review what it produced, you send it.

**To be clear:**

- The "five specialists" are roles the AI plays. They are not five people you need to hire.
- The licensed realtor (you, or whoever holds the license) reviews every client-facing message before it goes out. The AI drafts; you decide.
- Once you set this up in Claude, you talk to it in plain English. You don't need to know what "envelope" or "handoff_reason" mean to use it — those are how the system works under the hood, not what you have to learn.

If you want to know *why* the system is built this way — the design decisions, the trade-offs, the schema — read [`ARCHITECTURE.md`](ARCHITECTURE.md). If you just want to use it, keep reading.

---

## What you need before you start

- **A Claude.ai account.** The free tier works. Paid (Claude Pro) gives you longer conversations and faster responses; either is fine for trying it.
- **The Projects feature inside Claude.** It's in the sidebar at [claude.ai](https://claude.ai/) after you log in. Free accounts get it.
- **About 15 minutes.** Five to set up, ten to try it on a lead.
- **One real or made-up inbound lead** to walk through. There's one in the quickstart below if you don't have one ready.

That's it. No Zapier, no Make.com, no CRM integration. You can layer those in later if you want, but the system runs in plain Claude.

---

## Setup in 5 minutes

1. **Go to [claude.ai](https://claude.ai/)** and sign in. Click **Projects** in the left sidebar, then **Create Project**. Name it `Diana's Team` (or your team's name).

2. **Upload the files.** In the project's **Project knowledge** section, upload these files from this repo:
   - `HANDOFF_SCHEMA.md`
   - Everything inside `00_orchestrator/` (4 files)
   - Everything inside `01_lead_qualifier/` (4 files)
   - Everything inside `02_property_research/` (4 files)
   - Everything inside `03_client_communication/` (4 files)
   - Everything inside `04_transaction_coordinator/` (4 files)

   That's 21 files total. Drag-and-drop works — you can drag whole folders into Project knowledge in one go, or open each folder and drag the 4 files inside individually. Both work.

   **Don't upload `README.md`, `ARCHITECTURE.md`, or the `decisions/` folder.** Those are for you to read, not for Claude. Uploading them just clutters the project's context window.

   **Heads-up on filenames.** The 20 specialist files share four names across the five folders (every folder has its own `identity.md`, `rules.md`, `examples.md`, `handoff.md`). Each file starts with a header that names which specialist it belongs to (`# 00_orchestrator — identity`, `# 01_lead_qualifier — identity`, and so on), so Claude can tell them apart by content. You don't need to rename anything before uploading. If you ever want to sanity-check that Claude has the right file, just ask it: *"Which specialist's `identity.md` are you reading right now?"* — it can tell you.

3. **Set the system prompt.** In the project's **Custom instructions** (sometimes called "Project instructions"), paste the block below.

   This text is for Claude to read, not for you to learn. A few of the words inside it (*envelope*, *handoff_reason*, *back_data_missing* and so on) are how Claude tracks things under the hood. You will rarely see them in your day-to-day chats — Claude will say things like *"opening a case file"* or *"missing: timeline — handing back to the qualifier"* instead. The technical names live in the prompt so Claude behaves correctly; they don't live in your vocabulary.

   ```
   You are an AI operating system for a small real estate team. You play five specialist roles, defined in detail in the Project knowledge files:

   - 00_orchestrator — routes every inbound to the right specialist
   - 01_lead_qualifier — qualifies new prospects (timeline, pre-approval, budget)
   - 02_property_research — pulls comps and neighbourhood briefs
   - 03_client_communication — drafts emails and texts in the realtor's voice. You NEVER send to a client. The realtor reviews and sends.
   - 04_transaction_coordinator — tracks deadlines once a contract is executed

   Each specialist has four files in its folder: identity.md (role), rules.md (what they always and never do), examples.md (worked cases), handoff.md (what they receive and produce). Every time work moves between specialists, it travels in a structured "envelope" — see HANDOFF_SCHEMA.md.

   When I hand you work, start by telling me which specialist is the right starting point and why. Then play that role end-to-end. When that specialist is done, hand off to the next one via an envelope.

   If a specialist needs information they don't have, hand the work BACK with a typed reason (back_data_missing, back_scope_mismatch, back_quality_failure, or back_compliance_block). Don't fake missing data. Tell me what's missing and I'll provide it.

   Speak in plain English to me, the user. Use phrases like "opening a case file" or "handing this back because we're missing the timeline" rather than schema names like "envelope" or "back_data_missing" unless I ask for the technical view. When you refer to the human on the deal, always say "realtor" or "licensed agent" — never just "agent" — because some of the specialist files use "agent" for the human realtor and that can be confusing.

   I am the licensed realtor. I review every client-facing message before it sends.
   ```

4. **Open a new chat in the project.** That's it — you're set up. Try the quickstart below.

If Claude says it can't find a file referenced in the prompts, double-check the upload completed. The system needs all 21 files to function.

---

## Try it: your first lead in 15 minutes

Paste this into your project chat:

> Inbound at 14:32 — Zillow web form on the Tarrytown listing:
>
> *"Hi, saw your Tarrytown listing on Zillow, can someone reach out? — Sarah"*
>
> Walk this through the system. Start at 00_orchestrator.

Claude will respond by:

1. Routing the inbound (opening a fresh "case file" for it; stamping the source as `[UNVERIFIED]` because Zillow web forms don't tell you who's actually typing).
2. Handing off to `01_lead_qualifier` with everything it knows so far.
3. Running through what the qualifier would do — call the listed number, ask the three-question filter (timeline, pre-approval, budget).
4. Asking *you* for the answers, because the qualifier can't make them up.

You answer with something like:

> Sarah picked up. Pre-approved with Frost Bank for $850k. Wants to move in 60 days. Downsizing from Westlake.

Claude will:

5. Update the case file to `[CLIENT]` (identity confirmed during the call).
6. Hand off to `02_property_research`. The research role will tell you what it needs — typically 3-5 recent comparable sales for Tarrytown, plus the neighbourhood facts you'd put in a buyer brief (schools, commute, walkability, anything Sarah has flagged). **You pull this data from your MLS (UnlockMLS for Austin) and paste it into the chat.** Claude doesn't have access to MLS or paid data sources; it structures what you give it into a research brief.
7. Once the research brief is back, hand off to `03_client_communication` to draft a first-touch email in your voice. Claude will produce a draft.

That email lands in front of you. You read it, edit if needed, send it from your own email client (Gmail, Outlook, whatever). **The AI never sends. You always do.**

### One more thing: when the system hands work back

Suppose at step 4 above, you couldn't reach Sarah — voicemail, no callback yet. The qualifier can't qualify her without that call. So instead of pretending it can, it hands the work back: *"missing: timeline, pre-approval, budget — can't qualify until we reach her."*

That back-handoff is the most important thing about this system. It means the next specialist (in this case, the drafter) **never sees an incomplete case file**. If Sarah hadn't been reached, the drafter wouldn't get a half-baked half-personalised email request — they'd just not get the request yet. The realtor sees the hand-back, decides what to do (try again? leave a voicemail? wait?), and re-routes when ready.

That's the whole shape of the system: **work moves forward when it's complete, moves back with a typed note when it isn't.**

---

## Working with the system day-to-day

A few practical questions that come up after the first lead:

- **One chat per case, or all cases in one chat?** One chat per case. When a new lead arrives, open a new chat in the project. The case file (case ID, source stamp, log of who's done what) lives inside that chat. Keeping one case per chat means Claude isn't mixing up Sarah's Tarrytown enquiry with the McKenzie listing in your other tab.
- **Does the case file persist if I close the chat?** No — Claude doesn't have a persistent database. If you close a chat and reopen it later, the case file is still there *in the chat history*. If you start a fresh chat, it's a new case file. For a long-running active deal (post-contract through to close, often 30-45 days), keep the chat alive — or, before closing, ask Claude to *"summarise the current case file so I can paste it into a new chat to resume."* Save that summary alongside your other deal notes.
- **What if the chat hits a length limit?** On the free tier, long deals can run into Claude's per-conversation length cap. When that happens, ask Claude for a case-file summary (as above), open a new chat, paste the summary, and say *"resume this case at the transaction-coordinator stage."* Claude picks up where it left off.
- **What if I want to use ChatGPT or another AI instead of Claude?** You can — the files are plain markdown — but the system is calibrated for Claude. The system prompt above will need adapting (ChatGPT calls Custom Instructions something different and handles file uploads differently). Start with Claude, then port if you have a reason to.

---

## How the system thinks: the deal binder

If it helps, picture an old-school manila binder. One per deal. The case number is on the spine. Inside is every document about that case: the original inbound, the qualifier's notes, the research brief, drafts that went out, the executed contract, the deadline tracker. On the front cover, a sign-in/sign-out log shows who has held the binder, when, and why.

The binder physically moves between teammates. When the research role is done pulling comps for 2412 Hartford Rd, it signs out, drops the binder on the drafter's desk, and the drafter signs in. When the drafter notices the research role forgot to pull school-district data, the drafter doesn't fix it — the drafter walks the binder *back* with a sticky-tab note ("missing: school-district + Casis rating"), and the research role fills it in.

A few stamps on the cover indicate where the inbound came from:

- **[UNVERIFIED]** — anonymous (Zillow relay, web form with no name, voicemail with no callback). You don't know who the person actually is.
- **[CLIENT]** — identity confirmed by call or by linking to an existing case.
- **[TEAM]** — a team member logged this themselves (an inspection report attached, a status update typed in).

These stamps matter because the rules change downstream. A binder stamped **[UNVERIFIED]** can't get a "Hi Sarah" email drafted on it — neutral salutation only — until someone confirms who Sarah actually is.

This binder is conceptual: in practice, it's a structured note Claude maintains in the conversation. You don't see "envelope schema" while you're using the system. You see Claude saying *"I've opened a new case file (CASE-2026-0042) and stamped it [UNVERIFIED] — handing to the qualifier."* That's the same thing.

---

## What the AI does vs what you do

This is the most common confusion. Worth being explicit.

| Situation | The AI does | The realtor does |
|---|---|---|
| Inbound arrives | Routes it, opens a case file, stamps the source | Hands the inbound to the AI; reviews routing if it looks wrong |
| New prospect on the phone | Tells you the three-question filter to run; logs what you tell it | Makes the call; asks the questions |
| Comps for a listing | Pulls public data, structures a brief, gives a price *band* | Sets the actual price; uses the band as input |
| First-touch email | Drafts in your voice based on what's in the case file | Reads, edits, sends |
| Executed contract | Sets up the deadline tracker, flags T-24h items | Acts on the flags; signs documents; talks to the client |
| Anything client-facing | Drafts and proposes | **Always reviews and sends** |
| Compliance question (UPL, TRELA) | Refuses and tells you why (this is intentional) | Decides what to do — usually loops in your broker |

**The AI never makes price opinions. It never sends to a client. It never gives legal or financial advice.** If you ask it to, it will refuse and tell you it's refusing — that's by design. Use it as a fast, structured assistant, not as a replacement for licensed judgement.

---

## Adapting this to your team

The system as shipped is calibrated for a 4-person Austin team doing 60-80 residential transactions a year, working under Texas regulations. To adapt:

- **State regulations.** Open `01_lead_qualifier/rules.md` and `03_client_communication/rules.md`. Anywhere it cites TRELA, TREC, or §1101.563, swap in your state's equivalents. Your broker will know what these are.
- **Data sources.** Open `02_property_research/rules.md`. It currently references `austincad.org` (Travis County appraisal district) and UnlockMLS (Austin MLS). Swap in your local appraisal district and your MLS.
- **Team size.** Nothing in the system assumes four people. A solo agent runs all five roles themselves (you become your own router, qualifier, researcher, drafter, coordinator — Claude just structures the work for you). A 12-person team can have people own specific roles. The roles exist in the AI's instructions regardless of how you carve them up among humans.
- **Transaction volume.** 60-80/year is the calibration data. The system doesn't break at 200/year or at 20/year. It scales by how many parallel cases you keep open at once, not by annual volume.

What you should **not** change without thinking carefully: the `handoff.md` files. Those define the contract between specialists. If you break a handoff contract, the back-handoffs stop working and the system starts producing half-baked work. If you need to change a handoff, read [`ARCHITECTURE.md`](ARCHITECTURE.md) first — it explains *why* each field is there.

---

## When something feels wrong

- **Draft email reads cold or pushy.** Tell Claude. *"Re-draft that email — it sounds like a robot."* The voice calibrates through use; the more you redirect, the closer it gets to yours.
- **Claude refuses to draft a personalised email.** Check the case file stamp. If it's `[UNVERIFIED]`, Claude is doing the right thing — you don't send "Hi Sarah, great to meet you!" to someone whose identity you haven't confirmed. Confirm who they are, mark the case `[CLIENT]`, and ask again.
- **Claude proposes a price.** It won't, if the system is set up right. It will give you comps and a price band. The price is your call.
- **Claude tries to send something to a client directly.** Stop it. Tell it to draft only. The system prompt above is meant to prevent this — if you skipped that step, set it now.
- **Something else.** The five specialists each have a `failure modes` section inside `identity.md`. If a specialist behaves unexpectedly, that's the file to read. Or check `decisions/failure-modes.md` for the full catalogue.

---

## Where to go next

| Who you are | Read |
|---|---|
| New team member, want to use the system | This README, then try the quickstart above. Once it makes sense, open `00_orchestrator/identity.md` + `01_lead_qualifier/identity.md` (the two specialists that touch every new inbound). |
| Realtor wanting to adapt it to your team | "Adapting this to your team" above, then the relevant `rules.md` files. |
| Curious about the design | [`ARCHITECTURE.md`](ARCHITECTURE.md) — same system, full design rationale, real vocabulary. |
| Reviewing the engineering work | [`ARCHITECTURE.md`](ARCHITECTURE.md) + the `decisions/` folder. Three ADRs, a council adversarial review, a worked-trajectory file, an assumption ledger, and an open-questions register. |

---

This system fits in a folder. You can read it in an afternoon. You can teach it to a colleague. That's the design intent.
