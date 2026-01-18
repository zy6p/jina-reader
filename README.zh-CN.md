# Reader 本地部署版

这是基于 [Jina AI Reader](https://github.com/jina-ai/reader) 的本地化改造版，方便用 Docker 在本地部署使用。

## 它做什么
你可以通过 `http://127.0.0.1:3000/https://google.com` 这样的方式把任意网页转换为适合 LLM 的内容输入，便于 Agent 和 RAG 系统处理网页信息。

## 主要特性
- 本地 Docker 运行
- 无需 API Key，开箱即用
- 截图保存到本地，不上传到云存储
- 返回截图的可下载 URL
- 支持将网页转为 LLM 友好的内容格式

## 本地改动（相对上游）
- Docker 镜像使用 apt 安装 Chromium（不依赖 Google Chrome 官方仓库）。
- Docker 基础镜像可通过 `BASE_IMAGE` 覆盖（默认使用 SWR 镜像）。
- 默认端口为 3000（`PORT=3000`），容器入口为 `node build/server.js`。
- 截图保存到 `/app/local-storage/instant-screenshots` 并返回本地 URL。
- 增强了请求头控制：
  - `X-Referer`、`X-Accept-Language`、`X-Set-Cookie` 会透传给浏览器。
  - `X-Extra-Headers` 支持传入任意额外请求头（JSON 字符串）。
- Cookie 解析支持标准 `Set-Cookie` 字符串，并在缺失时补齐 URL。
- 通过环境变量设置默认浏览器指纹：`READER_USER_AGENT`、`READER_ACCEPT_LANGUAGE`、`READER_TIMEZONE`、`READER_VIEWPORT`。
- Docker 镜像默认清空代理相关环境变量，避免宿主机代理意外影响无头浏览器请求。

## 限制
- 目前不支持解析 PDF

## 演示环境
线上演示环境运行在一台 VPS 上，配置如下：
- CPU: 1 vCore
- RAM: 0.5 GB
- Web Server: nginx

这证明即便在最小资源配置下也能稳定运行。

## Docker 部署

### 方案 1：使用预构建镜像
1. 拉取镜像：
   ```bash
   docker pull ghcr.io/intergalacticalvariable/reader:latest
   ```
2. 运行容器：
   ```bash
   docker run -d -p 3000:3000 -e PORT=3000 -v /path/to/local-storage:/app/local-storage --name reader-container ghcr.io/intergalacticalvariable/reader:latest
   ```
   将 `/path/to/local-storage` 替换为你想保存截图的目录。
3. 停止容器：
   ```bash
   docker stop reader-container
   ```
4. 再次启动：
   ```bash
   docker start reader-container
   ```

### 方案 2：本地构建镜像
1. 克隆仓库：
   ```bash
   git clone https://github.com/intergalacticalvariable/reader.git
   cd reader
   ```
2. 构建镜像：
   ```bash
   docker build -t reader .
   ```
   可选：使用其他基础镜像（如内网镜像源）：
   ```bash
   docker build --build-arg BASE_IMAGE=swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/library/node:18-slim -t reader .
   ```
3. 运行容器：
   ```bash
   docker run -p 3000:3000 -v /path/to/local-storage:/app/local-storage reader
   ```

## 使用示例
容器启动后，可用 curl 请求不同的输出格式：

1. Markdown（绕过 readability 处理）：
   ```bash
   curl -H "X-Respond-With: markdown" 'http://127.0.0.1:3000/https://google.com'
   ```

2. HTML（返回 documentElement.outerHTML）：
   ```bash
   curl -H "X-Respond-With: html" 'http://127.0.0.1:3000/https://google.com'
   ```

3. Text（返回 document.body.innerText）：
   ```bash
   curl -H "X-Respond-With: text" 'http://127.0.0.1:3000/https://google.com'
   ```

4. 视口截图（返回截图 URL）：
   ```bash
   curl -H "X-Respond-With: screenshot" 'http://127.0.0.1:3000/https://google.com'
   ```

5. 全页截图（返回截图 URL）：
   ```bash
   curl -H "X-Respond-With: pageshot" 'http://127.0.0.1:3000/https://google.com'
   ```

## 请求头与参数

### 输出格式与响应类型
- `X-Respond-With`：指定输出格式，支持 `markdown` / `html` / `text` / `pageshot` / `screenshot`。
- `X-Return-Format`：`X-Respond-With` 的兼容别名。
- `Accept`：指定响应类型，支持 `text/event-stream`、`application/json`/`text/json`、`text/plain`。

### 反爬请求头（可选）
让请求更接近真实浏览器：

```bash
curl \
  -H "X-User-Agent: <your real UA>" \
  -H "X-Referer: https://help.aliyun.com/" \
  -H "X-Accept-Language: zh-CN,zh;q=0.9,en;q=0.8" \
  -H "X-Set-Cookie: cookie1=...; Path=/; Domain=help.aliyun.com, cookie2=...; Path=/; Domain=help.aliyun.com" \
  -H "X-Extra-Headers: {\"sec-ch-ua\":\"\\\"Chromium\\\";v=\\\"143\\\"\",\"x-custom\":\"value\"}" \
  'http://127.0.0.1:3000/https://help.aliyun.com/zh/xxx'
```

辅助脚本：如果你有浏览器导出的 curl（例如 `example-curl.sh`），可以转为环境变量：
```bash
node scripts/example-curl-to-env.mjs example-curl.sh
```
然后将输出映射到 `X-User-Agent`、`X-Referer`、`X-Accept-Language`、`X-Extra-Headers`。

### CSS 选择器
- `X-Wait-For-Selector`：等待指定元素出现后再返回。
- `X-Target-Selector`：仅返回匹配选择器的内容（会隐式设置 `X-Wait-For-Selector` 为同值）。
- `X-Remove-Selector`：从 HTML 中移除指定元素。

说明：多个选择器用 `, ` 分隔。

### 代理配置
- `X-Proxy-Url`：为单次请求指定代理，支持 `http` / `https` / `socks4` / `socks5`。
  - 带认证示例：`https://user:pass@host:port`

注意：容器内默认清空 `http_proxy` / `https_proxy` / `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` / `no_proxy`，避免宿主机代理泄漏到无头浏览器。如果确有需求，请显式传 `X-Proxy-Url` 或在 Dockerfile / compose 中自行配置。

### 缓存与超时
- `X-Cache-Tolerance`：设置缓存容忍秒数。
- `X-No-Cache`：忽略内部缓存（等价于 `X-Cache-Tolerance: 0`）。
- `X-Timeout`：请求超时秒数，最大 180。

### 其他可选参数
- `X-Set-Cookie`：以标准 `Set-Cookie` 语法为浏览器注入 cookie。
- `X-Keep-Img-Data-Url`：保留 data-url（仅对 markdown 有效）。
- `X-With-Generated-Alt`：为缺少 alt 的图片生成替代文本（当 `X-Respond-With` 指定时无效）。
- `X-With-Images-Summary`：输出图片摘要。
- `X-With-Links-Summary`：输出链接摘要。
- `X-With-Iframe`：允许处理 iframe 内容（必要时会放宽超时限制）。

## 默认浏览器设置（环境变量）
```
READER_USER_AGENT="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
READER_ACCEPT_LANGUAGE="zh-CN,zh;q=0.9,en;q=0.8"
READER_TIMEZONE="Asia/Shanghai"
READER_VIEWPORT="1366x768"
PORT="3000"
```

## 为什么使用 Chromium 而不是 Chrome
- **更容易在容器内安装**：Chromium 可直接通过系统包管理器安装，无需添加 Google 的额外仓库与签名。
- **更适合离线或受限网络环境**：在国内或内网环境中，系统镜像源更稳定，避免拉取 Chrome 官方源失败。
- **更符合发行版与可分发策略**：Chromium 为开源项目，镜像制作与分发更简单。
- **与 Puppeteer 兼容性好**：Puppeteer 对 Chromium 支持成熟，版本匹配更直接。

## 致谢
1. 原始的 [Jina AI Reader](https://github.com/jina-ai/reader)
2. [Harsh Gupta 的改造版](https://github.com/hargup/reader)

## 许可证
本项目与原始 Jina AI Reader 一样使用 Apache-2.0 协议。
