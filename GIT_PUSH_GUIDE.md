# Git Push 操作指南

> 如何將 Prometheus 監控教學文檔推送到 Git 倉庫

---

## 🚀 快速上手 (5 分鐘)

### Step 1: 初始化 Git 倉庫 (首次)

```bash
cd /Users/jesse/Documents/Claude/Projects/monitor

# 初始化 Git
git init

# 添加所有文件到暫存區
git add .

# 首次提交
git commit -m "Initial commit: Prometheus 企業級監控教學完整體系

- 4 份核心教學文檔
  ├─ Prometheus_Grafana_監控教學指南.md (Docker Compose 版本)
  ├─ Ansible_Prometheus監控部署指南.md (企業級推薦)
  ├─ 進階教學規劃與路線圖.md (6 大模塊)
  └─ 教學文檔索引與學習指南.md (快速導航)
- README.md (完整項目說明)
- 1000+ 行實戰代碼示例
- 5 個實踐項目詳細規劃"

# 查看提交
git log --oneline
```

---

### Step 2: 連接到遠端倉庫

```bash
# 添加遠端倉庫
git remote add origin https://github.com/YOUR_USERNAME/prometheus-monitoring-guide.git

# 驗證連接
git remote -v
```

---

### Step 3: 推送到遠端

```bash
# 首次推送 (設置 upstream)
git push -u origin main
# 或
git push -u origin master

# 後續推送 (簡化命令)
git push
```

---

## 📋 完整操作步驟

### 場景 A: 全新倉庫 (推薦)

```bash
# 1. 在 GitHub/GitLab 創建空倉庫 (不包含 README)
#    https://github.com/new

# 2. 進入本地目錄
cd /Users/jesse/Documents/Claude/Projects/monitor

# 3. 初始化 Git (如果還沒初始化)
git init

# 4. 創建 .gitignore 文件 (排除不必要的文件)
cat > .gitignore << 'EOF'
# Ansible
*.retry
.ansible/roles/*/files/*
!.ansible/roles/*/files/.gitkeep

# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.venv/
venv/

# Prometheus
prometheus_data/
alertmanager_data/
grafana_data/

# 編輯器
.vscode/
.idea/
*.swp
*.swo

# macOS
.DS_Store

# 其他
*.log
.env
secrets.yml
EOF

# 5. 添加 .gitignore
git add .gitignore

# 6. 提交所有文件
git add .
git commit -m "Initial commit: Prometheus 企業級監控教學體系

新增文檔：
- Prometheus_Grafana_監控教學指南.md (基礎版本)
- Ansible_Prometheus監控部署指南.md (企業推薦版)
- 進階教學規劃與路線圖.md (6 大進階模塊)
- 教學文檔索引與學習指南.md (快速導航)
- README.md (項目總覽)

核心特色：
✓ 1000+ 行 Prometheus 配置示例
✓ 完整的 Ansible playbooks 與 roles
✓ 5 個生產級實踐項目
✓ 從基礎到進階的完整學習路徑
✓ 中文文檔，實戰導向

作者：DevOps 團隊
時間：2026-04-30"

# 7. 連接遠端倉庫
git remote add origin https://github.com/YOUR_USERNAME/prometheus-monitoring-guide.git

# 8. 推送到遠端
git branch -M main
git push -u origin main
```

---

### 場景 B: 已有倉庫

```bash
# 1. 進入目錄
cd /Users/jesse/Documents/Claude/Projects/monitor

# 2. 檢查 Git 狀態
git status

# 3. 添加新文件
git add README.md
git add '*.md'

# 4. 提交
git commit -m "Add: Prometheus 教學文檔完整體系"

# 5. 推送
git push
```

---

### 場景 C: 已有 Git 倉庫但需要更新

```bash
# 1. 檢查當前狀態
git status

# 2. 查看未提交的變更
git diff

# 3. 如果有衝突，查看衝突文件
git diff --name-only --diff-filter=U

# 4. 拉取最新版本
git pull origin main

# 5. 解決衝突後提交
git add .
git commit -m "Merge: 合併 Prometheus 教學文檔更新"

# 6. 推送
git push origin main
```

---

## 📝 提交信息示例

### 好的提交信息格式

```
feat: 新增 Prometheus 企業級監控教學體系

- 創建 4 份核心教學文檔
  ├─ Prometheus_Grafana_監控教學指南.md
  ├─ Ansible_Prometheus監控部署指南.md
  ├─ 進階教學規劃與路線圖.md
  └─ 教學文檔索引與學習指南.md
- 提供 1000+ 行實戰配置示例
- 規劃 6 個進階模塊與 5 個實踐項目
- 編寫完整的 README.md 與快速開始指南

Relates to: #123
```

---

## 🔑 常用 Git 命令

### 查看與檢查

```bash
# 查看當前狀態
git status

# 查看提交歷史
git log --oneline -10

# 查看具體變更
git diff
git diff --staged

# 查看分支
git branch -a
```

---

### 添加與提交

```bash
# 添加所有變更
git add .

# 添加指定文件
git add filename.md
git add '*.md'

# 查看分階段的變更
git add -p

# 提交
git commit -m "提交信息"

# 修改上個提交 (未推送時)
git commit --amend
```

---

### 推送與拉取

```bash
# 推送到遠端
git push origin main

# 設置上游分支後簡化推送
git push -u origin main
git push  # 後續可直接用這個

# 拉取最新版本
git pull origin main

# 只拉取但不合併
git fetch origin
```

---

### 分支操作

```bash
# 創建新分支 (用於新功能/文檔)
git checkout -b feature/docker-monitoring

# 推送新分支到遠端
git push -u origin feature/docker-monitoring

# 切換分支
git checkout main

# 刪除本地分支
git branch -d feature/docker-monitoring

# 刪除遠端分支
git push origin --delete feature/docker-monitoring
```

---

## ✅ 推送前檢查清單

在推送前，確保：

```bash
# 1. 檢查所有文件都已添加
git status
# 應該顯示：On branch main, nothing to commit

# 2. 檢查提交信息清晰
git log --oneline -5

# 3. 檢查沒有敏感信息
grep -r "password\|secret\|token" .

# 4. 驗證遠端倉庫配置
git remote -v

# 5. 確認分支名稱正確
git branch

# 6. 如果有疑慮，先做本地測試
git log --oneline origin/main..main
```

---

## 🔒 處理敏感信息

### 如果不小心提交了密碼/Token

```bash
# 1. 停止推送！

# 2. 查看提交中的敏感信息
git log -p | grep -i password

# 3. 移除敏感信息 (使用 BFG Repo-Cleaner)
brew install bfg  # macOS

# 4. 清理歷史
bfg --replace-text passwords.txt

# 5. 清理本地倉庫
git reflog expire --expire=now --all && git gc --prune=now --aggressive

# 6. 強制推送 (謹慎使用!)
git push origin --force-with-lease
```

### 預防措施

```bash
# 創建 .gitignore 排除敏感文件
cat > .gitignore << 'EOF'
*.key
*.pem
secrets.yml
.env
.env.local
vault/
credentials/
EOF

git add .gitignore
git commit -m "chore: Add .gitignore to prevent committing secrets"
```

---

## 📊 推送工作流程圖

```
┌─────────────────────────────────────┐
│   本地文件系統                      │
│  /Users/jesse/Documents/...         │
└────────────┬────────────────────────┘
             │ git add
             ▼
┌─────────────────────────────────────┐
│   暫存區 (Staging Area)              │
│  已選擇的文件等待提交                │
└────────────┬────────────────────────┘
             │ git commit
             ▼
┌─────────────────────────────────────┐
│   本地 Git 倉庫 (.git)               │
│  包含完整的提交歷史                  │
└────────────┬────────────────────────┘
             │ git push
             ▼
┌─────────────────────────────────────┐
│   遠端倉庫 (GitHub/GitLab)           │
│  共享版本控制，團隊協作              │
└─────────────────────────────────────┘
```

---

## 🎯 推送成功標誌

推送完成後，驗證成功：

```bash
# 1. 檢查推送返回信息
# 應該看到：
#   [new branch]      main -> main
#   Branch 'main' set up to track 'origin/main'.

# 2. 驗證遠端倉庫
git branch -r

# 3. 查看遠端日誌
git log origin/main --oneline -5

# 4. 在 GitHub/GitLab Web UI 上查看
# https://github.com/YOUR_USERNAME/prometheus-monitoring-guide
```

---

## 🔄 持續更新工作流

推送後，後續更新文檔時：

```bash
# 1. 編輯文件
vim Prometheus_Grafana_監控教學指南.md

# 2. 檢查變更
git status

# 3. 添加變更
git add Prometheus_Grafana_監控教學指南.md

# 4. 提交
git commit -m "docs: 更新 Prometheus 基礎教學 - 新增 SSD 監控章節"

# 5. 推送
git push
```

---

## 🚨 常見問題

### Q1: Permission denied (publickey)

```bash
# 解決方案：配置 SSH 金鑰
# 1. 生成金鑰
ssh-keygen -t ed25519 -C "your_email@example.com"

# 2. 添加到 SSH agent
ssh-add ~/.ssh/id_ed25519

# 3. 複製公鑰到 GitHub Settings → SSH Keys
cat ~/.ssh/id_ed25519.pub | pbcopy

# 4. 測試連接
ssh -T git@github.com
```

---

### Q2: fatal: not a git repository

```bash
# 解決方案：初始化 Git
cd /Users/jesse/Documents/Claude/Projects/monitor
git init
git add .
git commit -m "Initial commit"
```

---

### Q3: fatal: 'origin' does not appear to be a 'git' repository

```bash
# 解決方案：添加遠端倉庫
git remote add origin https://github.com/YOUR_USERNAME/prometheus-monitoring-guide.git
git remote -v  # 驗證
git push -u origin main
```

---

### Q4: 想撤銷最後一次提交 (未推送)

```bash
# 保留文件變更，撤銷提交
git reset --soft HEAD~1

# 完全撤銷 (不保留文件變更)
git reset --hard HEAD~1
```

---

### Q5: 分支衝突

```bash
# 1. 拉取最新版本
git pull origin main

# 2. 手動解決衝突 (編輯文件，移除衝突標記)
# <<<<<<< HEAD
# your change
# =======
# their change
# >>>>>>> origin/main

# 3. 標記為已解決
git add conflicted_file.md

# 4. 完成合併
git commit -m "Merge: 解決分支衝突"

# 5. 推送
git push
```

---

## 📚 推薦資源

- [Git 官方文檔](https://git-scm.com/doc)
- [GitHub 幫助](https://docs.github.com/)
- [Pro Git 書籍](https://git-scm.com/book/en/v2)
- [Git Cheat Sheet](https://education.github.com/git-cheat-sheet-education.pdf)

---

## ✨ 推送後的建議

推送成功後：

1. **保持同步**
   ```bash
   # 定期拉取最新版本
   git pull origin main
   ```

2. **建立分支工作流** (用於協作)
   ```bash
   # 新功能/改進
   git checkout -b feature/new-module
   # ... 編輯文件 ...
   git push -u origin feature/new-module
   # 在 GitHub 上提交 Pull Request
   ```

3. **版本標籤** (里程碑記錄)
   ```bash
   # 創建版本標籤
   git tag -a v1.0 -m "Release version 1.0"
   git push origin v1.0
   ```

4. **自動化** (設置 CI/CD)
   - GitHub Actions 自動驗證文檔
   - 自動生成目錄
   - 自動檢查鏈接

---

**現在就推送你的 Prometheus 教學文檔到 Git 吧！** 🚀

