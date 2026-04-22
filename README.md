# NodeSeek 交易区关键词监控

通过 RSS 监控 NodeSeek 交易区新帖，命中关键词后用 Telegram Bot 推送。支持白名单、动态命令管理、排除词过滤。

## 功能

- 标题 + 正文同时匹配，多关键词 OR
- 排除词过滤（优先级高于关键词）
- TG 命令动态增删关键词，改完即时生效
- 白名单保护，只允许指定 TG 用户发命令
- 配置和去重记录持久化，重启不丢
- 轻量：运行内存 30-50MB，小鸡无压力

## 准备工作

1. **创建 Bot**：Telegram 找 `@BotFather` → `/newbot` → 拿到 `TG_BOT_TOKEN`
1. **拿 user_id**：给 `@userinfobot` 发任意消息，记下返回的 `Id`（自己和朋友的都要）

## 部署

### 1. 一键拉取配置文件

```bash
mkdir -p nodeseek-monitor && cd nodeseek-monitor && \
wget -O docker-compose.yml https://raw.githubusercontent.com/merlin-node/ns_monitor/main/docker-compose.yml && \
wget -O .env https://raw.githubusercontent.com/merlin-node/ns_monitor/main/.env.example
```

### 2. 编辑 `.env` 填入你自己的信息

```bash
nano .env
```

> 如果系统没装 nano：`apt install -y nano` (Debian/Ubuntu) 或 `yum install -y nano` (CentOS)。
> nano 操作：方向键移动，直接打字编辑，`Ctrl+O` 保存（回车确认），`Ctrl+X` 退出。

至少要改这两项：

```
TG_BOT_TOKEN=刚才BotFather给你的token
ALLOWED_USER_IDS=你的user_id1,你的user_id2
```

其他的 `KEYWORDS`、`EXCLUDES`、`INTERVAL` 可以先留空或随便填，之后都能用 TG 命令改。

### 3. 启动

```bash
docker compose up -d
docker compose logs -f
```

### 4. 绑定推送目标

TG 里找到你的 bot，发送 `/open`，它会自动把当前聊天设为推送目标。然后发 `/status` 确认运行正常。

## 常用命令

```
/help        查看帮助
/status      查看当前状态
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
```

非白名单用户发命令会被拒绝并记录日志。

## 维护

**更新镜像到最新版：**

```bash
docker compose pull && docker compose up -d
```

**查看日志：**

```bash
docker compose logs -f
```

**重置去重记录（会把当前 RSS 里的新帖当”已见过”）：**

```bash
docker compose down
rm -f data/seen.json
docker compose up -d
```

**彻底重置配置：**

```bash
docker compose down
rm -rf data/
docker compose up -d
```

## 常见问题

**收不到消息？**

- `docker compose logs -f` 看报错
- 发 `/status` 确认开关是开启且 `chat_id` 不为空
- 日志报 `TG sendMessage 失败` 说明机器访问不到 `api.telegram.org`，在 `.env` 里加一行 `HTTPS_PROXY=http://你的代理:端口`

**匹配规则？**

- 标题 + 正文摘要一起匹配，不区分大小写
- 多关键词是 OR 关系，命中任一即推送
- 排除词优先级高于关键词，命中任一排除词直接丢弃

**不想开机启动？**
把 `docker-compose.yml` 里的 `restart: unless-stopped` 改成 `restart: "no"`。

## 资源占用参考

- 镜像大小约 80MB
- 运行内存 30-50MB（compose 里已硬限 128M）
- CPU 轮询瞬时 ~5%，空闲接近 0
