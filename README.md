# funclaude

claude.ai 镜像站，致敬 [fuclaude](https://github.com/wozulong/fuclaude)。

## 特性

### 多用户 / 会话隔离

通过 oauth 登录实现会话隔离（跟 fuclaude 一样）。

生成 oauth_token 时指定 `unique_name`：

```bash
curl 'http://127.0.0.1:8282/manage-api/auth/oauth_token' \
  -H 'Content-Type: application/json' \
  -d '{
    "session_key": "sk-ant-sid01-xxx",
    "unique_name": "user1",
    "expires_in": 0
  }'
```

拿回的 login_url 发给用户。


## 快速开始

### 1. 前置条件

- 一台装好 Docker 的服务器
- 服务器能访问 claude.ai（海外机器通常直连即可；大陆机器需准备出墙代理，如 Cloudflare Warp SOCKS5）
- 一个域名，下面需要三类主机名：
  - 主站 `funclaude.yourdomain.com`
  - artifacts 子域 `claudeusercontent.yourdomain.com`（聊天内 HTML/docs artifact 预览）
  - frame 泛域 `*.frame.yourdomain.com`（Claude Code artifact 预览；claude.ai 给每个 artifact 分配一个随机子域）
- 证书需同时覆盖 `*.yourdomain.com` 和 `*.frame.yourdomain.com`（通配只匹配一级，两个 SAN 都要有；acme.sh / certbot 的 DNS-01 一条命令可以同时签）
- frame 泛域的 DNS 必须是直指服务器的泛域 A 记录（`*.frame.yourdomain.com → 服务器 IP`）。经 CDN 中转的方案对随机子域不可行，因为 CDN 不会为未登记的主机名提供 TLS
- Nginx 或同类反代

### 2. 克隆和配置

```bash
git clone https://github.com/aiporters/funclaude-deploy.git
cd funclaude-deploy
cp .env.example .env
cp docker-compose.example.yml docker-compose.yml
```

编辑 `.env`，填入必填项（见下面变量表）。

### 3. 启动

```bash
docker compose up -d
docker compose logs -f
```

默认监听 `0.0.0.0:8282`。

### 4. 接 nginx

参考 `nginx/funclaude.conf.example`，把 `funclaude.yourdomain.com`、`claudeusercontent.yourdomain.com` 和 `*.frame.yourdomain.com` 换成你的域名。三个 server 块都转到同一个 funclaude 后端，由 funclaude 按 Host 头路由。

## 环境变量

### 必填

| 变量 | 说明 |
|------|------|
| `FUNCLAUDE_COOKIE_SECRET` | Cookie 加密密钥，填任意足够长的随机字符串 |
| `FUNCLAUDE_DOMAIN` | 主域名（例如 `funclaude.yourdomain.com`） |

### 可选

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `FUNCLAUDE_USERCONTENT_DOMAIN` | 空 | artifacts 子域（如 `claudeusercontent.yourdomain.com`）。**留空则 HTML/docs artifacts 预览不可用**，强烈建议配置 |
| `FUNCLAUDE_FRAME_DOMAIN` | 空 | frame 泛域后缀（如 `frame.yourdomain.com`，不带 `*.`）。claude.ai 会把每个 Claude Code artifact 放在 `<id>.frame.claudeusercontent.com`，funclaude 把它映射到 `<id>.frame.yourdomain.com`。**留空则 Claude Code artifact（`/code/artifact/...`）预览不可用**。需要配套泛域 DNS 与证书，见前置条件 |
| `FUNCLAUDE_PROXY_URL` | 空（直连） | 出站代理。海外机器留空走直连；大陆机器按需填以下任一：<br>• SOCKS5 无认证：`socks5://host:port`（compose 里有 warp 容器则 `socks5://warp:1080`）<br>• SOCKS5 带认证：`socks5://user:pass@host:port`<br>• HTTP CONNECT 无认证：`http://host:port`<br>• HTTP CONNECT 带认证：`http://user:pass@host:port` |
| `FUNCLAUDE_SITE_PASSWORD` | 空 | 站点总门禁，空=不启用 |

## License

[MIT](./LICENSE)

## 致谢 · 友情链接

- 灵感来源：[fuclaude](https://github.com/wozulong/fuclaude)
- 友好社区：[linux.do](https://linux.do/)