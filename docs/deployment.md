# 部署指南

> 把 AnvilWiki 部署到 Cloudflare Workers，全程免费、两个 secret 搞定自动部署、无限带宽。
>
> 预计耗时：首次 10 分钟，熟练后 3 分钟。

---

## 前提条件

- 一个 [GitHub](https://github.com) 账号（免费）
- 一个 [Cloudflare](https://cloudflare.com) 账号（免费）
- 本地已安装 Node.js 22+ 和 pnpm
- 已经 fork 了 AnvilWiki 仓库并改好了配置层（见 [apply-template.md](./apply-template.md)）

> 还没 fork？看 [快速开始](../README.md#5-分钟快速开始)。

---

## 方式一：GitHub Actions 自动部署（推荐）

这是最简单的方式——配好两个 secret，之后每次 `git push` 自动构建部署。仓库已经自带 `.github/workflows/ci.yml`（检查代码）和 `cd.yml`（部署），fork 后开箱即用。

> 为什么不是 Cloudflare Pages 的「连一下 GitHub 仓库」一键式体验？Pages 的静态资源托管有个无法干净修复的固有 bug（见下方常见问题「为什么用 Workers 部署」），Workers 是修复方案，但 Workers 没有对等的 Pages Git 一键连接体验——GitHub Actions 是最贴近的替代，而且更透明：部署逻辑就在你的仓库里，不是锁在 Cloudflare dashboard 的黑盒配置里。

### Step 1 — 改个项目名

`wrangler.toml` 里的 `name = "anvilwiki"` 是 demo 站自己的 Worker 名。fork 后先改成你自己的（如 `anvil-quest-wiki`），不然会尝试部署到 demo 站的 Worker 上（没有 demo 站的权限，部署会失败，不会真的覆盖别人的站）：

```toml
name = "anvil-quest-wiki"  # 改这一行
compatibility_date = "2026-08-17"

[assets]
directory = "./dist"
html_handling = "drop-trailing-slash"
not_found_handling = "404-page"
```

### Step 2 — 推代码到 GitHub

```bash
# 在项目根目录
git init
git add .
git commit -m "Initial commit: AnvilWiki site"
git branch -M main
git remote add origin https://github.com/<你的用户名>/<你的仓库>.git
git push -u origin main
```

> 如果你 fork 的仓库，remote 已经配好了，直接 `git push`。

### Step 3 — 拿 Cloudflare API Token + Account ID

1. 打开 [dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens) → **Create Token**
2. 用模板 **"Edit Cloudflare Workers"**，Account Resources 选你自己的账号 → **Continue to summary** → **Create Token**
3. 复制生成的 token（只显示一次，先存好）
4. 回到 [dash.cloudflare.com](https://dash.cloudflare.com) → **Workers & Pages**，右上角能看到你的 **Account ID**，复制它

### Step 4 — 配 GitHub Secrets

进你仓库的 **Settings** → **Secrets and variables** → **Actions**：

**Secrets** 标签页，添加：

| Name                    | 值                    |
| ------------------------ | ---------------------- |
| `CLOUDFLARE_API_TOKEN`   | Step 3 拿到的 token     |
| `CLOUDFLARE_ACCOUNT_ID`  | Step 3 拿到的 Account ID |

**Variables** 标签页，至少加 `SITE_URL`（先用临时的，绑域名后再改）：

| Name                       | 值                              | 说明                       |
| --------------------------- | -------------------------------- | -------------------------- |
| `SITE_URL`                  | `https://anvil-quest-wiki.<你的子域>.workers.dev` | 部署一次后从 Cloudflare 拿到真实地址，先随便填个占位值也行，Step 5 后再回来改 |
| `PUBLIC_GA_ID`               | （可选）                          | Google Analytics ID        |
| `PUBLIC_GISCUS_REPO` 等      | （可选）                          | 见下方「环境变量清单」      |

> ⚠️ **`SITE_URL` 必须含 `https://` 前缀**（Astro 把它当 URL 解析，裸域名会让 build 报 `Invalid url`）。

### Step 5 — 触发部署

Push 任意一个 commit（哪怕是空的）到 `main`：

```bash
git commit --allow-empty -m "trigger deploy"
git push
```

GitHub Actions 会自动：`ci.yml` 跑 lint/typecheck/test/build/一堆内容检查 → 全部通过后触发 `cd.yml` → `wrangler deploy` 把 `dist/` 传到 Cloudflare。在仓库的 **Actions** 标签页能看到两个工作流依次跑完，`CD` 那个的日志最后会打印你的 `*.workers.dev` 地址。

---

## 绑定自定义域名

免费赠送的 `*.workers.dev` 域名可以一直用，但为了 SEO 和品牌，建议绑自定义域名。

### Step 1 — 买域名

推荐平台（按价格/易用度）：

| 平台                                                        | 后缀推荐         | 价格       |
| ----------------------------------------------------------- | ---------------- | ---------- |
| [Spaceship](https://spaceship.com)                          | `.wiki` / `.com` | ~十几元/年 |
| [Cloudflare Registrar](https://dash.cloudflare.com/domains) | `.com` / `.net`  | 成本价     |
| [Namecheap](https://namecheap.com)                          | `.xyz` / `.com`  | ~十几元/年 |

> 游戏 wiki 站首选 `.wiki` 后缀——便宜、相关性高、SEO 友好。

### Step 2 — 域名先转到 Cloudflare（如果还没在上面）

**Workers & Pages** → **Overview**，或直接在 dashboard 左侧菜单找 **Websites** → **Add a site**，输入你的域名，按提示把域名的 nameservers 改成 Cloudflare 给的两条——这一步是任何 Cloudflare 服务（Workers/Pages 都一样）绑自定义域名的前提，域名商那边操作，等生效（几分钟到几小时）。

### Step 3 — 在 Cloudflare 给 Worker 绑域名

1. **Workers & Pages** → 选中你的 Worker（Step 1 里改的名字）
2. **Settings** → **Domains & Routes** → **Add** → **Custom Domain**
3. 输入你的域名（如 `anvilquestwiki.wiki`），点 **Add Domain**
4. Cloudflare 自动配好 DNS + 签发 SSL 证书，几分钟内生效

> www 子域名想要的话，重复一遍上面的步骤，输入 `www.anvilquestwiki.wiki`。

### Step 4 — 更新 SITE_URL 并重新部署

回到仓库 **Settings** → **Secrets and variables** → **Actions** → **Variables**，把 `SITE_URL` 改成你的真实域名：

```
SITE_URL=https://anvilquestwiki.wiki
```

然后触发一次重新部署（`git commit --allow-empty -m "update SITE_URL" && git push`）。

> ⚠️ 这一步必做——否则 sitemap 里的 URL 还是 `*.workers.dev`，影响 SEO。

---

## 方式二：Wrangler CLI 手动部署（本地/进阶）

适合不想用 GitHub Actions、想在本地直接部署的场景。

### 前提

```bash
# 登录（会打开浏览器授权）
npx wrangler login
```

### 部署

```bash
# 设置好 SITE_URL 等环境变量后构建
SITE_URL=https://anvilquestwiki.wiki pnpm build

# 部署（读取 wrangler.toml 里的项目名，不用额外传参）
npx wrangler deploy
```

首次部署会自动创建 Worker。之后每次部署就是"设环境变量 build，然后 deploy"两行命令。

---

## 方式三：导出静态文件到其他平台

AnvilWiki 是纯静态站点（`dist/`），可以部署到任何静态托管：

| 平台             | 配置                                      | 免费额度     |
| ---------------- | ----------------------------------------- | ------------ |
| **Netlify**      | Build: `pnpm build`，Publish: `dist`      | 100GB/月带宽 |
| **Vercel**       | 自动识别 Astro                            | 100GB/月带宽 |
| **GitHub Pages** | 需配 `base`                               | 100GB/月带宽 |
| **自建 VPS**     | `scp -r dist/ user@vps:/var/www/` + nginx | 看你的 VPS   |

> ⚠️ 只有 Cloudflare（Workers 静态资源同 Pages 一样）是**无限带宽/无限请求数**——Cloudflare 官方文档明确写了「Requests to static assets are free and unlimited」，不占用 Workers Free 计划每天 10 万次调用的额度。其他平台超量后要么限速要么收费。这也是 AnvilWiki 默认推荐 Cloudflare 的原因。

---

## 环境变量清单

**本地开发**：复制 `.env.example` 为 `.env`，填你要用的值，Astro/Vite 会自动读取。

**CI/CD 部署**（方式一）：在仓库 **Settings → Secrets and variables → Actions → Variables** 里配置——`ci.yml` 的 Build 步骤会读取这些值传给 `pnpm build`。这两套（本地 `.env` 和 GitHub Actions Variables）互不影响，各自配各自的。

| 变量                        | 必填 | 说明                                                   |
| --------------------------- | ---- | ------------------------------------------------------ |
| `SITE_URL`                  | ✅   | 站点绝对 URL（含 `https://`，无尾斜杠），影响 sitemap/og:image/robots |
| `PUBLIC_ADSENSE_CLIENT`      | 可选 | AdSense Publisher ID（`ca-pub-XXXXXXXXXXXXXXXX`）      |
| `PUBLIC_ADSENSE_SLOT_STICKY` | 可选 | Sticky 粘顶横幅 slot ID                                |
| `PUBLIC_ADSENSE_SLOT_SIDEBAR`| 可选 | Sidebar 桌面端侧边栏 slot ID                           |
| `PUBLIC_ADSENSE_SLOT_INCONTENT` | 可选 | InContent 文章内 slot ID                            |
| `PUBLIC_GA_ID`              | 可选 | Google Analytics ID（有 cookie，经同意横幅门控）       |
| `PUBLIC_CF_BEACON_TOKEN`    | 可选 | Cloudflare Web Analytics beacon token（无 cookie）     |
| `PUBLIC_GSC_VERIFICATION`   | 可选 | Google Search Console 验证 meta token                 |
| `PUBLIC_SPONSOR_URL`        | 可选 | 赞助/捐赠卡链接（空 = 不渲染）                         |
| `PUBLIC_SPONSOR_IMAGE_URL`  | 可选 | 赞助卡二维码/横幅图（空 = 只显示文字卡）               |
| `PUBLIC_GISCUS_REPO`        | 可选 | Giscus 仓库（`owner/repo`，4 个必填项之一）            |
| `PUBLIC_GISCUS_REPO_ID`     | 可选 | Giscus 仓库 ID（4 个必填项之一）                       |
| `PUBLIC_GISCUS_CATEGORY`    | 可选 | Giscus Discussion 分类名（4 个必填项之一）             |
| `PUBLIC_GISCUS_CATEGORY_ID` | 可选 | Giscus 分类 ID（4 个必填项之一）                       |
| `PUBLIC_GISCUS_MAPPING`     | 可选 | Giscus 页面映射方式，默认 `pathname`（唯一可选项）     |

完整说明见 [`.env.example`](../.env.example)。所有广告/评论变量**留空时对应组件不渲染**——新手可以先不配广告把站上线，后续再加。

**部署认证**（`CLOUDFLARE_API_TOKEN` / `CLOUDFLARE_ACCOUNT_ID`）是单独一套，放在 **Secrets** 而不是 **Variables**，见上方「方式一」Step 4——这两个是 wrangler 部署本身用的凭证，跟上表的站点内容变量是两回事。

---

## 部署后验证清单

部署成功后，逐项检查：

```bash
# 1. 站点可访问
curl -I https://<你的域名>/
# 期望: HTTP/2 200

# 2. sitemap 可访问
curl https://<你的域名>/sitemap-index.xml
# 期望: 返回 XML，含你的所有页面 URL

# 3. robots.txt 可访问
curl https://<你的域名>/robots.txt
# 期望: 含 Sitemap: https://<你的域名>/sitemap-index.xml

# 4. 多语言页面可访问
curl -I https://<你的域名>/ja
curl -I https://<你的域名>/bosses

# 5. 文章页正常
curl -I https://<你的域名>/bosses/emberfang
# 期望: 200，不是 404

# 6. 法律页可访问
curl -I https://<你的域名>/about
curl -I https://<你的域名>/privacy-policy

# 7. 不带斜杠的 URL 不应该 308 跳转
curl -I https://<你的域名>/bosses
# 期望: HTTP/2 200 —— 如果是 308，检查 wrangler.toml 的 [assets] 是否还带着
# html_handling = "drop-trailing-slash"（有可能被误删）
```

### SEO 验证

1. **Google Rich Results Test**：https://search.google.com/test/rich-results
   - 输入你的首页 URL——只会看到 Organization + WebSite，这俩不是 rich-result 类型，显示「No items detected」是正常的，不代表出错
   - 输入一篇文章 URL，验证 Article + BreadcrumbList 有效（这俩才是真正会显示「valid items detected」的类型）

2. **Google Search Console**：
   - 添加你的域名（选"网域"方式 → DNS 验证）
   - 提交 `sitemap-index.xml`
   - 等 24-48 小时看收录情况

### 性能验证

1. **PageSpeed Insights**：https://pagespeed.web.dev
   - 输入你的域名，Lighthouse SEO 应该 100，Performance 应该 ≥ 90
   - Core Web Vitals 全绿（LCP < 2.5s，CLS < 0.1）

---

## 常见问题

### Q: GitHub Actions 里 Build 步骤失败，报 `Cannot find module 'astro:content'`

A: Node 版本可能不对。`ci.yml`/`cd.yml` 都用 `actions/setup-node` 读取仓库根目录的 `.nvmrc`（应该是 `22`），确认这个文件存在且内容正确：

```bash
cat .nvmrc
# 应该看到: 22
```

### Q: 构建/部署失败，报 `ERR_PNPM_IGNORED_BUILDS`

A: pnpm 版本较新，默认拒绝执行依赖的 postinstall 脚本，需要 `pnpm-workspace.yaml` 里的 `allowBuilds` 配置（仓库已自带）。确认文件存在且包含这三项：

```bash
cat pnpm-workspace.yaml
# 应该看到:
# allowBuilds:
#   esbuild: true
#   sharp: true
#   workerd: true
```

`workerd` 是 `cd.yml` 部署时 wrangler 自己的依赖需要的，只跑本地/CI 检查（`ci.yml`）不会触发这条。

### Q: 部署成功但页面 404

A: 检查 `wrangler.toml` 的 `[assets] directory` 是不是 `./dist`（不是 `./public` 或别的目录），以及 `pnpm build` 确实在部署前跑过、`dist/` 里有内容。

### Q: 图片不显示 / og:image 抓不到

A: og:image 必须是**绝对路径**。确认：

1. `SITE_URL`（GitHub Actions Variable）已配为最终域名并重新部署过
2. `public/images/hero.webp`（或你的封面图）确实存在且不是 0 字节占位文件
3. 用 `curl` 检查：`curl -I https://<你的域名>/images/hero.webp` 应返回 200

### Q: 为什么这个模板用 Workers 部署，而不是更简单的 Cloudflare Pages Git 自动连接？

A: **已修复，这就是为什么。** 早期版本用 Cloudflare Pages（配套「连一下 GitHub 仓库，之后自动构建部署」的一键式体验），但 Pages 有一个无法干净修复的固有 bug：访问不带斜杠的 URL（如 `/bosses/emberfang`）会 308 跳转到带斜杠的形式（`/bosses/emberfang/`），无论 `astro.config.ts` 里 `trailingSlash: 'never'` 怎么设——这个设置**只控制 Astro 自己生成的 `<a href>`**，不影响 Cloudflare 怎么处理构建产物的目录形态文件（`dist/bosses/emberfang/index.html`）。这是所有走目录形态静态资源托管的通用行为，不是配置错误。

**排查过程中验证并排除的方案**：

- `build.format: 'file'`（扁平文件，`dist/bosses/emberfang.html`，不再有 308）——看似解决了，但 Astro 在这个模式下 `Astro.url.pathname` 构建期会带字面的 `.html` 后缀，而 `canonical`、`og:url`、语言切换器的自引用链接全依赖它拼 URL，全站这些 URL 会错误地带上 `.html`，比原问题更严重。
- Pages 的 `public/_redirects` 文件（200 状态码 rewrite）——已实测：Cloudflare 的目录型静态资源自动跳转对 rewrite 目标同样生效，绕不过去。

**现在用的方案**：迁移到 Cloudflare **Workers + Static Assets**（Cloudflare 目前推荐的新架构），`wrangler.toml` 里设 `html_handling = "drop-trailing-slash"`——访问不带斜杠的路径直接 200 返回内容，带斜杠的形式反过来 307 跳转到不带斜杠的版本，正好匹配 `trailingSlash: 'never'` 的设计初衷。已经在真实项目里验证生效（隔离测试 + 生产切换都确认过），模板的 `wrangler.toml`/`ci.yml`/`cd.yml` 已经是这套配置，fork 出去直接可用，不需要再折腾这部分。

### Q: sitemap 里的 URL 还是 `*.workers.dev` 而不是自定义域名

A: `SITE_URL`（GitHub Actions Variable）没更新，或更新后没触发重新部署。改完必须 push 一个新 commit（哪怕是空的）才会重新构建部署，Variable 改动本身不会自动触发。

### Q: 日文页面显示英文 fallback

A: 这是设计行为，不是 bug。参见 [PRD §9.3](./PRD.md#93-文章-fallback-机制)：单篇文章缺失时自动回退英文，保证 URL 不 404；列表页不回退（该语言没内容就显示空状态）。

### Q: 我想加 Content-Security-Policy（CSP）

A: 模板默认不带 CSP（`public/_headers` 里已有 COOP/nosniff/XFO/Referrer-Policy 四条基础头）。如果自建 CSP，注意模板有**内联脚本**（防 FOUC 主题初始化、主题切换、搜索、AdSense/giscus 按需加载），`script-src` 需要 `'unsafe-inline'`（或逐脚本 hash）；开启广告还要放行 `pagead2.googlesyndication.com` 系域名，开启评论放行 `giscus.app`，开启 GA 放行 `googletagmanager.com`。一个可用起点（在 `public/_headers` 按路径追加，改完重新部署并逐项验证主题切换/搜索/评论/广告）：

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' pagead2.googlesyndication.com static.cloudflareinsights.com www.googletagmanager.com giscus.app; style-src 'self' 'unsafe-inline'; img-src 'self' data: i.ytimg.com pagead2.googlesyndication.com; frame-src youtube-nocookie.com giscus.app; connect-src 'self' cloudflareinsights.com region1.google-analytics.com;
```

---

## 下一步

- [套用模板指南](./apply-template.md)：把 demo 站换成真实游戏
- [内容格式](./content-format.md)：怎么写 MDX 文章
- 回到 [README](../README.md)
