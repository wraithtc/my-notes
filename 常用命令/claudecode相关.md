# Claude Code 常用操作

## 安装与更新

```bash
npm install -g @anthropic-ai/claude-code    # 安装
npm update -g @anthropic-ai/claude-code     # 更新
claude --version                             # 查看版本
```

## 认证

```bash
claude auth login          # OAuth 登录（Claude Pro 订阅）
```

全新环境先执行 `claude auth login`，打开代理后点击登录链接。

## 模型切换

使用 [[ccc相关|ccc]] 命令快速切换模型，详见 [[ccc相关]]。

```bash
ccc deepseek         # DeepSeek V4
ccc glm              # GLM-5
ccc claude           # Claude Sonnet 4.5
```

## 手动设置模型（不用 ccc）

在 `.bashrc` 中设置（注意 API Key 用 `ANTHROPIC_AUTH_TOKEN`）：

```bash
export ANTHROPIC_AUTH_TOKEN="your-api-key"
export ANTHROPIC_BASE_URL="https://open.bigmodel.cn/api/anthropic"
```

或写入 `~/.claude/settings.json`：

```json
{
  "env": {
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-4.5-air",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-5",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-5"
  },
  "skipDangerousModePermissionPrompt": true
}
```

## 常用参数

```bash
claude --dangerously-skip-permissions   # 跳过权限确认
claude --resume                          # 恢复上次会话
claude --model <model>                   # 指定模型
claude --print "prompt"                  # 非交互模式
```

## 会话内命令

| 命令 | 作用 |
|------|------|
| `/compact` | 手动压缩上下文 |
| `/clear` | 清空对话 |
| `/context` | 查看上下文使用情况 |
| `/config` | 打开设置 |
| `/autocompact` | 配置自动压缩窗口大小 |
| `/help` | 帮助 |

## Autocompact（自动压缩）

Claude Code 在上下文接近窗口上限时自动压缩。使用 `ccc` 启动时会自动设置正确的上下文窗口大小（`CLAUDE_CODE_MAX_CONTEXT_TOKENS`），无需手动配置。

手动调整：

```bash
# 设置自动压缩窗口（100K-1M）
export CLAUDE_CODE_AUTO_COMPACT_WINDOW=950000

# 禁用自动压缩（仍可手动 /compact）
export DISABLE_AUTO_COMPACT=true
```

## 状态栏插件 (claude-hud)

状态栏显示模型名、上下文使用率、用量等信息。

- 插件位置: `~/.claude/plugins/cache/claude-hud/claude-hud/0.0.11/`
- 配置文件: 插件目录下 `config.json`
- 已配置模型上下文窗口映射（DeepSeek=1M, GLM=200K 等）

### claude-hud 配置示例

```json
{
  "display": {
    "showModel": true,
    "showContextBar": true,
    "contextValue": "tokens",
    "showUsage": true,
    "showDuration": true
  }
}
```

`contextValue` 可选: `percent` / `tokens` / `remaining` / `both`
