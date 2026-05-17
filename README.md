# GitHub Trending 日报自动化

每天自动抓取 GitHub Trending 热门项目，调用 DeepSeek API 生成中文摘要，保存日报到本地并发送邮件。

## 触发条件

当用户要求执行以下任一操作时，加载此 skill：
- "执行 GitHub Trending 日报"
- "生成今日 GitHub 热门项目日报"
- "抓取 GitHub Trending"
- "运行 trending 日报脚本"

## 前置依赖

```bash
pip install openai beautifulsoup4 requests --break-system-packages
```

## 工作流

### 第一步：确认配置

在执行前，确认以下配置项。如果用户未提供，使用默认值或询问：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| DeepSeek API Key | sk-64def66e3fd449608ec95fa477c204fc | 用于生成中文摘要 |
| QQ 邮箱账号 | 3125022941@qq.com | 发件/收件邮箱 |
| SMTP 授权码 | yilrpkozeuuxddjc | QQ 邮箱 SMTP 授权码 |
| 日报保存路径 | /workspace/trending_YYYY-MM-DD.md | Markdown 日报文件 |
| 封面图 Prompt 路径 | /workspace/cover_prompt_YYYY-MM-DD.json | 封面图生成 prompt |

### 第二步：创建/更新脚本

将以下脚本保存到 `/data/user/work/github_trending_auto.py`：

```python
import json
import os
import socket
import ssl
import base64
import requests
from bs4 import BeautifulSoup
from openai import OpenAI
from datetime import date
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

# ============ 配置区 ============
DEEPSEEK_API_KEY = "sk-64def66e3fd449608ec95fa477c204fc"
SENDER_EMAIL = "3125022941@qq.com"
SENDER_PASSWORD = "yilrpkozeuuxddjc"  # SMTP 授权码
RECEIVER_EMAIL = "3125022941@qq.com"
# ================================

# --- 第一步：抓取 GitHub Trending 页面 ---
def fetch_trending():
    url = "https://github.com/trending?since=daily"
    headers = {"User-Agent": "Mozilla/5.0"}
    response = requests.get(url, headers=headers)
    response.raise_for_status()
    return response.text

# --- 第二步：解析 HTML，提取项目信息 ---
def parse_repos(html):
    soup = BeautifulSoup(html, "html.parser")
    repos = []
    for article in soup.select("article.Box-row"):
        link = article.select_one("h2 a")
        name = link["href"].strip("/") if link else "unknown"
        desc_tag = article.select_one("p.col-9")
        desc = desc_tag.text.strip() if desc_tag else "无描述"
        lang_tag = article.select_one('span[itemprop="programmingLanguage"]')
        lang = lang_tag.text.strip() if lang_tag else ""
        stars_tag = article.select_one("span.d-inline-block.float-sm-right")
        stars = stars_tag.text.strip() if stars_tag else ""
        repos.append(f"- {name} ({lang}) {stars}\n  {desc}")
    return repos

# --- 第三步：调用 DeepSeek API 生成中文摘要 ---
def summarize(repos):
    client = OpenAI(
        api_key=DEEPSEEK_API_KEY,
        base_url="https://api.deepseek.com"
    )
    repo_text = "\n".join(repos[:15])
    response = client.chat.completions.create(
        model="deepseek-chat",
        messages=[{
            "role": "user",
            "content": f"下面是今天 GitHub Trending 的热门项目列表，"
                       f"请用中文写一份简短的日报摘要，"
                       f"挑出最值得关注的 5 个项目，"
                       f"简要说明每个项目是做什么的、为什么值得关注。"
                       f"\n\n{repo_text}"
        }]
    )
    return response.choices[0].message.content

# --- 第四步：保存日报到本地 ---
def save_report(summary):
    filename = f"/workspace/trending_{date.today()}.md"
    with open(filename, "w", encoding="utf-8") as f:
        f.write(f"# GitHub Trending 日报 ({date.today()})\n\n")
        f.write(summary)
    print(f"日报已保存到 {filename}")
    return filename

# --- 第五步：生成封面图 prompt JSON ---
def extract_repo_names(repos, n=5):
    names = []
    for line in repos[:n]:
        first = line.split("\n", 1)[0]
        name = first.lstrip("- ").split(" ", 1)[0]
        names.append(name)
    return names

def build_image_prompt(repo_names, today):
    center = repo_names[0] if repo_names else ""
    corners = repo_names[1:5] + [""] * (4 - len(repo_names[1:5]))
    lt, rt, lb, rb = corners
    date_str = today.strftime("%Y.%m.%d")
    return (
        "An Instagram and Xiaohongshu cover image in a premium, minimalist black and gold style. "
        "The background is a very dark, matte black, nearly pure black with subtle, deep gold gradient "
        "highlights towards the top, like an abstract rising light. A single, very thin, horizontal gold "
        "line is at the top. The central content is text: \n\n"
        "Large, bold, high-contrast, pure white Chinese characters for \"本周的\" (This week's) on the first line. \n"
        "Immediately below it, an oversized, bold, matte amber-gold text \"GitHub\" in a clean sans-serif "
        "font, making it the dominant visual element. \n"
        "Below \"GitHub\", a smaller, clear, high-contrast white text for \"大家在关注什么好东西\" "
        "(What great things is everyone paying attention to?). \n\n"
        "In the middle-lower section, a single gold point acts as a separator. Below it, a clean, dark "
        "rectangular tag with a subtle texture and rounded corners holds smaller, white sans-serif text "
        "in a monospaced-style font, listed in a grid:\n\n"
        f"- Left top: {lt} (white) \n"
        f"- Right top: {rt} (white) \n"
        f"- Center (slightly larger): {center} (This specific text is in matte gold)\n"
        f"- Left bottom: {lb} (white) \n"
        f"- Right bottom: {rb} (white) \n\n"
        f"At the very bottom left, small, precise matte gold text for the date: \"{date_str}\" \n"
        "At the bottom right, small white text \"@九尾的ai日记\" and a clean, stylized minimalist portrait "
        "logo in white.\n"
        "The overall impression is sophisticated and high-tech."
    )

def save_image_prompt(repos):
    today = date.today()
    names = extract_repo_names(repos, n=5)
    prompt = build_image_prompt(names, today)
    payload = {"prompt": prompt}
    filename = f"/workspace/cover_prompt_{today}.json"
    with open(filename, "w", encoding="utf-8") as f:
        json.dump(payload, f, ensure_ascii=False, indent=2)
    print(f"封面图 prompt 已保存到 {filename}")

# --- 第六步：发送邮件（支持代理隧道） ---
class SMTPViaProxy:
    """通过代理隧道的 SMTP 客户端"""
    def __init__(self, sock):
        self.sock = sock
        self.file = sock.makefile('rb')
    def send(self, cmd):
        self.sock.sendall((cmd + '\r\n').encode())
    def recv(self):
        return self.file.readline().decode().strip()
    def close(self):
        self.sock.close()

def send_email_via_proxy(summary, filename, proxy_host, proxy_port):
    """通过 HTTP CONNECT 代理隧道发送邮件"""
    s = socket.create_connection((proxy_host, proxy_port), timeout=15)
    connect_req = f'CONNECT smtp.qq.com:465 HTTP/1.1\r\nHost: smtp.qq.com:465\r\n\r\n'
    s.sendall(connect_req.encode())
    response = s.recv(4096).decode()
    if '200' not in response.split('\n')[0]:
        s.close()
        raise ConnectionError(f"代理隧道建立失败: {response.strip()}")
    ctx = ssl.create_default_context()
    ss = ctx.wrap_socket(s, server_hostname='smtp.qq.com')
    smtp = SMTPViaProxy(ss)
    smtp.recv()  # 220 banner
    smtp.send('EHLO localhost')
    for _ in range(8):
        line = smtp.recv()
        if not line.startswith('250-'):
            break
    smtp.send('AUTH LOGIN')
    smtp.recv()
    smtp.send(base64.b64encode(SENDER_EMAIL.encode()).decode())
    smtp.recv()
    smtp.send(base64.b64encode(SENDER_PASSWORD.encode()).decode())
    auth_resp = smtp.recv()
    if '235' not in auth_resp:
        raise Exception(f"认证失败: {auth_resp}")
    msg = MIMEMultipart()
    msg['From'] = SENDER_EMAIL
    msg['To'] = RECEIVER_EMAIL
    msg['Subject'] = f"GitHub Trending 日报 - {date.today()}"
    body = f"<h2>GitHub Trending 日报 ({date.today()})</h2><hr>{summary.replace(chr(10), '<br>')}<hr><p>详细内容请查看附件。</p>"
    msg.attach(MIMEText(body, 'html', 'utf-8'))
    with open(filename, 'r', encoding='utf-8') as f:
        attachment = MIMEText(f.read(), 'plain', 'utf-8')
        attachment.add_header('Content-Disposition', 'attachment', filename=f"trending_{date.today()}.md")
        msg.attach(attachment)
    smtp.send(f'MAIL FROM:<{SENDER_EMAIL}>')
    smtp.recv()
    smtp.send(f'RCPT TO:<{RECEIVER_EMAIL}>')
    smtp.recv()
    smtp.send('DATA')
    smtp.recv()
    smtp.send(msg.as_string() + '\r\n.')
    smtp.recv()
    smtp.send('QUIT')
    smtp.close()
    print(f"✅ 邮件已成功发送到 {RECEIVER_EMAIL}")
    return True

def send_email(summary, filename):
    """发送邮件（自动检测代理环境）"""
    try:
        proxy = os.environ.get('https_proxy') or os.environ.get('HTTPS_PROXY') or ''
        if proxy:
            from urllib.parse import urlparse
            parsed = urlparse(proxy)
            proxy_host = parsed.hostname or '127.0.0.1'
            proxy_port = parsed.port or 18080
            print(f"检测到代理 {proxy_host}:{proxy_port}，通过代理隧道发送邮件...")
            return send_email_via_proxy(summary, filename, proxy_host, proxy_port)
        else:
            msg = MIMEMultipart()
            msg['From'] = SENDER_EMAIL
            msg['To'] = RECEIVER_EMAIL
            msg['Subject'] = f"GitHub Trending 日报 - {date.today()}"
            body = f"<h2>GitHub Trending 日报 ({date.today()})</h2><hr>{summary.replace(chr(10), '<br>')}<hr><p>详细内容请查看附件。</p>"
            msg.attach(MIMEText(body, 'html', 'utf-8'))
            with open(filename, 'r', encoding='utf-8') as f:
                attachment = MIMEText(f.read(), 'plain', 'utf-8')
                attachment.add_header('Content-Disposition', 'attachment', filename=f"trending_{date.today()}.md")
                msg.attach(attachment)
            server = smtplib.SMTP_SSL("smtp.qq.com", 465)
            server.login(SENDER_EMAIL, SENDER_PASSWORD)
            server.sendmail(SENDER_EMAIL, RECEIVER_EMAIL, msg.as_string())
            server.quit()
            print(f"✅ 邮件已成功发送到 {RECEIVER_EMAIL}")
            return True
    except Exception as e:
        print(f"❌ 邮件发送失败: {e}")
        return False

# --- 执行 Workflow ---
try:
    html = fetch_trending()
    repos = parse_repos(html)
    summary = summarize(repos)
    filename = save_report(summary)
    save_image_prompt(repos)
    send_email(summary, filename)
except Exception as e:
    print(f"执行失败: {e}")
```

### 第三步：安装依赖并执行

```bash
pip install openai beautifulsoup4 requests --break-system-packages
python3 /data/user/work/github_trending_auto.py
```

### 第四步：验证输出

执行完成后，检查以下输出：

1. **日报文件**：`/workspace/trending_YYYY-MM-DD.md` — 中文摘要日报
2. **封面图 Prompt**：`/workspace/cover_prompt_YYYY-MM-DD.json` — 用于生成封面图的 prompt
3. **邮件发送**：检查控制台输出是否包含 `✅ 邮件已成功发送`

### 第五步：向用户报告结果

向用户展示执行结果表格：

| 步骤 | 状态 |
|------|------|
| 抓取 GitHub Trending | ✅/❌ |
| DeepSeek API 生成摘要 | ✅/❌ |
| 保存日报到本地 | ✅/❌ |
| 发送邮件 | ✅/❌ |

并提供日报文件链接。

## 故障排查

### 邮件发送失败：`[Errno 99] Cannot assign requested address`

**原因**：环境有 HTTP 代理但 smtplib 不支持代理。

**解决方案**：脚本已内置 `SMTPViaProxy` 类，通过 HTTP CONNECT 隧道绕过限制。自动检测环境变量 `https_proxy` / `HTTPS_PROXY`。

### GitHub 页面解析失败

**原因**：GitHub 可能更新了 HTML 结构，导致 CSS 选择器失效。

**解决方案**：检查 `article.Box-row`、`h2 a`、`p.col-9` 等选择器是否仍然匹配当前页面结构。

### DeepSeek API 调用失败

**原因**：API Key 过期或余额不足。

**解决方案**：检查 `DEEPSEEK_API_KEY` 配置，确认账户状态。

## 注意事项

- 脚本在 SOLO 沙箱中执行，文件保存在 `/workspace/` 目录
- 如果用户需要在本地运行，需将脚本下载到本地并修改保存路径
- 邮件发送失败不会影响日报文件的保存
- 每次执行会覆盖同一天的已有日报文件
