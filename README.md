# AI Support Automation

## Overview
This n8n workflow automates customer support operations for a subscription-based mobile app (**PulseFit**). It processes a batch of incoming tickets, validates data, normalizes customer emails, classifies intents using OpenAI, applies deterministic business logic for subscriptions and troubleshooting, generates customer-facing responses, and logs all execution results into an n8n Data Table.

## How to Run
1. Import `workflow.json` into your n8n instance.
2. Ensure you have active OpenAI credentials configured in n8n.
3. The workflow starts with a mock data node containing `sample-input.json`. Click **Execute Workflow** to run the assessment.

## Main Design Decisions
- **Separation of Concerns:** AI is strictly used for probabilistic tasks (Intent Classification and Natural Language Response Generation). All financial calculations, subscription matching, and data validation are handled using deterministic JavaScript logic.
- **Data Preservation across LLM Nodes:** Standard AI nodes replace execution items. To prevent data loss, the workflow uses index-based mapping (`$('node_name').all()[i].json`) to securely merge back original ticket attributes (`ticket_id`, `product`, `subscriptions`) after AI processing.
- **Batch Resiliency:** Implemented non-blocking error handling. If a single ticket contains invalid data, it is safely routed to an error path while the rest of the batch continues processing normally.

## AI vs. Deterministic Logic
- **AI (Probabilistic):**
  - **Classifier:** Categorizes the request intent into `cancellation`, `cancellation_with_billing_clarification`, `refund`, `technical_issue`, or `other`.
  - **Response Generator:** Drafts a professional, context-aware reply based on the exact action status and results.
- **Deterministic (Rule-based):**
  - **Email Normalization:** Trims whitespace, converts to lowercase, and strips Gmail `+alias` tags.
  - **Subscription Matching:** Counts matching subscriptions by product regardless of status using pure code logic.
  - **Action Rules:** Determines whether to execute simulated success, require review, or skip based on active subscription counts.

## Error-Handling, Logging & Storage
- **Validation Path:** Tickets missing mandatory fields (`ticket_id`, `message`, `product`) are captured by an `IF` node and routed directly to the logging stage with `execution_status: Failed` and a clear error message.
- **Persistent Logging:** All processed records (both successful and failed) are stored natively using **n8n Data Tables** (`support_logs`), capturing technical metrics alongside AI-generated customer-facing response text.

## Handling Low-Confidence AI Results in Production
If the AI classifier returns a confidence score below a predefined threshold (e.g., `< 0.85`), the workflow bypasses automated action execution. Instead, the ticket is routed to a human agent queue (tagged as `requires_review`), ensuring ambiguous requests are handled safely by human agents.

## Integration Question: Zendesk & API Security
*In production, this workflow would receive tickets from Zendesk and call backend APIs for billing operations. How would you connect it and secure credentials?*
- **Ingestion:** Zendesk Webhooks triggered by specific routing rules (e.g., ticket creation) send payloads directly to the n8n Webhook node.
- **Credentials Management:** Zendesk API tokens and backend billing API keys are stored securely inside n8n’s native encrypted Credentials store (never hardcoded in nodes).
- **Network Security:** Webhook endpoint authentication (Bearer Token / Basic Auth) and strict IP allowlisting on the backend APIs to ensure requests originate solely from n8n.

## Production Question: Scaling to 10,000 Tickets/Month
*Imagine this system processes 10,000 tickets/month and triggers financial actions via backend APIs. What are the three most important improvements before deploying to production?*
1. **Idempotency & State Management:** Implement a persistent state store (e.g., Redis or PostgreSQL) to track processed `ticket_id` hashes. This prevents duplicate API calls (e.g., double refunds) if a workflow execution retries due to a network timeout.
2. **Human-in-the-Loop (HITL) Shadow Mode:** Initially deploy financial actions in a "Shadow Mode," where the system drafts responses and simulates actions as internal notes in Zendesk, requiring agent approval. This validates real-world AI accuracy before fully automating live transactions.
3. **Advanced Observability & Alerting:** Integrate monitoring tools (Datadog/Sentry) to track LLM latency, API failure rates, and execution bottlenecks, with instant Slack alerts configured for critical failures.

## Limitations & Future Improvements
- Implement automated retry mechanisms with exponential backoff for transient OpenAI API rate limits or network timeouts.
- Add sub-workflows to modularize large blocks (e.g., separating the branch logic into reusable sub-workflows).
