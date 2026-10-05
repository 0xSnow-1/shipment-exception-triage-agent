# Logistics Shipment Exception Triage Agent

[![Python](https://img.shields.io/badge/Python-3.10+-blue)](https://www.python.org/downloads/)
[![LangGraph](https://img.shields.io/badge/LangGraph-1.1.10-green)](https://langchain-ai.github.io/langgraph/)
[![LangChain](https://img.shields.io/badge/LangChain-1.2.18-blue)](https://python.langchain.com/)
[![AWS Bedrock](https://img.shields.io/badge/AWS-Bedrock-yellow)](https://aws.amazon.com/bedrock/)
[![LangSmith](https://img.shields.io/badge/LangSmith-Traced-orange)](https://smith.langchain.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

LangGraph/LangChain prototype for triaging logistics shipment exceptions with a supervisor-subagent architecture, deterministic tools, mock operational data, evidence logging, and human approval before customer-facing action.

## Problem

Logistics operators often triage shipment exceptions manually across carrier tracking, customer records, SLA rules, and internal communication channels. This creates slow response times, inconsistent escalation decisions, and risky customer updates when evidence is incomplete.

This prototype focuses on one narrow workflow:

```text
shipment exception input
-> load shipment record
-> classify exception
-> assess ETA/SLA/value/customer risk
-> inspect carrier tracking
-> choose resolution route
-> draft customer-safe update
-> save evidence
-> require human approval before customer-facing action
```

## Architecture

Graph ID:

```text
shipment_exception_supervisor
```

Main graph:

```text
agents.py:shipment_exception_supervisor
```

Supervisor responsibilities:

- load the shipment record first
- delegate independent analysis to subagents
- combine evidence into a final route and summary
- save audit artifacts
- pause for human approval before mock-sending a customer update

Subagents:

- `exception_classifier_agent`: normalizes carrier exception codes and classifies the business exception type
- `impact_assessment_agent`: evaluates ETA delta, SLA risk, shipment value risk, and severity
- `carrier_status_agent`: inspects tracking history, stale tracking, missing POD, and carrier follow-up needs
- `customer_risk_agent`: checks customer tier, complaints, SLA exposure, and notification risk
- `resolution_planner_agent`: chooses operational route and next action
- `communication_draft_agent`: drafts customer-safe and internal messages
- `evidence_report_agent`: saves final triage and evidence logs

Human-in-the-loop boundary:

- `mock_send_customer_update` interrupts before writing the approved customer update
- resume with `"approve"` in LangGraph Studio to approve the mock send
- resume with `"reject"` to refuse the send, or with `{"decision": "edit", "args": {...}}` to change the shipment, customer, or message and then approve
- approved sends are written to `outputs/approved_customer_updates.json`

## Repository Map

```text
agents.py                     supervisor and subagent wrappers
tools.py                      deterministic tools and mock persistence actions
langgraph.json                LangGraph graph export config
requirements.txt              pinned runtime dependencies
.env.example                  environment variable template
mock_data/shipments.json      source shipment records
mock_data/tracking_events.json mock carrier tracking events
mock_data/customers.json      mock customer/account records
mock_data/sla_rules.json      mock SLA rules
outputs/                      generated triage/evidence artifacts
outputs/archive/              earlier noisier runs kept for transparency
manual_workflow.md            manual process being automated
test_cases.md                 normal, messy, ambiguous, and HITL test cases
evidence_log.md               human-readable run evidence
failure_notes.md              bad outputs, fixes, and retests
what_stays_human.md           human approval boundaries
ai_usage.md                   AI usage disclosure
SUBMISSION.md                 challenge-style written response
```

## Setup

Requires Python 3.10 or newer.
Create and activate a virtual environment, then install dependencies:

```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Copy the environment template:

```bash
cp .env.example .env
```

```text
AWS_BEARER_TOKEN_BEDROCK=your_bedrock_token_here
LANGSMITH_TRACING=true
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGSMITH_PROJECT=shipment-exception-triage
```

`AWS_BEARER_TOKEN_BEDROCK` is the only required variable.
The `LANGSMITH_*` variables are optional and only control trace export.

## Run

Validate the graph:

```bash
langgraph validate
```

Start the local LangGraph server:

```bash
langgraph dev
```

The supervisor graph calls `global.anthropic.claude-opus-4-6-v1` through `ChatBedrock` in `us-east-1`, so the Bedrock credential must have access to that model.

Open the Studio URL printed by the command and select:

```text
shipment_exception_supervisor
```

## Demo

Recorded walkthrough of a live run, the tool-call trace, the human-in-the-loop approval, and the saved artifacts:

https://app.airtimetools.com/recorder/s/z_NSSd9oVPB3Err7TkVN1W

## Demo Prompts

Clean delay case:

```text
Triage shipment SHP-1001 using the mock data. Classify the exception, assess ETA/SLA/customer risk, inspect tracking, choose the route, draft a customer-safe update, save the evidence, and tell me whether human review is required.
```

Messy delivery dispute with HITL approval:

```text
Shipment SHP-1002 has carrier marked delivered, but customer says not received.
POD is missing.
Customer has recent delivery complaints.

Triage the shipment using the mock data. Classify the exception, assess ETA/SLA/customer risk, inspect tracking, choose the route, draft a customer-safe update, save the evidence, and then attempt to send the customer update through the approved customer update tool.
```

When the HITL interrupt appears, resume with:

```json
"approve"
```

`"reject"` refuses the send, and `{"decision": "edit", "args": {...}}` applies changed fields before approving.

Ambiguous missing-data case:

```text
Shipment SHP-1003 has missing promised delivery and unknown customer tier.
Tracking has no new scan after pickup.
```

## Expected Artifacts

Successful runs create or update:

```text
outputs/triage_results.json
outputs/evidence_log.jsonl
outputs/approved_customer_updates.json
```

Committed samples of all three files are already in `outputs/`.

The repo also includes human-readable evidence and failure analysis:

```text
evidence_log.md
failure_notes.md
```

## Verification

- `langgraph validate` reports a valid config and finds the one graph.
- `python -c "import agents"` compiles `shipment_exception_supervisor` without credentials.
- The deterministic tools in `tools.py` run offline against `mock_data/`.
- The four business test cases in `test_cases.md` were run by hand in LangGraph Studio; results are recorded in `evidence_log.md` and `failure_notes.md`.
- There is no automated test suite in this prototype.

## Safety Boundaries

The agent may draft, classify, score, recommend, and save evidence.

The agent must not autonomously approve refunds, credits, replacements, reships, carrier claims, financial remedies, or customer-facing messages. Customer-facing updates are gated through HITL approval.

## Known Limitations

- Uses mock logistics data instead of live TMS, carrier, CRM, or email integrations.
- Uses a mock customer update tool instead of real email sending.
- Does not include production authentication or role-based permissions.
- Some business thresholds are simplified for demo clarity.
- LangSmith is used for trace review/observability, not as a full automated evaluation suite in this prototype.

## License

MIT. See [LICENSE](LICENSE).
