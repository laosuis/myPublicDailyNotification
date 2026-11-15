# 每日定时 HTTP 请求（GitHub Actions）

每天北京时间 10:00，向指定地址发送一个 POST 请求，请求体为 JSON：

```json
{ "body": "我的内容" }
```

不需要自己的服务器，靠 GitHub Actions 的免费额度跑。**所有配置都写在 `.github/workflows/config.env` 里**，不用去 GitHub 网页上配 Secret。

三个文件都在 `.github/workflows/` 同一个目录下：

| 文件 | 作用 |
| --- | --- |
| `.github/workflows/config.env` | 唯一的配置文件（地址、token、文案都在这） |
| `.github/workflows/daily-http.yml` | 执行逻辑，一般不用改 |
| `.github/workflows/README.md` | 本文档（放在这里不会显示在仓库首页，纯备忘） |

---

## 一、配置 `config.env`

打开 `.github/workflows/config.env`，改三个值，提交推送即可生效：

```bash
TARGET_URL="https://example.com/api/webhook"
AUTH_TOKEN=""
PAYLOAD_BODY="我的内容"
```

| 变量 | 必填 | 说明 |
| --- | --- | --- |
| `TARGET_URL` | 是 | 请求地址，**必须带 `http://` 或 `https://`**，否则任务会直接报错退出 |
| `AUTH_TOKEN` | 否 | 填了就自动加 `Authorization: Bearer <token>` 头。留空表示不鉴权 |
| `PAYLOAD_BODY` | 否 | 请求体里 `body` 字段的内容，留空则默认「我的内容」 |

写法注意：

- 值用**英文双引号**包住。遇到空格、`#`、`&`、`?`、中文一律包，最省事。
- 以 `#` 开头是注释，不会被读。
- 改完记得 `git push`，下次定时触发才用新值。

### ⚠️ 关于 `AUTH_TOKEN` 的安全提醒

这个文件是**明文提交到仓库**的，会永久留在 git 历史里。

- 仓库是 **private**：基本没问题。
- 仓库是 **public**：**千万别把真 token 写进去**。GitHub 上有大量机器人 7×24 扫公开仓库里的 token，几分钟就能被捡走。
- token 一旦提交过，就算之后删掉，历史里依然能翻出来——要么立刻去服务端吊销重发，要么先把仓库转 private。

公开仓库的正确做法：`config.env` 里 `AUTH_TOKEN` 留空，token 放 **Settings → Secrets and variables → Actions → 新建 `AUTH_TOKEN`**。workflow 会自动兜底读取，不用改代码（见下方「配置优先级」）。

### 配置优先级

**文件位置**：workflow 会先找 `.github/workflows/config.env`，找不到再找仓库根目录的 `config.env`。两个位置都支持，放在 workflow 旁边更整齐，也不会和根目录其他文件混在一起。

**取值顺序**：

```
config.env 里有值  →  用它
config.env 没写/空 →  用 GitHub 的 Secrets / Variables
两边都没有         →  报错（仅 TARGET_URL）；AUTH_TOKEN 空则不鉴权；PAYLOAD_BODY 空则用「我的内容」
```

所以想混着用也行：地址和文案放 `config.env`，敏感的 token 放 Secret，互不冲突。

---

## 二、启用步骤

1. 把 `.github/workflows/` 下的 `config.env`、`daily-http.yml` 一起提交，推送到仓库的**默认分支**（通常是 `main`）。

   > 定时任务只认默认分支上的 workflow 文件。放在其他分支上不会定时触发。

2. 进仓库 **Actions** 标签页 → 左侧选 **Daily HTTP Request** → 右侧 **Run workflow** 手动跑一次。

3. 日志里能依次看到「已读取 config.env」「目标地址」「请求体」和 `HTTP 200`，就说明通了。

4. 之后每天北京时间 10:00 自动执行，不用管。

---

## 三、验证是否成功

**看运行结果**：Actions → Daily HTTP Request → 点最近一次运行，展开 `Send JSON payload` 步骤。日志会打印：

```
触发时间：2026-11-16 10:00:12 CST
已读取 config.env
目标地址：https://example.com/api/webhook
鉴权：未携带
请求体：{ "body": "我的内容" }
HTTP 200 | 耗时 0.312s
```

- 绿色对勾 = 请求成功（2xx/3xx）。
- 红色叉 = 失败，日志末尾有具体原因。
- 想确认对方真收到了，去自己服务端的访问日志里查，或让接口返回固定内容看日志里的响应。

---

## 四、时间怎么改

GitHub 的 cron **只认 UTC，不支持时区设置**。换算：**UTC = 北京时间 − 8 小时**。中国无夏令时，全年不用调。

| 想要的北京时间 | 要写的 cron |
| --- | --- |
| 每天 09:00 | `0 1 * * *` |
| 每天 10:00（当前） | `0 2 * * *` |
| 每天 20:00 | `0 12 * * *` |

改 `.github/workflows/daily-http.yml` 里的 `- cron: "..."`。五个字段依次是 `分 时 日 月 星期`，例如每周一 10:00 写成 `0 2 * * 1`。

---

## 五、请求内容怎么改

用 `jq` 拼 JSON，不是手写字符串，所以中文、引号、换行都不会拼坏。

**加字段**：改 workflow 里这一行：

```bash
payload=$(jq -n --arg body "$PAYLOAD_BODY" '{body: $body, source: "github-actions"}')
```

**多字段从 config 读**：可以在 `config.env` 里加 `PAYLOAD_TITLE="标题"`，然后：

```bash
payload=$(jq -n --arg body "$PAYLOAD_BODY" --arg title "$PAYLOAD_TITLE" '{body: $body, title: $title}')
```

**改成 GET**：删掉 `-X POST`、`-H "Content-Type: application/json"`、`--data-binary "$payload"` 三行。

**加别的请求头**：在 `headers+=(...)` 后面追加一行，例如 `headers+=(-H "X-Source: github")`。

---

## 六、需要知道的限制

1. **只能访问公网地址**。runner 在云端，`localhost`、`127.0.0.1`、公司内网 IP、家里路由器地址都访问不到。

2. **定时不保证准点**。官方说明高峰期可能延迟几分钟到一小时，需要精确到秒的场景不适合。

3. **60 天无活动自动停用**。仓库连续 60 天没有任何提交，schedule 会被暂停（GitHub 会提前发邮件）。手动跑一次或提交一次代码即可恢复。只影响 `schedule`，手动触发不受影响。

4. **仓库被 fork 后，副本默认不跑定时任务**，需在对方仓库 Actions 里手动启用。

5. **私有仓库消耗额度**：每月 2000 分钟免费（公开仓库不限）。本任务每次十几秒，每天一次，够用。

6. **失败默认不通知**。只在 Actions 页面显示红叉，需要主动告警见下一节。

7. **`config.env` 会进 git 历史**。别放真 token 到公开仓库（见第一节的安全提醒）。

---

## 七、可选扩展：失败告警

在 workflow 末尾追加，失败时发消息：

```yaml
      - name: Notify on failure
        if: failure()
        env:
          WEBHOOK: ${{ secrets.ALERT_WEBHOOK }}
        run: |
          [ -z "${WEBHOOK:-}" ] && exit 0
          curl -s -X POST "$WEBHOOK" \
            -H "Content-Type: application/json" \
            -d '{"msgtype":"text","text":{"content":"每日定时请求失败，请查看 GitHub Actions 日志"}}'
```

`ALERT_WEBHOOK` 填企微/钉钉群机器人地址，消息格式按机器人类型微调。也可以把 webhook 写进 `config.env` 一起管理。

---

## 八、可选：用 Environment 隔离配置

workflow 里已经有一行 `environment: production`。它和 `config.env` **不冲突**——有环境声明只是多开一扇门，让 job 能读到「Environment secrets」，`config.env` 依然是最高优先级。

只有在这些场景才需要它：

- 想区分 test / production 两套地址，同一份代码跑不同环境
- 想给「改配置」加一道人工审批

配置步骤：**Settings → Environments → New environment**，名字填 `production`（要与 workflow 里的一致），进去后可加 Environment secrets / variables。

三个副作用（定时任务的致命点在这）：

- **Required reviewers 一勾，每天 10 点不会自动发请求，而是挂起等人审批**。定时任务务必不勾。
- **Wait timer** 同理，设了延迟就真的等。
- **Deployment branches** 若限制为「Protected branches only」，要确认默认分支受保护。

如果不需要以上能力，直接把这行 `environment:` 删掉也行，对 `config.env` 毫无影响。

---

## 九、常见报错对照

| 现象 | 原因 |
| --- | --- |
| `TARGET_URL 为空` | `.github/workflows/config.env` 没写、写成空字符串，或文件名不是 `config.env` |
| `TARGET_URL 必须以 http:// 或 https:// 开头` | 地址漏了协议头，或写成了 `example.com/api` |
| 日志没有「已读取 config.env」 | 文件没提交、被 `.gitignore` 忽略、或放错目录（只认 `.github/workflows/` 与仓库根目录） |
| `Could not resolve host` | 域名写错，或目标是内网地址 |
| `HTTP 401 / 403` | 需要鉴权：`AUTH_TOKEN` 没填或已失效 |
| `HTTP 404` | 路径写错，注意 `/` 结尾可能有区别 |
| `Operation timed out` | 对方 30 秒内没响应，或屏蔽了 GitHub 出口 IP |
| 运行一直停在等待状态 | 环境开了 Required reviewers / Wait timer，在等审批，不是卡死 |
| 手动能跑，到点不跑 | workflow 不在默认分支，或仓库 60 天无活动 |
| 改了 `config.env` 但没生效 | 忘了 push，或改动在非默认分支上 |
