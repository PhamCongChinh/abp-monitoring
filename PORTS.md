# Ports & Cách dùng

## Bảng port

| Service | Host port | Internal port | URL |
|---|---|---|---|
| Grafana | `9900` | `3000` | http://localhost:9900 |
| Prometheus | `9901` | `9090` | http://localhost:9901 |
| AlertManager | `9902` | `9093` | http://localhost:9902 |
| node-exporter | `9100` | — | http://localhost:9100/metrics |

> Tất cả bind `127.0.0.1` — không expose ra internet. node-exporter dùng `network_mode: host` nên bind thẳng vào host.

---

## Khởi động

```bash
# Copy env và điền token Telegram
cp .env.example .env

# Chạy toàn bộ stack
docker compose up -d

# Kiểm tra trạng thái
docker compose ps
```

---

## Truy cập từ máy local (SSH tunnel)

```bash
# Mở tunnel cho cả 3 service một lần
ssh -L 9900:127.0.0.1:9900 \
    -L 9901:127.0.0.1:9901 \
    -L 9902:127.0.0.1:9902 \
    user@your-vps
```

Sau đó truy cập bình thường trên trình duyệt máy local.

---

## Grafana — http://localhost:9900

Đăng nhập: dùng `GF_ADMIN_USER` / `GF_ADMIN_PASSWORD` trong `.env`.

Thêm dashboard Node Exporter:
1. Dashboards → Import
2. Nhập ID `1860` (Node Exporter Full)
3. Chọn datasource `Prometheus` → Import

---

## Prometheus — http://localhost:9901

Kiểm tra targets đang scrape:

```
http://localhost:9901/targets
```

Reload config không cần restart (nhờ `--web.enable-lifecycle`):

```bash
curl -X POST http://localhost:9901/-/reload
```

Xem rules đang active:

```
http://localhost:9901/rules
http://localhost:9901/alerts
```

---

## AlertManager — http://localhost:9902

Xem alert đang firing:

```
http://localhost:9902/#/alerts
```

Silence alert tạm thời (ví dụ khi bảo trì):

```bash
# Silence mọi alert trên instance vps-main trong 2 giờ
curl -X POST http://localhost:9902/api/v2/silences \
  -H 'Content-Type: application/json' \
  -d '{
    "matchers": [{"name": "instance", "value": "vps-main", "isRegex": false}],
    "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
    "endsAt": "'$(date -u -d '+2 hours' +%Y-%m-%dT%H:%M:%SZ)'",
    "createdBy": "admin",
    "comment": "Maintenance"
  }'
```

---

## node-exporter — port 9100

Chạy trực tiếp trên host (không qua Docker network).
Prometheus scrape qua `host.docker.internal:9100` với label `instance: vps-main`.

Kiểm tra metrics thủ công:

```bash
curl http://localhost:9100/metrics | grep node_cpu
```

---

## Quản lý

```bash
# Xem logs realtime
docker compose logs -f prometheus
docker compose logs -f alertmanager

# Dừng stack
docker compose down

# Xóa toàn bộ data (không thể hoàn tác)
docker compose down -v
```
TEST