# funclaude

Self-hosted reverse proxy for claude.ai — 独立实现的镜像站，致敬 [fuclaude](https://github.com/wozulong/fuclaude)。

Docker image: [`ghcr.io/aiporters/funclaude`](https://github.com/orgs/aiporters/packages/container/package/funclaude)

## 这是什么

一个 Go 写的反向代理，把 claude.ai 的 web UI 映射到你自己的域名下。主要用途：不方便直连 claude.ai 的环境下自建镜像、会话隔离给多人使用。

## 快速开始

### 1. 前置条件

- 一台装好 Docker 的服务器
- 一个能出墙的上游代理（Cloudflare Warp SOCKS5、Clash、Xray 皆可）
- 一个域名 + 通配证书（推荐 `*.example.com`），用来分别接主站和 artifacts 子域
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

参考 `nginx/funclaude.conf.example`，把 `example.com` 换成你的域名，放到 nginx `sites-enabled/` 后 reload。

## 环境变量

### 必填

| 变量 | 说明 |
|------|------|
| `FUNCLAUDE_COOKIE_SECRET` | Cookie 加密密钥，填任意足够长的随机字符串 |
| `FUNCLAUDE_DOMAIN` | 主域名（例如 `claude.example.com`） |

### 可选

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `FUNCLAUDE_USERCONTENT_DOMAIN` | 空 | artifacts 子域（例如 `claudeusercontent.example.com`）。**留空则 HTML/docs artifacts 预览不可用**，强烈建议配置 |
| `FUNCLAUDE_PROXY_URL` | `socks5://warp:1080` | 出站代理。默认假设 docker 网络里有叫 `warp` 的容器；实际一般要改成自己的代理地址 |
| `FUNCLAUDE_BIND` | `:8282` | 监听地址 |
| `FUNCLAUDE_TIMEOUT` | `600` | 请求超时（秒），SSE 长连接 |
| `FUNCLAUDE_SITE_PASSWORD` | 空 | 站点总门禁，空=不启用 |
| `FUNCLAUDE_ISOLATION_DATA` | `data/conversations.json` | 会话隔离状态持久化路径 |

## 升级

```bash
docker compose pull
docker compose up -d
```

`./data/` 是数据卷，容器替换不丢会话隔离数据。

## 回滚

编辑 `docker-compose.yml`，把 image tag 指向之前的版本：

```yaml
image: ghcr.io/aiporters/funclaude:v0.1.0
```

然后 `docker compose up -d`。每个 `v*` tag 都不可变，永远可拉。

## 镜像 tag 说明

| Tag | 含义 |
|-----|------|
| `latest` | 最新发版（推荐生产用） |
| `v0.1.0` | 冻结版本 |
| `dev` | master 最新（不稳定，开发者自测用） |

## 多租户 / 会话隔离

通过 oauth 登录实现会话隔离（跟 fuclaude 一样）。生成 oauth_token 时指定 `unique_name`：

```bash
curl 'http://127.0.0.1:8282/manage-api/auth/oauth_token' \
  -H 'Content-Type: application/json' \
  -d '{
    "session_key": "sk-ant-sid01-xxx",
    "unique_name": "user1",
    "expires_in": 0
  }'
```

拿回的 login_url 发给用户。不传 `unique_name` 则不隔离（admin 视角）。

可选开关 `FUNCLAUDE_SITE_PASSWORD` 启用站点总门禁。

## License

[MIT](./LICENSE)

## 致谢 · 友情链接

- 灵感来源：[fuclaude](https://github.com/wozulong/fuclaude)（by Neo / pengzhile），独立实现。
- 同好社区：[linux.do](https://linux.do/)
