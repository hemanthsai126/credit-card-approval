# Kinro Marketing Reply Triage and Arrakis Promotion Plan

Prepared: June 11, 2026  
Audience: Internal technical team  
Decision: SendGrid-first campaign sending, triage before Arrakis, Oracle-qualified promotion, Ben same-thread execution

## 1. Executive Decision

Kinro should send marketing campaigns from a separate campaign identity, not from `ben@kinro.com`. Customers can reply directly to the campaign email if they want an insurance quote comparison or have questions. Those replies should first enter a campaign triage layer. Oracle classifies the reply, and only serious insurance enquiries are promoted into the Arrakis product surface.

The important boundary is:

> Non-serious campaign replies may be processed by backend triage for classification, compliance, suppression, and sender-health metrics, but they must not appear in the Arrakis dashboard, Ben workflow, or lead/conversation surface.

The recommended v1 stack is:

- **SendGrid** for marketing campaign sending, unsubscribe handling, event webhooks, and campaign-domain authentication.
- **Campaign triage layer** for reply ingestion, raw event handling, suppression, and Oracle classification.
- **Oracle** as the Arrakis brain that understands context, qualifies the reply, and prepares Ben's mission.
- **Arrakis** only for serious or manually approved potential leads.
- **Ben** as the execution identity, entering the same customer thread from `ben@kinro.com` only after Oracle qualifies the reply.
- **Smartlead** only as a fallback sender layer if SendGrid cannot safely support the required cold-outbound sender scale, warmup, inbox rotation, or subdomain routing.

## 2. Target Customer Flow

The customer experience should feel simple:

1. Kinro sends a compliant campaign email from a campaign address such as `quotes@updates.kinro.com`.
2. The email says something like: "Reply if you want an insurance quote comparison or have questions."
3. The customer replies directly to that email.
4. If the reply is serious, Ben joins the same email thread with context.
5. The customer does not see the internal filtering, Oracle classification, or Arrakis promotion steps.

Internally, the flow has a hard qualification gate before Arrakis:

```mermaid
flowchart TD
  A["SendGrid campaign send<br/>quotes@updates.kinro.com"] --> B["Customer reply"]
  B --> C["Campaign reply address<br/>reply+token@replies.kinro.com"]
  C --> D["Triage endpoint/store"]
  D --> E["Oracle classification"]
  E --> F{"Serious insurance enquiry?"}
  F -- "Yes" --> G["Promote to Arrakis<br/>lead + conversation"]
  G --> H["Oracle prepares Ben mission"]
  H --> I["Ben replies from<br/>ben@kinro.com"]
  I --> J["Same customer thread continues"]
  F -- "No" --> K["Suppress / log / metrics only<br/>no Arrakis dashboard item"]
```

## 3. Sending and Domain Plan

Use a campaign identity that is separate from the primary Ben identity.

Recommended sender pattern:

- Campaign sender: `quotes@updates.kinro.com`
- Reply routing: `reply+<campaign_recipient_token>@replies.kinro.com`
- Ben execution sender: `ben@kinro.com`

Do not send marketing blasts or broad cold campaigns from `ben@kinro.com`. Ben's mailbox and reputation should remain protected for qualified conversations and real follow-up.

SendGrid-first setup:

- Create one campaign subdomain first, such as `updates.kinro.com`.
- Configure SendGrid domain authentication for the campaign subdomain.
- Configure SPF, DKIM, DMARC monitoring, and custom return-path where available.
- Use SendGrid unsubscribe groups and one-click unsubscribe headers for campaign mail.
- Use SendGrid event webhooks for delivery, bounce, dropped, unsubscribe, spam report, and other campaign events.
- Use SendGrid Inbound Parse or mailbox forwarding for reply ingestion into the triage layer.

When SendGrid may not be enough:

- Kinro needs many separate outbound inboxes.
- Kinro needs cold-specific sender warmup.
- Kinro needs inbox rotation and sender-level daily caps.
- Kinro needs stronger outbound deliverability monitoring than SendGrid Marketing Campaigns provides.

If those become blockers, add Smartlead as the outbound sender layer. Smartlead should not become the lead system of record. Serious replies still promote into Arrakis, and Oracle still prepares Ben's mission.

## 4. Triage Boundary Before Arrakis

The campaign reply stream should not directly create Arrakis dashboard items. It should first go through a triage layer.

The triage layer receives:

- Raw reply body.
- Sender email.
- Recipient/campaign token.
- Original campaign metadata.
- Message headers, including `Message-ID`, `References`, and `In-Reply-To`.
- Provider event metadata from SendGrid or any fallback sender.

Oracle classifies each reply into one of these outcomes:

- **Serious enquiry:** customer asks about insurance quote comparison, coverage, price, renewal, carrier options, policy review, business details, or buying timeline.
- **Maybe serious:** potentially relevant but not enough information; requires manual review before promotion.
- **Unsubscribe / stop:** suppress globally and sync back to SendGrid immediately.
- **Complaint:** suppress, flag campaign health, and consider pausing sender/domain.
- **Off-topic:** log for metrics only; do not promote to Arrakis.
- **Out-of-office / automated reply:** log only; do not promote to Arrakis.
- **Bounce / delivery issue:** update suppression and sender-health metrics.

Promotion rule:

- Promote to Arrakis only when `classification = serious_enquiry`.
- Promote `maybe_serious` only after human approval.
- Everything else stays out of the Arrakis dashboard, Ben workflow, and lead/conversation surface.

## 5. Arrakis Promotion Behavior

When Oracle qualifies a reply as serious, the triage layer should create or update the Arrakis lead and conversation.

Arrakis should receive:

- Campaign source and campaign ID.
- Recipient and company identity.
- Original campaign email.
- Customer reply.
- Oracle classification and confidence.
- Oracle summary of customer intent.
- Recommended next question or action.
- Ben mission.
- Threading headers needed for same-thread continuity.

Oracle's mission for Ben should include:

- What the customer asked.
- Why the reply is serious.
- What insurance need is likely present.
- What Ben should ask next.
- Any risk flags, such as unclear business type, missing location, or urgent renewal timing.
- A suggested response draft, if appropriate.

Ben should only see the promoted item after Oracle has created the mission. Ben then replies from `ben@kinro.com` in the same thread.

## 6. Same-Thread Ben Handoff

The sender changes from the campaign identity to Ben:

- Initial campaign email: `quotes@updates.kinro.com`
- Ben follow-up: `ben@kinro.com`

That is acceptable as long as the handoff feels natural and the email thread remains continuous. Ben can open with something simple, such as:

> Thanks for replying. I can help you compare options.

The technical requirement is preserving thread continuity:

- Store the original campaign `Message-ID`.
- Store the customer's reply `Message-ID`.
- Preserve `References`.
- Preserve `In-Reply-To`.
- Preserve the campaign recipient token.
- Route future replies back into Arrakis conversation handling after Ben joins.

If the mail client still displays the sender switch, that is okay. The key is that the customer does not need to start a new conversation and Ben has all context.

## 7. Infrastructure Objects

The implementation should keep campaign triage separate from the Arrakis product surface.

Minimum backend objects:

- `campaigns`: campaign name, provider, audience, copy version, status, owner, and CTA.
- `campaign_senders`: sender address, domain/subdomain, provider, authentication state, status, daily cap, and health score.
- `campaign_recipients`: recipient email, company, source list, compliance basis, suppression state, and campaign token.
- `campaign_messages`: outbound message metadata, provider message ID, reply token, timestamps, and delivery status.
- `campaign_reply_events`: raw reply metadata, headers, body pointer, event type, and provider source.
- `oracle_triage_results`: classification, confidence, reason, extracted intent, next action, and review status.
- `global_suppressions`: unsubscribe, complaint, bounce, do-not-contact, source, timestamp, and provider sync state.
- `arrakis_promotions`: link between triage result and created/updated Arrakis lead or conversation.

The product-facing Arrakis tables/views should only receive data after promotion.

## 8. Compliance and Suppression

Every campaign must include:

- Accurate sender identity.
- Non-deceptive subject line.
- Clear company identity.
- Mailing address where required.
- Visible unsubscribe option.
- One-click unsubscribe where required by mailbox providers.
- Same-day suppression propagation.

Suppression must be global. If someone unsubscribes, complains, bounces hard, or asks to stop, the suppression should apply across:

- SendGrid Marketing Campaigns.
- Any fallback outbound platform.
- Campaign triage.
- Arrakis lead outreach.
- Future imported lists.

Suppression and complaint events should not wait for Ben or manual review.

## 9. Rollout Plan

### Phase 0: Access and DNS audit

Confirm:

- SendGrid plan and permissions.
- Current SendGrid API keys and webhook setup.
- DNS access for `kinro.com`.
- Existing SendGrid domain authentication.
- Inbound Parse or forwarding capability.
- Current suppression lists.
- Who owns campaign sending, Oracle triage, and Arrakis promotion.

Exit criteria:

- We know whether SendGrid can support the v1 campaign lane.
- We know which domain/subdomain will be used first.
- We know how replies will enter the triage endpoint.

### Phase 1: SendGrid pilot setup

Build/configure:

- One campaign subdomain, such as `updates.kinro.com`.
- One sender, such as `quotes@updates.kinro.com`.
- Domain authentication.
- Unsubscribe group.
- Event webhook.
- Reply-To routing to `reply+token@replies.kinro.com`.
- Triage endpoint.

Exit criteria:

- Internal seed emails send successfully.
- Replies reach triage.
- Unsubscribes update suppression.
- No reply appears in Arrakis unless promoted.

### Phase 2: Oracle triage

Build:

- Classification prompt/rules.
- Confidence thresholds.
- Serious enquiry promotion rule.
- Manual review state for `maybe_serious`.
- Suppression actions for unsubscribe, complaint, bounce, and stop requests.
- Metrics for filtered reply volume.

Exit criteria:

- Oracle correctly separates serious replies from noise in seed tests.
- Ben sees only serious or manually approved items.

### Phase 3: Arrakis promotion

Build:

- Internal promotion job/API from triage into Arrakis.
- Arrakis dashboard item only for serious leads.
- Ben mission object.
- Same-thread reply support.
- Future reply routing into the existing Arrakis conversation flow.

Exit criteria:

- A serious reply becomes an Arrakis lead/conversation.
- Oracle mission appears with context.
- Ben can reply from `ben@kinro.com`.
- Future replies stay attached to the conversation.

### Phase 4: Small campaign

Run:

- One high-fit audience segment.
- Low volume.
- Manual review of every promoted lead.
- Daily bounce, complaint, unsubscribe, positive reply, and promotion-rate review.

Exit criteria:

- Low bounce rate.
- Near-zero complaints.
- Suppression works.
- Arrakis stays clean.
- Ben receives only serious opportunities.

### Phase 5: Scale decision

Stay on SendGrid if:

- Deliverability remains healthy.
- Sender/subdomain needs are simple.
- Reply triage works.
- Campaign volume is manageable.

Add Smartlead if:

- Kinro needs multiple inboxes per domain.
- Kinro needs sender warmup and rotation.
- Kinro needs cold-outbound deliverability tooling.
- SendGrid campaign infrastructure becomes the limiting factor.

## 10. Success Metrics

Campaign health:

- Bounce rate under 2%.
- Spam complaints near zero.
- Same-day suppression propagation.
- Positive reply rate by campaign, sender, domain, and audience.

Triage quality:

- Percent of replies classified as serious.
- Percent classified as maybe serious.
- False positive rate into Arrakis.
- False negative rate found during manual QA.
- Time from customer reply to Oracle decision.

Arrakis cleanliness:

- Zero unsubscribe/off-topic/OOO/bounce replies in Ben workflow.
- Only serious or approved maybe-serious replies on dashboard.
- Every promoted lead has campaign context and Oracle mission.

Ben execution:

- Ben has enough context to reply without reading raw campaign logs.
- Same-thread replies work.
- Future customer replies remain attached to the Arrakis conversation.

## 11. Final Build Order

1. Confirm SendGrid can support the first campaign lane.
2. Choose the first campaign subdomain and reply subdomain.
3. Configure SendGrid authentication, unsubscribe, event webhooks, and reply routing.
4. Build the triage endpoint and campaign reply/event objects.
5. Build Oracle classification and suppression actions.
6. Build serious-reply promotion into Arrakis.
7. Build Ben mission generation.
8. Preserve same-thread reply headers for Ben.
9. Run internal seed tests.
10. Run a small campaign with manual review.
11. Decide whether SendGrid is enough or Smartlead is needed for scale.

## 12. References

- SendGrid Marketing Campaigns documentation: https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-email-with-marketing-campaigns
- SendGrid domain authentication: https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication
- SendGrid unsubscribe groups: https://www.twilio.com/docs/sendgrid/ui/sending-email/create-and-manage-unsubscribe-groups
- Gmail sender guidelines: https://support.google.com/mail/answer/81126
- FTC CAN-SPAM guide: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
