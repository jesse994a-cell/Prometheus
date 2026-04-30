# 從 Nagios 邁向現代化：Prometheus + Grafana + Alertmanager 全面監控實戰指南

## 目錄
1. [前言](#前言)
2. [為什麼要升級？](#為什麼要升級)
3. [架構概覽](#架構概覽)
4. [環境準備](#環境準備)
5. [Docker Compose 完整部署](#docker-compose-完整部署)
6. [Prometheus 核心配置](#prometheus-核心配置)
7. [告警規則實戰](#告警規則實戰)
8. [Alertmanager 配置](#alertmanager-配置)
9. [Grafana 儀表板](#grafana-儀表板)
10. [進階技巧](#進階技巧)
11. [故障排除](#故障排除)
12. [遷移計劃](#遷移計劃)

---

## 前言

如果你曾在 Nagios 的配置地獄中掙扎，手動編寫 hundreds 行 cfg 檔，為了看一份簡單的 CPU 趨勢圖而苦惱，那麼這份指南就是為你準備的。

**這份文件的目標是：**
- 幫你建立一套「開箱即用」的現代化監控系統
- 強調穩定性與可維護性，而非花哨功能
- 提供完整的配置範例與實戰經驗

**讀者預設：**
- 具備 Linux 系統管理基礎（熟悉 systemd、日誌等）
- 了解 Nagios 或其他傳統監控工具
- 願意擁抱以 YAML 為中心的配置方式
- 偏好透過指令碼化管理基礎設施

---

## 為什麼要升級？

### Nagios 的陣痛

| 面向 | Nagios | Prometheus + Grafana |
|------|--------|---------------------|
| **配置方式** | 繁瑣的 .cfg 檔案，易出錯 | YAML 配置，結構清晰 |
| **數據查詢** | 需要外掛組合查詢 | 原生時間序列數據庫 |
| **趨勢分析** | 需額外工具（Graphite、RRDtool） | Prometheus 內建，Grafana 原生支持 |
| **報警精度** | 閾值報警，難以表達複雜邏輯 | PromQL 表達式，靈活精準 |
| **服務發現** | 手動維護主機列表 | 自動發現（Consul、Kubernetes、EC2 等） |
| **可擴展性** | 集中式架構，性能瓶頸明顯 | 分散式採集，易於擴展 |

### 實際收益

✅ **快速上手**：30 分鐘完成基礎部署  
✅ **降低維護成本**：減少 70% 的配置管理工作  
✅ **提升告警質量**：基於實時趨勢的智能報警  
✅ **團隊協作**：Grafana Dashboard 易於分享與溝通  
✅ **容器友好**：天生支持 Docker 和 Kubernetes  

---

## 架構概覽

```
┌─────────────────────────────────────────────────────┐
│                  數據獲取層 (Exporters)             │
├─────────────────────────────────────────────────────┤
│  node_exporter    │  blackbox_exporter  │  app_exporter │
│  (系統指標)       │  (外部服務探測)     │  (應用指標)    │
└────────────┬──────────────────────────┬────────────┘
             │                          │
             ▼                          ▼
┌─────────────────────────────────────────────────────┐
│                 數據處理層 (Prometheus)             │
├─────────────────────────────────────────────────────┤
│  • 時間序列數據庫                                   │
│  • 告警規則評估                                     │
│  • 服務發現整合                                     │
│  • 數據保留策略 (15天 → 長期存儲)                  │
└────────────┬──────────────────────────┬────────────┘
             │                          │
             ▼                          ▼
┌──────────────────────┐        ┌─────────────────────┐
│   告警治理層         │        │   視覺化層          │
│ (Alertmanager)       │        │  (Grafana + Loki)   │
├──────────────────────┤        ├─────────────────────┤
│ • 報警去重/分組      │        │ • 自訂儀表板        │
│ • 報警抑制/升級      │        │ • 日誌關聯分析      │
│ • 多渠道通知         │        │ • 性能趨勢追蹤      │
│  (郵件/Slack/LINE)  │        │ • 告警歷史查詢      │
└──────────────────────┘        └─────────────────────┘
```

### 數據流向

```
監控對象
   ↓
[Exporters] 收集指標 (9100/9115/8080 等埠口)
   ↓
[Prometheus] 定期拉取 (Scrape)，儲存到 TSDB
   ↓
[Rules Engine] 評估告警規則，觸發告警事件
   ↓
[Alertmanager] 分組、去重、路由、通知
   ↓
[告警渠道] Email / Slack / LINE / Telegram / 自訂 Webhook
   ↓
[Grafana] 查詢數據，展示儀表板與告警狀態
```

---

## 環境準備

### 硬體需求

根據監控規模調整。以 **100 台主機 + 1000 指標/秒** 為基準：

| 元件 | CPU | 記憶體 | 磁碟 | 網路 |
|------|-----|--------|------|------|
| **Prometheus** | 2 核 | 4 GB | 50 GB SSD | 100 Mbps |
| **Grafana** | 1 核 | 2 GB | 10 GB | 100 Mbps |
| **Alertmanager** | 0.5 核 | 1 GB | 5 GB | 100 Mbps |
| **ELK/Loki** | 2 核 | 4 GB | 100 GB | 100 Mbps |

### 軟體依賴

```bash
# Ubuntu 22.04 LTS 為例
sudo apt-get update
sudo apt-get install -y \
  docker.io \
  docker-compose \
  curl \
  wget \
  git \
  jq
```

### 目錄結構

```
/opt/monitoring/
├── docker-compose.yml          # 整合編排配置
├── prometheus/
│   ├── prometheus.yml          # Prometheus 主配置
│   ├── alert_rules.yml         # 告警規則
│   └── targets/                # 服務發現配置
├── alertmanager/
│   └── config.yml              # Alertmanager 配置
├── grafana/
│   ├── provisioning/           # 儀表板預配置
│   └── dashboards/             # 儀表板 JSON 檔案
└── loki/
    └── loki-config.yaml        # Loki 配置
```

建立目錄結構：

```bash
mkdir -p /opt/monitoring/{prometheus/targets,alertmanager,grafana/provisioning/{dashboards,datasources},loki}
cd /opt/monitoring
```

---

## Docker Compose 完整部署

### 核心 docker-compose.yml

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
      - ./prometheus/targets:/etc/prometheus/targets:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'  # 允許熱重載
    restart: unless-stopped
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/config.yml:/etc/alertmanager/config.yml:ro
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
      - '--storage.path=/alertmanager'
    restart: unless-stopped
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources:ro
      - ./grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus
    restart: unless-stopped
    networks:
      - monitoring

  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
      - '--collector.netdev.device-exclude=^(veth.*)'
    restart: unless-stopped
    networks:
      - monitoring

  blackbox_exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox_exporter
    ports:
      - "9115:9115"
    volumes:
      - ./blackbox/config.yml:/etc/blackbox_exporter/config.yml:ro
    command:
      - '--config.file=/etc/blackbox_exporter/config.yml'
    restart: unless-stopped
    networks:
      - monitoring

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yaml:/etc/loki/local-config.yaml:ro
      - loki_data:/loki
    command:
      - '-config.file=/etc/loki/local-config.yaml'
    restart: unless-stopped
    networks:
      - monitoring

volumes:
  prometheus_data:
  alertmanager_data:
  grafana_data:
  loki_data:

networks:
  monitoring:
    driver: bridge
```

### 啟動服務

```bash
cd /opt/monitoring
docker-compose up -d

# 檢查狀態
docker-compose ps

# 檢視日誌
docker-compose logs -f prometheus
docker-compose logs -f alertmanager
docker-compose logs -f grafana
```

### 驗證部署

```bash
# 檢查 Prometheus
curl http://localhost:9090/metrics | head -20

# 檢查 Alertmanager
curl http://localhost:9093/api/v1/status

# 檢查 Grafana
curl http://localhost:3000/api/health

# 檢查 node_exporter
curl http://localhost:9100/metrics | grep node_

# 檢查 Blackbox exporter
curl http://localhost:9115/metrics
```

---

## Prometheus 核心配置

### prometheus.yml 詳解

```yaml
global:
  scrape_interval: 15s           # 預設拉取間隔
  scrape_timeout: 10s            # 單次拉取超時
  evaluation_interval: 15s       # 告警規則評估間隔
  external_labels:               # 外部標籤，附加至所有時間序列
    cluster: 'prod'              # 用於區分多個 Prometheus 實例
    env: 'production'

# 告警規則檔案
rule_files:
  - '/etc/prometheus/alert_rules.yml'

# 告警通知配置
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
      timeout: 10s

# Scrape 配置（數據獲取）
scrape_configs:

  # ==================== Prometheus 自身監控 ====================
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
    scrape_interval: 15s

  # ==================== 本地主機監控 ====================
  - job_name: 'node'
    static_configs:
      - targets: ['node_exporter:9100']
        labels:
          instance_type: 'docker-host'
          environment: 'production'

  # ==================== 遠端主機群組 (文件服務發現) ====================
  - job_name: 'linux-servers'
    file_sd_configs:
      - files: ['/etc/prometheus/targets/linux_servers.yml']
        refresh_interval: 30s
    relabel_configs:
      # 從標籤中提取主機名稱
      - source_labels: [__address__]
        target_label: hostname
        regex: '([^:]+)(?::\d+)?'
        replacement: '${1}'

  # ==================== Blackbox 外部服務監控 ====================
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]  # 檢查 HTTP 200 回應
    static_configs:
      - targets:
          - 'https://example.com'
          - 'https://api.example.com/health'
          - 'http://192.168.1.10:8080/metrics'
        labels:
          service: 'api-endpoint'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox_exporter:9115

  - job_name: 'blackbox-https-ssl'
    metrics_path: /probe
    params:
      module: [https_2xx]  # HTTPS + 憑證檢查
    file_sd_configs:
      - files: ['/etc/prometheus/targets/https_services.yml']
        refresh_interval: 30s
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox_exporter:9115

  - job_name: 'blackbox-ping'
    metrics_path: /probe
    params:
      module: [icmp]
    static_configs:
      - targets:
          - '192.168.1.1'
          - '8.8.8.8'
        labels:
          service: 'gateway'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox_exporter:9115

  # ==================== Docker 容器監控 (Consul 服務發現) ====================
  - job_name: 'docker-containers'
    consul_sd_configs:
      - server: 'consul:8500'
    relabel_configs:
      - source_labels: [__meta_consul_service]
        target_label: service
      - source_labels: [__meta_consul_node]
        target_label: node

  # ==================== 應用自訂指標 ====================
  - job_name: 'custom-app'
    static_configs:
      - targets: ['app-server:9200']
        labels:
          app: 'myapp'
          version: 'v2.1.0'
    scrape_interval: 30s  # 應用指標可減少拉取頻率

# ==================== 遠端存儲配置 (可選) ====================
# 用於長期存儲（Thanos、InfluxDB、S3 等）
# remote_write:
#   - url: http://thanos-receiver:19291/api/v1/receive
#     queue_config:
#       capacity: 10000
#       max_shards: 5
```

### 服務發現配置檔案

#### /prometheus/targets/linux_servers.yml

```yaml
- targets:
    - '192.168.1.10:9100'
    - '192.168.1.11:9100'
    - '192.168.1.12:9100'
  labels:
    group: 'production'
    datacenter: 'dc1'

- targets:
    - '192.168.2.10:9100'
    - '192.168.2.11:9100'
  labels:
    group: 'staging'
    datacenter: 'dc2'
```

#### /prometheus/targets/https_services.yml

```yaml
- targets:
    - 'https://api.example.com'
    - 'https://status.example.com'
  labels:
    service_type: 'api'
    alert_severity: 'critical'

- targets:
    - 'https://backup.example.com'
  labels:
    service_type: 'backup'
    alert_severity: 'warning'
```

### Blackbox Exporter 配置

創建 `/opt/monitoring/blackbox/config.yml`：

```yaml
modules:

  # HTTP 200-299 回應檢查
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 201, 202, 204, 206, 301, 302, 304, 307, 308]
      method: GET
      no_follow_redirects: false
      fail_if_body_not_matches_regexp:
        - 'ok'  # 可選：檢查回應內容
      fail_if_matches_regexp:
        - 'error'  # 拒絕包含 "error" 的回應

  # HTTPS 2xx 並檢查 SSL 憑證
  https_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 201, 202, 204, 206, 301, 302, 304, 307, 308]
      method: GET
      tls_config:
        insecure_skip_verify: false  # 驗證 SSL 憑證

  # ICMP Ping 檢查
  icmp:
    prober: icmp
    timeout: 5s
    icmp:
      preferred_ip_protocol: ipv4  # 優先使用 IPv4

  # TCP 連接檢查
  tcp_connect:
    prober: tcp
    timeout: 5s
    tcp:
      tls: false
      prefer_ipv4: true

  # DNS 查詢檢查
  dns_a:
    prober: dns
    timeout: 5s
    dns:
      preferred_ip_protocol: ipv4
      query_name: 'example.com'
      query_type: 'A'
```

---

## 告警規則實戰

### alert_rules.yml 詳解

將此檔案放在 `/opt/monitoring/prometheus/alert_rules.yml`：

```yaml
groups:
  # ==================== 主機資源告警 ====================
  - name: host_alerts
    interval: 15s
    rules:

      # CPU 使用率
      - alert: HostHighCpuUsage
        expr: |
          (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))
          * 100 > 85
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "主機 CPU 使用率過高 (實例: {{ $labels.instance }})"
          description: "{{ $labels.instance }} 的 CPU 使用率達 {{ $value | humanize }}% (閾值: 85%)"
          runbook_url: "https://wiki.example.com/alerts/high-cpu-usage"

      # 記憶體使用率
      - alert: HostHighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
          * 100 > 90
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "主機記憶體使用率過高 (實例: {{ $labels.instance }})"
          description: |
            {{ $labels.instance }} 的記憶體使用率達 {{ $value | humanize }}%
            可用記憶體: {{ (node_memory_MemAvailable_bytes / 1024 / 1024 / 1024) | humanize }}GB
          dashboard_url: "http://grafana:3000/d/sXQZcKkZz"

      # 磁碟使用率
      - alert: HostHighDiskUsage
        expr: |
          (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"} 
          / node_filesystem_size_bytes{fstype!~"tmpfs|fuse.lxcfs"}))
          * 100 > 85
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "主機磁碟空間不足 (實例: {{ $labels.instance }}, 裝置: {{ $labels.device }})"
          description: |
            {{ $labels.instance }} 上的 {{ $labels.device }} (掛載點: {{ $labels.mountpoint }})
            磁碟使用率達 {{ $value | humanize }}%
          device: "{{ $labels.device }}"

      # Inode 耗盡
      - alert: HostHighInodeUsage
        expr: |
          (1 - (node_filesystem_files_free{fstype!~"tmpfs|fuse.lxcfs"} 
          / node_filesystem_files{fstype!~"tmpfs|fuse.lxcfs"}))
          * 100 > 80
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "主機 Inode 耗盡風險 (實例: {{ $labels.instance }})"
          description: |
            {{ $labels.instance }} 上的 {{ $labels.device }} Inode 使用率達 {{ $value | humanize }}%
            建議立即清理舊日誌或臨時檔案

      # 磁碟 I/O 等待
      - alert: HostHighDiskIOWait
        expr: |
          avg by (instance) (rate(node_cpu_seconds_total{mode="iowait"}[5m]))
          * 100 > 20
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "主機磁碟 I/O 等待時間過長 (實例: {{ $labels.instance }})"
          description: "{{ $labels.instance }} 的 iowait 達 {{ $value | humanize }}% (閾值: 20%)"

      # 網路連接數
      - alert: HostHighNetworkConnections
        expr: |
          node_sockstat_TCP_alloc > 10000
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "主機 TCP 連接數過多 (實例: {{ $labels.instance }})"
          description: "{{ $labels.instance }} 的 TCP 連接數達 {{ $value | humanize }} (閾值: 10000)"

  # ==================== 磁碟健康度告警 ====================
  - name: disk_health_alerts
    interval: 15s
    rules:

      # SSD 壽命檢查 (需要 smartmontools exporter)
      - alert: SSDNearEndOfLife
        expr: |
          smartmon_ssd_remaining_lifespan_percent < 20
        for: 1h
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "SSD 壽命即將耗盡 (實例: {{ $labels.instance }}, 裝置: {{ $labels.device }})"
          description: |
            {{ $labels.instance }} 上的 {{ $labels.device }} 剩餘壽命只有 {{ $value | humanize }}%
            建議立即備份並規劃更換計畫
          device: "{{ $labels.device }}"

      # 磁碟讀取錯誤
      - alert: DiskReadErrors
        expr: |
          increase(node_disk_io_reads_merged_total[5m]) > 100
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "磁碟讀取錯誤增加 (實例: {{ $labels.instance }}, 裝置: {{ $labels.device }})"
          description: "{{ $labels.instance }} 上的 {{ $labels.device }} 在過去 5 分鐘內出現 {{ $value | humanize }} 次讀取合併"

  # ==================== 服務可用性告警 ====================
  - name: service_availability_alerts
    interval: 15s
    rules:

      # HTTP 服務無法存取
      - alert: HttpServiceDown
        expr: |
          probe_success{job="blackbox-http"} == 0
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "HTTP 服務不可用 (目標: {{ $labels.instance }})"
          description: |
            {{ $labels.instance }} 的 HTTP 檢查失敗
            狀態碼: {{ $value }}
            檢查頻率: 每 15 秒

      # HTTPS SSL 憑證即將過期
      - alert: HttpsSSLCertificateExpiringSoon
        expr: |
          probe_ssl_earliest_cert_expiry{job="blackbox-https-ssl"}
          - time() < 7 * 24 * 60 * 60  # 7 天
        for: 1h
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "SSL 憑證即將過期 (服務: {{ $labels.instance }})"
          description: |
            {{ $labels.instance }} 的 SSL 憑證將在 {{ $value | humanizeDuration }} 後過期
            建議立即更新憑證

      # HTTPS SSL 憑證已過期
      - alert: HttpsSSLCertificateExpired
        expr: |
          probe_ssl_earliest_cert_expiry{job="blackbox-https-ssl"}
          - time() < 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "SSL 憑證已過期 (服務: {{ $labels.instance }})"
          description: "{{ $labels.instance }} 的 SSL 憑證已於 {{ $value | humanizeTimestamp }} 過期，請立即更新"

      # HTTPS 回應時間過長
      - alert: HttpsHighResponseTime
        expr: |
          probe_duration_seconds{job="blackbox-https-ssl"} > 5
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "HTTPS 響應時間過長 (服務: {{ $labels.instance }})"
          description: "{{ $labels.instance }} 的平均回應時間達 {{ $value | humanize }} 秒"

  # ==================== Prometheus 自身健康檢查 ====================
  - name: prometheus_health_alerts
    interval: 15s
    rules:

      # Prometheus 配置載入失敗
      - alert: PrometheusConfigReloadFailure
        expr: |
          increase(prometheus_config_last_reload_successful[5m]) == 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Prometheus 配置重載失敗"
          description: "Prometheus 在過去 5 分鐘內無法重載配置，請檢查日誌"

      # Prometheus 告警評估耗時
      - alert: PrometheusSlowEvaluation
        expr: |
          histogram_quantile(0.95, rate(prometheus_rule_evaluation_duration_seconds_bucket[5m])) > 5
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Prometheus 告警規則評估速度過慢"
          description: "告警規則評估耗時達 {{ $value | humanize }} 秒 (P95)"

      # Alertmanager 無法連接
      - alert: AlertmanagerDown
        expr: |
          up{job="alertmanager"} == 0
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Alertmanager 服務不可用"
          description: "Alertmanager 無法連接，告警無法正常發送"

  # ==================== 自訂應用告警 ====================
  - name: application_alerts
    interval: 15s
    rules:

      # 應用錯誤率過高
      - alert: ApplicationHighErrorRate
        expr: |
          (sum(rate(http_requests_total{status=~"5.."}[5m])) by (job))
          / (sum(rate(http_requests_total[5m])) by (job))
          > 0.05
        for: 5m
        labels:
          severity: critical
          team: developers
        annotations:
          summary: "應用錯誤率過高 (應用: {{ $labels.job }})"
          description: "{{ $labels.job }} 的 5xx 錯誤率達 {{ $value | humanizePercentage }}"

      # 應用延遲過高
      - alert: ApplicationHighLatency
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
          team: developers
        annotations:
          summary: "應用響應延遲過高 (應用: {{ $labels.job }})"
          description: "{{ $labels.job }} 的 P95 延遲達 {{ $value | humanize }} 秒"
```

### 告警規則撰寫最佳實踐

#### 1. 選擇合適的 `for` 持續時間

```yaml
# 立即反應的告警（故障/緊急）
for: 1m
# 例如：服務宕機、磁碟滿、網路中斷

# 容忍瞬間波動（中長期趨勢）
for: 5m
# 例如：CPU 使用率、記憶體使用率

# 長期監控（緩慢問題）
for: 1h
# 例如：SSL 憑證即將過期、SSD 壽命耗盡
```

#### 2. 告警標籤分類

```yaml
labels:
  severity: critical    # critical / warning / info
  team: platform        # 負責團隊
  environment: prod     # 環境標識
  runbook: 'wiki/...'   # 應急手冊連結
```

#### 3. 有用的註釋

```yaml
annotations:
  summary: "簡明標題，{{ $labels.instance }} 出問題了"
  description: |
    詳細描述，包含：
    - 當前值：{{ $value }}
    - 閾值參考
    - 初步診斷建議
  dashboard_url: "http://grafana:3000/d/xxx"
  runbook_url: "https://wiki.example.com/..."
```

---

## Alertmanager 配置

### config.yml 詳解

創建 `/opt/monitoring/alertmanager/config.yml`：

```yaml
global:
  # 重新解析時間：如果告警未更新，多久後視為新的
  resolve_timeout: 5m
  # SMTP 配置（電郵通知）
  smtp_smarthost: 'smtp.example.com:587'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: '${SMTP_PASSWORD}'
  smtp_require_tls: true
  smtp_from: 'prometheus-alerts@example.com'

# 告警通知模板
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# 告警路由規則
route:
  # 預設分組鍵
  group_by: ['alertname', 'cluster', 'service']
  
  # 初次等待時間（待機期間聚合同類告警）
  group_wait: 30s
  
  # 後續等待時間（發送後，多久聚合新告警一起發送）
  group_interval: 5m
  
  # 重複發送間隔（同一告警每多久重新發送一次）
  repeat_interval: 4h
  
  # 預設收件者
  receiver: 'default'
  
  # 抑制規則：排除不需要通知的告警
  continue: false

  # 子路由：細粒度配置
  routes:
    # 臨界告警：立即通知 (Telegram + Email)
    - match:
        severity: critical
      receiver: 'critical-team'
      group_wait: 10s         # 縮短等待時間，立即發送
      group_interval: 15m
      repeat_interval: 1h     # 每小時重複通知一次
      continue: true

    # 警告告警：彙總通知 (Email)
    - match:
        severity: warning
      receiver: 'warning-team'
      group_wait: 2m
      group_interval: 30m
      repeat_interval: 12h

    # 平台團隊相關告警
    - match:
        team: platform
      receiver: 'platform-team'
      group_wait: 1m

    # 應用開發團隊相關告警
    - match:
        team: developers
      receiver: 'developer-team'

# 告警接收者配置
receivers:

  # 預設接收者
  - name: 'default'
    email_configs:
      - to: 'ops-team@example.com'
        headers:
          Subject: '[{{ .GroupLabels.alertname }}] 告警通知'

  # 臨界告警通知 (多渠道)
  - name: 'critical-team'
    # Telegram 通知
    telegram_configs:
      - bot_token: '${TELEGRAM_BOT_TOKEN}'
        chat_id: '${TELEGRAM_CHAT_ID_CRITICAL}'
        parse_mode: 'HTML'
        message: |
          🚨 <b>臨界告警</b>
          <b>告警:</b> {{ .GroupLabels.alertname }}
          <b>服務:</b> {{ .GroupLabels.service }}
          <b>數量:</b> {{ len .Alerts }}
          
          <b>詳情:</b>
          {{ range .Alerts }}
          • {{ .Labels.instance }} - {{ .Annotations.description }}
          {{ end }}
    
    # Email 通知
    email_configs:
      - to: 'critical-alerts@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'alerts@example.com'
        auth_password: '${SMTP_PASSWORD}'
        require_tls: true
        headers:
          Subject: '[{{ .Status | toUpper }}] 臨界告警: {{ .GroupLabels.alertname }}'
        html: |
          <h2>{{ .GroupLabels.alertname }}</h2>
          <p><b>狀態:</b> {{ .Status }}</p>
          {{ range .Alerts }}
            <hr>
            <p><b>實例:</b> {{ .Labels.instance }}</p>
            <p><b>描述:</b> {{ .Annotations.description }}</p>
            <p><b>值:</b> {{ .Annotations.summary }}</p>
          {{ end }}

    # 自訂 Webhook (用於整合內部系統)
    webhook_configs:
      - url: 'http://api.example.com/webhooks/alerts'
        send_resolved: true
        headers:
          X-Authorization: 'Bearer ${WEBHOOK_TOKEN}'

  # 警告告警
  - name: 'warning-team'
    email_configs:
      - to: 'ops-warnings@example.com'

  # 平台團隊
  - name: 'platform-team'
    # LINE Notify 通知
    webhook_configs:
      - url: 'https://notify-api.line.me/api/notify'
        send_resolved: true
        http_sd_configs:
          - authorization:
              credentials: '${LINE_NOTIFY_TOKEN}'
        headers:
          Content-Type: 'application/x-www-form-urlencoded'

  # 開發者團隊
  - name: 'developer-team'
    email_configs:
      - to: 'dev-team@example.com'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK_URL}'
        channel: '#alerts-development'
        title: 'Prometheus Alert'
        text: '{{ .CommonAnnotations.description }}'

# 抑制規則：阻止不必要的告警
inhibit_rules:

  # 父親故障告警，抑制子節點告警
  # 例如：主機宕機告警發出時，不需要再發 CPU 過高、磁碟滿等告警
  - source_match:
      severity: critical
      alertname: 'HostDown'
    target_match_re:
      severity: 'warning|info'
    equal: ['instance', 'cluster']

  # 資料庫故障時，抑制應用錯誤告警
  - source_match:
      severity: critical
      alertname: 'DatabaseDown'
    target_match:
      alertname: 'ApplicationHighErrorRate'
    equal: ['cluster']

  # 網路中斷時，抑制所有外部服務可用性告警
  - source_match:
      alertname: 'NetworkDown'
    target_match_re:
      alertname: '.*ServiceDown'
```

### 自訂通知模板

創建 `/opt/monitoring/alertmanager/templates/alerts.tmpl`：

```
{{/* Global constants */}}
{{ define "grafana_url" }}http://grafana:3000{{ end }}
{{ define "prometheus_url" }}http://prometheus:9090{{ end }}

{{/* 告警組摘要 */}}
{{ define "alert.group.summary" }}
{{ .GroupLabels.severity | toUpper }} 告警組
{{ if eq .Status "firing" }}
  🔴 {{ len .Alerts.Firing }} 個告警已觸發
{{ else }}
  ✅ {{ len .Alerts.Resolved }} 個告警已解決
{{ end }}
{{ end }}

{{/* HTML 格式的詳細訊息 */}}
{{ define "alert.html.details" }}
<table border="1" cellspacing="0" cellpadding="5">
  <thead>
    <tr style="background-color: #f2f2f2;">
      <th>告警名稱</th>
      <th>狀態</th>
      <th>實例</th>
      <th>描述</th>
      <th>開始時間</th>
    </tr>
  </thead>
  <tbody>
    {{ range .Alerts }}
    <tr>
      <td><b>{{ .Labels.alertname }}</b></td>
      <td>{{ .Status }}</td>
      <td>{{ .Labels.instance }}</td>
      <td>{{ .Annotations.description }}</td>
      <td>{{ .StartsAt.Format "2006-01-02 15:04:05" }}</td>
    </tr>
    {{ end }}
  </tbody>
</table>
{{ end }}

{{/* Slack 格式 */}}
{{ define "slack.default.text" }}
{{ if eq .Status "firing" }}
🚨 {{ .GroupLabels.alertname }}
{{ else }}
✅ {{ .GroupLabels.alertname }}
{{ end }}
集群: {{ .GroupLabels.cluster }}
{{ range .Alerts }}
  実例: {{ .Labels.instance }}
  詳細: {{ .Annotations.description }}
{{ end }}
{{ end }}
```

---

## Grafana 儀表板

### 初始化 Grafana

```bash
# 重置 Grafana 管理員密碼
docker exec -it grafana grafana-cli admin reset-admin-password newpassword

# 訪問 Grafana
# 瀏覽器: http://localhost:3000
# 預設帳號: admin / admin (或上面設定的密碼)
```

### 配置 Prometheus 數據源

創建 `/opt/monitoring/grafana/provisioning/datasources/prometheus.yml`：

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: true
```

### 匯入官方推薦的儀表板

#### 方式 1：使用 Grafana Dashboard ID

常用的官方儀表板 ID：

| Dashboard ID | 名稱 | 用途 |
|--------------|------|------|
| **1860** | Node Exporter for Prometheus | Linux 系統全面監控 |
| **3662** | Prometheus 2.0 Stats | Prometheus 自身性能 |
| **9851** | Alertmanager | 告警管理狀態 |
| **11074** | Node Exporter Server Metrics | 詳細的主機指標 |
| **12486** | Server Hardware Monitoring | 硬體 SMART 數據 |

**步驟：**
1. 登入 Grafana (http://localhost:3000)
2. 點選 `+` → `Import`
3. 輸入 Dashboard ID (例如 1860)
4. 選擇 Prometheus 作為數據源
5. 點選 Import

#### 方式 2：預先載入儀表板 JSON

創建 `/opt/monitoring/grafana/provisioning/dashboards/main.yml`：

```yaml
apiVersion: 1

providers:
  - name: 'Main Dashboards'
    orgId: 1
    folder: 'Monitoring'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

下載官方儀表板 JSON 檔案：

```bash
# 下載 Node Exporter 儀表板 (ID: 1860)
curl -L https://grafana.com/api/dashboards/1860/revisions/35/download \
  -o /opt/monitoring/grafana/dashboards/node-exporter-1860.json

# 下載 Prometheus 狀態儀表板 (ID: 3662)
curl -L https://grafana.com/api/dashboards/3662/revisions/2/download \
  -o /opt/monitoring/grafana/dashboards/prometheus-3662.json

# 重啟 Grafana
docker-compose restart grafana
```

### 自訂儀表板範例

以下範例展示如何創建一個完整的監控儀表板。可以手動在 Grafana UI 中建立，或者使用 JSON 模板。

#### 基礎架構監控儀表板 (JSON)

```json
{
  "dashboard": {
    "title": "基礎架構監控",
    "tags": ["monitoring", "infrastructure"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "主機狀態",
        "type": "stat",
        "targets": [
          {
            "expr": "count(up{job=\"node\"} == 1)"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "mappings": [
              {
                "type": "value",
                "options": {
                  "1": {
                    "text": "正常",
                    "color": "green"
                  }
                }
              }
            ]
          }
        }
      },
      {
        "id": 2,
        "title": "CPU 使用率",
        "type": "graph",
        "targets": [
          {
            "expr": "(1 - avg by (instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m]))) * 100",
            "legendFormat": "{{ instance }}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "custom": {
              "hideFrom": {
                "tooltip": false
              }
            },
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {
                  "color": "green",
                  "value": null
                },
                {
                  "color": "yellow",
                  "value": 70
                },
                {
                  "color": "red",
                  "value": 85
                }
              ]
            }
          }
        }
      },
      {
        "id": 3,
        "title": "記憶體使用率",
        "type": "gauge",
        "targets": [
          {
            "expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100",
            "legendFormat": "{{ instance }}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "max": 100,
            "min": 0,
            "thresholds": {
              "mode": "absolute",
              "steps": [
                { "color": "green", "value": null },
                { "color": "yellow", "value": 70 },
                { "color": "red", "value": 90 }
              ]
            }
          }
        }
      },
      {
        "id": 4,
        "title": "磁碟使用率 TOP 5",
        "type": "table",
        "targets": [
          {
            "expr": "topk(5, (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100)",
            "format": "table",
            "instant": true
          }
        ]
      },
      {
        "id": 5,
        "title": "外部服務健康狀態",
        "type": "stat",
        "targets": [
          {
            "expr": "probe_success{job=\"blackbox-https-ssl\"}",
            "legendFormat": "{{ instance }}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "mappings": [
              {
                "type": "value",
                "options": {
                  "1": { "text": "可用", "color": "green" },
                  "0": { "text": "不可用", "color": "red" }
                }
              }
            ]
          }
        }
      }
    ]
  }
}
```

---

## 進階技巧

### 1. SSD 剩餘壽命監控

#### 方法 A：使用 smartmontools

安裝依賴：

```bash
# Ubuntu/Debian
sudo apt-get install smartmontools

# 檢查 SSD 狀態
sudo smartctl -a /dev/sda | grep -i "life"
```

使用 `node_exporter` 的 textfile 收集器：

```bash
# 創建 textfile 收集器目錄
sudo mkdir -p /var/lib/node_exporter/textfile_collector

# 編寫 SSD 壽命檢查指令碼
cat > /usr/local/bin/ssd_health.sh << 'EOF'
#!/bin/bash
# /usr/local/bin/ssd_health.sh

TEXTFILE_DIR="/var/lib/node_exporter/textfile_collector"
TEMP_FILE="${TEXTFILE_DIR}/ssd_health.prom.$$"

# 遍歷所有磁碟
for disk in /dev/sd[a-z]; do
  if ! sudo smartctl -i "$disk" 2>/dev/null | grep -q "device"; then
    continue
  fi
  
  DEVICE=$(basename "$disk")
  
  # 獲取磁碟序號
  SERIAL=$(sudo smartctl -i "$disk" 2>/dev/null | grep "Serial Number" | awk '{print $NF}')
  
  # 獲取 SSD 剩餘壽命百分比 (NVMe 驅動器)
  REMAINING=$(sudo smartctl -A "$disk" 2>/dev/null | grep "Percent_Lifetime_Remain" | awk '{print $10}')
  
  if [ -n "$REMAINING" ]; then
    cat >> "$TEMP_FILE" << EOF_METRIC
# HELP smartmon_ssd_remaining_lifespan_percent SSD 剩餘壽命百分比
# TYPE smartmon_ssd_remaining_lifespan_percent gauge
smartmon_ssd_remaining_lifespan_percent{device="$DEVICE",serial="$SERIAL"} $REMAINING
EOF_METRIC
  fi
done

# 原子操作：確保檔案完整性
mv "$TEMP_FILE" "${TEXTFILE_DIR}/ssd_health.prom"
EOF

chmod +x /usr/local/bin/ssd_health.sh

# 設定 cron 定時執行
echo "0 * * * * root /usr/local/bin/ssd_health.sh" | sudo tee /etc/cron.d/ssd-health
```

修改 docker-compose.yml，啟用 textfile 收集器：

```yaml
node_exporter:
  image: prom/node-exporter:latest
  container_name: node_exporter
  ports:
    - "9100:9100"
  volumes:
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro
    - /:/rootfs:ro
    - /var/lib/node_exporter/textfile_collector:/var/lib/node_exporter/textfile_collector:ro
  command:
    - '--path.procfs=/host/proc'
    - '--path.sysfs=/host/sys'
    - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    - '--collector.textfile.directory=/var/lib/node_exporter/textfile_collector'
  restart: unless-stopped
  networks:
    - monitoring
```

### 2. Inode 耗盡監控與告警

在 `alert_rules.yml` 中添加詳細的 Inode 監控：

```yaml
- alert: HostInodeUsageAbove80
  expr: |
    (1 - (node_filesystem_files_free / node_filesystem_files))
    * 100 > 80
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "{{ $labels.instance }} Inode 使用率 {{ $value }}%"

- alert: HostInodeUsageAbove95
  expr: |
    (1 - (node_filesystem_files_free / node_filesystem_files))
    * 100 > 95
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "{{ $labels.instance }} Inode 即將耗盡！當前使用率 {{ $value }}%"
```

診斷 Inode 耗盡的原因：

```bash
# 尋找包含大量小檔案的目錄
find / -type d -exec sh -c 'echo "{}: $(find {} -maxdepth 1 | wc -l)"' \; 2>/dev/null | sort -t: -k2 -nr | head -20

# 使用 lsof 檢查已刪除但仍被程序佔用的檔案
lsof +L1

# 清理舊日誌
journalctl --vacuum=7d
logrotate -f /etc/logrotate.conf
```

### 3. 自訂應用指標的集成

創建應用 exporter（以 Python 為例）：

```python
#!/usr/bin/env python3
# /opt/app_exporter/app_metrics.py

from prometheus_client import Counter, Gauge, Histogram, start_http_server
import time
import random

# 定義指標
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

http_request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint'],
    buckets=(0.1, 0.5, 1.0, 2.0, 5.0)
)

app_workers = Gauge(
    'app_workers_count',
    'Active worker threads'
)

db_connections = Gauge(
    'db_connections_active',
    'Active database connections'
)

cache_hits = Counter(
    'cache_hits_total',
    'Cache hits',
    ['cache_type']
)

if __name__ == '__main__':
    # 啟動 HTTP 伺服器在埠口 8000
    start_http_server(8000)
    
    # 模擬指標生成
    app_workers.set(8)
    db_connections.set(15)
    
    while True:
        # 模擬 HTTP 請求
        for method in ['GET', 'POST']:
            for endpoint in ['/api/users', '/api/products']:
                for status in [200, 201, 400, 500]:
                    duration = random.uniform(0.1, 2.0)
                    http_request_duration.labels(method=method, endpoint=endpoint).observe(duration)
                    http_requests_total.labels(method=method, endpoint=endpoint, status=status).inc()
        
        cache_hits.labels(cache_type='redis').inc(random.randint(1, 100))
        cache_hits.labels(cache_type='memory').inc(random.randint(1, 50))
        
        time.sleep(10)
```

在 `prometheus.yml` 中添加應用 exporter：

```yaml
scrape_configs:
  - job_name: 'custom-app'
    static_configs:
      - targets: ['app-server:8000']
    scrape_interval: 30s
```

### 4. 日誌與指標的聯動 (Grafana + Loki)

創建 Loki 配置 `/opt/monitoring/loki/loki-config.yaml`：

```yaml
auth_enabled: false

ingester:
  chunk_idle_period: 3m
  chunk_retain_period: 1m
  max_chunk_age: 1h
  chunk_encoding: snappy
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema:
        version: v11
        index:
          prefix: index_
          period: 24h

server:
  http_listen_port: 3100
  log_level: info

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
  filesystem:
    directory: /loki/chunks

chunk_store_config:
  max_look_back_period: 0s
```

配置日誌收集（Promtail）：

```bash
# docker-compose.yml 中添加
promtail:
  image: grafana/promtail:latest
  container_name: promtail
  volumes:
    - /var/log:/var/log:ro
    - /var/lib/docker/containers:/var/lib/docker/containers:ro
  command:
    - '-config.file=/etc/promtail/config.yml'
  networks:
    - monitoring
```

在 Grafana 中創建日誌查詢面板：

```
{job="prometheus"} | json | level="error"
{job="alertmanager"} | json | status="failed"
```

---

## 故障排除

### 常見問題與解決方案

#### 1. Prometheus 無法抓取指標

**症狀：** Prometheus UI 顯示 "UP" 為 0

**診斷步驟：**

```bash
# 檢查 Prometheus 日誌
docker-compose logs prometheus | tail -50

# 檢查 Prometheus 配置語法
docker exec prometheus promtool check config /etc/prometheus/prometheus.yml

# 檢查是否能連接到目標
docker exec prometheus wget -O- http://node_exporter:9100/metrics

# 檢查防火牆
netstat -tuln | grep 9100

# 檢查容器網路連接
docker network inspect monitoring
```

**常見原因與修復：**

| 原因 | 症狀 | 修復方法 |
|------|------|--------|
| 目標 Exporter 未啟動 | UP = 0 | 檢查目標容器：`docker-compose ps` |
| 網路隔離 | 連接超時 | 確認容器在同一 network，或使用主機 IP |
| 防火牆阻止 | 連接被拒絕 | 檢查防火牆規則：`ufw status` |
| 錯誤的埠口 | 連接失敗 | 驗證 prometheus.yml 中的埠口與實際一致 |
| TLS 證書問題 | 關鍵錯誤 | 禁用 TLS 驗證：`insecure_skip_verify: true` |

#### 2. Alertmanager 不發送告警

**症狀：** 告警觸發但未收到通知

**診斷步驟：**

```bash
# 查看 Alertmanager 日誌
docker-compose logs alertmanager

# 檢查 Alertmanager 配置
docker exec alertmanager amtool config routes

# 查詢當前告警
curl http://localhost:9093/api/v1/alerts

# 檢查告警分組
curl http://localhost:9093/api/v2/alerts/groups

# 測試 Webhook
curl -H "Content-Type: application/json" \
  -d '{"alerts":[{"status":"firing","labels":{"alertname":"Test"}}]}' \
  http://localhost:9093/api/v1/alerts
```

**常見原因：**

```yaml
# 1. Alertmanager 未收到告警
# → 檢查 prometheus.yml 中的 alerting 配置

# 2. 告警被 inhibit 規則抑制
# → 檢查 alertmanager/config.yml 中的 inhibit_rules

# 3. 通知渠道配置錯誤
# → 檢查郵件伺服器、Telegram token 等認證

# 4. 告警標籤不匹配路由規則
# → 使用 `amtool` 檢查路由匹配情況
```

```bash
# 使用 amtool 測試路由
docker exec alertmanager amtool routes match \
  severity=critical \
  team=platform \
  alertname=HostDown
```

#### 3. Grafana 儀表板無數據

**症狀：** Grafana 面板顯示 "No data"

**診斷步驟：**

```bash
# 驗證 Prometheus 數據源連接
# → Grafana UI: Configuration → Data Sources → Test

# 檢查查詢語句
# → 在 Prometheus UI (http://localhost:9090/graph) 直接測試查詢

# 查看 Grafana 日誌
docker-compose logs grafana | grep -i error

# 檢查指標是否存在
curl http://localhost:9090/api/v1/label/__name__/values | grep node_
```

**常見原因與修復：**

```yaml
# 1. 時間範圍不正確
# → 調整 Grafana 面板的時間選擇器

# 2. 指標名稱錯誤
# → 使用指標瀏覽器自動補全，或查看 Prometheus Targets 標籤

# 3. 標籤篩選器過於嚴格
# → 檢查 label_filter 是否排除了所有時間序列

# 4. 數據未進入 Prometheus
# → 檢查 Prometheus scrape targets 是否為 UP
```

#### 4. 高記憶體使用率或磁碟滿

**症狀：** Prometheus 容器記憶體不斷增長

**調整 retention 策略：**

```bash
# 在 docker-compose.yml 中調整保留時間
prometheus:
  command:
    - '--storage.tsdb.retention.time=7d'    # 縮短為 7 天
    - '--storage.tsdb.retention.size=10GB'  # 限制磁碟使用
```

**清理歷史數據：**

```bash
# 重啟 Prometheus 並清理 WAL
docker-compose down
rm -rf prometheus_data/wal

# 離線清理（需停止 Prometheus）
docker exec prometheus prometheus \
  --storage.tsdb.path=/prometheus \
  --storage.tsdb.retention.time=7d
```

#### 5. Prometheus 配置重載失敗

**症狀：** `prometheus_config_last_reload_successful` = 0

**診斷：**

```bash
# 驗證配置語法
docker exec prometheus promtool check config /etc/prometheus/prometheus.yml

# 如有錯誤，修正後重載
curl -X POST http://localhost:9090/-/reload

# 或重啟容器
docker-compose restart prometheus
```

**常見配置錯誤：**

```yaml
# 1. YAML 縮排錯誤
# 使用 yamllint 檢查
docker run --rm -v /opt/monitoring:/data sdesbure/yamllint /data

# 2. 不存在的標籤轉換
# → 檢查 relabel_configs 中的 source_labels 是否真實存在

# 3. 無效的正規表達式
# → 在 Prometheus UI 的 Status → Configuration 中查看錯誤
```

---

## 遷移計劃

### 分階段從 Nagios 遷移

#### 第 1 階段：並行部署（1-2 週）

```
┌─────────────────────────────────────────┐
│         舊系統 (Nagios)                  │
│  ├─ 主動告警系統                       │
│  └─ 舊儀表板報告                       │
└─────────────────────────────────────────┘
          ↓
┌─────────────────────────────────────────┐
│      新系統 (Prometheus + Grafana)      │
│  ├─ 相同的監控對象                     │
│  ├─ 相同的告警規則（新語法）           │
│  └─ 改進的儀表板                       │
└─────────────────────────────────────────┘

目標：驗證新系統的準確性和穩定性
```

**任務清單：**

- [ ] 部署 Prometheus + Grafana + Alertmanager
- [ ] 配置與 Nagios 相同的監控對象
- [ ] 遷移告警規則（Nagios cfg → PromQL）
- [ ] 配置通知渠道（郵件、短信等）
- [ ] 運行 1-2 週，對比告警準確性
- [ ] 收集團隊反饋，調整敏感度

#### 第 2 階段：切流量（1-2 天）

- [ ] 暫停 Nagios 告警發送
- [ ] 保持 Nagios 監控（只收集數據，不告警）
- [ ] 使用新系統作為主告警源 24 小時
- [ ] 監控告警發送延遲、漏報、誤報情況
- [ ] 發現問題，快速回滾至並行模式

#### 第 3 階段：下線舊系統

- [ ] 停止 Nagios 服務
- [ ] 歸檔 Nagios 配置與歷史數據
- [ ] 更新運維文檔，刪除 Nagios 相關內容

### Nagios 配置遷移範例

#### Nagios cfg 轉換至 PromQL

**Nagios 原始配置：**

```cfg
# nagios/objects/localhost.cfg
define service {
    use                     local-service
    host_name               web-server-01
    service_description     CPU Load
    check_command           check_nrpe!check_load
    check_interval          5
    max_check_attempts      3
    notification_interval   30
}

define command {
    command_name check_load
    command_line $USER1$/check_local_load -w 4,3,2 -c 6,5,4
}
```

**Prometheus 等價配置：**

```yaml
# prometheus/alert_rules.yml
- alert: HostHighLoadAverage
  expr: |
    (node_load1 > 4) or
    (node_load5 > 3) or
    (node_load15 > 2)
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "{{ $labels.instance }} 負載過高"
```

#### 常見 Nagios 模式的遷移對照

| Nagios 檢查 | 檢查方法 | Prometheus 遷移方案 |
|------------|---------|-------------------|
| check_http | HTTP 200 | blackbox_exporter (http_2xx) |
| check_https | HTTPS 200 | blackbox_exporter (https_2xx) |
| check_tcp | TCP 連接 | blackbox_exporter (tcp_connect) |
| check_dns | DNS 查詢 | blackbox_exporter (dns_a) |
| check_ping | ICMP | blackbox_exporter (icmp) |
| check_local_load | /proc/loadavg | node_exporter (node_load*) |
| check_local_disk_free | df | node_exporter (node_filesystem_*) |
| check_local_cpu | top | node_exporter (node_cpu_seconds_total) |
| check_nrpe | NRPE 遠端執行 | 自訂 Exporter |

### 告警規則遷移工具

編寫簡單的遷移指令碼：

```python
#!/usr/bin/env python3
# nagios_to_prometheus.py - 輔助遷移工具

import yaml
import json

def parse_nagios_threshold(nagios_check):
    """
    解析 Nagios 閾值語法，輸出 Prometheus PromQL
    例如：
      -w 80,70,60 -c 90,80,70
      → warning: 80, critical: 90
    """
    # 簡化版本，實際需要更複雜的解析
    import re
    
    pattern = r'-w\s+([\d,]+)\s+-c\s+([\d,]+)'
    match = re.search(pattern, nagios_check)
    
    if match:
        warn = match.group(1).split(',')[0]
        crit = match.group(2).split(',')[0]
        
        return {
            'warning': warn,
            'critical': crit
        }
    
    return None

# 遷移配置模板
prometheus_template = """
- alert: {alert_name}
  expr: {metric_name} > {threshold}
  for: 5m
  labels:
    severity: {severity}
  annotations:
    summary: "{description}"
"""

# 使用範例
nagios_checks = {
    "CPU Load": {
        "metric": "node_load1",
        "nagios_cmd": "-w 4 -c 6",
        "description": "CPU 負載過高"
    },
    "Disk Free": {
        "metric": "1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)",
        "nagios_cmd": "-w 20 -c 10",
        "description": "磁碟空間不足"
    }
}

for check_name, config in nagios_checks.items():
    thresholds = parse_nagios_threshold(config["nagios_cmd"])
    print(f"# {check_name}")
    print(prometheus_template.format(
        alert_name=check_name.replace(" ", ""),
        metric_name=config["metric"],
        threshold=thresholds.get("warning", "80"),
        severity="warning",
        description=config["description"]
    ))
```

---

## 效能優化建議

### 1. Prometheus 優化

```yaml
# prometheus.yml 優化
global:
  scrape_interval: 30s          # 減少拉取頻率，降低 CPU
  evaluation_interval: 60s      # 減少規則評估頻率
  external_labels:
    cluster: 'prod'

# 針對高基數指標的優化
metric_relabel_configs:
  # 例如：只保留 pod 級別的指標，捨棄 container 級別
  - source_labels: [__name__]
    regex: 'container_.*'
    action: drop
  
  # 捨棄特定高基數標籤
  - source_labels: [endpoint]
    regex: '(status|debug|trace)'
    action: drop
```

### 2. Grafana 優化

```yaml
# 減少儀表板查詢複雜度
# 避免：
- expr: sum without(job) (rate(http_requests_total[5m]))  # 容易產生高基數

# 優先選擇：
- expr: sum by(instance, method) (rate(http_requests_total[5m]))
```

### 3. 告警規則優化

```yaml
# 不推薦：每個實例都定義一個告警
- alert: Instance1HighCPU
- alert: Instance2HighCPU
# ...x100

# 推薦：使用標籤進行通用告警
- alert: HostHighCpuUsage
  expr: cpu_usage > 85
  labels:
    instance: "{{ $labels.instance }}"
```

---

## 安全性加固

### 基本安全措施

```yaml
# 1. Prometheus 反向代理認證
# nginx 配置
upstream prometheus {
    server prometheus:9090;
}

server {
    listen 80;
    location /prometheus {
        auth_basic "Prometheus";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://prometheus;
    }
}

# 2. Grafana 強制 HTTPS
environment:
  - GF_SERVER_ROOT_URL=https://grafana.example.com
  - GF_SECURITY_ADMIN_PASSWORD=${ADMIN_PASSWORD}
  - GF_SECURITY_SECRET_KEY=${SECRET_KEY}

# 3. Alertmanager SMTP 加密
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: '${SMTP_PASSWORD}'  # 使用環境變數
  smtp_require_tls: true
```

---

## 總結與最佳實踐

| 項目 | 建議 | 說明 |
|------|------|------|
| **拉取間隔** | 15-30 秒 | 平衡實時性與負載 |
| **保留時間** | 7-15 天 | 本地存儲；超過可用 Thanos |
| **評估間隔** | 15-30 秒 | 與拉取間隔一致 |
| **告警等待** | 30 秒~5 分鐘 | 避免瞬間波動觸發告警 |
| **重複通知** | 1-4 小時 | 根據嚴重程度調整 |
| **標籤命名** | snake_case | 便於查詢與分組 |
| **儀表板分享** | JSON 導出 | 便於版本控制與協作 |

---

## 參考資源

### 官方文檔
- [Prometheus 文檔](https://prometheus.io/docs/)
- [Grafana 文檔](https://grafana.com/docs/)
- [Alertmanager 文檔](https://prometheus.io/docs/alerting/latest/overview/)
- [node_exporter 指標列表](https://github.com/prometheus/node_exporter)

### 有用的工具
- `promtool`：Prometheus 配置驗證工具
- `amtool`：Alertmanager 管理工具
- `prometheus-query`：PromQL 查詢助手

### 社群資源
- [PromQL 教學](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Grafana Dashboard 社群](https://grafana.com/grafana/dashboards/)
- [Prometheus Reddit](https://www.reddit.com/r/prometheus/)

---

## 進階閱讀

推薦進一步學習的主題：
1. **高可用部署**：Prometheus 聯邦、Thanos 分佈式架構
2. **容器監控**：Kubernetes + Prometheus 整合、Helm Charts
3. **時間序列優化**：數據壓縮、樣本限制、查詢優化
4. **AIOps 集成**：與 PagerDuty、Opsgenie、Slack 深度整合
5. **成本優化**：遠端存儲、採樣策略、指標聚合

---

**文件版本**：v1.0  
**最後更新**：2026-04-30  
**維護者**：DevOps 團隊  
**反饋與建議**：請透過內部 Wiki 或 Slack 提出

