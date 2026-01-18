# 📚 Reader: Local Deployment Edition

This is an adapted version of [Jina AI's Reader](https://github.com/jina-ai/reader) for local deployment using Docker. 

## 🎯 What it does
It converts any URL to an LLM-friendly input with `http://127.0.0.1:3000/https://google.com`. Get improved output for your agent and RAG systems at no cost. This tool helps you prepare web content for Large Language Models, making it easier to process and analyze online information.

## 🚀 Key Features
- 🏠 Runs locally using Docker
- 🔑 No API keys required - works out of the box!
- 🖼️ Saves screenshots locally instead of uploading to Google Cloud Storage
- 📥 Provides download URLs for saved screenshots
- 🌐 Converts web content to LLM-friendly formats

## 🔄 Differences vs Upstream (jina-ai/reader)
- Local-first Docker image with Chromium installed via apt (no Google Chrome repo).
- Docker base image is configurable via `BASE_IMAGE` (defaults to an SWR mirror).
- Default port is 3000 (`PORT=3000`) and `node build/server.js` is the container entrypoint.
- Screenshots are saved under `/app/local-storage/instant-screenshots` with local URLs.
- Request header controls expanded:
  - `X-Referer`, `X-Accept-Language`, `X-Set-Cookie` are forwarded to the browser.
  - `X-Extra-Headers` lets you pass any extra headers (JSON string).
- Cookie parsing accepts standard `Set-Cookie` strings and fills a URL when missing.
- Default browser identity can be set via env: `READER_USER_AGENT`, `READER_ACCEPT_LANGUAGE`,
  `READER_TIMEZONE`, `READER_VIEWPORT`.
- Docker image clears proxy envs by default to avoid leaking host proxies into headless requests.

## ⚠️ Limitations
- 📄 Currently does not support parsing PDFs

## 💻 Demo Environment
The live demo is running on a VPS with the following specifications:
- CPU: 1 vCore
- RAM: 0.5 GB
- Web Server: nginx

This demonstrates that the Reader can run effectively even on minimal hardware resources.
## 🐳 Docker Deployment

### Option 1: Using the pre-built image
1. Pull the latest image:
   ```bash
   docker pull ghcr.io/intergalacticalvariable/reader:latest
   ```
2. Run the container:
   ```bash
   docker run -d -p 3000:3000 -e PORT=3000 -v /path/to/local-storage:/app/local-storage --name reader-container ghcr.io/intergalacticalvariable/reader:latest
   ```
   Replace `/path/to/local-storage` with the directory where you want to store screenshots.
   This command does the following:
   - Maps port 3000 of the container to port 3000 on your host
   - Mounts a volume for local storage
   - Names the container `reader-container`
3. To stop the container:
   ```bash
   docker stop reader-container
   ```
4. To start the container again:
   ```bash
   docker start reader-container
   ```

### Option 2: Building the image locally
1. Clone the repository:
   ```bash
   git clone https://github.com/intergalacticalvariable/reader.git
   cd reader
   ```
2. Build the Docker image:
   ```bash
   docker build -t reader .
   ```
   Optional: override the base image mirror:
   ```bash
   docker build --build-arg BASE_IMAGE=swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/library/node:18-slim -t reader .
   ```
3. Run the container:
   ```bash
   docker run -p 3000:3000 -v /path/to/local-storage:/app/local-storage reader
   ```

## 🖥️ Usage
Once the Docker container is running, you can use curl to make requests. Here are examples for different response types:

1. 📝 Markdown (bypasses readability processing):
   ```bash
   curl -H "X-Respond-With: markdown" 'http://127.0.0.1:3000/https://google.com'
   ```

2. 🌐 HTML (returns documentElement.outerHTML):
   ```bash
   curl -H "X-Respond-With: html" 'http://127.0.0.1:3000/https://google.com'
   ```

3. 📄 Text (returns document.body.innerText):
   ```bash
   curl -H "X-Respond-With: text" 'http://127.0.0.1:3000/https://google.com'
   ```

4. 📸 Screen-Size Screenshot (returns the URL of the webpage's screenshot):
   ```bash
   curl -H "X-Respond-With: screenshot" 'http://127.0.0.1:3000/https://google.com'
   ```

5.  📸 Full-Page Screenshot (returns the URL of the webpage's screenshot):
   ```bash
   curl -H "X-Respond-With: pageshot" 'http://127.0.0.1:3000/https://google.com'
   ```

## 🔧 Request Headers (Optional)
You can pass these headers to make the browser request look more like a real user session:

```bash
curl \
  -H "X-User-Agent: <your real UA>" \
  -H "X-Referer: https://help.aliyun.com/" \
  -H "X-Accept-Language: zh-CN,zh;q=0.9,en;q=0.8" \
  -H "X-Set-Cookie: cookie1=...; Path=/; Domain=help.aliyun.com, cookie2=...; Path=/; Domain=help.aliyun.com" \
  -H "X-Extra-Headers: {\"sec-ch-ua\":\"\\\"Chromium\\\";v=\\\"143\\\"\",\"x-custom\":\"value\"}" \
  'http://127.0.0.1:3000/https://help.aliyun.com/zh/xxx'
```

### Anti-crawl header bundle helper
If you have a real browser curl (like `example-curl.sh`), convert it to env vars:
```bash
node scripts/example-curl-to-env.mjs example-curl.sh
```
Then map the output to `X-User-Agent`, `X-Referer`, `X-Accept-Language`, and `X-Extra-Headers`.

## ⚙️ Default Browser Settings (Env)
These env vars set defaults for all requests:

```
READER_USER_AGENT="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"
READER_ACCEPT_LANGUAGE="zh-CN,zh;q=0.9,en;q=0.8"
READER_TIMEZONE="Asia/Shanghai"
READER_VIEWPORT="1366x768"
PORT="3000"
```

## 🙏 Acknowledgements
This project is based on the excellent work done by multiple contributors:
1. The original [Jina AI Reader project](https://github.com/jina-ai/reader), which provided the foundation for this tool.
2. [Harsh Gupta's adaptation](https://github.com/hargup/reader), which served as the immediate basis for this Docker deployment version.

## 📜 License
This project is licensed under Apache-2.0 same as the original Jina AI Reader project.
