# Issue #6680 日志分析：日志显示今日活跃度不满 100，但用户认为实际已满

## Issue 概要

- issue：`#6680`（<https://github.com/MaaEnd/MaaEnd/issues/6680>）
- 版本 / 控制器：MaaEnd `v2.35.0-beta.1`，MXU（Windows），控制器 `Win32-Front`，游戏 1920x1080 全屏，实例「全套日常（`6gxwgx1`）」
- 任务 / 相关选项：
    - `📅日常奖励领取（DailyRewards）`，选项 `活跃度未满时告警（DailyActivityFullWarning）`（默认开启）
    - 相关前置任务 `⚔️协议空间（ProtocolSpace）`，选项 `ProtocolSpaceTab = OperatorProgression`、`ProtocolSpaceLevel = 05`、`ProtocolSpaceUseSpMedication = UseMedication`
- 用户现象：跑完全套日常后，日志出现红色告警「今日活跃度不满100，请留意日常奖励领取情况，或调整任务设置与顺序！」，但用户认为游戏内实际活跃度已经是 100。
- 日志包完整性：issue 正文、评论与事件流中仅存在 `MaaEnd-logs-v2.35.0-beta.1-20261021-002030-part01.zip` 一个分包，未见 `part02` 及后续分包；包内无 `.dmp` 文件。12:20 那次更早运行的 `maafw` 日志已被滚动覆盖，不在包内，但不影响对 23:08 这次最新复现的判断。

## 关键证据

<details><summary>点击此处展开</summary>

### 复现锁定

- `maafw` 进程 `Px8428` 于 `2026-10-20 23:06:20` 启动（MaaFramework `v5.15.0-beta.1`）。
- `mxu-tauri.log`：`23:08:44` 一次性提交 25 个任务，`task_id` 为 `200000001`–`200000025`；其中
    - 第 19 个 `ProtocolSpaceSchedule` → `task_id 200000020`
    - 第 20 个 `DailyRewardStart` → `task_id 200000021`
    - 第 24 个 `MXU_NOTIFY` → `task_id 200000025`
- 用户于 `00:00:20` 手动 `stop_agent_impl`，此时 25 个任务均已结束。
- 告警出现在 `DailyRewards`（`task_id 200000021`）执行期间，`mxu-tauri.log` 在 `23:43:49` 有 `[MXU_NOTIFY] Notification sent successfully`，与告警 focus 的 `display: ["log","notification"]` 对应。

### `maafw.bak.2026.10.20-23.59.08.617.log`（本次复现主证据）

**① 告警触发点（23:43:48.712，task_id 200000021）**

```
[2026-10-20 23:43:48.711] ... [msg=Node.Recognition.Starting]
[details={"focus":{"Node.Recognition.Succeeded":{"content":"$option.DailyActivityFullWarning.warning",
"display":["log","notification"]}},"name":"DailyTaskActivityFullCheck","reco_id":400014375,"task_id":200000021}]

[2026-10-20 23:43:48.712] ... [OCRer::analyze] DailyTaskActivityFullCheck
[all_results_=[{"box":[455,560,132,17],"score":0.999852,"text":"协议空间·技能提升"}]]
[filtered_results_=[]] [best_result_=null]
[param_.expected=["今日活跃度已达上限","今日活躍度已達上限","(?i)Daily\\s*Activity\\s*Points\\s*Maxed",
"上限到達","通行证等级提升","通行證等級提升","(?i)Pass\\s*level\\s*up","通行証レべル"]]

[2026-10-20 23:43:48.712] ... pipeline_data.inverse is true, reverse the result
[data.name=DailyTaskActivityFullCheck] [result.box=null]
```

- `DailyTaskActivityFullCheck` 是 inverse OCR 节点：只在 ROI `[330,520,500,100]` 内识别「今日活跃度已达上限」（及繁中/英/日与「通行证等级提升」变体），**未命中即判定活跃度未满 100**，并向日志与系统通知推送红色告警（节点定义见下方代码依据）。
- 该 ROI 正是日常面板底部那条黄色提示横幅（可用仓库截图核对：横幅在 720p 下约 `x≈387–614, y≈550–570`，与 OCR box `[455,560,132,17]` 完全重合）。
- 也就是说：检查那一刻，黄色横幅显示的是「**协议空间·技能提升**」，而不是「今日活跃度已达上限」，也不是「通行证等级提升」。

**② 同一心跳内的 OCR 还读到了 3 个「前往」**

```
[2026-10-20 23:43:48.678] ... OCRer::analyze
DailyTaskClaimSingleRewardOCR+DailyTaskActivityFullCheck+
[all_results_=[
 {"box":[455,560,132,17],"score":0.999852,"text":"协议空间·技能提升"},
 {"box":[990,213,72,21],"score":0.999772,"text":"40~80"},
 {"box":[991,330,70,18],"score":0.999788,"text":"40~80"},
 {"box":[991,445,70,18],"score":0.999759,"text":"40~80"},
 {"box":[991,560,70,18],"score":0.999757,"text":"40~80"},
 {"box":[1009,261,37,23],"score":0.999513,"text":"前往"},
 {"box":[1009,492,46,23],"score":0.998962,"text":"前往"},
 {"box":[1010,378,35,20],"score":0.999691,"text":"前往"}]]
```

- 日常任务列表区域仍有 3 个「前往」按钮，说明当时仍有**未完成的日常任务**；「前往」是任务未完成态的操作入口，活跃度已满时该区域不会再给「前往」。
- 结合横幅文字，可以判定 23:43:48 这一刻游戏 UI 呈现的状态是「活跃度未满、还有日常没做」，而不是「已达上限」。

**③ 告警前 37 秒，协议空间因理智不足提前结束（23:43:11，task_id 200000020）**

```
[2026-10-20 23:42:11.077] [msg=Tasker.Task.Starting] [{"entry":"ProtocolSpaceSchedule","task_id":200000020}]

[2026-10-20 23:43:05.023] ... [node_name=ProtocolSpacePopupSanityLowUseSpMedicationPrepare]  // 弹出理智不足，尝试用理智药剂
[2026-10-20 23:43:11.302] ... [cur_node_=__AutoUseSpMedicationNoUseEmergencySpBoosterFinish]
                          [data.name=ProtocolSpacePopupSanityLowNoMedicationFinish] | enter
[2026-10-20 23:43:11.303] ... [Recognizer::direct_hit] ProtocolSpacePopupSanityLowNoMedicationFinish
[2026-10-20 23:43:11.520] ... [Actuator::stop_task] Action: StopTask
[2026-10-20 23:43:11.522] [msg=Tasker.Task.Succeeded] [{"entry":"ProtocolSpaceSchedule","task_id":200000020}]
```

- `ProtocolSpacePopupSanityLowNoMedicationFinish` 的 desc 是「无可用药剂，结束任务」，动作是 `StopTask`，focus 文案为「理智不足，结束任务」。
- `StopTask` 被记为 `Tasker.Task.Succeeded`，所以 MXU 任务列表上仍然显示绿色成功，用户很容易以为协议空间已经跑完。
- 本次运行的 `ProtocolSpace` 选项是 `OperatorProgression` + 等级 05，目标就是「协议空间·技能提升」类日常；它因理智不足**没有完成**。

**④ 领取循环已穷尽，之后没有第二次活跃度检查**

- `23:43:48.705` `DailyTaskClaimSingleReward` 识别失败（OCR 无「领取」，模板匹配最高仅 0.575）；
- `23:43:48.7xx` `DailyTaskClaimActivityRewards`（左侧黄色待领活跃度徽章 ColorMatch）识别失败；
- `23:43:49` `DailyTaskComplete` → `DailyRewards` 于 `23:44:55` `Tasker.Task.Succeeded`，随后 `SeizeDeliveryJobs`（23:59:08）、`CreditShoppingMain`（00:00:19）、`MXU_NOTIFY`（00:00:20）均正常结束；
- 本次运行中活跃度检查只发生这一次，没有第二次复查。

### `mxu-tauri.log`

- `23:43:49` `[MXU_NOTIFY] Notification sent successfully`，与告警的 notification 一致；
- `00:00:20` `stop_agent_impl`，为用户手动停止 agent，非异常退出；
- `23:08:44` 提交 25 个任务，顺序与 `assets/tasks/preset/DailyFull.json` 的「全套日常」一致（`ProtocolSpace` 在 `DailyRewards` 之前）。

### `go-service.log`

- `23:42:40` 为 `ProtocolSpace` 应用 `PipelineOverride`（`SupplyPlanOverride_TACHYON_SCREENING_LATTICE`，10 个节点）；
- 23:40–23:45 区间没有 `error` / `warn` / `fail` 级别的日志，`go-service.stderr.log` 为空；
- 与本次活跃度告警无关联，不构成根因。

### `on_error/`

- 3 张截图时间均为 `2026.10.20-23.45.16`，节点分别是 `AutoDeliveryDeliveryMissionNotFound`、`AutoDeliveryOpenDeliveryMission`、`SeizeDeliveryJobsCheckOngoingJob`，画面为大世界委托界面，与日常面板 / 活跃度检查无关；
- 即 23:43:48 那一刻没有保存现场截图，无法用截图直接复核横幅，但上面的 OCR 原始结果已足够还原该时刻的 UI 文本。

### `config/mxu-MaaEnd.json`

- 实例 `6gxwgx1`：`controllerName = Win32-Front`，任务列表包含 `ProtocolSpace` 与 `DailyRewards`；
- `DailyRewards` 实际选项：`DailyEmailRewards=True`、`DailyTaskRewards=True`、`DailyClaimDeliveryJobsRewards=True`、`DailyEventRewards=True`、`DailyProtocolPassRewards=True`、`DailyActivityFullWarning=True`；
- `ProtocolSpace` 实际选项：`ProtocolSpaceTab=OperatorProgression`、`ProtocolSpaceLevel=05`、`ProtocolSpaceUseSpMedication=UseMedication`；
- 与用户口述的「全套日常 + 默认告警开启」一致。

### 代码依据

- `DailyTaskActivityFullCheck`（inverse OCR + focus 红字告警 + 默认 `enabled: false`）与 `DailyTaskEnterTab`（点击日常标签后 `post_wait_freezes: 100`，约 0.7s 后进入检查）：
  <https://github.com/MaaEnd/MaaEnd/blob/12d3d3c4bcb9e15dcbd03e500017a757627ed6eb/assets/resource/pipeline/DailyRewards/Tasks.json#L25-L54>
  <https://github.com/MaaEnd/MaaEnd/blob/12d3d3c4bcb9e15dcbd03e500017a757627ed6eb/assets/resource/pipeline/DailyRewards/Tasks.json#L215-L259>
- `DailyActivityFullWarning` 开关（`default_case: "Yes"`，通过 `pipeline_override` 置 `enabled: true`）：
  <https://github.com/MaaEnd/MaaEnd/blob/12d3d3c4bcb9e15dcbd03e500017a757627ed6eb/assets/tasks/DailyRewards.json#L33-L52>
- 告警文案：
  <https://github.com/MaaEnd/MaaEnd/blob/12d3d3c4bcb9e15dcbd03e500017a757627ed6eb/assets/locales/interface/zh_cn.json#L1525-L1526>
- 协议空间理智不足收工节点：
  <https://github.com/MaaEnd/MaaEnd/blob/12d3d3c4bcb9e15dcbd03e500017a757627ed6eb/assets/resource/pipeline/ProtocolSpace/InSpace.json#L190-L196>
- 该告警功能由 <https://github.com/MaaEnd/MaaEnd/pull/6358> 新增；<https://github.com/MaaEnd/MaaEnd/pull/6365> 修复了「通行证等级提升」横幅导致的误判，用户版本 `v2.35.0-beta.1` 已包含这两处改动。

</details>

## 根因判断

- 直接结论：**这次告警不是代码误报，而是按设计正确触发**；用户所说「实际活跃度就是 100」与 23:43:48 这一时刻的日志证据不符。
- 证据链：
    1. `23:43:11.303` `⚔️协议空间（ProtocolSpace）` 命中 `ProtocolSpacePopupSanityLowNoMedicationFinish` → `StopTask`（「理智不足，结束任务」），本次运行**没有完成**「协议空间·技能提升」这个日常；
    2. `23:43:48.712` `DailyTaskActivityFullCheck` 在日常面板底栏 ROI 内**没有**读到「今日活跃度已达上限」，读到的是「协议空间·技能提升」；同一帧 OCR 还读到 3 个「前往」按钮 → 当时日常任务确实还有未完成项；
    3. 同一心跳内单任务奖励与活跃度奖励均不可领，领取循环正常穷尽，不存在「漏领」；
    4. 因此检查时刻游戏 UI 呈现的就是「活跃度未满」，告警内容准确。
- 与 MaaEndBot AI 分析的关系：结论方向一致（正确触发，非误报），本报告补强了两点：
    - 用仓库截图确认 `[330,520,500,100]` 就是黄色提示横幅区域，因此「协议空间·技能提升」是**底栏推荐/去向提示**，不是列表里的任务行；
    - 同一帧 OCR 同时读到 3 个「前往」，进一步排除了「横幅恰好处于切换动画 / 未渲染」的误报路径（文字与按钮都稳定高置信识别）。
- 用户看到「活跃度 100」的可能解释：用户很可能是在 `00:00:20` 全部任务结束之后、手动补做了协议空间技能提升，或在次日/其他时刻查看；也可能 12:20 那次运行补做了部分任务。日志本身不支持「23:43:48 当时已满 100」。
- 潜在设计脆弱点（**未被本次日志证实为根因**，但值得加固）：该检查依赖「底栏横幅文字」这一多用途提示位。仓库截图显示活跃度满 100 时，横幅仍可能显示「通行证等级提升，可前往查看奖励」（#6365 修的就是这一类），说明横幅不是专用于活跃度的指示器；若横幅出现其他文案（新活动、其他系统提示等），即使满 100 也可能被判为未满。本次日志中横幅显示的是未完成任务提示，故不判定为本次根因。

## 给用户的建议

- 用户现在可以直接尝试的动作：
    - 检查理智与理智药剂库存。本次运行中 `⚔️协议空间` 因「理智不足且无可用紧急理智药剂」提前结束，`协议空间·技能提升` 日常没做完，这是活跃度不满 100 的直接原因；
    - 在游戏内查看日常任务列表，确认 `协议空间·技能提升` 是否处于「前往/未完成」状态；
    - 回看 23:43 前后的运行日志，应在红色告警前约 37 秒看到「理智不足，结束任务」提示。
- 是否建议升级 / 重下完整包 / 同步资源 / 重置配置：**都不需要**，当前版本行为与设计一致。
- 是否需要等待开发者修复：针对本次现象不需要。若后续仍怀疑误报，可关注 `DailyTaskActivityFullCheck` 的后续加固。
- 临时绕过方案：可在 `📅日常奖励领取` 里把 `活跃度未满时告警` 关掉（`DailyActivityFullWarning = No`），但不推荐，会同时失去真正的漏领提醒。推荐做法是保证 `⚔️协议空间` 有足够理智/药剂，并在任务顺序上把它排在 `📅日常奖励领取` 之前（当前「全套日常」顺序已经如此）。

## 修复方案

1. 代码 / Pipeline / 配置层修复
    - 建议为 `DailyTaskActivityFullCheck` 增加「画面已稳定在日常标签页」的前置确认（例如对日常标签选中态做模板/OCR 确认，或对列表锚点做一次复查）；无法确认时按「无法判定」处理，降级为普通日志而不是红色告警。
    - 可考虑把判定依据从「单一横幅文字」改为更可靠的活跃度指示（例如左侧活跃度进度条 / `100/100` 数值区域的颜色或 OCR），横幅仅作辅助证据，降低多用途提示位带来的误报风险。
    - 告警文案可附带检测到的横幅原文（如「协议空间·技能提升」），帮助用户直接定位未完成的日常。
2. 需要补充的测试或日志
    - 为 `DailyTaskActivityFullCheck` 补充用例：①「今日活跃度已达上限」→ 不命中；②「通行证等级提升」→ 不命中；③未完成日常提示（如「协议空间·技能提升」）→ 命中并告警；截图按 `tests/` 规范脱敏。
    - 日志侧无需额外埋点：`reco_details.detail.all` 已完整保留 OCR 原文与置信度。
3. 不属于不支持场景，无需限制入口；建议在 issue 回复中向用户说明「理智不足 → 协议空间日常未完成 → 活跃度未满」的因果链。

## 给修复 AI 的建议（可复制）

<details><summary>点击此处展开</summary>

```text
现象：
用户在 v2.35.0-beta.1 全套日常运行后看到红色告警「今日活跃度不满100…」（DailyTaskActivityFullCheck 触发），用户认为活跃度实际已满 100。

关键证据：
- 23:43:11.303 ProtocolSpacePopupSanityLowNoMedicationFinish（StopTask，focus「理智不足，结束任务」）：协议空间因理智不足且无紧急理智药剂提前结束，「协议空间·技能提升」日常未完成。
- 23:43:48.712 DailyTaskActivityFullCheck Node.Recognition.Succeeded（inverse OCR 未命中「今日活跃度已达上限」），ROI [330,520,500,100] 内 OCR 结果为 {"box":[455,560,132,17],"score":0.999852,"text":"协议空间·技能提升"}，随后触发 focus 告警（log+notification）。
- 同一帧 OCR 还读到 3 个「前往」按钮：{"box":[1009,261,...],"text":"前往"}、{"box":[1010,378,...],"text":"前往"}、{"box":[1009,492,...],"text":"前往"}，说明日常任务仍有未完成项。
- 同一心跳 DailyTaskClaimSingleReward / DailyTaskClaimActivityRewards 均失败（无可领奖励），DailyRewards 于 23:44:55 正常 Succeeded。

可能相关线索（待验证）：
1. 本次告警为正确触发；用户看到的 100 可能来自事后手动补做协议空间。
2. 潜在脆弱点：检查依赖日常面板底栏横幅这一多用途提示位；活跃度满 100 时横幅也可能显示其他文案（如「通行证等级提升」），#6365 只覆盖了已知的一种。
3. 改进方向：增加日常标签页渲染确认节点；改用更可靠的活跃度指示（进度条/数值）；无法判定时降级为普通日志；告警文案附带横幅原文。
```

</details>

## 置信度

- **中高**。告警按设计触发、检查时刻横幅为未完成日常提示、同一帧还有 3 个「前往」、37 秒前协议空间因理智不足收工，四条证据互相印证。
- 还缺什么证据：
    - 23:43:48 那一刻的完整现场截图（`save_draw: false`，`on_error/` 只覆盖 23:45:16 的 `SeizeDeliveryJobs`）；
    - 12:20 那次运行的 `maafw` 日志已被滚动覆盖，无法确认当日中午是否完成过「协议空间·技能提升」；
    - 日志包只有 `part01`，若上游还有未上传的后续分包，证据链可能不完整；
    - 用户看到「100」的具体时间点（运行结束后手动补做前 / 后）需要用户确认。

## English translation

<details><summary>Click here to expand</summary>

# Issue #6680 Log Analysis: Log Reports Daily Activity Below 100, but the User Believes It Is Full

## Issue Summary

- Issue: `#6680` (<https://github.com/MaaEnd/MaaEnd/issues/6680>)
- Version / controller: MaaEnd `v2.35.0-beta.1`, MXU (Windows), `Win32-Front` controller, 1920x1080 fullscreen, instance "Full Daily (`6gxwgx1`)"
- Tasks / options involved:
    - `📅 Daily Rewards (DailyRewards)`, option `Warn when activity below 100 (DailyActivityFullWarning)` (enabled by default)
    - Preceding task `⚔️ Protocol Space (ProtocolSpace)`, options `ProtocolSpaceTab = OperatorProgression`, `ProtocolSpaceLevel = 05`, `ProtocolSpaceUseSpMedication = UseMedication`
- Symptom: after the full daily run, a red warning appeared: "Today's activity is below 100. Please check daily reward claiming, or adjust task settings and order!" The user believes the in-game activity was already 100.
- Log package completeness: only `MaaEnd-logs-v2.35.0-beta.1-20261021-002030-part01.zip` is visible in the issue body, comments and timeline; no `part02` or later parts were found. No `.dmp` file is present. The earlier 12:20 run's `maafw` logs were rotated out of the package, which does not affect the analysis of the latest 23:08 reproduction.

## Key Evidence

<details><summary>Click here to expand</summary>

### Reproduction lock-in

- The `maafw` process `Px8428` started at `2026-10-20 23:06:20` (MaaFramework `v5.15.0-beta.1`).
- `mxu-tauri.log`: at `23:08:44`, 25 tasks were submitted at once with `task_id` `200000001`–`200000025`:
    - #19 `ProtocolSpaceSchedule` → `task_id 200000020`
    - #20 `DailyRewardStart` → `task_id 200000021`
    - #24 `MXU_NOTIFY` → `task_id 200000025`
- The user manually called `stop_agent_impl` at `00:00:20`, by which time all 25 tasks had finished.
- The warning fired during `DailyRewards` (`task_id 200000021`); `mxu-tauri.log` has `[MXU_NOTIFY] Notification sent successfully` at `23:43:49`, matching the warning focus `display: ["log","notification"]`.

### `maafw.bak.2026.10.20-23.59.08.617.log` (main evidence)

**① Warning trigger (23:43:48.712, task_id 200000021)**

```
[2026-10-20 23:43:48.711] ... [msg=Node.Recognition.Starting]
[details={"focus":{"Node.Recognition.Succeeded":{"content":"$option.DailyActivityFullWarning.warning",
"display":["log","notification"]}},"name":"DailyTaskActivityFullCheck","reco_id":400014375,"task_id":200000021}]

[2026-10-20 23:43:48.712] ... [OCRer::analyze] DailyTaskActivityFullCheck
[all_results_=[{"box":[455,560,132,17],"score":0.999852,"text":"协议空间·技能提升"}]]
[filtered_results_=[]] [best_result_=null]
[param_.expected=["今日活跃度已达上限", ..., "通行证等级提升", ...]]

[2026-10-20 23:43:48.712] ... pipeline_data.inverse is true, reverse the result
```

- `DailyTaskActivityFullCheck` is an inverse OCR node: it only looks for "Today's activity has reached the cap" (plus traditional Chinese / English / Japanese variants and the "pass level up" variants) inside ROI `[330,520,500,100]`. Not finding it means "activity is not full", which triggers a red warning to log and notification.
- That ROI is exactly the yellow hint banner at the bottom of the daily panel (verified against repository screenshots: the banner spans roughly `x≈387–614, y≈550–570` at 720p, overlapping the OCR box `[455,560,132,17]`).
- In other words, at the check moment the banner read "**协议空间·技能提升**", not "activity maxed" and not "pass level up".

**② The same heartbeat also OCR'd three "前往" (Go) buttons**

```
[2026-10-20 23:43:48.678] ... [all_results_=[
 {"box":[455,560,132,17],"score":0.999852,"text":"协议空间·技能提升"},
 {"box":[990,213,72,21],"score":0.999772,"text":"40~80"},
 {"box":[991,330,70,18],"score":0.999788,"text":"40~80"},
 {"box":[991,445,70,18],"score":0.999759,"text":"40~80"},
 {"box":[991,560,70,18],"score":0.999757,"text":"40~80"},
 {"box":[1009,261,37,23],"score":0.999513,"text":"前往"},
 {"box":[1009,492,46,23],"score":0.998962,"text":"前往"},
 {"box":[1010,378,35,20],"score":0.999691,"text":"前往"}]]
```

- The daily task list still had three "前往" buttons, i.e. tasks still incomplete. When activity is maxed this area no longer offers "前往".
- Combined with the banner text, the game UI at 23:43:48 was showing "activity not full, dailies remain", not "activity maxed".

**③ 37 seconds earlier, Protocol Space ended early due to insufficient sanity (23:43:11, task_id 200000020)**

```
[2026-10-20 23:42:11.077] [msg=Tasker.Task.Starting] [{"entry":"ProtocolSpaceSchedule","task_id":200000020}]

[2026-10-20 23:43:05.023] ... [node_name=ProtocolSpacePopupSanityLowUseSpMedicationPrepare]
[2026-10-20 23:43:11.302] ... [data.name=ProtocolSpacePopupSanityLowNoMedicationFinish] | enter
[2026-10-20 23:43:11.303] ... [Recognizer::direct_hit] ProtocolSpacePopupSanityLowNoMedicationFinish
[2026-10-20 23:43:11.520] ... [Actuator::stop_task] Action: StopTask
[2026-10-20 23:43:11.522] [msg=Tasker.Task.Succeeded] [{"entry":"ProtocolSpaceSchedule","task_id":200000020}]
```

- `ProtocolSpacePopupSanityLowNoMedicationFinish` has desc "no usable medication, end task", action `StopTask`, and focus text "insufficient sanity, task ended".
- `StopTask` is recorded as `Tasker.Task.Succeeded`, so MXU still shows a green success, which easily makes users think Protocol Space finished normally.
- This run used `OperatorProgression` + level 05, i.e. it targeted the "Protocol Space · skill upgrade" daily; it did not complete it due to insufficient sanity.

**④ Claim loop exhausted; no second activity check**

- `23:43:48.705` `DailyTaskClaimSingleReward` failed (no "领取" OCR; best template match only 0.575);
- `23:43:48.7xx` `DailyTaskClaimActivityRewards` failed (no yellow claimable badge via ColorMatch);
- `23:43:49` `DailyTaskComplete`, then `DailyRewards` `Tasker.Task.Succeeded` at `23:44:55`; `SeizeDeliveryJobs` (23:59:08), `CreditShoppingMain` (00:00:19) and `MXU_NOTIFY` (00:00:20) all ended normally.
- The activity check ran only once in this run; there was no second check.

### `mxu-tauri.log`

- `23:43:49` `[MXU_NOTIFY] Notification sent successfully`, matching the warning notification;
- `00:00:20` `stop_agent_impl` — user-initiated stop, not an abnormal exit;
- At `23:08:44` the 25 tasks were submitted in the same order as the "Full Daily" preset (`ProtocolSpace` before `DailyRewards`).

### `go-service.log`

- At `23:42:40` a `PipelineOverride` was applied for `ProtocolSpace` (`SupplyPlanOverride_TACHYON_SCREENING_LATTICE`, 10 nodes);
- No `error` / `warn` / `fail` entries between 23:40 and 23:45; `go-service.stderr.log` is empty;
- Nothing related to the activity warning; not a root cause.

### `on_error/`

- All three screenshots are from `2026.10.20-23.45.16`, for `AutoDeliveryDeliveryMissionNotFound`, `AutoDeliveryOpenDeliveryMission` and `SeizeDeliveryJobsCheckOngoingJob`, i.e. the open-world delivery UI — unrelated to the daily panel / activity check;
- No on-site screenshot was captured at 23:43:48, but the raw OCR results above already reconstruct the UI text at that moment.

### `config/mxu-MaaEnd.json`

- Instance `6gxwgx1`: `controllerName = Win32-Front`, task list contains `ProtocolSpace` and `DailyRewards`;
- `DailyRewards` options: `DailyEmailRewards=True`, `DailyTaskRewards=True`, `DailyClaimDeliveryJobsRewards=True`, `DailyEventRewards=True`, `DailyProtocolPassRewards=True`, `DailyActivityFullWarning=True`;
- `ProtocolSpace` options: `ProtocolSpaceTab=OperatorProgression`, `ProtocolSpaceLevel=05`, `ProtocolSpaceUseSpMedication=UseMedication`;
- Consistent with the user's description ("full daily, warning enabled by default").

### Code evidence

- `DailyTaskActivityFullCheck` (inverse OCR + red warning + default `enabled: false`) and `DailyTaskEnterTab` (click daily tab, `post_wait_freezes: 100`, check starts ~0.7s later):
  <https://github.com/MaaEnd/MaaEnd/blob/12d3d3c4bcb9e15dcbd03e500017a757627ed6eb/assets/resource/pipeline/DailyRewards/Tasks.json#L25-L54>
  <https://github.com/MaaEnd/MaaEnd/blob/12d3d3c4bcb9e15dcbd03e500017a757627ed6eb/assets/resource/pipeline/DailyRewards/Tasks.json#L215-L259>
- `DailyActivityFullWarning` switch (`default_case: "Yes"`, enables the node via `pipeline_override`):
  <https://github.com/MaaEnd/MaaEnd/blob/12d3d3c4bcb9e15dcbd03e500017a757627ed6eb/assets/tasks/DailyRewards.json#L33-L52>
- Warning text:
  <https://github.com/MaaEnd/MaaEnd/blob/12d3d3c4bcb9e15dcbd03e500017a757627ed6eb/assets/locales/interface/zh_cn.json#L1525-L1526>
- Protocol Space sanity-out node:
  <https://github.com/MaaEnd/MaaEnd/blob/12d3d3c4bcb9e15dcbd03e500017a757627ed6eb/assets/resource/pipeline/ProtocolSpace/InSpace.json#L190-L196>
- The warning feature was added by <https://github.com/MaaEnd/MaaEnd/pull/6358>; <https://github.com/MaaEnd/MaaEnd/pull/6365> fixed the false positive caused by the "pass level up" banner. `v2.35.0-beta.1` already contains both changes.

</details>

## Root Cause

- Direct conclusion: **the warning is not a false positive; it fired correctly by design.** The user's claim that "activity was actually 100" does not match the log evidence at 23:43:48.
- Evidence chain:
    1. At `23:43:11.303`, `⚔️ Protocol Space (ProtocolSpace)` hit `ProtocolSpacePopupSanityLowNoMedicationFinish` → `StopTask` ("insufficient sanity, task ended"), so the "Protocol Space · skill upgrade" daily was **not completed** in this run;
    2. At `23:43:48.712`, `DailyTaskActivityFullCheck` did **not** see "activity maxed" in the daily-panel bottom banner ROI; it saw "协议空间·技能提升", and the same frame also contained three "前往" buttons — daily tasks were still incomplete;
    3. In the same heartbeat neither single-task rewards nor activity rewards were claimable, so the claim loop was properly exhausted; nothing was missed;
    4. Therefore the game UI at check time really was "activity not full", and the warning content was accurate.
- Relation to the MaaEndBot AI analysis: the direction agrees (correct trigger, not a false positive). This report adds two pieces of reinforcement:
    - Repository screenshots confirm `[330,520,500,100]` is exactly the yellow hint banner area, so "协议空间·技能提升" is a **bottom banner recommendation / destination hint**, not a task-list row;
    - The same frame also contained three "前往" buttons, which rules out the "banner caught mid-transition / not yet rendered" false-positive path (both text and buttons were recognized stably with high confidence).
- A possible explanation for the user's "activity 100": the user likely manually completed the Protocol Space skill upgrade after the run ended at `00:00:20`, or viewed it at another time; the logs do not support "already 100 at 23:43:48".
- Potential design fragility (**not proven as the root cause here**): the check relies on the bottom banner, a multi-purpose hint area. Repository screenshots show that even at activity 100 the banner can display "pass level up, go check rewards" (#6365 fixed that known case), so the banner is not a dedicated activity indicator. Any other banner text could cause a false positive when activity is actually full. In this log, however, the banner showed an incomplete-daily recommendation, so this is not the root cause here.

## Advice for the User

- What the user can do right now:
    - Check sanity and sanity-medication stock. In this run `⚔️ Protocol Space` ended early because sanity was insufficient and no emergency sanity booster was available, leaving the `Protocol Space · skill upgrade` daily undone. This is the direct reason activity was below 100.
    - Check the in-game daily task list and confirm whether `Protocol Space · skill upgrade` is still in the "go / incomplete" state.
    - Look at the run log around 23:43; about 37 seconds before the red warning, "insufficient sanity, task ended" should appear.
- Upgrade / reinstall / resync resources / reset config: **none needed**; the current behavior matches the design.
- Wait for a developer fix: not for this symptom. If false positives are still suspected later, watch for future hardening of `DailyTaskActivityFullCheck`.
- Temporary workaround: switch `Warn when activity below 100` to No under `📅 Daily Rewards` (`DailyActivityFullWarning = No`), but this is not recommended because it also removes genuine missing-reward alerts. The recommended practice is to keep enough sanity/medication for `⚔️ Protocol Space` and keep it before `📅 Daily Rewards` (the current "Full Daily" order already does this).

## Fix Plan

1. Code / Pipeline / config fixes
    - Add a precondition to `DailyTaskActivityFullCheck` that the daily tab page is fully rendered (e.g. template/OCR confirmation of the selected daily tab, or a re-check of a list anchor). If it cannot be confirmed, treat the check as "undetermined" and degrade to a normal log instead of a red warning.
    - Consider replacing the single banner-text criterion with a more reliable activity indicator (e.g. the left activity progress bar / `100/100` numeric area), using the banner only as supporting evidence.
    - Include the detected banner text (e.g. "协议空间·技能提升") in the warning message to help users locate the unfinished daily.
2. Tests / logging to add
    - Add cases for `DailyTaskActivityFullCheck`: ① "activity maxed" → no hit; ② "pass level up" → no hit; ③ incomplete-daily hint (e.g. "协议空间·技能提升") → hit and warn. Desensitize screenshots per `tests/` conventions.
    - No extra logging needed: `reco_details.detail.all` already preserves the raw OCR text and confidence.
3. This is not an unsupported scenario, so no entry restriction is needed; in the issue reply, explain the causal chain "insufficient sanity → Protocol Space daily not completed → activity below 100".

## Confidence

- **Medium-high.** The warning fired by design, the banner showed an incomplete-daily hint, three "前往" buttons were present in the same frame, and Protocol Space had stopped 37 seconds earlier due to insufficient sanity — four mutually corroborating pieces of evidence.
- What is still missing:
    - A full screenshot at 23:43:48 (`save_draw: false`; `on_error/` only covers `SeizeDeliveryJobs` at 23:45:16);
    - The 12:20 run's `maafw` logs were rotated out, so it cannot be confirmed whether "Protocol Space · skill upgrade" was completed at noon;
    - Only `part01` is available; if further parts were uploaded, the evidence chain could be incomplete;
    - The exact time the user saw "100" (before / after manual completion) needs user confirmation.

</details>
