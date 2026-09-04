# Meme雷达 X与KOL Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在通过验收的链上核心版上增加可彻底暂停、费用可控的 X KOL 低市值币发现能力。

**Architecture:** X API 由单一适配器和费用闸门包裹，调度器只处理开启状态下的新增帖子。帖子解析先生成候选提及，再由独立匹配器结合链上代币身份确认；只有已确认且通过链上条件的结果可以通知。

**Tech Stack:** 阶段1技术栈、X API v2、httpx、APScheduler、pytest、respx

**Spec:** `docs/superpowers/specs/2026-09-04-meme-radar-design.md`

## Global Constraints

- 阶段1必须已通过用户验收。
- 最多10位KOL，每5分钟检查一次，仅处理启动或重新开启之后的新帖子。
- X总开关关闭时，任何代码路径都不得调用X API。
- 默认月预算30美元，80%警告，100%暂停；页面明确费用是本地估算。
- 同名多币不得自动匹配；待确认提及不得发送强通知。
- 测试全部使用模拟响应，不消耗真实X额度。

---

### Task 1: X设置、费用账本与硬闸门

**Files:**
- Create: `src/meme_radar/x/budget.py`
- Create: `src/meme_radar/x/gate.py`
- Modify: `src/meme_radar/config.py`
- Modify: `src/meme_radar/storage/orm.py`
- Create: `migrations/versions/0002_x_kol.py`
- Create: `tests/x/test_gate.py`

**Interfaces:**
- Produces: `XRequestGate.is_allowed(now) -> GateDecision`, `XUsageLedger.record(kind, resources, estimated_usd)`

- [ ] **Step 1: 写关闭状态零请求测试**

```python
async def test_disabled_gate_never_calls_transport(gate, transport):
    gate.settings.set_x_enabled(False)
    result = await gate.execute("timeline", transport.get_new_posts)
    assert result.status == "disabled"
    assert transport.call_count == 0
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/x/test_gate.py -v`  
Expected: FAIL because gate is missing.

- [ ] **Step 3: 实现开关持久化和月度费用闸门**

```python
@dataclass(frozen=True)
class GateDecision:
    allowed: bool
    reason: Literal["enabled", "disabled", "budget_exhausted"]

async def execute(self, kind: str, call: Callable[[], Awaitable[XResult]]) -> XResult:
    decision = await self.is_allowed(self.clock.now())
    if not decision.allowed:
        return XResult(status=decision.reason, posts=[])
    result = await call()
    await self.ledger.record(kind, len(result.posts), self.prices.estimate(kind, len(result.posts)))
    return result
```

- [ ] **Step 4: 测试80%、100%、跨月和重启状态**

Run: `pytest tests/x/test_gate.py -v`  
Expected: PASS; transport call count remains zero when disabled or budget exhausted.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/x src/meme_radar/config.py src/meme_radar/storage migrations tests/x
git commit -m "feat: add persistent X API budget gate"
```

### Task 2: X API增量时间线适配器

**Files:**
- Create: `src/meme_radar/sources/x_api.py`
- Create: `tests/fixtures/x_user_timeline.json`
- Create: `tests/sources/test_x_api.py`

**Interfaces:**
- Produces: `XApiSource.resolve_user(username) -> XUser`, `fetch_since(user_id, since_id) -> XTimelineResult`

- [ ] **Step 1: 写游标与限流测试**

```python
async def test_fetch_since_sends_since_id(x_source, respx_mock):
    route = respx_mock.get("https://api.x.com/2/users/42/tweets").mock(
        return_value=httpx.Response(200, json=timeline_fixture())
    )
    await x_source.fetch_since("42", "100")
    assert route.calls[0].request.url.params["since_id"] == "100"
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/sources/test_x_api.py -v`  
Expected: FAIL because source is missing.

- [ ] **Step 3: 实现官方API适配器**

Use bearer authentication, `exclude=retweets,replies`, `tweet.fields=created_at,public_metrics,entities`, bounded pagination, explicit timeout and `Retry-After`. Return typed errors instead of raw exceptions. Never log authorization headers.

- [ ] **Step 4: 运行成功、401、429和超时测试**

Run: `pytest tests/sources/test_x_api.py -v`  
Expected: PASS with no live API calls.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/sources/x_api.py tests/fixtures tests/sources/test_x_api.py
git commit -m "feat: add incremental X timeline adapter"
```

### Task 3: KOL名单与暂停控制

**Files:**
- Create: `src/meme_radar/x/kols.py`
- Create: `tests/x/test_kols.py`
- Modify: `src/meme_radar/storage/repositories.py`

**Interfaces:**
- Produces: `KolService.add(username)`, `pause(kol_id)`, `resume(kol_id)`, `remove(kol_id)`, `active(limit=10)`

- [ ] **Step 1: 写10位上限和单独暂停测试**

```python
async def test_rejects_eleventh_kol(service):
    for n in range(10):
        await service.add(f"kol{n}")
    with pytest.raises(KolLimitError):
        await service.add("kol10")
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/x/test_kols.py -v`  
Expected: FAIL because service is missing.

- [ ] **Step 3: 实现规范化、唯一性与游标保存**

Normalize usernames by removing a leading `@` and lowercasing for uniqueness while preserving display case. Store `since_id`, `enabled`, `last_checked_at` and `last_error` per KOL.

- [ ] **Step 4: 运行测试**

Run: `pytest tests/x/test_kols.py -v`  
Expected: PASS for add, duplicate, pause, resume, remove and limit.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/x/kols.py src/meme_radar/storage tests/x/test_kols.py
git commit -m "feat: manage bounded KOL watchlist"
```

### Task 4: 帖子解析与平衡匹配

**Files:**
- Create: `src/meme_radar/matching/x_mentions.py`
- Create: `src/meme_radar/matching/token_matcher.py`
- Create: `tests/matching/test_mentions.py`
- Create: `tests/matching/test_token_matcher.py`

**Interfaces:**
- Produces: `parse_mentions(post) -> list[MentionCandidate]`, `TokenMatcher.match(candidate, created_at) -> MatchResult`

- [ ] **Step 1: 写合约优先与同名拒绝测试**

```python
async def test_contract_address_match_is_confirmed(matcher, candidate):
    result = await matcher.match(candidate, candidate.created_at)
    assert result.status == "confirmed"
    assert result.method == "contract_address"

async def test_duplicate_symbol_stays_ambiguous(matcher, symbol_candidate):
    result = await matcher.match(symbol_candidate, symbol_candidate.created_at)
    assert result.status == "ambiguous"
    assert result.token_id is None
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/matching -v`  
Expected: FAIL because parser and matcher are missing.

- [ ] **Step 3: 实现解析优先级**

Extract EVM addresses first, then cashtags, then explicit token names. Validate addresses with checksum and chain records. Name/symbol candidates may become `tentative` only when one recent chain candidate exists; multiple candidates always produce `ambiguous`.

- [ ] **Step 4: 运行测试**

Run: `pytest tests/matching -v`  
Expected: PASS for address, symbol, name, unrelated text and duplicates.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/matching tests/matching
git commit -m "feat: match KOL mentions to chain tokens safely"
```

### Task 5: 5分钟调度与低市值通知资格

**Files:**
- Create: `src/meme_radar/x/poller.py`
- Modify: `src/meme_radar/jobs/scheduler.py`
- Modify: `src/meme_radar/notifications/service.py`
- Create: `tests/x/test_poller.py`

**Interfaces:**
- Produces: `KolPoller.run_once(started_at) -> PollSummary`

- [ ] **Step 1: 写无补采和确认状态测试**

```python
async def test_first_run_sets_cursor_without_backfill(poller, x_source, now):
    summary = await poller.run_once(now)
    assert summary.posts_processed == 0
    assert all(kol.since_id for kol in await poller.kols.active())

async def test_tentative_match_never_notifies(poller, tentative_post):
    summary = await poller.process(tentative_post)
    assert summary.notifications_sent == 0
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/x/test_poller.py -v`  
Expected: FAIL because poller is missing.

- [ ] **Step 3: 实现轮询资格**

Schedule every five minutes only when the global gate is enabled. Confirmed matches qualify only when effective market cap or labeled FDV is at most 5,000,000 USD and existing chain classification is `focus`. Count KOL alerts inside the same daily limit as chain alerts.

- [ ] **Step 4: 运行测试**

Run: `pytest tests/x/test_poller.py -v`  
Expected: PASS for disabled, first-run, incremental, duplicate, tentative and confirmed cases.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/x/poller.py src/meme_radar/jobs src/meme_radar/notifications tests/x
git commit -m "feat: poll KOL posts without backfill"
```

### Task 6: KOL页面、X开关和详情页提及

**Files:**
- Create: `src/meme_radar/web/templates/kols.html`
- Modify: `src/meme_radar/web/templates/token_detail.html`
- Modify: `src/meme_radar/web/templates/settings.html`
- Modify: `src/meme_radar/web/routes.py`
- Create: `tests/web/test_x_routes.py`

**Interfaces:**
- Produces: `/kols`, `/kols/add`, `/kols/{id}/pause`, `/settings/x`, token detail X section

- [ ] **Step 1: 写关闭开关和24小时展示测试**

```python
def test_turning_x_off_persists_and_disables_refresh(client, settings_repo):
    response = client.post("/settings/x", data={"enabled": "false"})
    assert response.status_code == 303
    assert settings_repo.get_bool("x_enabled") is False
    assert client.post("/kols/refresh").status_code == 409
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/web/test_x_routes.py -v`  
Expected: FAIL because X routes are missing.

- [ ] **Step 3: 实现页面**

Show collection switch separately from notification switch. KOL page displays enabled state, last checked, last error and recent matched posts. Token detail shows count from the previous 24 hours and at most five representative posts ordered by KOL priority then engagement; use external X links.

- [ ] **Step 4: 运行Web测试**

Run: `pytest tests/web/test_x_routes.py -v`  
Expected: PASS for enabled, disabled, budget exhausted, empty and error states.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/web tests/web/test_x_routes.py
git commit -m "feat: add KOL controls and token X mentions"
```

### Task 7: 阶段2端到端验收

**Files:**
- Create: `tests/integration/test_x_pipeline.py`
- Create: `docs/USER_GUIDE_STAGE2.md`
- Create: `docs/TEST_RESULTS_STAGE2.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: all stage2 components
- Produces: reproducible KOL-to-alert fixture pipeline

- [ ] **Step 1: 写完整模拟链路测试**

```python
async def test_kol_contract_mention_becomes_bounded_alert(stage2_app, x_post, focus_token):
    await stage2_app.x_poller.process(x_post)
    mention = await stage2_app.mentions.get_by_post(x_post.id)
    assert mention.match_status == "confirmed"
    assert await stage2_app.alerts.count_for(focus_token.id) == 1
```

- [ ] **Step 2: 运行X专用和全量测试**

Run: `pytest tests/x tests/matching tests/web/test_x_routes.py tests/integration/test_x_pipeline.py -v`  
Expected: PASS with mocked X transport.

- [ ] **Step 3: 验证零真实X请求**

Run: `pytest -v --disable-socket`  
Expected: PASS; all tests either use in-process clients or explicit mocked hosts.

- [ ] **Step 4: 运行质量检查并写用户指南**

Run: `ruff check . && git diff --check`  
Expected: PASS. Guide covers API token entry, budget meaning, KOL management, X pause and confirmation that paused state persists.

- [ ] **Step 5: 提交并停止**

```bash
git add tests/integration docs README.md
git commit -m "test: complete X and KOL acceptance"
```

Report evidence and stop. Do not start stabilization until the user explicitly approves stage2.

