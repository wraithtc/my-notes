proxy codex --search --dangerously-bypass-approvals-and-sandbox  -C ~/dev/task/paper-write/

 所以你重启后正确的恢复方式应该是：

  cd /home/tongchao/dev/task/aipet/pet-id
  codex resume --last

  如果还看不到，就直接用 id：

  codex resume 019d4826-4690-71a1-ae4f-c6b98b83afd8

  如果你不在原目录，要用：

  codex resume --all

  或者：

  codex -C /home/tongchao/dev/task/aipet/pet-id resume --last

  这次我还验证到一个关键点：

  - codex resume --last 在非交互环境里会直接报 Error: stdin is not a terminal

  所以如果你是在脚本、某些 IDE 内置终端代理、非标准 shell 面板里跑，也会失败。