# 代码规范与SOLID设计指南

本文档为 TradingSystem 项目的通用代码规范。适用于仓库中的所有语言和目录，随着项目演进可以继续扩充。

## 目标
- 提升代码可读性与一致性，降低审查成本。
- 借助 SOLID 原则强化系统的可维护性与可扩展性。
- 通过文档、测试和工具链保证质量可追踪。

## 通用风格
1. **命名**
   - 使用有语义的英文单词，变量与函数采用 `lower_snake_case`，类与结构体使用 `UpperCamelCase`。
   - 常量统一为 `UPPER_SNAKE_CASE`，配置项需提供默认值和说明。
2. **格式化**
   - Python 代码遵循 PEP 8，使用 `ruff format` 或 `black`；TypeScript/JavaScript 使用 `eslint --fix`；Go 使用 `gofmt`。
   - 统一使用 4 个空格缩进，不混用 Tab。
3. **注释与文档**
   - 公共方法和模块必须包含描述功能、输入、输出和异常的文档注释。
   - 对复杂算法、业务规则提供行内注释，必要时附参考链接。
4. **错误处理**
   - 不吞掉异常，捕获后记录上下文再向上抛出或转换为领域错误。
   - 日志使用结构化格式，至少包含 trace id / 请求 id。

## SOLID 设计要求
1. **单一职责原则（Single Responsibility Principle）**
   - 每个模块只关注一类功能。例如：行情采集、策略决策、订单执行分别由不同的服务或类实现。
   - 将跨领域逻辑抽到独立的 utility/adapter，避免“上帝类”。
2. **开闭原则（Open/Closed Principle）**
   - 在添加新交易所适配、交易策略或风控时，应通过扩展接口而非修改核心引擎。
   - 通过插件化注册表、配置驱动或依赖注入实现可扩展性。
3. **里氏替换原则（Liskov Substitution Principle）**
   - 统一约定接口契约（例如 `OrderExecutor`、`MarketDataFeed`）。派生实现不可收紧输入条件或破坏返回约束，确保可替换性。
   - 在测试中对接口编写一致性的断言，避免实现偏差。
4. **接口隔离原则（Interface Segregation Principle）**
   - 拆分大接口，保证调用方只依赖其需要的方法。例如区分 `QuoteSubscriber` 与 `HistoricalDataProvider`。
   - 对外暴露的 API 分层（DTO/领域模型）避免泄漏内部细节。
5. **依赖倒置原则（Dependency Inversion Principle）**
   - 高层模块依赖抽象（接口、协议）而非具体实现，通过构造注入或服务定位器传递依赖。
   - 结合配置文件或容器（如 FastAPI Depends、Spring Bean）完成装配，使得替换实现（模拟/真实）只需修改绑定。

## 测试与质量
- 所有新功能需提供单元测试，并覆盖关键路径；策略或交易逻辑需提供回测用例。
- PR 必须通过静态检查（例如 `ruff check`, `eslint`, `go vet`）。
- 对外行为变更需在 `CHANGELOG.md` 或 PR 描述中记录。

## 文档与示例
- 重要模块在 `docs/` 下提供架构图、时序图或示例脚本。
- SOLID 设计示例：
  - `MarketDataFeed` 接口提供 `subscribe()`、`unsubscribe()`；
  - `StrategyEngine` 接收 `SignalGenerator` 接口，允许不同策略实现；
  - `OrderExecutor` 根据接口定义执行真实或模拟订单。

遵守以上规范与 SOLID 原则可以确保 TradingSystem 在扩展新资产、交易所与策略时保持稳定和高可维护性。
