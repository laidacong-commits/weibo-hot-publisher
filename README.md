# 微博热榜观点发布 Skill

让 Codex 从微博热搜中挑出值得评论的事件，写短句观点、找事件图片，并在用户授权后发布与回查。

## 主要规则

- 汽车及产业优先，其次出行和用车，再看其他有具体争议或消费影响的热点。
- 根据真实热榜与来源判断；不把旧草稿、失败尝试或同品牌新闻一概当成重复内容。
- 开头给判断，用一两个已核实事实解释；默认正文约 50–120 字、4–7 行，不留空行，话题放最后。
- 使用对应事件、产品或车型的真实图片；有覆盖水印就换图，不移除水印。
- 用户授权后自动发布，发送前记录尝试，发出后核对账号、正文、图片和实际微博链接。
- 发送结果不明时停发并回查，防止重复发布。
- 建议 Luna 中等推理；明确指定时可用极高。默认每轮最多四次 Codex 启动，重试也计入。

## 安装到 Codex

需要已安装 Git。下面会把仓库安装到用户的 Skills 目录，已有同名目录时先停止，避免覆盖。

### Windows PowerShell

```powershell
$skillPath = Join-Path $env:USERPROFILE '.codex\skills\weibo-hot-publisher'
if (Test-Path -LiteralPath $skillPath) { throw '同名 Skill 已存在，请先检查，不要覆盖' }
git clone https://github.com/laidacong-commits/weibo-hot-publisher.git $skillPath
```

### macOS / Linux

```sh
skill_path="$HOME/.codex/skills/weibo-hot-publisher"
if [ -e "$skill_path" ]; then
  echo "同名 Skill 已存在，请先检查，不要覆盖"
else
  git clone https://github.com/laidacong-commits/weibo-hot-publisher.git "$skill_path"
fi
```

安装后，在后续 Codex 对话中调用 `$weibo-hot-publisher`。若当前对话尚未识别新技能，可重新打开对话。

## 调用示例

只写草稿：

```text
$weibo-hot-publisher 从当前微博热榜选择汽车或出行话题
写一条短观点，话题放末尾，先不发布
```

跑一轮并发布：

```text
$weibo-hot-publisher 用 Luna 极高跑一轮
从实时热榜选题、核实、写观点、配事件图
核验通过后发布到我已确认的微博账号，并给我实际链接
```

设置或检查定时运营：

```text
$weibo-hot-publisher 检查现有定时任务
按 UTC+8 每天 10:00–18:30 每半小时检查热搜
沿用我的自动发布授权，避免重复建立任务
```

## 仓库内容

- [SKILL.md](SKILL.md)：选题、写作、素材、发布与用量规则。
- [agents/openai.yaml](agents/openai.yaml)：Codex 界面名称与默认调用提示。
- [references/editorial-examples.md](references/editorial-examples.md)：观点角度与示例，示例不能当作已核实新闻直接发布。
- [references/local-runtime.md](references/local-runtime.md)：兼容运行项目的对接说明与排错入口。

## 能力边界

这是一份可安装的 Skill 规则集，**不是开箱即用的微博发布程序，也不包含官方 Weibo CLI 或登录凭据**。实际采集与发布需要当前 Codex 环境中可用的浏览器能力，或另外部署兼容的运行项目。参考文档中的 Python 命令依赖外部项目，不能仅克隆本仓库后直接运行。

定时执行依赖外部调度器，创建 Skill 不会自动产生定时任务。账号须由使用者本人登录和授权；不绕过验证码、风控或访问限制。结构检查通过不代表无人值守发布已通过实测。

四次 Codex 启动上限不等于严格的 token 或费用上限；一次启动内部仍可能包含多次模型请求。
