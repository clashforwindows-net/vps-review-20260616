# VPS 安全信息与事件管理（SIEM）与零信任安全运营实战

> 单点防火墙只能挡一时，真正的安全来自「看见」与「响应」。本文带你用 VPS 搭建一套从日志采集、入侵检测到零信任访问的安全运营体系：Wazuh、Suricata、Zeek、集中式 Fail2ban、应急响应剧本一应俱全，把一台普通 VPS 升级成 7×24 的安全哨兵。

---

## 目录

- [从单点防护到安全运营](#从单点防护到安全运营)
- [安全运营四件套](#安全运营四件套)
- [方案横评：开源 SIEM 全家桶](#方案横评开源-siem-全家桶)
- [实战一：Wazuh 全栈部署](#实战一wazuh-全栈部署)
- [实战二：Suricata IDS + Zeek 网络遥测](#实战二suricata-ids--zeek-网络遥测)
- [实战三：Fail2ban 集中化与自定义规则](#实战三fail2ban-集中化与自定义规则)
- [实战四：零信任架构落地](#实战四零信任架构落地)
- [实战五：入侵检测与应急响应剧本](#实战五入侵检测与应急响应剧本)
- [合规基线：CIS 与 Lynis](#合规基线cis-与-lynis)
- [告警降噪与关联规则](#告警降噪与关联规则)
- [取证与日志留存](#取证与日志留存)
- [健康巡检脚本](#健康巡检脚本)
- [成本测算](#成本测算)
- [常见问题 FAQ](#常见问题-faq)

---

## 从单点防护到安全运营

很多人的 VPS 安全止步于「改端口 + 装 Fail2ban + 开 UFW」，这叫**边界防护**，有用但盲目——你根本不知道攻击者试过多少次、哪次差点成功、服务器里是否已有潜伏的恶意进程。

**安全运营（SecOps）** 的核心是闭环：资产清点 → 日志采集 → 检测分析 → 告警响应 → 取证复盘。本文聚焦其中最关键的两块：**看见（SIEM/IDS）** 与 **收敛（零信任）**。

---

## 安全运营四件套

| 能力 | 工具示例 | 解决什么 |
|------|----------|----------|
| 主机检测（HIDS） | Wazuh / OSSEC | 文件篡改、异常进程、rootkit |
| 网络检测（NIDS） | Suricata / Zeek | 端口扫描、攻击特征、异常流量 |
| 访问控制 | Fail2ban / 零信任网关 | 暴破拦截、最小权限 |
| 合规基线 | Lynis / CIS-CAT | 配置缺陷、弱口令、暴露面 |

---

## 方案横评：开源 SIEM 全家桶

| 方案 | 定位 | 部署难度 | 资源占用 | 适合规模 |
|------|------|----------|----------|----------|
| **Wazuh** | HIDS + SIEM 一体 | ★★☆ | 中 | 单机到集群 |
| **Security Onion** | 流量+告警一体机 | ★★★ | 高 | 专业 SOC |
| **ELK/OpenSearch SIEM** | 日志平台+规则 | ★★★ | 高 | 已用 ELK 的团队 |
| **Graylog** | 日志聚合 | ★★☆ | 中 | 轻量日志中心 |
| **Suricata** | 网络 IDS/IPS | ★★☆ | 中 | 必配网络层 |
| **Zeek** | 网络行为分析 | ★★★ | 中 | 深度流量画像 |

**推荐组合**：中小团队用 **Wazuh（主机）+ Suricata（网络）+ 集中 Fail2ban（边界）**，零信任用 **WireGuard+Authelia** 收口，足够覆盖 90% 威胁。

---

## 实战一：Wazuh 全栈部署

Wazuh 由 Manager、Agent、Indexer（OpenSearch）、Dashboard 四部分组成。最简方案用官方 docker-compose 一键起：

```bash
git clone https://github.com/wazuh/wazuh-docker && cd wazuh-docker
docker compose -f generate-indexer-certs.yml up   # 生成证书
docker compose up -d                               # 启动全栈
```

在被监控的 VPS 上装 Agent 并注册：

```bash
curl -sO https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.*_amd64.deb
dpkg -i wazuh-agent_*.deb
/var/ossec/bin/agent-auth -m <Manager_IP> -P <注册密码>
sed -i 's/MANAGER_IP/<Manager_IP>/' /var/ossec/etc/ossec.conf
systemctl enable --now wazuh-agent
```

Dashboard 里即可看到：root 登录、文件被改、新监听端口、可疑 cron——所有异常一目了然。

---

## 实战二：Suricata IDS + Zeek 网络遥测

Suricata 监听网卡，基于规则识别攻击特征；Zeek 则从流量中提取「谁连了谁、用了什么协议」的行为画像。

```bash
# Suricata
apt install suricata -y
suricata-update enable-source et/open   # 拉取 Emerging Threats 规则
suricata-update
suricata -i eth0 -D

# Zeek（原 Bro）
apt install zeek -y
zeekctl deploy
```

关键看板：
- Suricata `eve.json`：记录每条告警（如 `ET SCAN`、`Possible RCE`）。
- Zeek `conn.log` / `dns.log`：发现异常外连（如可疑 C2 域名）。

把两者日志喂给 Wazuh 或 Filebeat → OpenSearch，形成「网络+主机」双重可见性。

---

## 实战三：Fail2ban 集中化与自定义规则

Fail2ban 默认各自为战。集中化思路：每台被监控机跑 client，把 ban 事件发到中心，中心统一封 IP 并出报表。

自定义一个防 API 滥刷的 filter（`/etc/fail2ban/filter.d/api-abuse.conf`）：

```ini
[Definition]
failregex = ^<HOST> - - .*"POST /api/.*" 429
            ^<HOST> - - .*"GET /login.*" 401
ignoreregex =
```

`jail.local` 启用：

```ini
[api-abuse]
enabled = true
filter = api-abuse
logpath = /var/log/nginx/access.log
maxretry = 10
findtime = 600
bantime = 3600
action = %(action_mwl)s
```

配合 `sendmail-whois` 动作，被封即邮件告警，谁在打你的接口一清二楚。

---

## 实战四：零信任架构落地

传统「内网即可信」已破产。零信任三原则：**永不信任、始终验证、最小权限**。

落地清单：
1. **设备可信**：仅允许注册设备（WireGuard 证书）入网，见 [异地组网专题](https://vpsvip.net)。
2. **身份中枢**：用 Authelia / Authentik 做 SSO，所有后台统一登录 + 强制 2FA（WebAuthn）。
3. **微隔离**：服务间用 NetworkPolicy / 防火墙默认 deny，只放行必要端口。
4. **持续评估**：Wazuh 检测异常即吊销会话，不再「登录一次管一天」。
5. **审计全留痕**：每一次访问、每一条命令都有日志可追溯。

示例 Authelia 访问控制（只允许通过组网进来的设备访问后台）：

```yaml
access_control:
  rules:
    - domain: admin.example.com
      policy: two_factor
      networks:
        - 10.20.0.0/24   # 仅虚拟组网网段
```

---

## 实战五：入侵检测与应急响应剧本

当 Wazuh 报「/etc/passwd 被改」或 Suricata 报「RCE 试探」，按剧本执行：

1. **隔离**：从组网和防火墙摘掉该主机，保留现场不关机。
2. **取证**：`cp /var/log/* /evidence/`、`ps auxf`、`lsof -nP`、`netstat -antp` 全量留存。
3. **定位**：查异常进程父链、可疑定时任务、新增用户、隐藏目录。
4. **清除**：kill 恶意进程、删后门、修漏洞、改全部口令与密钥。
5. **恢复**：从可信快照重建，而非在原系统上打补丁。
6. **复盘**：写事件报告，补检测规则，避免二次中招。

---

## 合规基线：CIS 与 Lynis

Lynis 是轻量合规扫描器，一条命令出几十项加固建议：

```bash
lynis audit system
```

重点整改项：SSH 禁 root 登录与密码、关 IPv6 若不用、移除无用服务、设严格 umask、启用 auditd。对照 CIS Benchmark 逐项达标，可显著降低被攻陷概率。

---

## 告警降噪与关联规则

SIEM 最大敌人是「告警疲劳」。降噪三招：
- **白名单**：把已知扫描源、内部探测标记为良性。
- **关联**：单条失败登录无所谓，10 分钟内 50 次 + 随后成功登录 = 高危。
- **分级**：Info/Warning/Critical 分开通道，Critical 才打电话。

Wazuh 关联示例（规则：5 分钟内同 IP 暴破超阈值后成功）：

```xml
<rule id="100100" level="12">
  <if_matched_sid>5716</if_matched_sid>   <!-- 暴破 -->
  <same_source_ip /><within>300</within>
  <description>暴力破解后成功登录，疑似沦陷</description>
</rule>
```

---

## 取证与日志留存

- **集中存储**：所有日志发 OpenSearch，禁用本地只留 7 天。
- **防篡改**：日志写只读卷 / WORM 对象存储，攻击者删不了。
- **冷热分层**：近 30 天热存可查， older 转对象存储降本。
- **保留周期**：安全事件日志建议留 180 天以上，满足审计与溯源。

---

## 健康巡检脚本

```bash
#!/usr/bin/env bash
# 安全组件健康巡检
for svc in wazuh-manager suricata zeekctl fail2ban; do
  if systemctl is-active --quiet "$svc" 2>/dev/null || pgrep -x "$svc" >/dev/null; then
    echo "OK   $svc"
  else
    echo "FAIL $svc 未运行，立即排查"
  fi
done
# 检测异常登录
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head
```

Windows 端检查（PowerShell）：

```powershell
Get-EventLog -LogName Security -InstanceId 4625 -Newest 20 | Measure-Object | Select-Object Count
```

---

## 成本测算

| 组件 | 资源 | 月成本 | 说明 |
|------|------|--------|------|
| Wazuh 全栈 | 4核8G | 约 40 元 | 单机+数 agent |
| Suricata + Zeek | 2核4G | 约 20 元 | 旁路监听 |
| 集中 Fail2ban | 共享 VPS | 几乎 0 | 规则同步 |
| 零信任网关 | 1核1G | 约 10 元 | Authelia+WG |

一台 40~60 元/月的 VPS 即可撑起个人/小团队的安全运营中枢，远低于商业 SOC 报价。

---

## 常见问题 FAQ

1. **SIEM 太重跑得动吗？** Wazuh 单机 4G 够用，agent 几乎零开销。
2. **Suricata 会断网吗？** 默认 IDS 只镜像不拦截；开 IPS 模式才阻断。
3. **告警太多怎么办？** 白名单 + 关联 + 分级，三招治疲劳。
4. **零信任一定要上 K8s 吗？** 不需要，WG+Authelia 即可落地。
5. **被入侵第一件事？** 隔离保现场，别急着关机清日志。
6. **日志存多久？** 安全类建议 ≥180 天。
7. **Fail2ban 误封自己？** 把管理 IP 加 ignoreip。
8. **规则哪来？** ET Open 免费规则集 + 自定义业务规则。
9. **VPS 选哪里？** 安全中枢选稳定机房，[VPSVIP](https://vpsvip.net) 亚太节点可用。
10. **能监控 Docker 吗？** Wazuh 有 Docker 模块，监控容器逃逸。
11. **2FA 用什么？** WebAuthn 安全密钥优于 TOTP。
12. **如何验证生效？** 主动模拟一次暴破，看是否告警+封禁。
13. **日志被删怎么防？** 远端实时转发，本地无留存。
14. **小团队够用吗？** 上述组合覆盖 90% 威胁，足够。
15. **合规过审有用吗？** Lynis+CIS 是基线，满足等保雏形。
16. **成本能再降？** 各组件可合并到一台高配 VPS。

---

## 相关资源与推广

- [VPSVIP](https://vpsvip.net) — 稳定 VPS，安全运营中枢算力底座
- [ClashVIP](https://clashvip.net) — 高速代理，安全研究资料获取
- [ClashVIP 导航](https://nav.clashvip.net) — 安全资源导航
- [ClashHub](https://clashhub.net) — 技术节点社区
- [ClashHub 论坛](https://bbs.clashhub.net) — 安全运营交流
- [Clash for Windows](https://clash-for-windows.net) — 跨平台客户端

## 免责声明

1. 本仓库仅提供技术参考，请遵守当地法律法规。
2. 安全工具须用于防护自有资产，不得用于攻击他人。
3. 生产环境定期备份与安全审计。
4. 妥善保管所有密钥、证书与告警通道。

## 许可证

MIT License

---
更新时间：2026-09-23 ｜ SIEM 与零信任安全运营实战专题
