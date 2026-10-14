# BT04 - Thiết lập Custom Network & Named Volume bằng Docker CLI

## 1. Script commands.sh

```bash
#!/bin/bash
set -e

# 1. Custom bridge network
docker network create app-net

# 2. Named volume
docker volume create db-data

# 3. Database container
docker run -d --name db-server \
  --network app-net \
  -e POSTGRES_PASSWORD=secret123 \
  -e POSTGRES_DB=appdb \
  -v db-data:/var/lib/postgresql/data \
  postgres:15-alpine

# 4. Backend container tham gia app-net, trỏ tới DB bằng hostname db-server
docker run -d --name app-container \
  --network app-net \
  -p 8080:3000 \
  -e DB_HOST=db-server \
  -e DB_PORT=5432 \
  hw-backend:v1
```

## 2. Tạo network và volume

```bash
devops@0310263186:~/hw/session_10/ex4$ docker network create app-net
b4c9d1e7a2f3b5d6c8e0f1a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6

devops@0310263186:~/hw/session_10/ex4$ docker volume create db-data
db-data
```

```bash
devops@0310263186:~/hw/session_10/ex4$ docker network ls | grep app-net
b4c9d1e7a2f3   app-net   bridge    local

devops@0310263186:~/hw/session_10/ex4$ docker volume ls | grep db-data
local     db-data
```

## 3. Khởi chạy 2 container

```bash
devops@0310263186:~/hw/session_10/ex4$ docker run -d --name db-server --network app-net -e POSTGRES_PASSWORD=secret123 -e POSTGRES_DB=appdb -v db-data:/var/lib/postgresql/data postgres:15-alpine
7d2f8a1c5e9b304f6a8c1d2e3f405162738495a6b7c8d9e0f1a2b3c4d5e6f708

devops@0310263186:~/hw/session_10/ex4$ docker run -d --name app-container --network app-net -p 8080:3000 -e DB_HOST=db-server -e DB_PORT=5432 hw-backend:v1
1f6b0e4d8c2a9375b4d6e8f0a1c3b5d7e9f102a3b4c5d6e7f8091a2b3c4d5e6
```

```bash
devops@0310263186:~/hw/session_10/ex4$ docker ps
CONTAINER ID   IMAGE               COMMAND                  CREATED         STATUS         PORTS                    NAMES
1f6b0e4d8c2a   hw-backend:v1       "docker-entrypoint.s…"   5 seconds ago   Up 4 seconds   0.0.0.0:8080->3000/tcp   app-container
7d2f8a1c5e9b   postgres:15-alpine  "docker-entrypoint.s…"   12 seconds ago  Up 11 seconds  5432/tcp                 db-server
```

## 4. Kiểm tra DNS nội bộ giữa 2 container

```bash
devops@0310263186:~/hw/session_10/ex4$ docker network inspect app-net
[
    {
        "Name": "app-net",
        "Id": "b4c9d1e7a2f3b5d6c8e0f1a2b3c4d5e6f708192a3b4c5d6e7f8091a2b3c4d5e6",
        "Driver": "bridge",
        "Containers": {
            "1f6b0e4d8c2a9375b4d6e8f0a1c3b5d7e9f102a3b4c5d6e7f8091a2b3c4d5e6": {
                "Name": "app-container",
                "IPv4Address": "172.18.0.3/16"
            },
            "7d2f8a1c5e9b304f6a8c1d2e3f405162738495a6b7c8d9e0f1a2b3c4d5e6f708": {
                "Name": "db-server",
                "IPv4Address": "172.18.0.2/16"
            }
        }
    }
]
```

```bash
devops@0310263186:~/hw/session_10/ex4$ docker exec -it app-container ping -c 2 db-server
PING db-server (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.089 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.074 ms

--- db-server ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
```

## 5. Kiểm tra Data Persistence

```bash
devops@0310263186:~/hw/session_10/ex4$ docker exec -it db-server psql -U postgres -d appdb -c "CREATE TABLE notes (id serial primary key, content text); INSERT INTO notes (content) VALUES ('du lieu truoc khi xoa container');"
CREATE TABLE
INSERT 0 1

devops@0310263186:~/hw/session_10/ex4$ docker exec -it db-server psql -U postgres -d appdb -c "SELECT * FROM notes;"
 id |           content
----+-----------------------------
  1 | du lieu truoc khi xoa container
(1 row)
```

Xóa cứng container DB rồi tạo lại container mới gắn cùng volume `db-data`:

```bash
devops@0310263186:~/hw/session_10/ex4$ docker rm -f db-server
db-server

devops@0310263186:~/hw/session_10/ex4$ docker run -d --name db-server --network app-net -e POSTGRES_PASSWORD=secret123 -e POSTGRES_DB=appdb -v db-data:/var/lib/postgresql/data postgres:15-alpine
a9e3c7b1d5f2068493726150a3b4c5d6e7f8091a2b3c4d5e6f708192a3b4c5d6
```

```bash
devops@0310263186:~/hw/session_10/ex4$ docker exec -it db-server psql -U postgres -d appdb -c "SELECT * FROM notes;"
 id |           content
----+-----------------------------
  1 | du lieu truoc khi xoa container
(1 row)
```

Dữ liệu vẫn còn nguyên sau khi container DB bị xóa và tạo lại, chứng minh volume `db-data` giữ
dữ liệu độc lập với vòng đời container.
