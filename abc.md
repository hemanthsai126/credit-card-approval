# Plan: Arrakis Coterie Portal Agent

Date: 2026-06-12
Status: Proposed
Builds on:

- `apps/plans/arrakis-quoting-source-subagents-plan.md`
- `apps/arrakis/backoffice/agents/coterie_portal_agent/`
- `apps/arrakis/backoffice/agents/quote_sources/`
- `apps/arrakis/data_api/src/quote_sources/coterie_portal/otp.ts`
- `apps/arrakis/backoffice/agents/quoting_agent/data/coterie-quote-flow-questions.md`

## Objective

Build the Coterie browser-portal source agent at
`apps/arrakis/backoffice/agents/coterie_portal_agent`.

Desired end state:

- The quote-source orchestrator can route Coterie work to a real
  `browser_portal` subagent instead of the current `needs_build` stub.
- The subagent uses Browserbase for portal automation and keeps the Browserbase
  integration modular so future carrier portal agents can reuse the same
  session, login, OTP, artifact, selector, and timeout patterns.
- The subagent opens Coterie at
  `https://dashboard-v2.coterieinsurance.com/quotes`.
- If the Browserbase session is not authenticated, the subagent logs in with
  Coterie credentials from secrets and completes OTP using the existing carrier
  OTP webhook/polling path.
- For new quote work, the subagent navigates through the Coterie quote flow and
  stops at quote review. It never clicks `Continue to pay`, never binds, and
  never enters payment information.
- For existing quote work, the subagent can select a quote from the quotes list
  or load `/quote/edit/{portalQuoteId}`, open the editable accordion sections,
  apply allowed updates, and re-extract quote state.
- Session state, checkpoints, replay URLs, quote artifacts, blocking facts, and
  operator-handoff reasons are visible in the existing quote-source dashboard
  card/modal.

Out of scope for V0:

- Binding, payment, issuing policies, saving payment methods, or crossing the
  Coterie `Continue to pay` boundary.
- Solving CAPTCHA or any browser security challenge.
- Long-term Browserbase context reuse across many sessions. V0 can log in fresh
  and rely on OTP. Add persistent Browserbase contexts later if OTP volume
  becomes noisy.
- Full coverage of every Coterie industry-specific questionnaire branch. V0
  should cover the common GL/BOP/PL flow captured in the existing docs and hand
  off when the portal asks an unknown question.
- Portal automation for other carriers. The shared Browserbase modules should be
  designed for reuse, but only Coterie is implemented in this plan.

## Current Starting Point

Already present:

- `coterie_portal_agent` exists as a quote-source subagent stub:
  - `manifest.ts` declares `sourceKey = "coterie_portal"`,
    `kind = "browser_portal"`, and a portal-shaped state schema.
  - `runtime.ts` records updates, then returns `status: "needs_build"`.
  - `bundle.local.json` exists.
- `login-smoke.ts` already proves the key primitives:
  - create a Browserbase session,
  - connect Playwright over CDP,
  - create a Coterie OTP request in the Arrakis Data API,
  - poll for OTP,
  - fill the OTP,
  - consume the OTP request.
- Data API already has Coterie OTP support:
  - `POST /v1/internal/quote-sources/coterie-portal/otp-requests`
  - `GET /v1/internal/quote-sources/coterie-portal/otp-requests/:id?includeOtp=true`
  - `POST /v1/internal/quote-sources/coterie-portal/otp-requests/:id/consume`
  - inbound email projection into `arrakis.carrier_login_otps`.
- Quote-source sessions and updates already exist, including pause/resume/retry
  and dashboard rendering.
- Prior quote-flow capture already documents most Coterie questions in
  `coterie-quote-flow-questions.md`.

Live portal observations from the logged-in Chrome session on 2026-06-12:

- The quotes URL lands at `/quotes` when authenticated.
- The quotes list includes search, user filter, quote type filter, archive
  selection, row checkboxes, `Edit Quote ...`, and `See Quote Details ...`.
- `New Quote` navigates to `/quote` and starts with:
  - heading `What is the business name?`
  - `Legal Business Name`
  - optional DBA checkbox
  - `Next Step: Business Address`
- Existing quotes open at `/quote/edit/{portalQuoteId}`.
- The edit page exposes:
  - total due,
  - yearly/monthly payment toggle,
  - `Download Quote`,
  - `Quote Snapshot`,
  - policy effective date,
  - `Bind as`,
  - policy selection/detail controls,
  - `Additional Insureds`, `Business Details`, and `Contact Details`
    accordions,
  - `Continue to pay` as the bind/payment boundary.
- `Business Details` includes legal name, DBA, disabled industry/address,
  estimated revenue, annual payroll, employees, business start date, and
  premises square footage.
- `Contact Details` includes first name, last name, phone, policyholder email,
  mailing address, disabled city/state/zip, and `Update Address Manually`.
- `Additional Insureds` starts with an `Individual Additional Insured` checkbox.

## Architecture

