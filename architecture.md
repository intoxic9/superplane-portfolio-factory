# Portfolio Factory Architecture

Flow of the SuperPlane canvas in `canvas.yaml`, including success path and error branches.

```mermaid
flowchart TD
  A[Slack mention<br/>On Mention] --> B[Acknowledge request<br/>Ack Start]
  B --> C[Clean prompt<br/>Clean Prompt]
  C -->|failed| E1[Notify Error]
  C -->|passed| D[Save thread message<br/>Save Thread Message]
  D --> E[Load thread history<br/>Load Thread Context]
  E -->|found / notFound| F[Build accumulated prompt<br/>Build Prompt]
  F -->|failed| E1
  F -->|passed| G[Call GPT-5 through OpenRouter<br/>Generate Site]
  G -->|failed| E1
  G -->|passed| H{Check Need Info}

  H -->|NEED_MORE_INFO| I[Ask More Info<br/>insufficient-information]
  I --> Z[On Error]

  H -->|ok| J{Validate HTML}
  J -->|invalid| K[Notify Validate Error<br/>validation-error]
  K --> Z

  J -->|valid| L[Push preview branch<br/>Push Preview]
  L -->|failed| M[Notify Preview Error<br/>GitHub-error]
  M --> Z

  L -->|passed| N[Slack approval gate<br/>Approval Gate]
  N -->|timeout| O[Notify Approval Timeout<br/>timeout]
  O --> Z

  N -->|received| P{Check Approval}
  P -->|reject| Q[Notify Rejected<br/>rejection]
  Q --> Z

  P -->|approve| R[Push approved HTML to main<br/>Push to GitHub]
  R -->|failed| S[Notify Push Error<br/>GitHub-error]
  S --> Z

  R -->|passed| T[Wait for Render deployment<br/>Wait for Deploy loop]
  T -->|next| U[Poll Wait 15s]
  U --> V[Poll Deploy]
  V -->|failure| W[Notify Render Error<br/>deployment-error]
  W --> Z
  V -->|success| T

  T -->|done| X[Refresh Deploy]
  X -->|failure| W
  X -->|success| Y{Check Deploy Status}

  Y -->|not live| DE[Notify Deploy Error<br/>deployment-error]
  DE --> Z

  Y -->|live| GU[Get URL]
  GU -->|failure| W
  GU -->|success| VL[Verify live website<br/>Verify Live]

  VL -->|failure| VE[Notify Verify Error<br/>deployment-error]
  VE --> Z

  VL -->|success| SP[Save portfolio metadata<br/>Save Portfolio]
  SP --> NS[Notify user in Slack<br/>Notify Success]
  NS --> Z

  E1 --> Z
```

## Node map (happy path)

| Step | Canvas node |
|---|---|
| Trigger | On Mention |
| Ack | Ack Start |
| Normalize Slack text | Clean Prompt |
| Memory write | Save Thread Message |
| Memory read | Load Thread Context |
| Context → brief | Build Prompt |
| LLM | Generate Site |
| Info gate | Check Need Info |
| HTML gate | Validate HTML |
| Preview | Push Preview |
| Human gate | Approval Gate → Check Approval |
| Production | Push to GitHub |
| Deploy wait | Wait for Deploy → Poll Wait → Poll Deploy → Refresh Deploy |
| Status gate | Check Deploy Status |
| Live URL | Get URL → Verify Live |
| Persist | Save Portfolio |
| Done | Notify Success |

## Error branches

| Branch | Trigger | Node(s) |
|---|---|---|
| insufficient-information | Model returns `NEED_MORE_INFO…` | Ask More Info |
| validation-error | HTML check fails | Notify Validate Error |
| GitHub-error | Preview or production push fails | Notify Preview Error / Notify Push Error |
| rejection | User clicks Reject | Notify Rejected |
| timeout | Approval Gate times out (10 min) | Notify Approval Timeout |
| deployment-error | Render API failure, non-live status, or verify ≠ 200 | Notify Render Error / Notify Deploy Error / Notify Verify Error |
| generic | Clean / Build / Generate failures | Notify Error |

All notification paths terminate at the **On Error** noop node so the run ends in a defined state.
