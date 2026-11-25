<h1>Plexo</h1>

Hệ sinh thái ứng dụng cá nhân: một ứng dụng chính (mạng xã hội — đang lên kế hoạch) và các ứng dụng mở rộng (game…), dùng chung tài khoản và đăng nhập.

Repo này chứa tài liệu chung (kiến trúc, hướng dẫn chạy) và cấu hình hạ tầng / deploy (`infrastructure/`, `service/`). Mọi repo nằm trong GitHub organization **`thanhldt-plexo`**.

## Repository

| Repository | Loại | Vai trò | Route (prod) |
|---|---|---|---|
| `Plexo` | Tài liệu / DevOps | Kiến trúc, hướng dẫn, docker-compose hạ tầng và deploy | — |
| `Admin_Service` | BE (Go) | User dùng chung, xác thực tập trung, resource dùng chung | `api.plexo.thanhldt.site/admin` |
| `Caro_Game_Service` | BE (Go) | Game Caro: trận đấu, bot, matchmaking, lịch sử, thống kê | `api.plexo.thanhldt.site/caro-game` |
| `Notification_Service` | BE (Go) | Thông báo (chưa có nghiệp vụ) | `api.plexo.thanhldt.site/notification` |
| `Plexo_UI` | FE (Next.js) | Ứng dụng chính: đăng nhập, danh sách ứng dụng mở rộng | `plexo.thanhldt.site/` |
| `Caro_Game_UI` | FE (Next.js) | Giao diện game Caro | `plexo.thanhldt.site/games/caro` |

## Môi trường

Domain `thanhldt.site` dùng chung cho mọi dự án cá nhân: mỗi app / môi trường một cặp host `<app>[-<env>].thanhldt.site` (FE) và `api.<app>[-<env>].thanhldt.site` (BE).

| | prod | dev |
|---|---|---|
| FE / API | `plexo.thanhldt.site` / `api.plexo.thanhldt.site` | `plexo-dev.thanhldt.site` / `api.plexo-dev.thanhldt.site` |
| Nguồn | Tag version `v*` tạo từ nhánh `main` | Nhánh `develop` |
| Image | `:v1.0.2`, `:latest`, `:<sha>` | `:develop-latest`, `:develop-<sha>` |
| Deploy | **Thủ công** trên server (pull khi phát hành) | **Tự động** sau mỗi push `develop` (self-hosted runner `homelab-server-01`) |
| Thư mục trên server | `/srv/app-prod/plexo` | `/srv/app-dev/plexo` |
| `STACK_NAME` | `plexo-prod` | `plexo-dev` |

Push `main` không build image (chỉ thông báo commit). Pull request vào `main` / `develop` chạy CI (kiểm tra code). Không tạo tag version từ `develop`.

## Kiến trúc

```
 Người dùng ──HTTPS──▶ Cloudflare ──Tunnel──▶ cloudflared ──▶ Traefik ──▶ service / ứng dụng
                                          (mỗi môi trường một tunnel + một Traefik riêng, không mở port)

 <web-domain>  /games/caro/* ──▶ Caro_Game_UI :3000     <api-domain>  /admin/*        ──▶ Admin_Service        :8000
               /*            ──▶ Plexo_UI     :3000                   /caro-game/*    ──▶ Caro_Game_Service    :8000
                                                                      /notification/* ──▶ Notification_Service :8000

        Cookie plexo_session (httpOnly, domain = host FE của môi trường) dùng chung cho mọi ứng dụng FE và gửi kèm tới API

 Caro_Game_Service ──(network nội bộ)──▶ Admin_Service : verify token, lấy thông tin User
 Admin_Service     ──▶ Postgres (plexo_admin)     + Redis (prefix admin:)
 Caro_Game_Service ──▶ Postgres (plexo_caro_game) + Redis (prefix caro:)
```

**Nguyên tắc**

- Mỗi môi trường là một **stack** độc lập (`STACK_NAME=<app>-<env>`): Postgres, Redis, Traefik, Cloudflare Tunnel và các service riêng; mọi tên container / network / router đều có tiền tố stack nên không đụng môi trường hay app khác trên cùng server.
- BE chạy port 8000 trong container, route theo tên service và phiên bản API (`/<service>/v1/...`); FE chạy port 3000, cùng origin, phân biệt bằng path (ứng dụng chính ở root, game ở `/games/<game-name>`).
- **Mỗi service một database + user riêng**, **một ACL user + prefix key riêng** trên Redis. Không service nào đọc dữ liệu của service khác — trao đổi qua API.
- `Admin_Service` là nguồn sự thật duy nhất về User. Service khác chỉ lưu tham chiếu `user_uuid`.
- BE (Go 1.26): chia theo tầng (`model / dto / service / repository / api/v1`), Huma v2, bun, `log/slog`, cấu hình `config/config.json`, lỗi RFC 9457 + `code`, JSON snake_case. Code hạ tầng dùng chung nằm trong thư mục `common/` giống hệt ở mọi service.
- FE: Next.js 15, React 19, Tailwind CSS 3, react-query, next-intl (vi / en), Atomic Design. Cấu hình `.env`; biến `NEXT_PUBLIC_*` gắn vào image lúc build (CI lấy từ GitHub Environment của môi trường).

Chi tiết: [`docs/architecture.md`](docs/architecture.md). API và giao thức của từng service: README của repo tương ứng.

## Chạy local

Clone các repository cạnh nhau trong cùng một thư mục workspace. Yêu cầu: Docker, Go 1.26+, Node 22 + pnpm.

**1. Hạ tầng (Postgres + Redis)** — trong `Plexo/infrastructure`:

```bash
cp .env.example .env    # STACK_NAME=plexo-local, điền POSTGRES_PASSWORD, bỏ trống CLOUDFLARE_TUNNEL_TOKEN
cp postgres/init/01-create-databases.sql.example postgres/init/01-create-databases.sql   # đổi mật khẩu từng service
cp redis/users.acl.example redis/users.acl                                              # đổi mật khẩu từng service
docker compose up -d postgres redis    # local không cần Traefik / cloudflared (profile server)
```

> File init của Postgres chỉ chạy khi thư mục `postgres/data` còn trống (lần khởi tạo đầu tiên).

**2. Backend** — trong mỗi repo BE: `cp config/config.json.example config/config.json` (mật khẩu khớp bước 1), rồi `go run ./cmd/server`. Chạy `Admin_Service` trước: `Caro_Game_Service` dùng nó để xác thực.

**3. Frontend** — trong mỗi repo FE: `pnpm install`, `cp .env.example .env`, `pnpm dev`. Mở `http://localhost:3000`; game Caro mở qua `http://localhost:3000/games/caro` (cùng origin với Plexo_UI). Tài khoản dev seed sẵn: `seed.dev_users` trong `config.json` của `Admin_Service`.

| Thành phần | URL local |
|---|---|
| `Plexo_UI` | http://localhost:3000 (chuyển tiếp `/games/caro` sang `Caro_Game_UI`) |
| `Caro_Game_UI` | http://localhost:3001/games/caro |
| `Notification_Service` | http://localhost:8001/notification |
| `Admin_Service` | http://localhost:8002/admin |
| `Caro_Game_Service` | http://localhost:8003/caro-game |
| Postgres / Redis | `localhost:5432` / `localhost:6379` (theo `POSTGRES_PORT` / `REDIS_PORT`) |

Mỗi BE có tài liệu OpenAPI tự sinh tại `<base_path>/docs`.

## Thiết lập lần đầu (domain, Cloudflare, GitHub, server)

### 1. Domain → Cloudflare

Domain `thanhldt.site` mua ở TENTEN; DNS chuyển sang Cloudflare để dùng Cloudflare Tunnel (server không cần IP công khai / mở port, HTTPS do Cloudflare cấp).

1. Cloudflare dashboard → **Add a domain** → `thanhldt.site` → gói **Free**. Cloudflare quét bản ghi DNS (đang trống, bỏ qua) và cấp **2 nameserver** (dạng `xxx.ns.cloudflare.com`).
2. Trang quản lý tên miền của TENTEN → `thanhldt.site` → **Đổi Name Server** (DNS server) sang 2 nameserver Cloudflare vừa cấp, xóa nameserver cũ của TENTEN.
3. Chờ Cloudflare báo domain **Active** (thường vài phút, tối đa ~24 giờ). Kiểm tra: `dig NS thanhldt.site +short`.
4. Cloudflare → **SSL/TLS** → chế độ **Full** (hoặc Flexible — tunnel đã mã hóa đoạn Cloudflare ↔ server) và bật **Always Use HTTPS**.

### 2. Cloudflare Tunnel (mỗi môi trường một tunnel)

Cloudflare **Zero Trust → Networks → Tunnels → Create a tunnel** (loại *Cloudflared*), đặt tên `plexo-dev` / `plexo-prod`. Ở bước cài connector chọn Docker, chỉ **lấy token** (chuỗi sau `--token`) đưa vào `CLOUDFLARE_TUNNEL_TOKEN` của `infrastructure/.env` tương ứng (bước 5) — container `cloudflared` của stack sẽ chạy nó. Thêm **Public hostname** (Cloudflare tự tạo bản ghi DNS):

| Tunnel | Hostname | Service |
|---|---|---|
| `plexo-dev` | `plexo-dev.thanhldt.site` | `HTTP` → `traefik:80` |
| `plexo-dev` | `api.plexo-dev.thanhldt.site` | `HTTP` → `traefik:80` |
| `plexo-prod` | `plexo.thanhldt.site` | `HTTP` → `traefik:80` |
| `plexo-prod` | `api.plexo.thanhldt.site` | `HTTP` → `traefik:80` |

`traefik` là tên service trong network của stack — cloudflared của stack nào chỉ thấy Traefik của stack đó. WebSocket (`wss://`) đi qua tunnel không cần cấu hình thêm.

### 3. GitHub organization `thanhldt-plexo`

- **Secrets** (cấp organization, Repository access: các repo Plexo): `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`.
- **Docker Hub**: repository `personal-project-plexo-<service>` cho 5 service (`notification_service`, `admin_service`, `caro_game_service`, `plexo_ui`, `caro_game_ui`).
- **Nhánh**: tạo `develop` từ `main` ở mọi repo có CI.
- **Environments** `dev` và `prod` trong từng repo có CI (Settings → Environments), Variables:

| Repo | Environment `dev` | Environment `prod` |
|---|---|---|
| Mọi repo có CI | `DEPLOY_DIR=/srv/app-dev/plexo` | — |
| `Plexo_UI` | `NEXT_PUBLIC_ADMIN_API_URL=https://api.plexo-dev.thanhldt.site/admin/v1` | `NEXT_PUBLIC_ADMIN_API_URL=https://api.plexo.thanhldt.site/admin/v1` |
| `Caro_Game_UI` | `NEXT_PUBLIC_CARO_GAME_API_URL=https://api.plexo-dev.thanhldt.site/caro-game/v1`<br>`NEXT_PUBLIC_CARO_GAME_WS_URL=wss://api.plexo-dev.thanhldt.site/caro-game/v1/ws`<br>`NEXT_PUBLIC_ADMIN_API_URL=https://api.plexo-dev.thanhldt.site/admin/v1`<br>`NEXT_PUBLIC_PLEXO_UI_URL=https://plexo-dev.thanhldt.site` | Như `dev`, đổi host sang `plexo.thanhldt.site` / `api.plexo.thanhldt.site` |

  Environment `prod` có thể giới hạn **Deployment branches and tags** chỉ cho tag `v*`.

### 4. Self-hosted runner

Runner `homelab-server-01` đăng ký ở **cấp organization** (Settings → Actions → Runners), có **custom label `homelab-server-01`** (workflow chọn bằng `runs-on: [self-hosted, homelab-server-01]`) và runner group cho phép các repo Plexo dùng. User chạy runner phải dùng được Docker (`sudo usermod -aG docker <user>`) và có quyền đọc / ghi `/srv/app-dev/plexo`. Nếu repository Docker Hub để private: `docker login` một lần trên server bằng user đó.

### 5. Server: một bản clone repo `Plexo` cho mỗi môi trường

```bash
git clone https://github.com/thanhldt-plexo/Plexo.git /srv/app-dev/plexo
cd /srv/app-dev/plexo/infrastructure
cp .env.example .env    # STACK_NAME=plexo-dev, POSTGRES_PORT=5433, REDIS_PORT=6380 (khác prod), CLOUDFLARE_TUNNEL_TOKEN của tunnel plexo-dev
cp postgres/init/01-create-databases.sql.example postgres/init/01-create-databases.sql   # đổi mật khẩu
cp redis/users.acl.example redis/users.acl                                              # đổi mật khẩu
docker compose --profile server up -d

cd ../service
cp .env.example .env    # STACK_NAME=plexo-dev, IMAGE_TAG=develop-latest, WEB_DOMAIN=plexo-dev.thanhldt.site, API_DOMAIN=api.plexo-dev.thanhldt.site
for s in notification_service admin_service caro_game_service; do
  cp $s/config/config.json.dev.example $s/config/config.json    # đổi mật khẩu khớp bước trên, admin_api_key
done
docker compose up -d
```

Môi trường prod làm tương tự trong `/srv/app-prod/plexo` với `STACK_NAME=plexo-prod`, port 5432 / 6379, tunnel `plexo-prod`, `IMAGE_TAG=latest`, host prod và file `config.json.prod.example`. Lần đầu cần có tag version đã build (xem *Phát hành*) trước khi `docker compose up -d` ở `service/`.

## Phát hành

1. Làm việc trên `develop`: mỗi push tự build image `:develop-latest` / `:develop-<sha>` và deploy lên dev (Telegram báo kết quả) → kiểm thử trên `plexo-dev.thanhldt.site`.
2. Pull request `develop` → `main` ở các repo thay đổi; CI chạy trên PR.
3. Merge rồi tạo tag version trên `main`:

   ```bash
   git checkout main && git pull
   git tag v1.0.2 && git push origin v1.0.2
   ```

   CI build image `:v1.0.2`, `:latest`, `:<sha>` với cấu hình prod; Telegram báo image sẵn sàng.
4. Trên server prod (`IMAGE_TAG=latest`, hoặc ghim `v1.0.2` trong `service/.env`):

   ```bash
   cd /srv/app-prod/plexo/service
   docker compose pull && docker compose up -d
   ```

**Rollback**: đặt `IMAGE_TAG` trong `service/.env` về phiên bản trước (ví dụ `v1.0.1`, hoặc `develop-<sha>` ở dev) rồi `docker compose pull && docker compose up -d`.

## Vận hành

- Log: `docker compose logs -f <service>` (trong `service/`); file log trong `service/<service>/tmp/`.
- Dừng / gỡ hạ tầng của stack phải kèm profile: `docker compose --profile server down` (thiếu profile thì Traefik / cloudflared không bị gỡ).
- Dữ liệu Postgres / Redis của stack nằm trong `infrastructure/postgres/data`, `infrastructure/redis/data` của bản clone đó.
- Cập nhật cấu hình compose: `git pull` trong bản clone rồi `docker compose up -d` lại.
