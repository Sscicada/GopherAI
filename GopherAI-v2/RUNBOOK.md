# GopherAI-v2 本地跑通记录

本文档记录在 Windows 本机跑通 `GopherAI-v2` 的完整过程。示例路径以本机实际目录为准：

```bat
C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2
```

## 1. 已安装的软件

### Go

项目 `go.mod` 使用 Go 1.24 toolchain，因此安装 Go 1.24.x。

本机安装位置：

```bat
D:\env\go1.24.10
```

验证：

```bat
D:\env\go1.24.10\bin\go.exe version
```

期望输出类似：

```text
go version go1.24.10 windows/amd64
```

为了尽量减少 C 盘占用，Go 缓存使用 D 盘目录：

```bat
set GOPATH=D:\env\gopath
set GOCACHE=D:\env\go-build-cache
```

### Node.js / npm

本机已有：

```bat
node --version
npm --version
```

已验证版本：

```text
node v20.15.1
npm 10.7.0
```

### Docker Desktop

本项目推荐用 Docker 跑基础设施。

验证：

```bat
docker version
docker ps
```

如果普通终端访问 Docker 报权限问题，使用管理员身份启动 Docker Desktop，并在管理员 PowerShell/cmd 中执行 Docker 命令。

## 2. 下载/准备的资源

### 项目源码

项目仓库：

```text
https://github.com/youngyangyang04/GopherAI
```

本机源码目录：

```bat
C:\Users\ZL\Documents\GopherAI\source
```

### 图像识别资产

为了跑通图像识别，额外下载并放入：

```text
GopherAI-v2\models\mobilenetv2\mobilenetv2-7.onnx
GopherAI-v2\models\imagenet_classes.txt
GopherAI-v2\models\onnxruntime\onnxruntime.dll
```

来源：

- `mobilenetv2-7.onnx`：ONNX Model Zoo / Hugging Face 镜像
- `onnxruntime.dll`：Microsoft ONNX Runtime Windows x64 release
- `imagenet_classes.txt`：ImageNet 1000 类标签

运行只需要以上 3 个文件，不需要保留下载 zip。

## 3. 基础设施 Docker Compose

在 `GopherAI-v2` 下新增了 `docker-compose.yml`，用于启动：

- MySQL 8.0，端口 `3306`
- Redis Stack，端口 `6379`
- RabbitMQ Management，端口 `5672` / `15672`

启动：

```bat
cd /d C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2
docker compose up -d
```

查看状态：

```bat
docker ps
```

期望看到：

```text
gopherai-mysql
gopherai-redis
gopherai-rabbitmq
```

RabbitMQ 管理台：

```text
http://127.0.0.1:15672
```

账号密码：

```text
root / 123456
```

停止基础设施：

```bat
cd /d C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2
docker compose down
```

## 4. 后端配置

配置文件：

```bat
C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2\config\config.toml
```

关键修改：

```toml
[rabbitmqConfig]
host = "127.0.0.1"
port = 5672
username = "root"
password = "123456"
vhost = "/"
```

原因：后端在 Windows 主机运行，RabbitMQ 在 Docker 容器中暴露到本机端口，所以 host 要使用 `127.0.0.1`。

## 5. 下载后端依赖并编译

进入后端目录：

```bat
cd /d C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2
```

下载 Go 依赖：

```bat
set GOPATH=D:\env\gopath
set GOCACHE=D:\env\go-build-cache
D:\env\go1.24.10\bin\go.exe mod download
```

编译：

```bat
set GOPATH=D:\env\gopath
set GOCACHE=D:\env\go-build-cache
D:\env\go1.24.10\bin\go.exe build -buildvcs=false -o .\gopherai-server.exe .
```

说明：`-buildvcs=false` 用于避免 Git ownership / VCS stamping 导致构建失败。

## 6. 插入测试用户

注册功能依赖邮箱验证码。为了先跑通登录，可以直接插入测试用户。

```bat
docker exec gopherai-mysql mysql -uroot -p123456 GopherAI -e "INSERT INTO users (name,email,username,password,created_at,updated_at) VALUES ('Test User','test@example.com','test','e10adc3949ba59abbe56e057f20f883e',NOW(),NOW()) ON DUPLICATE KEY UPDATE password=VALUES(password), email=VALUES(email), name=VALUES(name), updated_at=NOW(), deleted_at=NULL;"
```

测试账号：

```text
用户名：test
密码：123456
```

密码使用项目里的 MD5 规则，`123456` 的 MD5 是：

```text
e10adc3949ba59abbe56e057f20f883e
```

## 7. 启动后端

后端启动前需要设置模型 API 环境变量。

如果使用阿里云百炼 / DashScope OpenAI 兼容接口：

```bat
cd /d C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2

set OPENAI_API_KEY=你的API_KEY
set OPENAI_MODEL_NAME=qwen-turbo
set OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1

gopherai-server.exe
```

看到以下日志表示后端启动成功：

```text
Listening and serving HTTP on 0.0.0.0:9090
```

注意：

- 不要把 API Key 写入 Git。
- cmd 使用 `set KEY=value`。
- PowerShell 使用 `$env:KEY="value"`。
- 设置环境变量后必须在同一个窗口启动后端。

## 8. 前端安装和启动

进入前端目录：

```bat
cd /d C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2\vue-frontend
```

安装依赖：

```bat
npm install --cache .\.npm-cache
```

如果安装脚本因为权限访问 `C:\Users\ZL` 失败，用管理员终端执行同一条命令。

构建验证：

```bat
npm run build
```

启动开发服务器：

```bat
npm run serve
```

访问：

```text
http://localhost:8080
```

前端代理配置位于：

```bat
vue-frontend\vue.config.js
```

代理规则：

```js
target: 'http://localhost:9090'
pathRewrite: {
  '^/api': '/api/v1'
}
```

## 9. 启动顺序

推荐顺序：

```text
Docker 基础设施 -> Go 后端 -> Vue 前端
```

也就是：

```bat
cd /d C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2
docker compose up -d
```

然后启动后端：

```bat
cd /d C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2
set OPENAI_API_KEY=你的API_KEY
set OPENAI_MODEL_NAME=qwen-turbo
set OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
gopherai-server.exe
```

最后启动前端：

```bat
cd /d C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2\vue-frontend
npm run serve
```

## 10. 功能配置和使用

### 10.1 普通 AI 聊天

必需：

```bat
set OPENAI_API_KEY=你的API_KEY
set OPENAI_MODEL_NAME=qwen-turbo
set OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
```

前端登录后，在聊天页选择普通模型即可。

### 10.2 RAG 文件问答

必需条件：

- 后端启动时设置了 `OPENAI_API_KEY`
- Redis Stack 容器正在运行
- 上传 `.md` 或 `.txt` 文件

前端操作：

1. 登录 `test / 123456`
2. 进入聊天页面
3. 模型选择 `阿里百炼 RAG`
4. 点击上传文档
5. 上传 `.md` 或 `.txt`
6. 提问文档相关问题

RAG 配置：

```toml
[ragModelConfig]
embeddingModel = "text-embedding-v4"
chatModelName = "qwen-turbo"
docDir = "./docs"
baseUrl = "https://dashscope.aliyuncs.com/compatible-mode/v1"
dimension = 1024
```

限制：

- 当前只支持 `.md` / `.txt`
- 每个用户只保留一个 RAG 文件
- 再次上传会删除该用户旧文件和旧索引
- 当前实现未做复杂分块，大文件效果不一定好

测试账号上传文件位置：

```bat
GopherAI-v2\uploads\test
```

### 10.3 图像识别

已修复 Windows 本地运行问题。

默认资产路径：

```bat
models\mobilenetv2\mobilenetv2-7.onnx
models\imagenet_classes.txt
models\onnxruntime\onnxruntime.dll
```

支持环境变量覆盖：

```bat
set IMAGE_MODEL_PATH=models\mobilenetv2\mobilenetv2-7.onnx
set IMAGE_LABEL_PATH=models\imagenet_classes.txt
set ONNXRUNTIME_DLL_PATH=models\onnxruntime\onnxruntime.dll
```

一般不需要手动设置，默认路径已经可用。

使用方式：

1. 重启后端，确保使用新编译的 `gopherai-server.exe`
2. 登录前端
3. 进入图像识别页面
4. 上传 `.jpg` / `.png` 图片

说明：

- 这是 ImageNet 1000 分类模型
- 返回英文类别标签
- 适合识别猫、狗、车、瓶子等分类
- 不是视觉问答模型，不会理解复杂场景

### 10.4 MCP 天气工具

需要单独启动 MCP 服务：

```bat
cd /d C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2\common\mcp
D:\env\go1.24.10\bin\go.exe run . -mode server -http-addr :8081
```

主后端硬编码访问：

```text
http://localhost:8081/mcp
```

MCP 天气工具内部调用：

```text
https://wttr.in
```

因此需要能联网。

### 10.5 TTS 语音

需要百度语音服务配置：

```toml
[voiceServiceConfig]
voiceServiceApiKey = "你的百度 API Key"
voiceServiceSecretKey = "你的百度 Secret Key"
```

修改后需要重启后端。

### 10.6 邮箱验证码注册

如果要使用正常注册流程，需要配置 QQ 邮箱 SMTP 授权码：

```toml
[emailConfig]
authcode = "你的 QQ 邮箱授权码"
email = "你的 QQ 邮箱"
```

注意：这里不是 QQ 密码，而是 QQ 邮箱 SMTP 授权码。

## 11. 常见问题

### 后端启动失败，连不上 MySQL / Redis / RabbitMQ

先检查容器：

```bat
docker ps
```

如果容器没启动：

```bat
cd /d C:\Users\ZL\Documents\GopherAI\source\GopherAI-v2
docker compose up -d
```

### 聊天没有回复或模型调用失败

检查后端启动窗口是否设置了：

```bat
set OPENAI_API_KEY=你的API_KEY
set OPENAI_MODEL_NAME=qwen-turbo
set OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
```

设置后必须重新启动后端。

### cmd 里 `$env:OPENAI_API_KEY=...` 报错

`$env:` 是 PowerShell 写法。cmd 使用：

```bat
set OPENAI_API_KEY=你的API_KEY
```

PowerShell 使用：

```powershell
$env:OPENAI_API_KEY="你的API_KEY"
```

### 前端登录失败

确认：

- 后端正在监听 `9090`
- 前端运行在 `8080`
- 使用测试账号 `test / 123456`

### RAG 上传失败

检查：

- 文件必须是 `.md` 或 `.txt`
- 后端启动时有 `OPENAI_API_KEY`
- Redis Stack 容器在运行
- API Key 有 embedding 模型权限

### 图像识别失败

检查文件是否存在：

```bat
dir models\mobilenetv2\mobilenetv2-7.onnx
dir models\imagenet_classes.txt
dir models\onnxruntime\onnxruntime.dll
```

修改过模型路径或 DLL 路径后，需要重新启动后端。

## 12. 当前已知生成物

运行/构建过程中会出现这些目录或文件：

```text
gopherai-server.exe
models/
uploads/
vue-frontend/node_modules/
vue-frontend/dist/
vue-frontend/.npm-cache/
```

这些是本地运行需要或构建生成的文件，通常不应提交到 Git 仓库。
