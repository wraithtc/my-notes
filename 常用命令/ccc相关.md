# ccc - Claude Code 模型快速切换器

基于 `ccm.sh` 的 Claude Code 模型切换启动器，一条命令切换不同 AI 模型并启动 Claude Code。

- 启动器: `/home/tongchao/.local/bin/ccc`
- 核心脚本: `/home/tongchao/.local/share/ccm/ccm.sh`
- 配置文件: `~/.ccm_config`（API Key 等配置）
- 账号文件: `~/.ccm_accounts`（Claude Pro 账号管理）

---

## 基本用法

```bash
ccc <model> [region|variant] [claude-options]
```

## 支持的模型

### 官方模型

| 命令 | 模型 | API 端点 | 上下文窗口 |
|------|------|---------|-----------|
| `ccc deepseek` / `ccc ds` | deepseek-chat (V4) | api.deepseek.com/anthropic | **1M** |
| `ccc glm` / `ccc glm5` | glm-5 | api.z.ai/api/anthropic | **200K** |
| `ccc glm china` | glm-5 | open.bigmodel.cn/api/anthropic | **200K** |
| `ccc kimi` / `ccc kimi2` | kimi-k2.5 | api.moonshot.ai/anthropic | **128K** |
| `ccc kimi china` | kimi-k2.5 | api.moonshot.cn/anthropic | **128K** |
| `ccc qwen` | qwen3-max | coding-intl.dashscope.aliyuncs.com | **128K** |
| `ccc qwen china` | qwen3-max | coding.dashscope.aliyuncs.com | **128K** |
| `ccc minimax` / `ccc mm` | MiniMax-M2.5 | api.minimax.io/anthropic | **1M** |
| `ccc minimax china` | MiniMax-M2.5 | api.minimaxi.com/anthropic | **1M** |
| `ccc seed` / `ccc doubao` | ark-code-latest | ark.cn-beijing.volces.com/api/coding | **128K** |
| `ccc claude` / `ccc sonnet` / `ccc s` | Claude Sonnet 4.5 | api.anthropic.com | **200K** |

### 豆包 Seed-Code 变体

```bash
ccc seed              # 默认 (ark-code-latest)
ccc seed doubao       # doubao-seed-code
ccc seed glm          # glm-5 (通过豆包)
ccc seed deepseek     # deepseek-v3.2 (通过豆包)
ccc seed kimi         # kimi-k2.5 (通过豆包)
```

### OpenRouter 模型

```bash
ccc open <provider>   # 通过 OpenRouter 路由
ccc open kimi         # 例: OpenRouter 上的 Kimi
```

### Claude Pro 账号切换

```bash
ccc <account_name>              # 切换到已保存的账号并启动
ccc claude:<account_name>       # 切换账号 + 使用 Claude 模型
```

---

## 常用示例

```bash
# 最常用 - 直接启动不同模型
ccc deepseek                      # DeepSeek V4
ccc glm                           # GLM-5 (默认 global)
ccc glm china                     # GLM-5 (国内端点)
ccc claude                        # Claude Sonnet 4.5

# 传递 Claude Code 参数
ccc deepseek --dangerously-skip-permissions
ccc glm --resume

# 切换 Claude Pro 账号
ccc woohelps                      # 切换到 woohelps 账号
ccc claude:work                   # 切换到 work 账号 + Claude 模型
```

---

## ccm 命令行工具

`ccm` 是底层工具，`ccc` 在其上封装。ccm 也可直接使用：

```bash
# 环境变量输出 (推荐用 eval)
eval "$(ccm deepseek)"           # 设置环境变量
eval "$(ccm glm china)"          # 设置 GLM 国内环境

# 用户级设置 (写入 ~/.claude/settings.json，最高优先级)
ccm user glm global              # 全局使用 GLM
ccm user deepseek                # 全局使用 DeepSeek
ccm user reset                   # 移除用户级设置，恢复环境变量控制

# 项目级设置 (写入 .claude/settings.local.json)
ccm project glm global           # 当前项目使用 GLM
ccm project reset                # 移除项目覆盖

# 账号管理
ccm save-account <name>          # 保存当前 Claude Pro 账号
ccm switch-account <name>        # 切换到已保存的账号
ccm list-accounts                # 列出所有保存的账号
ccm delete-account <name>        # 删除已保存的账号
ccm current-account              # 显示当前账号信息

# 其他
ccm status                       # 显示当前配置
ccm config                       # 编辑配置文件
ccm update-config                # 更新模型 ID 到最新默认值
```

---

## 配置

### API Key 配置

在 `~/.ccm_config` 中设置（首次运行 `ccm config` 生成模板）：

```bash
DEEPSEEK_API_KEY=sk-xxx
GLM_API_KEY=xxx.xxxx
KIMI_API_KEY=sk-xxx
QWEN_API_KEY=sk-xxx
MINIMAX_API_KEY=xxx
ARK_API_KEY=xxx              # 豆包 Seed-Code
STEPFUN_API_KEY=sk-xxx
OPENROUTER_API_KEY=sk-or-xxx
CLAUDE_API_KEY=sk-ant-xxx    # Claude Pro (OAuth 优先)
```

### 模型名覆盖

可通过环境变量覆盖默认模型名：

```bash
export DEEPSEEK_MODEL=deepseek-v4-pro    # 默认: deepseek-chat
export GLM_MODEL=glm-5.1                 # 默认: glm-5
export CLAUDE_MODEL=claude-opus-4-6      # 默认: claude-sonnet-4-5-20250929
```

### 上下文窗口

`ccc` 启动时自动设置 `CLAUDE_CODE_MAX_CONTEXT_TOKENS`，控制 autocompact 触发时机：

| 模型 | 上下文窗口 | 说明 |
|------|-----------|------|
| DeepSeek | 1,000,000 | DeepSeek V4 Pro/Flash |
| GLM-5/5.1 | 204,800 | 约 200K |
| Claude | 200,000 | Sonnet/Opus/Haiku |
| MiniMax M2.5 | 1,000,000 | 1M |
| Kimi K2.5 | 131,072 | 128K |
| Qwen | 131,072 | 128K |
| Seed/Doubao | 131,072 | 128K |
| StepFun | 131,072 | 128K |

切换模型时自动生效，无需手动设置。

---

## 环境变量一览

`ccc` 启动时 ccm 设置的关键环境变量：

| 变量 | 作用 |
|------|------|
| `ANTHROPIC_MODEL` | 当前使用的模型 |
| `ANTHROPIC_BASE_URL` | API 端点地址 |
| `ANTHROPIC_AUTH_TOKEN` | API Key |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Sonnet 模型名 |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Opus 模型名 |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Haiku 模型名 |
| `CLAUDE_CODE_SUBAGENT_MODEL` | 子代理使用的模型 |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | 上下文窗口大小（控制 autocompact） |

---

## 优先级

设置生效的优先级（从高到低）：

1. **命令行参数** — `ccc <model>`
2. **用户级设置** — `ccm user <model>` (写入 `~/.claude/settings.json`)
3. **项目级设置** — `ccm project <model>` (写入 `.claude/settings.local.json`)
4. **环境变量** — `export ANTHROPIC_MODEL=xxx` (在 `.bashrc` 中)
5. **默认值** — ccm 内置默认

---

## 启动信息

`ccc` 启动时会显示当前配置：

```
🚀 Launching Claude Code...
   Model: deepseek-chat
   Base URL: https://api.deepseek.com/anthropic
   Context: 1000000
```
