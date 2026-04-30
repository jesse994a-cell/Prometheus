# Prometheus 企業級監控自動化部署指南

> **完整的 Prometheus + Grafana + Alertmanager + Loki 監控系統教學體系**
>
> 包含基礎部署、Ansible 自動化、K8s 集成、高可用架構、可觀測性三層模型及 AIOps 智能告警的完整教程。

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Prometheus](https://img.shields.io/badge/Prometheus-Latest-orange.svg)](https://prometheus.io/)
[![Ansible](https://img.shields.io/badge/Ansible-2.10%2B-red.svg)](https://www.ansible.com/)
![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen.svg)

---

## 📋 項目概述

本項目提供 **系統化的 Prometheus 監控教學體系**，包含：

- ✅ **基礎部署**：Docker Compose 快速上手
- ✅ **企業級方案**：Ansible 自動化部署（推薦）
- ✅ **進階教學**：6 大模塊 + 5 個實踐項目
- ✅ **完整文檔**：1000+ 行實戰代碼示例
- ✅ **故障排除**：生產環境常見問題診斷

**目標讀者**：

- 🎯 運維工程師：快速建立監控系統
- 🎯 DevOps 工程師：企業級可觀測性架構
- 🎯 系統架構師：設計監控戰略
- 🎯 應用開發者：實現應用埋點

---

## 📚 文檔體系結構

```
monitoring-guide/
│
├── README.md (本文件)
│
├── 📖 核心文檔
│   ├── Prometheus_Grafana_監控教學指南.md
│   │   └─ Docker Compose 版本，適合快速開始
│   │
│   ├── Ansible_Prometheus監控部署指南.md ⭐ (推薦)
│   │   └─ 企業級自動化部署，支持多主機
│   │
│   ├── 進階教學規劃與路線圖.md
│   │   └─ 6 個進階模塊 + 5 個實踐項目
│   │
│   └── 教學文檔索引與學習指南.md
│       └─ 快速導航與學習路線建議
│
├── 📁 部署文件 (Ansible)
│   ├── inventory.ini
│   ├── ansible.cfg
│   ├── playbooks/
│   ├── roles/
│   └── templates/
│
└── 📁 其他資源
    ├── examples/      (Prometheus/Grafana 配置示例)
    ├── scripts/       (輔助腳本)
    └── docs/          (補充文檔)
```

---

## 🚀 快速開始

### 5 分鐘了解全貌

```bash
# 1. 克隆本倉庫
git clone <repo-url>
cd prometheus-monitoring-guide

# 2. 先讀快速導航
cat 教學文檔索引與學習指南.md | head -100

# 3. 選擇適合的版本
# 選項 A: Docker Compose (簡單，適合單機)
cat Prometheus_Grafana_監控教學指南.md | grep -A 20 "docker-compose"

# 選項 B: Ansible (推薦，適合多主機)
cat Ansible_Prometheus監控部署指南.md | grep -A 30 "docker-compose.yml"
```

### 30 分鐘完成第一次部署 (Docker Compose)

```bash
# 複製 Docker Compose 配置並啟動
mkdir -p monitoring && cd monitoring

# 創建配置目錄結構
mkdir -p {prometheus/targets,alertmanager,grafana/provisioning/{dashboards,datasources},loki}

# 參考 Prometheus_Grafana_監控教學指南.md 的 "Docker Compose 完整部署" 章節
# 複製相應的 docker-compose.yml 和配置文件

docker-compose up -d

# 訪問服務
# Prometheus: http://localhost:9090
# Grafana: http://localhost:3000 (admin/admin)
# Alertmanager: http://localhost:9093
```

### 2-3 小時完成 Ansible 部署 (推薦)

```bash
# 準備環境
git clone <repo-url>
cd prometheus-monitoring-guide

# 編輯 inventory.ini，配置你的主機列表
vim inventory.ini

# 驗證 Ansible 連接
ansible all -i inventory.ini -m ping

# 執行部署 playbook
ansible-playbook playbooks/deploy_all.yml

# 驗證部署
ansible-playbook playbooks/health_check.yml
```

---

## 📖 文檔詳解

### 1️⃣ **Prometheus_Grafana_監控教學指南.md** (基礎版)

**何時使用：**
- ✅ Docker 單機環境
- ✅ 快速原型驗證
- ✅ 學習 Prometheus 概念

**包含內容：**
- 架構概覽與對比分析
- Docker Compose 完整配置
- Prometheus/Alert/Grafana 詳細配置
- 18+ 告警規則實例
- 故障排除指南
- SSD 和 Inode 監控進階技巧

**預期投入：** 4-6 週

---

### 2️⃣ **Ansible_Prometheus監控部署指南.md** ⭐ (推薦)

**何時使用：**
- ✅ 多主機環境
- ✅ 團隊協作與版本控制
- ✅ 生產環境部署
- ✅ 持續優化與升級

**包含內容：**
- 企業級架構設計
- Ansible 角色與 playbooks
- 服務發現與動態配置
- Jinja2 模板示例
- Server/Client 角色詳解
- 部署驗證與健康檢查
- 故障診斷工具

**核心優勢：**
```
✓ 支持 10+ 台主機並行部署
✓ 版本控制友好 (Git 管理所有配置)
✓ 幂等性 (重複執行同樣結果)
✓ 快速升級與回滾
✓ 易於團隊協作
```

**預期投入：** 2-4 週

---

### 3️⃣ **進階教學規劃與路線圖.md**

**何時使用：**
- ✅ 掌握基礎後深化
- ✅ 設計企業級可觀測性平台
- ✅ 團隊進階培訓

**6 大進階模塊：**

| 模塊 | 難度 | 時間 | 目標 |
|------|------|------|------|
| Docker 容器監控 | ⭐⭐ | 5 天 | cAdvisor 監控 |
| Kubernetes 監控 | ⭐⭐⭐ | 2-3 週 | Prometheus Operator |
| Thanos 高可用 | ⭐⭐⭐ | 1-2 週 | 多集群 + 長期存儲 |
| Jaeger 分佈式追蹤 | ⭐⭐⭐⭐ | 1-2 週 | 應用埋點 |
| PromQL 進階 | ⭐⭐⭐⭐ | 2-3 週 | 異常檢測 + ML |
| AIOps 自動化 | ⭐⭐ | 3-5 天 | ChatOps + 修復 |

**5 大實踐項目：**
1. Docker Swarm 監控 (3 週)
2. K8s 多集群統一管理 (4 週)
3. 微服務完整可觀測性 (4 週)
4. 智能異常檢測系統 (3 週)
5. 企業 AIOps 平台 (4 週)

**預期投入：** 3-5 個月 (進階全套)

---

### 4️⃣ **教學文檔索引與學習指南.md**

**何時使用：**
- ✅ 快速定位所需內容
- ✅ 規劃個人學習路線
- ✅ 組織團隊培訓

**包含內容：**
- 按角色的推薦路線
- 時間預算導向的學習計劃
- 功能快速查找表
- 進度追蹤清單
- 常見問題解答
- 學習小組活動建議

---

## 🎯 根據角色快速開始

### 👨‍💼 **運維經理 / 架構師**

```bash
# 閱讀時間：2 小時
# 決策要點：架構、成本、ROI

推薦閱讀順序：
1. 本 README.md
2. Ansible_Prometheus監控部署指南.md → 架構設計章節
3. 進階教學規劃與路線圖.md → 成本分析部分

核心指標：
- 部署時間：2-4 週
- 人力投入：2-3 人
- 成本節省：年度 $5600+ (使用 Thanos)
```

---

### 👨‍💻 **系統管理員 / SRE (新手)**

```bash
# 學習時間：4-6 週
# 目標：獨立部署與維護

推薦學習路線：
Week 1-2: 讀完 Ansible_Prometheus監控部署指南.md
Week 3-4: 在 3+ 台測試主機上部署
Week 5-6: 設置告警規則、優化配置

驗收標準：
✓ 無文檔即可部署完整系統
✓ 能快速定位故障
✓ 告警誤報率 < 5%
```

---

### 🚀 **高級 DevOps / 架構師**

```bash
# 學習時間：3-5 個月
# 目標：企業級可觀測性平台

推薦學習路線：
Month 1: 基礎 (Ansible 自動化)
Month 2: Docker + K8s 監控
Month 3: Thanos 高可用
Month 4: Jaeger + 可觀測性
Month 5: PromQL + 異常檢測

完成 5 個實踐項目，達到企業級應用水平
```

---

## 📊 學習時間線

```
Week 1-4:    基礎部署 (Ansible)
            ↓
Week 5-8:    Docker + Kubernetes 監控
            ↓
Week 9-12:   Thanos 高可用架構
            ↓
Week 13-16:  Jaeger + 可觀測性三層
            ↓
Week 17-20:  PromQL 進階 + 異常檢測
            ↓
Week 21+:    AIOps + 企業應用
```

---

## 📁 目錄結構 (完整版)

```
prometheus-monitoring-guide/
│
├── README.md                                    # 本文件
├── LICENSE                                      # MIT License
│
├── 📖 教學文檔
│   ├── Prometheus_Grafana_監控教學指南.md      # Docker Compose 版本
│   ├── Ansible_Prometheus監控部署指南.md       # Ansible 企業級版本 ⭐
│   ├── 進階教學規劃與路線圖.md                 # 6 個進階模塊
│   └── 教學文檔索引與學習指南.md              # 快速導航
│
├── 📁 ansible/                                 # Ansible 自動化部署
│   ├── inventory.ini                          # Inventory 配置示例
│   ├── inventory/                             # 動態 inventory (可選)
│   │   └── aws_ec2.yml
│   ├── playbooks/                             # Playbook 腳本
│   │   ├── deploy_monitoring_server.yml
│   │   ├── deploy_monitoring_clients.yml
│   │   ├── deploy_all.yml
│   │   ├── health_check.yml
│   │   ├── troubleshoot.yml
│   │   └── upgrade.yml
│   ├── roles/                                 # Ansible 角色
│   │   ├── prometheus_server/
│   │   ├── node_exporter/
│   │   ├── blackbox_exporter/
│   │   ├── grafana_server/
│   │   ├── alertmanager_server/
│   │   ├── loki_server/
│   │   └── common/
│   ├── group_vars/                            # 分組變數
│   │   ├── all.yml
│   │   ├── monitoring_server.yml
│   │   └── linux_servers.yml
│   ├── host_vars/                             # 主機特定變數
│   └── ansible.cfg                            # Ansible 配置
│
├── 📁 examples/                                # 配置示例
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   ├── alert_rules.yml
│   │   └── targets/
│   ├── alertmanager/
│   │   └── config.yml
│   ├── grafana/
│   │   ├── provisioning/
│   │   └── dashboards/
│   └── docker-compose/
│       └── docker-compose.yml
│
├── 📁 scripts/                                 # 輔助腳本
│   ├── prometheus_backup.sh
│   ├── ssd_health_monitor.sh
│   ├── cluster_health_check.sh
│   └── cost_analyzer.py
│
└── 📁 docs/                                   # 補充文檔
    ├── FAQ.md                                 # 常見問題
    ├── TROUBLESHOOTING.md                     # 故障排除
    ├── BEST_PRACTICES.md                      # 最佳實踐
    └── MIGRATION_GUIDE.md                     # 遷移指南 (Nagios → Prometheus)
```

---

## 🔧 核心特性

### 功能涵蓋

```
✅ 指標收集 (Prometheus)
   ├─ node_exporter (Linux 主機)
   ├─ blackbox_exporter (外部服務)
   ├─ 應用自訂 exporter
   └─ Docker/Kubernetes 監控

✅ 告警管理 (Alertmanager)
   ├─ 告警分組與去重
   ├─ 告警抑制與升級
   ├─ 多渠道通知 (郵件/Slack/Telegram/LINE)
   └─ 動態路由規則

✅ 可視化 (Grafana)
   ├─ 官方推薦儀表板
   ├─ 自訂儀表板設計
   └─ 告警狀態面板

✅ 日誌聚合 (Loki)
   ├─ 日誌收集與查詢
   └─ 日誌與指標關聯

✅ 自動化 (Ansible)
   ├─ 一鍵部署
   ├─ 動態配置管理
   ├─ 持續升級與維護
   └─ 故障診斷與修復
```

### 部署能力

| 能力 | Docker Compose | Ansible |
|------|---------------|---------|
| **單機部署** | ✅ | ✅ |
| **多主機部署** | ❌ | ✅✅✅ |
| **並行執行** | ❌ | ✅ |
| **版本控制** | ❌ | ✅✅ |
| **自動回滾** | ❌ | ✅ |
| **團隊協作** | ❌ | ✅✅ |
| **實施難度** | ⭐ | ⭐⭐ |

---

## 💻 系統要求

### 最低配置

```yaml
Prometheus Server:
  CPU: 2 核
  記憶體: 4 GB
  磁碟: 50 GB SSD
  網路: 100 Mbps

Monitoring Client:
  CPU: 0.5 核
  記憶體: 512 MB
  磁碟: 2 GB
  網路: 10 Mbps
```

### 軟體依賴

```bash
# 基礎
- Docker 20.10+
- Docker Compose 2.0+
- Linux (Ubuntu 20.04+ / CentOS 7+)

# Ansible 部署
- Ansible 2.10+
- Python 3.8+
- SSH 訪問權限

# 進階
- Kubernetes 1.19+ (K8s 監控)
- Git (版本控制)
- Python 3.8+ (ML 異常檢測)
```

---

## 🚀 部署步驟總結

### 方案 A: Docker Compose (快速開始)

```bash
# 1. 準備環境
mkdir -p monitoring
cd monitoring

# 2. 複製配置 (參考 Prometheus_Grafana_監控教學指南.md)
cp docker-compose.yml .
mkdir -p {prometheus/targets,alertmanager,grafana/provisioning,loki}

# 3. 啟動服務
docker-compose up -d

# 4. 驗證 (訪問 http://localhost:9090)
curl http://localhost:9090/-/healthy

時間投入: 30 分鐘
適用場景: 測試、學習、小規模環境
```

---

### 方案 B: Ansible (推薦生產)

```bash
# 1. 準備 Ansible 環境
git clone <repo-url>
cd prometheus-monitoring-guide/ansible

# 2. 編輯 inventory
vim inventory.ini
# 配置你的服務器列表

# 3. 驗證連接
ansible all -i inventory.ini -m ping

# 4. 執行部署
ansible-playbook playbooks/deploy_all.yml

# 5. 健康檢查
ansible-playbook playbooks/health_check.yml

時間投入: 2-3 小時
適用場景: 生產環境、多主機、團隊部署
```

---

## 📖 進階學習

完成基礎部分後，推薦按以下順序深入：

```
1. Docker 容器監控           (5 天)
   ↓
2. Kubernetes 監控            (2-3 週)
   ↓
3. Thanos 多集群高可用        (1-2 週)
   ↓
4. Jaeger 分佈式追蹤          (1-2 週)
   ↓
5. PromQL 進階 + 異常檢測     (2-3 週)
   ↓
6. AIOps 智能告警 + 自動化    (3-5 天)
```

詳見：**進階教學規劃與路線圖.md**

---

## 🔍 快速查找

### 我想... → 讀哪個文件

| 需求 | 文件位置 |
|------|---------|
| **快速了解** | README.md (本文件) |
| **選擇部署方案** | 教學文檔索引與學習指南.md → 架構對比 |
| **Docker 快速開始** | Prometheus_Grafana_監控教學指南.md → Docker Compose |
| **企業級部署** | Ansible_Prometheus監控部署指南.md → 完整內容 |
| **編寫告警規則** | Ansible_Prometheus監控部署指南.md → 告警規則實戰 |
| **故障排除** | Prometheus_Grafana_監控教學指南.md → 故障排除章節 |
| **K8s 監控** | 進階教學規劃與路線圖.md → Module 2 |
| **成本優化** | 進階教學規劃與路線圖.md → Thanos 成本分析 |
| **異常檢測** | 進階教學規劃與路線圖.md → Module 5 |
| **自動化告警** | 進階教學規劃與路線圖.md → Module 6 |

---

## 🎓 學習建議

### 推薦每周安排

```
新手 (Week 1-4):
  - Monday: 理論學習 (2 小時)
  - Tuesday-Thursday: 動手實驗 (4 小時/天)
  - Friday: 總結與討論 (2 小時)
  - 周末: 鞏固知識 (4 小時)
  → 周投入: ~22 小時

進階 (Week 5+):
  - 減少理論，增加實踐項目
  - 閱讀源代碼與社區文章
  - 參與開源貢獻
```

### 團隊學習活動

```
✅ 周會 (1 小時)
   - 進度分享
   - 問題討論
   - 下周計劃

✅ 月度分享會 (2 小時)
   - 深度技術講解
   - 案例研究
   - 最佳實踐分享

✅ 線上討論小組
   - 隨時提問
   - 經驗分享
   - 資源推薦
```

---

## 📞 獲取幫助

### 外部資源

- [Prometheus 官方文檔](https://prometheus.io/docs/)
- [Prometheus 社區論壇](https://groups.google.com/forum/#!forum/prometheus-users)
- [Ansible 官方文檔](https://docs.ansible.com/)
- [Stack Overflow - Prometheus 標籤](https://stackoverflow.com/questions/tagged/prometheus)

---

## 📄 許可證

本項目採用 **MIT License**，詳見 LICENSE 文件。

---

## 👥 作者與致謝

**編寫**：DevOps 團隊  
**最後更新**：2026-04-30  
**貢獻者**：感謝所有提供反饋與改進建議的同事

---

## 🌟 重要提示

### 推薦使用

```
✅ 推薦：Ansible_Prometheus監控部署指南.md
   - 企業級架構設計
   - 支持多主機自動化部署
   - 版本控制友好
   - 易於團隊協作與維護

⚠️ Docker Compose 版本
   - 適合單機學習與測試
   - 不推薦生產環境直接使用
   - 可作為參考與理解基礎概念
```

### 部署前檢查清單

- [ ] 已仔細閱讀相關文檔
- [ ] 已在測試環境驗證配置
- [ ] 已備份重要數據
- [ ] 已測試 SSH 無密碼連接
- [ ] 已通知相關人員計劃維護時間
- [ ] 已準備回滾方案

---

## 📈 預期效果

成功部署後，你將獲得：

```
✓ 完整的監控覆蓋
  - 系統資源監控
  - 應用性能監控
  - 外部服務監控
  - 日誌聚合分析

✓ 高效的告警機制
  - 智能告警分組
  - 多渠道通知
  - 低誤報率 (< 5%)
  - 快速故障定位 (< 5 分鐘)

✓ 企業級可靠性
  - 99%+ 可用性
  - 自動化故障轉移
  - 長期數據保留 (使用 Thanos)
  - 成本優化 (年省 $5600+)

✓ 團隊技能提升
  - 掌握現代監控技術
  - 建立最佳實踐
  - 能力模型完善
```

---

## 🎯 快速開始檢查清單

```
□ 已閱讀本 README.md
□ 已了解 4 份核心文檔的用途
□ 已根據角色選擇學習路線
□ 已準備合適的硬體環境
□ 已安裝必要的軟體依賴
□ 已測試 SSH 連接 (Ansible)
□ 已準備測試環境進行試驗
□ 已組建學習小組或找到技術顧問
□ 已制定 4-6 週的學習計劃
□ 已準備好開始第一次部署 ✨
```

---

**現在就開始吧！** 🚀

1. 🔗 **根據角色選擇合適的文檔**（見上方"根據角色快速開始"）
2. 📖 **按推薦順序閱讀文檔**（見"快速開始"）
3. 💻 **在測試環境進行部署**（2-3 天）
4. 🎓 **根據進度進入進階學習**（3-5 個月）

有任何問題，歡迎隨時提問！

---

**文檔版本**：v1.0  
**最後更新**：2026-04-30  
**維護者**：DevOps 團隊  
**狀態**：✅ 生產就緒

