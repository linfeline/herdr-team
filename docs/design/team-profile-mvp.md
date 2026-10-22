# Herdr Team — Team Profile MVP 设计

状态：设计稿（尚未实现）  
适配基线：Herdr 0.9.4；上游 zackshen/herdr-workspace v0.1.2；Codex CLI 0.162.1；Claude Code 2.1.293。升级版本须重新验证相关接口。

## 1. 定位与边界

Herdr Team 是 herdr-workspace 的兼容性扩展，提供“选择团队模板 → 创建 Workspace/Worktree → 启动原生 CLI Agent → 恢复可发现的角色绑定”。长期团队身份独立于 Goal；任务目标仍由 Lead 使用原生 Codex/Claude 工作流处理。

- 保留原来的 Solo、Pair、Full 布局模板、弹窗以及 Workspace/Worktree 创建链路。
- 新增可复用 Team Definition、角色启动上下文、Role/Instance Directory 和恢复校验。
- 复用 Herdr 的 Workspace/Pane/Agent、Agent prompt、Agent view、startup hook；不实现独立任务调度器、消息队列、自动审批代理。
- 不自动修改现有工作区文件、初始化 Git/Trellis，也不自动操作实体设备、串口或数据库。
- 插件 ID、二进制、Release 名称分别改为 herdr-team，避免与 herdr-workspace 的注册与安装配置冲突。

## 2. 分层

| 层 | 数据/职责 | 生命周期 |
| --- | --- | --- |
| Layout Profile | 选用团队、Tabs、Panes、角色落位 | 用户配置，可跨项目复用 |
| Team Definition | 逻辑角色、CLI kind、指令文件、权限预设、协作关系 | 静态、独立于 Goal |
| Project Rules | 项目 AGENTS.md / CLAUDE.md / Trellis 与工作目录约束 | 项目持久化 |
| Agent Instance | Herdr 返回的实际 agent_name、pane_id、workspace_id、CLI 会话标识（如可获得） | 当前 Session 运行时 |
| Peer Directory | role → 有效的 Agent Instance 映射；发现和投递前实时校验 | 可重建 |
| Goal / Tasks | Lead 的 /goal、项目任务状态和交付文件 | 原生 Agent / 项目体系负责 |

“lead”“builder”只是逻辑角色，不能直接当作 Herdr Agent name。角色和实例为 1:0..1 关系；离线、冲突或未验证的实例不得成为有效通信目标。

## 3. 配置结构

兼容上游 config.yaml 的 profiles[]。Team 模板为可选能力：profile.team 指向 teams/<id>.yaml，pane.role 指向模板中的逻辑角色。旧版 pane.agent/kind/args 维持原行为；一个 pane 不能同时声明 role 与 agent/command。

### config.yaml 示例

~~~yaml
profiles:
  - id: dev-triad
    name: Development Team
    description: Lead + Builder + Reviewer
    team: dev-triad
    tabs:
      - name: work
        panes:
          - id: lead
            name: lead
            role: lead
          - id: builder
            name: builder
            split_from: lead
            direction: right
            ratio: 0.5
            role: builder
      - name: review
        panes:
          - id: reviewer
            name: reviewer
            role: reviewer
~~~

### teams/dev-triad.yaml 示例

~~~yaml
schema_version: 1
id: dev-triad
roles:
  lead:
    kind: codex
    instructions: roles/lead.md
    peers: [builder, reviewer]
    args: []
    permissions: workspace-write
  builder:
    kind: claude
    instructions: roles/builder.md
    peers: [lead, reviewer]
    args: []
    permissions: interactive
  reviewer:
    kind: codex
    instructions: roles/reviewer.md
    peers: [lead]
    args: []
    permissions: read-only
~~~

校验规则：role 键唯一、每个引用均存在、每角色恰有一个目标 pane、所有 instructions 必须位于可信配置目录的受控相对路径、args 是 argv 数组而不是任意 shell 脚本、权限预设必须被对应 CLI 支持。项目目录不作为 Team Definition 的默认写入位置。未显式选择的模型采用 CLI 自身配置，不硬编码。

## 4. 启动时序

1. 表单读取 Layout Profile 和 Team Definition，完整校验并生成不可变启动计划。
2. 调用现有 apply.rs 创建 Workspace/Tab/Pane；记录 Herdr 实际返回的 workspace_id 和 pane_id。
3. 弹窗关闭后，由现有 detached starter 并行调用 herdr agent start，继续沿用上游的启动等待与名称冲突策略。
4. 每次启动成功后，以 Agent API 返回结果及 agent.get/pane 核验实际名称；记录因冲突产生的 -2/-3 等后缀，绝不推导目标名。
5. 生成 Role/Instance Directory 和角色可读取的运行时说明；给各 Pane 附加 Herdr 元数据（team_id、role），必要时通过 agent.view.set 调整团队呈现。
6. 结果按角色记录 ready / blocked / failed；部分失败仍保留已创建 Workspace，不静默视为全部成功。

原则：先建立实际实例绑定，再允许跨角色发消息；不能把 herdr agent start 或 agent prompt 的成功响应当作业务任务完成。

## 5. 角色指令和发现

每个 Agent 的静态启动上下文包含：本人职责、其他角色简述、Herdr 通信方式、当前团队/Workspace 定位、权限边界，以及“每次投递前动态解析目标”的指令。不得在长期 Prompt 中写死实际 Agent name 或 pane_id。

- Codex：通过启动参数 -c developer_instructions=<TOML 字符串> 附加角色说明，沿用原来的 AGENTS.md / 项目配置；禁止使用会替代 AGENTS.md 的 model_instructions_file 来实现增量角色注入。
- Claude Code：通过 --append-system-prompt <文字> 附加角色说明，保留 CLAUDE.md；不把 --agents 子代理定义误当作顶层 CLI Session 的团队角色机制。
- 上述参数来自当前 CLI help 和官方配置参考。重启、resume 与 compact 后的规则保留语义，必须在隔离环境分别验收，不预设一定延续。
- 新增本地 herdr-team team list / team peers / team resolve <role> / team send <role> <message> 命令。角色指令要求 Agent 先 resolve，再 send；没有角色专用调度常驻进程。
- resolve 在同一命名 Herdr Session 与 Workspace 下查找本团队记录，重新调用 agent.get 验证 agent_name、pane_id、kind 和 Workspace 归属；歧义时拒绝投递。
- send 使用原生 herdr agent prompt，返回 submitted / blocked / stale / failed。submitted 仅代表 Herdr 接受了输入，不表示接收方已完成工作；MVP 不提供持久排队、自动重试或业务 ACK。
- Peer Directory 允许角色间通信，也允许 Lead 在发送前发现实际 Builder/Reviewer 实例；跨 Workspace 仅在显式指定目标、且目标独立校验后开放，不做全局模糊查找。

## 6. 权限与安全

| 预设 | 初版映射 | 保证边界 |
| --- | --- | --- |
| read-only | Codex --sandbox read-only | 仅限 Codex 沙箱实际覆盖的工具；外部接口仍需额外限制 |
| workspace-write | Codex --sandbox workspace-write + 默认审批策略 | 非授权目录、网络和外部设备不得因角色指令自动获得权限 |
| interactive | Claude Code 默认交互式权限 | 明确使用其原生允许/拒绝及审批机制 |

Reviewer 的“只读”不得仅由 Prompt 宣称；若所选 CLI、外部工具或共享路径无法满足强制只读，应标为 advisory 或在严格模式拒绝启动。初版不得默认授予 danger-full-access、bypassPermissions 或危险自动批准。角色可以通过 Herdr 通信，但对真实设备、数据库、外部部署等动作仍须独立授权。

## 7. 持久化与恢复

配置放在 HERDR_PLUGIN_CONFIG_DIR；运行实例状态放在 HERDR_PLUGIN_STATE_DIR。每个命名 Session / Workspace 独立记录，例如：state/<session-key>/<workspace-id>/team-runtime.json。session-key 的稳定取得方式应以实际 Herdr 0.9.4 命名 Session 环境/API 验证为准，未确认时禁止跨 Session 自动解析。

~~~json
{
  "schema_version": 1,
  "team_id": "dev-triad",
  "profile_id": "dev-triad",
  "session_key": "default",
  "workspace_id": "w1",
  "definition_revision": 1,
  "members": {
    "lead": {
      "kind": "codex",
      "agent_name": "sample-lead",
      "pane_id": "w1:p1",
      "state": "ready"
    }
  }
}
~~~

- 使用原子写入和针对同一团队实例的互斥，避免并发创建、名称竞争、半写入状态。
- 启动新版本时按 schema_version / definition_revision 检查兼容性，不自动迁移未知格式，不保存凭据。
- Herdr [[startup]] hook 在 Session 恢复后执行一次 reconcile：读取现有 state、向当前 server 查询真实 Agent、标记 stale/missing/blocked，恢复元数据或 Agent View；默认不自动重启缺失 Agent，也不重发旧消息。
- 同 Pane 发生新进程占用、Agent 重命名或移动时，绝不依旧名称盲投。仅当当前实例身份能唯一验证时更新映射；否则标记需人工重新绑定。
- CLI 自身 resume 是另一层恢复：必须校验角色上下文仍生效且没有误把其他会话绑定为该角色。若无法判定，拒绝自动发送。
- 关闭 Workspace 仅清除/归档该 Workspace 的 Team 运行态，不触碰其他 Workspace 的 Agent。

## 8. 修改落点

| 文件/模块 | 改造内容 |
| --- | --- |
| herdr-plugin.toml、构建/安装/发布脚本 | 新 plugin ID 和二进制 herdr-team；声明 startup hook |
| src/config.rs | 兼容旧 schema；增加 profile.team 和 pane.role |
| src/team_config.rs（新） | 解析、校验 teams/<id>.yaml 及角色指令 |
| src/apply.rs | 返回结构化启动计划，包含 pane_id ↔ role，而非仅 argv |
| src/popup.rs、src/main.rs | 完成弹窗后启动、逐角色结果收集、失败留痕 |
| src/role_context.rs（新） | Codex/Claude 特化 argv、配置引用、权限预设 |
| src/team_state.rs（新） | 持久化、冲突校验、resume/reconcile |
| src/team_cli.rs（新） | list/peers/resolve/send；会话内校验 |
| src/herdr.rs | 统一解析 Herdr API 的实际目标和错误 |

优先在既有源码中扩展，避免 fork 后整体重写 UI 与布局流程。初版不引入数据库、常驻后台服务或新的通用 Agent SDK。

## 9. 验收标准

1. 同一 Herdr Session 中并行创建两套同名团队，role 可解析到不同、正确的 agent_name/pane_id，名称冲突不会错投。
2. Lead/Builder/Reviewer 各自加载不同且可核对的角色说明；原有 AGENTS.md/CLAUDE.md 仍有效。
3. Lead 通过逻辑角色发现并向正确 Builder 发消息；阻塞、离线或目标被替换时拒绝或明确报错。
4. Reviewer 的只读限制由实际 CLI 沙箱验证，不能通过简单 Prompt 绕过文件写入限制。
5. 原版 Solo/Pair/Full 仍可创建；New Worktree 功能行为不回归。
6. Herdr Server 重启后，能重新匹配恢复的实例、识别失效实例；绝不重新投递先前消息。
7. Compact、CLI resume、Pane 移动/替换、配置升级和部分启动失败各有确定的恢复或拒绝路径。
8. 所有集成测试在独立命名 Session 与隔离目录进行，不影响当前研发 Agent 或真实设备。

## 10. 实施顺序

Phase 1：插件重命名、旧模板兼容、Team Schema 与静态校验。  
Phase 2：角色原生启动指令、结构化实例记录、动态解析与发送。  
Phase 3：startup reconcile、状态恢复与严格权限验收。  
Phase 4（非 MVP）：可选 Agent View 优化、业务 ACK/交付确认和跨 Workspace 显式协作。

## 11. 技术依据

- 上游源码与 Profile 规范：https://github.com/zackshen/herdr-workspace/blob/v0.1.2/docs/profiles.md
- 上游启动实现：https://github.com/zackshen/herdr-workspace/blob/v0.1.2/src/main.rs
- Herdr 插件生命周期：https://herdr.dev/docs/plugins/
- Herdr Socket API（agent.*、agent.view.*、session.snapshot、pane.report_metadata）：https://herdr.dev/docs/socket-api/
- Herdr CLI、命名 Session：https://herdr.dev/docs/cli-reference/
- Codex 官方配置参考（developer_instructions、model_instructions_file）：https://developers.openai.com/codex/config-reference
- Claude Code CLI 与权限：https://code.claude.com/docs/en/cli-reference ，https://code.claude.com/docs/en/permissions

开放验证点：角色上下文在 Codex/Claude resume 与 compact 后是否保留；命名 Session 的稳定命名来源；Herdr 0.9.4 中 pane 元数据令牌与 Agent Session 标识的重启行为；CLI 只读对外部工具和跨 Pane 通信的实际约束。