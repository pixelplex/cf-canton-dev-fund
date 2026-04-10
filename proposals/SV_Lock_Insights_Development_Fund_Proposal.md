# Development Fund Proposal

## Super Validator Intelligence & Risk Management Dashboard (SV Lock Insights)

**Author:** Yury Korzun (yury.korzun@pixelplex.io)  
**Website:** https://ccview.io  
**Existing prototype:** https://ccview.io/super-validators/?table=svLocking  
**Status:** Submission Draft  
**Updated:** 2026-04-09

---

## 1. Abstract

This proposal requests **550,000 CC** from the Canton Development Fund to deliver **SV Lock Insights**, a production-grade operational intelligence layer for Canton Network Super Validators.

The proposal builds on the existing SV locking dashboard prototype and expands it into a full operational product that helps Super Validators:

- maintain compliance with locking and tier requirements,
- understand current and future unlocking exposure,
- reconcile rewards and lock-related value flows,
- receive proactive alerts before operational issues become incidents,
- reduce manual analysis through wallet intelligence and transaction labeling.

The requested work is designed as a sequence of concrete milestones, each with clear deliverables, measurable acceptance criteria, and ecosystem-facing utility. The final milestone includes a six-month support and iteration period to ensure that the delivered system remains useful as validator needs, data patterns, and network rules evolve.

This proposal is intentionally focused on **operational tooling and analytics**, not on transaction custody or validator node operations. The result will be a read-only intelligence layer that improves visibility, reduces operator risk, and creates a more mature operational baseline for the Canton validator ecosystem.

---

## 2. Problem Statement

Super Validators are expected to monitor locking positions, tier thresholds, reward flows, unlocking schedules, and operational buffers with a high degree of accuracy. In practice, much of this analysis is fragmented across raw wallet activity, manual calculations, spreadsheets, and ad hoc interpretation of transaction history.

This creates several problems:

1. **Compliance risk.** Operators may not immediately see how much additional locking is required to preserve a tier, or how much can be safely unlocked without falling below a threshold.
2. **Poor visibility into future state.** Unlocking tranches, gradual unlock rates, and reward interactions are difficult to reconstruct from raw data alone.
3. **Delayed reaction time.** Without alerts, operators only discover problems after a threshold is crossed or a buffer has already become too thin.
4. **High manual overhead.** Wallets and transactions often need manual interpretation to distinguish lock, unlock, treasury, reward, or ambiguous flows.
5. **No standard operational view.** Different operators may arrive at different conclusions from the same data because there is no consistent analytics layer focused on SV operations.

A dedicated Super Validator operations dashboard addresses these issues by converting protocol-relevant data into actionable decisions.

---

## 3. Objective and Ecosystem Value

The objective of this proposal is to turn the current dashboard prototype into a production-ready **Super Validator intelligence and risk management system**.

The delivered system will provide value to the Canton ecosystem in five ways:

- **Reduce avoidable operator error** by translating protocol state into clear actions.
- **Improve resilience** by surfacing warnings before tier, locking, or reward issues become urgent.
- **Standardize visibility** across Super Validators through a shared, repeatable analytics model.
- **Lower operational overhead** by automating classification, reconciliation, and monitoring.
- **Create reusable ecosystem tooling** that can inform future analytics, reporting, and operational best practices.

This proposal is aligned with the development fund’s purpose of supporting dev tools, critical infrastructure, and long-term ecosystem utility.

---

## 4. Current State

A first version of the SV locking dashboard is already live and demonstrates the feasibility of the analytics approach. However, the current prototype is intentionally narrow and does not yet provide:

- recommendation logic for maintaining or improving tier status,
- unlocking tranche analytics and forward-looking timelines,
- reward reconciliation views,
- alerting across external channels,
- wallet classification and transaction intelligence,
- structured support and iteration after initial delivery.

The requested funding expands this prototype into a durable operational product.

---

## 5. Scope

### In Scope

- Super Validator lock and tier compliance analytics
- recommendation logic and action-oriented UI
- unlocking tranche detection and projections
- reward flow aggregation and reconciliation views
- multi-channel notifications
- wallet intelligence and transaction labeling
- documentation, operator feedback loops, and production support

### Out of Scope

- changes to Canton Network consensus, tokenomics, or governance rules
- mandatory changes to Canton, Daml, or Splice core repositories for the initial delivery

The proposal is designed to be delivered primarily as an off-chain, read-only analytics system operating on top of existing network data sources and public or operator-approved endpoints.

---

## 6. Delivery Approach

SV Lock Insights will be delivered as an extension of the existing ccview.io analytics stack.

### 6.1 Architecture Approach

The system will use a layered architecture:

1. **Data ingestion layer** for collecting and normalizing validator-relevant wallet, lock, unlock, and reward data.
2. **Rules and computation layer** for tier logic, threshold calculations, unlocking schedules, and alert conditions.
3. **Presentation layer** for dashboard views, historical visualizations, and action panels.
4. **Notification layer** for alert delivery, deduplication, and channel-specific routing.
5. **Intelligence layer** for wallet classification, transaction labeling, confidence scoring, and manual overrides.

### 6.2 Product Principles

The implementation will follow these principles:

- **Deterministic calculations:** operators should be able to understand how each recommendation was produced.
- **Near real-time visibility:** the dashboard should reflect newly observed state quickly enough to be operationally useful.
- **Auditability:** wallet classifications and action logic must be explainable and reviewable.
- **Operator-first design:** the system should answer “what happened?”, “what changes next?”, and “what should I do now?”
- **Low protocol dependency:** the product should deliver value without requiring changes to core Canton repositories.

### 6.3 Operational Model

The dashboard itself will be public where appropriate, while notification preferences and administrative overrides will be controlled through privileged access. This keeps the base analytics broadly useful while preserving operator-specific workflows.

---

## 7. Detailed Milestones

### Milestone 1 — Compliance & Recommendation Engine

**Estimated duration:** 4 weeks  
**Funding requested:** **130,000 CC**

#### Objective

Provide clear, deterministic, and near real-time operational guidance that tells Super Validators what action is required to preserve their current position or move to a stronger one.

#### Detailed scope of work

1. **SV compliance state model**
   - Build a normalized internal model of SV status including current lock amount, relevant thresholds, current tier position, buffer above threshold, and pending lock-related changes.
   - Separate current observed state from projected state so operators can distinguish what is true now versus what becomes true after unlock progression or newly observed events.

2. **Recommendation engine**
   - Compute the minimum additional lock required to maintain the current tier.
   - Compute the minimum additional lock required to reach the next tier.
   - Compute the maximum amount that can be safely unlocked while preserving the current tier.
   - Support conservative calculations that account for pending unlock exposure where relevant.

3. **Actionability layer**
   - Add a dashboard panel showing current status as one of: healthy, warning, action needed, or upgrade opportunity.
   - Show exact values, threshold distance, recommended action, and rationale.
   - Highlight changes from the previous snapshot so operators can quickly identify why status changed.

4. **API and data export**
   - Expose recommendation outputs through API endpoints.
   - Provide machine-readable outputs for downstream dashboards, internal scripts, or alerting logic.

5. **Documentation**
   - Publish calculation notes explaining how each recommendation is derived.
   - Document assumptions, known limitations, and data freshness expectations.

#### Deliverables

- Production recommendation engine
- Required Actions panel in the dashboard
- API endpoints for recommendation and compliance data
- Public calculation notes and operator-facing usage documentation

#### Acceptance criteria

- Calculations are deterministic and reproducible from the same input state.
- Dashboard clearly shows current tier position, threshold distance, and recommended next action.
- API responses match dashboard values.
- The system updates frequently enough to be operationally useful for day-to-day monitoring.
- Documentation is sufficient for a reviewer to validate the recommendation logic.

#### Evidence of completion

- Live deployment on the production dashboard
- Demo covering several real or representative SV scenarios
- Written logic notes and endpoint reference

---

### Milestone 2 — Unlocking & Rewards Visibility

**Estimated duration:** 4 weeks  
**Funding requested:** **110,000 CC**

#### Objective

Give Super Validators a clear forward-looking view of unlocking mechanics and reward flows so they can understand not only current state, but also upcoming changes to their position.

#### Detailed scope of work

1. **Unlocking tranche detection**
   - Model lock positions as identifiable tranches where the underlying data permits.
   - Track tranche status, start date, progress, and remaining amount.
   - Distinguish between locked, unlocking, and fully unlocked states.

2. **Daily unlock progression**
   - Implement daily unlock progression logic, including the expected **1/365** release pattern where applicable.
   - Surface both the current unlocked portion and the projected next unlock points.

3. **Timeline visualization**
   - Add a time-based view showing upcoming unlock progression, remaining locked exposure, and projected release path.
   - Provide wallet-level and validator-level views where possible.

4. **Reward aggregation and reconciliation**
   - Aggregate historical reward flows relevant to SV positions.
   - Add views for reward history over time.
   - Create reconciliation logic so operators can compare lock-related expectations with observed reward-related activity.

5. **Ghost rewards tracking**
   - Surface “ghost rewards” as a dashboard concept for value that is economically attributable to an SV position but not immediately obvious from raw wallet history alone.
   - Show how these values are derived and where they fit into the broader reward picture.

6. **Exports and audit support**
   - Add exportable tables or machine-readable outputs for tranche, reward, and timeline data.

#### Deliverables

- Unlocking tranche tracker
- Daily unlock rate computation
- Unlocking timeline visualization
- Historical reward aggregation
- Ghost rewards view and reconciliation support
- Exportable reward and unlocking datasets

#### Acceptance criteria

- Unlocking tranches are consistently detected and presented.
- Daily unlock progression is correctly reflected in the timeline view.
- Reward history is aggregated in a way that is consistent with observed wallet activity and dashboard logic.
- Ghost reward values are clearly explained and not presented as unexplained balances.
- Reviewers can inspect both current values and forward projections.

#### Evidence of completion

- Live unlocking and reward views in production
- Example walkthrough using historical data
- Documentation of reward and unlocking interpretation

---

### Milestone 3 — Notification System

**Estimated duration:** 4 weeks  
**Funding requested:** **100,000 CC**

#### Objective

Enable proactive monitoring so operators receive warnings and action prompts before threshold breaches or other important events require urgent manual intervention.

#### Detailed scope of work

1. **Alert rule engine**
   - Implement alert rules for threshold breach risk, low compliance buffer, action-required conditions, reward anomalies, and unlock-related milestones.
   - Support both immediate alerts and scheduled summaries.

2. **Event lifecycle management**
   - Add deduplication, cooldowns, re-notify logic, and acknowledgement states.
   - Prevent noisy repeated alerts while preserving escalation for unresolved issues.

3. **Delivery channels**
   - Deliver notifications through:
     - dashboard inbox / UI notifications,
     - email,
     - webhooks,
     - Slack,
     - Telegram.

4. **Configuration layer**
   - Allow per-rule configuration of thresholds, channels, and notification preferences.
   - Support enable/disable settings for different alert types.

5. **Observability and reliability**
   - Record delivery attempts and message status.
   - Provide basic admin visibility into notification health and failures.

#### Deliverables

- Production notification engine
- Multi-channel alert delivery
- Alert settings and configuration UI
- Delivery logging and basic alert observability

#### Acceptance criteria

- Alerts can be configured without code changes.
- Threshold and action events generate timely notifications.
- Delivery works across the announced channels.
- Repeated alerts are controlled through cooldown or acknowledgement logic.
- Operators can test and validate their channel configuration.

#### Evidence of completion

- Working alert demonstrations across supported channels
- Configuration guide
- Sample alert catalogue with trigger definitions

---

### Milestone 4 — Wallet Intelligence Layer

**Estimated duration:** 4 weeks  
**Funding requested:** **100,000 CC**

#### Objective

Reduce manual interpretation by automatically classifying wallets and labeling lock-related transactions so operators can understand which flows matter and why.

#### Detailed scope of work

1. **Wallet classification engine**
   - Classify wallets into operational categories such as lock-related, reward-related, treasury-related, operational, unknown, or other relevant classes.
   - Use multiple signals including transaction patterns, recurring counterparties, amounts, behavior over time, and memo or metadata where available.

2. **Transaction labeling**
   - Label transactions as lock, unlock, reward, transfer, internal movement, or unknown where confidence is sufficient.
   - Display these labels directly in the dashboard to reduce manual reconciliation effort.

3. **Confidence and review workflow**
   - Introduce confidence scoring for automated classification.
   - Route low-confidence classifications into a review path rather than overstating certainty.

4. **Admin override tools**
   - Add internal tools to correct wallet mappings and transaction labels.
   - Preserve an audit trail of manual overrides so changes remain explainable.

5. **Continuous improvement path**
   - Make the ruleset maintainable and extensible so new patterns discovered during production use can be added without reworking the whole system.

#### Deliverables

- Wallet classification engine
- Transaction labeling layer
- Confidence scoring and review logic
- Admin override interface and audit trail

#### Acceptance criteria

- The system meaningfully reduces manual classification effort for known SV-related wallets and flows.
- Unknown or ambiguous cases are surfaced as such rather than misrepresented as certain.
- Manual overrides are reflected in the dashboard and preserved in an auditable way.
- The ruleset can be updated as new wallet patterns appear.

#### Evidence of completion

- Before/after examples showing reduced manual interpretation effort
- Admin workflow demo
- Documentation of classification categories and override behavior

---

### Milestone 5 — Support & Iteration Period

**Duration:** 6 months following acceptance of Milestone 4  
**Funding requested:** **110,000 CC**

#### Objective

Ensure the system remains stable, accurate, and useful in production after launch, while incorporating validator feedback and adapting to network or data-source changes.

#### Detailed scope of work

1. **Bug fixing and maintenance**
   - Resolve production issues affecting accuracy, performance, or usability.
   - Maintain availability and data integrity of dashboard features delivered in Milestones 1–4.

2. **Protocol and data adaptation**
   - Update dashboard logic if network rules, endpoint formats, or observable data patterns change.
   - Adjust calculations and displays where rule interpretation becomes clearer through ecosystem feedback.

3. **Wallet mapping and intelligence updates**
   - Continue improving wallet classification and known-entity mapping.
   - Refine transaction labels as additional real-world patterns are observed.

4. **Operator feedback loop**
   - Gather feedback from SV operators and prioritize practical improvements.
   - Deliver small iteration releases that improve clarity and operational usefulness.

5. **Performance and reliability improvements**
   - Optimize data refresh, processing time, and dashboard responsiveness where necessary.
   - Improve observability and internal support tooling.

6. **Documentation and handover readiness**
   - Maintain updated calculation notes, feature docs, and operator guidance.
   - Keep the delivered system understandable enough that future contributors can continue the work if needed.

#### Deliverables

- Six months of active maintenance and iteration
- Ongoing wallet mapping and logic updates
- Monthly or periodic improvement releases as needed
- Updated documentation and handover-ready operational notes

#### Acceptance criteria

- Delivered features remain functional and aligned with current observable network behavior.
- Operator feedback results in meaningful iterative improvements during the support period.
- Documentation remains current enough to support review, maintenance, and future continuation.
- Critical issues are triaged and addressed within a reasonable operational timeframe.

#### Evidence of completion

- Maintenance changelog covering the support period
- Summary of fixes and improvements delivered
- Updated documentation set at the end of the support window

---

## 8. Delivery Timeline

Assuming prompt approval, the expected delivery schedule is:

| Period | Milestone |
|---|---|
| Weeks 1–4 | Milestone 1 — Compliance & Recommendation Engine |
| Weeks 5–8 | Milestone 2 — Unlocking & Rewards Visibility |
| Weeks 9–12 | Milestone 3 — Notification System |
| Weeks 13–16 | Milestone 4 — Wallet Intelligence Layer |
| Months 5–10 after approval (approximately) | Milestone 5 — Support & Iteration |

The first four milestones are designed to be completed within roughly one quarter each in accordance with the development fund’s preference for incremental, reviewable delivery.

---

## 9. Funding Request

### 9.1 Milestone Budget

| Milestone | Funding (CC) |
|---|---:|
| Milestone 1 — Compliance & Recommendation Engine | 130,000 |
| Milestone 2 — Unlocking & Rewards Visibility | 110,000 |
| Milestone 3 — Notification System | 100,000 |
| Milestone 4 — Wallet Intelligence Layer | 100,000 |
| Milestone 5 — Support & Iteration Period | 110,000 |
| **Total** | **550,000** |

### 9.2 Payment Structure

Funding is requested on a **milestone acceptance basis**.

- Milestones 1–4 are payable upon acceptance of the corresponding deliverables.
- Milestone 5 is proposed as a maintenance milestone payable in **two quarterly tranches of 55,000 CC** each, subject to continuation review.

### 9.3 Price Volatility Treatment

This proposal is denominated in **CC**.

- For **Milestones 1–4**, funding amounts are fixed in CC and the recipient accepts normal market volatility.
- **Milestone 5** extends beyond six months from proposal approval. For that reason, before the second 55,000 CC maintenance tranche is approved, the parties may request a committee review if the **30-day average CC/USD reference price** has moved by more than **40%** versus the 30-day average at the time of proposal approval.
- Unless the Tech & Ops Committee decides otherwise after such a review, the milestone remains payable in CC at the stated amount.

This clause is included to make long-dated maintenance funding explicit and reviewable while keeping the proposal simple and CC-native.

---

## 10. Dependencies and Assumptions

This proposal assumes:

- continued availability of the network data needed to observe lock, unlock, reward, and wallet activity,
- access to public or operator-approved endpoints sufficient for dashboard computation,
- timely clarification of edge-case rule interpretation if required,
- no mandatory core protocol changes are required for the initial implementation.

If protocol or endpoint changes occur after delivery of the initial milestones, they will be handled within Milestone 5 where feasible.

---

## 11. Security and Operational Considerations

SV Insights is intended to be a **read-only operational intelligence tool**.

It will not:

- custody user funds,
- initiate transactions,
- sign on behalf of validators,
- alter validator or protocol state.

Notification channel configuration and any privileged administrative features will be handled with appropriate operational care, and internal override tooling will be restricted to authorized maintainers.

---

## 12. Co-Marketing and Ecosystem Contribution

No separate co-marketing budget is requested.

However, this proposal would benefit from normal ecosystem coordination after approval, including:

- permission to publicly announce the grant,
- permission to publish milestone updates,
- one shared ecosystem demo or walkthrough upon material completion,
- inclusion of the delivered dashboard in relevant Canton ecosystem tooling references if the committee finds it useful.

The goal is not only to build the product, but also to make its utility visible to the validator community.

---

## 13. Why This Proposal Is a Good Fit for the Development Fund

This proposal fits the development fund for three reasons:

1. **It is a developer and operator tool.** The deliverable is infrastructure-like analytics that improves how ecosystem participants operate.
2. **It creates durable utility.** Once built, the dashboard becomes a reusable operational layer rather than a one-off report.
3. **It includes maintenance.** The proposal does not stop at launch; it includes a structured support period so the tool remains useful and reviewable.

The work also has a practical, ecosystem-facing outcome: it makes Super Validator operations easier to understand, easier to monitor, and less dependent on manual interpretation.

---

## 14. Conclusion

SV Insights turns an early dashboard prototype into a production-ready Super Validator operations product.

By combining compliance recommendations, unlock and reward visibility, proactive alerts, wallet intelligence, and a six-month support period, this proposal delivers concrete operational value to the Canton ecosystem.

The requested **550,000 CC** will fund a structured sequence of milestones with clear outputs, measurable acceptance criteria, and direct relevance to validator resilience and ecosystem maturity.

