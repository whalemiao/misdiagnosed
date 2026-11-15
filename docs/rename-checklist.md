# 改用户名 checklist（sb1145148 → whalemiao）

> 一次性把所有牵连的地方改干净。**按顺序做**，前一步没完成别做下一步。

---

## 阶段 0：改名前的准备（现在就能做）

- [ ] **确认新域名已就绪**
  - DigitalPlat 申请 `whalemiao.dpdns.org`
  - Cloudflare 加 zone / Pages 加自定义域
  - 验证：`curl -I https://whalemiao.dpdns.org` 返回 200

- [ ] **代码已支持双域名** ✅ 已完成（commit `452eefc`）
  ```
  frontend/core.js, admin.js
  frontend/functions/{cards,views.svg,sitemap.xml}.js
  frontend/functions/a/[slug].js
  frontend/sw.js
  worker/worker.js, worker/wrangler.toml
  ```

- [ ] **启用 Worker 新路由**
  ```powershell
  cd D:\Backup\Documents\deepseek-harness\default-workspace\profile-site\worker
  # 取消 wrangler.toml 里 api.whalemiao.dpdns.org 那行的注释
  npx wrangler deploy
  ```

- [ ] **验证新域名整条链路**
  - `https://whalemiao.dpdns.org` 能打开
  - 浏览器 F12 → Network 里 `/api/*` 请求打到 `api.whalemiao.dpdns.org`
  - 文章详情页 `/a/<slug>` 的 canonical 指向新域

- [ ] **列出「服务方后台里存着旧名字 / 旧域名」的地方**

  这一条比上面三条都重要，因为它**不在代码里，`grep` 搜不到**，只能靠列清单。

  改名时逐条确认（每条的检查方式都写在下表）：

  | 存在哪 | 存了什么 | 怎么查 | 不改会怎样 |
  | --- | --- | --- | --- |
  | **Cloudflare Turnstile** | widget 的 hostname 白名单 | Dashboard → Turnstile → Settings → Hostnames | 留言板人机校验报「无法连接到网站」（300030） |
  | **D1 `settings` 表** | `github_user`（还有 `github_snapshot` 快照） | `wrangler d1 execute <db> --remote --command "SELECT key,substr(value,1,40) FROM settings WHERE key LIKE 'github%'"` | 首页 GitHub 卡片一直显示旧用户名 + 「缓存快照」 |
  | **Worker 的 `ALLOWED_ORIGINS` / 前端 `SITE_ROOTS`** | 允许的来源域名 | `grep -rn "dpdns.org" worker/ frontend/` | 新域名接口 403、CORS 失败 |
  | **Cloudflare 账号显示名** | `xxx@163.com's Account` | Dashboard 右上角 | 只是显示，不影响功能（可改可不改） |
  | **Email Routing / 发信地址** | `hi@<旧域名>` 之类 | Dashboard → Email → Routing | 邮件退回或指向旧域 |
  | **GitHub 侧的描述 / 主页 / social** | 个人简介里的链接 | `https://github.com/<新名>` 看 Profile | 简介里还链着旧地址 |
  | **第三方（Discord / B站 / 友链）** | 你的主页链接 | 人工看 | 旧链接 404（GitHub 仓库会 301，主页链接不会） |

  **这里的关键不是"改"，而是"知道自己有哪些地方存了名字"。**
  改名当天把这张表过一遍，比改名两周后被用户报告要好。

  > 实测：这次改名后**第二天**就碰到前两条（Turnstile、D1），
  > 两次都是"用着用着突然发现"，不是主动检查出来的 —— 见
  > [identity-rename.md](identity-rename.md)。

---

## 阶段 1：改名（你在 GitHub 上操作）

- [ ] `Settings → Account → Change username` → 填 `whalemiao`
- [ ] 改完后**立刻**把 git 邮箱换成带数字 ID 的格式（ID 不变，永久有效）
  ```powershell
  git config --global user.email "234821665+whalemiao@users.noreply.github.com"
  git config --global user.name "whalemiao"
  ```
  > ⚠️ 旧的 `234821665+whalemiao@users.noreply.github.com` 在改名后**不再关联账号**，
  > 新提交必须用带 ID 的格式，否则贡献图不亮。

---

## 阶段 2：本地 11 个仓库换 remote（我来跑）

```powershell
$repos = @('bili','profile-site','dsh-backup','dsh-backup-tools','dsh-tools',
           'gh-cards','learning-notes','md-check','meme-backup','misdiagnosed',
           'vscode-config-backup')
foreach ($r in $repos) {
    $p = "D:\Backup\Documents\deepseek-harness\$r"
    if (Test-Path $p) {
        git -C $p remote set-url origin "https://github.com/whalemiao/$r.git"
    }
}
```

- [ ] 跑上面这段
- [ ] 每个仓库 `git push --dry-run` 验证（靠 GitHub 重定向也能过，但换掉更干净）

---

## 阶段 3：个人主页仓库（关键，不做主页会空）

- [ ] **重命名** `sb1145148/sb1145148` → `whalemiao/whalemiao`
  （Settings → Repository name）
- [ ] 确认两个 workflow 还在跑：`metrics.yml`（UTC 03:30）、`snake.yml`（UTC 03:00）
- [ ] README 里的 badge 改掉
  ```diff
  - <a href="https://whalemiao.dpdns.org"><img src="https://img.shields.io/badge/个人站-whalemiao.dpdns.org-...">
  + <a href="https://whalemiao.dpdns.org"><img src="https://img.shields.io/badge/个人站-whalemiao.dpdns.org-...">
  ```

---

## 阶段 4：文件里的引用（353 处 / 91 个文件）

- [ ] 批量替换（先 dry-run 看清单，再执行）
  ```powershell
  # 预览：哪些文件会被改
  Get-ChildItem D:\Backup\Documents\deepseek-harness -Recurse -File -Include *.md,*.ps1,*.py,*.json,*.yml,*.js,*.html |
    Where-Object { $_.FullName -notmatch '\\\.git\\|node_modules|\\logs\\' } |
    Select-String -Pattern 'sb1145148' -SimpleMatch | Group-Object Path | Sort-Object Count -Descending
  ```
- [ ] **替换范围要分两类**（不能无脑全换）：
  | 类型 | 例子 | 换吗 |
  | --- | --- | --- |
  | 指向自己的链接 | `github.com/whalemiao/...` | ✅ 换 |
  | 域名 | `whalemiao.dpdns.org` | ✅ 换（旧域保留但新链接用新域） |
  | **历史记录** | 学习笔记里"我当时叫 sb1145148" | ❌ 不换（历史就是历史） |
  | **`GH_USER` 常量** | `worker`/`cards.js` 里查 GitHub 用 | ✅ 换（否则 gh-cards 查不到） |

---

## 阶段 5：外部引用（只能手动）

- [ ] **Cute-Dress 的 README**（你要改的两行，B 站不动）
  ```diff
  - - GitHub：[点击喵~](https://github.com/whalemiao)
  - - 个人站：[点我喵~](https://whalemiao.dpdns.org)
  + - GitHub：[点击喵~](https://github.com/whalemiao)
  + - 个人站：[点我喵~](https://whalemiao.dpdns.org)
    - bilibili：[点这里~](...)      ← 不改
  ```
  > 那是别人的仓库 → 要**再提一个 PR**，只改你自己目录下的 `README.md`。

- [ ] Cloudflare 账号名 `Sb1145148@163.com's Account`（可改可不改，只是显示名）
- [ ] `whalemiao.dpdns.org` 这个旧域名：保留（双域名并存）还是停用？

---

## 阶段 5.5：藏在服务方配置里的旧名字（改名第二天才发现的）

改完 GitHub 和文件不等于结束 —— **有些地方记着你的旧名字，但你平时根本不会去看**。
两处都是**静默失效**：页面能打开、其它功能正常，只有那一个组件坏了。
（2026-11-14 改名，11-15 才发现 —— 都不是主动检查发现的，是用着用着碰到的。）

### 1. Cloudflare Turnstile 的 Hostnames 白名单

**症状**：留言板的人机校验框显示「无法连接到网站 / 故障排除」
（Turnstile 自己的报错 UI，带 Cloudflare logo，看起来像"网站挂了"）

**真实原因**：widget 只允许创建时填的那些域名，新域名不在白名单 → 错误码 **300030**

**修法**：

```
Cloudflare Dashboard → Turnstile → 找到那个 widget
→ Settings → Hostnames → Add hostname
   新域名、旧域名、pages.dev 默认域名 都留着
→ 保存，立即生效，不用重新部署
```

**怎么区分"是我的站坏了"还是"这个组件坏了"**：
别的页面/接口都正常，只有这一块出 Cloudflare 的报错框 → 是**这个组件**的配置问题。

### 2. D1 `settings.github_user`

**症状**：GitHub 卡片显示 `@旧用户名`，下面还有一行「实时数据暂时拿不到，显示的是缓存快照」

**真实原因**：这值存在数据库的 settings 表里（后台可改），改名时没人想起它

**修法**：

```powershell
# 改用户名
node wrangler.js d1 execute profile-db --remote `
  --command "UPDATE settings SET value='新用户名' WHERE key='github_user'"

# 清掉旧账号的抓取快照，否则卡片会一直显示旧数据
node wrangler.js d1 execute profile-db --remote `
  --command "DELETE FROM settings WHERE key='github_snapshot'"

# 验证
curl.exe -sS https://api.<域名>/api/github
```

**一个坑**：Cloudflare 的出口 IP 拉 `api.github.com` 经常失败或限流（未认证 60 次/小时，
出口 IP 还是共享的），而你自己本机 curl 却是 200。所以改完设置**可能还是 `unavailable`**。

**救急办法**（本机能拉就用本机）：

```python
# 本机抓一份，写成 SQL，再灌进 D1
# 数据结构和 routes/misc.js 的 github() 输出保持一致：
# {login,name,avatar,bio,...,repos:[{name,url,desc,stars,forks,lang,updated_at}],
#  fetched_at, stale:true}
INSERT INTO settings (key, value) VALUES ('github_snapshot', '<整段 JSON>')
ON CONFLICT(key) DO UPDATE SET value = excluded.value;
```

**根治**：给 Worker 加一个 GitHub token（`wrangler secret put GITHUB_TOKEN`，5000 次/小时）。
但 token 只救"限流"，救不了"网络被拦"。

---

## 「改名后静默失效」的完整清单

按**症状**反查，比按"哪里可能存名字"瞎搜快得多：

| 症状 | 该查哪 |
| --- | --- |
| 留言板人机校验报"无法连接到网站" | **Turnstile → Hostnames** |
| GitHub 卡片是旧用户名 / 一直"缓存快照" | **D1 `settings.github_user`** + 清 `github_snapshot` |
| 卡片数据要等很久才出 | 边缘缓存 1 小时（`caches.default`，key 里带用户名） |
| 新域名能开但接口 403 | Worker `ALLOWED_ORIGINS` / `functions` 里的 `SITE_ROOTS` |
| 旧仓库地址 301 到新地址 | 这是 GitHub 的正常重定向，不用改 |
| 邮件 / 头像 / RSS 还是旧地址 | Profile 仓库 README、Worker 顶部常量、`content/` 里的链接 |

---

## 阶段 6：验证

- [ ] 提交一次测试提交，看 GitHub 上作者是不是 `whalemiao`
- [ ] 个人主页 `/whalemiao` 显示 README（不是空白）
- [ ] 网站新域名能打开、API 正常、文章页 canonical 正确
- [ ] 贡献图里旧提交是否还在（旧 noreply 格式的很可能断档 —— 这是**唯一的不可逆损失**）

---

## 已知的不可逆损失（改名之前要接受）

| 损失 | 说明 |
| --- | --- |
| 旧提交的贡献图归属 | 旧 noreply 邮箱不含 ID，改名后大概率脱离 |
| `@sb1145148` 的历史提及 | 不重定向，全部变死链 |
| 签名 commit 的 Verified | 用旧 noreply 邮箱签的会掉标记 |

---

## 一句话

> **改名不是"改个显示名"，是搬一次家。**
> 搬家清单里最容易被忘的三样：**个人主页仓库、git 邮箱格式、外部引用（Cute-Dress 那种）。**
>
> 还有一样更隐蔽的：**服务方后台里的域名/用户名白名单**（Turnstile 的 Hostnames、数据库里的 `github_user`）—— 它们不在代码里，搜不到，只能靠「哪个组件坏了」去反查。
