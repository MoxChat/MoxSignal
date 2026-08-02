# MoxSignal

[English](./README.md)

> 这是二进制发布仓库。源码、通话架构和信令规格维护在 Mox 主源码仓库；本文只说明当前打包版本的部署方式。

MoxSignal 是 MoxChat 的 WebRTC 信令服务，用于语音通话、视频通话、直播和群聊 SFU 协商。HTTP API 负责通话创建和已认证控制命令，WebSocket API 负责实时交换 offer、answer、ICE、hangup 和 SFU 事件。

## 发布文件

从发布页下载与你的部署目标匹配的文件：

| 目标 | 文件 |
| --- | --- |
| 懒猫微服 | `moxsignal.lpk` |
| Linux x64 | `moxsignal-linux-amd64` |
| Linux arm64 | `moxsignal-linux-arm64` |
| macOS Intel | `moxsignal-darwin-amd64` |
| macOS Apple Silicon | `moxsignal-darwin-arm64` |
| Windows x64 | `moxsignal-windows-amd64.exe` |
| Windows arm64 | `moxsignal-windows-arm64.exe` |

## 懒猫微服部署

1. 下载 `moxsignal.lpk`。
2. 在懒猫应用界面安装，或使用 CLI：

```sh
lzc-cli app install moxsignal.lpk
```

3. 打开分配到的 `moxsignal` 子域名。
4. 检查 `https://<moxsignal-host>/healthz`。

LPK 内包含 MoxSignal 进程和 PostgreSQL 服务，并通过 UDP ingress 暴露 `18982` 端口给群聊 SFU 媒体使用。

## Linux 部署

先创建 PostgreSQL 数据库：

```sql
CREATE USER moxsignal WITH PASSWORD 'change-me';
CREATE DATABASE moxsignal OWNER moxsignal;
```

启动二进制：

```sh
chmod +x ./moxsignal-linux-amd64
export MOXSIGNAL_ADDR=:8982
export MOXSIGNAL_DB_DSN='postgres://moxsignal:change-me@127.0.0.1:5432/moxsignal?sslmode=disable'
export MOXSIGNAL_PUBLIC_BASE_URL='https://moxsignal.example.com'
export MOXSIGNAL_SFU_UDP_PORT=18982
./moxsignal-linux-amd64
```

arm64 主机使用 `moxsignal-linux-arm64`。如果启用群聊 SFU，需要把配置的 UDP 端口暴露给客户端。

## macOS 部署

使用匹配的 macOS 二进制：

```sh
chmod +x ./moxsignal-darwin-arm64
export MOXSIGNAL_ADDR=:8982
export MOXSIGNAL_DB_DSN='postgres://moxsignal:change-me@127.0.0.1:5432/moxsignal?sslmode=disable'
export MOXSIGNAL_PUBLIC_BASE_URL='https://moxsignal.example.com'
./moxsignal-darwin-arm64
```

如果 macOS 拦截下载的二进制，移除 quarantine 属性：

```sh
xattr -d com.apple.quarantine ./moxsignal-darwin-arm64
```

## Windows 部署

先创建 PostgreSQL 数据库，然后在 PowerShell 中启动：

```powershell
$env:MOXSIGNAL_ADDR = ":8982"
$env:MOXSIGNAL_DB_DSN = "postgres://moxsignal:change-me@127.0.0.1:5432/moxsignal?sslmode=disable"
$env:MOXSIGNAL_PUBLIC_BASE_URL = "https://moxsignal.example.com"
$env:MOXSIGNAL_SFU_UDP_PORT = "18982"
.\moxsignal-windows-amd64.exe
```

Windows arm64 主机使用 `moxsignal-windows-arm64.exe`。

如果启用群聊 SFU，需要在 Windows 防火墙和网络边界放行配置的 UDP 端口。

## 环境变量

发布目录中已经包含默认 `.env` 文件。从该目录启动二进制时，MoxSignal 会自动加载 `.env`；如需使用其他文件，可设置 `MOXSIGNAL_ENV_FILE`。默认 `.env` 是部署模板，生产环境必须先替换 `change-me` 数据库密码，并根据网络情况配置公网地址、ICE 和 SFU 参数。

| 变量 | 必填 | 说明 |
| --- | --- | --- |
| `MOXSIGNAL_ADDR` | 否 | HTTP API、WebSocket 信令、健康检查和运维状态页监听地址，默认 `:8982`。 |
| `MOXSIGNAL_DB_DSN` | 是 | PostgreSQL 连接串，用于存储通话会话、参与者、信令事件、API 认证状态和表结构状态。 |
| `MOXSIGNAL_ENV_FILE` | 否 | 指定其他 env 文件路径；不填时服务会在当前目录或上级目录查找 `.env`。 |
| `MOXSIGNAL_PUBLIC_BASE_URL` | 建议填写 | 外部可访问的 HTTPS 地址，会写入通话邀请，确保双方连接到同一个信令中继。 |
| `MOXSIGNAL_RTC_ICE_SERVERS` | 否 | 下发给客户端的 STUN/TURN servers JSON 数组，用于 WebRTC candidate 获取。 |
| `MOXSIGNAL_WS_TOKEN_TTL_SECONDS` | 否 | 安全 API 签发的短期 WebSocket token 有效期，默认 `900` 秒。 |
| `MOXSIGNAL_EVENT_TTL_SECONDS` | 否 | 未投递信令事件保留时间，后台清理超时事件，默认 `120` 秒。 |
| `MOXSIGNAL_START_CALL_RATE_PER_MIN` | 否 | 单身份创建通话会话的限流阈值，默认每分钟 `30` 次。 |
| `MOXSIGNAL_WS_MESSAGE_RATE_PER_SEC` | 否 | 单 WebSocket 连接消息限流，用于降低滥用和异常客户端风险，默认每秒 `60` 条。 |
| `MOXSIGNAL_SFU_UDP_PORT` | 仅 SFU | 群聊 SFU 媒体流固定 UDP 端口；启用群聊媒体时需要暴露该端口。 |
| `MOXSIGNAL_SFU_PUBLIC_IPS` | NAT 后 SFU | SFU 位于 NAT 后时，对外公布到 ICE candidate 的公网 IP，多个用英文逗号分隔。 |
| `MOXSIGNAL_SFU_PUBLIC_IPS_AS_HOST` | 否 | 设为 `true` 时把主机 IP 候选作为公网 SFU candidate 发布，默认 `false`。 |
| `MOXSIGNAL_SFU_UDP4_ONLY` | 否 | 设为 `true` 时 SFU UDP socket 仅使用 IPv4。 |
| `MOXSIGNAL_SFU_TOKEN_SECRET` | 建议填写 | SFU join token 签名密钥；不填时会基于 DSN 生成部署派生的回退密钥。 |
| `MOXSIGNAL_SFU_TOKEN_TTL_SECONDS` | 否 | SFU join token 有效期，默认 `120` 秒。 |
| `MOXSIGNAL_SFU_ICE_DISCONNECTED_SECONDS` | 否 | SFU peer connection 的 ICE disconnected 超时时间，默认 `8` 秒。 |
| `MOXSIGNAL_SFU_ICE_FAILED_SECONDS` | 否 | SFU peer connection 的 ICE failed 超时时间，默认 `25` 秒。 |
| `MOXSIGNAL_SFU_ICE_KEEPALIVE_SECONDS` | 否 | SFU ICE keepalive 间隔，默认 `2` 秒。 |
| `MOXSIGNAL_CHALLENGE_TTL_SECONDS` | 否 | challenge-response 认证验证码有效期，默认 `60` 秒。 |
| `MOXSIGNAL_AUTH_FAIL_WINDOW_MINUTES` | 否 | 按源 IP 统计认证失败次数的时间窗口，默认 `30` 分钟。 |
| `MOXSIGNAL_AUTH_FAIL_BAN_MINUTES` | 否 | 源 IP 超过失败阈值后的临时封禁时长，默认 `30` 分钟。 |
| `MOXSIGNAL_AUTH_FAIL_BAN_THRESHOLD` | 否 | 统计窗口内允许的认证失败次数，超过后封禁该源 IP，默认 `10`。 |

ICE 配置示例：

```sh
export MOXSIGNAL_RTC_ICE_SERVERS='[{"urls":["stun:stun.example.com:3478"]},{"urls":["turn:turn.example.com:3478?transport=udp"],"username":"user","credential":"secret"}]'
```

## 健康检查和运维

- `GET /healthz` 检查 HTTP 进程是否可达。
- `POST /api/secure/health` 检查已认证的信令功能、数据库访问、`wsPath`、ICE 配置和 SFU 媒体配置。
- `GET /` 打开运维状态页。
- `GET /status.json` 返回状态快照。
- `GET /status/stream` 返回实时状态流。

生产环境建议通过 HTTPS 暴露服务，并确认反向代理支持 WebSocket upgrade。MoxChat 客户端应填写外部可访问的信令地址，例如 `https://moxsignal.example.com`。
