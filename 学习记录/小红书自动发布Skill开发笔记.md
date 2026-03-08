# 小红书自动发布 Skill 开发笔记

> 开发时间: 2026-03-08
> 状态: 已完成

## 项目背景

开发一个小红书自动发布的 skill，实现自动登录和发布笔记功能。

## 技术栈

- **Python 3.12** + **Playwright** (浏览器自动化)
- **WSL2** (开发环境)
- **Windows Edge** (远程浏览器，通过 CDP 连接)

## 核心问题与解决方案

### 问题1: WSL2 与 Windows IP 不一致导致 Cookie 失效

**现象**: 在 Windows 上登录获取的 Cookie，在 WSL2 中使用时因为 IP 不同而被小红书服务器拒绝。

**尝试过的方案**:
1. ❌ WSL2 镜像网络模式 - Windows Build 19045.6466 不支持
2. ❌ 代理 - 导致 CDN 连接超时 60+ 秒，JS 无法执行

**最终方案**: Playwright CDP 连接到 Windows Edge

```
WSL2 (Python/Playwright) --CDP--> Windows Edge (实际浏览器)
```

这样浏览器运行在 Windows 上，IP 一致，Cookie 有效。

### 问题2: WSL2 无法访问 Windows localhost

**现象**: Edge 监听 `127.0.0.1:9222`，WSL2 无法直接访问。

**解决方案**:
1. Windows 端口转发 (管理员 PowerShell):
```powershell
netsh interface portproxy add v4tov4 listenport=9223 listenaddress=0.0.0.0 connectport=9222 connectaddress=127.0.0.1
```

2. Windows 防火墙规则:
```powershell
New-NetFirewallRule -DisplayName "WSL Edge Debug 9223" -Direction Inbound -LocalPort 9223 -Protocol TCP -Action Allow
```

3. WSL2 使用 Windows 主机 IP:
```bash
# Windows 主机 IP
172.27.0.1

# CDP 连接地址
http://172.27.0.1:9223
```

### 问题3: CDP 模式下文件上传失败

**现象**: CDP 远程浏览器无法访问 WSL2 本地文件路径。

**解决方案**: 使用 Playwright 的 buffer/payload 方式上传

```python
# 读取文件内容为 bytes
with open(abs_path, 'rb') as f:
    file_content = f.read()

# 使用 buffer 方式上传
await upload_input.set_input_files(
    files=[{
        "name": file_name,
        "mimeType": mime_type,
        "buffer": file_content  # raw bytes
    }]
)
```

### 问题4: 发布流程点击"上传图文"标签失败

**现象**: 元素在 viewport 外，Playwright click 失败。

**解决方案**: 使用 JavaScript 直接点击

```python
clicked = await page.evaluate('''() => {
    const tabs = document.querySelectorAll('.creator-tab');
    for (const tab of tabs) {
        if (tab.textContent.includes('上传图文') && !tab.classList.contains('active')) {
            tab.click();
            return true;
        }
    }
    return false;
}''')
```

## 发布流程

小红书创作者中心发布页面的正确流程：

```
1. 访问 /publish/publish?source=official
2. 点击「上传图文」标签 (默认是「上传视频」)
3. 上传图片
4. 页面自动跳转到编辑器
5. 填写标题
6. 填写正文
7. 点击发布按钮
```

### 关键选择器

| 元素 | 选择器 |
|------|--------|
| 上传图文 Tab | `.creator-tab:has-text("上传图文")` |
| 文件上传 | `input[type="file"]` |
| 标题输入 | `input[placeholder*="填写标题"]` |
| 正文输入 | `div[contenteditable="true"]` |
| 发布按钮 | `button:has-text("发布")` |

## 使用方法

### 1. 在 Windows 上启动 Edge 远程调试

```powershell
# 先关闭所有 Edge
taskkill /F /IM msedge.exe

# 启动调试模式
& "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" --remote-debugging-port=9222 --user-data-dir="C:\temp\edge-debug"
```

### 2. 在 WSL2 中运行

```bash
cd /home/tongchao/clawd/skills/xiaohongshu-auto
source .venv/bin/activate

# 登录（首次使用）
python scripts/publish.py login --cdp http://172.27.0.1:9223

# 发布笔记
python scripts/publish.py publish \
    -t "标题" \
    -c "内容" \
    -i image1.jpg image2.jpg \
    --cdp http://172.27.0.1:9223
```

## 项目结构

```
/home/tongchao/clawd/skills/xiaohongshu-auto/
├── lib/
│   ├── xhs_client.py    # 小红书客户端核心逻辑
│   ├── browser.py       # Playwright 浏览器管理
│   ├── cookie_manager.py # Cookie 管理
│   └── database.py      # SQLite 数据库
├── scripts/
│   ├── publish.py       # 发布脚本
│   └── start_edge_debug.bat  # Windows 启动脚本
├── data/
│   └── test_images/     # 测试图片
├── config.json          # 配置文件
└── session.json         # Cookie 存储
```

## 经验总结

1. **WSL2 网络问题**: WSL2 默认使用 NAT 模式，与 Windows IP 不同。需要通过端口转发或使用 Windows 主机 IP 解决。

2. **Playwright CDP 模式**: 连接远程浏览器时，文件操作需要在客户端处理（读取为 buffer），不能依赖文件路径。

3. **小红书反爬**: 使用真实浏览器 + CDP 连接比无头模式更不容易被检测。

4. **页面元素定位**: 复杂 UI 的元素可能在 viewport 外，使用 JavaScript 点击更可靠。

---

#小红书 #自动化 #Playwright #WSL2 #CDP
