# Ansible + Prometheus：企業級監控自動化部署指南

## 目錄
1. [架構設計](#架構設計)
2. [Ansible 環境準備](#ansible-環境準備)
3. [Inventory 規劃](#inventory-規劃)
4. [目錄結構與角色組織](#目錄結構與角色組織)
5. [Server 端部署 (Prometheus/Grafana/Alertmanager)](#server-端部署)
6. [Client 端部署 (Exporter)](#client-端部署)
7. [動態配置與服務發現](#動態配置與服務發現)
8. [部署驗證與故障排除](#部署驗證與故障排除)
9. [持續維護與更新](#持續維護與更新)
10. [進階教學規劃](#進階教學規劃)

---

## 架構設計

### 部署拓撲

```
┌──────────────────────────────────────────────┐
│         Ansible Control Node                 │
│  (运维工作站或 CI/CD 服务器)                 │
│  • Playbooks                                 │
│  • Inventory                                 │
│  • Roles                                     │
└────┬──────────────────────────┬──────────────┘
     │                          │
     ▼                          ▼
┌─────────────────────┐  ┌────────────────────────────┐
│  Server Group       │  │  Client Group (多台主机)   │
│ (1-3 台)            │  │ (生产、预发、测试)         │
├─────────────────────┤  ├────────────────────────────┤
│ • Prometheus 9090   │  │ • node_exporter 9100       │
│ • Grafana 3000      │  │ • blackbox_exporter 9115   │
│ • Alertmanager 9093 │  │ • 应用 exporter (custom)   │
│ • Loki 3100         │  │ • Promtail (日志收集)      │
│ • Docker 容器       │  │ • Docker 容器              │
└─────────────────────┘  └────────────────────────────┘
     ▲                          │
     │ pull metrics             │
     └──────────────────────────┘
```

### 部署對象分類

| 角色 | 數量 | 服務 | 網路 | 磁碟 |
|------|------|------|------|------|
| **Server** | 1-3 | Prometheus/Grafana/Alertmanager/Loki | 100 Mbps | 100GB+ SSD |
| **Client** (Linux) | N | node_exporter/Promtail | 10 Mbps | 2GB |
| **Client** (應用) | N | 應用 exporter | 10 Mbps | 1GB |
| **Client** (網絡設備) | N | blackbox_exporter | 10 Mbps | 1GB |

---

## Ansible 環境準備

### 控制節點配置

```bash
# 1. 安裝 Ansible (Ubuntu 22.04)
sudo apt-get update
sudo apt-get install -y ansible git python3-jinja2

# 驗證版本
ansible --version  # 應為 2.10+

# 2. 安裝額外模組
sudo apt-get install -y \
  python3-docker \
  python3-yaml \
  python3-netaddr

# 3. 安裝 Ansible Galaxy 集合
ansible-galaxy collection install community.docker
ansible-galaxy collection install ansible.posix
```

### SSH 無密碼連接

```bash
# 生成 SSH 金鑰對（如果未有）
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# 部署公鑰至所有目標主機
for host in 10.0.1.{10..20}; do
  ssh-copy-id -i ~/.ssh/id_ed25519.pub user@$host
done

# 測試連接
ansible all -i inventory.ini -m ping
```

### Ansible 配置優化

創建 `~/.ansible/ansible.cfg`：

```ini
[defaults]
# 基本設定
inventory = ./inventory.ini
host_key_checking = False
remote_user = ansible
private_key_file = ~/.ssh/id_ed25519
forks = 10                    # 並行執行數
timeout = 30

# 輸出優化
force_color = True
log_path = /var/log/ansible.log
display_args_dont_log = ['password', 'secret']

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False       # 須配置 sudoers 無密碼
```

設定無密碼 sudo：

```bash
# 在所有目標主機上
echo "ansible ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/ansible
sudo chmod 0440 /etc/sudoers.d/ansible
```

---

## Inventory 規劃

### inventory.ini 結構

```ini
# ==================== 監控服務器組 ====================
[monitoring_server]
# Prometheus + Grafana + Alertmanager
monitor-server-01 ansible_host=10.0.1.10 datacenter=dc1 env=production
monitor-server-02 ansible_host=10.0.1.11 datacenter=dc1 env=production

[monitoring_server:vars]
# 服務器特有變數
prometheus_retention_days=15
prometheus_memory_limit=4g
grafana_admin_password={{ vault_grafana_password }}
alertmanager_smtp_host=smtp.example.com
alertmanager_smtp_port=587

# ==================== Linux 主機監控 ====================
[linux_servers]
# 生產環境
prod-web-01 ansible_host=10.0.2.10 group=web datacenter=dc1 env=production
prod-web-02 ansible_host=10.0.2.11 group=web datacenter=dc1 env=production
prod-db-01 ansible_host=10.0.3.10 group=database datacenter=dc1 env=production
prod-db-02 ansible_host=10.0.3.11 group=database datacenter=dc1 env=production

# 預發佈環境
staging-web-01 ansible_host=10.0.4.10 group=web datacenter=dc2 env=staging
staging-db-01 ansible_host=10.0.4.20 group=database datacenter=dc2 env=staging

# 開發環境
dev-server-01 ansible_host=10.0.5.10 group=misc datacenter=dc3 env=development

[linux_servers:vars]
# Linux 主機通用變數
node_exporter_port=9100
node_exporter_version=latest
collect_filesystem=true
collect_diskstats=true

# ==================== 應用服務器 ====================
[application_servers]
app-server-01 ansible_host=10.0.2.20 app_name=myapp app_port=8000 env=production
app-server-02 ansible_host=10.0.2.21 app_name=myapp app_port=8000 env=production
api-server-01 ansible_host=10.0.2.30 app_name=api app_port=9200 env=production

[application_servers:vars]
app_exporter_port=9201
app_metrics_path=/metrics

# ==================== 外部服務監控 ====================
[external_services]
# 僅在 Prometheus 配置中使用，無需 SSH 連接
external_api ansible_host=api.example.com url=https://api.example.com module=https_2xx
external_status ansible_host=status.example.com url=https://status.example.com module=https_2xx
backup_service ansible_host=backup.example.com url=https://backup.example.com module=https_2xx

[external_services:vars]
blackbox_module=https_2xx

# ==================== 分組變數 ====================
[all:vars]
# 全局變數
ansible_user=ansible
ansible_password={{ vault_ansible_password }}
docker_registry=docker.io
docker_compose_version=2.20.0

# Prometheus 全局設定
prometheus_scrape_interval=15s
prometheus_scrape_timeout=10s
prometheus_evaluation_interval=15s

# 告警通知
alertmanager_slack_webhook={{ vault_slack_webhook }}
alertmanager_telegram_token={{ vault_telegram_token }}
alertmanager_line_token={{ vault_line_token }}

# 環境標籤
prometheus_external_labels='{cluster: "prod", env: "{{ env }}"}'
```

### 動態 Inventory 方案 (可選)

適用於**雲環境**（AWS、阿里雲等），使用**插件自動發現**：

```yaml
# inventory/aws_ec2.yml
plugin: aws_ec2
regions:
  - us-east-1
  - ap-southeast-1
keyed_groups:
  - key: aws_ec2_tag_Environment
    prefix: env
  - key: aws_ec2_tag_Role
    prefix: role
hostnames:
  - private-ip-address  # 優先使用私有 IP
  - dns-name
```

使用方法：

```bash
# 測試動態 inventory
ansible-inventory -i inventory/aws_ec2.yml --list

# 執行 Playbook
ansible-playbook -i inventory/aws_ec2.yml playbooks/deploy.yml
```

---

## 目錄結構與角色組織

### 標準 Ansible 項目結構

```
prometheus-monitoring/
├── ansible.cfg                    # Ansible 全局配置
├── inventory.ini                  # 靜態 Inventory
├── inventory/                     # 動態 Inventory 目錄
│   ├── aws_ec2.yml               # AWS 自動發現
│   └── docker_compose.yml        # Docker 動態源
├── roles/                         # Ansible 角色
│   ├── prometheus_server/        # Server 部署角色
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── docker.yml
│   │   │   └── configure.yml
│   │   ├── templates/
│   │   │   ├── prometheus.yml.j2
│   │   │   ├── alert_rules.yml.j2
│   │   │   └── docker-compose.yml.j2
│   │   ├── defaults/
│   │   │   └── main.yml          # 預設變數
│   │   ├── vars/
│   │   │   └── main.yml          # 角色特定變數
│   │   └── handlers/
│   │       └── main.yml          # 服務重啟等事件
│   ├── node_exporter/            # node_exporter 角色
│   │   ├── tasks/
│   │   ├── templates/
│   │   └── defaults/
│   ├── blackbox_exporter/        # blackbox_exporter 角色
│   ├── grafana_server/           # Grafana 服務器角色
│   ├── alertmanager_server/      # Alertmanager 角色
│   ├── loki_server/              # Loki 日誌聚合角色
│   └── common/                   # 通用基礎配置角色
├── playbooks/                     # Playbook 腳本
│   ├── deploy_monitoring_server.yml
│   ├── deploy_monitoring_clients.yml
│   ├── deploy_all.yml
│   ├── update_rules.yml
│   ├── health_check.yml
│   └── rollback.yml
├── group_vars/                    # 分組變數
│   ├── monitoring_server.yml     # Server 組變數
│   ├── linux_servers.yml         # Linux 客戶端組變數
│   └── all.yml                   # 全局變數
├── host_vars/                     # 主機特定變數
│   ├── monitor-server-01.yml
│   └── prod-web-01.yml
├── library/                       # 自訂模組（可選）
├── vault/                         # Ansible Vault 加密檔案
│   └── credentials.yml            # 敏感訊息（加密）
├── files/                         # 靜態檔案
│   ├── prometheus_rules/
│   ├── grafana_dashboards/
│   └── scripts/
├── templates/                     # 動態模板
└── tests/                         # 測試用例
    ├── test_prometheus.yml
    └── test_exporter.yml
```

---

## Server 端部署

### prometheus_server 角色詳解

#### roles/prometheus_server/defaults/main.yml

```yaml
---
# Prometheus 預設配置
prometheus_version: "latest"
prometheus_port: 9090
prometheus_data_dir: "/var/lib/prometheus"
prometheus_config_dir: "/etc/prometheus"
prometheus_user: "prometheus"
prometheus_group: "prometheus"

# 數據保留策略
prometheus_retention_days: 15
prometheus_retention_size: "50GB"

# 資源限制
prometheus_memory_limit: "4g"
prometheus_memory_reservation: "2g"
prometheus_cpus: "2"

# Scrape 配置
prometheus_scrape_interval: "15s"
prometheus_scrape_timeout: "10s"
prometheus_evaluation_interval: "15s"

# 告警相關
prometheus_alert_rules_dir: "/etc/prometheus/rules"
alertmanager_host: "alertmanager"
alertmanager_port: 9093

# Docker 設定
prometheus_container_name: "prometheus"
prometheus_network: "monitoring"
prometheus_restart_policy: "unless-stopped"
```

#### roles/prometheus_server/tasks/main.yml

```yaml
---
- name: "Prometheus Server 部署流程"
  block:

    - name: "引入 Docker 相關任務"
      include_tasks: docker.yml

    - name: "建立 Prometheus 目錄結構"
      file:
        path: "{{ item }}"
        state: directory
        owner: "{{ prometheus_user }}"
        group: "{{ prometheus_group }}"
        mode: '0755'
      loop:
        - "{{ prometheus_data_dir }}"
        - "{{ prometheus_config_dir }}"
        - "{{ prometheus_alert_rules_dir }}"

    - name: "部署 Prometheus 主配置檔"
      template:
        src: prometheus.yml.j2
        dest: "{{ prometheus_config_dir }}/prometheus.yml"
        owner: "{{ prometheus_user }}"
        group: "{{ prometheus_group }}"
        mode: '0644'
        backup: yes
      notify: "重啟 Prometheus"

    - name: "部署告警規則檔"
      template:
        src: alert_rules.yml.j2
        dest: "{{ prometheus_alert_rules_dir }}/rules.yml"
        owner: "{{ prometheus_user }}"
        group: "{{ prometheus_group }}"
        mode: '0644'
      notify: "重載 Prometheus 配置"

    - name: "部署 Docker Compose 配置"
      template:
        src: docker-compose.yml.j2
        dest: "{{ prometheus_config_dir }}/docker-compose.yml"
        owner: "{{ prometheus_user }}"
        group: "{{ prometheus_group }}"
        mode: '0644'
      notify: "重啟 Prometheus Docker"

    - name: "啟動 Prometheus 容器"
      docker_compose:
        project_src: "{{ prometheus_config_dir }}"
        state: present
        pull: yes
      register: prometheus_compose_output

    - name: "等待 Prometheus 健康檢查"
      uri:
        url: "http://localhost:{{ prometheus_port }}/-/healthy"
        method: GET
        status_code: 200
      register: prometheus_health
      until: prometheus_health.status == 200
      retries: 30
      delay: 5

    - name: "驗證 Prometheus 配置"
      uri:
        url: "http://localhost:{{ prometheus_port }}/api/v1/status/config"
        method: GET
      register: prometheus_config_status
      failed_when: prometheus_config_status.status != 200

  rescue:
    - name: "部署失敗：收集日誌"
      command: "docker-compose -f {{ prometheus_config_dir }}/docker-compose.yml logs"
      register: docker_logs

    - name: "部署失敗：輸出日誌"
      debug:
        msg: "{{ docker_logs.stdout }}"

    - name: "部署失敗：拋出錯誤"
      fail:
        msg: "Prometheus 部署失敗，請檢查上述日誌"
```

#### roles/prometheus_server/tasks/docker.yml

```yaml
---
- name: "安裝 Docker 依賴"
  block:
    - name: "安裝必要套件"
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
        update_cache: yes

    - name: "新增 Docker GPG 金鑰"
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present

    - name: "新增 Docker 存儲庫"
      apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/{{ ansible_lsb.id | lower }} {{ ansible_lsb.codename }} stable"
        state: present

    - name: "安裝 Docker CE"
      apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-compose-plugin
        state: present
        update_cache: yes

    - name: "建立 Docker 使用者"
      user:
        name: "{{ prometheus_user }}"
        groups: docker
        append: yes

    - name: "啟動 Docker 服務"
      systemd:
        name: docker
        enabled: yes
        state: started
```

#### roles/prometheus_server/templates/prometheus.yml.j2

```yaml
# Prometheus 配置檔 - 自動生成，請勿手動編輯
# 生成時間：{{ ansible_date_time.iso8601 }}

global:
  scrape_interval: {{ prometheus_scrape_interval }}
  scrape_timeout: {{ prometheus_scrape_timeout }}
  evaluation_interval: {{ prometheus_evaluation_interval }}
  external_labels:
    cluster: '{{ cluster_name | default("default") }}'
    env: '{{ env }}'
    datacenter: '{{ datacenter | default("unknown") }}'

rule_files:
  - '/etc/prometheus/rules/*.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - {{ alertmanager_host }}:{{ alertmanager_port }}
      timeout: 10s

scrape_configs:

  # Prometheus 自身
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
    scrape_interval: 15s

  # ==================== Linux 主機監控 ====================
{% for group in groups.get('linux_servers', []) %}
  - job_name: 'node-{{ hostvars[group].get('group', 'ungrouped') }}'
    static_configs:
      - targets:
{% for host in groups.get('linux_servers', []) %}
{% if hostvars[host].get('group') == hostvars[group].get('group') %}
          - '{{ hostvars[host].ansible_host }}:{{ node_exporter_port | default(9100) }}'
{% endif %}
{% endfor %}
        labels:
          group: '{{ hostvars[group].get('group', 'ungrouped') }}'
          environment: '{{ hostvars[group].get('env', 'unknown') }}'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
{% endfor %}

  # ==================== 應用服務監控 ====================
{% if groups.get('application_servers', []) %}
  - job_name: 'custom-app'
    static_configs:
      - targets:
{% for host in groups.get('application_servers', []) %}
          - '{{ hostvars[host].ansible_host }}:{{ hostvars[host].get('app_exporter_port', 9201) }}'
{% endfor %}
        labels:
{% for host in groups.get('application_servers', []) %}
      - targets:
          - '{{ hostvars[host].ansible_host }}:{{ hostvars[host].get('app_exporter_port', 9201) }}'
        labels:
          app: '{{ hostvars[host].get('app_name', 'unknown') }}'
          environment: '{{ hostvars[host].get('env', 'unknown') }}'
{% endfor %}
{% endif %}

  # ==================== 外部服務監控 ====================
{% if groups.get('external_services', []) %}
  - job_name: 'blackbox-https'
    metrics_path: /probe
    params:
      module: [https_2xx]
    static_configs:
      - targets:
{% for service in groups.get('external_services', []) %}
          - '{{ hostvars[service].get('url', hostvars[service].ansible_host) }}'
{% endfor %}
        labels:
          service_type: 'external-api'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 'blackbox_exporter:9115'
{% endif %}
```

#### roles/prometheus_server/templates/alert_rules.yml.j2

```yaml
# Prometheus 告警規則 - 自動生成
# 生成時間：{{ ansible_date_time.iso8601 }}

groups:
  - name: host_alerts
    interval: 15s
    rules:

      - alert: HostHighCpuUsage
        expr: |
          (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])))
          * 100 > 85
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "主機 {{ $labels.instance }} CPU 使用率過高"
          description: "當前使用率：{{ $value | humanize }}%"

      - alert: HostHighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
          * 100 > 90
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "主機 {{ $labels.instance }} 記憶體使用率過高"
          description: "當前使用率：{{ $value | humanize }}%"

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
          summary: "主機 {{ $labels.instance }} 磁碟空間不足"
          description: "{{ $labels.device }} ({{ $labels.mountpoint }}) 使用率：{{ $value | humanize }}%"
```

#### roles/prometheus_server/templates/docker-compose.yml.j2

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:{{ prometheus_version }}
    container_name: {{ prometheus_container_name }}
    ports:
      - "{{ prometheus_port }}:9090"
    volumes:
      - /etc/prometheus:/etc/prometheus:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time={{ prometheus_retention_days }}d'
      - '--storage.tsdb.retention.size={{ prometheus_retention_size }}'
      - '--web.enable-lifecycle'
    environment:
      - TZ=Asia/Taipei
    mem_limit: {{ prometheus_memory_limit }}
    mem_reservation: {{ prometheus_memory_reservation }}
    cpus: '{{ prometheus_cpus }}'
    restart: {{ prometheus_restart_policy }}
    networks:
      - {{ prometheus_network }}

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - /etc/prometheus/alertmanager:/etc/alertmanager:ro
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
    restart: {{ prometheus_restart_policy }}
    networks:
      - {{ prometheus_network }}

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD={{ grafana_admin_password }}
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
      - TZ=Asia/Taipei
    volumes:
      - grafana_data:/var/lib/grafana
    restart: {{ prometheus_restart_policy }}
    networks:
      - {{ prometheus_network }}

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - loki_data:/loki
    restart: {{ prometheus_restart_policy }}
    networks:
      - {{ prometheus_network }}

volumes:
  prometheus_data:
  alertmanager_data:
  grafana_data:
  loki_data:

networks:
  {{ prometheus_network }}:
    driver: bridge
```

#### roles/prometheus_server/handlers/main.yml

```yaml
---
- name: "重啟 Prometheus"
  docker_compose:
    project_src: /etc/prometheus
    state: restarted
    services:
      - prometheus

- name: "重載 Prometheus 配置"
  uri:
    url: "http://localhost:9090/-/reload"
    method: POST
    status_code: 200
  retries: 3
  delay: 5

- name: "重啟 Prometheus Docker"
  docker_compose:
    project_src: /etc/prometheus
    state: restarted

- name: "重啟 Alertmanager"
  docker_compose:
    project_src: /etc/prometheus
    state: restarted
    services:
      - alertmanager
```

---

## Client 端部署

### node_exporter 角色

#### roles/node_exporter/defaults/main.yml

```yaml
---
node_exporter_version: "1.6.1"
node_exporter_port: 9100
node_exporter_user: "node-exporter"
node_exporter_group: "node-exporter"
node_exporter_install_dir: "/opt/node_exporter"

# 啟用的收集器
node_exporter_collectors:
  - filesystem
  - diskstats
  - meminfo
  - netdev
  - cpu
  - loadavg
  - vmstat
  - textfile

# 禁用的收集器
node_exporter_disabled_collectors:
  - nfs
  - nfsd
  - edac
  - mdadm
```

#### roles/node_exporter/tasks/main.yml

```yaml
---
- name: "部署 node_exporter"
  block:

    - name: "建立 {{ node_exporter_user }} 使用者"
      user:
        name: "{{ node_exporter_user }}"
        groups: "{{ node_exporter_group }}"
        shell: /usr/sbin/nologin
        home: "{{ node_exporter_install_dir }}"
        createhome: no
        state: present

    - name: "建立安裝目錄"
      file:
        path: "{{ node_exporter_install_dir }}"
        state: directory
        owner: "{{ node_exporter_user }}"
        group: "{{ node_exporter_group }}"
        mode: '0755'

    - name: "下載 node_exporter"
      get_url:
        url: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
        dest: "/tmp/node_exporter-{{ node_exporter_version }}.tar.gz"
        mode: '0644'

    - name: "解壓 node_exporter"
      unarchive:
        src: "/tmp/node_exporter-{{ node_exporter_version }}.tar.gz"
        dest: "/tmp"
        remote_src: yes

    - name: "複製 node_exporter 二進制檔"
      copy:
        src: "/tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/node_exporter"
        dest: "{{ node_exporter_install_dir }}/node_exporter"
        owner: "{{ node_exporter_user }}"
        group: "{{ node_exporter_group }}"
        mode: '0755'
        remote_src: yes

    - name: "建立 textfile 收集器目錄"
      file:
        path: "{{ node_exporter_install_dir }}/textfile_collector"
        state: directory
        owner: "{{ node_exporter_user }}"
        group: "{{ node_exporter_group }}"
        mode: '0755'

    - name: "建立 systemd service"
      template:
        src: node_exporter.service.j2
        dest: /etc/systemd/system/node_exporter.service
        mode: '0644'
      notify: "重啟 node_exporter"

    - name: "啟用並啟動 node_exporter"
      systemd:
        name: node_exporter
        enabled: yes
        state: started
        daemon_reload: yes

    - name: "驗證 node_exporter 正常運行"
      uri:
        url: "http://localhost:{{ node_exporter_port }}/metrics"
        method: GET
      register: node_exporter_metrics
      until: node_exporter_metrics.status == 200
      retries: 5
      delay: 2

    - name: "清理臨時檔案"
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - "/tmp/node_exporter-{{ node_exporter_version }}.tar.gz"
        - "/tmp/node_exporter-{{ node_exporter_version }}.linux-amd64"

  rescue:
    - name: "部署失敗：檢查日誌"
      command: "journalctl -u node_exporter -n 50"
      register: node_exporter_logs

    - name: "輸出日誌"
      debug:
        msg: "{{ node_exporter_logs.stdout }}"

    - name: "拋出錯誤"
      fail:
        msg: "node_exporter 部署失敗"
```

#### roles/node_exporter/templates/node_exporter.service.j2

```ini
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
Type=simple
User={{ node_exporter_user }}
Group={{ node_exporter_group }}
ExecStart={{ node_exporter_install_dir }}/node_exporter \
  --web.listen-address=:{{ node_exporter_port }} \
  --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/) \
  --collector.netdev.device-exclude=^(veth.*|docker.*|br-.*|lo) \
{% for collector in node_exporter_collectors %}
  --collector.{{ collector }} \
{% endfor %}
{% for collector in node_exporter_disabled_collectors %}
  --no-collector.{{ collector }} \
{% endfor %}
  --collector.textfile.directory={{ node_exporter_install_dir }}/textfile_collector

Restart=always
RestartSec=10

# Security hardening
ProtectSystem=strict
ProtectHome=yes
NoNewPrivileges=yes
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
```

#### roles/node_exporter/handlers/main.yml

```yaml
---
- name: "重啟 node_exporter"
  systemd:
    name: node_exporter
    enabled: yes
    state: restarted
    daemon_reload: yes
```

### blackbox_exporter 角色 (概要)

```yaml
# roles/blackbox_exporter/tasks/main.yml

---
- name: "部署 blackbox_exporter (Docker)"
  docker_container:
    name: blackbox_exporter
    image: prom/blackbox-exporter:latest
    ports:
      - "9115:9115"
    volumes:
      - /etc/blackbox_exporter:/config:ro
    command: "--config.file=/config/blackbox.yml"
    restart_policy: unless-stopped
    networks:
      - name: monitoring
```

---

## 動態配置與服務發現

### Playbook：自動化生成 Prometheus 配置

#### playbooks/update_targets.yml

```yaml
---
- name: "動態更新 Prometheus 目標"
  hosts: monitoring_server
  gather_facts: yes
  vars_files:
    - ../group_vars/all.yml

  tasks:

    - name: "收集所有客戶端資訊"
      set_fact:
        all_targets: "{{ groups | dict2items | map(attribute='value') | flatten }}"

    - name: "生成目標檔案 (linux_servers.yml)"
      template:
        src: ../templates/linux_servers.yml.j2
        dest: /etc/prometheus/targets/linux_servers.yml
      vars:
        targets: "{{ groups.get('linux_servers', []) }}"
      notify: "重載 Prometheus 配置"

    - name: "生成目標檔案 (https_services.yml)"
      template:
        src: ../templates/https_services.yml.j2
        dest: /etc/prometheus/targets/https_services.yml
      vars:
        targets: "{{ groups.get('external_services', []) }}"
      notify: "重載 Prometheus 配置"

    - name: "列出已更新的目標"
      debug:
        msg: |
          已更新以下目標：
          - Linux 伺服器：{{ groups.get('linux_servers', []) | length }}
          - 應用伺服器：{{ groups.get('application_servers', []) | length }}
          - 外部服務：{{ groups.get('external_services', []) | length }}

  handlers:
    - name: "重載 Prometheus 配置"
      uri:
        url: "http://localhost:9090/-/reload"
        method: POST
```

### Jinja2 模板：動態目標生成

#### templates/linux_servers.yml.j2

```yaml
# 自動生成的 Prometheus 目標檔案
# 生成時間：{{ ansible_date_time.iso8601 }}
# 不要手動編輯此檔案

{% for group_name in groups.keys() if group_name in ['linux_servers'] %}
- targets:
{% for host in groups.get(group_name, []) %}
    - '{{ hostvars[host].ansible_host | default(hostvars[host].inventory_hostname) }}:{{ hostvars[host].get('node_exporter_port', 9100) }}'
{% endfor %}
  labels:
    job: 'node'
    group: '{{ group_name }}'
    environment: '{{ hostvars[groups.get(group_name, [])[0]].get('env', 'unknown') }}'
    datacenter: '{{ hostvars[groups.get(group_name, [])[0]].get('datacenter', 'unknown') }}'
{% endfor %}
```

---

## 部署驗證與故障排除

### playbooks/health_check.yml

```yaml
---
- name: "監控系統健康檢查"
  hosts: all
  gather_facts: yes

  tasks:

    - name: "[Server] 檢查 Prometheus 健康狀態"
      uri:
        url: "http://localhost:9090/-/healthy"
        method: GET
      register: prometheus_health
      failed_when: prometheus_health.status != 200
      when: inventory_hostname in groups['monitoring_server']

    - name: "[Server] 檢查 Prometheus 目標狀態"
      uri:
        url: "http://localhost:9090/api/v1/targets"
        method: GET
      register: prometheus_targets
      when: inventory_hostname in groups['monitoring_server']

    - name: "[Server] 分析目標狀態"
      set_fact:
        up_targets: "{{ prometheus_targets.json.data.activeTargets | selectattr('health', 'equalto', 'up') | list }}"
        down_targets: "{{ prometheus_targets.json.data.activeTargets | selectattr('health', 'equalto', 'down') | list }}"
      when: inventory_hostname in groups['monitoring_server']

    - name: "[Server] 輸出目標統計"
      debug:
        msg: |
          Prometheus 目標統計：
          UP: {{ up_targets | length }}
          DOWN: {{ down_targets | length }}
          {% if down_targets %}
          故障目標：
          {% for target in down_targets %}
          - {{ target.labels.instance }} ({{ target.labels.job }})
          {% endfor %}
          {% endif %}
      when: inventory_hostname in groups['monitoring_server']

    - name: "[Client] 檢查 node_exporter 狀態"
      uri:
        url: "http://localhost:9100/metrics"
        method: GET
      register: node_exporter_health
      failed_when: node_exporter_health.status != 200
      when: inventory_hostname in groups['linux_servers']

    - name: "[Client] 檢查 node_exporter 收集器"
      command: "curl -s http://localhost:9100/metrics | grep -E '^# HELP' | wc -l"
      register: collector_count
      when: inventory_hostname in groups['linux_servers']

    - name: "[Summary] 生成健康報告"
      set_fact:
        health_report:
          timestamp: "{{ ansible_date_time.iso8601 }}"
          host: "{{ inventory_hostname }}"
          service_status: "{{ prometheus_health.status | default('N/A') }}"
          metrics_count: "{{ collector_count.stdout | default('N/A') }}"
      register: report

    - name: "輸出最終報告"
      debug:
        msg: "{{ health_report }}"

  handlers:
    - name: "重啟 Prometheus"
      docker_compose:
        project_src: /etc/prometheus
        state: restarted
        services:
          - prometheus
```

### playbooks/troubleshoot.yml

```yaml
---
- name: "監控系統故障診斷"
  hosts: "{{ target_host | default('localhost') }}"
  gather_facts: yes

  vars:
    debug_level: 3

  tasks:

    - name: "收集系統資訊"
      set_fact:
        system_info:
          hostname: "{{ inventory_hostname }}"
          ip: "{{ hostvars[inventory_hostname].ansible_host }}"
          os: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
          kernel: "{{ ansible_kernel }}"

    - name: "[Prometheus] 檢查配置語法"
      command: "docker exec prometheus promtool check config /etc/prometheus/prometheus.yml"
      register: prometheus_config_check
      when: inventory_hostname in groups['monitoring_server']
      ignore_errors: yes

    - name: "[Prometheus] 檢查告警規則"
      command: "docker exec prometheus promtool check rules /etc/prometheus/rules/rules.yml"
      register: prometheus_rules_check
      when: inventory_hostname in groups['monitoring_server']
      ignore_errors: yes

    - name: "[Docker] 檢查容器狀態"
      command: "docker ps --all"
      register: docker_containers

    - name: "[Docker] 檢查容器日誌"
      command: "docker logs {{ item }} --tail=50"
      register: container_logs
      loop: "{{ docker_containers.stdout_lines[1:] | map('split', ' ') | map('attribute', '0', default='') | list }}"
      ignore_errors: yes
      when: item | length > 0

    - name: "[Network] 檢查埠口開放情況"
      command: "ss -tuln | grep LISTEN"
      register: listening_ports

    - name: "[Network] 連接性測試"
      command: "nc -zv {{ hostvars[item].ansible_host }} {{ hostvars[item].get('node_exporter_port', 9100) }}"
      register: connectivity_test
      loop: "{{ groups.get('linux_servers', []) }}"
      when: inventory_hostname in groups['monitoring_server']
      ignore_errors: yes

    - name: "輸出診斷報告"
      debug:
        msg: |
          === Prometheus 故障診斷報告 ===
          
          系統資訊：
          {{ system_info | to_nice_yaml }}
          
          配置驗證：
          {% if prometheus_config_check.rc == 0 %}
          ✓ Prometheus 配置正常
          {% else %}
          ✗ Prometheus 配置有誤：
          {{ prometheus_config_check.stderr }}
          {% endif %}
          
          規則驗證：
          {% if prometheus_rules_check.rc == 0 %}
          ✓ 告警規則正常
          {% else %}
          ✗ 告警規則有誤：
          {{ prometheus_rules_check.stderr }}
          {% endif %}
          
          容器狀態：
          {{ docker_containers.stdout }}
          
          開放埠口：
          {{ listening_ports.stdout }}
```

---

## 持續維護與更新

### playbooks/upgrade.yml

```yaml
---
- name: "升級 Prometheus 元件"
  hosts: "{{ target_hosts | default('monitoring_server') }}"
  serial: 1  # 逐台升級，避免全部宕機
  vars_files:
    - ../group_vars/all.yml

  tasks:

    - name: "進行升級前備份"
      block:
        - name: "建立備份目錄"
          file:
            path: "/backup/prometheus-{{ ansible_date_time.date }}"
            state: directory

        - name: "備份 Prometheus 配置"
          copy:
            src: /etc/prometheus
            dest: "/backup/prometheus-{{ ansible_date_time.date }}/prometheus-config"
            remote_src: yes

        - name: "備份 Prometheus 數據"
          command: "tar czf /backup/prometheus-{{ ansible_date_time.date }}/prometheus-data.tar.gz /var/lib/prometheus"

    - name: "停止 Prometheus 容器"
      docker_compose:
        project_src: /etc/prometheus
        state: stopped
        services:
          - prometheus

    - name: "拉取最新鏡像"
      docker_image:
        name: "prom/prometheus:{{ prometheus_version_new }}"
        source: pull

    - name: "更新 docker-compose.yml"
      template:
        src: ../templates/docker-compose.yml.j2
        dest: /etc/prometheus/docker-compose.yml

    - name: "啟動更新後的容器"
      docker_compose:
        project_src: /etc/prometheus
        state: present
        pull: no

    - name: "等待 Prometheus 就緒"
      uri:
        url: "http://localhost:9090/-/healthy"
        method: GET
      register: prometheus_health
      until: prometheus_health.status == 200
      retries: 30
      delay: 2

    - name: "驗證升級"
      uri:
        url: "http://localhost:9090/api/v1/status/buildinfo"
        method: GET
      register: prometheus_info

    - name: "輸出升級結果"
      debug:
        msg: "Prometheus 已升級至版本：{{ prometheus_info.json.data.version }}"

  handlers:
    - name: "回滾升級"
      block:
        - name: "停止容器"
          docker_compose:
            project_src: /etc/prometheus
            state: stopped

        - name: "恢復備份"
          copy:
            src: "/backup/prometheus-{{ ansible_date_time.date }}/prometheus-config/"
            dest: /etc/prometheus
            remote_src: yes

        - name: "啟動容器"
          docker_compose:
            project_src: /etc/prometheus
            state: present
```

---

## 進階教學規劃

### 🚀 推薦的進階學習路徑

基於當前基礎文檔，以下是組織化的進階教學模塊：

---

## **第一階段：容器 & Kubernetes 監控**

### 模塊 1：Docker 容器監控
- **內容**：cAdvisor + Prometheus 監控容器資源
- **難度**：⭐⭐
- **實戰項目**：搭建 Docker Swarm 監控系統
- **成果物**：Docker 監控儀表板、容器告警規則

### 模塊 2：Kubernetes 監控 (K8s + Prometheus)
- **內容**：
  - Kubernetes 原生 metrics-server
  - kube-state-metrics 監控集群狀態
  - kubelet 指標收集
  - Kubernetes RBAC 與 ServiceAccount 配置
- **難度**：⭐⭐⭐
- **工具**：Prometheus Operator、Helm Charts
- **實戰項目**：k8s 生產集群監控部署
- **成果物**：K8s 多租戶監控方案、Pod/Node 告警

### 模塊 3：Istio / 服務網格監控
- **內容**：Envoy 指標、服務流量監控、延遲分析
- **難度**：⭐⭐⭐⭐
- **工具**：Istio Prometheus 集成、Grafana Loki (日誌關聯)

---

## **第二階段：高可用與分佈式架構**

### 模塊 4：Thanos - 長期存儲與全局查詢
- **內容**：
  - Thanos 架構（Sidecar/Query/Store）
  - S3/雲存儲集成
  - 跨集群查詢（多 Prometheus 實例）
  - 數據壓縮與成本優化
- **難度**：⭐⭐⭐
- **實戰項目**：多數據中心 Prometheus 統一管理
- **成果物**：
  - Thanos deployment playbook
  - 成本分析報告（原始 vs Thanos）

### 模塊 5：Prometheus 高可用設計
- **內容**：
  - Remote storage backend (InfluxDB, VictoriaMetrics)
  - Prometheus federation 與副本
  - 一致性哈希負載均衡
  - 故障轉移策略
- **難度**：⭐⭐⭐⭐
- **工具**：VictoriaMetrics、Consul

---

## **第三階段：深度可觀測性**

### 模塊 6：分佈式追蹤 (Distributed Tracing)
- **內容**：
  - Jaeger / Zipkin 架構
  - OpenTelemetry SDK 集成
  - Trace-Log 關聯（Loki + Jaeger）
  - 性能瓶頸分析
- **難度**：⭐⭐⭐⭐
- **語言示例**：Go, Python, Node.js
- **實戰項目**：微服務應用端到端追蹤

### 模塊 7：應用性能監控 (APM)
- **內容**：
  - 應用指標埋點（自訂 exporter）
  - 業務指標 vs 系統指標區別
  - 關鍵用戶體驗指標 (Core Web Vitals)
  - SLI/SLO 定義與監控
- **難度**：⭐⭐⭐
- **工具**：Prometheus client libraries
- **實戰項目**：電商平臺 SLO 監控系統

### 模塊 8：日誌分析與關聯
- **內容**：
  - Loki 日誌聚合（vs ELK）
  - LogQL 查詢語言
  - 日誌與指標關聯分析
  - 異常日誌自動告警
- **難度**：⭐⭐⭐
- **實戰項目**：應用故障根因分析系統

---

## **第四階段：自動化與智能告警**

### 模塊 9：PromQL 進階與告警引擎
- **內容**：
  - PromQL 複雜查詢（聚合函數、時間序列操作）
  - 告警規則最佳實踐
  - 告警漏報 vs 誤報分析
  - Alertmanager 高級路由策略
- **難度**：⭐⭐⭐⭐
- **實戰項目**：
  - 智能告警系統（基於歷史基線）
  - 異常檢測 (Anomaly Detection)

### 模塊 10：機器學習輔助告警
- **內容**：
  - Prometheus 數據 → ML 管道
  - Prophet / ARIMA 預測分析
  - 異常檢測演算法（Isolation Forest）
  - 智能告警優化（降低誤報）
- **難度**：⭐⭐⭐⭐⭐
- **工具**：Python (scikit-learn, Facebook Prophet)
- **實戰項目**：
  - 基于历史数据的SSD寿命预测告警
  - 網路流量異常自動檢測

### 模塊 11：AIOps 集成
- **內容**：
  - Prometheus + PagerDuty / Opsgenie 整合
  - ChatOps 告警（Slack/釘釘/企業微信）
  - 自動化修復 (Self-Healing)
  - 告警聚合與智能路由
- **難度**：⭐⭐⭐
- **實戰項目**：
  - 告警自動降級與恢復系統
  - 釘釘 + Prometheus 告警機器人

---

## **第五階段：成本優化與運維自動化**

### 模塊 12：Prometheus 性能優化
- **內容**：
  - 查詢性能調優 (PromQL 優化)
  - 基數爆炸問題診斷與解決
  - 告警規則評估性能
  - 存儲空間最佳化
- **難度**：⭐⭐⭐⭐
- **成果物**：性能優化清單、基數管理工具

### 模塊 13：成本分析與優化
- **內容**：
  - Prometheus 軟硬體成本計算
  - Thanos + S3 vs 本地存儲成本對比
  - 指標採樣策略（降低成本）
  - 多租戶成本隔離
- **難度**：⭐⭐
- **工具**：PromQL、成本分析腳本

### 模塊 14：運維自動化進階
- **內容**：
  - Ansible 動態 inventory 最佳實踐
  - 基於事件的自動化修復 (Event-driven)
  - Prometheus 與 Ansible Tower 整合
  - GitOps 監控配置管理
- **難度**：⭐⭐⭐
- **工具**：Ansible, Git, ArgoCD
- **實戰項目**：
  - 監控配置即代碼 (IaC)
  - 自動化告警響應工作流

---

## **第六階段：行業應用 & 專業化**

### 模塊 15：行業應用案例
- **15A. 互聯網應用監控**
  - Web 應用、API Gateway 監控
  - CDN 性能監控
  - 實戰項目：雙十一流量監控大盤

- **15B. 金融系統監控**
  - 合規要求與監控日誌
  - 交易系統延遲監控
  - 風控系統告警
  - 實戰項目：金融級監控系統架構

- **15C. IoT / 邊緣計算監控**
  - 邊緣節點的輕量級監控
  - MQTT broker 監控
  - 實戰項目：智能家居監控系統

### 模塊 16：合規與安全監控
- **內容**：
  - Prometheus HTTPS/TLS 配置
  - RBAC 與多租戶隔離
  - 審計日誌（誰修改了監控規則）
  - 數據加密與隐私保護
  - 等保 / SOC 2 合規要求
- **難度**：⭐⭐⭐

---

## **推薦進階內容清單**

### 📋 快速參考表

| 模塊 | 優先級 | 學習時間 | 前置要求 |
|------|--------|---------|--------|
| Docker 監控 | ⭐⭐ | 5 天 | 無 |
| K8s 監控 | ⭐⭐⭐ | 2 週 | Docker + Linux 基礎 |
| Thanos | ⭐⭐⭐ | 1 週 | Prometheus 進階 |
| Jaeger | ⭐⭐⭐ | 1 週 | 微服務架構 |
| PromQL 進階 | ⭐⭐⭐⭐ | 2 週 | SQL 基礎 |
| 機器學習告警 | ⭐⭐⭐⭐⭐ | 4 週 | Python + ML 基礎 |
| AIOps 集成 | ⭐⭐ | 3 天 | Prometheus 基礎 |
| 成本優化 | ⭐⭐ | 2 天 | Prometheus 基礎 |

---

## **進階教學成果物規劃**

每個進階模塊應產出：

### 成果物類型

1. **技術文檔**
   - 完整教學指南（Markdown）
   - API 文檔
   - 故障排除指南

2. **可執行代碼**
   - Ansible playbooks
   - Dockerfile / Helm charts
   - Python 腳本（ML、自動化）
   - Prometheus 規則集

3. **測試用例**
   - 單元測試
   - 集成測試
   - 性能基準測試

4. **演示系統**
   - 完整部署環境
   - 樣本數據集
   - 自動化演示腳本

---

## **預計進階文檔結構**

```
進階教學模塊/
├── 01-Docker-監控/
│   ├── Docker-Monitoring-Guide.md
│   ├── ansible/
│   └── examples/
├── 02-Kubernetes-監控/
│   ├── K8s-Monitoring-Guide.md
│   ├── helm-charts/
│   └── examples/
├── 03-Thanos-分佈式/
│   ├── Thanos-Deployment-Guide.md
│   ├── terraform/
│   └── cost-analysis/
├── 04-分佈式追蹤/
│   ├── Jaeger-Integration-Guide.md
│   └── opentelemetry-examples/
├── 05-PromQL-進階/
│   ├── PromQL-Advanced.md
│   └── query-examples/
├── 06-機器學習告警/
│   ├── ML-Alerting-Guide.md
│   ├── python-scripts/
│   └── models/
├── 07-AIOps集成/
│   ├── AIOps-Integration.md
│   ├── slack-bot/
│   └── webhook-handlers/
└── 08-成本優化/
    ├── Cost-Optimization-Guide.md
    └── analysis-tools/
```

---

## **學習建議**

### 推薦學習順序

```
基礎 (本文檔)
    ↓
[Docker 監控] → [K8s 監控]
    ↓
[Thanos] + [Jaeger] (並行)
    ↓
[PromQL 進階] → [AIOps 整合]
    ↓
[ML 告警] (可選，高級)
    ↓
[成本優化] + [行業應用]
    ↓
自訂應用開發 & 內部平台建設
```

### 每周時間投入估算

- **基礎階段**：4-6 週（本文檔為主）
- **進階階段**（K8s/Thanos）：8-10 週
- **專深階段**（ML/AIOps）：12-16 週
- **總計**：3-5 個月達到企業級應用

---

## **驗證進階知識的實踐項目**

1. **項目 1：多集群監控統一管理**
   - 使用 Thanos 統一 3+ Prometheus 實例
   - 實現跨集群告警

2. **項目 2：K8s + Prometheus 完整平台**
   - 部署 Kubernetes 監控
   - 配置 Prometheus Operator
   - 自訂告警規則

3. **項目 3：微服務可觀測性系統**
   - 集成 Prometheus + Jaeger + Loki
   - 實現三層可觀測性（指標、追蹤、日誌）

4. **項目 4：智能告警系統**
   - 使用 Python + Prophet 預測分析
   - 降低告警誤報率
   - 實現自動化告警降級

5. **項目 5：成本優化案例研究**
   - 對比不同存儲方案成本
   - 提出優化建議與方案
   - 量化投資回報率

---

**此規劃表確保讀者從入門到精通的完整學習路徑，並提供行業實踐指導。**

