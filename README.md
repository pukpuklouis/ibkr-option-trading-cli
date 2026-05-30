# IBKR Option Trading CLI

A **pure Rust** interactive terminal (TUI) and headless CLI for option trading via Interactive Brokers. Built on the [`ibapi`](https://github.com/wboayue/rust-ibapi) crate for direct TWS/IB Gateway connectivity — no Python sidecars, no IPC overhead.

## Features

### MVP (Phase 1)
- **Option Chain** — View strikes, expirations, bid/ask for any underlying (SPX, SPY, AAPL, etc.)
- **Real-Time Quotes** — Stream OPRA-level T-quotes (bid/ask/size/exchange) via tick-by-tick subscription
- **Order Placement** — Market and limit orders for single-leg options with real-time order status
- **Account Overview** — Net liquidation, cash balance, margin requirements, open positions
- **Interactive TUI** — Keyboard-driven interface with scrollable chain table, order entry form, status bar
- **Headless CLI** — Same engine, accessible via subcommands for scripting and automation

### Phase 2 (Planned)
- Greeks display (on-demand computation in chain columns)
- Bracket orders (entry + take profit + stop loss)
- Live position tracker with P&L
- Rate limiter (IBKR-compliant 50 msg/sec)
- Multi-expiry tab switching
- Search/filter strikes by delta, moneyness, DTE

### Phase 3 (Planned)
- Multi-leg strategies (vertical spreads, iron condors, butterflies)
- Live Greeks streaming via generic tick type 104
- Historical data sparklines
- Price/IV/Greeks alerts
- Session recording

## Architecture

```
┌─────────────────────────────────────────────┐
│            CLI Binary (pure Rust)            │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │  CLI Layer (clap)                       │ │
│  │  Commands: chain, watch, trade, pos     │ │
│  └─────────────┬───────────────────────────┘ │
│  ┌─────────────▼───────────────────────────┐ │
│  │  TUI Layer (ratatui + crossterm)        │ │
│  │  - Option Chain Table                   │ │
│  │  - Price/Position Panels                │ │
│  │  - Order Entry Form                     │ │
│  └─────────────┬───────────────────────────┘ │
│  ┌─────────────▼───────────────────────────┐ │
│  │  Trading Engine (tokio async)           │ │
│  │  - Market Data Connector                │ │
│  │  - Order Manager                        │ │
│  │  - Account Monitor                      │ │
│  └─────────────┬───────────────────────────┘ │
│  ┌─────────────▼───────────────────────────┐ │
│  │  ibapi v3.0 Client (TWS-native socket)  │ │
│  │  - Option Chain Subscription            │ │
│  │  - Tick-by-Tick Bid/Ask Streaming       │ │
│  │  - Greeks Computation                   │ │
│  │  - Order Placement & Status             │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
            ┌──────────────────┐
            │  TWS / IB Gateway │
            │  (port 7497)      │
            └──────────────────┘
```

## Prerequisites

- **Rust** 1.75+ (install via [rustup](https://rustup.rs/))
- **TWS** or **IB Gateway** running locally
  - Paper trading: `127.0.0.1:7497`
  - Live trading: `127.0.0.1:7496`
  - IB Gateway: `127.0.0.1:4001` (live) / `4002` (paper)
- An Interactive Brokers account with market data subscriptions

## Quick Start

```bash
# Clone and build
git clone https://github.com/YOUR_USER/ibkr-option-trading-cli.git
cd ibkr-option-trading-cli
cargo build --release

# Configure
cp config.example.toml config.toml
# Edit config.toml with your TWS host, port, and client ID

# Interactive TUI
ibkr-tui

# Headless CLI
ibkr chain SPX --expiry 20250117 --strikes 10
ibkr watch SPX 4000C --fields bid,ask,delta
ibkr trade --symbol SPX --expiry 20250117 --strike 4000 --right C --qty 1 --action buy --type market
ibkr positions
ibkr status
```

## TUI Keybindings

| Key | Action |
|---|---|
| `↑`/`↓` | Scroll option chain |
| `←`/`→` | Switch expiry tab |
| `Enter` | Select row / Confirm order |
| `Tab` | Cycle order entry fields |
| `/` | Search/filter strikes |
| `r` | Refresh chain data |
| `q` | Quit |
| `Esc` | Cancel / Back to normal mode |

## Configuration

```toml
[connection]
host = "127.0.0.1"
port = 7497
client_id = 101

[account]
default_account = "DU1234567"

[display]
refresh_rate_ms = 33
color_scheme = "default"
```

## Project Structure

```
src/
├── main.rs              # Entry point, dispatches TUI vs headless
├── app.rs               # Application state and mode management
├── cli.rs               # clap CLI argument parsing
├── config.rs            # TOML configuration loading
├── ibkr_client.rs       # ibapi client wrapper
├── models.rs            # Data types (OptionContract, Quote, Order, etc.)
├── error.rs             # Error types and handling
└── ui/
    ├── mod.rs           # TUI main loop, layout, rendering
    ├── chain_table.rs   # Option chain table widget
    ├── order_form.rs    # Order entry form
    ├── status_bar.rs    # Connection/account status display
    └── events.rs        # Keyboard event handling
```

## License

MIT

## Inspiration

This project is inspired by [OpenAlice](https://github.com/TraderAlice/OpenAlice) —
an open-source AI trading agent (cloned at ~/Dev/OpenAlice). While OpenAlice is a
TypeScript/Node.js monorepo focusing on AI-driven multi-broker trading, our project
distills the IBKR option trading experience into a fast, native Rust TUI and CLI.

Key architectural patterns borrowed from OpenAlice:
- **Validated contract construction** — enforced required fields per security type
- **IBKR error classification** — numeric error codes → typed, actionable errors
- **Clean broker interface design** — modular separation of concerns

