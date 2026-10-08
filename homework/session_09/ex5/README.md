# Bài 5: Khởi chạy dịch vụ đơn giản với Docker Compose

## 1. Mục tiêu

- Viết `docker-compose.yml` tối giản để khởi chạy 1 dịch vụ web duy nhất (Nginx) bằng Docker Compose CLI v2.
- Thay thế lệnh `docker run -d -p 8082:80 nginx:alpine` bằng khai báo dạng file, để quản lý dịch vụ bằng `docker compose up -d`, `ps`, `down`.
- Kiểm chứng: `docker compose ps` hiển thị service `web` chạy trên cổng 8082, `curl http://localhost:8082` trả về trang chào mừng của Nginx.

## 2. Môi trường

| Thành phần | Giá trị (đo thực tế) |
|------------|---------------------|
| Hệ điều hành | Windows 11, shell Git Bash |
| Docker Desktop | Client 29.7.2 / Server 29.7.2 (engine Linux qua WSL2) |
| Docker Compose | v5.5.0 (plugin, gọi bằng `docker compose`) |
| Image | `nginx:alpine` (nginx/1.31.6) |
| Thư mục chạy | `compose-web`, nên tên project là `compose-web` |

Phiên bản đo bằng `docker version --format '{{.Client.Version}} / {{.Server.Version}}'` (kết quả `29.7.2 / 29.7.2`) và `docker compose version` (kết quả `Docker Compose version v5.5.0`).

## 3. File `docker-compose.yml`

```yaml
# Khai báo 1 dịch vụ web duy nhất chạy Nginx (Bài 5, Session 09)
services:
  web:
    image: nginx:alpine
    ports:
      - "8082:80"   # cổng máy chủ 8082 -> cổng container 80
```

Nội dung khai báo đúng như đề bài; chỉ bổ sung hai dòng comment `#` tiếng Việt. Không thêm `container_name` để giữ đúng đề, vì vậy Compose tự đặt tên container theo mẫu `<project>-<service>-<số thứ tự>`.

### Giải thích từng khóa

| Khóa | Ý nghĩa |
|------|---------|
| `services:` | Khóa gốc, chứa danh sách các dịch vụ (mỗi dịch vụ sinh ra một hoặc nhiều container). |
| `web:` | Tên dịch vụ do người dùng tự đặt. Tên này là DNS name trong mạng của project và là đối số của `docker compose logs web`, `exec web`... |
| `image: nginx:alpine` | Image dùng để tạo container; tag `alpine` là bản Nginx nền Alpine Linux, nhỏ gọn. Nếu máy chưa có image, Compose tự pull. |
| `ports:` | Danh sách ánh xạ cổng từ máy chủ vào container. |
| `"8082:80"` | Định dạng `HOST:CONTAINER`. Cổng 8082 của máy chủ chuyển tới cổng 80 của container (Nginx lắng nghe cổng 80 bên trong). |

Vì sao để `"8082:80"` trong dấu nháy: YAML 1.1 có thể hiểu các số dạng `xx:yy` là số hệ sáu mươi (sexagesimal), ví dụ `22:22` bị đọc thành số 1342. Đặt trong dấu nháy buộc YAML hiểu đây là chuỗi, và Compose tự tách thành cổng host và cổng container. Đây là khuyến nghị của tài liệu Compose.

Thụt lề: YAML dùng 2 dấu cách, không dùng Tab. Kiểm tra bằng grep (exit 1 nghĩa là không tìm thấy ký tự Tab nào):

```
$ LC_ALL=C.UTF-8 grep -nP "\t" docker-compose.yml; echo "exit=${PIPESTATUS[0]}"
exit=1
```

Ghi chú: trong Git Bash cần đặt `LC_ALL=C.UTF-8`, nếu không `grep -P` báo `grep: -P supports only unibyte and UTF-8 locales` (exit=2) vì file có tiếng Việt trong comment.

Kiểm tra cú pháp và xem cấu hình đã chuẩn hóa bằng `docker compose config`:

```
$ docker compose config
name: compose-web
services:
  web:
    image: nginx:alpine
    networks:
      default: null
    ports:
      - mode: ingress
        target: 80
        published: "8082"
        protocol: tcp
networks:
  default:
    name: compose-web_default
```

Có thể thấy: tên project `compose-web` (lấy từ tên thư mục), `8082` (published) -> `80` (target), và một mạng `default` tên `compose-web_default` được Compose tự thêm vào dù file không khai báo.

## 4. Sơ đồ hoạt động

```
  Máy chủ Windows (host)                    Docker Desktop (engine Linux, WSL2)
 +-------------------------+     +--------------------------------------------------+
 |                         |     |  network: compose-web_default (bridge)           |
 |  curl http://localhost  |     |   +------------------------------------------+   |
 |        :8082            |---->|   | container: compose-web-web-1             |   |
 |                         | 8082|   |   image: nginx:alpine                    |   |
 |  (cổng 8082 của host)   |  -> |   |   nginx lắng nghe cổng 80                |   |
 +-------------------------+  80 |   +------------------------------------------+   |
                                 +--------------------------------------------------+
        ports: "8082:80"  =  HOST 8082  ->  CONTAINER 80
```

## 5. Các bước thực hiện (output thật)

Toàn bộ lệnh và output nằm trong `terminal_log.txt`. Các lệnh chạy trong thư mục chứa `docker-compose.yml`.

### Bước 1: Khởi chạy `docker compose up -d`

Máy chưa có image `nginx:alpine` nên Compose pull image trước. Dùng `--progress plain` để output không chứa ký tự tiến trình động (các dòng `Creating`/`Created` của network xuất hiện hai lần là output thật của chế độ plain).

```
$ docker compose --progress plain up -d
 Image nginx:alpine Pulling 
 Image nginx:alpine Pulled 
 Network compose-web_default Creating 
 Network compose-web_default Creating 
 Network compose-web_default Created 
 Network compose-web_default Created 
 Container compose-web-web-1 Creating 
 Container compose-web-web-1 Created 
 Container compose-web-web-1 Starting 
 Container compose-web-web-1 Started 
```

Compose thực hiện theo thứ tự: pull image, tạo network `compose-web_default`, tạo container `compose-web-web-1`, khởi động container. Cờ `-d` (detached) giúp chạy nền, trả lại terminal.

### Bước 2: Kiểm tra `docker compose ps`

```
$ docker compose ps
NAME                IMAGE          COMMAND                  SERVICE   CREATED         STATUS                  PORTS
compose-web-web-1   nginx:alpine   "/docker-entrypoint.…"   web       2 seconds ago   Up Less than a second   0.0.0.0:8082->80/tcp, [::]:8082->80/tcp
```

Service `web` ở trạng thái Up, cổng `0.0.0.0:8082->80/tcp` (và bản IPv6 `[::]:8082->80/tcp`). Đúng kết quả mong đợi của đề.

Mạng do Compose tạo:

```
$ docker network ls --filter name=compose-web
NETWORK ID     NAME                  DRIVER    SCOPE
4665605c0e42   compose-web_default   bridge    local
```

### Bước 3: Xem log `docker compose logs web`

```
$ docker compose logs web
web-1  | /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
web-1  | /docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
web-1  | /docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
web-1  | 10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
web-1  | 10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
web-1  | /docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
web-1  | /docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
web-1  | /docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
web-1  | /docker-entrypoint.sh: Configuration complete; ready for start up
web-1  | 2026/10/08 16:49:59 [notice] 1#1: using the "epoll" event method
web-1  | 2026/10/08 16:49:59 [notice] 1#1: nginx/1.31.6
...
```

(Rút gọn: bản đầy đủ gồm thêm dòng `start worker process` cho từng worker, xem `terminal_log.txt`.) Tiền tố `web-1` là tên service kèm số thứ tự bản sao.

### Bước 4: Kiểm tra bằng curl (đúng đề)

Lệnh `curl http://localhost:8082`. Khi output bị pipe, curl in thêm bảng tiến trình ra stderr; lần chạy gốc có bảng đó trong `terminal_log.txt`. Dưới đây là output của lần chạy bổ sung với `-s` để bỏ bảng tiến trình:

```
$ curl -s http://localhost:8082
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy, 
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional 
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Kiểm tra header và tiêu đề trang:

```
$ curl -I http://localhost:8082
HTTP/1.1 200 OK
Server: nginx/1.31.6
Date: Thu, 08 Oct 2026 16:50:09 GMT
Content-Type: text/html
Content-Length: 896
Last-Modified: Tue, 15 Sep 2026 14:18:52 GMT
Connection: keep-alive
ETag: "6aa953cc-380"
Accept-Ranges: bytes

$ curl -s http://localhost:8082 | grep -o "<title>.*</title>"
<title>Welcome to nginx!</title>
```

(Với `curl -I`, bảng tiến trình curl cũng bị chép vào log gốc; phần trên bỏ bảng đó, chỉ giữ phần header thật.) Kết quả: HTTP 200, Server nginx, trang "Welcome to nginx!". Yêu cầu của đề đạt.

### Bước 5: Các thao tác quản lý thêm

Dừng dịch vụ nhưng giữ container, rồi xem bằng `ps -a`:

```
$ docker compose stop
 Container compose-web-web-1 Stopping 
 Container compose-web-web-1 Stopped 

$ docker compose ps -a
NAME                IMAGE          COMMAND                  SERVICE   CREATED          STATUS                              PORTS
compose-web-web-1   nginx:alpine   "/docker-entrypoint.…"   web       13 seconds ago   Exited (0) Less than a second ago   
```

Container ở trạng thái `Exited (0)` và cột PORTS trống vì không còn lắng nghe. Khởi động lại bằng `start`, rồi `restart`:

```
$ docker compose start
 Container compose-web-web-1 Starting 
 Container compose-web-web-1 Started 

$ docker compose ps
NAME                IMAGE          COMMAND                  SERVICE   CREATED          STATUS        PORTS
compose-web-web-1   nginx:alpine   "/docker-entrypoint.…"   web       15 seconds ago   Up 1 second   0.0.0.0:8082->80/tcp, [::]:8082->80/tcp

$ docker compose restart
 Container compose-web-web-1 Restarting 
 Container compose-web-web-1 Started 
```

Xem tiến trình bên trong container bằng `docker compose top` (1 tiến trình master và 16 worker nginx, chi tiết trong `terminal_log.txt`). Chạy lệnh trong container:

```
$ docker compose exec web nginx -v
nginx version: nginx/1.31.6
```

Lưu ý riêng cho Git Bash trên Windows: lệnh `docker compose exec web cat /etc/nginx/conf.d/default.conf` bị Git Bash tự đổi đường dẫn `/etc/...` thành `C:/Program Files/Git/etc/...` (lần chạy đầu trong log báo `cat: can't open 'C:/Program Files/Git/etc/nginx/conf.d/default.conf'`). Cách khắc phục là đặt `MSYS_NO_PATHCONV=1` (hoặc viết `//etc/...`). Chạy lại ở lần bổ sung trong log (dịch vụ được `up -d` lại bằng đúng file này):

```
$ MSYS_NO_PATHCONV=1 docker compose exec web cat /etc/nginx/conf.d/default.conf | head -12
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

```

### Bước 6: Hạ dịch vụ `docker compose down`

```
$ docker compose down
 Container compose-web-web-1 Stopping 
 Container compose-web-web-1 Stopped 
 Container compose-web-web-1 Removing 
 Container compose-web-web-1 Removed 
 Network compose-web_default Removing 
 Network compose-web_default Removed 
```

Kiểm tra đã dọn sạch:

```
$ docker compose ps
NAME      IMAGE     COMMAND   SERVICE   CREATED   STATUS    PORTS

$ docker ps -a --filter name=compose-web
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

$ docker network ls --filter name=compose-web
NETWORK ID   NAME      DRIVER    SCOPE

$ docker images nginx:alpine
IMAGE          ID             DISK USAGE   CONTENT SIZE   EXTRA
nginx:alpine   df221db836e1       94.4MB         27.2MB        
```

Container và network đã bị xóa; image `nginx:alpine` được giữ lại để lần `up` sau không phải pull lại.

## 6. Mở rộng

### 6.1 So sánh `docker run` và Docker Compose

Lệnh tương đương với file Compose của bài:

```
docker run -d --name compose-web-web-1 -p 8082:80 nginx:alpine
```

| Tiêu chí | `docker run` | Docker Compose |
|----------|-------------|----------------|
| Khai báo | Tham số rải trong dòng lệnh, dễ quên | Nằm trong file YAML, đọc và review được |
| Tái lập | Phải nhớ hoặc ghi lại lệnh | `docker compose up -d` ở bất kỳ máy nào có file |
| Nhiều dịch vụ | Chạy nhiều lệnh, tự nối mạng | Khai báo nhiều service trong cùng file, cùng mạng |
| Mạng | Mặc định bridge chung, hoặc tự tạo mạng | Tự tạo mạng riêng `<project>_default`, các service gọi nhau bằng tên service |
| Đặt tên | Tự đặt bằng `--name` | Theo project: `<project>-<service>-<n>`, tên project mặc định là tên thư mục |
| Dọn dẹp | Xóa từng container, mạng | `docker compose down` xóa cả container và network của project |

### 6.2 Ví dụ mở rộng file Compose (chỉ trình bày, không chạy)

Thêm `volumes` để mount thư mục `html` của máy chủ vào thư mục phục vụ web của Nginx, và `restart` để tự khởi động lại:

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8082:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro   # thư mục html cạnh file compose, chỉ đọc
    restart: unless-stopped               # tự chạy lại khi Docker khởi động, trừ khi đã stop thủ công
```

Khi đó, đặt file `html/index.html` tùy ý, `curl http://localhost:8082` sẽ trả về nội dung file đó thay cho trang mặc định. Phần này chỉ trình bày, chưa chạy trong bài.

### 6.3 Khác biệt `down` và `stop`

| Lệnh | Container | Network | Volume đặt tên | Dùng khi |
|------|-----------|---------|----------------|----------|
| `docker compose stop` | Dừng, giữ lại (`Exited`) | Giữ | Giữ | Tạm dừng, sau đó `start` lại nhanh |
| `docker compose down` | Dừng và xóa | Xóa | Giữ | Kết thúc project, dọn sạch tài nguyên |
| `docker compose down -v` | Dừng và xóa | Xóa | Xóa luôn volume khai báo trong file | Dọn cả dữ liệu; cẩn thận vì mất dữ liệu |

Với bài này file không khai báo volume nên `down` và `down -v` cho kết quả giống nhau.

## 7. Bảng lệnh Compose thường dùng

| Lệnh | Tác dụng |
|------|----------|
| `docker compose up -d` | Tạo và chạy các dịch vụ ở chế độ nền |
| `docker compose ps` | Liệt kê container của project (`-a` để hiện cả container đã dừng) |
| `docker compose logs web` | Xem log của dịch vụ `web` (`-f` để theo dõi liên tục) |
| `docker compose config` | Kiểm tra cú pháp, in cấu hình đã chuẩn hóa |
| `docker compose stop` / `start` | Dừng / chạy lại container, giữ container |
| `docker compose exec web <lệnh>` | Chạy lệnh trong container đang chạy |
| `docker compose down` | Dừng và xóa container, network của project |
| `docker compose down -v` | Như trên, xóa thêm volume |

## 8. Xử lý sự cố

| Hiện tượng | Nguyên nhân | Cách xử lý |
|-----------|-------------|-----------|
| `Bind for 0.0.0.0:8082 failed: port is already allocated` | Cổng 8082 đã bị container hoặc tiến trình khác chiếm | Dùng `docker ps` tìm container đang dùng cổng 8082 rồi dừng, hoặc đổi cổng host (ví dụ `"8083:80"`) |
| `yaml: line N: found character that cannot start any token` | Dùng ký tự Tab để thụt lề, hoặc ký tự lạ trong file | Thay Tab bằng 2 dấu cách; kiểm tra bằng `grep -nP "\t" docker-compose.yml` và `docker compose config` |
| `Cannot connect to the Docker daemon` / `error during connect` | Docker Desktop (daemon) chưa chạy | Mở Docker Desktop, đợi trạng thái running, kiểm tra bằng `docker version` |
| Tên file `compose.yaml` hay `docker-compose.yml`? | Compose v2 ưu tiên `compose.yaml` (dạng khuyến nghị), vẫn chấp nhận `docker-compose.yml` | Đề bài dùng `docker-compose.yml` nên giữ nguyên tên; không để đồng thời hai file khác nhau trong một thư mục |
| `docker-compose: command not found` | Lệnh cũ `docker-compose` (v1, viết liền) đã ngừng hỗ trợ | Dùng `docker compose` (có dấu cách, plugin v2) |
| `cat: can't open 'C:/Program Files/Git/etc/...'` khi `exec` trong Git Bash | Git Bash tự đổi đường dẫn bắt đầu bằng `/` | Đặt `MSYS_NO_PATHCONV=1` trước lệnh hoặc viết `//etc/...` |

## 9. Đối chiếu yêu cầu đề bài

| Yêu cầu | Kết quả | Bằng chứng |
|---------|---------|-----------|
| Tạo `docker-compose.yml` với image `nginx:alpine`, ports `"8082:80"` | Đạt | Mục 3 và file `docker-compose.yml` |
| YAML thụt lề 2 dấu cách, không Tab | Đạt | `grep -nP "\t"` trả `exit=1` |
| `docker compose up -d` | Đạt | Network `compose-web_default` Created, container `compose-web-web-1` Started |
| `docker compose ps` hiển thị `web` chạy cổng 8082 | Đạt | `0.0.0.0:8082->80/tcp` |
| `curl http://localhost:8082` trả trang chào mừng Nginx | Đạt | `<title>Welcome to nginx!</title>`, HTTP 200, `Server: nginx/1.31.6` |
| `docker compose down` | Đạt | Container và network Removed; `ps`, `docker ps -a`, `network ls` đều rỗng |
| Nộp `docker-compose.yml` và `README.md` | Đạt | Thư mục `homework/session_09/ex5/` |

## 10. Kết luận

Chỉ với một file YAML 6 dòng, Docker Compose khởi chạy được dịch vụ Nginx tương đương `docker run -d -p 8082:80 nginx:alpine`, đồng thời tự tạo mạng riêng `compose-web_default` và đặt tên container `compose-web-web-1` theo tên project. Ba lệnh `up -d`, `ps`, `down` đủ để bật, kiểm tra và dọn sạch toàn bộ tài nguyên của project, trong khi image `nginx:alpine` vẫn được giữ lại. Cách làm này dễ tái lập, dễ lưu trong Git và là nền tảng để mở rộng sang nhiều dịch vụ (ứng dụng, cơ sở dữ liệu) trong cùng một file.

Các file nộp: `docker-compose.yml`, `README.md` (báo cáo này), `terminal_log.txt` (toàn bộ lệnh và output thật).
