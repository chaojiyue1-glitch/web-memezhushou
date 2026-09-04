# Meme雷达链上核心 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建可在 Windows 本地运行的 Pons/Uniswap 新池扫描、基础评分、网页展示与系统通知版本。

**Architecture:** FastAPI 进程同时承载本地网页和后台异步任务；所有外部来源先转换成统一领域模型，再由分析服务评分并写入 SQLite。通知只读取已持久化评分结果，网页不直接调用外部 API。

**Tech Stack:** Python 3.12、FastAPI、Uvicorn、SQLAlchemy 2.x、Alembic、httpx、Web3.py、APScheduler、Jinja2、Pydantic Settings、pytest、pytest-asyncio、respx、ruff

**Spec:** `docs/superpowers/specs/2026-09-04-meme-radar-design.md`

## Global Constraints

- Windows 10/11，本地地址仅绑定 `127.0.0.1`。
- Robinhood Chain Mainnet Chain ID 固定为 `4663`。
- 不连接钱包、不签名、不交易、不记录仓位。
- 停机期间不补采数据。
- 默认最低流动性 10,000 美元、5分钟20笔交易、12位独立买家、热度线70、提醒风险上限49、每日提醒上限20。
- 原始数据保留7天。
- 数据未知时显示“未知/数据不足”，不以0替代。
- 所有外部请求经适配器；合约地址集中配置。

---

### Task 1: 可安装的项目骨架与配置

**Files:**
- Create: `pyproject.toml`
- Create: `src/meme_radar/__init__.py`
- Create: `src/meme_radar/config.py`
- Create: `src/meme_radar/app.py`
- Create: `tests/test_config.py`
- Modify: `.env.example`

**Interfaces:**
- Produces: `Settings`, `create_app(settings: Settings | None = None) -> FastAPI`

- [ ] **Step 1: 写配置失败测试**

```python
def test_defaults_bind_only_to_localhost(monkeypatch):
    monkeypatch.delenv("APP_HOST", raising=False)
    from meme_radar.config import Settings
    settings = Settings(_env_file=None)
    assert settings.app_host == "127.0.0.1"
    assert settings.robinhood_chain_id == 4663
    assert settings.x_enabled is False
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/test_config.py -v`  
Expected: FAIL because `meme_radar.config` does not exist.

- [ ] **Step 3: 创建最小配置与健康端点**

```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")
    app_host: str = "127.0.0.1"
    app_port: int = 8765
    robinhood_chain_id: int = 4663
    x_enabled: bool = False

def create_app(settings: Settings | None = None) -> FastAPI:
    app = FastAPI(title="Meme雷达")
    app.state.settings = settings or Settings()
    @app.get("/health")
    async def health():
        return {"status": "ok"}
    return app
```

- [ ] **Step 4: 运行测试与静态检查**

Run: `pytest tests/test_config.py -v && ruff check .`  
Expected: PASS.

- [ ] **Step 5: 提交**

```bash
git add pyproject.toml src tests/test_config.py .env.example
git commit -m "build: add local FastAPI project skeleton"
```

### Task 2: 统一领域模型与 SQLite 迁移

**Files:**
- Create: `src/meme_radar/domain/models.py`
- Create: `src/meme_radar/storage/database.py`
- Create: `src/meme_radar/storage/orm.py`
- Create: `src/meme_radar/storage/repositories.py`
- Create: `alembic.ini`
- Create: `migrations/env.py`
- Create: `migrations/versions/0001_initial.py`
- Create: `tests/storage/test_repositories.py`

**Interfaces:**
- Produces: `TokenIdentity`, `PoolObservation`, `ScoreResult`, `TokenRepository.upsert_pool()`, `TokenRepository.save_score()`

- [ ] **Step 1: 写合约地址去重失败测试**

```python
async def test_upsert_pool_deduplicates_chain_and_address(repo, observation):
    first = await repo.upsert_pool(observation)
    second = await repo.upsert_pool(observation)
    assert first.id == second.id
    assert await repo.count_tokens() == 1
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/storage/test_repositories.py -v`  
Expected: FAIL because repository classes do not exist.

- [ ] **Step 3: 实现模型和仓储**

```python
@dataclass(frozen=True)
class TokenIdentity:
    chain_id: int
    address: str
    symbol: str | None
    name: str | None

class TokenRepository:
    async def upsert_pool(self, item: PoolObservation) -> TokenRecord:
        token_address = Web3.to_checksum_address(item.token.address)
        pool_address = Web3.to_checksum_address(item.pool_address)
        stmt = select(TokenRecord).join(PoolRecord).where(
            TokenRecord.chain_id == item.token.chain_id,
            TokenRecord.address == token_address,
            PoolRecord.address == pool_address,
        )
        existing = await self.session.scalar(stmt)
        if existing is not None:
            return existing
        token = TokenRecord(
            chain_id=item.token.chain_id,
            address=token_address,
            symbol=item.token.symbol,
            name=item.token.name,
        )
        token.pools.append(PoolRecord(address=pool_address, source=item.source))
        self.session.add(token)
        await self.session.flush()
        return token
```

Migration must create the entities listed in spec section 11 and unique indexes on token/pool/event identifiers.

- [ ] **Step 4: 验证迁移和仓储**

Run: `alembic upgrade head && pytest tests/storage/test_repositories.py -v`  
Expected: PASS and a temporary SQLite database contains revision `0001`.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/domain src/meme_radar/storage migrations alembic.ini tests/storage
git commit -m "feat: add domain models and SQLite storage"
```

### Task 3: 数据源协议与健康状态

**Files:**
- Create: `src/meme_radar/sources/base.py`
- Create: `src/meme_radar/sources/health.py`
- Create: `tests/sources/test_source_contract.py`

**Interfaces:**
- Produces: `PoolSource.stream(from_block: int) -> AsyncIterator[PoolEvent]`, `MarketSource.snapshot(token, pool) -> MarketSnapshot`, `SourceHealthRegistry`

- [ ] **Step 1: 写适配器契约失败测试**

```python
async def test_source_failure_is_recorded_without_fake_values(registry, failing_source):
    result = await registry.capture("pons", failing_source.fetch)
    assert result is None
    assert registry.get("pons").status == "error"
    assert registry.get("pons").last_error
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/sources/test_source_contract.py -v`  
Expected: FAIL because source protocols are missing.

- [ ] **Step 3: 实现协议与隔离状态**

Define typed protocols, UTC timestamps, error categories `timeout`, `rate_limited`, `invalid_data`, `unavailable`, and redact exception messages before persistence.

- [ ] **Step 4: 运行测试**

Run: `pytest tests/sources/test_source_contract.py -v`  
Expected: PASS.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/sources tests/sources
git commit -m "feat: define external source contracts"
```

### Task 4: Pons 新池事件适配器

**Files:**
- Create: `src/meme_radar/sources/pons.py`
- Create: `tests/fixtures/pons_token_launched.json`
- Create: `tests/sources/test_pons.py`

**Interfaces:**
- Consumes: `PoolSource`, configured Pons factory address
- Produces: `PonsSource.decode_log(log: dict) -> PoolEvent`

- [ ] **Step 1: 保存已核验事件样本并写失败测试**

```python
def test_decodes_token_launched_fixture(pons_source, fixture):
    event = pons_source.decode_log(fixture)
    assert event.chain_id == 4663
    assert event.source == "pons"
    assert event.token_address.startswith("0x")
    assert event.pool_address.startswith("0x")
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/sources/test_pons.py -v`  
Expected: FAIL because decoder is missing.

- [ ] **Step 3: 核验并实现 Pons 适配器**

Use the active factory and `TokenLaunched` ABI from `https://docs.ponsfamily.com/`; record the source URL and verification date in `src/meme_radar/sources/contracts.py`. Reject logs whose address does not match configured factory.

- [ ] **Step 4: 运行单元测试**

Run: `pytest tests/sources/test_pons.py -v`  
Expected: PASS without a live RPC call.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/sources/pons.py src/meme_radar/sources/contracts.py tests
git commit -m "feat: decode Pons token launch events"
```

### Task 5: Uniswap V3 新池适配器

**Files:**
- Create: `src/meme_radar/sources/uniswap.py`
- Create: `tests/fixtures/uniswap_pool_created.json`
- Create: `tests/sources/test_uniswap.py`

**Interfaces:**
- Consumes: official Robinhood Chain Uniswap V3 factory address
- Produces: `UniswapV3Source.decode_log(log: dict) -> PoolEvent | None`

- [ ] **Step 1: 写非WETH池过滤测试**

```python
def test_accepts_pool_with_configured_quote_asset(source, weth_pool_log):
    event = source.decode_log(weth_pool_log)
    assert event is not None
    assert event.quote_symbol == "WETH"
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/sources/test_uniswap.py -v`  
Expected: FAIL because adapter is missing.

- [ ] **Step 3: 核验工厂地址并实现事件解析**

Verify factory address against Robinhood/Uniswap official material or verified Blockscout contract. Support configured quote assets; normalize token ordering so the discovered non-quote token is the candidate.

- [ ] **Step 4: 运行测试**

Run: `pytest tests/sources/test_uniswap.py -v`  
Expected: PASS for accepted, ignored, duplicate, and malformed logs.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/sources/uniswap.py src/meme_radar/sources/contracts.py tests
git commit -m "feat: detect Uniswap V3 pools on Robinhood Chain"
```

### Task 6: 市场、持仓与基础合约数据补充

**Files:**
- Create: `src/meme_radar/sources/blockscout.py`
- Create: `src/meme_radar/sources/market.py`
- Create: `src/meme_radar/analysis/contract_checks.py`
- Create: `tests/sources/test_enrichment.py`

**Interfaces:**
- Produces: `TokenEnricher.enrich(event: PoolEvent) -> AnalysisInput`

- [ ] **Step 1: 写缺失数据测试**

```python
async def test_missing_holder_data_remains_unknown(enricher):
    result = await enricher.enrich(sample_event())
    assert result.top10_holder_pct is None
    assert "holder_data" in result.unknown_fields
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/sources/test_enrichment.py -v`  
Expected: FAIL because enricher is missing.

- [ ] **Step 3: 实现缓存、限流与标准化**

Exclude pool, burn, zero and known locker addresses from holder concentration. Store `market_cap_usd` and `fdv_usd` separately. A successful nonzero quoter response sets `sell_quote_available=True`; never label it as guaranteed sellability.

- [ ] **Step 4: 运行测试**

Run: `pytest tests/sources/test_enrichment.py -v`  
Expected: PASS for success, partial data, 429 and timeout fixtures.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/sources src/meme_radar/analysis tests/sources
git commit -m "feat: enrich pools with market and risk inputs"
```

### Task 7: 可解释的热度与风险评分

**Files:**
- Create: `src/meme_radar/analysis/scoring.py`
- Create: `src/meme_radar/analysis/classification.py`
- Create: `tests/analysis/test_scoring.py`

**Interfaces:**
- Produces: `score_heat(input) -> ScoreBreakdown`, `score_risk(input) -> ScoreBreakdown`, `classify(input, heat, risk) -> Classification`

- [ ] **Step 1: 写边界失败测试**

```python
def test_focus_requires_all_gates(sample_input):
    heat = ScoreBreakdown(total=70, reasons=[])
    risk = ScoreBreakdown(total=49, reasons=[])
    assert classify(sample_input, heat, risk).label == "focus"
    sample_input.liquidity_usd = Decimal("9999")
    assert classify(sample_input, heat, risk).label == "observe"
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/analysis/test_scoring.py -v`  
Expected: FAIL because scoring functions are missing.

- [ ] **Step 3: 实现权重、门槛和中文原因**

Implement exact weights from spec section 8. Every contribution must return metric name, points, observed value and Chinese reason. Unknown required data prevents `focus`; hard-risk conditions force `high_risk`.

- [ ] **Step 4: 运行评分测试**

Run: `pytest tests/analysis/test_scoring.py -v`  
Expected: PASS at 69/70 heat, 49/50/70 risk, age and liquidity boundaries.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/analysis tests/analysis
git commit -m "feat: add explainable heat and risk scoring"
```

### Task 8: 扫描编排、去重与7天清理

**Files:**
- Create: `src/meme_radar/scanner/service.py`
- Create: `src/meme_radar/jobs/scheduler.py`
- Create: `src/meme_radar/storage/retention.py`
- Create: `tests/scanner/test_service.py`
- Create: `tests/storage/test_retention.py`

**Interfaces:**
- Produces: `ScannerService.handle_event(event)`, `RetentionService.purge(now) -> PurgeResult`

- [ ] **Step 1: 写重复事件与保留测试**

```python
async def test_same_event_is_analyzed_once(scanner, event):
    await scanner.handle_event(event)
    await scanner.handle_event(event)
    assert scanner.analyzer.calls == 1
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/scanner/test_service.py tests/storage/test_retention.py -v`  
Expected: FAIL because services are missing.

- [ ] **Step 3: 实现当前时刻启动和清理**

Persist the startup block and begin at that block; do not request older blocks. Purge raw snapshots older than seven days while preserving settings, KOL records, alerts and alerted-token score summaries.

- [ ] **Step 4: 运行测试**

Run: `pytest tests/scanner/test_service.py tests/storage/test_retention.py -v`  
Expected: PASS.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/scanner src/meme_radar/jobs src/meme_radar/storage tests
git commit -m "feat: orchestrate scanning and retention"
```

### Task 9: Windows通知、去重和每日限额

**Files:**
- Create: `src/meme_radar/notifications/base.py`
- Create: `src/meme_radar/notifications/windows.py`
- Create: `src/meme_radar/notifications/service.py`
- Create: `tests/notifications/test_service.py`

**Interfaces:**
- Produces: `NotificationService.consider(token_id, score_id, now) -> NotificationDecision`

- [ ] **Step 1: 写限额和30分钟冷却测试**

```python
async def test_twentieth_alert_is_sent_and_twenty_first_is_suppressed(service):
    for n in range(20):
        assert (await service.consider(f"token-{n}", "focus", now)).send
    assert not (await service.consider("token-20", "focus", now)).send
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/notifications/test_service.py -v`  
Expected: FAIL because notification service is missing.

- [ ] **Step 3: 实现后端Toast和降级**

Notification title includes token symbol; body includes source, heat, risk and one reason. Store decision before attempting Windows Toast. On Toast failure, retain alert with `delivery_status="failed"`.

- [ ] **Step 4: 运行测试**

Run: `pytest tests/notifications/test_service.py -v`  
Expected: PASS with a fake notifier and no real popup.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/notifications tests/notifications
git commit -m "feat: add bounded Windows opportunity alerts"
```

### Task 10: 本地网页和手动分析

**Files:**
- Create: `src/meme_radar/web/routes.py`
- Create: `src/meme_radar/web/forms.py`
- Create: `src/meme_radar/web/templates/base.html`
- Create: `src/meme_radar/web/templates/opportunities.html`
- Create: `src/meme_radar/web/templates/tokens.html`
- Create: `src/meme_radar/web/templates/manual_analysis.html`
- Create: `src/meme_radar/web/templates/token_detail.html`
- Create: `src/meme_radar/web/templates/settings.html`
- Create: `src/meme_radar/web/static/app.css`
- Create: `src/meme_radar/web/static/chart.js`
- Create: `tests/web/test_routes.py`

**Interfaces:**
- Consumes: repositories and analysis application services
- Produces: `/`, `/tokens`, `/analyze`, `/tokens/{address}`, `/settings`, `/api/tokens/{address}/series`

- [ ] **Step 1: 写页面与错误输入测试**

```python
def test_invalid_contract_returns_clear_message(client):
    response = client.post("/analyze", data={"address": "not-an-address"})
    assert response.status_code == 422
    assert "请输入有效的Robinhood Chain合约地址" in response.text
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/web/test_routes.py -v`  
Expected: FAIL because routes are missing.

- [ ] **Step 3: 实现已批准的六个视图状态**

Use Jinja templates and small vanilla-JS helpers. Draw price/volume with local SVG code in `chart.js`; no Node build step. Render unknown values as `未知` and distinguish market cap from FDV. Add disclaimer and verified external links.

- [ ] **Step 4: 运行Web测试**

Run: `pytest tests/web/test_routes.py -v`  
Expected: PASS for empty, loading, data, unknown and error states.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/web tests/web
git commit -m "feat: add local Meme Radar web interface"
```

### Task 11: Windows安装、启动、停止与卸载脚本

**Files:**
- Create: `install.bat`
- Create: `start.bat`
- Create: `stop.bat`
- Create: `uninstall.bat`
- Create: `scripts/process_guard.py`
- Create: `tests/scripts/test_process_guard.py`

**Interfaces:**
- Produces: PID file `run/meme-radar.pid`; commands `process_guard.py start|stop|status`

- [ ] **Step 1: 写只停止本项目的失败测试**

```python
def test_stop_rejects_pid_whose_command_does_not_match(tmp_path, fake_process):
    result = stop_from_pidfile(tmp_path / "meme-radar.pid", fake_process)
    assert result.status == "refused"
    assert fake_process.terminated is False
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/scripts/test_process_guard.py -v`  
Expected: FAIL because guard is missing.

- [ ] **Step 3: 实现脚本**

`install.bat` creates `.venv`, installs the locked project, copies `.env.example` only when `.env` is absent, runs migrations and health self-test. `start.bat` writes a verified PID and opens the browser only after `/health` succeeds. `stop.bat` validates executable path and command line. `uninstall.bat` lists targets and requires confirmation before deleting only `.venv`, `data`, `logs`, `run` and generated shortcuts.

- [ ] **Step 4: 运行自动测试和Windows手动检查清单**

Run: `pytest tests/scripts/test_process_guard.py -v`  
Expected: PASS. On Windows Sandbox, verify install → start → health → stop; record results in `docs/TEST_RESULTS_STAGE1.md`.

- [ ] **Step 5: 提交**

```bash
git add install.bat start.bat stop.bat uninstall.bat scripts tests/scripts docs/TEST_RESULTS_STAGE1.md
git commit -m "feat: add safe Windows lifecycle scripts"
```

### Task 12: 阶段1端到端验收

**Files:**
- Create: `tests/integration/test_chain_pipeline.py`
- Create: `docs/USER_GUIDE_STAGE1.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: all stage1 components
- Produces: reproducible fixture pipeline and user test guide

- [ ] **Step 1: 写端到端固定样本测试**

```python
async def test_pool_event_becomes_persisted_focus_alert(stage1_app, focus_fixture):
    await stage1_app.scanner.handle_event(focus_fixture.event)
    token = await stage1_app.tokens.get(focus_fixture.token_address)
    assert token.latest_classification == "focus"
    assert await stage1_app.alerts.count_for(token.id) == 1
```

- [ ] **Step 2: 运行端到端测试并修复接口差异**

Run: `pytest tests/integration/test_chain_pipeline.py -v`  
Expected: PASS using fixtures and fake external adapters.

- [ ] **Step 3: 运行阶段1全量验证**

Run: `pytest -v && ruff check . && git diff --check`  
Expected: all tests pass, no lint or whitespace errors.

- [ ] **Step 4: 编写非技术用户试用步骤**

Guide must cover install, configuration status, start, test notification, manual address validation, stop and safe uninstall. It must not require terminal commands for ordinary use.

- [ ] **Step 5: 提交并停止**

```bash
git add tests/integration docs/USER_GUIDE_STAGE1.md README.md
git commit -m "test: complete stage one chain acceptance"
```

Report test evidence and user steps. Do not start the X/KOL plan until the user explicitly approves stage1.
