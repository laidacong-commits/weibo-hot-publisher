# 运行环境对接

仅在实际运行、查状态或改定时时读取。本技能提供操作规则，不包含完整运行程序、登录会话或凭据，也不会自动创建定时任务。以下命令对接已有的兼容 Windows 运行项目；没有该项目时，需要先准备采集、去重、发布和回查能力。

## 先定位并检查

先确认用户已有运行项目的位置，不从示例路径推断项目存在。路径示例：

```text
D:\Projects\auto-weibo-agent
```

程序入口：`scheduled.py` 采集热榜并调度；`auto_pipeline.py` 选题、核实、图片检查和发布；`editorial.py` 文案排版；`publish_workflow.py` 与 `publish_guard.py` 防重复、保留不明结果；`live_transport.py` 网页发布及核验。Chrome 登录由浏览器保管。

任务名以用户实际配置为准；下文用 `WeiboHotPublisher` 作示例，使用前替换为实际名称。建议 UTC+8 每日 10:00–18:30 每半小时触发。必须检查真实入口与模式，Skill 中写有时间不代表任务已经创建或当前有效。

在此项目目录执行前，检查 `data/STOP`、`publish_attempts` 中的 `uncertain`、任务是否正在运行及是否已有手动试跑。避免并行打开同一个发布流程或干扰已运行任务；不要清空去重库重新试发。

## 查看状态

```powershell
$env:PYTHONIOENCODING='utf-8'
Get-ScheduledTask -TaskName 'WeiboHotPublisher' |
  Select-Object TaskName,State
Get-ScheduledTask -TaskName 'WeiboHotPublisher' |
  Get-ScheduledTaskInfo |
  Select-Object LastRunTime,LastTaskResult,NextRunTime
Get-Content -Encoding UTF8 .\data\agent.log -Tail 35
py -3.12 .\agent.py status
```

`查看运行进度.cmd` 会持续显示日志。日志展示选题、观点概要、来源、操作与结果，不是模型内部思维链。核对时只读必要行，不遍历 Chrome 会话、Cookie 或密钥文件。

## 单次运行

以下命令在已定位项目目录执行。发布命令只在本次用户要求发布，或本会话已有适用的自动发布授权时使用。

草稿及图片上传预演，**不点击发送**：

```powershell
$env:PYTHONIOENCODING='utf-8'
py -3.12 -c 'import scheduled,auto_pipeline; scheduled.collect_hot(); print(auto_pipeline.run(dry_run=True))'
```

单次正式流程，沿用项目默认模型设置：

```powershell
py -3.12 -c 'import scheduled,auto_pipeline; scheduled.collect_hot(); print(auto_pipeline.run())'
```

本轮用户明确指定 Luna 极高时：

```powershell
py -3.12 -c 'import scheduled,auto_pipeline; scheduled.collect_hot(); print(auto_pipeline.run(model="gpt-6-luna",effort="xhigh"))'
```

先读代码确认参数仍受支持。实时运行不使用 mock 或改写采集时间去通过过期校验。轮询运行进展时控制输出，至少约每分钟向用户说明真实阶段；不要为追求成功无限开启新轮。

停止或恢复：

```powershell
py -3.12 .\agent.py stop
py -3.12 .\agent.py resume
```

只有用户要求或处理适用的故障时使用。恢复前确认不是用户主动停发，且不明发布记录已核对；`resume` 不会清除不明发布记录。`agent.py run` 仅处理旧模拟队列，不能拿它充当真实发帖。

## 返回结果

| 结果 | 解释与处理 |
| --- | --- |
| `published` | 程序取得发布证据；再从后台记录读取真实链接，必要时打开微博确认 |
| `preview` | 正文与图片上传预演完成，未发出 |
| `skipped` | 没有选出合格内容，查日志具体原因 |
| `budget_exhausted` | 本轮 Codex 启动上限不足以完成余下步骤，不能当成成功 |
| `quota_exhausted` | 用量或请求限额已到，结束本轮，不继续同一模型的备用路径 |
| `stopped` / `blocked` | 停发或被检查拦截，先查原因 |
| `uncertain` | 可能已经发送；保持停发，先查主页，禁止再点发送 |

只有实际帖子链接、正确账号下的正文和图片证据能确认发帖。数据库中手动写入一条 URL、本地程序返回码 0、成功生成文件，都不能独立证明发布成功。

## 已知限制与排错入口

- 现有采集或筛选代码曾因要求“搜索链接必须含完全一致的井号话题”漏掉可用候选。看到用户热榜截图与采集名单不一致时，检查 `research_draft.has_exact_hashtag_link`、榜单原始条目和 `produce` 的筛选；不能凭技能文字宣称这个问题已修好。
- `research_draft.recent_topics()` 同时含已发正文和近期草稿。按“已发”和“仅草稿”分别处理，不能用整个序列化字符串的包含关系封禁相关主题。先查事实、时间与观点是否重复。
- 当前程序可能只保留前两条成功核实的短引文。正文中的其他事实也必须得到支持，必要时读取同页上下文或补充来源，不能因网页可访问就全部放行。
- 默认排版脚本比自然写作建议严格。若用户改变格式，先检查脚本限制，避免写出必然被拦截的文案后反复耗模型修稿。
- 当前自动发布链路用网页，不代表官方 Weibo CLI 的认证、服务开通或发布 API 权限已完成。需要官方接口时重新核实真实文档与认证要求，不从网页登录推导 API 权限。
- 安装、静态检查、模拟通过、登录和一次人工发帖，均不等于无人值守全链路已验证。修改模型或流程后，把测试与真实运行结果分别报告。
- 修改或验证运行程序时使用隔离的数据目录，确认 `AUTO_WEIBO_DATA` 在导入 `agent` 之前生效。不要让测试日志、模拟发布记录或停发文件进入运营数据目录。

## 改定时任务

先读取现有任务的入口、触发器、时区、下次时间和实例策略，再按用户要求修改，保留未要求变更的设置。现有本机方案优先复用现有 Windows 任务；若用户要求在 Codex 中创建或管理自动化，先发现并使用 app 的 `automation_update` 工具。两种方案不要重复建立导致同一时段双发。

不把“下次计划时间”说成一定会发：关机、休眠、断网、退出登录、平台验证、来源不足、素材或额度限制都可能阻止本轮发布。这里只记录与实际状态有关的原因，不为每轮增加泛化审核流程。
