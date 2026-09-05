# AI WhatsApp Automation for a Cross-Border Logistics Company
**`n8n` · `PostgreSQL` · `LLM API` · `WhatsApp`**
 
> A conversational WhatsApp agent that handles shipment tracking, shipping quotes, and partner registration through natural conversation — then automatically scores every conversation for buying intent and hands the sales team a ranked daily follow-up list.
 
---
 
## The problem
The company paid for ads that drove customers to WhatsApp, but the WhatsApp line was a manual bottleneck: every enquiry needed a human, so responses were slow and inconsistent, and repeat questions (rates, transit times, "where's my order") ate staff time. Worse, paid leads leaked — interested customers who didn't complete an enquiry vanished with no record and no source attribution, so the business was flying blind on its most expensive acquisition channel.
 
## What I built
I turned the WhatsApp number into an always-on agent *and* an instrumented top-of-funnel. It answers customers instantly, captures structured data from unstructured chat, and makes the paid-lead pipeline measurable end to end — from ad click to conversation to registration to first shipment.
 
## Architecture
Core principle — **"AI interprets, the orchestrator decides and records."** The LLM understands and phrases; the orchestrator owns all state, validation gates, and side effects. That boundary is the spine of the design.
 
```
WhatsApp (customer)
   │  inbound webhook
   ▼
n8n workflow  (single webhook, returns 200 immediately)
   parse → outbound-echo check → dedup → session read
        → decision node (track / quote / partner / faq / human)
        → ONE LLM call: intent + answer + field extraction
        → validation gates (plausibility, completeness, confirm)
        → prepare reply → record outbound → send → session write
   │                 │                    │
   ▼                 ▼                    ▼
Pricing API      Tracking API        Google Sheets
(deterministic)  (order status)      (lead lists)
 
Scheduled companion workflows:
  • abandoned-enquiry sweeper   • rule-based intent scoring (ranked daily list)
  • weekly funnel digest        • session-health heartbeat
```
 
Data lives in PostgreSQL (JSONB session context, CTE-based upserts): per-user flow state, webhook dedup, event/funnel counters, outbound-message hashes, human-pause flags, and paid-ad attribution. One LLM call per turn returns an answer, an intent classification with confidence, and structured field extraction against a requested schema. Lead scoring is deliberately rule-based, so every score is explainable and tunable.
 
## Engineering decisions worth discussing
- **Treated the LLM as untrusted.** A permanent per-user model session periodically bled one conversation's data into another. I added a plausibility gate — an extracted field is accepted only if the user's actual message could plausibly contain it, while still allowing legitimate derived values (dimensions → volume, city → country). Treating the model as a useful-but-untrusted component rather than a source of truth is the decision I'm most proud of.
- **Human-takeover detection with no API support for it.** The gateway gave no "sent by a human" signal, so I hash every message the bot sends; when an outbound message returns through the webhook with no matching hash, a human typed it — pause the bot for that customer. Fails open, with an allowlist so automated sends don't trip it.
- **Idempotent webhook handling.** The gateway retries on any non-2xx and can redeliver, so the workflow returns 200 instantly and does the slow work after, de-duplicating on the stable message ID (with a content-hash fallback added after a redelivery got answered twice 31 minutes apart).
- **Fail-open, fail-towards-silence as an invariant.** Every guard degrades to a safe default — never an invented price, a delivery promise we don't honour, or a bot permanently silent on a live customer. In a system talking to paying customers, the failure *mode* mattered more than the failure *rate*.
- **Allowlist input parsing over blocklist.** WhatsApp delivers group/privacy/newsletter identifiers that survive a naive digits-only scrub and become convincing fake phone numbers. An allowlist processes only real individual senders — a blocklist is always one format behind.
## Impact
I won't invent before/after metrics I don't have. At a point in time the system held ~1,390 stored conversations; a lead-scoring run independently ranked the exact customer the business owner had manually flagged as highest-value — a strong qualitative signal the model reflects reality. The headline framing is true today: **I instrumented a previously-invisible paid-lead funnel end to end** (ad click → conversation → registration → first shipment), making acquisition measurable for the first time. Metrics I'd query before publishing: automation rate (% handled without a human), median reply latency vs. the old manual baseline, recovered-lead count from the abandoned-enquiry sweeper, and full-funnel conversion.
 
## Tech stack
`n8n (self-hosted, Kubernetes)` · `PostgreSQL (JSONB, CTE upserts)` · `LLM intent/extraction API + RAG` · `deterministic pricing REST API` · `order-tracking REST API` · `Google Sheets` · `team-alert webhooks` · `WhatsApp gateway` · `JavaScript` · `SQL`
 
## Retrospective — what I'd do differently
- **Use the official WhatsApp Business API, not a linked-device gateway.** The gateway drove half the hard problems — redeliveries, no human-sent signal, no templates, ban risk on outreach. The Business API removes that whole class of workaround.
- **Scope LLM sessions to a conversation, not permanently per user.** That single upstream choice caused the memory-bleed I had to build a plausibility gate to defend against. I patched the symptom well; the root cause wasn't mine to control.
- **Break the monolith into sub-workflows.** One large workflow means one bad import risks every flow. I'd extract reusable pieces (message send, session I/O) — I'd spec'd this but it never rose above live firefighting.
- **Observability from day one.** I bolted on a heartbeat and weekly digest late; monitoring should be a first-class feature, not an afterthought.
**What I learned:** the hard part of production AI automation isn't the AI — it's everything around it. Idempotency, treating the model as untrusted, choosing your failure mode deliberately, and distrusting inputs (a "phone number" often isn't one). Most incidents were bad *data*, not bad *logic*, and the highest-leverage habit was simulating each change against real payloads before shipping.
