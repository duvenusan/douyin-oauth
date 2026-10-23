# 回调页怎么挂上 GitHub Pages

抖音授权需要一个 `https` 的回调地址。这个目录就是那个回调页——**纯静态、无后端、无依赖**，
所以任何静态托管都能放。下面用 GitHub Pages，因为免费且不用绑卡。

整个目录只有两个文件，都**不含任何密钥**，公开也没关系：

```
deploy/
├── index.html      根路径落地页（不放也行，只是直接访问域名时不会 404）
└── callback.html   真正干活的：接住 code、显示、一键复制命令
```

---

## 一、已经挂好了

本项目**已部署**，正式回调地址是：

```
https://duvenusan.github.io/douyin-oauth/callback.html
```

仓库：<https://github.com/duvenusan/douyin-oauth>（Public，Pages 已开，main / root）

浏览器打开上面那个地址，应该看到「这是回调页 / 没有授权参数，正常」。看到就说明可用。

> 以后若要改页面内容：改 `deploy/callback.html` → 提交 → 推送到该仓库，
> 1 分钟内生效。不需要重新配置 Pages。

---

## 二、把这个地址填进两个地方

### 1）抖音开放平台后台

应用详情 → 基本信息 → **授权回调域**

填：`duvenusan.github.io`

> 有些版本的表单要求带协议，那就填 `https://duvenusan.github.io`。
> **以输入框的提示为准**，别自己加路径。
> 回调域是**域级**的，所以不用填 `/douyin-oauth/callback.html` 这一段。

### 2）`collector/config.json`

不用手写，跑向导即可：

```bash
python collector/douyin.py setup
```

它问回调地址时，只填 `duvenusan` 也能自动拼成完整地址。手工版长这样：

```json
{
  "douyin": {
    "client_key": "你的 Client Key",
    "client_secret": "你的 Client Secret",
    "redirect_uri": "https://duvenusan.github.io/douyin-oauth/callback.html",
    "scope": "user_info,video.list,video.data"
  }
}
```

**两处必须是同一个域名。** 回调域是域级，redirect_uri 是完整地址；
只要 redirect_uri 落在这个域下面就行。

---

## 三、然后就能跑通了

```bash
python collector/douyin.py auth
```

**只需要这一条。** 它会自己开浏览器、自己接住授权码、自己换 token、自己拉第一次数据。
你要做的只有两件：在抖音弹窗里点「同意授权」，然后回终端看结果。

页面上会直接显示「已经送到你的终端了」，不用复制任何东西。

原理：脚本在本机 `127.0.0.1:8724` 开了个只收授权码的小服务（只绑回环地址，
不对外、不出网），端口号藏在 `state` 里带过来，回调页用一个 1×1 图片把码打回去。

之后：

```bash
python collector/douyin.py sync --loop 600
```

每 10 分钟自动同步一次，工作台刷新就是最新的。

### 手机上授权怎么办

手机访问不到你电脑的 `127.0.0.1`，这时回调页会显琥珀色的「没连上本地接收端」，
把命令和码摆在你面前，复制到电脑终端粘一次即可 —— 这是唯一的兜底路径，不是 bug。

更省事的做法：`auth` 在等待期间是接受终端输入的，手机上拿到码后发微信「文件传输助手」，
电脑上直接粘进终端回车，脚本会接住。

---

## 四、手机授权的场景

抖音授权经常是在手机上完成（要扫码或跳 App）。这时 code 会显示在手机浏览器里，
电脑上拿不到。

**最省事的做法**：手机上点「复制 code」→ 发到微信「文件传输助手」→ 电脑上复制粘贴。
code 有效期约 10 分钟，够用。

---

## 五、如果 github.io 被驳回

有些平台对境外免费域名有意见。抖音偶尔会以「需备案域名」为由卡 github.io。
真遇到就换这两条，回调页不用改，原样上传：

| 备选 | 地址形态 | 要不要备案 | 备注 |
|---|---|---|---|
| 你自己的域名 + 任意静态空间 | `https://你的域名/callback.html` | 国内服务器要 | 最稳 |
| Cloudflare Pages | `https://项目名.pages.dev/callback.html` | 不用 | 免费，需绑卡验证 |
| Vercel | `https://项目名.vercel.app/callback.html` | 不用 | 免费，国内偶尔慢 |

换完之后，记得**两处一起改**（后台回调域 + config.json），改一处必失败。

---

## 六、排错

| 现象 | 原因 |
|---|---|
| 回调页显示「授权被中断了」 | 点了取消，或链接过期。重跑 `login` |
| 页面报 `redirect_uri` 不一致 | 后台回调域、config.json、实际地址三者没对齐 |
| 跳转后是 GitHub 404 | 仓库里没有 `callback.html`，或 Pages 还没构建完（等 2 分钟） |
| 页面能开，但 `token` 命令报 `code 无效` | code 只能用一次，且约 10 分钟过期。重新走一遍 `login` |
| 手机上授权完，电脑上没法粘 | 走上面第四节的微信文件传输助手 |

---

## 七、安全

- 这两个文件里**没有任何密钥**，Client Secret 只存在你本地的 `collector/config.json`
- 回调页拿到的是**一次性授权码**，用完即废，且 10 分钟过期
- 仓库设成 Public 是为了免费 Pages；如果你介意，可以换成私有仓库 + Cloudflare Pages
- 别把 `collector/config.json` 和 `collector/.douyin-token.json` 传到任何仓库
