# n8n LLM Alert Triage for Uptime Kuma
Tags: n8n-workflow | ai-ops | automation | alert-management | telegram-bot | kubernetes
An n8n workflow that keeps noisy infrastructure alerts away from human operators by letting a Large Language Model perform the last-mile triage. The flow is designed for Uptime Kuma heartbeats today, but its building blocks make it easy to plug in other alert sources.
<img width="1140" height="308" alt="image" src="https://github.com/user-attachments/assets/4f19639a-78a0-4d66-982c-fa27ef2fc79c" />

## Key Features
- Receives Uptime Kuma heartbeat webhooks and tracks per-monitor incident state in n8n static data for clean up/down transitions.
- Waits for configurable grace periods (default 3 minutes) to filter out brief flaps before revalidating the endpoint.
- Performs a follow-up HTTP check with custom headers to gather objective evidence (status code, headers, response body).
- Calls Google Gemini 2.5 Pro through n8n LangChain nodes, using a strict Traditional Chinese prompt to reply with a concise summary or the literal token `SILENT` when no escalation is needed.
- Deduplicates alerts with incident IDs so the same outage cannot page the on-call channel multiple times.
- Sends curated Telegram notifications only when the LLM flags a real production issue, tagging individual responders and team rooms separately.

## Repository Layout
- `uptime_kuma_alert.json` – the exported n8n workflow.
- `LICENSE` – MIT License for the project.

## Architecture
```
Uptime Kuma (heartbeat webhook)
        │
        ▼
   Webhook node ──► If down? ──┐
        │                      │false (recover) → Code1 records last UP
 true (down)                   │
        ▼                      │
      Code2     (mark incident + timestamp)
        │
        ▼
       Wait (3 minutes)
        │
        ▼
      Code3 (check if recovered / already notified)
        │
        ▼
   If no up pending? ──► HTTP Request (evidence)
                              │
                              ▼
                    GeminiModel1 + Basic LLM Chain
                              │
                              ▼
                       Code (enforce SILENT policy)
                              │
                              ▼
                     If need alert?
                      │          │
                      ▼          ▼
                 Send Cracya   Send Wiz-運維
```

### Incident State Machine
- `Code1` stores the timestamp of the last successful heartbeat (`lastUpAt`).
- `Code2` records a down event (`lastDownAt`) and creates an `incidentId`.
- `Wait` + `Code3` confirm the outage is still active after the grace period; if the service recovered, the workflow stops quietly.
- `If no up` ensures only active, non-notified incidents continue to verification.

### LLM Triage Layer
The LangChain `Basic LLM Chain` injects a prompt that requires the model to:
1. Describe active incidents in ≤180 Traditional Chinese characters.
2. Return the literal string `SILENT` (no extra spaces) when no escalation is needed.
`Code` enforces this contract and exposes `shouldNotify`/`message` flags for the downstream `If need alert` check.

## Prerequisites
- n8n 1.50 or later with Community Nodes enabled (for `@n8n/n8n-nodes-langchain`).
- An Uptime Kuma instance capable of sending heartbeat webhooks.
- A Google Gemini API key with access to `models/gemini-2.5-pro`.
- A Telegram bot token plus target chat IDs (one-to-one and team channel) with permission to send messages.

## Getting Started
1. **Import the workflow**
   - In the n8n UI choose *Import from File*, select `uptime_kuma_alert.json`, and confirm.
2. **Configure credentials**
   - Update the `Google Gemini (PaLM) API account` credential with your API key and project information.
   - Point both Telegram nodes to your bot credential and adjust `chatId` values to your team.
3. **Set the webhook URL**
   - The provided path is `f74fcc24-574b-4141-b404-7a9c25262405`. Adjust it if you prefer a new path and redeploy the webhook node.
   - Ensure your n8n instance is reachable from Uptime Kuma (public URL or tunnel).
4. **Connect Uptime Kuma**
   - Edit the monitor, enable *Heartbeat > Push*, and paste the webhook URL (including the n8n base URL and `/webhook/` prefix).
5. **Test the flow**
   - Trigger a synthetic down event in Uptime Kuma. Observe the run in n8n: you should see the `Wait` hold, the verification HTTP request, and—if still failing—a Telegram message summarizing the incident.

## Customization Tips
- **Grace period:** Change the `Wait` node duration to fit your tolerance for flapping services.
- **Verification request:** Modify the `HTTP Request` node (method, headers, auth) to match the monitored endpoint.
- **LLM policy:** Edit the prompt inside `Basic LLM Chain` to switch languages, adjust summarization style, or inject runbook links.
- **Notification channels:** Replace the Telegram nodes with Slack, PagerDuty, or any other connector by swapping node types while keeping the `shouldNotify` contract.
- **New alert sources:** Duplicate the `Webhook` + branching logic to handle additional payload formats. Just normalize the incoming JSON before passing it into the existing state machine.

## Operational Notes
- Incident state (timestamps, dedupe flags) lives in n8n `global` static data. Restarting n8n clears this cache, so consider persisting critical state elsewhere if you rely on long outages.
- The workflow assumes Uptime Kuma sends `body.heartbeat.status === 0` for downtime. Update the `If down` condition if your data differs.
- `active: true` in the export enables auto-execution on import; disable it before editing if you need a safe staging run.

## Roadmap Ideas
- Add automated runbook links based on monitor name or tags.
- Ship notifications with JSON attachments for richer observability.
- Support additional LLM providers or a local model fallback.
- Persist incident metadata in an external store for audit trails.

## Contributing
Issues and pull requests are welcome. Please open a discussion if you plan major changes to the workflow structure or alerting semantics.

## License
Released under the [MIT License](./LICENSE).
