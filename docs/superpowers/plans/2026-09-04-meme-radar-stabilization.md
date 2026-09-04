# Meme雷达稳定与交付 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将通过阶段1和阶段2验收的功能验证为可由非技术Windows用户安全安装、连续运行、停止和卸载的版本。

**Architecture:** 不增加产品功能，只强化重连、限流、日志脱敏、运行观测和Windows生命周期。用固定数据回放验证评分，用24小时试运行记录真实稳定性，所有默认阈值调整必须有前后证据。

**Tech Stack:** 既有技术栈、pytest、pytest-timeout、pytest-socket、ruff、Windows Sandbox或干净Windows测试机

**Spec:** `docs/superpowers/specs/2026-09-04-meme-radar-design.md`

## Global Constraints

- 阶段1和阶段2必须已通过用户验收。
- 不增加其他平台、公链、云端、AI、钱包或交易功能。
- 阈值只能基于记录的真实样本调整，且必须更新文档。
- 最终交付不得包含真实API密钥、本地数据库或用户日志。

---

### Task 1: 断线、限流和单源故障恢复

**Files:**
- Create: `src/meme_radar/resilience/retry.py`
- Modify: `src/meme_radar/scanner/service.py`
- Modify: `src/meme_radar/sources/health.py`
- Create: `tests/resilience/test_retry.py`
- Create: `tests/integration/test_source_isolation.py`

**Interfaces:**
- Produces: `RetryPolicy.delays() -> Iterator[float]`, isolated source lifecycle states

- [ ] **Step 1: 写指数退避与隔离测试**

```python
def test_retry_delays_are_bounded():
    policy = RetryPolicy(initial=1, multiplier=2, maximum=60)
    assert list(itertools.islice(policy.delays(), 7)) == [1, 2, 4, 8, 16, 32, 60]

async def test_pons_failure_does_not_stop_uniswap(app):
    await app.sources.pons.fail_once(TimeoutError())
    await app.sources.uniswap.emit(sample_pool_event())
    assert await app.tokens.count() == 1
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/resilience tests/integration/test_source_isolation.py -v`  
Expected: FAIL before resilience policy is wired.

- [ ] **Step 3: 实现带抖动的有界重试**

Retry timeouts and 5xx; obey `Retry-After` for 429; do not retry validation errors. Persist source status and last successful timestamp. After repeated failure, keep other sources running and expose degraded status.

- [ ] **Step 4: 运行测试**

Run: `pytest tests/resilience tests/integration/test_source_isolation.py -v`  
Expected: PASS with fake clocks and no real waiting.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/resilience src/meme_radar/scanner src/meme_radar/sources tests
git commit -m "fix: isolate source failures and bound retries"
```

### Task 2: 密钥脱敏与仓库安全检查

**Files:**
- Create: `src/meme_radar/security/redaction.py`
- Create: `scripts/check_secrets.py`
- Create: `tests/security/test_redaction.py`
- Modify: `pyproject.toml`

**Interfaces:**
- Produces: `redact(value: object) -> object`, repository secret scan command

- [ ] **Step 1: 写授权头和URL密钥脱敏测试**

```python
def test_redacts_bearer_and_query_keys():
    raw = {"Authorization": "Bearer secret-token", "url": "https://x.test?a=1&apikey=secret"}
    clean = redact(raw)
    assert "secret-token" not in repr(clean)
    assert "apikey=secret" not in repr(clean)
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/security/test_redaction.py -v`  
Expected: FAIL because redaction is missing.

- [ ] **Step 3: 实现递归脱敏和提交前扫描**

Mask keys matching `authorization`, `api_key`, `apikey`, `token`, `secret`, and URL query values with those names. `scripts/check_secrets.py` scans tracked text files and fails on known credential formats while allowing `.env.example` empty values.

- [ ] **Step 4: 运行安全检查**

Run: `pytest tests/security/test_redaction.py -v && python scripts/check_secrets.py && git check-ignore .env data/test.sqlite3`  
Expected: PASS; both sensitive local paths are ignored.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/security scripts/check_secrets.py tests/security pyproject.toml
git commit -m "security: redact credentials and scan tracked files"
```

### Task 3: 评分回放与阈值校准报告

**Files:**
- Create: `scripts/replay_scores.py`
- Create: `tests/analysis/test_replay.py`
- Create: `docs/SCORING_CALIBRATION.md`
- Modify: `.env.example`

**Interfaces:**
- Produces: `replay(records, thresholds) -> CalibrationReport`

- [ ] **Step 1: 写确定性回放测试**

```python
def test_same_fixture_produces_same_report(calibration_fixture):
    first = replay(calibration_fixture, Thresholds.defaults())
    second = replay(calibration_fixture, Thresholds.defaults())
    assert first.model_dump() == second.model_dump()
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/analysis/test_replay.py -v`  
Expected: FAIL because replay tool is missing.

- [ ] **Step 3: 实现匿名样本导出和离线回放**

Export only token/pool metrics and computed results; exclude API keys and user settings. Report sample count, focus/observe/high-risk counts, unknown-data rate, duplicate-alert rate and each proposed threshold change.

- [ ] **Step 4: 收集至少24小时样本并记录决定**

Run: `python scripts/replay_scores.py --database data/meme_radar.sqlite3 --output docs/SCORING_CALIBRATION.md`  
Expected: report contains the exact old/default thresholds and either evidence-backed new values or the explicit decision `保持规格书默认阈值`.

- [ ] **Step 5: 运行测试并提交**

```bash
pytest tests/analysis/test_replay.py -v
git add scripts/replay_scores.py tests/analysis/test_replay.py docs/SCORING_CALIBRATION.md .env.example
git commit -m "test: calibrate scoring with recorded observations"
```

### Task 4: 24小时运行观测

**Files:**
- Create: `src/meme_radar/observability/runtime_metrics.py`
- Create: `scripts/soak_report.py`
- Create: `tests/observability/test_runtime_metrics.py`
- Create: `docs/SOAK_TEST.md`

**Interfaces:**
- Produces: local counters for source reconnects, queue depth, scan latency, notification suppression, database size and X estimated cost

- [ ] **Step 1: 写指标不含敏感数据测试**

```python
def test_runtime_snapshot_has_only_approved_fields(metrics):
    snapshot = metrics.snapshot().model_dump()
    assert set(snapshot) == {"uptime_s", "reconnects", "queue_depth", "scan_latency_ms", "alerts_sent", "alerts_suppressed", "database_bytes", "x_estimated_usd"}
```

- [ ] **Step 2: 运行失败测试**

Run: `pytest tests/observability/test_runtime_metrics.py -v`  
Expected: FAIL because runtime metrics are missing.

- [ ] **Step 3: 实现本地观测与报告**

Metrics remain local and contain no addresses, posts, API headers or keys. `soak_report.py` summarizes snapshots and exits nonzero for unbounded queue growth, repeated duplicate alerts, database write failures or crashed scheduler.

- [ ] **Step 4: 执行24小时试运行**

Run: `python scripts/soak_report.py --database data/meme_radar.sqlite3 --output docs/SOAK_TEST.md` after at least 24 hours uptime.  
Expected: report records start/end time, uptime, failures, database growth, alert counts and pass/fail for each acceptance condition.

- [ ] **Step 5: 提交**

```bash
git add src/meme_radar/observability scripts/soak_report.py tests/observability docs/SOAK_TEST.md
git commit -m "test: document 24 hour runtime stability"
```

### Task 5: 干净Windows端到端生命周期验收

**Files:**
- Create: `docs/WINDOWS_ACCEPTANCE.md`
- Modify: `install.bat`
- Modify: `start.bat`
- Modify: `stop.bat`
- Modify: `uninstall.bat`
- Modify: `docs/USER_GUIDE_STAGE1.md`
- Modify: `docs/USER_GUIDE_STAGE2.md`

**Interfaces:**
- Produces: signed-off Windows acceptance checklist

- [ ] **Step 1: 在干净Windows环境执行安装**

Run: double-click `install.bat`.  
Expected: `.venv` and database are created, `.env` is not overwritten if present, and a clear completion message appears.

- [ ] **Step 2: 执行启动与浏览器验收**

Run: double-click `start.bat`, close browser, use the test-notification action, then reopen `http://127.0.0.1:8765`.  
Expected: one backend process, healthy pages, and Windows notification while browser is closed.

- [ ] **Step 3: 验证X暂停**

Run: turn X on with a test/mocked endpoint, observe one scheduled cycle, turn X off, restart the app and observe two schedule intervals.  
Expected: disabled state persists and X request count remains unchanged.

- [ ] **Step 4: 验证停止和卸载边界**

Run: start a separate Python process, run `stop.bat`, then `uninstall.bat`.  
Expected: separate process remains; uninstall asks before deleting listed project directories and does not remove system Python.

- [ ] **Step 5: 记录结果并提交**

```bash
git add install.bat start.bat stop.bat uninstall.bat docs
git commit -m "docs: complete clean Windows acceptance"
```

### Task 6: 最终全量验证与交付

**Files:**
- Create: `CHANGELOG.md`
- Create: `docs/FINAL_RELEASE_CHECKLIST.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: all project components and evidence
- Produces: version `0.1.0` release candidate documentation

- [ ] **Step 1: 运行全量自动测试**

Run: `pytest -v --disable-socket`  
Expected: all tests pass; tests requiring explicit mocked hosts are registered and no unapproved network connection occurs.

- [ ] **Step 2: 运行质量与安全检查**

Run: `ruff check . && git diff --check && python scripts/check_secrets.py`  
Expected: all commands exit zero.

- [ ] **Step 3: 核对需求覆盖**

Compare every item in `docs/ACCEPTANCE_TESTS.md` with stage test results, `SCORING_CALIBRATION.md`, `SOAK_TEST.md`, and `WINDOWS_ACCEPTANCE.md`. Each item must link to evidence and be marked pass or explicitly blocked with reason.

- [ ] **Step 4: 更新交付文档**

README must contain nontechnical install/start/stop/uninstall links, security disclaimer, configuration status explanation and project limitations. CHANGELOG records only delivered behavior under `0.1.0`.

- [ ] **Step 5: 提交并停止**

```bash
git add README.md CHANGELOG.md docs/FINAL_RELEASE_CHECKLIST.md
git commit -m "release: prepare Meme Radar 0.1.0"
```

Report final evidence, remaining external-data limitations and exact user installation steps. Do not add features or publish a release without explicit user approval.

