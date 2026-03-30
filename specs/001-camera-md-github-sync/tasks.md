---
description: "Task list for 001-camera-md-github-sync — camera-md-sync CLI"
---

# Tasks: Camera 经验仓 Markdown 自动同步

**Input**: Design documents from `E:\cursor\Camera\specs\001-camera-md-github-sync\`  
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/)

**Tests**: 规格强调可测性；下列包含**针对性单元测试**（配置、路径匹配、退出码映射），不强制全量 TDD。

**Organization**: 按用户故事分组，便于独立实现与验收。

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 可并行（不同文件、无前后依赖）
- **[Story]**: `US1` / `US2` / `US3` 或 `—`（跨故事基础设施）

## Path Conventions

仓库根目录：`E:\cursor\Camera\`  
源码：`E:\cursor\Camera\src\camera_md_sync\`  
测试：`E:\cursor\Camera\tests\`

---

## Phase 1: Setup（共享基建）

**Purpose**: 仓库与包骨架

- [x] T001 [—] 在 `E:\cursor\Camera\` 创建 `src/camera_md_sync/` 与 `tests/` 空包结构（含 `src/camera_md_sync/__init__.py`）
- [x] T002 [—] 添加 `E:\cursor\Camera\pyproject.toml`：`camera-md-sync` 包、`python>=3.11`、依赖 `typer`、`watchfiles`、`pydantic`、`pyyaml`、`pytest`、`ruff`（dev）
- [x] T003 [P] [—] 添加 `E:\cursor\Camera\.gitignore`：`.venv/`、`__pycache__/`、`.env`、`*token*`、可选 `camera-md-sync.local.yaml`
- [x] T004 [P] [—] 添加 `E:\cursor\Camera\README.md`：一句话说明、链到 `specs/001-camera-md-github-sync/quickstart.md`

---

## Phase 2: Foundational（阻塞性前置）

**Purpose**: 所有用户故事开始前必须完成的核心模块

**⚠️ CRITICAL**: 未完成本阶段前不得开始 US1/US2/US3 功能开发

- [x] T005 [—] 在 `E:\cursor\Camera\src\camera_md_sync\models.py` 实现 Pydantic 模型，对齐 [data-model.md](./data-model.md) 的 `SyncConfig` / `SyncJob` 字段（禁止在模型中出现 token 字段）
- [x] T006 [—] 在 `E:\cursor\Camera\src\camera_md_sync\config_loader.py` 实现：加载 YAML、`CAMERA_MD_SYNC_CONFIG` 覆盖、合并环境占位、用 `contracts/config.schema.json` 校验（可用 `jsonschema` 或 pydantic 手工对齐 schema）
- [x] T007 [P] [—] 在 `E:\cursor\Camera\src\camera_md_sync\paths.py` 实现：`content_root` 下 `include_globs` / `exclude_globs` 判断某绝对路径是否应监视（规范化路径分隔符）
- [x] T008 [—] 在 `E:\cursor\Camera\src\camera_md_sync\logging_config.py` 配置 stderr 日志格式；支持 `--quiet` / `--verbose`（由 `cli.py` 传入）
- [x] T009 [—] 在 `E:\cursor\Camera\src\camera_md_sync\git_sync.py` 实现：`run_git(args, cwd)` 封装、`map_git_error_to_exit_code`（对齐 [contracts/cli.md](./contracts/cli.md) 的 1–5 类失败），**不**打印 token
- [x] T010 [P] [—] 在 `E:\cursor\Camera\tests\test_config.py` 编写配置样例与非法 YAML（含敏感键）的断言
- [x] T011 [P] [—] 在 `E:\cursor\Camera\tests\test_paths.py` 编写 Windows 路径下 glob 包含/排除用例
- [x] T012 [P] [—] 在 `E:\cursor\Camera\tests\test_git_exit_codes.py` 用 mock subprocess 断言 stderr 输出映射到正确退出码类别

**Checkpoint**: 模型、配置加载、路径规则、Git 封装可单独单元测试通过

---

## Phase 3: User Story 1 — 配置目标经验仓 (Priority: P1) 🎯 MVP

**Goal**: 用户可配置 `repo_path`、`content_root`、分支与监视规则，并完成一次「连通性」验证（`doctor`），可选生成模板配置（`init-config`）。

**Independent Test**: 配置有效仓库后运行 `doctor` 返回 0；无效配置返回 1 且信息可读（见 spec US1）。

### Implementation（US1）

- [x] T013 [US1] 在 `E:\cursor\Camera\src\camera_md_sync\cli.py` 注册 Typer：`doctor`（`--config`），调用校验：`repo_path` 存在且含 `.git`、`content_root` 位于仓库内
- [x] T014 [US1] `doctor` 可选执行 `git ls-remote origin`（超时处理归为网络类），成功则打印简短健康摘要
- [x] T015 [US1] 实现 `init-config`：写出最小 `camera-md-sync.yaml` 至 `--output`（默认 `./camera-md-sync.yaml`），**不含** token；打印提示使用 `GITHUB_TOKEN` / `CAMERA_MD_SYNC_TOKEN`
- [x] T016 [US1] 在 `git_sync.py` 或 `cli.py` 确保配置保存路径与 `FR-007` 一致：示例中说明勿提交含密钥的本地覆盖文件

**Checkpoint**: 用户仅凭 US1 可完成仓库配置与健康检查（尚可不运行 watch）

---

## Phase 4: User Story 2 — 保存后自动同步 Markdown (Priority: P2)

**Goal**: 监视 `content_root` 下符合规则的 `.md` 保存事件，去抖后 `git add` → `commit` → `push`。

**Independent Test**: 保存监视范围内 md 后，远程在约定时间内可见（spec US2）；范围外不触发（T023）。

### Implementation（US2）

- [x] T017 [US2] 在 `E:\cursor\Camera\src\camera_md_sync\watcher.py` 使用 `watchfiles.watch` 递归监视 `repo_path` 下与 `content_root` 相关的目录，过滤非 md / 排除 glob
- [x] T018 [US2] 在 `watcher.py` 实现按路径去抖（默认 `debounce_ms`，合并同文件多次事件）
- [x] T019 [US2] 实现 `sync` 子命令：对当前工作区变更执行流水线——`git add`（限定变更文件）、`git commit -m`（`commit_message_template`）、`git push origin <branch>`
- [x] T020 [US2] 实现 `watch` 子命令：循环监听 + 去抖后调用与 `sync` 相同流水线；SIGINT 优雅退出
- [x] T021 [US2] `watch` / `sync` 支持可选 `--dry-run`：仅日志，不执行 `commit`/`push`（若契约需调整，同步更新 `E:\cursor\Camera\specs\001-camera-md-github-sync\contracts\cli.md`）
- [x] T022 [P] [US2] 在 `E:\cursor\Camera\tests\test_debounce.py` 用假时钟或短间隔模拟去抖合并行为
- [x] T023 [US2] 手动或轻量集成：在临时 git 仓库中修改 `content_root` 下文件，断言 `sync` 产生提交（可在 CI 可选跳过）

**Checkpoint**: US1+US2 即完整「编辑保存 → 远端更新」MVP

---

## Phase 5: User Story 3 — 失败可见与可恢复 (Priority: P3)

**Goal**: 失败非静默；日志可区分网络、鉴权、冲突、配置；用户修复后可重试成功。

**Independent Test**: 断网或错误 token 下同步失败有明确输出与退出码；恢复后重试成功（spec US3）。

### Implementation（US3）

- [x] T024 [US3] 在 `git_sync.py` 解析 `git` stderr，填充 `SyncJob.failure_category`（`network` / `auth` / `conflict` / `config` / `unknown`）
- [x] T025 [US3] 在 `watch`/`sync` 每次同步结束打日志：状态、`failure_category`、耗时；**禁止**在日志中输出完整 remote URL 若含 token
- [x] T026 [US3] 文档化重试：在 `E:\cursor\Camera\README.md` 或 `quickstart` 链中说明「修复凭据后再次运行 `sync`/`watch`」
- [x] T027 [P] [US3] 在 `tests/test_git_exit_codes.py` 扩展：模拟非快进 / 鉴权失败字符串，断言分类

**Checkpoint**: US3 完成则满足 FR-005 与 SC-003 的工程落地

---

## Phase 6: Polish & Cross-Cutting

**Purpose**: 横切收尾

- [x] T028 [P] [—] 在 `E:\cursor\Camera\pyproject.toml` 配置 `[project.scripts]`：`camera-md-sync = camera_md_sync.cli:app`（或 Typer 入口名一致）
- [x] T029 [P] [—] 运行 `ruff check E:\cursor\Camera\src` 并修复可自动修复项
- [x] T030 [—] 按 [quickstart.md](./quickstart.md) 在干净 venv 中走通 `pip install -e .`、`doctor`、`watch`（冒烟，可记为手动验收清单）
- [x] T031 [—] 更新 `E:\cursor\Camera\.cursor\rules\specify-rules.mdc` 中 Commands 与实际 `pytest`/`ruff` 路径一致（若已实现）

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1** → **Phase 2** → **Phase 3–5（按 P1→P2→P3）** → **Phase 6**
- **US2** 依赖 **US1**（需有效配置与 `doctor` 路径）
- **US3** 依赖 **US2** 的同步流水线（在流水线上增强观测）

### Parallel Opportunities

- T003、T004；T010、T011、T012；T022、T027；T028、T029 可并行
- US1/US2/US3 串行更符合依赖，但 T013–T016 内部部分可并行于 T017 之前的文档任务

---

## Implementation Strategy

### MVP First（仅 US1）

1. Phase 1 + Phase 2  
2. Phase 3（US1）  
3. **停止并验收**：`init-config` + `doctor` 独立可用  

### Incremental Delivery

1. 加上 Phase 4（US2）→ 完整自动同步  
2. 加上 Phase 5（US3）→ 可运维  
3. Phase 6 发布就绪  

---

## Notes

- 所有任务描述中的路径均基于绝对根 `E:\cursor\Camera\`；克隆到其他机器时请替换前缀。  
- 若实现 **research.md 方案 B**（笔记目录与克隆分离），需新增任务调整 `git_sync` 与 `paths`，本列表默认 **方案 A**。  
- 提交：建议每完成一个 User Story 打一个 checkpoint tag 或 PR，便于独立演示。

<!-- camera-md-sync: 测试同步至 GitHub Camera_database（2026-03-30） -->
