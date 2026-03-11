全新环境先执行claude auth login,打开代理的点击登录链接
是设置国内的模型到bashrc，注意apikey要用ANTHROPIC_AUTH_TOKEN    ！！！
export ANTHROPIC_AUTH_TOKEN="04fb5a83afea44ba87cbd2e82566237c.EkObWWaFJwCaoudx"
export ANTHROPIC_BASE_URL="https://open.bigmodel.cn/api/anthropic"
# Wrote /home/tongchao/.claude/settings.json
{
  "env": {
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-4.5-air",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-5",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-5"
  },
  "skipDangerousModePermissionPrompt": true
}

claude --dangerously-skip-permissions   //跳过权限检查

