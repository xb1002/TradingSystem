# 统一 USDT 永续交易所层需求说明

## 背景与目标
- 打造一套可复用的 Python 3.11+ 库，为多家中心化衍生品交易所（Binance、OKX、Bybit、Bitget、Gate 等）的 USDT 本位永续合约提供统一接入能力。
- 策略、订单路由、风控等上层模块仅依赖与交易所无关的抽象接口，具体适配层负责处理各交易所 REST/WebSocket 异构协议。
- 架构必须遵守 SOLID 原则，以便在无侵入现有核心逻辑的情况下增量支持新的交易所或新的交易工具类型。

## 项目范围
1. **行情接口**
   - WebSocket：提供 BBO、聚合成交、K 线、资金费率等实时订阅，支持批量订阅、自动重连、自动重放订阅、生命周期管理（`start()`/`stop()`）。
   - REST：提供历史 K 线、订单簿快照、资金费率历史等查询，带时间窗口、分页、限速与重试控制，并统一转换为强类型数据模型。
2. **交易与账户接口**
   - 统一订单生命周期管理：创建、撤销、查询在途订单，维持统一的状态机与错误转换。
   - 统一账户视图：返回仓位、资产、杠杆、保证金等信息，并进行最小化输入校验（价格步长、数量精度、保证金模式等）。
3. **连接与可靠性**
   - 提供可复用的 REST/WebSocket 客户端基类，内置鉴权、速率限制、重连、心跳、异常分级处理、延迟告警。
4. **可扩展性**
   - 通过 `IUSDTPerpExchange` 抽象组合市场数据、交易、账户接口，新增交易所仅需实现该接口并注册到工厂/配置中，即可被上层模块消费。

## 功能需求
### 行情
- WebSocket 订阅/退订需支持多交易对并行，回调应传递强类型模型；连接断开后自动重连并恢复订阅；提供统一日志与错误回调。
- REST 查询需支持 `start_time`、`end_time`、`limit` 等参数；当触达速率限制时采用可配置的指数退避或漏桶策略；所有响应需转换为统一模型并附带时间戳。

### 交易与账户
- 统一的枚举类型（`OrderSide`、`OrderType`、`TimeInForce` 等），适配器负责映射到具体交易所字符串。
- 输入校验：根据合约元数据（最小下单数量、价格步长、杠杆范围等）进行预验证，避免无效请求触发网络调用。
- 创建订单时支持客户端订单号确保幂等；异常统一映射为领域异常（如 `OrderRejected`, `RateLimitExceeded`）。
- 账户接口需返回持仓、余额、资金费率 PnL、杠杆等，并支持逐仓/全仓等模式信息。

### 配置与部署
- 使用结构化配置（如 Pydantic Settings）描述 API Key/Secret、Passphrase、REST & WS 端点、速率限制、代理、订阅交易对等。
- 支持依赖注入：可传入自定义 HTTP/WebSocket Session Factory、指标上报钩子、序列化策略，便于测试与运维集成。

## 非功能需求
- **性能**：围绕高频场景设计，使用 `asyncio` + `httpx`/`aiohttp`/`websockets`；尽量减少对象分配与序列化开销。
- **可靠性**：内置自动重连、心跳、抖动重试与状态监控；支持持久化最新序列号以便快速追平。
- **可观测性**：统一的结构化日志、指标、追踪上下文透传；重要事件可触发告警。
- **安全性**：标准化签名流程、密钥加密存储、NTP 同步；敏感日志脱敏。
- **测试**：适配器需具备单元测试（Mock HTTP/WS）；必要时提供对接沙盒环境的集成测试示例。
- **文档**：Markdown 文档覆盖接口说明、添加新交易所指南、运行维护手册。

## 数据与领域模型要求
- 使用 dataclasses 或 Pydantic 定义如下统一模型：
  - 标的：`Symbol`（base/quote）、`InstrumentType` 枚举（`USDT_PERP` 等）、`Instrument`（价格步长、数量步长、最小下单量、杠杆限制等）。
  - 行情：`BboData`、`DepthData`、`TradeData`、`KlineData`、`FundingData`。
  - 交易账户：`OrderSide`、`OrderType`、`TimeInForce`、`NewOrderRequest`、`CancelOrderRequest`、`Order`、`Trade`、`Position`、`Balance`。
- 所有模型需具备类型注解、基础校验（例如价格/数量为正）、可序列化能力。

## 接口与抽象
- 通过 Python `abc` 定义以下接口：
  - `IMarketDataStream`：提供 `subscribe_bbo`、`subscribe_trades`、`subscribe_kline`、`subscribe_funding_rate`、`start()`、`stop()` 等异步方法，回调签名明确。
  - `IMarketDataRest`：提供 `get_klines`、`get_orderbook`、`get_funding_history` 等异步查询接口。
  - `ITrading`：提供 `create_order`、`cancel_order`、`get_open_orders`、`get_positions`、`get_balances` 等。
  - `IUSDTPerpExchange`：组合上述接口，并扩展 USDT 永续特有的 `get_mark_price`、`get_index_price` 等辅助方法。
- 上层模块仅依赖这些接口，通过依赖注入获取具体实现。

## 错误处理与弹性
- 建立统一异常层次结构，区分网络故障、速率限制、业务拒单等场景。
- REST 调用遵循可配置的重试策略（最大次数、退避间隔、可重试错误码）；WebSocket 自动重连并恢复订阅。
- 在请求发起前进行输入校验，提前抛出领域异常，避免无意义的网络请求。

## 目录结构与模块职责
```
TradingSystem/
├── CODE_GUIDELINES.md            # 仓库级编码与 SOLID 指南。
├── README.md                     # 项目简介。
├── docs/
│   └── UNIFIED_EXCHANGE_REQUIREMENTS.md  # 本需求文档。
├── exchanges/
│   ├── base/
│   │   ├── interfaces.py         # 市场数据 / 交易 / 账户抽象接口及组合接口。
│   │   ├── models.py             # 统一实体模型与枚举定义。
│   │   ├── client_ws.py          # 可复用的异步 WebSocket 客户端基类，含重连与订阅管理。
│   │   └── client_rest.py        # 异步 REST 客户端基类，封装鉴权、限速、重试逻辑。
│   ├── binance/
│   │   └── usdt_perp_exchange.py # Binance USDT 永续实现，演示接口落地。
│   ├── okx/
│   │   └── usdt_perp_exchange.py # OKX 实现（后续扩展）。
│   └── ...
├── core/
│   ├── config.py                 # 结构化配置与依赖注入入口。
│   ├── router.py                 # 工厂 / 注册中心，返回指定交易所实例。
│   └── exceptions.py             # 领域异常定义。
└── tests/
    ├── test_interfaces.py        # 验证接口契约与默认实现。
    └── test_binance_usdt_perp_exchange.py # Binance 适配器的单元 / 集成测试。
```

## 下一步
1. 按照本文档完成 `exchanges/base/models.py` 与 `interfaces.py` 的模型与接口实现。
2. 实现 REST/WebSocket 客户端基类并嵌入重连、限速、指标钩子。
3. 基于上述基类实现 Binance USDT 永续适配器，覆盖行情订阅、K 线查询、下单、查询余额等核心能力。
4. 在 `core/router.py` 提供工厂函数，结合配置动态加载交易所实现，并编写测试验证扩展性。
