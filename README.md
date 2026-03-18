# CloudFlare-R2-LocalPicManager — 自建图床（Cloudflare R2 + Worker）

基于 Cloudflare R2 对象存储 + Cloudflare Worker 的免费自建图床方案，附带单文件 HTML 管理界面和 AI Agent 集成支持。

**免费额度**：R2 每月 10GB 存储 + 每月 100 万次 A 类操作免费，Worker 每天 10 万次请求免费。个人博客图床完全够用。

## 功能

- **图片管理**：上传、删除、重命名、批量操作
- **文件夹**：创建/删除文件夹，在文件夹间移动文件
- **预览**：Lightbox 全屏预览，支持键盘导航
- **图片编辑**：裁剪、压缩、调整尺寸、格式转换（内置，无需安装）
- **防盗链**：Referer 检查，防止其他网站盗用图片
- **AI Agent 集成**：Python SDK + Node.js SDK，支持 URL 抓取上传和 Base64 上传
- **OpenClaw Skill**：让 AI Agent 用自然语言管理图床

---

## 第一步：Cloudflare 准备工作

### 1.1 注册 Cloudflare 账号

前往 [cloudflare.com](https://cloudflare.com) 注册，免费账号即可。

### 1.2 创建 R2 存储桶

1. 登录 Cloudflare Dashboard
2. 左侧菜单点击 **R2 对象存储**
3. 点击 **创建存储桶**
4. 输入名称（例如 `my-images`），地区选 **自动**
5. 点击创建

记住你的桶名称，后面会用到。

---

## 第二步：部署 Cloudflare Worker

### 2.1 安装 Wrangler

Wrangler 是 Cloudflare 官方命令行工具，用于部署 Worker。

```bash
npm install -g wrangler
```

### 2.2 登录 Cloudflare

```bash
wrangler login
```

会自动打开浏览器，在网页上授权即可。

### 2.3 修改配置文件

编辑 `wrangler.toml`：

```toml
name = "my-picbed"           # Worker 名称，随便起
main = "src/index.js"
compatibility_date = "2024-01-01"

[[r2_buckets]]
binding = "BUCKET"
bucket_name = "my-images"    # ← 改成你第一步创建的桶名称

[vars]
CUSTOM_DOMAIN = ""           # 有自定义域名就填，没有留空
```

### 2.4 设置 Auth Token（重要）

Auth Token 是你的图床访问密码，**不要**写进 `wrangler.toml`（会被 git 记录）。用以下命令单独设置：

```bash
wrangler secret put AUTH_TOKEN
```

输入命令后会提示你输入值，输入一个你自己想好的密码（建议 16 位以上的随机字符串），回车确认。

### 2.5 配置防盗链（可选）

编辑 `src/index.js`，找到 `allowedHosts` 数组，填入你允许引用图片的域名：

```javascript
const allowedHosts = [
  "example.com",       // 你的博客域名
  "blog.example.com",  // 子域名
  "localhost",         // 本地开发
  "127.0.0.1",
];
```

如果不需要防盗链（图片完全公开），把数组清空：

```javascript
const allowedHosts = [];
```

### 2.6 部署

```bash
wrangler deploy
```

部署成功后会看到类似输出：

```
Published my-picbed (1.23 sec)
  https://my-picbed.your-account.workers.dev
```

记住这个 URL，这就是你的 Worker 地址。

---

## 第三步：使用前端管理界面

直接在浏览器打开 `PicManager.html`（双击文件即可，不需要服务器）。

### 3.1 首次配置

首次打开会弹出设置窗口，填入：

| 字段 | 说明 |
|------|------|
| **Worker URL** | 第二步部署后得到的 URL |
| **Auth Token** | 你在 `wrangler secret put` 设置的密码 |
| **自定义域名** | 可选，有的话填在这里 |

点保存即可开始使用。配置保存在浏览器的 localStorage 中，不会泄露。

### 3.2 主要功能

- **上传**：点击上传按钮，或直接拖拽文件到页面
- **文件夹**：左侧侧边栏可创建文件夹，点击进入
- **预览**：点击图片缩略图进入 Lightbox
- **编辑**：在 Lightbox 中点击编辑按钮，可裁剪/调整尺寸/转换格式
- **复制链接**：右键图片或使用操作按钮，可复制 URL / Markdown / HTML 格式
- **批量操作**：勾选多张图片后批量删除或移动

---

## 第四步：绑定自定义域名（可选）

默认的 `workers.dev` URL 比较长，可以绑定自己的域名，得到类似 `https://img.example.com/photo.jpg` 的短链接。

**前提**：你的域名已经托管在 Cloudflare（nameserver 指向 Cloudflare）。

1. Cloudflare Dashboard → **Workers 和 Pages** → 你的 Worker
2. 点击 **设置** → **域和路由** → **添加** → **自定义域**
3. 输入子域名（如 `img.example.com`），确认
4. Cloudflare 自动配置 DNS 和 SSL，等待几分钟生效

生效后，在 PicManager 设置里填入自定义域名，复制的链接会自动使用短链接。

同时更新 `wrangler.toml` 里的 `CUSTOM_DOMAIN`，重新 `wrangler deploy` 一次，Worker 返回的图片 URL 也会用自定义域名。

---

## 第五步：AI Agent 集成（可选）

如果你使用 AI Agent 写作，可以让 Agent 直接管理图床。

### Python SDK

```bash
pip install requests
```

```python
from picmanager import PicManager

pm = PicManager(
    worker_url="https://my-picbed.your-account.workers.dev",
    auth_token="your-auth-token",
    custom_domain="https://img.example.com"  # 可选
)

# 从 URL 抓图上传
result = pm.upload_from_url("https://example.com/photo.jpg", folder="blog")
print(result["url"])

# 上传本地文件
result = pm.upload_file("/path/to/image.png", folder="blog")

# 获取 Markdown 引用
print(pm.markdown("blog/photo.jpg", alt="配图"))
# ![配图](https://img.example.com/blog/photo.jpg)

# 列出所有图片
images = pm.list_all()

# 获取统计
info = pm.info()
print(f"共 {info['total_files']} 张图片，{info['total_size_human']}")
```

### Node.js SDK

零依赖，仅需 Node.js 18+。

```javascript
import { PicManager } from './picmanager.mjs';

const pm = new PicManager({
  workerUrl: 'https://my-picbed.your-account.workers.dev',
  authToken: 'your-auth-token',
  customDomain: 'https://img.example.com',
});

const result = await pm.uploadFromUrl('https://example.com/photo.jpg', { folder: 'blog' });
console.log(result.url);
```

### OpenClaw Skill

[OpenClaw](https://openclaw.ai) 是一个 AI Agent 平台，通过 Skill 扩展能力。安装后可以用自然语言管理图床。

**安装步骤：**

1. 把 `skills/picmanager/` 目录复制到 `~/.openclaw/workspace/skills/picmanager/`

2. 同时把 `picmanager.mjs` 复制进去：
   ```bash
   cp picmanager.mjs ~/.openclaw/workspace/skills/picmanager/
   ```

3. 创建配置文件：
   ```bash
   cp skills/picmanager/.env.example ~/.openclaw/workspace/skills/picmanager/.env
   ```

   编辑 `.env` 填入你的实际配置：
   ```bash
   PICMANAGER_WORKER_URL=https://my-picbed.your-account.workers.dev
   PICMANAGER_AUTH_TOKEN=your-auth-token
   PICMANAGER_CUSTOM_DOMAIN=https://img.example.com
   ```

4. 重启 OpenClaw，它会自动加载新 Skill。

**使用方式：**

对 OpenClaw 说：
> "帮我把桌面上的 screenshot.png 上传到 blog 文件夹，给我 Markdown 链接"

OpenClaw 会自动调用图床 API，完成上传并返回链接。

---

## 文件说明

```
picbed/
├── src/
│   └── index.js          # Cloudflare Worker 后端（~160 行）
├── wrangler.toml          # Wrangler 部署配置
├── PicManager.html        # 前端管理界面（单文件，无需服务器）
├── picmanager.py          # Python SDK
├── picmanager.mjs         # Node.js SDK
└── skills/
    └── picmanager/
        ├── SKILL.md       # OpenClaw Skill 描述
        ├── cli.mjs        # CLI 命令行工具（供 Agent 调用）
        └── .env.example   # 配置模板（复制为 .env 后填写）
```

---

## 常见问题

**Q: 上传提示 401 Unauthorized？**

检查 Auth Token 是否正确。`wrangler secret put AUTH_TOKEN` 设置的值，和 PicManager 设置里填的值必须完全一致。

**Q: 图片能上传，但在前端打开编辑器时图片加载失败？**

确认 Worker URL 填写正确，且 Worker 已成功部署。可以直接在浏览器访问 `https://your-worker.workers.dev/your-image.jpg` 测试图片是否可访问。

**Q: 防盗链把我自己的网站也挡住了？**

检查 `src/index.js` 里 `allowedHosts` 数组，确保你的域名已添加。注意：子域名需要单独添加，或利用 `endsWith` 逻辑自动匹配（代码已支持）。

**Q: 自定义域名绑定后图片 URL 还是 workers.dev？**

需要同时：①在 Cloudflare Dashboard 绑定自定义域；②在 `wrangler.toml` 里的 `CUSTOM_DOMAIN` 填入自定义域名；③重新 `wrangler deploy`。

**Q: R2 免费额度够用吗？**

个人博客完全够用。免费额度：10GB 存储、每月 100 万次 A 类操作（写入）、1000 万次 B 类操作（读取）。超出才收费。

---

## License

MIT
