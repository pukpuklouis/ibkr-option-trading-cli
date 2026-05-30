# IBKR Option Trading CLI — Implementation Plan

> Pure Rust option trading CLI using `ibapi` (TWS-native) + `ratatui` TUI.
> MVP: option chain quotes, real-time OPRA-level pricing, order placement.

## Module Dependency Graph

```
main.rs
  ├── cli.rs          (clap: subcommand parsing → dispatch)
  ├── config.rs       (TOML config → AppConfig)
  ├── ibkr_client.rs  (ibapi::Client wrapper → IBRClient)
  ├── models.rs       (shared types: OptionContract, Quote, Order, etc.)
  ├── app.rs          (AppState, InputMode enum)
  ├── error.rs        (AppError enum, eyre integration)
  └── ui/
      ├── mod.rs      (TuiApp: main loop, Layout, render dispatch)
      ├── chain_table.rs  (OptionChain widget: rows, TableState, Scrollbar)
      ├── order_form.rs   (OrderEntry widget: fields, validation)
      ├── status_bar.rs   (StatusBar widget: connection, account, P&L)
      └── events.rs       (EventReader: crossterm→mpsc→tokio::select!)
```

**Data flow:**
```
IB Gateway ↔ ibapi socket ↔ IBRClient (tokio task)
                                │
                     mpsc::channel<MarketEvent>
                                │
                         AppState (Arc<RwLock>)
                                │
                 ┌──────────────┼──────────────┐
              EventReader   MarketData     OrderStatus
                 │               │              │
                 └─────── tokio::select! ───────┘
                               │
                         terminal.draw() @ 30fps
```

## Phase Breakdown

### Phase 1: Scaffold — Project Setup & CLI Skeleton

**Goal:** `cargo init`, dependency wiring, `--help` works, module structure in place.

**Tasks:**

1. **`cargo init` + Cargo.toml** — Add all dependencies:
   ```toml
   [dependencies]
   ibapi = "3.0"
   ratatui = "0.29"
   crossterm = "0.28"
   tokio = { version = "1.42", features = ["full"] }
   clap = { version = "4.5", features = ["derive"] }
   serde = { version = "1", features = ["derive"] }
   config = "0.14"
   color-eyre = "0.6"
   chrono = { version = "0.4", features = ["serde"] }
   tui-textarea = "0.7"
   ```

2. **`config.rs`** — Structs + serde deserialize for `config.toml`
   - `AppConfig { connection: ConnectionConfig { host, port, client_id }, account, display }`

3. **`cli.rs`** — clap derive enum:
   ```rust
   #[derive(Parser)]
   enum Command {
       Tui,                          // Interactive TUI mode (default)
       Chain { symbol, expiry, .. }, // Headless: fetch option chain
       Watch { contract, fields },   // Headless: stream quotes
       Trade { symbol, expiry, .. }, // Headless: place order
       Positions,                    // Headless: list positions
       Status,                       // Headless: account summary
   }
   ```

4. **`models.rs`** — Core data types:
   - `OptionContract { symbol, expiry, strike, right, multiplier, exchange }`
   - `Quote { bid_price, bid_size, ask_price, ask_size, timestamp }`
   - `OptionRow { strike, call_quote, put_quote, call_greeks, put_greeks }`
   - `Order { id, contract, action, quantity, order_type, price }`
   - `MarketEvent { Tick | OrderUpdate | AccountUpdate | Error }`

5. **`error.rs`** — Wraps `ibapi::Error`, config errors, TUI errors into `color-eyre`

6. **`main.rs`** — `parse()` → dispatch to `run_tui()` or `run_headless()`

**Files created:** 6 (`main.rs`, `config.rs`, `cli.rs`, `models.rs`, `error.rs`, `Cargo.toml`)

---

### Phase 2: Core Connection + Headless Mode

**Goal:** Connect to TWS/Gateway, implement `chain`, `watch`, `trade`, `positions`, `status` subcommands.

**Tasks:**

1. **`ibkr_client.rs`** — Wrapper around `ibapi::Client`:
   - `connect(config) -> Result`
   - `disconnect() -> Result`
   - `fetch_option_chain(symbol, exchange, sec_type) -> Vec<OptionRow>`
   - `subscribe_quotes(contracts) -> mpsc::Receiver<Quote>` (tick-by-tick bid/ask)
   - `place_order(contract, action, qty, order_type, price) -> Result<OrderId>`
   - `subscribe_order_updates() -> mpsc::Receiver<OrderStatus>`
   - `get_account_summary() -> AccountSummary`
   - `get_positions() -> Vec<Position>`
   - `calculate_greeks(contract, underlying_price, iv, expiry) -> Greeks`
   - Auto-reconnect with exponential backoff

2. **Headless `chain`** — Fetch option chain, format as table (using `tabled` or manual), print to stdout. Support `--json` flag for machine output.

3. **Headless `watch`** — Subscribe to tick-by-tick for one contract, print stream to stdout (`bid|ask|delta|iv`). Optional `--csv` output.

4. **Headless `trade`** — Validate inputs, `place_order()`, print order ID + status.

5. **Headless `positions` / `status`** — Fetch and display account data.

**Files created/edited:** `ibkr_client.rs` (new), `main.rs` (headless dispatch), `cli.rs` (argument refinement)

---

### Phase 3: TUI Foundation

**Goal:** Terminal raw mode, event loop, render throttling, 5-zone layout.

**Tasks:**

1. **`events.rs`** — `EventReader` tokio task:
   - `crossterm::event::read()` in spawned thread → `mpsc::Sender<KeyEvent>`
   - Maps keys: Up, Down, Left, Right, Enter, Esc, Tab, Char('/'), Char('q'), Char('r')

2. **`ui/mod.rs`** — `TuiApp` struct:
   - `InputMode` enum: `Normal | EditingOrder { field, buffer }`
   - `select!` main loop:
     ```rust
     loop {
         select! {
             Some(event) = event_rx.recv() => handle_event(event),
             Some(data)  = market_rx.recv() => app_state.update(data),
             Some(order) = order_rx.recv()  => app_state.update_order(order),
             _ = ticker.tick()              => terminal.draw(|f| render(f, &app))?,
         }
     }
     ```
   - `render()`: 5-zone `ratatui::Layout`

3. **Layout wiring:** Top Bar (3%) → Chain Table (55%) → Order Form (25%) → Status Bar (5%) → Activity Log (2%)

4. **Panic safety:** `color-eyre` hook restores terminal on panic

**Files created:** `events.rs`, `ui/mod.rs`

---

### Phase 4: Option Chain Table

**Goal:** Scrollable table showing calls || strike || puts with color coding.

**Tasks:**

1. **`ui/chain_table.rs`** — `OptionChainWidget`:
   - Uses `ratatui::Table` with custom `Cell` rendering
   - Each row: `Call Bid | Call Ask | Call Delta || Strike || Put Delta | Put Ask | Put Bid`
   - `TableState` handles scrolling
   - `Scrollbar` widget for large chains (500+ rows)
   - ATM row highlighted with distinctive background
   - Bid/ask color-coded: green for bid, red for ask
   - Expiry tab bar at top (Left/Right to switch)

2. **Keybindings:**
   - `Up/Down` → `TableState::scroll()`
   - `Left/Right` → Switch expiry
   - `Enter` → Select row → enter Order Entry mode with pre-filled contract
   - `/` → Filter mode (by delta range, moneyness, DTE)

3. **Performance:** Pre-render rows into `Vec<Row>`, benchmark against 500+ strikes — target <33ms per draw

**Files created:** `ui/chain_table.rs`

---

### Phase 5: Real-Time Quotes

**Goal:** Live bid/ask updates flow into chain table + price panel.

**Tasks:**

1. Wire `ibkr_client.subscribe_quotes()` → `mpsc::channel<Quote>` → `AppState`
2. `AppState` stores latest quotes per contract in `HashMap<ContractId, Quote>`
3. Chain table reads quotes from `AppState` snapshot during render
4. Option contracts that have no live quote show `--` or `Awaiting...`
5. Price changes flash briefly (1 frame highlight on update)

**Files edited:** `app.rs`, `chain_table.rs`, `ibkr_client.rs`

---

### Phase 6: Order Entry

**Goal:** Keyboard-driven order form, submit via `ibapi`, see status update.

**Tasks:**

1. **`ui/order_form.rs`** — `OrderEntryWidget`:
   - Fields: Action (Buy/Sell), Qty, Order Type (Market/Limit/Stop), Price (if limit/stop), TIF (Day/GTC)
   - `Tab` cycles fields, number keys edit, `Enter` submits
   - Shows selected contract info: `SPX 4000C | Bid 3.60 Ask 3.70`
   - Confirmation dialog before final submit (toggle with `o`)

2. **Submission flow:** Validate → `ibkr_client.place_order()` → show order ID + status in activity log

3. **Order status streaming:** Wire `order_update_stream()` → `AppState` → status bar shows last order status

**Files created:** `ui/order_form.rs`

---

### Phase 7: Account + Positions Display

**Goal:** Status bar shows real-time account info.

**Tasks:**

1. **`ui/status_bar.rs`** — `StatusBarWidget`:
   - Account: NetLiquidation, TotalCashValue (color-coded green/red)
   - Positions: count, delta exposure
   - Connection status: `Connected (7497)` vs `Disconnected`
   - Last order status: `#1042 Filled @3.60`

2. Background poll: `account_summary()` + `positions()` every 5 seconds

**Files created:** `ui/status_bar.rs`

---

### Phase 8: Polish

**Goal:** Error handling, config file, packaging, docs.

**Tasks:**

1. `config.example.toml` with annotated defaults
2. CLI help text polish with all examples
3. `color-eyre` panic hooks that fully restore terminal (important for TUI)
4. Graceful shutdown: disconnect IBKR on Ctrl+C / `q`
5. Connection retry: auto-reconnect with exponential backoff (1s, 2s, 4s, 8s, max 30s)
6. `--json` output for all headless commands
7. Documentation in `docs/` folder

---

## File Tree (Final)

```
ibkr-option-trading-cli/
├── Cargo.toml
├── config.example.toml
├── config.toml              (gitignored)
├── README.md
├── plan.md
├── .gitignore
├── src/
│   ├── main.rs              # Entry point, TUI/headless dispatcher
│   ├── app.rs               # AppState, InputMode, update logic
│   ├── cli.rs               # clap CLI definition
│   ├── config.rs            # TOML config structs + loader
│   ├── ibkr_client.rs       # ibapi::Client wrapper
│   ├── models.rs            # Data types
│   ├── error.rs             # Error types
│   └── ui/
│       ├── mod.rs           # TuiApp, main loop, Layout, render
│       ├── chain_table.rs   # Option chain table widget
│       ├── order_form.rs    # Order entry form widget
│       ├── status_bar.rs    # Status/account bar widget
│       └── events.rs        # Event reader + key mapping
└── docs/
    └── architecture.md      # Detailed design doc
```

## Validation Plan

### Unit Tests
- `ibkr_client`: connect/disconnect lifecycle, option chain parsing, order builder validation
- `models`: serde round-trip for all types, `OptionRow` ordering by strike
- `cli`: argument parsing for all subcommands, error on missing required args

### Integration Tests (Paper Account)
- `test_connect_paper()` — connect to local TWS paper, verify connection callback
- `test_fetch_chain()` — fetch SPX chain, verify >0 expirations and strikes
- `test_subscribe_quotes()` — subscribe to SPX option, verify tick received within 5s
- `test_place_market_order()` — place small market order, verify fill
- `test_account_summary()` — verify NetLiquidation > 0

### Performance Benchmarks
- `bench_chain_render()` — render 500-row table under 33ms
- `bench_tick_throughput()` — process 1000 ticks/sec without backlog

## Key Risks & Mitigations

| Risk | Mitigation |
|---|---|
| `ibapi` v3.0 protobuf bugs | Prototype vs paper first; pin v2.x fallback |
| No streaming Greeks helper | Manual generic tick type 104 subscription |
| No rate limiter in `ibapi` | Custom tokio-based limiter (50 msg/sec) |
| TWS/Gateway protocol changes | `ibapi` maintainer handles updates |
| Large chain TUI perf (500+ rows) | Benchmark early, `Vec` pre-allocation, limit visible |
