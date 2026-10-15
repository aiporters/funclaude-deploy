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

参考 `nginx/funclaude.conf.example`，把 `funclaude.yourdomain.com`、`claudeusercontent.yourdomain.com` 和 `*.frame.yourdomain.com` 换成你的域名。三个 server 块都转到同一个 funclaude 后端，由 funclaude 按 Host 头路由。三类主机名各自的 DNS、证书要求和排错方法见下面「域名配置详解」。

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

## 域名配置详解

一套 funclaude 需要三类主机名。它们都指向同一个 funclaude 后端，由 funclaude 按请求的 Host 头决定映射到 claude.ai 的哪个上游。

| 主机名 | 映射的上游 | 用途 | 环境变量 |
|---|---|---|---|
| `funclaude.yourdomain.com` | `claude.ai` | 主站，必填 | `FUNCLAUDE_DOMAIN` |
| `claudeusercontent.yourdomain.com` | `www.claudeusercontent.com` | 聊天里的 HTML / docs artifact 预览 | `FUNCLAUDE_USERCONTENT_DOMAIN` |
| `*.frame.yourdomain.com` | `<id>.frame.claudeusercontent.com` | Claude Code artifact（`/code/artifact/...`）预览。claude.ai 给每个 artifact 分配一个随机子域 | `FUNCLAUDE_FRAME_DOMAIN` |

### `.env` 示例

```
FUNCLAUDE_COOKIE_SECRET=<任意足够长的随机串>
FUNCLAUDE_DOMAIN=funclaude.yourdomain.com
FUNCLAUDE_USERCONTENT_DOMAIN=claudeusercontent.yourdomain.com
FUNCLAUDE_FRAME_DOMAIN=frame.yourdomain.com
FUNCLAUDE_PROXY_URL=            # 海外机留空；大陆机填 socks5://... 或 http://...
FUNCLAUDE_SITE_PASSWORD=        # 空 = 不启用站点门禁
```

- `FUNCLAUDE_FRAME_DOMAIN` 只填后缀，**不带 `*.`**。funclaude 会把 `<id>.frame.claudeusercontent.com` 改写成 `<id>.frame.yourdomain.com`。
- 它不能等于主域名，也不能是主域名或 artifacts 子域的父域（例如主站是 `claude.example.com` 时不能填 `example.com`），否则启动直接报错退出。
- 后缀不一定要嵌套在主站下面。`frame.yourdomain.com` 和平级的 `frame-claude.yourdomain.com` 都可以，只要下面的泛域 DNS 和证书覆盖的是 `*.<FUNCLAUDE_FRAME_DOMAIN>`。
- 后两个变量留空时容器照样能跑，只是对应的预览功能不可用。

### DNS

- 主站和 artifacts 子域随便配，A 记录或 CNAME、走不走 CDN 都行。
- frame 泛域**必须是直指服务器 IP 的泛域 A 记录**：`*.frame.yourdomain.com → 服务器 IP`。不能经 Cloudflare 一类 CDN 中转，因为 CDN 不会给未登记的随机子域做 TLS，浏览器会报 `refused to connect`。

### 证书

- 证书要同时含两个 SAN：`*.yourdomain.com` 和 `*.frame.yourdomain.com`。通配符只匹配一级，`*.yourdomain.com` 覆盖不到 `abc.frame.yourdomain.com`。
- 用 acme.sh 或 certbot 的 DNS-01 验证，一条命令带两个 `-d` 就能一起签出来，例如：

```bash
acme.sh --issue --dns dns_ali -d yourdomain.com -d '*.yourdomain.com' -d '*.frame.yourdomain.com'
```

### nginx 配置示例

完整文件见 `nginx/funclaude.conf.example`，下面是可以直接套用的最小配置。把三个域名换成你自己的，证书路径按实际情况改。

```nginx
# 主站：映射 claude.ai
server {
    listen 443 ssl http2;
    server_name funclaude.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    client_max_body_size 150M;

    location / {
        proxy_pass http://127.0.0.1:8282;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_buffering off;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }
}

# artifacts 子域：映射 www.claudeusercontent.com
server {
    listen 443 ssl http2;
    server_name claudeusercontent.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8282;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }
}

# frame 泛域：映射 <id>.frame.claudeusercontent.com（Claude Code artifact 预览）
server {
    listen 443 ssl http2;
    server_name *.frame.yourdomain.com;

    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8282;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_buffering off;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }
}

# HTTP → HTTPS
server {
    listen 80;
    server_name funclaude.yourdomain.com claudeusercontent.yourdomain.com *.frame.yourdomain.com;
    return 301 https://$host$request_uri;
}
```

nginx 和 funclaude 都跑在 Docker 里、处于同一个 network 时，把三处 `proxy_pass http://127.0.0.1:8282` 换成容器名 `http://funclaude:8282`。

### nginx 要点

- 三个 `server` 块都 `proxy_pass` 到同一个 funclaude 后端，`proxy_set_header Host $host` 必须保留，funclaude 靠它分流。
- 第三个块 `server_name *.frame.yourdomain.com;`，证书用上面那张双 SAN 的。
- 主站和 frame 块带 `proxy_http_version 1.1`、`proxy_set_header Upgrade $http_upgrade`、`proxy_set_header Connection "upgrade"`，因为有 SSE 和 WebSocket。
- `proxy_buffering off`，`proxy_read_timeout` / `proxy_send_timeout` 放到 600s。
- 80 端口的跳转块 `server_name` 把三类主机名都列上。
- nginx 也跑在 Docker 里、`proxy_pass` 写的是容器名（如 `http://funclaude:8282`）时，funclaude 容器每次重建后要执行 `docker kill --signal=HUP nginx`。nginx 只在加载配置时解析一次容器名，容器换了 IP 它还打旧地址，表现为 502。

### 预览打不开时先查这三项

1. `dig abc.frame.yourdomain.com` 要直接返回服务器 IP，不是 CDN 的 IP。
2. `openssl s_client -connect 服务器IP:443 -servername abc.frame.yourdomain.com </dev/null | openssl x509 -noout -ext subjectAltName`，SAN 列表里要有 `*.frame.yourdomain.com`。
3. `docker compose logs funclaude | grep "funclaude starting"`，启动行里 `usercontent:` 和 `frame:` 两个字段都要非空。

三项都正常，artifact 预览就能出来。

## License

[MIT](./LICENSE)

## 致谢 · 友情链接

- 灵感来源：[fuclaude](https://github.com/wozulong/fuclaude)
- 友好社区：[linux.do](https://linux.do/)