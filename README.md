# Play Store MCP Server

MCP server kết nối trợ lý AI với Google Play Developer API để đọc thông tin ứng dụng, quản lý bản phát hành, store listing, review, tester, sản phẩm và subscription.

Hướng dẫn này đi từ tạo Google service account đến chạy local, deploy HTTP và thêm MCP vào Claude, Codex, Cursor hoặc Gemini CLI.

> Các lệnh shell dùng Bash/Zsh trên macOS/Linux. Thay domain, username, đường dẫn và package name bằng giá trị của bạn. Các ví dụ bắt đầu với chế độ chỉ đọc; xem mục 7 khi cần bật thao tác ghi.

## Mục lục

- [1. Chọn cách chạy](#chon-cach-chay)
- [2. Cấu hình Google auth](#google-auth)
- [3. Cài đặt và chạy local](#cai-dat)
- [4. Deploy server HTTP](#deploy-http)
- [5. Thêm MCP sau khi deploy](#them-mcp)
- [6. Kiểm tra kết nối và Google auth](#kiem-tra)
- [7. Quyền ghi và code mode](#quyen-ghi)
- [8. Biến môi trường](#bien-moi-truong)
- [9. Xử lý lỗi](#xu-ly-loi)
- [10. Công cụ và phát triển](#phat-trien)

<a id="chon-cach-chay"></a>

## 1. Chọn cách chạy

| Cách chạy | Phù hợp khi | Client kết nối bằng |
| --- | --- | --- |
| Local qua `stdio` | Dùng trên máy cá nhân, thao tác với APK/AAB trên máy | `command` + `args`; client tự khởi động server |
| HTTP với credential trên server | Một cá nhân/nhóm dùng chung service account | URL `/mcp` và token của reverse proxy |
| HTTP với credential theo request | Mỗi người dùng service account riêng | URL `/mcp`, header Google và token proxy nếu có |

Với HTTP, có hai lớp xác thực độc lập:

- **Google auth:** service account quyết định server được làm gì trong Play Console.
- **Quyền truy cập MCP:** reverse proxy/VPN quyết định ai được gọi server.

**`/mcp` chưa có xác thực truy cập tích hợp.** `PLAY_STORE_MCP_ADMIN_TOKEN` chỉ bảo vệ `POST /credentials`, không bảo vệ `/mcp`. Nếu đặt credential Google trên server rồi công khai `/mcp`, người truy cập được endpoint có thể dùng chung quyền Google đó.

<a id="google-auth"></a>

## 2. Cấu hình Google auth

### 2.1. Chuẩn bị

Bạn cần quyền tạo service account/key trong Google Cloud và quyền mời người dùng, cấp quyền ứng dụng trong Google Play Console. Chuẩn bị package name thật, ví dụ `com.yourcompany.yourapp`, đã tồn tại trong Play Console.

Repo dùng **service account JSON key**. Không cần tạo OAuth Client ID, consent screen, redirect URI hay đăng nhập Google qua trình duyệt cho cách cấu hình này.

### 2.2. Tạo project và bật API

1. Mở [Google Cloud Console](https://console.cloud.google.com/), tạo hoặc chọn project.
2. Trong **APIs & Services → Library**, tìm **Google Play Android Developer API** (`androidpublisher.googleapis.com`).
3. Chọn **Enable** trong đúng project sẽ chứa service account.

Theo [hướng dẫn Google Play Developer API](https://developers.google.com/android-publisher/getting_started), không còn bắt buộc liên kết developer account với Cloud project qua trang “API access” cũ.

### 2.3. Tạo service account và tải JSON key

1. Vào **IAM & Admin → Service Accounts → Create service account**.
2. Đặt tên, ví dụ `play-store-mcp`, rồi hoàn tất. Quyền truy cập app sẽ cấp trong Play Console; không cần cấp Cloud Project Owner cho việc này.
3. Mở service account → **Keys → Add key → Create new key → JSON → Create**.
4. Lưu file tải xuống ở thư mục riêng ngoài repo, ví dụ `/Users/you/.config/play-store-mcp/service-account.json`.

File cần có `type: "service_account"`, `client_email`, `private_key` và `token_uri`. Giữ nguyên nội dung file Google cấp. Nếu tổ chức chặn tạo key, nhờ quản trị viên kiểm tra quyền/chính sách tạo service account key. Xem [tài liệu tạo key của Google](https://cloud.google.com/iam/docs/keys-create-delete).

Trên macOS/Linux:

```bash
chmod 600 /Users/you/.config/play-store-mcp/service-account.json
```

Không commit JSON key hoặc Base64 của key. Nếu key bị lộ, xóa key đó trong tab **Keys** và tạo key thay thế.

### 2.4. Cấp quyền trong Google Play Console

1. Mở [Google Play Console](https://play.google.com/console/) → **Users and permissions → Invite new users**.
2. Nhập **`client_email` trong JSON key**, dạng `play-store-mcp@YOUR_PROJECT.iam.gserviceaccount.com`.
3. Trong **App permissions**, chọn app cần truy cập và cấp quyền theo nhu cầu.
4. Chọn **Invite user**, kiểm tra app và quyền đã lưu.

| Nhu cầu | Quyền Play Console cần xem xét |
| --- | --- |
| Đọc thông tin ứng dụng | View app information (read-only) |
| Đẩy bản lên track thử nghiệm | Release apps to testing tracks |
| Phát hành production | Release to production, exclude devices, and use Play App Signing |
| Quản lý tester | Manage testing tracks and edit tester lists |
| Sửa listing/sản phẩm | Manage store presence |
| Trả lời review | Reply to reviews |
| Billing, đơn hàng, subscription | View financial data, orders, and cancellation survey responses; Manage orders and subscriptions |

Tên quyền có thể khác theo ngôn ngữ giao diện. Cấp theo tác vụ và phạm vi app cần dùng; tham khảo [mô tả quyền Play Console](https://support.google.com/googleplay/android-developer/answer/9844686). Google liệt kê hai quyền Billing ở dòng cuối trong [hướng dẫn thiết lập](https://developers.google.com/android-publisher/getting_started).

**Có JSON key chưa đủ:** service account vẫn cần được thêm vào Play Console. Quyền IAM của Cloud project không thay thế quyền app trong Play Console.

### 2.5. Chọn cách cung cấp credential

**Cách A — đường dẫn file trên máy chạy server:**

```bash
export GOOGLE_APPLICATION_CREDENTIALS="/Users/you/.config/play-store-mcp/service-account.json"
```

Dùng đường dẫn tuyệt đối. Trong Docker, đây phải là đường dẫn **bên trong container** sau khi mount.

**Cách B — JSON trong biến môi trường, phù hợp secret của hosting:**

```bash
export GOOGLE_PLAY_STORE_CREDENTIALS="$(cat /absolute/path/service-account.json)"
```

Trong dashboard hosting, dán toàn bộ JSON vào secret `GOOGLE_PLAY_STORE_CREDENTIALS`, không thêm dấu nháy shell bên ngoài. Nếu đặt cả hai biến, server ưu tiên `GOOGLE_PLAY_STORE_CREDENTIALS` trước đường dẫn file. Chỉ đặt cách bạn muốn dùng để tránh chọn nhầm tài khoản.

**Cách C — header riêng trên mỗi HTTP request:**

Tạo Base64 một dòng bằng Python trên macOS/Linux:

```bash
export PLAY_STORE_GOOGLE_CREDENTIALS_B64="$(python3 -c 'import base64, pathlib, sys; print(base64.b64encode(pathlib.Path(sys.argv[1]).read_bytes()).decode())' /absolute/path/service-account.json)"
```

Client gửi header `X-Google-Credentials-Base64` chứa giá trị này. Base64 vẫn chứa toàn bộ private key, không phải mã hóa bảo mật. Chỉ gửi tới server tin cậy qua HTTPS hoặc kết nối local/private được bảo vệ.

Header được ưu tiên trước credential chung trên server. Với nhiều người dùng, **không đặt** `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_PLAY_STORE_CREDENTIALS` và không dùng `POST /credentials` để tạo credential chung. Khi đó, request thiếu header không thể dùng danh tính Google của người khác để gọi API.

<a id="cai-dat"></a>

## 3. Cài đặt và chạy local

### 3.1. Cài từ mã nguồn repo này

Cần Git và Python 3.11+. Cài `uv` theo [tài liệu Astral](https://docs.astral.sh/uv/getting-started/installation/), rồi chạy:

```bash
git clone https://github.com/quyendang/play-store-mcp.git
cd play-store-mcp
uv sync --frozen
uv run --frozen play-store-mcp --help
```

Chạy qua `stdio` sau khi cấu hình credential ở mục 2:

```bash
uv run --frozen play-store-mcp --read-only
```

Server chờ dữ liệu MCP trên stdin, không có giao diện web khi chạy `stdio`. Thông thường client sẽ tự chạy lệnh này; xem mẫu Claude Desktop/Codex ở mục 5.

### 3.2. Package phát hành qua uvx hoặc pip

Các lệnh sau dùng bản trên PyPI; bản đó có thể khác mã nguồn fork này:

```bash
uvx play-store-mcp --help
```

Hoặc cài vào virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install play-store-mcp
play-store-mcp --help
```

Để chạy fork bằng pip, ở thư mục repo dùng `python -m pip install -e .` thay cho cài package từ PyPI.

### 3.3. Thử HTTP trên máy cá nhân

```bash
export PLAY_STORE_MCP_DOWNLOAD_DIR="/tmp/play-store-downloads"
mkdir -p "$PLAY_STORE_MCP_DOWNLOAD_DIR"
uv run --frozen play-store-mcp \
  --transport streamable-http --host 127.0.0.1 --port 8000 --read-only
```

Trong terminal khác:

```bash
curl --fail http://127.0.0.1:8000/health
```

Kết quả mong đợi:

```json
{"status":"healthy","service":"play-store-mcp"}
```

URL client là **`http://127.0.0.1:8000/mcp`**. `/health` chỉ xác nhận server đang chạy, chưa xác nhận quyền Google.

<a id="deploy-http"></a>

## 4. Deploy server HTTP

### 4.1. Build Docker image từ repo

Trên máy deploy có Docker, tại thư mục repo:

```bash
docker build -t play-store-mcp:local .
```

Khởi động với credential chung, chỉ mở cổng trên loopback của host:

```bash
export GOOGLE_PLAY_STORE_CREDENTIALS="$(cat /absolute/path/service-account.json)"

docker run -d --name play-store-mcp --restart unless-stopped \
  -p 127.0.0.1:8000:8000 \
  -e GOOGLE_PLAY_STORE_CREDENTIALS \
  -e PLAY_STORE_MCP_READ_ONLY=1 \
  -e PLAY_STORE_MCP_DOWNLOAD_DIR=/tmp/play-store-downloads \
  play-store-mcp:local \
  --transport streamable-http --host 0.0.0.0 --port 8000
```

Docker lấy JSON từ biến môi trường hiện tại; với hosting, dùng secret manager để cung cấp biến này. Người có quyền quản trị Docker có thể đọc môi trường container.

Nếu muốn dùng file, thay `-e GOOGLE_PLAY_STORE_CREDENTIALS` bằng hai tùy chọn sau:

```bash
--mount type=bind,src=/absolute/path/service-account.json,dst=/creds/key.json,readonly \
-e GOOGLE_APPLICATION_CREDENTIALS=/creds/key.json
```

Đây là các tùy chọn ghép vào lệnh `docker run`, không phải lệnh chạy riêng. Container chạy user không phải root: file key phải đọc được bởi user trong container. `readonly` chỉ ngăn ghi, không cấp quyền đọc; dùng secret mount/quyền UID phù hợp nếu file mode `600` trên host.

Kiểm tra:

```bash
docker logs --tail 50 play-store-mcp
curl --fail http://127.0.0.1:8000/health
```

Image `ghcr.io/lusky3/play-store-mcp:latest` là image upstream. Build từ repo để dùng đúng mã nguồn fork.

### 4.2. Truy cập riêng qua SSH tunnel

Có thể giữ cổng 8000 ở loopback. Trên máy có Claude/Codex:

```bash
ssh -N -L 8000:127.0.0.1:8000 your-user@your-server
```

Giữ terminal này mở. Client dùng `http://127.0.0.1:8000/mcp`, không cần domain, certificate hay token proxy. Nếu cổng local 8000 đã bận, đổi số đầu tiên trong `-L` và URL client tương ứng.

### 4.3. Công khai qua HTTPS và reverse proxy

Chuẩn bị domain trỏ đến máy deploy và reverse proxy có TLS. Ví dụ dành cho Nginx chạy trên host cùng container, với certificate đã cấp cho `mcp.example.com`.

Tạo token riêng cho người được gọi MCP:

```bash
openssl rand -hex 32
```

Lưu token vào nơi quản lý secret. Thay `REPLACE_WITH_RANDOM_TOKEN` dưới đây bằng token đó. Đặt `map` và `server` trong khối `http` của Nginx:

```nginx
map $http_authorization $play_store_mcp_allowed {
    default 0;
    "Bearer REPLACE_WITH_RANDOM_TOKEN" 1;
}

server {
    listen 443 ssl;
    server_name mcp.example.com;

    ssl_certificate /etc/letsencrypt/live/mcp.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mcp.example.com/privkey.pem;

    location = /health {
        proxy_set_header Host localhost;
        proxy_pass http://127.0.0.1:8000;
    }

    location /mcp {
        if ($play_store_mcp_allowed = 0) { return 401; }
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host localhost;
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_read_timeout 300s;
    }

    location / { return 404; }
}
```

Cấu hình chặn `/credentials`, kiểm tra Bearer token ở proxy và giữ HTTP streaming. Các header Google và MCP được chuyển tiếp mặc định. Tham khảo Nginx [map](https://nginx.org/en/docs/http/ngx_http_map_module.html) và [proxy](https://nginx.org/en/docs/http/ngx_http_proxy_module.html).

Sau khi lưu vào vị trí Nginx đang load:

```bash
sudo nginx -t
sudo nginx -s reload
curl --fail https://mcp.example.com/health
curl -i https://mcp.example.com/mcp
```

Request cuối thiếu token phải trả **401**. Thử kết nối đầy đủ bằng MCP client ở mục 5. Giữ cổng 8000 ở loopback để không thể bỏ qua proxy.

Mẫu Nginx gửi `Host: localhost`, nên giữ được bảo vệ DNS rebinding mặc định. Nếu hosting/proxy giữ public hostname và server báo `Invalid host header`, chỉ đặt `PLAY_STORE_MCP_DISABLE_DNS_REBINDING=1` khi đã có proxy kiểm soát domain và quyền truy cập.

### 4.4. Hosting khác và nhiều người dùng

Dùng Dockerfile của repo hoặc chạy Python với:

```bash
play-store-mcp --transport streamable-http --host 0.0.0.0 --port 8000
```

- Đặt `MCP_PORT` theo cổng hosting yêu cầu nếu dùng biến môi trường; khi đó bỏ `--port 8000` để tránh ghi đè. Repo không tự đọc biến `PORT`.
- Health check path: `/health`. URL MCP: `/mcp`.
- Với credential chung, lưu JSON vào secret `GOOGLE_PLAY_STORE_CREDENTIALS` và bảo vệ `/mcp` bằng gateway/proxy có auth.
- Với credential theo request, **bỏ `-e GOOGLE_PLAY_STORE_CREDENTIALS` trong lệnh Docker ở mục 4.1**, không mount key và không đặt `GOOGLE_APPLICATION_CREDENTIALS`. Dùng HTTPS và chuyển tiếp header `X-Google-Credentials-Base64`.
- Đặt `PLAY_STORE_MCP_DOWNLOAD_DIR` vào thư mục ghi được, ví dụ `/tmp/play-store-downloads`. File tạm có thể mất sau restart.

`POST /credentials` thay đổi credential **chung của process**, không phải đăng nhập riêng cho từng người. Nếu dùng để xoay key, xem [quản lý credential từ xa](docs/remote-credentials.md); khi endpoint đi qua proxy phải đặt `PLAY_STORE_MCP_ADMIN_TOKEN`.

<a id="them-mcp"></a>

## 5. Thêm MCP sau khi deploy

Các mẫu remote dùng:

- URL: `https://mcp.example.com/mcp`.
- `PLAY_STORE_MCP_ACCESS_TOKEN`: token do **reverse proxy** kiểm tra ở mục 4.3; repo không tự đọc biến này.
- `PLAY_STORE_GOOGLE_CREDENTIALS_B64`: Base64 ở mục 2.5, chỉ cần khi server không có credential Google chung.

Đặt biến trong shell dùng để khởi động client:

```bash
export PLAY_STORE_MCP_ACCESS_TOKEN='REPLACE_WITH_YOUR_PROXY_TOKEN'
# Chỉ khi dùng credential theo request:
# export PLAY_STORE_GOOGLE_CREDENTIALS_B64='YOUR_BASE64_CREDENTIALS'
```

Nếu dùng SSH tunnel, thay URL bằng `http://127.0.0.1:8000/mcp` và bỏ header Bearer. Nếu server có credential Google chung, bỏ header `X-Google-Credentials-Base64`. Không dùng đồng thời mẫu local/remote với cùng tên server trong cùng client.

### 5.1. Claude Code

Thêm HTTP server ở user scope:

```bash
claude mcp add --scope user --transport http play-store \
  https://mcp.example.com/mcp \
  --header "Authorization: Bearer ${PLAY_STORE_MCP_ACCESS_TOKEN}"
```

Với credential theo request, dùng lệnh này thay cho lệnh trên:

```bash
claude mcp add --scope user --transport http play-store \
  https://mcp.example.com/mcp \
  --header "Authorization: Bearer ${PLAY_STORE_MCP_ACCESS_TOKEN}" \
  --header "X-Google-Credentials-Base64: ${PLAY_STORE_GOOGLE_CREDENTIALS_B64}"
```

Lệnh lưu giá trị header đã được shell mở rộng vào cấu hình cá nhân. Tránh lưu credential trong config project được commit. Kiểm tra bằng `claude mcp list`, rồi mở Claude Code và dùng `/mcp`. Xem [Claude Code MCP](https://code.claude.com/docs/en/mcp).

### 5.2. Codex CLI / ứng dụng Codex

Thêm remote server:

```bash
codex mcp add play-store --url https://mcp.example.com/mcp \
  --bearer-token-env-var PLAY_STORE_MCP_ACCESS_TOKEN
codex mcp list
```

Hoặc thêm trực tiếp vào `~/.codex/config.toml`:

```toml
[mcp_servers.play-store]
url = "https://mcp.example.com/mcp"
bearer_token_env_var = "PLAY_STORE_MCP_ACCESS_TOKEN"

# Chỉ thêm khi dùng credential theo request:
[mcp_servers.play-store.env_http_headers]
X-Google-Credentials-Base64 = "PLAY_STORE_GOOGLE_CREDENTIALS_B64"
```

Codex lấy token/header từ môi trường process client. Mở chat mới sau khi đổi cấu hình; trong CLI dùng `/mcp` kiểm tra. Với app desktop, bảo đảm app có các biến này; `export` trong terminal không tự cập nhật app đã mở. Có thể dùng `http_headers` với giá trị tĩnh trong config cá nhân khi cần, nhưng file sẽ chứa secret. Xem [tài liệu MCP của OpenAI](https://developers.openai.com/codex/mcp/).

Nếu app desktop không nhận biến môi trường, dùng cấu hình tĩnh sau **thay cho** block `play-store` phía trên, rồi mở chat mới:

```toml
[mcp_servers.play-store]
url = "https://mcp.example.com/mcp"
http_headers = { Authorization = "Bearer REPLACE_WITH_YOUR_PROXY_TOKEN" }
```

Với credential theo request, thêm `"X-Google-Credentials-Base64" = "YOUR_BASE64_CREDENTIALS"` vào cùng bảng `http_headers`. Không commit config chứa giá trị thật.

Repo chưa triển khai OAuth cho MCP, nên `codex mcp login play-store` không thay thế cấu hình service account/header.

**Chạy mã nguồn local qua Codex:**

```bash
codex mcp add play-store \
  --env GOOGLE_APPLICATION_CREDENTIALS=/absolute/path/service-account.json \
  -- uv --directory /absolute/path/play-store-mcp run --frozen play-store-mcp --read-only
```

### 5.3. Claude Desktop

**Local qua stdio:** mở file cấu hình theo [hướng dẫn MCP cho Claude Desktop](https://modelcontextprotocol.io/docs/develop/connect-local-servers):

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

Ghép entry `play-store` vào `mcpServers` hiện có:

```json
{
  "mcpServers": {
    "play-store": {
      "command": "uv",
      "args": ["--directory", "/absolute/path/play-store-mcp", "run", "--frozen", "play-store-mcp", "--read-only"],
      "env": {
        "GOOGLE_APPLICATION_CREDENTIALS": "/absolute/path/service-account.json"
      }
    }
  }
}
```

Dùng đường dẫn thực trên Windows và escape `\\` trong JSON. Nếu app không tìm thấy `uv`, thay `command` bằng đường dẫn tuyệt đối của executable. Thoát hoàn toàn rồi mở lại Claude Desktop.

**Server đã deploy:** để gửi custom header, có thể dùng cầu nối stdio [mcp-remote](https://github.com/punkpeye/mcp-remote). Cài Node.js LTS kèm `npx`, dùng entry này thay cho entry local:

```json
{
  "mcpServers": {
    "play-store": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote", "https://mcp.example.com/mcp",
        "--header", "Authorization:${PLAY_STORE_MCP_AUTH_HEADER}",
        "--header", "X-Google-Credentials-Base64:${PLAY_STORE_GOOGLE_CREDENTIALS_B64}"
      ],
      "env": {
        "PLAY_STORE_MCP_AUTH_HEADER": "Bearer REPLACE_WITH_YOUR_PROXY_TOKEN",
        "PLAY_STORE_GOOGLE_CREDENTIALS_B64": "YOUR_BASE64_CREDENTIALS"
      }
    }
  }
}
```

`mcp-remote` mở rộng biến từ `env`. Với credential Google chung, xóa cặp `--header` dành cho Google và biến tương ứng. File cấu hình chứa secret, cần giữ riêng trên máy.

### 5.4. Cursor

Thêm vào config cá nhân `~/.cursor/mcp.json`. Cursor dùng `url` cho remote và `${env:NAME}` cho biến môi trường theo [tài liệu Cursor MCP](https://cursor.com/docs/context/mcp):

```json
{
  "mcpServers": {
    "play-store": {
      "url": "https://mcp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:PLAY_STORE_MCP_ACCESS_TOKEN}",
        "X-Google-Credentials-Base64": "${env:PLAY_STORE_GOOGLE_CREDENTIALS_B64}"
      }
    }
  }
}
```

Bỏ dòng header Google khi dùng credential chung. Khởi động lại Cursor với biến môi trường đã có, kiểm tra server trong MCP settings và bật tool trong Agent.

### 5.5. Gemini CLI

Ghép entry vào `mcpServers` của `~/.gemini/settings.json`. Streamable HTTP dùng **`httpUrl`** theo [tài liệu Gemini CLI](https://geminicli.com/docs/tools/mcp-server/):

```json
{
  "mcpServers": {
    "play-store": {
      "httpUrl": "https://mcp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer REPLACE_WITH_YOUR_PROXY_TOKEN",
        "X-Google-Credentials-Base64": "YOUR_BASE64_CREDENTIALS"
      }
    }
  }
}
```

Thay placeholder trong config cá nhân, bỏ header Google nếu dùng credential chung. Khởi động lại Gemini CLI và dùng `/mcp` kiểm tra trạng thái.

<a id="kiem-tra"></a>

## 6. Kiểm tra kết nối và Google auth

Sau khi client báo connected, thử theo thứ tự:

1. **“Dùng Play Store MCP kiểm tra package name `com.yourcompany.yourapp` có hợp lệ không.”** `validate_package_name` chỉ kiểm tra định dạng, chưa gọi Google.
2. **“Dùng Play Store MCP lấy các bản phát hành của `com.yourcompany.yourapp` bằng `get_releases`. Chỉ đọc dữ liệu.”** Thay bằng package thật đã cấp quyền. Google trả kết quả thành công mới xác nhận credential và quyền cho API này.
3. Thử listing/review nếu có nhu cầu. Thành công với một API chưa có nghĩa đã có quyền cho mọi tác vụ.

Mặc định có thể chỉ thấy `search`, `get_schema`, `execute` vì code mode. Trợ lý cần tìm tool và lấy schema trước khi gọi; bạn không cần tự nhập code sandbox để kiểm tra setup.

Với server remote, file `.aab`/`.apk` phải tồn tại **trên máy chạy server** hoặc volume đã mount. File trên laptop không tự được gửi lên chỉ vì bạn đưa đường dẫn cho trợ lý.

<a id="quyen-ghi"></a>

## 7. Quyền ghi và code mode

### Bật thao tác ghi

Để sửa listing, trả lời review, upload hoặc phát hành:

1. Cấp đúng quyền Play Console ở mục 2.4.
2. Bỏ `--read-only` và unset `PLAY_STORE_MCP_READ_ONLY`, hoặc đặt biến thành `0`.
3. Restart server. Với Docker, tạo lại container với cấu hình mới; với stdio, mở lại client/server.

Tắt read-only không tự cấp thêm quyền Google; cả hai lớp đều phải cho phép tác vụ.

### Code mode

Code mode bật mặc định, gom công cụ thành `search`, `get_schema`, `execute`. Một `execute` có thể gọi tối đa 50 tool, gồm thao tác ghi, sau một lần client phê duyệt. Read-only vẫn được kiểm tra bên trong sandbox.

Để hiện từng tool riêng, đặt biến **trên process server** rồi restart:

```bash
export CODE_MODE=0
```

Với remote, đặt biến ở hosting/container; đặt ở máy client không đổi tool list của server remote. `CODE_MODE` không có CLI flag. Dùng danh sách tool riêng khi muốn client phê duyệt từng thao tác ghi rõ ràng hơn.

<a id="bien-moi-truong"></a>

## 8. Biến môi trường

Repo đọc biến môi trường process, không tự load `.env`. Cần export, cấu hình secret hosting hoặc dùng cơ chế env của client/container.

| Biến | Ý nghĩa / mặc định |
| --- | --- |
| `GOOGLE_APPLICATION_CREDENTIALS` | Đường dẫn service account JSON trên máy chạy server |
| `GOOGLE_PLAY_STORE_CREDENTIALS` | JSON credential hoặc đường dẫn file; ưu tiên trước biến trên |
| `MCP_TRANSPORT` | `stdio` (mặc định), `streamable-http`, `sse` |
| `MCP_HOST` | Python CLI: `127.0.0.1`; Dockerfile: `0.0.0.0` |
| `MCP_PORT` | `8000`; CLI `--port` ghi đè biến này |
| `PLAY_STORE_MCP_READ_ONLY` | Bật bằng `1`, `true`, `yes`, `on`; mặc định tắt |
| `CODE_MODE` | Mặc định bật; `0`, `false`, `no`, `off` để hiện từng tool |
| `PLAY_STORE_MCP_DOWNLOAD_DIR` | Giới hạn nơi ghi file tải xuống; mặc định thư mục làm việc |
| `PLAY_STORE_MCP_LOG_LEVEL` | `INFO`; có thể dùng `DEBUG`, `WARNING`, `ERROR` |
| `PLAY_STORE_MCP_DISABLE_DNS_REBINDING` | Mọi chuỗi không rỗng đều tắt kiểm tra Host, kể cả `0`; unset để bật bảo vệ |
| `PLAY_STORE_MCP_ADMIN_TOKEN` | Bearer token chỉ cho `POST /credentials`; không bảo vệ `/mcp` |

`PLAY_STORE_MCP_ACCESS_TOKEN`, `PLAY_STORE_MCP_AUTH_HEADER`, `PLAY_STORE_GOOGLE_CREDENTIALS_B64` trong ví dụ là quy ước phía client/proxy, không phải biến auth tích hợp của server.

<a id="xu-ly-loi"></a>

## 9. Xử lý lỗi

| Triệu chứng | Cách kiểm tra |
| --- | --- |
| `uv`, `uvx`, `npx` not found / `ENOENT` | Kiểm tra executable; với app desktop dùng đường dẫn tuyệt đối nếu PATH khác terminal |
| `No valid credentials found` / `No credentials provided` | Đặt credential đúng process server hoặc gửi header; kiểm tra mount và quyền đọc file |
| `Invalid base64 or JSON` | Encode toàn bộ file gốc thành một dòng, không thêm dấu nháy/xuống dòng vào header |
| API chưa bật / `SERVICE_DISABLED` | Bật `androidpublisher.googleapis.com` trong project của service account |
| Google báo `The caller does not have permission` / 403 | Kiểm tra `client_email`, app và quyền tác vụ trong Play Console; thử lại sau khi thay quyền được áp dụng |
| Package không tồn tại / Google 404 | Dùng application ID thật, kiểm tra app thuộc đúng developer account và đã cấp quyền |
| `Invalid host header` / HTTP 400 | Kiểm tra Host proxy chuyển tiếp; dùng `Host localhost` như mẫu hoặc xem mục 4.3 |
| 401 khi kết nối MCP | Kiểm tra token proxy và header client; admin token `/credentials` không phải token proxy |
| Connected nhưng chỉ có ba tool | Code mode đang bật; dùng `search`/`get_schema` hoặc đặt `CODE_MODE=0` trên server |
| `/health` OK nhưng tool lỗi | Health check không kiểm tra Google; gọi `get_releases` với package đã cấp quyền |
| Write operation disabled | Bỏ flag read-only, kiểm tra env server, restart và kiểm tra quyền Play Console |
| Không tìm thấy APK/AAB | File phải nằm trên máy server; Docker cần volume và đường dẫn bên trong container |
| Tải file bị từ chối | Đích phải trong `PLAY_STORE_MCP_DOWNLOAD_DIR` và thư mục cần ghi được |
| Mất session sau restart/cân bằng tải | Kết nối lại; bắt đầu với một instance, định tuyến phù hợp session khi chạy nhiều replica |

Log Docker:

```bash
docker logs --tail 100 play-store-mcp
```

Log Claude Desktop trên macOS: `~/Library/Logs/Claude/`. Đọc thêm [troubleshooting](docs/troubleshooting.md) và [cấu hình nâng cao](docs/configuration.md).

<a id="phat-trien"></a>

## 10. Công cụ và phát triển

| Nhóm | Ví dụ công cụ |
| --- | --- |
| Phát hành | `deploy_app`, `deploy_app_multilang`, `get_releases`, `promote_release`, `halt_release`, `update_rollout`, `batch_deploy` |
| Listing/hình ảnh | `get_listing`, `list_all_listings`, `update_listing`, `list_images`, `upload_image` |
| Review/tester | `get_reviews`, `reply_to_review`, `get_testers`, `update_testers` |
| Sản phẩm/subscription | `list_in_app_products`, `list_subscriptions`, `get_subscription_status` |
| Đơn hàng | `get_order`, `refund_order`, `list_voided_purchases` |
| APK/AAB/tài nguyên | `list_apks`, `list_bundles`, `list_generated_apks`, `get_expansion_file` |
| Kiểm tra đầu vào | `validate_package_name`, `validate_track`, `validate_listing_text` |

Tham số chi tiết: [Tools reference](docs/tools-reference.md). Khả năng gọi API phụ thuộc quyền service account, trạng thái app và chế độ read-only.

Cài dependency phát triển:

```bash
uv sync --frozen --extra dev
uv run pytest -v --cov=src/play_store_mcp
uv run ruff check src/ tests/
uv run ruff format --check src/ tests/
uv run mypy src/
```

Dự án dùng [FastMCP](https://github.com/PrefectHQ/fastmcp) và [Google Play Developer API](https://developers.google.com/android-publisher). Fork từ [lusky3/play-store-mcp](https://github.com/lusky3/play-store-mcp), lấy cảm hứng từ [antoniolg/play-store-mcp](https://github.com/antoniolg/play-store-mcp).

Giấy phép MIT: xem [LICENSE](LICENSE).
