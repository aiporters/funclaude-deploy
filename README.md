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
- 一个域名 + 通配证书（推荐 `*.yourdomain.com`），用来分别接主站和 artifacts 子域
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

参考 `nginx/funclaude.conf.example`，把 `funclaude.yourdomain.com` 和 claudeusercontent.yourdomain.com 换成你的域名

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
| `FUNCLAUDE_PROXY_URL` | 空（直连） | 出站代理。海外机器留空走直连；大陆机器按需填以下任一：<br>• SOCKS5 无认证：`socks5://host:port`（compose 里有 warp 容器则 `socks5://warp:1080`）<br>• SOCKS5 带认证：`socks5://user:pass@host:port`<br>• HTTP CONNECT 无认证：`http://host:port`<br>• HTTP CONNECT 带认证：`http://user:pass@host:port` |
| `FUNCLAUDE_SITE_PASSWORD` | 空 | 站点总门禁，空=不启用 |

## License

[MIT](./LICENSE)

## 致谢 · 友情链接

- 灵感来源：[fuclaude](https://github.com/wozulong/fuclaude)
- 友好社区：[linux.do](https://linux.do/)