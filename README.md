# abp-monitoring

Stack monitoring server: Prometheus + Grafana + AlertManager + Node Exporter

## Ports

| Service       | Port   | URL                          |
| ------------- | ------ | ---------------------------- |
| Grafana       | `9900` | `http://<server-ip>:9900`    |
| Prometheus    | `9901` | `http://<server-ip>:9901`    |
| AlertManager  | `9902` | `http://<server-ip>:9902`    |
| node-exporter | `9100` | chỉ nội bộ                   |

---

## Cài đặt lần đầu

### 1. Clone và chuẩn bị config

```bash
git clone <repo-url>
cd abp-monitoring

cp .env.example .env
nano .env  # đổi GF_ADMIN_PASSWORD
```

### 2. Điền Telegram token vào AlertManager

```bash
nano alertmanager/alertmanager.yml
```

Tìm và thay 2 chỗ:

- `bot_token: ''` → token bot Telegram
- `chat_id: 0` → chat_id của group/channel

> Lấy bot token: nhắn `/newbot` cho [@BotFather](https://t.me/BotFather)
> Lấy chat_id: thêm [@userinfobot](https://t.me/userinfobot) vào group

### 3. Mở firewall (nếu dùng ufw)

```bash
sudo ufw allow 9900/tcp
sudo ufw allow 9901/tcp
sudo ufw allow 9902/tcp
```

### 4. Khởi động

```bash
docker compose up -d
docker compose ps  # kiểm tra tất cả Up
```

---

## Theo dõi CPU / RAM trên Grafana

Import dashboard Node Exporter Full:

1. Vào `http://<server-ip>:9900` → đăng nhập
2. Menu trái → **Dashboards** → **Import**
3. Nhập ID `1860` → **Load**
4. Chọn datasource **Prometheus** → **Import**
5. Dropdown **instance** chọn `vps-main`

Dashboard hiển thị: CPU, RAM, Disk I/O, Network, Load Average, Disk Space.

---

## Thêm VPS mới vào monitoring

Trên VPS cần monitor, cài node-exporter:

```bash
docker run -d \
  --name node-exporter \
  --restart unless-stopped \
  --pid host \
  --network host \
  -v /proc:/host/proc:ro \
  -v /sys:/host/sys:ro \
  -v /:/rootfs:ro \
  prom/node-exporter:v1.8.2 \
  --path.procfs=/host/proc \
  --path.sysfs=/host/sys \
  --path.rootfs=/rootfs \
  --web.listen-address=:9100
```

Sau đó thêm vào `prometheus/prometheus.yml`:

```yaml
- job_name: 'node-exporter-vps2'
  static_configs:
    - targets: ['<vps2-ip>:9100']
      labels:
        instance: 'vps2'
        env: 'production'
```

Reload Prometheus (không cần restart):

```bash
curl -X POST http://localhost:9901/-/reload
```

---

## Điều kiện trigger cảnh báo Telegram

Cấu hình trong `prometheus/rules/node-alerts.yml`:

| Alert              | Điều kiện                 | Liên tục | Mức       | Nhắc lại |
| ------------------ | ------------------------- | -------- | --------- | -------- |
| HostHighCpuLoad    | CPU > 85%                 | 5 phút   | warning   | 4 giờ    |
| HostOutOfMemory    | RAM còn < 10%             | 5 phút   | critical  | 1 giờ    |
| HostOutOfDiskSpace | Disk `/` đã dùng > 60%    | 5 phút   | critical  | 1 giờ    |
| HostSwapUsageHigh  | Swap > 80%                | 10 phút  | warning   | 4 giờ    |
| NodeExporterDown   | Mất kết nối node exporter | 2 phút   | critical  | 1 giờ    |

Khi có `critical` firing trên một máy → toàn bộ `warning` cùng máy bị chặn (tránh spam).

Khi tự khắc phục → gửi tin ✅ ĐÃ KHẮC PHỤC sau 5 phút.

---

## Quản lý

```bash
# Xem logs realtime
docker compose logs -f prometheus
docker compose logs -f alertmanager

# Reload config Prometheus không cần restart
curl -X POST http://localhost:9901/-/reload

# Dừng stack
docker compose down

# Xóa toàn bộ data (không thể hoàn tác)
docker compose down -v
```

---

## Truy cập từ xa qua SSH tunnel

Nếu cần truy cập từ máy cá nhân mà không mở public:

```bash
ssh -L 9900:127.0.0.1:9900 \
    -L 9901:127.0.0.1:9901 \
    -L 9902:127.0.0.1:9902 \
    -N user@<server-ip>
```

Sau đó mở `http://localhost:9900` trên máy local.
