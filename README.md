# NodeSeek 交易区关键词监控

通过 RSS 监控 NodeSeek 交易区新帖，命中关键词后用 Telegram Bot 推送。支持白名单、动态命令管理、排除词过滤。

## 功能

- 标题 + 正文同时匹配，多关键词 OR
- 排除词过滤（优先级高于关键词）
- TG 命令动态增删关键词，不用改配置重启
- 白名单保护，只允许指定 TG 用户发命令
- 配置和去重记录持久化，重启不丢
- 轻量：运行内存 30-50MB，2C2G 小鸡无压力

## 准备工作

1. **创建 Bot**：Telegram 里找 `@BotFather` → `/newbot` → 拿到 `TG_BOT_TOKEN`
1. **拿 user_id**：给 `@userinfobot` 发任意消息，记下返回的 `Id`（自己和朋友的都要，用于白名单）

## 部署方式一：使用预构建镜像（推荐）

GitHub Actions 已自动构建好镜像推到 GHCR，直接拉取即可。

在 VPS 上创建目录和文件：

```bash
mkdir nodeseek-monitor && cd nodeseek-monitor
```

创建 `docker-compose.yml`：

```yaml
services:
  nodeseek-monitor:
    image: ghcr.io/merlin-node/ns_monitor:latest
    container_name: nodeseek-monitor
    restart: unless-stopped
    environment:
      TG_BOT_TOKEN: "${TG_BOT_TOKEN}"
      ALLOWED_USER_IDS: "${ALLOWED_USER_IDS}"
      TG_CHAT_ID: "${TG_CHAT_ID:-}"
      KEYWORDS: "${KEYWORDS:-}"
      EXCLUDES: "${EXCLUDES:-}"
      INTERVAL: "${INTERVAL:-120}"
      CATEGORY: "交易"
    volumes:
      - ./data:/data
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: "0.3"
    logging:
      driver: json-file
      options:
        max-size: "5m"
        max-file: "3"
```

创建 `.env`：

```bash
TG_BOT_TOKEN=1234567890:AAExxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ALLOWED_USER_IDS=123456789,987654321
TG_CHAT_ID=
KEYWORDS=甲骨文,oracle,4837
EXCLUDES=求购,收购
INTERVAL=120
```

启动：

```bash
docker compose pull
docker compose up -d
docker compose logs -f
```

更新镜像：

```bash
docker compose pull && docker compose up -d
```

> 如果 `docker pull` 报权限错误，去 GitHub 仓库右侧 Packages → 点进镜像 → Package settings → 拉到底把 Visibility 改成 **Public**。

## 部署方式二：本地构建

```bash
git clone https://github.com/merlin-node/ns_monitor.git
cd ns_monitor
cp .env.example .env
vim .env    # 填 TG_BOT_TOKEN 和 ALLOWED_USER_IDS
docker compose up -d --build
```

此时 `docker-compose.yml` 里要把 `image:` 那行改成 `build: .`。

## 首次使用

启动后给你的 bot 发 `/open`，它会自动把当前聊天设为推送目标。之后所有配置都可以通过 TG 命令改，改完即时生效，重启容器也不会丢（配置存在 `./data/config.json`）。

## 命令列表

```
/help        查看帮助
/chat_id     查看当前 chat_id
/add 词      添加关键词
/del 词      删除关键词
/keys        查看关键词
/clear_keys  清空关键词
/add_ex 词   添加排除词
/del_ex 词   删除排除词
/ex_keys     查看排除词
/clear_ex    清空排除词
/interval N  设置轮询间隔(秒, 最小 30)
/open        开启提醒（并绑定当前聊天）
/close       关闭提醒
/status      查看当前状态
```

非白名单用户发命令会被拒绝并记录日志。

## 常见问题

**收不到消息？**

- `docker compose logs -f` 看报错
- 发 `/status` 确认开关开启且 `chat_id` 不为空
- 日志报 `TG sendMessage 失败`，说明机器访问不到 `api.telegram.org`，在 compose 的 environment 里加一行 `HTTPS_PROXY: "http://你的代理:端口"` 即可

**匹配太多或太少？**

- 匹配对象是标题 + 正文摘要，不区分大小写
- 多关键词是 OR 关系，命中任一即推送
- 排除词优先级高于关键词，命中任一排除词直接丢弃

**想重置去重记录？**
停容器，删 `./data/seen.json`，重启即可。首次启动不会推历史帖，只记录当前 RSS 里已有的 ID，下次轮询起推新增的。

**怎么更新？**

- 只改 `.env`：`docker compose up -d`（会自动重启）
- 改了代码并 push 到 main：等 Actions 构建完后 `docker compose pull && docker compose up -d`

## 资源占用参考

- 镜像大小约 80MB
- 运行内存 30-50MB（compose 里已硬限 128M）
- CPU 轮询瞬时 ~5%，空闲接近 0
