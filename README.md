# FileTF

一个轻量级的安全文件传输工具，单个 Go 文件，无需任何依赖。支持 HTTP / HTTPS，通过随机 Token 鉴权，适合在服务器之间快速传输文件。

---

## 特性

- **零依赖**：单文件，只用 Go 标准库，编译即用
- **双模式**：同一个二进制文件既是服务端也是客户端
- **Token 鉴权**：服务端启动时自动生成随机 Token，也可手动指定
- **HTTPS 支持**：原生支持 TLS，兼容 Let's Encrypt 证书和自签名证书
- **目录遍历防护**：下载接口对路径做严格校验，禁止访问存储目录以外的文件
- **上传大小限制**：单次请求最大 2GB，防止磁盘被写满
- **通配符上传**：支持 `*.tar.gz` 等 glob 模式批量上传

---

## 快速开始

### 编译

```bash
go build -o filetf filetf.go
```

交叉编译（例如在 macOS 上编译 Linux 二进制）：

```bash
GOOS=linux GOARCH=amd64 go build -o filetf filetf.go
```

---

## 用法

### 服务端

```
filetf -s [-p 端口] [-d 目录] [-t TOKEN] [-c 文件] [-k 文件]
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-s` | — | 启动服务端模式（必填） |
| `-p` | `9999` | 监听端口 |
| `-d` | `.`（当前目录） | 文件存储目录（上传保存到此，下载从此读取） |
| `-t` | 自动生成 | 鉴权 Token，不填则启动时随机生成并打印 |
| `-c` | — | TLS 证书路径（`.pem`），与 `-key` 配合启用 HTTPS |
| `-k` | — | TLS 私钥路径（`.pem`） |

### 客户端

```
filetf [-tls] [-ca CA文件] -t TOKEN <服务器[:端口]> [get|post] <文件...>
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-tls` | `false` | 使用 HTTPS 连接服务端 |
| `-ca` | — | 自签名证书场景下指定 CA 文件；Let's Encrypt 证书无需此参数 |
| `-t` | — | 鉴权 Token（必填） |
| `get` / `post` | `get` | 操作模式：下载或上传，默认为下载 |

---

## 使用示例

### HTTP 模式（内网）

**启动服务端，自动生成 Token：**

```bash
./filetf -s -p 9999 -d /tmp/share
# 输出：临时 Token: XK7PQ2MN   ← 请复制给客户端
```

**客户端下载：**

```bash
./filetf -t XK7PQ2MN 192.168.1.100:9999 get report.pdf
```

**客户端上传（支持通配符）：**

```bash
./filetf -t XK7PQ2MN 192.168.1.100:9999 post *.tar.gz
```

---

### HTTPS 模式（公网，Let's Encrypt 证书）

**启动服务端：**

```bash
./filetf -s -p 9999 \
  -cert /etc/letsencrypt/live/your.domain.com/fullchain.pem \
  -key  /etc/letsencrypt/live/your.domain.com/privkey.pem \
  -d /tmp/share \
  -t ABC12345
```

**客户端下载：**

```bash
./filetf -tls -t ABC12345 your.domain.com:9999 get backup.tar.gz
```

**客户端上传：**

```bash
./filetf -tls -t ABC12345 your.domain.com:9999 post dist.zip
```

---

### HTTPS 模式（自签名证书）

**生成自签名证书（仅测试用）：**

```bash
openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt \
  -days 3650 -nodes -subj "/CN=localhost"
```

**启动服务端：**

```bash
./filetf -s -p 9999 -c server.crt -k server.key -d ./share -t ABC12345
```

**客户端需要通过 `-ca` 指定证书以验证服务端：**

```bash
./filetf -tls -ca server.crt -t ABC12345 192.168.1.100:9999 get file.zip
```

---

**建议使用 filetf 客户端模式**

### 使用 `wget` / `curl`

Token 通过请求头 `X-Transfer-Token` 传递：

```bash
# wget 下载
wget --header="X-Transfer-Token: ABC12345" \
  "https://your.domain.com:9999/transfer?file=report.pdf" \
  -O report.pdf

# curl 下载
curl -H "X-Transfer-Token: ABC12345" \
  "https://your.domain.com:9999/transfer?file=report.pdf" \
  -o report.pdf

# curl 上传
curl -H "X-Transfer-Token: ABC12345" \
  -F "file=@report.pdf" \
  "https://your.domain.com:9999/transfer"
```

> **注意**：不要将 Token 放在 URL 的 query string 里（如 `?token=xxx`），URL 会出现在服务端日志和浏览器历史中，存在泄露风险。

---

## 接口说明

| 路径 | 方法 | 鉴权 | 说明 |
|------|------|------|------|
| `/` | GET | 否 | 返回 `FileTF: running`，用于确认服务存活 |
| `/transfer?file=<文件名>` | GET | 是 | 下载指定文件 |
| `/transfer` | POST | 是 | 上传文件（multipart/form-data） |

鉴权方式：在请求头中携带 `X-Transfer-Token: <TOKEN>`，Token 错误或缺失返回 `401 Unauthorized`。

---

## 安全说明

- **HTTPS + Token**：Token 在 TLS 加密的请求头中传输，公网使用请务必开启 HTTPS
- **HTTP 模式**：Token 明文传输，仅建议在可信内网使用，服务端启动时会打印警告
- **目录遍历**：下载接口强制使用 `filepath.Base()` 提取文件名，禁止 `../` 等路径穿越
- **Token 生命周期**：Token 仅存活于内存，服务端每次重启自动更换（除非通过 `-t` 手动固定）
- **文件大小**：单次上传请求限制 2GB，超出返回 `400`
