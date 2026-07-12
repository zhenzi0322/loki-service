loki-service
============

[![PyPI version](https://img.shields.io/pypi/v/loki-service.svg)](https://pypi.org/project/loki-service/)
[![Python version](https://img.shields.io/badge/python-3.6%2B-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/pypi/l/loki-service.svg)](https://opensource.org/licenses/MIT)

Python logging handler for [Grafana Loki](https://grafana.com/loki)，支持自定义 HTTP Headers、Basic 认证、异步队列、结构化日志。

## 特性

- **开箱即用** — `LokiLogger` 一行代码完成异步队列 + 控制台 + Loki 推送
- **生产级健壮性** — Loki 异常静默隔离、HTTP 请求超时保护、队列满时非阻塞丢弃、进程退出自动 flush
- **结构化日志** — 自动注入 `path`（调用者文件:行号）、`ip`（主机名）、`trace_err`（异常堆栈）等上下文字段
- **控制台着色** — 内置 `ColorFormatter` 按日志级别彩色输出，也支持自定义 `logging.Formatter`
- **多租户支持** — 通过 `headers` 传入 `X-Scope-OrgID` 实现多租户隔离
- **灵活组合** — 底层 `LokiHandler` / `LokiQueueHandler` 可自由搭配标准 `logging` 模块
- **Loki API V0 & V1** — 支持 Loki < 0.4.0（V0）和 >= 0.4.0（V1）两种协议

## 安装

```bash
pip install loki-service
```

## 快速开始

### LokiLogger（推荐）

开箱即用的生产级 Logger，内置异步队列 + 控制台输出：

```python
from loki_service import LokiLogger

# 完整模式：控制台 + Loki 推送
logger = LokiLogger(
    app_name="my-app",
    loki_url="http://loki:3100/loki/api/v1/push",
    level="INFO",
    env="production",
    headers={"X-Scope-OrgID": "my-tenant"},
    auth=("username", "password"),
)

# 本地模式：仅控制台输出，不启动 Loki 推送（loki_url 默认为 None）
logger = LokiLogger(app_name="my-app")

# 基本用法
logger.info("用户登录成功")
logger.error("请求失败", status_code=500, request_id="abc123")

# 传入 dict 作为消息
logger.warning({"event": "timeout", "cost": 3200})

# 在 except 块中调用，自动捕获异常堆栈到 trace_err 字段
try:
    1 / 0
except ZeroDivisionError:
    logger.error("除零异常")
    # 输出 JSON 中将包含 "trace_err": "Traceback (most recent call last)..."

# 如果不需要异常堆栈，显式关闭
logger.info("普通消息", is_trace=False)

# 额外关键字段直接传入
logger.error("订单失败", order_id="ORD-001", cost=3200)
# 输出: {"msg": "订单失败", "order_id": "ORD-001", "cost": 3200, "path": "...", "ip": "...", "trace_err": "..."}

# 位置参数 + msg 关键字同时传入时，关键字值存为 msg_extra
logger.debug("主消息", msg="补充信息")
# 输出: {"msg": "主消息", "msg_extra": "补充信息", ...}

# 启用 asctime 注入
logger.info("带时间戳", is_asctime=True)
# 输出: {"msg": "带时间戳", "asctime": "2026-07-08 15:30:00.123", ...}
```

### 自动注入字段

`LokiLogger` 的 `info` / `error` / `warning` / `debug` / `critical` / `success` 方法会将消息序列化为 JSON，并自动注入以下上下文字段：

| 字段 | 说明 | 示例 |
|---|---|---|
| `msg` | 消息文本（当 `msg` 参数为 `str` 时） | `"用户登录成功"` |
| `asctime` | 时间戳（毫秒精度），仅在 `is_asctime=True` 时注入 | `"2026-07-08 15:30:00.123"` |
| `msg_extra` | 当位置参数和 `msg=` 关键字同时传入时，关键字值存于此字段 | `"补充信息"` |
| `path` | 调用者文件路径及行号 | `"/app/main.py:42"` |
| `ip` | 主机名（`HOSTNAME` 环境变量或 `socket.gethostname()`，`-` 替换为 `.`） | `"my.server.com"` |
| `trace_err` | 当前异常堆栈（仅在 `except` 块中调用且 `is_trace=True` 时注入） | `"Traceback (most recent call last)..."` |

额外的 `**kwargs` 会直接作为 JSON 字段输出：

```python
logger.error("订单失败", order_id="ORD-001", cost=3200)
# 输出: {"msg": "订单失败", "order_id": "ORD-001", "cost": 3200, "path": "...", "ip": "..."}
```

**LokiLogger 参数说明：**

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `app_name` | `str` | 必填 | 应用名称，作为 Loki 标签 `app` |
| `loki_url` | `str` / `None` | `None` | Loki push API 地址；传 `None` 时仅作为本地日志使用，不启动`Loki`推送 |
| `level` | `str` / `int` | `"DEBUG"` | 日志级别，支持字符串（大小写不敏感）或 `logging.DEBUG` 等整数；无效字符串回退到 `DEBUG` |
| `env` | `str` | `"dev"` | 环境标识，作为 Loki 标签 `env` |
| `extra_tags` | `dict` | `None` | 额外的 Loki 标签 |
| `queue_size` | `int` | `5000` | 异步日志队列大小；队列满时日志静默丢弃，不阻塞业务线程 |
| `enable_console` | `bool` | `True` | 是否同时输出到控制台（`stdout`） |
| `headers` | `dict` | `None` | 自定义 HTTP 请求头 |
| `auth` | `tuple` | `None` | Basic HTTP 认证 `(username, password)` |
| `formatter` | `logging.Formatter` | `None` | 公共自定义 Formatter，同时应用于控制台、文件和 Loki 输出；各输出未指定时使用各自默认格式；传入 `ColorFormatter()` 可启用彩色输出 |
| `timeout` | `float` | `10.0` | HTTP 请求超时秒数，防止 Loki 不可达时阻塞程序退出 |
| `log_file` | `str` / `None` | `None` | 日志文件路径；传 `None` 时不写文件；目录不存在时自动创建 |
| `max_bytes` | `int` | `10485760` (10MB) | 单个日志文件最大字节数，超出后自动转轮 |
| `backup_count` | `int` | `3` | 转轮保留的旧日志文件数（即最多 `backup_count + 1` 个文件） |
| `is_asctime` | `bool` | `False` | 是否在日志 JSON 中自动注入 `asctime` 字段；每次调用时也可通过 `is_asctime=True/False` 单独覆盖 |

### 文件日志（转轮）

通过 `log_file` 参数启用文件日志，内置 `RotatingFileHandler` 自动转轮：

```python
from loki_service import LokiLogger

logger = LokiLogger(
    app_name="my-app",
    log_file="/var/log/my-app/app.log",  # 目录不存在时自动创建
    max_bytes=10 * 1024 * 1024,          # 单文件最大 10MB
    backup_count=5,                        # 保留 5 个旧文件（共 6 个文件）
)

logger.info("这条日志同时输出到控制台和文件")
```

转轮规则：当 `app.log` 达到 `max_bytes` 时，自动重命名为 `app.log.1`，新日志写入新的 `app.log`。最多保留 `backup_count` 个旧文件，最旧的被删除。

### LokiLogger 生命周期管理

`LokiLogger` 在进程退出时（`atexit`）会自动 flush 队列中剩余日志。也可以手动关闭：

```python
logger = LokiLogger(app_name="my-app")
# ... 使用 logger ...
logger.close()  # 停止队列监听，等待剩余日志发送完毕
```

如果需要获取底层 `logging.Logger` 实例（例如添加自定义 Handler）：

```python
raw_logger = logger.get_logger()
```

### 自定义格式化

各输出未指定 `formatter` 时使用各自默认格式。通过 `formatter` 参数可统一自定义：

```python
from loki_service import LokiLogger, ColorFormatter
import logging

# 启用彩色输出（按日志级别着色：DEBUG灰/INFO绿/WARNING黄/ERROR红/CRITICAL紫）
logger = LokiLogger(app_name="my-app", formatter=ColorFormatter())

# 完全自定义格式（同时应用于控制台、文件、Loki）
logger = LokiLogger(
    app_name="my-app",
    formatter=logging.Formatter("%(asctime)s [%(levelname)s] %(message)s"),
)
```

### 重复初始化警告

如果同一个 `app_name` 多次创建 `LokiLogger`，由于 `logging.getLogger()` 返回同一实例，**新的配置参数不会生效**，并在 `DEBUG` 级别输出警告。请确保每个 `app_name` 只初始化一次。

---

### LokiHandler（底层 Handler）

适合需要自行组合 Handler 的场景：

```python
import logging
import loki_service

handler = loki_service.LokiHandler(
    url="http://loki:3100/loki/api/v1/push",
    tags={"application": "my-app"},
    auth=("username", "password"),
    version="1",                              # Loki >= 0.4.0 使用 "1"，< 0.4.0 使用 "0"
    headers={"X-Scope-OrgID": "my-tenant"},
)

logger = logging.getLogger("my-logger")
logger.addHandler(handler)
logger.error("Something happened", extra={"tags": {"service": "my-service"}})
```

**LokiHandler 参数说明：**

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `url` | `str` | 必填 | Loki push API 地址 |
| `tags` | `dict` | `None` | 默认 Loki 标签，每条日志都会携带 |
| `auth` | `tuple` | `None` | Basic HTTP 认证 `(username, password)` |
| `version` | `str` | 自动 | Emitter 版本：`"0"`（Loki < 0.4.0）或 `"1"`（Loki >= 0.4.0）；未指定时使用 `"0"` 并触发 DeprecationWarning |
| `headers` | `dict` | `None` | 自定义 HTTP 请求头 |
| `timeout` | `float` | `5.0` | HTTP 请求超时秒数 |

### SafeLokiHandler

`SafeLokiHandler` 继承自 `LokiHandler`，在 Loki 推送异常时**完全静默**（吞掉所有异常），绝不影响业务逻辑。`LokiLogger` 内部默认使用的就是此 Handler。

```python
from loki_service import SafeLokiHandler

handler = SafeLokiHandler(
    url="http://loki:3100/loki/api/v1/push",
    tags={"app": "my-app"},
    version="1",
)
```

### LokiQueueHandler（异步队列模式）

使用 `LokiQueueHandler` 在独立线程中发送日志，不阻塞主线程：

```python
import logging
import loki_service
from multiprocessing import Queue

handler = loki_service.LokiQueueHandler(
    Queue(-1),
    url="http://loki:3100/loki/api/v1/push",
    tags={"application": "my-app"},
    version="1",
)

logger = logging.getLogger("my-logger")
logger.addHandler(handler)
```

## Loki 标签体系

Loki 使用标签（Labels）对日志流进行索引和查询。以下是各组件的标签行为：

| 标签 | 来源 | 说明 |
|---|---|---|
| `app` | `LokiLogger.app_name` | 应用名称 |
| `env` | `LokiLogger.env` | 环境标识 |
| `host` | `socket.gethostname()` | 主机名 |
| `severity` | 自动注入 | 日志级别小写（`info` / `error` 等） |
| `logger` | 自动注入 | `logging.Logger` 名称 |
| 自定义 | `extra_tags` 或 `extra={"tags": {...}}` | 用户额外标签 |

> 标签名只允许字母、数字和下划线。包含 `.` `-` 空格等的标签名会被自动替换为 `_`。

## 多租户（X-Scope-OrgID）

Loki 多租户模式通过 `X-Scope-OrgID` 请求头区分租户，使用 `headers` 参数传入：

```python
from loki_service import LokiLogger

logger = LokiLogger(
    app_name="order-service",
    loki_url="http://loki:3100/loki/api/v1/push",
    headers={"X-Scope-OrgID": "tenant-abc"},
)

logger.info("订单创建成功", order_id="ORD-001")
```

等效 curl 请求：

```bash
curl -X POST http://loki:3100/loki/api/v1/push \
  -H "Content-Type: application/json" \
  -H "X-Scope-OrgID: tenant-abc" \
  -d '{"streams": [{"stream": {"app": "order-service", "severity": "info"}, "values": [["1234567890000000000", "订单创建成功"]]}]}'
```

## 生产环境注意事项

- **队列溢出保护**：当异步队列满时，新日志会被静默丢弃而非阻塞业务线程，防止 Loki 故障拖垮整个服务。
- **异常隔离**：`SafeLokiHandler` 会吞掉所有 Loki 推送异常，`logging.raiseExceptions` 也被设为 `False`。
- **HTTP 超时保护**：每次 Loki 推送请求默认 10 秒超时（可通过 `timeout` 参数调整），防止 Loki 不可达时 listener 线程无限阻塞导致程序无法退出。
- **自动 flush**：进程退出时通过 `atexit` 自动停止所有 `QueueListener`，确保队列中剩余日志发送完毕。
- **重复初始化**：相同 `app_name` 重复创建 `LokiLogger` 时，新参数不生效，请注意避免。

## 许可证

[MIT License](https://opensource.org/licenses/MIT)

