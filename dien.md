# Giám Sát Điện Năng Tiêu Thụ Real-time (Power Monitoring) - K3s Cluster

Tài liệu kỹ thuật chi tiết về hệ thống đo đạc, thu thập và trực quan hóa điện năng tiêu thụ (Watt / kWh / Chi phí) theo thời gian thực cho cụm server On-Premise K3s.

---

## 1. Thông Số Phần Cứng & Công Suất Tiêu Thụ Từng Server

| Server | Hostname | IP (Tailscale) | Cấu hình phần cứng | Công suất CPU (Intel RAPL) | Công suất thực tế tại ổ cắm (Wall Power) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **server-tang2** | `node1` | `100.69.133.58` | • **Mainboard**: X99H<br>• **CPU**: Intel Xeon E5-2678 v3 (12C/24T, TDP 120W)<br>• **RAM**: 16GB (2x 8GB DDR3 1600)<br>• **SSD**: 128GB SATA TXRUI X550<br>• **VGA**: NVIDIA GeForce 210 (GT218 - card xuất hình) | **~11.8 W** (Idle)<br>• Cảm biến: `package-0`<br>• Nhiệt độ: ~41°C | **~55W – 65W** (Idle)<br>• Khi Full load 100%: ~160W – 180W<br>*(Bao gồm CPU + GT210 11W + Main/quạt/RAM 25W + hao phí nguồn ATX)* |
| **server-tang3** | `monitor` | `100.112.150.56` | • **Thiết bị**: Laptop HP ProBook 430 G3<br>• **CPU**: Intel Core i3-6100U (2C/4T, TDP 15W)<br>• **RAM**: 8GB (2x 4GB DDR3L 1600)<br>• **SSD**: 120GB Kingston SA400<br>• **VGA**: Intel HD Graphics 520 (onboard) | **~3.4 W** (Idle)<br>• Core: 2.89W<br>• DRAM: 0.63W | **~10W – 12W** (Idle)<br>• Khi Full load 100%: ~20W – 25W<br>*(Laptop chip U siêu tiết kiệm điện khi tắt/gập màn hình)* |
| **server-tang4** | `node-X79` | `100.72.138.65` | • **Mainboard**: X79<br>• **CPU**: Intel Xeon E5-2696 v2 (12C/24T, TDP 115W)<br>• **RAM**: 32GB DDR3 ECC REG<br>• **SSD**: 120GB Kingston SV300<br>• **VGA**: NVIDIA GeForce 210 (GT218 - card xuất hình) | **~24.5 W** (Idle)<br>• Core: ~15W<br>• Nhiệt độ: ~37°C | **~75W – 80W** (Idle)<br>• Khi Full load 100%: ~160W – 180W<br>*(Xeon v2 22nm ăn điện idle cao hơn v3 + GT210 + Main/quạt/32GB RAM)* |

### Tổng hợp toàn cụm:
* **Tổng công suất CPU (RAPL)**: `~40W – 45W`
* **Tổng công suất đo tại ổ cắm (Wall Power)**: **`~136W – 145W`**
* **Điện năng tiêu thụ ước tính**: **`~3.3 – 3.5 kWh (số điện) / ngày`** $\rightarrow$ **`~100 – 110 số điện / tháng`**
* **Tiền điện ước tính**: Khoảng **250.000đ – 280.000đ / tháng** (tính theo đơn giá bình quân ~2.500đ/kWh).

> [!TIP]
> Hai server `server-tang2` và `server-tang4` đang cắm card xuất hình GT210. Card này chạy ngầm tốn khoảng **10W - 12W/card**. Nếu bo mạch hỗ trợ chạy không cần card hình (headless boot), rút bỏ 2 card này sẽ giảm được thêm **~22W – 25W** (~16 - 18 số điện/tháng).

---

## 2. Kiến Trúc Giám Sát (Architecture)

```text
┌───────────────────────┐
│ server-tang2          │──> port 9101 (/metrics) ──┐
│ (Python Micro-Exporter)│                           │
└───────────────────────┘                           │
┌───────────────────────┐                           │   Scrape (mỗi 2s)
│ server-tang3 (Monitor)│──> port 9101 (/metrics) ──┼──────────────────> [ Prometheus ]
│ (Python Micro-Exporter)│                           │                    (port 9090)
└───────────────────────┘                           │                         │
┌───────────────────────┐                           │                         ▼
│ server-tang4          │──> port 9101 (/metrics) ──┘                  [ Grafana Dashboard ]
│ (Python Micro-Exporter)│                                                 (port 3000)
└───────────────────────┘                                          UID: k3s-power-monitor
```

### 2.1. Micro-Exporter trên từng Server
* **Vị trí file**: `/opt/power-exporter/power_exporter.py`
* **Cơ chế hoạt động**:
  - Chạy ngầm bằng thread nền, đọc trực tiếp cảm biến phần cứng **Intel RAPL** (`/sys/class/powercap/intel-rapl/intel-rapl:0/energy_uj`) định kỳ mỗi 1.0 giây.
  - Tính toán công suất CPU:
    $$\Delta E = E_2 - E_1, \quad P_{\text{CPU}} = \frac{\Delta E / 1.000.000}{\Delta t} \quad (\text{Watts})$$
  - Tính toán công suất thực tế tại ổ cắm (Wall Power) dựa trên profile phần cứng:
    $$P_{\text{Wall}} = (P_{\text{CPU}} + \text{Base Offset}) \times \text{PSU Loss Multiplier}$$
    - `server-tang2`: `Offset = 36W` (GT210 11W + Main X99/quạt/RAM 25W), `Multiplier = 1.15`
    - `server-tang3`: `Offset = 6W` (Main laptop, màn hình gập), `Multiplier = 1.10`
    - `server-tang4`: `Offset = 39W` (GT210 11W + Main X79/quạt/32GB RAM 28W), `Multiplier = 1.15`
* **Endpoint HTTP**: Cung cấp `/metrics` (Prometheus format) và `/json` tại port `9101`.

### 2.2. Prometheus & Grafana trên máy Monitor (`server-tang3`)
* **Thư mục quản lý**: `/home/monitor/monitoring-power/`
* **Prometheus**:
  - Scrape interval: `2s` (thu thập gần như thời gian thực).
  - Kết nối qua địa chỉ IP Tailscale của cả 3 node.
* **Grafana**:
  - Datasource UID: `prometheus-power` (trỏ về container `http://power-prometheus:9090`).
  - Chế độ truy cập: Bật sẵn **Anonymous Viewer** (không cần đăng nhập vẫn xem được).
  - Tự động nạp sẵn Dashboard **⚡ K3s Cluster Power Consumption Monitor**.

---

## 3. Đường Dẫn Truy Cập Trực Tiếp (URLs)

* **Grafana Dashboard (Real-time)**:
  👉 **[http://100.112.150.56:3000/d/k3s-power-monitor/](http://100.112.150.56:3000/d/k3s-power-monitor/)**
  - Xem trực tiếp: Không cần login.
  - Tài khoản Admin (nếu cần chỉnh sửa): `admin` / `admin`

* **Prometheus Targets Status**:
  👉 **[http://100.112.150.56:9090/targets](http://100.112.150.56:9090/targets)**

* **Raw Metrics Endpoint từng node**:
  - `server-tang2`: [http://100.69.133.58:9101/metrics](http://100.69.133.58:9101/metrics)
  - `server-tang3`: [http://100.112.150.56:9101/metrics](http://100.112.150.56:9101/metrics)
  - `server-tang4`: [http://100.72.138.65:9101/metrics](http://100.72.138.65:9101/metrics)

---

## 4. Các Metrics & Câu Lệnh PromQL Mẫu

### 4.1. Bảng Metrics
| Metric Name | Ý nghĩa | Đơn vị |
| :--- | :--- | :--- |
| `node_power_cpu_watts` | Công suất CPU Package đo trực tiếp từ Intel RAPL | Watt |
| `node_power_estimated_wall_watts` | Công suất tổng cả case máy tại ổ cắm điện | Watt |
| `node_power_dram_watts` | Công suất thanh RAM tiêu thụ (nếu sensor hỗ trợ) | Watt |
| `node_cpu_temperature_celsius` | Nhiệt độ CPU | °C |

### 4.2. Câu lệnh PromQL thông dụng trên Grafana
```promql
# 1. Tổng công suất tiêu thụ cả cụm tại ổ cắm (Realtime):
sum(node_power_estimated_wall_watts)

# 2. Tổng công suất riêng 3 CPU:
sum(node_power_cpu_watts)

# 3. Ước tính số tiền điện / tháng (chạy 24/7, đơn giá 2.500đ/kWh):
sum(node_power_estimated_wall_watts) * 24 * 30 / 1000 * 2500

# 4. Công suất riêng từng node:
node_power_estimated_wall_watts{server="server-tang2"}
node_power_estimated_wall_watts{server="server-tang3"}
node_power_estimated_wall_watts{server="server-tang4"}
```

> [!IMPORTANT]
> - Trong câu lệnh PromQL của Grafana, nhãn điều kiện phải dùng **dấu nháy kép** (ví dụ: `{server="server-tang2"}`), không dùng nháy đơn.
> - Đối với các panel đồng hồ **Gauge** hoặc thẻ **Stat**, luôn bật tùy chọn **Instant: true** trong target để lấy giá trị tức thời thay vì đợi query theo dải thời gian (Range).

---

## 5. Hướng Dẫn Vận Hành & Quản Trị Hệ Thống

### 5.1. Quản lý Micro-Exporter trên từng Server
Service được cấu hình tự khởi động cùng OS qua `systemd`:
```bash
# Kiểm tra trạng thái service
sudo systemctl status power-exporter

# Khởi động lại service
sudo systemctl restart power-exporter

# Dừng service
sudo systemctl stop power-exporter

# Xem log đo đạc realtime
sudo journalctl -u power-exporter -f
```
* **File mã nguồn**: `/opt/power-exporter/power_exporter.py`
* **File service**: `/etc/systemd/system/power-exporter.service`

### 5.2. Quản lý Cụm Monitor (Prometheus & Grafana) trên `server-tang3`
```bash
cd /home/monitor/monitoring-power

# Xem trạng thái các container
docker compose ps

# Khởi động lại toàn bộ stack
docker compose restart

# Dừng hệ thống
docker compose down

# Khởi động lại sau khi chỉnh sửa config
docker compose up -d

# Xem log Prometheus hoặc Grafana
docker compose logs -f power-prometheus
docker compose logs -f power-grafana
```
* **Cấu hình Prometheus**: `/home/monitor/monitoring-power/prometheus/prometheus.yml`
* **Datasource Grafana**: `/home/monitor/monitoring-power/grafana/provisioning/datasources/prometheus.yml`
* **JSON Dashboard Grafana**: `/home/monitor/monitoring-power/grafana/dashboards/power_dashboard.json`

### 5.3. Cấu Hình Tự Động Khởi Động Cùng Hệ Thống (Auto-start on Boot)
Toàn bộ hệ thống đã được cài đặt tự động khởi động (auto-start) khi máy tính bật nguồn hoặc reboot:

1. **Trên cả 3 Server (`server-tang2`, `server-tang3`, `server-tang4`)**:
   - `power-exporter.service` đã được enable qua systemd:
     ```bash
     sudo systemctl enable power-exporter
     ```
   - Trạng thái kiểm tra: `systemctl is-enabled power-exporter` $\rightarrow$ `enabled`
   - Khi server reboot, `systemd` tự động nạp `/opt/power-exporter/power_exporter.py` chạy nền.

2. **Trên Server Monitor (`server-tang3`)**:
   - Docker daemon được kích hoạt chạy cùng OS:
     ```bash
     sudo systemctl enable docker
     ```
   - Tạo systemd unit `/etc/systemd/system/monitoring-power.service` để tự động kéo stack Docker Compose:
     ```ini
     [Unit]
     Description=Power Monitoring Stack (Prometheus & Grafana)
     Requires=docker.service
     After=docker.service network.target

     [Service]
     Type=oneshot
     RemainAfterExit=yes
     WorkingDirectory=/home/monitor/monitoring-power
     ExecStart=/usr/bin/docker compose up -d
     ExecStop=/usr/bin/docker compose stop
     TimeoutStartSec=0

     [Install]
     WantedBy=multi-user.target
     ```
   - Kích hoạt service:
     ```bash
     sudo systemctl enable --now monitoring-power.service
     ```
   - Khi máy `server-tang3` bật nguồn, systemd sẽ tự động khởi động cả container **Prometheus** và **Grafana** mà không cần can thiệp thủ công.
