# TUI Design Reference

> Inspired by a tastyworks-style option trading terminal.
> Reference image: `docs/screenshot-reference.png`

---

## Five-Zone Layout

```
┌──────────────────────────────────────────────────────────────────┐
│ TOP STATUS BAR: Symbol, Price, Expiry, Last Action, Status       │ ◄ 5%
├─────────────────────────────────┬────────────────────────────────┤
│                                 │ STRATEGIES PANEL               │
│ OPTION CHAIN TABLE              │ • Call Credit Spread           │ ◄ 50%
│ OI | Call Ltp | STRIKE | Put   │ • Put Credit Spread            │
│ ██ | 108.3    | 26200◄ | 65.8  │ •▶Put Debit Spread             │
│ █  |  65.8    | 26100◄ | 42.1  │ • Call Debit Spread            │
│    |  42.1    | 26000   | 33.5 │ • Iron Condor                  │
│    +1 BUY 26200 (green badge)  │ • Butterfly                    │
│    -1 SELL 26100 (red badge)   │                                │
├─────────────────────────────────┴────────────────────────────────┤
│ ANALYSIS PANEL (left)        │ PAYOFF GRAPH (right)              │ ◄ 30%
│ Active Legs:                 │  ┌──────────────────────────────┐ │
│   +1 BUY PE @108.3 26200    │  │  P&L vs Underlying           │ │
│   -1 SELL PE @65.8  26100   │  │  ──┐                   ┌──── │ │
│                              │  │    └──────────────────┘      │ │
│ Max Profit: ₹3,734          │  │  0 ── Spot ──────────────     │ │
│ Max Loss:   -₹2,765         │  │              ┌────────        │ │
│ Breakeven:  26157           │  │              │                │ │
└──────────────────────────────┴──┴──────────────────────────────┘
```

---

## Zone Details

### 1. Top Status Bar (5% of screen)

```
Nifty 50  26129.60  Exp:2026-01-06 │ Last: POS(26200/26100)
LIVE MARKET DATA            q:quit  ↑↓:nav  S:shortcuts
```

| Element | Description |
|---|---|
| Symbol | Market ticker + current underlying price |
| Expiry | Currently selected option expiry date |
| Last Action | Short summary of recent action (e.g., `POS(26200/26100)`) |
| Status Badge | `LIVE MARKET DATA` / `DELAYED` / `DISCONNECTED` with color coding |
| Keybinding Hints | Compact hints: `q` quit, `↑↓` navigate, `S` shortcuts |

### 2. Option Chain Table (50% — main zone)

The core data display, modeled after tastyworks:

```
CALL OI  Call LTP │ STRIKE │ Put LTP  PUT OI
   ████    108.3  │  26200◄│   65.8     ██████  ← Active: +1 BUY (green)
   ███      65.8  │  26100◄│   42.1     █████   ← Active: -1 SELL (red/orange)
   ██       42.1  │  26000 │   33.5     ███
   █        33.0  │  25900 │   25.8     ██
                   │  25800 │   18.2     █
```

| Column | Details |
|---|---|
| **OI Bar** (Call side) | Horizontal bar chart proportional to Open Interest, green color |
| **Call LTP** | Last traded price for the call option |
| **STRIKE** | Centered, bold. ATM strike has distinctive background highlight |
| **Put LTP** | Last traded price for the put option |
| **OI Bar** (Put side) | Horizontal bar chart proportional to Open Interest, red color |
| **Position Overlay** | Colored `+1 BUY` / `-1 SELL` badges on strikelines with existing positions |

Key UX patterns:
- Strike column visually separates calls (left) from puts (right)
- ATM (At-The-Money) row is highlighted — user's eye goes there first
- OI bars give instant visual read of liquidity concentration
- Active position badges let you see your current position in context

### 3. Strategies Panel (right column, ~25% width)

Quick-select menu of common multi-leg option strategies:

```
▶ Put Debit Spread
  Call Credit Spread
  Put Credit Spread
  Call Debit Spread
  Iron Condor
  Iron Butterfly
  Straddle
  Strangle
  Covered Call
  Collar
```

- Current selection highlighted with `▶` indicator
- Selecting a strategy auto-fills legs in the analysis panel
- User can then tweak strikes manually

### 4. Analysis Panel (bottom-left, ~15%)

```
Active Legs:
  +1 BUY  PE @108.3  Str: 26200
  -1 SELL PE @ 65.8  Str: 26100

Max Profit:  ₹3,734
Max Loss:   -₹2,765
Breakeven:   26157
Net Premium: ₹42.50
Delta:       0.35 (bearish)
```

| Metric | Description |
|---|---|
| **Active Legs** | Each leg: action (BUY/SELL), quantity, option type, premium paid/received, strike |
| **Max Profit** | Theoretical maximum profit for the selected strategy |
| **Max Loss** | Theoretical maximum loss for the selected strategy |
| **Breakeven** | Underlying price at which P&L = 0 |
| **Net Premium** | Net debit or credit for the spread |
| **Delta** | Net delta for the spread (directionality indication) |

### 5. Payoff Graph (bottom-right, ~15%)

```
        ┌──────────────────────────────┐
₹3,700 ┤                    ┌─────────┐
        │                    │         │
        │                  ┌─┘         │
  ₹0   ┤──────────────────┘           │
        │           Spot ▼             │
        │                        ┌─────┘
-₹3,000 ┤────────────────────────┘
        │                 26050         │
        └──────────────────────────────┘
```

- **X-axis:** Underlying price range
- **Y-axis:** P&L in currency units
- **Zero line:** Horizontal line at P&L = 0
- **P&L curve:** ASCII/Unicode line chart showing strategy payoff
- **Spot marker:** Vertical dashed line at current underlying price
- Profit region above zero is shaded green, loss below zero shaded red

---

## State Machine (Input Modes)

```
NORMAL MODE
  │
  ├── ↑/↓       → Scroll option chain (TableState)
  ├── ←/→       → Switch expiry date
  ├── Enter     → Select row → enter ORDER ENTRY mode
  ├── Tab       → Focus shift between panels
  ├── S         → Open keybinding help overlay
  ├── /         → Enter SEARCH mode (filter strikes)
  ├── 1-5       → Quick strategy select (1=Call Credit, 2=Put Credit, etc.)
  ├── r         → Refresh data
  └── q         → Quit (confirm if open orders)

ORDER ENTRY MODE
  ├── Tab       → Cycle fields (Action, Qty, Type, Price, TIF)
  ├── Number    → Edit field value
  ├── Enter     → Show confirmation dialog
  │   ├── Y     → Submit order
  │   └── N     → Cancel
  └── Esc       → Back to NORMAL MODE

STRATEGY MODE (Phase 2)
  ├── ↑/↓       → Scroll strategy list
  ├── Enter     → Select strategy, auto-fill legs
  ├── ←/→       → Adjust strikes for selected legs
  └── Esc       → Back to NORMAL MODE
```

---

## Color Scheme

| Element | Color | Context |
|---|---|---|
| Call LTP | Green (`#00FF00`) | Bullish color association |
| Put LTP | Red (`#FF4500`) | Bearish color association |
| ATM strike background | Cyan/Blue highlight | Visual anchor for the chain |
| Bid price | Green | Standard exchange convention |
| Ask price | Red | Standard exchange convention |
| OI Bar (Calls) | Light green | Volume/OI visualization |
| OI Bar (Puts) | Light red | Volume/OI visualization |
| BUY badge | Green background + white text | Position long indicator |
| SELL badge | Red/orange background | Position short indicator |
| Live status | Green badge | Connected to market data |
| Delayed status | Yellow badge | Delayed data |
| Disconnected | Red badge | No connection |
| Profit | Green | Positive P&L |
| Loss | Red | Negative P&L |
| Zero line | Dimmed white/gray | Baseline in payoff graph |

---

## Layout Evolution

### Phase 1 (MVP) — Minimal Layout

```
┌── Top Bar: Symbol, Price, Status ──────────────────────────────┐
├── Option Chain Table (full width): Call | Strike | Put ────────┤
│   Bid/Ask/Delta      Bid/Ask/Delta                              │
├── Order Entry Form ─────────────────────────────────────────────┤
├── Status Bar: Positions, P&L, Orders ───────────────────────────┤
└─────────────────────────────────────────────────────────────────┘
```

### Phase 2 — Add Analysis

```
┌── Top Bar ───────────────────────────────────────┐
├── Chain Table (70%)         │ Strategies (30%) ──┤
├── Analysis (50%)            │ Payoff Graph (50%) ┤
└─────────────────────────────┴────────────────────┘
```

### Phase 3 — Full Layout

```
┌── Top Bar ───────────────────────────────────────┐
├── Chain Table (70%)         │ Strategies (30%) ──┤
│  + OI bars, position badges                      │
├── Analysis (50%)            │ Payoff Graph (50%) ┤
│  + Greeks, IV, volume                            │
└─────────────────────────────┴────────────────────┘
```

---

## Ratatui Implementation Notes

### Widget Mapping

| UI Element | ratatui Widget | Notes |
|---|---|---|
| Top Bar | `Paragraph` with `Line::styled` spans | Composite using `Span` with different styles |
| Option Chain | `Table` with custom `Cell` rendering | Use `TableState` for scroll state, `Scrollbar` widget |
| OI Bars | `Gauge` widget | Horizontal progress bars, width = OI proportion |
| Position Badge | `Paragraph` with colored background | Rendered as overlay cell in the chain table row |
| Strategies Panel | `List` widget with `ListState` | Simple selectable list |
| Analysis Panel | `Paragraph` with formatted text | Key-value pairs with alignment |
| Payoff Graph | Custom `Canvas` / `Paragraph` drawing | Use Unicode block chars or ratatui's `Canvas` |
| Order Form | `Paragraph` + input handling | Custom state machine for field editing |
| Status Bar | `Paragraph` | Persistent bottom bar |
| Keybinding Help | `Popup` / `Clear` overlay | Modal window on top of current content |

### Rendering Performance

- Render at 30fps (33ms interval) via `tokio::time::interval`
- Market data updates arrive on separate channel, update `AppState` in-place
- `terminal.draw()` reads current `AppState` snapshot
- For 500+ strike rows: use `TableState` to limit visible rows, use `Vec::with_capacity` pre-allocation
- OI bars are pre-computed once per data refresh (not per frame)

### Terminal Sizing

- Minimum width: 120 characters (for 5-zone layout)
- Minimum height: 30 rows (for all zones)
- On smaller terminals: fall back to Phase 1 minimal layout
- Handle `Resize` events via `crossterm::event::Event::Resize`
