# Landing Chat Lead Finalization Plan

## Goal

Landing chat should create useful insurance leads from the full web chat transcript without racing the live conversation.

The final behavior should be:

- Ben uses the full chat context, not a truncated recent-message window.
- Lead creation is not triggered by contact info alone.
- The web chat agent can finalize a lead when the transcript has insurance intent, contact info, and enough useful business context.
- Lead finalization creates or updates the lead from the full stored transcript.
- Lead finalization does not send email or SMS.
- Oracle reviews the finalized transcript before any post-chat follow-up is planned.
- Outbound follow-up is suppressed if the visitor keeps chatting and makes the planned follow-up stale.

## Production Scope

The production rollout should target the active landing surface first:

- `apps/landing` / `landing` Cloud Run is the primary `kinro.com/insurance` landing surface.
- `apps/pivot-landing` / `pivot-landing` Cloud Run should receive the same behavior only if `kinro.insure` remains live or needs to stay aligned as a rollback surface.
- `arrakis-data-api` owns transcript persistence, lead creation/update, finalization, fallback scheduling, and Oracle triggers.
- `arrakis-backoffice` owns Ben's web chat prompt and tool behavior.
- `arrakis-dashboard` should display the extracted lead details and full chat transcript.

This plan is scoped to landing chat intake and lead creation. It does not require connecting the anonymous web chat session directly to the existing lead Ben agent conversation.

## Current Problem

The current flow is too contact-triggered:

1. Visitor chats with Ben.
2. The landing app sends messages to Arrakis Data API.
3. Data API persists transcript events and projects them into inbox/lead state.
4. If a message contains email or phone, the backend can create/link a lead immediately.
5. Generic `createLead()` can start the default new-lead Ben mission and schedule follow-up.

That creates two problems:

- A lead can be created before the visitor finishes giving useful details like name, revenue, years in business, coverage needs, employee count, or state.
- Default Ben outreach can start while the visitor may still be actively chatting.

## Desired Flow

During live web chat:

- Persist every user and Ben message to the landing chat transcript.
- Let Ben continue answering and asking useful in-chat questions.
- Do not create a lead from contact info alone.
- Do not start the default new-lead Ben mission or sequence.
- Do not send outbound email/SMS while the visitor is still chatting or while a newer chat message may make the outreach stale.

When Ben has enough information:

- Ben calls a finalization tool.
- The backend reloads the full stored transcript.
- The backend extracts all useful details from the transcript with an LLM.
- The backend creates or updates the lead only if the transcript contains email or mobile number.
- The backend stores structured fields where possible and extra facts/transcript metadata where needed.
- The backend suppresses default new-lead Ben automation for this source.
- The backend triggers Oracle to review the finalized transcript and decide next steps.

After Oracle review:

- If nothing important is missing, no external follow-up is needed.
- If information is missing, Oracle can create a targeted follow-up mission or sequence.
- Any external email/SMS must pass stale checks before it sends.

## Finalization Tool

Tool name direction: `finalize_landing_chat_lead`.

The tool should be available to the web chat agent. It is the primary lead-finalization signal, instead of waiting for a long idle timeout.

Ben should call the tool only when all of these are true:

- The transcript contains email or mobile number.
- The visitor has clear insurance intent, such as asking for a quote, coverage guidance, certificate help, policy review, or insurance recommendations.
- The transcript contains enough business context to create a useful lead, such as business type plus at least one or two meaningful details like state, coverage interest, staffing, revenue, years in business, operations, or contractor use.

The tool should not be called for contact info alone.

Tool input should include:

- `agentRunId` or session identifier.
- Latest user message sequence used by Ben when deciding the lead is ready.
- A short readiness summary.
- Optional Ben rationale.

Server-side behavior:

- Re-read the full stored transcript.
- Confirm email/mobile exists.
- Confirm the tool call is not stale compared with the latest stored user message sequence.
- Run LLM extraction over the full transcript.
- Create or update one lead idempotently for the session/contact.
- Attach transcript turns, QA pairs, extracted fields, and additional facts to lead metadata.
- Add the full chat transcript to the lead timeline.
- Suppress default new-lead Ben mission/sequence for this lead source.
- Trigger Oracle for post-chat review.

The tool must not send email/SMS.

## Stale And Still-Chatting Safety

The system should treat these as separate actions:

- Live web chat replies.
- Lead finalization.
- External email/SMS follow-up.

Lead finalization can happen before we have a perfect "chat ended" signal. External follow-up cannot.

Use transcript sequence numbers to detect stale actions:

- Every user message in `agent_transcript_events` has a sequence number.
- Tool calls include the latest user sequence Ben saw.
- Oracle follow-up plans and scheduled Ben email/SMS steps store the latest user sequence they were planned against.
- Before sending external email/SMS, the sender checks whether a newer landing-chat user message exists.
- If a newer user message exists, the outbound send is stale. Suppress it and trigger/requeue Oracle review with the updated transcript.

Example:

1. Visitor gives contact info and enough business context.
2. Ben calls `finalize_landing_chat_lead`.
3. The backend creates/updates the lead and triggers Oracle.
4. The visitor waits five minutes and sends another web chat message.
5. The new message has a higher transcript sequence.
6. Any pending outbound email/SMS planned from the older sequence is stale and must not send.
7. Ben continues the live web chat, and Oracle can re-review the newer transcript if follow-up is still needed.

## Idle Fallback

Idle detection should be a fallback, not the main completion strategy.

Recommended fallback behavior:

- Every new user message schedules or reschedules a short fallback finalization job.
- The fallback job only runs if Ben has not already finalized with the tool.
- The fallback job uses the same server-side finalization path and readiness requirements as the tool.
- If a newer user message arrived after the fallback was scheduled, the fallback exits as stale.
- The timeout should be short and configurable, for example 1-2 minutes, only to handle abandoned chats.

## Oracle And Default Ben Mission

Landing-chat leads should not receive the generic default new-lead Ben mission/sequence at creation time.

Reason:

- The web chat is already the intake interaction.
- Generic follow-up can duplicate questions already answered in chat.
- Oracle should review the full finalized transcript before deciding whether follow-up is needed.
- Holding default outreach prevents contact-info-only lead creation from sending messages while the visitor may still be chatting.

Expected implementation direction:

- Skip default mission/sequence creation when the lead source is landing chat, for example `externalSource === "pivot-landing-chat"` or the equivalent active landing source.
- Mark the lead metadata so automatic lead engagement is suppressed for the default path.
- Let Oracle create any follow-up mission after transcript finalization.
- Require stale/still-chatting checks before any Oracle-planned outbound send.

## Data Extraction

Final extraction should use the full transcript and an LLM. Do not rely on regex-only extraction for lead details.

At minimum, extraction should capture:

- Name.
- Email and/or phone.
- Company name.
- State/location.
- Business type / industry.
- Coverage requested.
- Years in business.
- Annual revenue.
- Employee count, contractor count, or notes when staffing is described differently.
- Any other useful facts stated in chat, even without a first-class lead column.

Canonical lead fields should be populated where available. Extra facts, transcript turns, and QA pairs should be stored in metadata.

## Dashboard And Timeline

The lead timeline should expose the full chat transcript on the landing-page lead creation/finalization event so operators can review the actual conversation.

The dashboard lead details should show the extracted structured fields and make additional collected facts visible through metadata-backed sections where needed.

## Acceptance Criteria

- Active production landing chat uses full chat context.
- Contact info alone does not create a lead.
- Ben can call `finalize_landing_chat_lead` after insurance intent, contact info, and useful business context exist.
- Finalization creates or updates one lead from the full transcript only when email or mobile exists.
- Finalization does not send email/SMS.
- Default new-lead Ben mission/sequence is skipped for landing-chat-created leads.
- Oracle receives finalized transcript-derived context before planning follow-up.
- External follow-up sends only after Oracle approval and stale/still-chatting checks.
- If the visitor sends a newer web chat message, older planned outbound follow-up is suppressed and re-reviewed.
- The lead timeline includes the full chat transcript.
- The dashboard shows extracted lead details.
