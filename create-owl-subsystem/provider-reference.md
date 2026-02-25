# owl 框架可用 Provider 速查

实现功能前先查此表，**已有的直接在 `ServiceProviders()` 中注册使用，不要自己实现**。若不确定某个 Provider 的用法，先读 `owl/provider/<名称>/` 下的源码。

| Provider | 包路径 | 提供能力 | 典型注入类型 |
|----------|--------|----------|-------------|
| `router.RouterServiceProvider` | `provider/router` | HTTP 路由、路由注册、菜单、响应封装 | `*gin.Engine` |
| `db.DBServiceProvider` | `provider/db` | 数据库连接（sqlite/mysql/pgsql）、BaseModel、BaseRepository、AutoMigrate | `*gorm.DB` |
| `redis.RedisServiceProvider` | `provider/redis` | Redis 连接、分布式锁 `LockerFactory` | `*redis.Client`、`redis.LockerFactory` |
| `storage.StorageServiceProvider` | `provider/storage` | 文件上传/下载/删除（local/S3/MinIO/COS/七牛）、`FileHandle`（现成上传接口） | `*storage.StorageManager` |
| `permission.GuardProvider` | `provider/permission` | Casbin RBAC 权限中间件 | 权限中间件 |
| `captcha.CaptchaServiceProvider` | `provider/captcha` | 验证码生成与校验 | `*captcha.Service` |
| `pay.PayServiceProvider` | `provider/pay` | 支付（微信/支付宝等）、回调去重 | `*pay.Pay` |
| `ocr.OcrServiceProvider` | `provider/ocr` | OCR 识别（腾讯/百度/阿里） | `*ocr.Service` |
| `mqtt.MqttServiceProvider` | `provider/mqtt` | MQTT 消息 | MQTT client |
| `rabbitmq.RabbitMQServiceProvider` | `provider/rabbitmq` | RabbitMQ 消息队列 | RabbitMQ connection |
| `socketio.SocketIOServiceProvider` | `provider/socketio` | Socket.IO 实时通信 | Socket.IO server |
| `event.EventServiceProvider` | `provider/event` | 进程内事件总线 | 事件发布/订阅 |
| `conf.ConfServiceProvider` | `provider/conf` | 配置文件加载与热更新 | `*conf.Configure` |
| `log.LogServiceProvider` | `provider/log` | 日志 | Logger |
| `validator.ValidatorServiceProvider` | `provider/validator` | 请求校验（go-playground/validator） | `*validator.Validate` |

**使用方式**：在 SubApp 的 `ServiceProviders()` 返回值中追加需要的 Provider，然后在 service/handle 的构造函数参数中声明所需类型，DI 容器自动注入。

**示例**：CMS 需要文件上传 → `ServiceProviders()` 加 `&storage.StorageServiceProvider{}`，`Binds()` 加 `storage.NewFileHandle`，route 中为 FileHandle 注册上传路由；service 中如需直接操作存储，构造函数参数加 `*storage.StorageManager`。
