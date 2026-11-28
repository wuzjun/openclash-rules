# GitHub/npm 分流排查记录

> 记录时间：2026-11-28。现象是下载 Codex 时 `github.com` / `registry.npmjs.org` 连接被 RST，`google.com` 却正常，几分钟后自愈。本文记录根因判断和「下次复现时怎么快速定位」。

## 一、现象

- `curl https://github.com` / `api.github.com` / `raw.githubusercontent.com` → 连接被重置（`exit 35` / `SSL_ERROR_SYSCALL`）
- `curl https://registry.npmjs.org` → 000
- `curl https://www.google.com` / `baidu.com` → 200（**同时刻正常**）
- 几分钟后 github/npm 全部恢复 200

## 二、结论（根因）

规则本身没错，github 分流是通的。问题出在**出口节点被 GitHub/npm 间歇性拉黑**：

- github 走 `GEOSITE,github → 🛠 GitHub → 🇯🇵 JP → 日本 A13 移动延迟优化路由`（中转/移动优化共享节点）
- 这类共享节点出口 IP 会被 GitHub/npm 拉黑或限速（GitHub 对机房/中转 IP 敏感，Google 不管）
- 所以表现为「google 正常、github/npm 被 RST」
- `🇯🇵 JP` 是 url-test，每 300 秒重选节点，选到干净节点就自动恢复 → 自愈

## 三、本次确认过的事实（运行态）

调路由器 mihomo API 确认（`192.168.3.1:9090`，secret 见 `nozomi.yaml` 的 `secret:` 字段）：

- `🛠 GitHub` = Selector → `🇯🇵 JP`（alive）
- `🇯🇵 JP` = URLTest → `日本 A13 移动延迟优化路由`
- `🔎 Google` = Selector → `🇯🇵 JP`（同节点，google 正常 → 印证是「节点被 github 拉黑」而非「节点全挂」）
- 规则运行态有 `GeoSite github → 🛠 GitHub`、`GeoSite cloudflare → ☁️ cloudflare`、`GeoSite gfw → 🚀 手动选择`
- DNS 正常：github/npm/google 都解析成 fake-ip（`198.18.x.x`），流量确实进了 OpenClash

## 四、顺带发现的配置缺口（未改，留档）

1. **npm 没有显式规则**：`registry.npmjs.org` 靠 `GEOSITE,cloudflare` / `gfw` 兜底，不稳定。若要显式化：
   ```yaml
   - DOMAIN-SUFFIX,npmjs.org,🛠 GitHub
   - DOMAIN-SUFFIX,npmjs.com,🛠 GitHub
   ```
2. **`SRC-IP-CIDR,192.168.2.0/24,DIRECT` 是旧网段**：当前网络已是 `192.168.3.0/24`（网关/路由器 `192.168.3.1` = ImmortalWrt），这条现在等于没生效。若还想「某网段全直连」，要改成 `.3.0/24`。
3. **mihomo 是 alpha 版**（`alpha-gbc76c13`），稳定性一般，属次要诱因。

## 五、下次复现时快速定位（照抄跑）

```bash
# 1. 看当前分流组选了哪个节点（重点看 github 组 now 指向的节点名）
curl -s -H "Authorization: Bearer <secret>" http://192.168.3.1:9090/proxies | python -m json.tool | grep -A2 -E 'GitHub|cloudflare|Google'

# 2. 看实时日志，确认 github 连接匹配到了哪条规则/哪个节点
curl -s -H "Authorization: Bearer <secret>" http://192.168.3.1:9090/logs

# 3. 看运行态规则是否齐全（github / cloudflare / gfw 是否在列）
curl -s -H "Authorization: Bearer <secret>" http://192.168.3.1:9090/rules | python -m json.tool
```

判断标准：
- 若 github 组 `now` 指向的节点名含「移动/联通/电信优化」这类中转，且 google 同节点正常、github 被 reset → **节点被 github 拉黑**，去 OpenClash 面板把 `🛠 GitHub` 手动换一个节点（优先原生 IP / 美国节点），或等 url-test 重选。
- 若 `GEOSITE,github` 规则从 `/rules` 里消失了 → geosite 数据没加载/下载失败（`geox-url` 指向 `testingcf.jsdelivr.net`，被墙或 CDN 挂了），要换 geosite 下载源。

## 六、若想根治（未实施，方案备选）

给 github/npm 建一个**用 github 地址做健康检查的专属 url-test**，只选「真能连上 github」的节点，从根上避免选到被拉黑节点：

```yaml
proxy-groups:
  - name: 🛠 GitHub
    type: url-test
    url: https://github.com/robots.txt    # 健康检查指向 github 本身
    interval: 300
    tolerance: 50
    include-all: true
    filter: '(?i)(美国|香港|日本|\bUS\b|\bHK\b|\bJP\b)'  # 按需收窄

rules:
  - GEOSITE,github,🛠 GitHub
  - DOMAIN-SUFFIX,npmjs.org,🛠 GitHub
  - DOMAIN-SUFFIX,npmjs.com,🛠 GitHub
```
