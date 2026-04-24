# FxGuru SMC Hub EA

**Production MQL5 Expert Advisor for MetaTrader 5**
Bridges smart money concept (SMC) technical analysis with an external
Python AI signal engine via REST API — managing the full position
lifecycle through a four-state machine architecture on a live broker account.

> **Deployment Status:** LIVE — Headway MT5 (EURUSD · GBPUSD · USDJPY · XAUUSD)
> **Companion System:** [AI Signal Bot](https://github.com/kingkay000/ai-signal-bot)
> — the FastAPI/Python backend this EA communicates with

---

## Overview

Most retail Expert Advisors are stateless: they evaluate conditions on
every tick and act independently each time OnTick() fires. This creates
re-entry bugs, race conditions on open positions, and fragile error
handling when the broker connection drops or the external signal API
returns an unexpected response.

FxGuru SMC Hub EA solves this with a **state machine architecture** that
makes the EA's behaviour at any given tick a function of its current
state — not a fresh evaluation of all conditions from scratch. The EA
knows exactly what it was doing, what it is waiting for, and how to
recover if something goes wrong.

The EA does not generate signals internally. It is a **bridge and
execution layer**: it pushes structured OHLCV and position data to the
AI Signal Bot, receives a structured JSON trade signal in return, and
manages the resulting position through its full lifecycle — from entry
to final close — using a four-stage stop-loss migration system.

---

## Architecture

### The Four-State Machine
┌─────────────────────────────────────────────────────────────────┐
│                    OnTick() Entry Point                         │
└─────────────────────────┬───────────────────────────────────────┘
│
┌───────────▼───────────┐
│                       │
│       BOOT            │  Initial validation.
│                       │  Checks broker connection,
│  → Validates handles  │  initialises indicator
│  → Checks history     │  handles via CopyBuffer,
│  → Transitions to     │  confirms sufficient
│    POLLING            │  historical bars exist.
└───────────┬───────────┘
│
┌───────────▼───────────┐
│                       │
│      POLLING          │  Steady-state loop.
│                       │  Monitors open positions
│  → Checks positions   │  for stage transition
│  → Evaluates signal   │  triggers. Requests new
│    readiness          │  signal from AI bot when
│  → Routes to          │  no position is open and
│    DATA_PUSH          │  signal conditions are met.
└───────────┬───────────┘
│
┌───────────▼───────────┐
│                       │
│     DATA_PUSH         │  Signal acquisition.
│                       │  Serialises OHLCV candle
│  → Builds payload     │  data and current position
│  → POSTs to AI bot    │  context into JSON payload.
│  → Parses response    │  POSTs to the FastAPI
│  → Executes entry     │  signal endpoint. Parses
│    or returns to      │  structured JSON response.
│    POLLING            │  Executes entry order if
└───────────┬───────────┘  signal is actionable.
│
┌───────────▼───────────┐
│                       │  Fault isolation.
│   ERROR_RECOVERY      │  Triggered by HTTP errors,
│                       │  malformed API responses,
│  → Logs error code    │  broker execution failures,
│  → Backs off          │  or invalid signal fields.
│  → Resets to POLLING  │  Isolated from position state
│    after cooldown     │  — open trades are never 
└───────────────────────┘  touched during recovery.

**Why a state machine over linear execution:**
MT5 calls OnTick() on every price change — potentially hundreds of
times per minute. Without state management, every function evaluates
all conditions independently on every tick, creating race conditions
on open positions and re-triggering actions that should only fire
once per signal cycle. The state machine ensures each action executes
exactly once per state, in the correct sequence, with predictable
recovery behaviour.

---

### The Four-Stage Position Management System

Once a position is open, the EA manages its stop-loss through four
progressive stages. Transitions are triggered by price movement, time
thresholds, and volatility conditions — not fixed pip values.

STAGE 1 — BREAK-EVEN
Trigger:  Price moves N × ATR in trade direction
Action:   Stop-loss moved to entry price + spread buffer
Purpose:  Eliminate downside risk on confirmed momentum

STAGE 2 — ATR TRAIL
Trigger:  Price continues beyond the break-even trigger point
Action:   Stop-loss trails at a dynamic distance calculated
as M × ATR(period) from current price
Purpose:  Lock in profit proportional to current volatility
rather than a fixed pip distance

STAGE 3 — PARTIAL TAKE-PROFIT
Trigger:  Price reaches the first TP level
Action:   Closes a defined percentage of the position volume.
Remaining volume continues under ATR trail.
Purpose:  Secures realised profit while keeping exposure
to the full target move

STAGE 4 — STAGNATION EXIT
Trigger:  Position open for > N bars without stage progression
AND ATR has contracted below a minimum threshold
Action:   Full position close at market
Purpose:  Exits trades where momentum has demonstrably
failed — avoiding the slow bleed of a stagnating
position consuming margin and swap costs

Stage state is encoded into the **position comment** using the schema:

IR=1.08425|IV=0.85
│            └── IV: Initial volatility (ATR at entry × multiplier)
└── IR: Initial risk price level (entry-side reference point)

This encoding makes stage identification deterministic across MT5
restarts, connection drops, and VPS reboots — the EA can reconstruct
full position context from the comment string alone without relying
on in-memory state that would be lost on a terminal restart.

---

### Signal Request Payload

On every DATA_PUSH cycle the EA serialises and POSTs the following
JSON structure to the AI Signal Bot endpoint:

```json
{
  "symbol": "EURUSD",
  "timeframe": "H1",
  "candles": [
    {
      "time": 1714521600,
      "open": 1.07823,
      "high": 1.08156,
      "low": 1.07741,
      "close": 1.08042,
      "tick_volume": 14832
    }
  ],
  "current_position": {
    "open": true,
    "type": "BUY",
    "open_price": 1.07950,
    "stop_loss": 1.07950,
    "take_profit": 1.08450,
    "volume": 0.10,
    "comment": "IR=1.07650|IV=0.00305",
    "stage": 2
  },
  "account_context": {
    "balance": 0,
    "equity": 0,
    "free_margin": 0,
    "margin_level": 0
  }
}
```

> Account figures are sent as 0 in the sanitised open-source version.
> The live EA sends real account context — configure in
> `config/ea_settings.mqh`.

---

### Fundamental Analyst Filter

Before any signal request is made, the EA checks a three-mode filter
that gates execution based on current market conditions. The filter
reads a discrete bias value (-1 / 0 / +1) passed via the signal
response and cross-references it against:

| Source | Input | Output |
|---|---|---|
| Central bank bias | YAML config (updated manually) | Directional bias -1/0/+1 |
| DXY momentum | stooq.com price feed (via AI bot) | USD strength -1/0/+1 |
| Economic calendar | High-impact event window check | Block flag true/false |
| CNN Fear & Greed | Index level (via AI bot) | Sentiment bias -1/0/+1 |

**Filter modes:**

- `MODE_STRICT` — All four sources must align with trade direction.
  Signal blocked if any source disagrees.
- `MODE_STANDARD` — Three of four sources must align.
  Single disagreement is tolerated.
- `MODE_DISABLED` — Filter bypassed entirely.
  Technical signal executed regardless of fundamental context.

---

## Key Technical Decisions

**Position identification via SelectPositionByMagicAndSymbol()**
The EA uses a custom selection function rather than the standard
`PositionSelect(symbol)`. In a multi-EA, multi-symbol environment,
PositionSelect() selects by symbol alone — creating ambiguity when
multiple EAs trade the same instrument. SelectPositionByMagicAndSymbol()
combines magic number and symbol for unambiguous identification,
preventing one EA instance from acting on another's positions.

**CopyBuffer() with pre-created handles**
Indicator handles (ATR, moving averages) are created once in OnInit()
and stored as global variables. CopyBuffer() is called with these
stored handles on every relevant tick rather than creating handles
on-the-fly. Creating handles inside OnTick() generates a new indicator
calculation thread on each call — causing handle leaks, memory
accumulation, and erratic buffer values on fast-moving markets.

**HTTP 200/404 differentiation in error handling**
WebRequest() returns the HTTP response code as a return value.
The EA distinguishes between HTTP 200 (signal available), HTTP 404
(no signal this cycle — normal state), and all other codes (genuine
error). HTTP 404 transitions back to POLLING silently. All other
non-200 codes transition to ERROR_RECOVERY. This distinction prevents
the EA from treating "no signal available yet" as a system failure.

**Dual EA instance support**
Two instances of the EA can run simultaneously on the same terminal
across different symbols — each with its own magic number, state
machine instance, and independent position tracking. Shared resources
(account margin, terminal connection) are read-only. No cross-instance
state mutation is possible by design.

---

## Configuration

All configurable parameters are declared in `config/ea_settings.mqh`.
Copy `config/ea_settings.example.mqh`, rename it, and set your values
before compiling.

```mql5
// ── Signal Bot Connection ────────────────────────────────
#define SIGNAL_BOT_URL        "https://your-render-app.onrender.com"
#define SIGNAL_ENDPOINT       "/signal"
#define HTTP_TIMEOUT_MS       5000
#define HTTP_RETRY_LIMIT      3

// ── Position Sizing ──────────────────────────────────────
#define LOT_SIZE              0.10
#define MAGIC_NUMBER          20240001   // Unique per EA instance

// ── Stage Thresholds ────────────────────────────────────
#define BREAKEVEN_ATR_MULT    1.0        // ATR multiples to trigger BE
#define TRAIL_ATR_MULT        1.5        // Trail distance in ATR
#define PARTIAL_TP_PERCENT    50.0       // % of position to close at TP1
#define STAGNATION_BARS       48         // Bars before stagnation check
#define STAGNATION_ATR_MIN    0.0003     // Minimum ATR to stay in trade

// ── Fundamental Filter ───────────────────────────────────
#define FILTER_MODE           MODE_STANDARD   // STRICT / STANDARD / DISABLED
```

---

## Repository Structure
fxguru-smc-ea/
│
├── FxGuru_SMC_Hub_EA.mq5        # Main EA file — compile this
│
├── config/
│   ├── ea_settings.example.mqh  # Configuration template
│   └── ea_settings.mqh          # Your config (gitignored)
│
├── include/
│   ├── StateMachine.mqh         # Four-state machine implementation
│   ├── PositionManager.mqh      # Four-stage SL migration logic
│   ├── SignalRequester.mqh      # HTTP POST / response parser
│   ├── FundamentalFilter.mqh    # Three-mode filter logic
│   ├── CommentEncoder.mqh       # IR=|IV= comment schema
│   └── Utils.mqh                # SelectPositionByMagicAndSymbol()
│                                #   and shared helpers
│
├── docs/
│   ├── ARCHITECTURE.md          # State machine and stage diagrams
│   ├── SIGNAL_SCHEMA.md         # Full JSON request/response schema
│   └── STAGE_TRANSITIONS.md     # Stage trigger conditions and logic
│
├── .gitignore                   # Excludes ea_settings.mqh
├── LICENSE                      # MIT
└── README.md

---

## Companion System

This EA is the execution layer of a two-component trading system.
The signal intelligence layer lives in a separate repository:

**[kingkay000/ai-signal-bot](https://github.com/kingkay000/ai-signal-bot)**
FastAPI/Python backend deployed on Render. Receives the OHLCV payload
from this EA, applies SMC confluence scoring, routes to LLM for
structured signal analysis, and returns a JSON trade signal.

[MetaTrader 5]                    [Render Cloud]
│                                  │
│  FxGuru SMC Hub EA               │  AI Signal Bot
│  (this repository)               │  (companion repository)
│                                  │
│  ── HTTP POST /signal ────────►  │  SMC scoring engine
│                                  │  LLM signal analysis
│  ◄─── JSON trade signal ──────   │  Fundamental filter data
│                                  │
│  Executes entry / manages SL     │

---

## Deployment Notes

1. Clone this repo to your MT5 `Experts/` directory
2. Copy `config/ea_settings.example.mqh` → `config/ea_settings.mqh`
3. Set your Signal Bot URL and magic number
4. Compile `FxGuru_SMC_Hub_EA.mq5` in MetaEditor
5. Attach to chart (EURUSD H1 recommended for initial testing)
6. Ensure the AI Signal Bot is deployed and accessible at the
   configured URL before attaching the EA

**Platform:** MetaTrader 5 (build 3000+)
**Broker tested:** Exness (Raw Spread account)
**Symbols tested:** EURUSD · GBPUSD · USDJPY

---

## Security Notes

`ea_settings.mqh` is gitignored. It contains your broker endpoint
URL and any authentication headers. Never commit this file.

All account balance, equity, and margin figures in the signal payload
are set to 0 in this open-source release. The position structure,
candle data, and state information are fully functional.

---

## Licence

MIT — see `LICENSE`

Part of the ACE College International open-source infrastructure
portfolio. Built and maintained by the Principal & Lead Developer,
ACE College International, Lagos, Nigeria.

[github.com/kingkay000](https://github.com/kingkay000)
