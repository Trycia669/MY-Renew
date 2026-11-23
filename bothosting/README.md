# bot-hosting-renew-public

Bot-hosting 自动续期 —— **公开库只放 Workflow，脚本在私有库**。

```
┌─────────────────────────┐        checkout(repository=私有库, token=PAT)        ┌──────────────────────────┐
│ 公开库(Public)           │ ───────────────────────────────────────────────────▶ │ 私有库(Private)           │
│ .github/workflows/       │                                                    │ app.py                   │
│   bothosting.yml  ← 仅编排     │ ◀────────── xvfb-run python3 app.py ────────────── │ requirements.txt         │
└─────────────────────────┘                                                    └──────────────────────────┘
        账号 / PAT / TG 全在公开库的 Actions Secrets 里（加密，代码里没有明文）
```

好处：公开库可以随便 fork、star、公开分享，别人只看得到流程图和步骤名，看不到 `app.py` 的实现；私有库代码改动后不用碰公开库。

━━━━━━━━━━━━━━━━━━━━━━

## 一、私有库（脚本本体）

新建一个 **Private** 仓库，例如 `yourname/bot-hosting-renew`，把 `private-repo/` 里的文件推上去：

```
app.py
requirements.txt
accounts.example.txt
README.md
```

## 二、公开库（本仓库）

新建一个 **Public** 仓库，只保留：

```
.github/workflows/bothosting.yml
```

## 三、创建 PAT（让公开库能读私有库 + 提交时间戳）

GitHub → Settings → Developer settings → **Personal access tokens**

* **Fine-grained token（推荐）**
  * Repository access：Only select repositories → 只勾 `bot-hosting-renew`
  * Permissions：`Contents: Read and write`（读代码 + keepalive 提交时间戳；只给 Read-only 也能续期，只是私有库时间戳提交会跳过）
* **Classic token**
  * 勾选 `repo`（完整 repo 权限，能用但权限偏大）

把 token 复制好，下一步填进 Secrets。

## 四、公开库的 Secrets 配置

`Settings → Secrets and variables → Actions → New repository secret`

| Secret | 必填 | 说明 |
|--------|------|------|
| `hosting_PRIVATE_REPO` | ✅ | 私有库全名，如 `yourname/bot-hosting-renew` |
| `GH_PAT` | ✅ | 上一步的 token（只读私有库内容即可） |
| `PRIVATE_REF` | ❌ | 私有库分支/标签，默认 `main` |
| `ACCOUNTS` | ✅ | 多账号配置，见下 |
| `EMAIL_1` / `PASSWORD_1` ... | ❌ | 或改用编号 secrets，`bothosting.yml` 已接线 `EMAIL_1..EMAIL_10` / `PASSWORD_n` / `PROXY_n`；超过 10 个请用 `ACCOUNTS` |
| `EMAIL` / `PASSWORD` | ❌ | 单账号向下兼容写法 |
| `NODE_LINK` | ❌ | 代理链接（vless/vmess/trojan/hysteria2/tuic/anytls/socks5），工作流级 sing-box 代理 |
| `TG_BOT_TOKEN` / `TG_CHAT_ID` | ❌ | Telegram 通知 |

`ACCOUNTS` 写法（每行一个账号，`|` 分隔）：

```
a@example.com|password1
b@example.com|password2
c@example.com|password3|http://127.0.0.1:1080
d@example.com|password4|socks5://user:pass@host:port|123456789
```

字段：`EMAIL|PASSWORD[|该账号专用代理][|该账号TG chat_id]`；也支持 JSON 数组：

```json
[{"email":"a@example.com","password":"password1","proxy":"","chat_id":""}]
```

## 五、启用

1. 公开库 `Actions` 菜单 → 允许 workflows。
2. 手动 `Run workflow` 试跑一次（可填 `account_filter` 只跑一个账号验证）。
3. 按自己的到期时间改 `bothosting.yml` 里的 cron（默认每天 UTC 22:30 = 北京 06:30）。

━━━━━━━━━━━━━━━━━━━━━━

## 运行逻辑

1. 校验 `hosting_PRIVATE_REPO` / `GH_PAT` / 账号配置是否齐全，缺一个直接 `::error::` 退出。
2. `actions/checkout` 用 PAT 把私有库拉到 runner 工作目录（`persist-credentials: false`，结束后不留凭证）。
3. **写时间戳保活**：往私有库的 `.github/last-run.txt` 和公开库的 `.keepalive` 各提交一次 UTC 时间戳（见下节）。
4. 装 xvfb/chromedriver、起 sing-box 代理。
5. `xvfb-run python3 app.py`：多账号串行，每账号独立浏览器会话，单账号失败只记结果不中断。
6. 每账号一条 TG 通知（`[序号/总数]` + 脱敏邮箱），结束时一条汇总；全部失败/冷却则非 0 退出。

━━━━━━━━━━━━━━━━━━━━━━

> 全部参数（secrets 清单、账号格式、时序、选择器、状态码、退出码）见私有库的 `PARAMETERS.md`。

## ⏱️ 防 60 天休眠（keepalive）

GitHub 的规则：**仓库连续 60 天没有任何活动（commit），scheduled workflow 会被自动禁用。**
本工作流每次运行都会提交时间戳，两个仓库都保持“有活动”：

| 步骤 | 写入文件 | 推送凭证 | 说明 |
|------|----------|----------|------|
| ⏱️ 私有库时间戳 | 私有库 `.github/last-run.txt` | `GH_PAT` | 内容含 UTC 时间、触发方式、来源库、run 链接 |
| ⏱️ 公开库时间戳 | 公开库 `.keepalive` | 内置 `GITHUB_TOKEN` | 用 GITHUB_TOKEN 推送**不会**触发新 workflow，不会死循环 |

两个步骤都是 `continue-on-error: true`，推送失败（分支保护 / token 只读）只打日志，不影响续期本身。

手动验证：`Run workflow` 时把 `skip_renew` 选 `true`，就只跑 keepalive、跳过浏览器和续期，几秒结束。

### 死锁问题（重要）

keepalive 只能在工作流**还能跑**的时候生效。如果仓库已经 60 天没动、定时任务被 GitHub 停用了，就再也没人写时间戳了——这是循环依赖。

所以额外提供了一个外部触发器：[tools/keepalive-dispatch.sh](tools/keepalive-dispatch.sh)

```bash
export GH_TOKEN=ghp_xxx                        # 需要 actions:write
export REPO=yourname/bot-hosting-renew-public
./tools/keepalive-dispatch.sh                  # 调 workflow_dispatch, skip_renew=true
```

挂到任意一台 VPS / cron-job.org / Cloudflare Worker，**每周跑一次**：

```cron
0 3 * * 1 /path/to/keepalive-dispatch.sh >> /var/log/gh-keepalive.log 2>&1
```

这样即使 GitHub 侧定时任务被停用，外部定时器也会把它重新唤醒，唤醒后工作流自己会补上 commit。

━━━━━━━━━━━━━━━━━━━━━━

## 手动触发参数

| 输入 | 说明 |
|------|------|
| `account_filter` | 只跑指定账号：`1,3` 或 `abc@x.com`，留空跑全部 |
| `headless` | 是否无头，默认 `false`（过头条盾更稳） |
| `skip_renew` | `true` 时只写时间戳不跑续期，用于测试 keepalive |

## 安全说明

* 公开库里**没有任何账号、代理、token 明文**，只有 `${{ secrets.XXX }}` 引用。
* PAT 只给 `Contents: Read-only` 且只授权那一个私有库，泄露面最小；建议 90 天轮换。
* Fork 出来的公开库不会继承你的 Secrets，别人 fork 后跑不起来你的流程。
* 私有库代码永远不出现在公开库的 commit / Release / artifact 里。

## ⚠️ 免责声明

本程序仅供学习了解，非盈利目的，如转载须注明来源。使用本程序须遵守部署服务器所在地、所在国家和用户所在国家的法律法规，作者不对使用者任何不当行为负责。
