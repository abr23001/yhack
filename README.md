# Agentic AI for Inventory Management (Planning Doc)

This document captures all information needed before implementation.

Scope: design an agentic AI system using LangChain to improve inventory planning, replenishment, fulfillment reliability, and exception handling.

Constraint from project owner: do not build yet, only define what is needed.

## 1. Product Goal

Build an AI operations assistant that can:
1. Monitor inventory health in near real time.
2. Forecast demand and identify stockout or overstock risk.
3. Recommend and optionally trigger replenishment actions.
4. Explain decisions in plain business language.
5. Escalate uncertain or high-risk actions to humans.

## 2. Business Outcomes and KPIs

Primary KPIs:
1. Stockout rate (% of SKUs with zero inventory during demand).
2. Fill rate / service level.
3. Inventory turnover.
4. Days of inventory on hand (DOH).
5. Carrying cost.
6. Expedited shipping spend.
7. Forecast accuracy (MAPE, WAPE).

Secondary KPIs:
1. Planner productivity (hours saved).
2. Mean time to detect exceptions.
3. Recommendation acceptance rate.
4. False positive and false negative alert rates.

## 3. Users and Roles

1. Inventory Planner: reviews forecasts, reorder proposals, exceptions.
2. Procurement Manager: approves purchase orders, supplier decisions.
3. Warehouse Operations: receives, picks, cycle counts, slotting feedback.
4. Finance: reviews working capital and carrying cost impact.
5. Executive/Ops Lead: needs summary dashboards and risk posture.
6. System Admin: handles integrations, permissions, audits.

## 4. Core Use Cases

1. Reorder recommendation by SKU-location with confidence score.
2. Dynamic safety stock tuning based on demand volatility and lead-time uncertainty.
3. Multi-echelon transfer suggestions (warehouse-to-warehouse).
4. Slow-moving and dead-stock identification with markdown options.
5. Supplier delay risk detection from PO and lead-time drift.
6. Exception triage agent that prioritizes highest business impact.
7. Natural language Q&A:
	- What will stock out next week?
	- Why did SKU A reorder point increase?
	- What is tied-up cash in excess stock for Category X?

## 5. Functional Requirements

Data and analytics:
1. Ingest historical sales, on-hand inventory, inbound POs, returns, lead times, seasonality signals.
2. Maintain SKU-location daily feature table.
3. Support demand forecast methods (baseline statistical + optional ML).
4. Compute reorder point, EOQ/min-max, and safety stock.
5. Produce scenario simulations (supplier delay, demand spike).

Agent behavior:
1. Tool-using agent that can query systems, run analytics, draft actions.
2. Policy-aware decisioning (approval gates by dollar amount/risk level).
3. Structured output for actions: action type, sku, qty, location, rationale, confidence.
4. Human-in-the-loop approvals for critical actions.
5. Persistent task context and event history.

Workflow and UI:
1. Daily briefing digest.
2. Exception inbox with severity ranking.
3. Decision log with explainability and audit trace.
4. Chat interface for planners and managers.

## 6. Non-Functional Requirements

1. Availability target (example): 99.9% for read workflows.
2. Latency targets:
	- Interactive chat under 6 seconds for common queries.
	- Batch recommendations produced before business day starts.
3. Security: RBAC, least privilege, encrypted data in transit and at rest.
4. Compliance and auditability: immutable action logs and model traceability.
5. Reliability: idempotent tool calls, retries, dead-letter queue.
6. Cost controls: token budget limits, model routing by complexity.
7. Observability: traces, metrics, alerting.

## 7. Data Requirements

Required entities:
1. SKU master: sku_id, category, unit_cost, supplier_id, pack_size, shelf_life.
2. Location master: location_id, type, capacity, region.
3. Inventory snapshots: sku_id, location_id, on_hand, reserved, available, timestamp.
4. Demand history: sku_id, location_id, date, quantity_sold, returns.
5. Purchase orders: po_id, supplier_id, sku_id, qty, order_date, eta, actual_receipt.
6. Lead-time history by supplier/SKU.
7. Transfer orders and fulfillment records.
8. Price/promotions and seasonality calendar.

Data quality checks:
1. Missingness thresholds by column.
2. Outlier detection (abnormal demand/lead-time jumps).
3. Late-arriving event handling.
4. Unit of measure normalization.
5. Duplicate event dedupe keys.

## 8. System Integrations

Potential source systems:
1. ERP (POs, supplier master, financial dimensions).
2. WMS (on-hand, movements, cycle counts).
3. OMS/e-commerce (orders, cancellations, returns).
4. BI warehouse (historical analytics and reporting).
5. Supplier portals/EDI.

Integration method options:
1. Batch ETL for nightly planning.
2. Event streaming for near-real-time exceptions.
3. API connectors for on-demand agent actions.

## 9. LangChain-Centric Architecture

Recommended building blocks:
1. LangChain Agents for orchestration of tool calls and reasoning.
2. LangGraph for durable, stateful multi-step workflows.
3. Tool abstractions for:
	- SQL warehouse query tool.
	- Forecast service tool.
	- Policy/rules engine tool.
	- ERP action tool (create draft PO, create transfer request).
4. Retrieval pipeline for SOPs, policy docs, supplier contracts (RAG).
5. Structured output parsing with strict JSON schema.
6. Memory/state store for case context, prior decisions, and approvals.
7. Callback/tracing integration (LangSmith or equivalent) for observability.

High-level agent graph:
1. Trigger node (schedule/event/user chat).
2. Context builder node (fetch data + relevant docs).
3. Analyst node (forecast/risk/scenario calculations).
4. Policy node (approval and compliance checks).
5. Decision node (recommend, request approval, or execute).
6. Notification node (chat/email/dashboard updates).
7. Audit logger node.

## 10. Agent Design Details

Agent roles (multi-agent optional):
1. Triage Agent: classifies and prioritizes exceptions.
2. Planning Agent: computes replenishment and safety stock recommendations.
3. Execution Agent: creates draft actions in ERP/WMS.
4. Explainer Agent: produces manager-friendly rationale.

Prompting requirements:
1. System prompt must encode business policies and forbidden actions.
2. Use few-shot examples for common exception classes.
3. Enforce citation of data fields and timestamps used in decisions.
4. Force abstain/clarify behavior on low confidence or missing data.

Output contract example fields:
1. recommendation_id
2. action_type
3. sku_id
4. location_id
5. qty
6. expected_service_level_impact
7. expected_working_capital_impact
8. confidence_score
9. rationale
10. required_approval_role

## 11. Governance, Risk, and Safety

1. Approval policy matrix by risk and dollar impact.
2. Hard guardrails:
	- Never auto-order controlled/restricted SKUs without explicit approval.
	- Never exceed supplier contract caps.
3. Hallucination controls:
	- Tool-grounded responses only for numeric claims.
	- If data unavailable, explicitly say unknown.
4. Drift detection for forecast error and lead-time behavior.
5. Rollback and kill switch for automated actions.
6. Full decision trace retention.

## 12. Evaluation Plan

Offline evaluation:
1. Backtest reorder policy on 6-24 months of historical data.
2. Compare against baseline planner policy.
3. Metrics: stockout reduction, carrying cost delta, service level uplift.

Online evaluation:
1. Shadow mode first (recommend only).
2. A/B or phased rollout by category or region.
3. Track acceptance, overrides, and business impact.

LLM quality evaluation:
1. Tool-use correctness.
2. JSON schema adherence.
3. Policy compliance rate.
4. Explanation faithfulness.

## 13. MLOps and LLMOps Requirements

1. Prompt/version control.
2. Model registry and routing strategy.
3. Automated regression test set for prompts and tools.
4. Continuous evaluation dashboards.
5. Incident playbooks for bad recommendations or integration failures.

## 14. Delivery Phases

Phase 0: Discovery (no build)
1. Confirm business rules, approval matrix, data availability.
2. Define success metrics and reporting cadence.

Phase 1: Data and baseline analytics
1. Build data contracts and quality checks.
2. Implement baseline forecasting and reorder logic.

Phase 2: Agent in shadow mode
1. LangChain/LangGraph orchestration.
2. Recommendation generation and explanation only.

Phase 3: Human-approved execution
1. Draft PO/transfer creation with approvals.
2. Expand to more categories/locations.

Phase 4: Controlled automation
1. Auto-execution for low-risk scenarios.
2. Continuous optimization and governance.

## 15. Team and Ownership

1. Product owner: operations objectives and prioritization.
2. Data engineering: pipelines and quality.
3. Data science: forecast and optimization methods.
4. AI engineering: LangChain orchestration and guardrails.
5. Platform/SRE: deployment, monitoring, reliability.
6. Business SMEs: planner and procurement validation.

## 16. Open Questions to Resolve Before Build

1. What planning cadence is required: hourly, daily, weekly?
2. Which actions can be fully automated vs always approved?
3. What is the acceptable tradeoff between service level and carrying cost?
4. Are there SKU classes needing separate policy (perishable, regulated, seasonal)?
5. Which systems are source of truth for on-hand and lead time?
6. What data latency is acceptable for each workflow?
7. What budget constraints exist for model and infrastructure costs?

## 17. Minimum Dataset and Access Checklist

1. Read access to ERP, WMS, OMS, and warehouse tables.
2. Historical demand and inventory snapshots for at least 12 months.
3. Supplier lead-time and PO outcome history.
4. Policy documentation (approval thresholds, contract constraints).
5. Named stakeholders and approval owners.
6. Sandbox environment for safe shadow testing.

## 18. Definition of Ready (Before Coding)

Implementation can start only when:
1. KPI targets are agreed and baselines measured.
2. Data contracts and ownership are signed off.
3. Approval matrix and guardrails are documented.
4. Evaluation plan and launch criteria are approved.
5. Security and compliance requirements are accepted.
