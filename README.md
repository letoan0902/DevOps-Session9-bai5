# DevOps Session 09 - Bài 5: Khởi chạy dịch vụ đơn giản với Docker Compose

Bài tập viết `docker-compose.yml` tối giản để chạy 1 dịch vụ web Nginx bằng Docker Compose CLI v2, thay cho lệnh `docker run` thủ công. Toàn bộ lệnh đã được chạy thật trên Windows 11 với Docker Desktop (Docker 29.7.2, Docker Compose v5.5.0).

Báo cáo chi tiết: [homework/session_09/ex5/README.md](homework/session_09/ex5/README.md)

## Danh sách file

| File | Mô tả |
|------|-------|
| `homework/session_09/ex5/docker-compose.yml` | File Compose khai báo dịch vụ `web` (nginx:alpine, cổng 8082:80) |
| `homework/session_09/ex5/README.md` | Báo cáo: giải thích, các bước chạy với output thật, bảng lệnh, xử lý sự cố |
| `homework/session_09/ex5/terminal_log.txt` | Toàn bộ lệnh và output thật khi thực hiện |
