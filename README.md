# Android 上 FlClash / Clash Meta 与 AdGuardHomeForRoot 共存去广告指南

适用于：Android + root（KernelSU / Magisk）+ AdGuardHomeForRoot，以及 FlClash 或 Clash Meta for Android（CMFA）。

测试环境：Android 17 + KernelSU；FlClash 0.8.98；Clash Meta for Android 2.11.34.Meta。

---

## 1. 问题现象

代理软件运行时：

- 开启系统代理 → 外网正常，但其他 App 仍然有广告；
- 关闭系统代理 → 广告消失，但外网不通。

## 2. 根因

AdGuard Home 只能过滤 **DNS 查询**。代理软件开启后：

- DNS 由代理软件自己的 `fake-ip` / DoH 解析；
- 或者域名直接交给代理节点在远端解析；
- 系统代理模式下，App 直接把域名交给 `127.0.0.1:7890`。

这三种情况都不会经过 AdGuard Home，所以广告域名照常放行。

关闭代理后，App 走系统 DNS，iptables 把 53 端口重定向到 AGH（`127.0.0.1:5591`），广告被拦；但没有代理，外网自然不通。

**结论：广告过滤必须放在代理软件的规则层（`REJECT`）。DNS 层只能覆盖直连/非代理流量。**

---

## 3. FlClash 方案

### 3.1 规则层拦截（推荐）

FlClash → 工具 → 进阶配置 → 脚本（覆写脚本） → 右上角添加，粘贴：

```js
function main(config, profileName) {
  // 放在所有订阅规则最前面，命中即拒绝
  const adRules = [
    "GEOSITE,category-ads-all,REJECT",
  ];
  const oldRules = Array.isArray(config.rules) ? config.rules : [];
  config.rules = [...adRules, ...oldRules.filter(r => !adRules.includes(r))];
  return config;
}
```

保存脚本后：

配置 → 订阅卡片右边的三个点 → 更多 → 覆写 → 覆写模式选「脚本」→ 选中刚建的脚本 → 保存 → 彻底杀掉 FlClash 后台再重新打开 → 启动连接。

验证规则是否生效：

```bash
su -c 'grep -n "category-ads-all" /data/data/com.follow.clash/files/config.yaml'
```

应能看到：

```yaml
rules:
  - "GEOSITE,category-ads-all,REJECT"
  ...
```

### 3.2 可选：DNS 也接 AdGuard Home

工具 → 进阶配置 → DNS → 打开「覆写 DNS」：

| 字段 | 填什么 |
|---|---|
| 域名服务器（Nameservers） | `127.0.0.1:5591` |
| 默认域名服务器（default-nameserver） | `114.114.114.114`、`119.29.29.29`、`223.5.5.5` |
| 代理域名服务器（proxy-server-nameserver） | `https://doh.pub/dns-query` 或 `223.5.5.5` |

注意：

- 默认域名服务器只能填**纯 IP**（可带端口），不能填 DoH/DoT 网址或域名；
- 必须真正打开「覆写 DNS」开关，只改字段不打开开关无效；
- `fake-ip` 模式下 DNS 主要影响直连解析，广告拦截仍主要靠 3.1 的规则。

### 3.3 TUN 还是系统代理

- **TUN / 虚拟网卡**：全量接管，推荐；
- **系统代理**：只覆盖认 HTTP 代理的 App，不认代理的 App 没有外网；
- TUN 开启时系统代理可以关闭，避免同一份流量被处理两遍；
- FlClash 与 CMFA 不能同时开：两者都要占用 VPN，而且 mixed-port 都是 `7890`。

---

## 4. Clash Meta for Android（CMFA）方案

### 4.1 URL 订阅为什么不能直接改

CMFA 2.11.34 对 URL 类型的配置是只读的（源码中 `configurationEditable = (type != Url)`）：

- 「配置 → ⋮ → 编辑」只修改名称、URL、自动更新，看不到 `rules`；
- `rules` 在 `config.yaml` 里，入口是「编辑 → 浏览文件 → config.yaml」；
- 但 URL 类型只能看、不能改。

### 4.2 复制成 File 配置后再加规则（推荐）

1. 配置 → 订阅右侧「⋮」→ **复制**；
   - CMFA 会生成一个 **File 类型**的本地副本，可直接编辑；
2. 自动进入「参数」页 → 点「浏览文件」→ 打开 `config.yaml`；
3. 在 `rules:` 第一行插入（缩进与原文件一致）：

   ```yaml
   rules:
       - 'GEOSITE,category-ads-all,REJECT'
       - 'DOMAIN,api.ip.sb,REJECT'
       ...
   ```

4. 保存并返回 → 激活这个副本 → 启动。

如果没有可用的文本编辑器：

- 在文件列表里对 `config.yaml` 选「导出」，存到下载目录；
- 用 Termux 或 MT 管理器编辑后，再用「导入」覆盖回去。

注意：File 类型副本**不会自动更新**。原 URL 订阅更新后，需要重新复制并重新加规则。

### 4.3 备选：DNS 覆写 + redir-host（不改规则）

设置 → 覆写 → DNS：

| 字段 | 填什么 |
|---|---|
| 增强模式 | `redir-host` |
| Name Server | `127.0.0.1:5591` |
| Default Name Server | `114.114.114.114`、`119.29.29.29`、`223.5.5.5` |
| Fallback Name Server | 清空或保留 |

这样 CMFA 会把域名交给 AGH 解析，广告域名返回 `0.0.0.0` 直接失败。

- 优点：URL 订阅完全不用改，订阅更新也不影响；覆写保存在 `override.json`。
- 缺点：只拦 DNS 层；App 自带 DoH/DoT 或直接走 HTTP 代理的流量会绕过；`redir-host` 比 `fake-ip` 慢。

---

## 5. 为什么处理后网络会变慢

1. **`redir-host` 比 `fake-ip` 慢**
   - `redir-host` 需要先解析出真实 IP 再连接；
   - `fake-ip` 可以立即返回假 IP，连接时再按域名分流；
   - CMFA 用了 4.3 的 `redir-host` 时，每个新域名都会多一次本地 DNS 往返。

2. **FlClash 打开 DNS 覆写后，整套 DNS 配置被替换**
   - `fake-ip-filter` 从订阅的大列表变成只有 `*.lan`、`localhost.ptlogin2.qq.com` 等极少数条目；
   - 音乐、视频、STUN、NTP 等域名原来会走真实 DNS，现在可能被 fake-ip 接管，部分 App 会变慢或行为异常；
   - `nameserver-policy` 里如果有 `geosite:cn → https://doh.pub/dns-query`，国内域名会走加密 DoH，增加 TLS 握手；
   - `fallback` 到 `tls://8.8.4.4`、`tls://1.1.1.1` 时，部分域名会多一轮查询，网络不通时还会等待超时；
   - `nameserver` 只留 AGH 时，冷缓存多一跳（实测缓存命中约 1ms，首次查询约 35–105ms）。

3. **AGH 本身是一跳额外 DNS**
   - 缓存命中很快，但冷查询取决于 AGH 的上游；
   - 上游慢或使用加密 DNS 时，首次访问会明显变慢。

4. **规则本身开销很小**
   - `GEOSITE,category-ads-all,REJECT` 的匹配开销通常可以忽略；
   - 被 REJECT 的广告连接会快速失败（实测约 0.02 秒）；
   - 某些 App 在广告请求失败后会重试/等待，可能造成轻微延迟。

5. **同时运行两个代理/VPN**
   - FlClash 与 CMFA 同时开会争抢 VPN 和 7890 端口，导致连接异常或变慢。

### 5.1 优化建议

- **只想稳定去广告**：FlClash 关闭「覆写 DNS」，保留 `GEOSITE` 规则。这样恢复订阅原本的 DoH + 完整 fake-ip-filter，通常最快。
- **如果坚持让 AGH 接管 DNS**：
  - 删除 `nameserver-policy` 里的 `geosite:cn → DoH`，让国内域名走 AGH/普通 UDP；
  - 清空 `fallback`，或换成国内 DoH；
  - 恢复订阅原来的 `fake-ip-filter` 列表；
  - 把 AGH 和一个 DoH 一起放进 `nameserver` 做冗余。
- **CMFA**：增强模式改回 `fake-ip`，用 4.2 的 File 副本 + 规则；`redir-host` 仅在没有别的办法时使用。
- 同一时间只开一个代理/VPN。
- 改完 DNS 后清理缓存并重启应用；首次访问会慢一些，之后恢复。

---

## 6. 验证

### FlClash

```bash
su -c 'grep -n "category-ads-all" /data/data/com.follow.clash/files/config.yaml'

# 广告域名：应快速返回 000
curl -x http://127.0.0.1:7890 -o /dev/null -w '%{http_code}\n' https://pagead2.googlesyndication.com/

# 正常站点：应为 204
curl -x http://127.0.0.1:7890 -o /dev/null -w '%{http_code}\n' https://www.google.com/generate_204
```

### CMFA

```bash
su -c 'grep -n "category-ads-all" /data/data/com.github.metacubex.clash.meta/files/processing/config.yaml'
```

启动 CMFA 后，同样用上面的两条 `curl` 测试。

---

## 7. 回滚

- **FlClash**：配置 → 订阅三个点 → 更多 → 覆写 → 关闭覆写，或删除脚本；或删除 `GEOSITE,category-ads-all,REJECT` 规则。
- **CMFA**：激活原来的 URL 订阅；或删除复制的 File 配置；或关闭 DNS 覆写。
- AdGuard Home 模块本身不需要修改。

## 8. 安全提示

- 不要把订阅链接、token、节点密码提交到公开仓库。
- 本指南不包含任何订阅凭据。
