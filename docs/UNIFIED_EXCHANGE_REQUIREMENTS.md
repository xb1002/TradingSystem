# Unified Crypto Exchange Layer Requirements

## Background & Goals
- Provide a reusable Python 3.11+ library encapsulating unified access to USDT-margined perpetual instruments across major centralized crypto derivatives exchanges (Binance, OKX, Bybit, Bitget, Gate).
- Strategies, order routers, and risk modules interact exclusively with exchange-agnostic abstractions while the layer handles heterogeneous REST/WebSocket APIs.
- The design must follow SOLID principles so onboarding a new exchange or instrument type requires additive implementations only.

## Scope
1. **Market Data Interface**
   - Real-time WebSocket subscriptions for best bid/offer (BBO), aggregated trades, klines, and funding-rate streams.
   - REST retrieval of historical klines, depth snapshots, and funding history for backfills and reconciliation.
2. **Trading & Account Interface**
   - Unified order lifecycle management (create, cancel, query).
   - Unified account state retrieval (positions, balances, leverage constraints).
3. **Connection Management**
   - Shared WebSocket/REST client bases with retry, throttling awareness, and auto-resubscription after reconnects.
4. **Extensibility**
   - Per-exchange adapters implement a `IUSDTPerpExchange` contract composed from segregated interfaces (market data stream, market data REST, trading, account).
   - Central factory/registry instantiates adapters based on configuration.

## Functional Requirements
### Market Data
- **WebSocket**: Subscribe/unsubscribe to multiple symbols concurrently, deliver typed models via callbacks/async queues, enforce reconnection with exponential backoff, log anomalies, and expose lifecycle hooks (`start()`, `stop()`).
- **REST**: Support time-bounded historical queries with pagination/backoff, convert exchange-native payloads into canonical typed models, respect rate limits with configurable sleep/retry policies.

### Trading & Accounts
- Normalized enums for order side/type/time-in-force; adaptors translate to exchange-specific strings.
- Input validation for price/quantity/contract filters using instrument metadata.
- Provide idempotent order submission (client order id), consistent status mapping, and error translation into domain exceptions.
- Surfaces positions, balances, funding PnL, and leverage via strongly typed models.

### Configuration & Deployment
- Accept structured config (e.g., Pydantic settings) covering API keys, passphrases, endpoints (REST/WebSocket), rate limits, proxies, and instrument filters.
- Allow dependency injection for HTTP/WebSocket session factories to support custom transports, observability, or mocking in tests.

## Non-Functional Requirements
- **Performance**: Optimized for high-frequency trading (low allocation, async IO via `asyncio`, `httpx`, `aiohttp`, or `websockets`).
- **Reliability**: Auto-reconnect with jitter, heartbeat monitoring, resubscribe on drop, and pluggable persistent storage for last-seen sequence ids.
- **Observability**: Structured logging, metrics hooks, and tracing context propagation.
- **Security**: Secure credential handling, request signing per exchange, and clock synchronization.
- **Testing**: Unit tests for adapters using fixtures/mocks; integration tests leveraging sandbox endpoints where available.
- **Documentation**: Markdown references for interfaces, onboarding new exchanges, and operational runbooks.

## Data & Domain Model Requirements
- Canonical dataclasses/Pydantic models for instruments, market data (BBO, depth, trades, klines, funding), and account entities (orders, positions, balances, trades).
- Enumerations for instrument type, order side/type, time-in-force, and taker/maker sides.
- Models must be serializable and include validation (min/max tick sizes, quantity steps, etc.).

## Interface Requirements
- Use Python ABCs to define:
  - `IMarketDataStream`: async subscription methods, callback signatures, lifecycle control.
  - `IMarketDataRest`: async fetching methods returning typed lists/iterables.
  - `ITrading`: async order/position/balance operations.
  - `IUSDTPerpExchange`: composes above interfaces plus USDT-perp-specific helpers (mark/index price, leverage info).
- Interfaces remain exchange-agnostic and support dependency inversion for routers/strategies.

## Error Handling & Resilience
- Central exception hierarchy distinguishing transient vs fatal failures.
- Retry policies with configurable limits/backoff strategies for REST; WebSocket auto-resubscribe preserving subscription state.
- Validation errors raised before network calls when requests violate instrument constraints.

## Directory Structure & Module Responsibilities
```
TradingSystem/
├── CODE_GUIDELINES.md            # Repository-wide coding & SOLID guidance.
├── README.md                     # High-level project intro.
├── docs/
│   └── UNIFIED_EXCHANGE_REQUIREMENTS.md  # This requirements document.
├── exchanges/
│   ├── base/
│   │   ├── interfaces.py         # ABCs for market data/trading plus exchange contracts.
│   │   ├── models.py             # Typed dataclasses/Pydantic models shared by adapters.
│   │   ├── client_ws.py          # Reusable async WebSocket client with reconnect logic.
│   │   └── client_rest.py        # Reusable async REST client with auth/rate limiting helpers.
│   ├── binance/
│   │   └── usdt_perp_exchange.py # Binance implementation of IUSDTPerpExchange.
│   ├── okx/
│   │   └── usdt_perp_exchange.py # OKX implementation (future).
│   └── ...
├── core/
│   ├── config.py                 # Typed settings, API credentials, rate-limit configs.
│   ├── router.py                 # Factory/registry returning concrete exchange adapters.
│   └── exceptions.py             # Domain-specific exception hierarchy.
└── tests/
    ├── test_interfaces.py        # Ensures adapters respect ABC contracts via mocks.
    └── test_binance_usdt_perp_exchange.py # Behavioral tests for Binance adapter (mocked IO).
```

## Next Steps
1. Finalize the abstract models and interfaces (`exchanges/base/models.py`, `interfaces.py`).
2. Implement shared client bases for REST/WebSocket communication with resilience features.
3. Build the Binance USDT perpetual exchange adapter as reference implementation.
4. Provide core router factory and tests to validate extensibility.
