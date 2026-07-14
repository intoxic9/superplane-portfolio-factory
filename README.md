# Portfolio Factory

**Portfolio Factory** is a SuperPlane automation that turns Slack conversations into live personal portfolio websites. Mention a Slack bot with a design brief, review a generated preview, approve in one click, and deploy to production through GitHub and Render.

This repository contains a sanitized, import-ready SuperPlane canvas (`canvas.yaml`) and documentation for recreating the workflow in your own workspace.

---

## What Portfolio Factory does

Portfolio Factory listens for Slack `@bot` mentions, accumulates thread context as conversational memory, asks GPT-5 (via OpenRouter) to produce a complete single-file HTML portfolio, pushes a preview branch to GitHub for review, waits for human approval in Slack, then deploys the approved site to production and monitors the Render deploy until it is live.

---

## The problem it solves

Building a portfolio from scratch usually means writing HTML/CSS, iterating on design feedback, configuring hosting, and deploying manually. That loop is slow when someone just wants a polished site from a short brief—or when they refine the design mid-conversation (“make it darker”, “add a Speaking section”).

Portfolio Factory compresses that loop into a single Slack thread:

1. Describe what you want (or refine a prior version in-thread).
2. Review a browser preview.
3. Approve to ship to production.

No local build tooling is required for the generated site; each portfolio is a self-contained `index.html`.

---

## End-to-end workflow

1. **Slack mention** — A user mentions the Slack app in a channel (or replies in a thread).
2. **Acknowledge** — The bot posts an ack message in the thread.
3. **Clean prompt** — Mentions, Slack link markup, and optional `repo:owner/name` overrides are stripped/normalized.
4. **Save thread message** — The cleaned text is stored in SuperPlane memory under the `thread_messages` namespace.
5. **Load thread history** — Prior messages for the same Slack thread are loaded.
6. **Build accumulated prompt** — History + latest message become the LLM brief (later messages override earlier ones when they conflict).
7. **Generate with GPT-5** — OpenRouter is called with `openai/gpt-5` to produce HTML (or a `NEED_MORE_INFO` response).
8. **Validate HTML** — Output must look like HTML (starts with `<`, meaningful length). Insufficient briefs go back to Slack with a clarifying question.
9. **Push preview branch** — Valid HTML is force-pushed to `preview/<slack_user_id>` and a [htmlpreview.github.io](https://htmlpreview.github.io) URL is built.
10. **Slack approval gate** — Approve & Deploy / Reject buttons are posted (10-minute timeout).
11. **Push to main** — On approval, HTML is committed and pushed to the default branch.
12. **Wait for Render** — The canvas polls the Render deploy API until `live` or a failure status.
13. **Verify live website** — A GET against the service URL checks for HTTP 200.
14. **Save portfolio metadata** — Success details are stored in the `portfolios` memory namespace.
15. **Notify in Slack** — The live URL is posted back to the thread.

Rejection, timeout, validation, GitHub, and Render failure paths notify the user and end the run cleanly. See [architecture.md](./architecture.md) for the full flowchart.

---

## Slack conversational memory

The canvas uses two SuperPlane memory namespaces:

| Namespace | Purpose |
|---|---|
| `thread_messages` | Per-thread history keyed by Slack `thread_anchor` (and message `ts`) |
| `portfolios` | Final deploy metadata keyed by Slack `user_id` |

Every `@bot` mention is saved before generation. On later messages in the same thread, the workflow loads the full history, so refinements such as “make it darker” or “drop the blog section” combine with the original brief. The Build Prompt node formats oldest-first history and emphasizes the latest message.

---

## GPT-5 website generation through OpenRouter

The Generate Site node calls OpenRouter’s chat completions API with model `openai/gpt-5`. The system-style brief asks for:

- A complete, self-contained `index.html` (inline CSS/JS only)
- Responsive layout, semantic HTML, and accessible contrast
- Sensible defaults when details are missing
- `NEED_MORE_INFO: …` when the brief is too vague to build

The OpenRouter API key is referenced as the SuperPlane secret `OPENROUTER_API_KEY` (never embedded in the canvas).

---

## Preview branches in GitHub

On successful HTML validation, Push Preview:

- Clones the target repository (default `YOUR_GITHUB_USERNAME/YOUR_REPOSITORY`, or a per-request `repo:owner/name` override)
- Writes `index.html` on branch `preview/<slack_user_id>`
- Force-pushes that branch
- Returns a rendered preview URL via htmlpreview.github.io

This lets reviewers see the page without merging to production.

---

## Human approval in Slack

The Approval Gate node posts Slack buttons:

- **Approve & Deploy** — continue to production push + Render monitoring
- **Reject** — discard this version; the user can refine in-thread (memory is retained)

If no one clicks within 10 minutes, a timeout notification is sent with the preview link still available.

---

## Render deployment monitoring

After a production push, the canvas:

1. Polls `GET /v1/services/YOUR_RENDER_SERVICE_ID/deploys?limit=1` on a 15-second interval (loop up to 30 iterations / 15 minutes)
2. Stops when status is `live`, `build_failed`, `update_failed`, or `canceled`
3. On `live`, fetches the service URL and verifies HTTP 200
4. Notifies Slack on success or failure

Your Render static/web service should be connected to the same GitHub repository and deploy from the production branch.

---

## Error handling

The canvas includes dedicated Slack notifications (and a shared On Error noop sink) for:

| Path | When |
|---|---|
| Insufficient information | Model returns `NEED_MORE_INFO…` |
| Validation error | Output is not valid-looking HTML |
| GitHub / preview error | Clone or push preview fails |
| Rejection | User clicks Reject |
| Approval timeout | No button click within 10 minutes |
| Production push error | Push to main fails |
| Render API / deploy error | Render unreachable or deploy not `live` |
| Verify error | Live URL does not return 200 |
| Generic pipeline error | Clean Prompt / Build Prompt / Generate Site failures |

Error messages scrub GitHub token material from logs before writing results.

---

## Technologies used

- **[SuperPlane](https://superplane.com)** — canvas orchestration, secrets, memory, Slack integration nodes
- **Slack** — trigger (`onAppMention`), approval buttons, notifications
- **OpenRouter** — GPT-5 chat completions for HTML generation
- **GitHub** — preview branches + production `index.html`
- **Render** — hosting and deploy status API
- **Bash + jq + curl + git** — runner scripts inside SuperPlane action nodes

---

## Repository structure

```text
.
├── LICENSE
├── README.md
├── architecture.md
├── canvas.yaml              # Public, sanitized SuperPlane canvas (import this)
├── canvas.private.yaml      # Local working canvas (gitignored; not for public use)
├── .env.example
├── .gitignore
├── docs/                    # Extra notes / diagrams you add later
├── examples/
│   └── sample-slack-prompt.md
└── screenshots/
    └── README.md            # Guidance for demo screenshots
```

---

## Setup prerequisites

Before importing the canvas:

1. A SuperPlane workspace with permission to create canvases and secrets
2. A Slack workspace where you can install / connect a Slack app
3. A GitHub repository that will host `index.html` (empty or existing)
4. A Render service connected to that GitHub repository
5. An OpenRouter account with access to `openai/gpt-5` (or change the `MODEL` literal in Generate Site)
6. API tokens listed under [Required secrets](#required-secrets)

Replace placeholders in `canvas.yaml` after import (or before):

| Placeholder | Replace with |
|---|---|
| `YOUR_CANVAS_ID` | SuperPlane-assigned canvas ID (or leave for import to assign) |
| `YOUR_GITHUB_USERNAME/YOUR_REPOSITORY` | Target GitHub repo |
| `YOUR_SLACK_CHANNEL_ID` | Slack channel ID for approval buttons |
| `YOUR_SLACK_CHANNEL_NAME` | Human-readable channel name (metadata) |
| `YOUR_RENDER_SERVICE_ID` | Render service ID (`srv-…`) |
| `YOUR_SUPERPLANE_INTEGRATION_ID` | Slack integration ID in SuperPlane |
| `YOUR_APP_SUBSCRIPTION_ID` | Slack app subscription ID for the mention trigger |

---

## Required secrets

Configure these as SuperPlane secrets (names must match). Values stay in SuperPlane—not in this repository.

| Secret name | Used for |
|---|---|
| `OPENROUTER_API_KEY` | OpenRouter Authorization bearer for GPT-5 generation |
| `GH_TOKEN` | Git clone/push (needs repo contents read/write) |
| `SLACK_BOT_TOKEN` | `chat.postMessage` acknowledgements and error notifications |
| `RENDER_API_KEY` | Render service/deploy API calls |

`.env.example` lists the same names for local documentation. Do not commit real values.

---

## How to import `canvas.yaml` into SuperPlane

1. Open your SuperPlane workspace.
2. Create or open a canvas and choose the import / upload canvas YAML option supported by your SuperPlane UI.
3. Select `canvas.yaml` from this repository.
4. After import, reconnect Slack integration bindings:
   - On Mention → your Slack app subscription / integration IDs
   - Approval Gate → your Slack channel + integration ID
5. Set secret references for `OPENROUTER_API_KEY`, `GH_TOKEN`, `SLACK_BOT_TOKEN`, and `RENDER_API_KEY`.
6. Replace GitHub default repo and Render service ID placeholders with your values.
7. Save and enable the canvas / trigger.

Exact menu labels may vary by SuperPlane UI version; the imported graph preserves nodes, edges, scripts, prompts, and memory configuration from this file.

---

## How to configure Slack, GitHub, OpenRouter, and Render

### Slack

1. Connect your Slack app / workspace in SuperPlane Integrations.
2. Ensure the bot can join the target channel and has permission to post messages and interactive buttons.
3. Set Approval Gate `channel` to your Slack channel ID (`YOUR_SLACK_CHANNEL_ID`).
4. Invite the bot to that channel.
5. Confirm `On Mention` is wired to the connected app subscription.

### GitHub

1. Create (or reuse) a repository for portfolio HTML.
2. Create a personal access token or fine-grained token with contents read/write on that repo.
3. Store it as SuperPlane secret `GH_TOKEN`.
4. Set `DEFAULT_REPO` on Push Preview and Push to GitHub to `owner/repo`.
5. Optional per-request override: include `repo:owner/name` in the Slack message; Clean Prompt extracts it.

### OpenRouter

1. Create an API key at [openrouter.ai](https://openrouter.ai).
2. Store it as SuperPlane secret `OPENROUTER_API_KEY`.
3. Confirm the Generate Site node’s `MODEL` value (`openai/gpt-5`) is available on your account, or change it.

### Render

1. Create a static site (or web service) that deploys from your GitHub repo’s production branch.
2. Note the service ID (`srv-…`).
3. Create a Render API key and store it as `RENDER_API_KEY`.
4. Replace `YOUR_RENDER_SERVICE_ID` in Get URL, Poll Deploy, Refresh Deploy, and related Slack error message URLs.

---

## How to run the automation

1. Enable the canvas in SuperPlane.
2. In the configured Slack channel, mention the bot with a portfolio brief, for example:

   ```text
   @PortfolioFactory Build a software engineer portfolio for Alex Rivera…
   ```

   See [examples/sample-slack-prompt.md](./examples/sample-slack-prompt.md) for a fuller example.

3. Wait for the acknowledgment (~1–3 minutes for generation depending on model latency).
4. Open the preview link from the approval message.
5. Click **Approve & Deploy** or **Reject**.
6. On approve, wait for the success message with the live Render URL.
7. To refine: reply **in the same thread** with changes (e.g. `@bot use a darker palette`). Memory retains prior thread context.

---

## Security considerations

- `canvas.private.yaml` is intentionally gitignored. Never publish workspace IDs, channel IDs, or real tokens.
- Secret **names** appear in `canvas.yaml`; secret **values** must live only in SuperPlane (or a private `.env` that stays untracked).
- `GH_TOKEN` is used over HTTPS as `x-access-token`; scripts scrub token substrings from error payloads before writing SuperPlane results.
- Prefer least-privilege tokens (single-repo GitHub access; scoped Render/Slack tokens).
- Approval Gate is a human control: production push only runs after Approve.
- Preview branches are force-pushed; treat them as disposable review artifacts, not protected production history.
- Do not commit `.env`, `secrets.yaml`, PEM/key files, or screenshots that expose tokens or private channel IDs.

---

## Future improvements

- Multi-page or multipage-site generation (beyond a single `index.html`)
- Design system / brand kit injection from a shared template
- Stronger HTML/CSS lint or accessibility checks before preview push
- Parallel preview environments (per-user Render preview services)
- Slack slash commands or modal forms for structured briefs
- Automatic screenshot capture of the preview for the approval message
- Rolling conversation summarization when thread history grows large
- Support for selecting models or temperature per request
- Audit log / admin dashboard of deployed portfolios from the `portfolios` namespace

---

## License

MIT — see [LICENSE](./LICENSE).
