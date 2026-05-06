# MnEx 网站部署指南 — Mac Mini + DDNS

> 目标：用 Mac Mini 作为服务器，通过 DDNS 解决家庭宽带无固定 IP 的问题，让 mnex.com 全球可访问。

---

## 整体方案

```
Mac Mini  ──(每5分钟)──→ DuckDNS ──→ mnex.duckdns.org（永远指向最新IP）
                                         ↑
新网 DNS ── CNAME: www.mnex.com ─────────┘
新网 DNS ── URL转发: mnex.com → www.mnex.com
```

---

## 第一步：注册 DuckDNS 并获取 Token

1. 打开 https://duckdns.org
2. 用 GitHub 或 Google 账号登录
3. 在输入框里填 `mnex`，点击「add domain」
4. 页面会显示你的 token（一串字母数字），**复制保存好**
5. 你会拥有 `mnex.duckdns.org` 这个免费域名

---

## 第二步：配置 DDNS 自动更新

回到 Mac Mini 终端：

```bash
cd /Volumes/D/Web_mnex.com
chmod +x ddns-update.sh
```

然后编辑脚本，把 Token 填进去：

```bash
nano ddns-update.sh
```

找到这一行：
```
TOKEN="YOUR_DUCKDNS_TOKEN_HERE"
```

把 `YOUR_DUCKDNS_TOKEN_HERE` 替换成你在 DuckDNS 网站看到的真实 Token。

保存（Ctrl+O → 回车 → Ctrl+X）。

**手动测试一下：**

```bash
./ddns-update.sh
cat ddns.log
```

日志最后一行应该显示 `Update successful → mnex.duckdns.org = 你的公网IP`。

---

## 第三步：安装自动运行（每 5 分钟更新一次）

```bash
# 把 launchd 配置文件复制到系统目录
cp /Volumes/D/Web_mnex.com/com.mnex.ddns.plist ~/Library/LaunchAgents/

# 加载并启动
launchctl load ~/Library/LaunchAgents/com.mnex.ddns.plist
launchctl start com.mnex.ddns
```

验证是否在运行：
```bash
launchctl list | grep mnex
```

---

## 第四步：运行 nginx 部署脚本

```bash
cd /Volumes/D/Web_mnex.com
chmod +x setup-mnex-server.sh
./setup-mnex-server.sh
```

---

## 第五步：设置路由器端口转发

1. Mac 局域网 IP：系统设置 → 网络 → Wi-Fi/以太网 → 详细信息 → IP 地址
2. 登录路由器管理页面
3. 端口转发添加两条：

| 外部端口 | 内部 IP（Mac） | 内部端口 | 协议 |
|---------|---------------|---------|------|
| 80 | Mac 的局域网 IP | 80 | TCP |
| 443 | Mac 的局域网 IP | 443 | TCP |

---

## 第六步：配置新网 DNS

登录新网（xinnet.com）→ 域名管理 → mnex.com → DNS 解析管理。

**添加 CNAME 记录：**

| 主机记录 | 记录类型 | 记录值 | TTL |
|---------|---------|--------|-----|
| www | CNAME | mnex.duckdns.org. | 600 |

注意：记录值最后要加一个点 `.`

**设置 URL 转发（根域名）：**

新网后台找「URL 转发」/「显性URL」/「域名转发」功能：

| 源地址 | 目标地址 | 类型 |
|-------|---------|------|
| mnex.com | http://www.mnex.com | 301 永久 |

> 如果新网不支持根域名的 URL 转发，可以暂时只配置 CNAME 记录，
> 用户访问 `www.mnex.com` 即可。`mnex.com` 后续想办法解决。

---

## 第七步：测试

等 DNS 生效后（5-30 分钟）：

1. 手机用流量访问 `http://mnex.duckdns.org` → 应该看到页面
2. 访问 `http://www.mnex.com` → 应该看到页面
3. 访问 `http://mnex.com` → 应该跳转到 www 或直接看到页面

---

## 第八步：HTTPS（可选，DDNS + Let's Encrypt）

由于用了 DDNS，证书申请需要同时覆盖三个域名：

```bash
brew install certbot

sudo certbot certonly --webroot \
  -w /Volumes/D/Web_mnex.com \
  -d mnex.duckdns.org \
  -d www.mnex.com \
  -d mnex.com
```

拿到证书后：
```bash
# 替换为 HTTPS 配置
NGINX_SERVERS="$(brew --prefix)/etc/nginx/servers"
cp /Volumes/D/Web_mnex.com/mnex.com.nginx.ssl.conf "$NGINX_SERVERS/mnex.com.conf"
nginx -t && nginx -s reload
```

---

## 日常维护

```bash
# 查看 DDNS 日志
cat /Volumes/D/Web_mnex.com/ddns.log

# 查看 nginx 状态
brew services list | grep nginx

# 重载 nginx（改配置后）
nginx -t && nginx -s reload

# 手动更新 DDNS
/Volumes/D/Web_mnex.com/ddns-update.sh
```

---

## 常见问题

**Q: Mac 休眠导致服务中断？**
系统设置 → 电池 → 电源适配器 → 打开「防止自动休眠」

**Q: DDNS 不更新？**
检查 Token 是否正确：`cat /Volumes/D/Web_mnex.com/ddns-update.sh | grep TOKEN`
手动跑一次看日志：`/Volumes/D/Web_mnex.com/ddns-update.sh`

**Q: 外网访问不了？**
1. 先用流量访问 http://mnex.duckdns.org 测试（这个不依赖新网 DNS）
2. 如果 DuckDNS 都访问不了 → 检查路由器端口转发和 Mac 防火墙
3. 如果 DuckDNS 可以但 mnex.com 不行 → 检查新网 DNS 配置

---

## 文件清单

| 文件 | 用途 |
|------|------|
| `index.html` | 网站页面 |
| `setup-mnex-server.sh` | 一键部署 nginx |
| `ddns-update.sh` | DDNS IP 更新脚本 |
| `com.mnex.ddns.plist` | launchd 自动运行配置 |
| `mnex.com.nginx.conf` | nginx HTTP 配置 |
| `mnex.com.nginx.ssl.conf` | nginx HTTPS 配置 |
| `DEPLOY.md` | 本文档 |
