# 改名字 / 身份迁移

> 改名不是「改个显示名」，是搬一次家。
>
> 前两类坑（代码里的旧名字、GitHub 上的旧地址）改完就没事了 ——
> **这两条是改名两周后才发现的：名字存在服务方的配置里，不在代码里，`grep` 搜不到。**

完整改名流程见 [改名 checklist](rename-checklist.md)。

---

## 39. 改名后留言板的人机校验一直报「无法连接到网站」

**现象**

留言板里 Cloudflare Turnstile 的验证框显示：

```
无法连接到网站
故障排除                    ← 右下角有 Cloudflare 的标志
```

**站内其它一切都正常**：首页能开、文章页能开、`/api/profile` 返回 200、留言列表能加载。
只有这一块是坏的，而且看起来像「这个网站挂了」。

**判断信号**

同一个部署，新旧域名各打开一次留言板：

```powershell
# 两个域名都返回 200（说明站点本身没坏）
curl.exe -sS -o NUL -w "%{http_code}`n" https://新域名/
curl.exe -sS -o NUL -w "%{http_code}`n" https://旧域名/
```

- **旧域名下验证框正常、新域名下报错** → 就是这条（域名白名单）
- **两个域名都报错** → 不是这条，去看 widget 的 sitekey 是否配错
- **整站 5xx / 打不开** → 也不是这条，是部署或 Functions 的问题

补充信号：浏览器控制台里 Turnstile 的 `error-callback` 会收到 **300030**
（但多数实现只 `console.warn`，页面上看不到 —— 见「解法」第 3 步）。

**根因**

1. Turnstile 的 widget 在创建时就绑定了 **hostname 白名单**
2. 改名/加域名后，新域名不在白名单里
3. 组件加载时校验失败 → 直接显示它自己的报错 UI（不是你的站点渲染的）
4. 后端 `verifyTurnstile` 拿不到令牌 → 留言接口拒绝 → 用户看到「发不出留言」

**关键是**：这个配置在 **Cloudflare 后台**，不在仓库里，**代码搜索发现不了**。

**解法**

按成本从低到高：

1. **加白名单（根本修复，1 分钟，不用重新部署）**

   ```
   Cloudflare Dashboard → Turnstile → 选那个 widget
   → Settings → Hostnames → Add hostname
   → 新域名、旧域名、pages.dev 默认域名 都留着
   ```

   保存后几十秒生效。

2. **如果同时存在多个域名**，全部加进去 —— 只加新的会让旧域名反过来坏掉。

3. **让报错可见**（顺手改的，避免下次再从截图猜）：

   ```js
   // 原来：只打日志，用户什么都看不到
   'error-callback': () => console.warn('[turnstile] 组件报错'),

   // 改成：把错误码写到页面上
   'error-callback': (code) => {
     console.warn('[turnstile] 组件报错', code);
     el.insertAdjacentHTML('beforeend',
       '<p class="text-xs text-amber-600 mt-1">人机校验组件出错（' + code +
       '）。站长请检查 Turnstile 的 Hostnames 设置。</p>');
   }
   ```

**代价**

排查约 **2 小时**（从看到截图到定位）。

时间全花在错误方向上：先怀疑 Functions 路由、再怀疑 CORS、再怀疑 D1 里 sitekey 被清空
（实际 `/api/profile` 一直正常返回 sitekey）——

**真正的浪费是：这个报错框是 Cloudflare 画的，不是你的站点画的**，
所以它不提供任何"你的系统"里的线索。

**适用边界**

- 只适用于「**单个组件**报错、站点其余部分正常」的情况
- 如果 Turnstile 的 sitekey 本身被清空（`/api/profile` 里是空串），现象是**整个验证框不出现**，
  而不是报错框 —— 那是另一条路径
- 自建的人机校验（不依赖第三方）不适用
- 白名单是**按域名**的，`www.` 子域要单独加

---

## 40. 改名后 GitHub 卡片一直显示旧用户名

**现象**

个人站的 GitHub 卡片显示：

```
@旧用户名 · China Fujian
个人AI开发者
12 仓库 · 4 关注者
⚠️ 实时数据暂时拿不到，显示的是缓存快照
```

**卡片能显示**（说明接口没挂）、**数字也合理**（说明曾经抓到过真数据），
但你已经在 GitHub 上改名很久了。

**判断信号**

```powershell
# 1. 看数据库里存的名字（这条最关键）
node wrangler.js d1 execute <数据库名> --remote `
  --command "SELECT key, substr(value,1,40) FROM settings WHERE key LIKE 'github%'"

# 2. 看接口现在返回什么
curl.exe -sS https://api.<你的域名>/api/github
```

判定：

- `github_user` 是**旧名字** → 就是这条
- 名字已经是新的，但接口返回 `"unavailable": true` → **不是这条**，是出口网络/限流问题
  （见「代价」里的说明）
- 接口返回新名字但页面还是旧的 → 边缘缓存（1 小时），等或换聚合键

**根因**

1. 这套站把 GitHub 用户名存在 **D1 的 `settings` 表**里（后台可改），而不是写在代码里
   —— 源码里的默认值早就改成了新名字，但**数据库里的旧值优先级更高**
2. 卡片逻辑是「先拉 API，拉不到就用 D1 里的 `github_snapshot` 快照兜底」
3. 快照里存的是**旧账号抓取时的完整 JSON**（含 `login`、头像、仓库列表）
4. 于是：接口失败 → 回落到旧快照 → 页面显示旧用户名 + 「缓存快照」提示

**改名时只改了代码，没改数据库** —— 这和 [31 号坑（编译期常量）](platform-quirks.md#31-powershell-脚本不带-bom中文注释把解析搞崩)
是同一类：**你以为改了，其实改的是副本。**

**解法**

```powershell
# 1. 改数据库里的用户名
node wrangler.js d1 execute <数据库名> --remote `
  --command "UPDATE settings SET value='新用户名' WHERE key='github_user'"

# 2. 清掉旧账号的快照（不清就会一直显示旧数据）
node wrangler.js d1 execute <数据库名> --remote `
  --command "DELETE FROM settings WHERE key='github_snapshot'"

# 3. 验证
curl.exe -sS https://api.<你的域名>/api/github
```

**如果第 3 步还是 `unavailable: true`**：说明 Worker 出口拉不到 `api.github.com`。

在本机（能连的时候）抓一份，写成 SQL 灌进去 —— 数据结构要和接口输出一致：

```python
# 抓 https://api.github.com/users/<名字>
#   + /users/<名字>/repos?sort=updated&per_page=100
# 组装成 {login,name,avatar,bio,...,repos:[{name,url,desc,stars,forks,lang,updated_at}],
#         fetched_at, stale: true}
# 再写：
# INSERT INTO settings (key,value) VALUES ('github_snapshot','<整段 JSON>')
# ON CONFLICT(key) DO UPDATE SET value = excluded.value;
```

**代价**

排查约 **1 小时**。

时间花在两个错误假设上：

1. 「GitHub API 挂了」—— 其实本机 curl 一直是 200
2. 「改名后 GitHub 的 API 会自动跟随」—— 对 `/users/<旧名>` 确实会重定向，
   但**我们根本没在用旧名请求**：数据库里存的旧值让 Worker 走快照分支

还有一个**反直觉的坑**：Cloudflare 出口访问 `api.github.com` **经常失败或限流**
（未认证 60 次/小时，且出口 IP 是共享的），而**你自己本机 curl 是 200**。
所以「我这边明明是通的」不能证明 Worker 那边通。

**适用边界**

- 只适用于「用户名存在数据库/后台配置里」的设计；硬编码在代码里的改完重部署即可
- 快照兜底本身是**好设计**（API 挂了卡片不至于空着），代价是**旧数据会一直显示到被覆盖**
- 加 GitHub token（`wrangler secret put GITHUB_TOKEN`，5000 次/小时）能解决**限流**，
  但解决不了**出口被拦**
- 边缘缓存的 key 里带用户名，所以改完名字后**缓存会自动换 key**，不用手动清
