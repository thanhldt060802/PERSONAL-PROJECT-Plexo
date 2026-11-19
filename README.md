<h1>Hello, World!</h1>

Deploy:
- Tạo repository trên Docker Hub để setup github workflow trên Github.
- Tạo Telegram Bot (lấy bot token và chat id) để setup github workflow trên Github.
- Setup secrets cho repository trên Github: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`.
- Push code lên Github.
- Chuẩn bị trước khi build:
    - Tạo folder/file infrastructure/traefik chứa các cấu hình cần thiết trước cho Traefik.
    - Tạo folder/file service/notification_service/config/config.json (từ config.json.example của service) chứa cấu hình port internal là 8000. (tương tự với các service khác)
- Build từng service từ trên xuống (vì đây là bản demo nên dùng domain free).
