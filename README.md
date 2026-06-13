# abp-monitoring

Stack monitoring cho hệ thống social-listening: Prometheus + Grafana + AlertManager + các exporter.

---

## Services & Ports

| Service | Port | Ghi chú |
| --- | --- | --- |
| Grafana | `9900` | Dashboard — bind `0.0.0.0` |
| Prometheus | `9901` | Query metrics — bind `127.0.0.1` |
| AlertManager | `9902` | Quản lý alert — bind `127.0.0.1` |
| node-exporter | `9101` | CPU/RAM/Disk host — `network_mode: host` |
| cAdvisor | `8080` | Container metrics — nội bộ |
| mongodb-exporter | `9216` | MongoDB metrics — nội bộ |
| redis-exporter | `9121` | Redis metrics — nội bộ |
| postgres-exporter | `9187` | PostgreSQL metrics — nội bộ |
| kafka-exporter | `9308` | Kafka lag — nội bộ |

---

## Cài đặt lần đầu

```bash
git clone <repo>
cd abp-monitoring
cp .env.example .env
nano .env        # điền token, password, connection strings
bash deploy.sh
```

### Biến môi trường

| Biến | Mô tả |
| --- | --- |
| `TELEGRAM_BOT_TOKEN` | Token bot Telegram — lấy từ [@BotFather](https://t.me/BotFather) |
| `TELEGRAM_CHAT_ID` | ID group/channel nhận cảnh báo |
| `GF_ADMIN_USER` | Tài khoản đăng nhập Grafana |
| `GF_ADMIN_PASSWORD` | Mật khẩu Grafana |
| `MONGODB_URI` | URI MongoDB — ký tự đặc biệt phải URL-encode (`@` → `%40`) |
| `REDIS_ADDR` | Địa chỉ Redis (`redis://host:port`) |
| `REDIS_PASSWORD` | Mật khẩu Redis — plain text, không encode |
| `POSTGRES_DSN` | DSN PostgreSQL — ký tự đặc biệt phải URL-encode |

---

## CI/CD

Push lên branch `main` → GitHub Actions SSH vào server → `git pull origin main` → `bash deploy.sh`.

`deploy.sh` thực hiện theo thứ tự:

1. Load `.env`
2. Generate `alertmanager/alertmanager.yml` từ template (inject token Telegram)
3. `docker compose up -d --remove-orphans`
4. Reload Prometheus áp dụng rules mới: `curl -X POST http://localhost:9901/-/reload`

---

## Luồng hoạt động

```text
Gateway (4416)            ─┐
node-exporter (9101)      ─┤
cAdvisor (8080)           ─┤──► Prometheus scrape 15s ──► evaluate rules ──► AlertManager ──► Telegram
mongodb-exporter (9216)   ─┤                          └──► Grafana dashboards
redis-exporter (9121)     ─┤
postgres-exporter (9187)  ─┤
kafka-exporter (9308)     ─┤
kafka-mongo-stream (9800) ─┘
```

---

## Alert Rules

### Critical — nhắc lại mỗi 1 giờ

| Alert | Điều kiện | For |
| --- | --- | --- |
| `ServiceDown` | Bất kỳ job nào không phản hồi | 2 phút |
| `NodeExporterDown` | node-exporter mất kết nối | 2 phút |
| `MongoContainerDown` | Container `mongodb_new` không thấy qua cAdvisor | 2 phút |
| `MongoDBDown` | `mongodb_up == 0` | 1 phút |
| `PostgreSQLDown` | `pg_up == 0` | 1 phút |
| `KafkaDown` | kafka-exporter không phản hồi | 1 phút |
| `PlatformNoData2h` | Crawler platform không có data 2h | 5 phút |
| `OOMKillDetected` | Kernel OOM Kill xảy ra | ngay |
| `DiskCritical` | Disk còn < 10% | 1 phút |
| `KafkaTotalLagExplosion` | Tổng Kafka lag > 200,000 message | 3 phút |
| `GatewayKafkaWriteFailure` | Ghi Kafka thất bại liên tục | 2 phút |
| `GatewayHigh500Rate` | Tỉ lệ HTTP 500 > 1% | 2 phút |

### Warning — nhắc lại mỗi 4 giờ

| Alert | Điều kiện | For |
| --- | --- | --- |
| `KafkaLagGrowing` | Kafka lag tăng đều > 50 msg/giây | 5 phút |
| `KafkaHighLag` | Lag theo topic > 500 message | 5 phút |
| `MongoDBTooManyConnections` | Kết nối MongoDB > 500 | 5 phút |
| `MongoDBConnectionsHigh` | Tỉ lệ connection MongoDB > 80% | 5 phút |
| `PlatformDataStalled` | facebook/youtube/web không có bài mới 1h | 5 phút |
| `GatewayDLQNotEmpty` | Dead letter queue > 0 | 5 phút |
| `GatewayUnknownPlatformBot` | Platform lạ > 100 req/s | 5 phút |

> Khi có `critical` firing trên một máy → toàn bộ `warning` cùng máy bị chặn (tránh spam).
>
> Platform `news_page`, `tt`, `thread` được loại trừ khỏi `PlatformNoData2h`.

---

## Quản lý

```bash
# Xem logs realtime
docker compose logs -f prometheus
docker compose logs -f alertmanager

# Reload Prometheus sau khi sửa rules hoặc config (không cần restart)
curl -X POST http://localhost:9901/-/reload

# Silence alert khi bảo trì (ví dụ: 2 giờ)
curl -X POST http://localhost:9902/api/v2/silences \
  -H 'Content-Type: application/json' \
  -d '{
    "matchers": [{"name": "instance", "value": "vps-main", "isRegex": false}],
    "startsAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
    "endsAt": "'$(date -u -d '+2 hours' +%Y-%m-%dT%H:%M:%SZ)'",
    "createdBy": "admin",
    "comment": "Maintenance"
  }'

# Dừng stack
docker compose down

# Xóa toàn bộ data (không thể hoàn tác)
docker compose down -v
```

---

## Truy cập từ máy local (SSH tunnel)

Prometheus và AlertManager chỉ bind `127.0.0.1`, cần SSH tunnel:

```bash
ssh -L 9900:127.0.0.1:9900 \
    -L 9901:127.0.0.1:9901 \
    -L 9902:127.0.0.1:9902 \
    -N user@103.97.125.64
```

Sau đó truy cập trên máy local:

- Grafana: <http://localhost:9900>
- Prometheus targets: <http://localhost:9901/targets>
- Prometheus alerts: <http://localhost:9901/alerts>
- AlertManager: <http://localhost:9902>

---

## Thêm VPS mới vào monitoring

**1.** Cài node-exporter trên VPS mới:

```bash
docker run -d --name node-exporter --restart unless-stopped \
  --pid host --network host \
  -v /proc:/host/proc:ro -v /sys:/host/sys:ro -v /:/rootfs:ro \
  prom/node-exporter:v1.8.2 \
  --path.procfs=/host/proc --path.sysfs=/host/sys \
  --path.rootfs=/rootfs --web.listen-address=:9101
```

**2.** Thêm job vào `prometheus/prometheus.yml`:

```yaml
- job_name: 'node-exporter-vps2'
  static_configs:
    - targets: ['<vps2-ip>:9101']
      labels:
        instance: 'vps2'
        env: 'production'
```

**3.** Reload Prometheus:

```bash
curl -X POST http://localhost:9901/-/reload
```
