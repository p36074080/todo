# Couple Shared Memory System

情侶共享記憶系統 - 一個用於管理待辦事項、採買清單和行事曆的共享工具。

## 專案概述

這個專案包含三個主要部分：
1. **XR Display Dashboard** - 橫向顯示的儀表板，用於常駐顯示待辦事項和行事曆
2. **Editor App** - 手機編輯應用，用於建立和管理項目
3. **Google Apps Script Backend** - 後端 API，連接 Google Sheet 和 Google Calendar

## 專案結構

```
todo/
├── xr-display/
│   └── index.html          # XR Display 應用（橫向顯示）
├── editor/
│   └── index.html          # Editor 應用（手機編輯）
├── GOOGLE_SHEET_SETUP.md   # Google Sheet 設定指南
└── README.md               # 本文件
```

## 快速開始

### 1. 設定 Google Sheet 和 Apps Script

請參考 [GOOGLE_SHEET_SETUP.md](./GOOGLE_SHEET_SETUP.md) 完成以下步驟：

1. 建立 Google Sheet 並設定欄位
2. 建立 Google Apps Script 專案並貼上程式碼
3. 部署為 Web App 並取得 API URL

### 2. 設定前端應用

#### XR Display App

1. 開啟 `xr-display/index.html`
2. 找到第 175 行的 `API_BASE_URL` 變數
3. 將 `YOUR_GOOGLE_APPS_SCRIPT_URL_HERE` 替換為你的 Google Apps Script Web App URL

```javascript
const API_BASE_URL = 'https://script.google.com/macros/s/YOUR_DEPLOYMENT_ID/exec';
```

#### Editor App

1. 開啟 `editor/index.html`
2. 找到第 195 行的 `API_BASE_URL` 變數
3. 將 `YOUR_GOOGLE_APPS_SCRIPT_URL_HERE` 替換為你的 Google Apps Script Web App URL

```javascript
const API_BASE_URL = 'https://script.google.com/macros/s/YOUR_DEPLOYMENT_ID/exec';
```

### 3. 使用應用

#### XR Display App

- **用途**：橫向顯示的儀表板，適合放在 XR 裝置或平板電腦上
- **功能**：
  - 顯示待辦事項和採買清單（僅 active 狀態）
  - 顯示未來 3 天的 Google Calendar 事件
  - 自動每 5-10 分鐘刷新
  - 使用 localStorage 快取，離線時顯示快取資料

#### Editor App

- **用途**：手機編輯應用，用於建立和管理項目
- **功能**：
  - **代辦事項** 和 **採買清單** 標籤：顯示 active 狀態的項目
  - **已完成** 標籤：顯示 done 狀態的項目
  - 點擊 checkbox：切換項目狀態（active ↔ done）
  - 左滑項目：顯示刪除按鈕，將項目設為 archived（邏輯刪除）
  - 點擊「＋ 新增」：開啟底部表單新增項目

## 功能說明

### 資料模型

每個項目包含以下欄位：

- `id`: 唯一識別碼（格式：`it_1700000123456`）
- `text`: 項目內容
- `type`: 類型（`todo` 或 `shopping`）
- `subtype`: 子類型（僅 todo 使用，`task` 或 `discuss`）
- `priority`: 優先級（`high`、`mid`、`low`）
- `status`: 狀態（`active`、`done`、`archived`）
- `createdAt`: 建立時間（epoch milliseconds）
- `updatedAt`: 更新時間（epoch milliseconds）
- `dueDate`: 到期日（選填，格式：YYYY-MM-DD）
- `createdBy`: 建立者（選填）
- `note`: 備註（選填）

### 狀態語意

- **active**: 在 XR Display 和 Editor 的 active 標籤中顯示
- **done**: 僅在 Editor 的「已完成」標籤中顯示
- **archived**: 隱藏（邏輯刪除，不會真正刪除資料）

### 排序規則

- **Active 列表**：優先級升序（high → mid → low），然後 updatedAt 降序
- **Done 列表**：updatedAt 降序

## API 端點

詳細的 API 說明請參考 [GOOGLE_SHEET_SETUP.md](./GOOGLE_SHEET_SETUP.md#4-api-端點使用說明)。

### GET /items
取得項目列表

### POST /items
建立新項目

### PATCH /items/:id
更新項目

### PATCH /items/:id/archive
封存項目

### GET /calendar
取得 Google Calendar 事件（未來 N 天）

## 開發說明

### 本地開發

1. 使用本地伺服器開啟 HTML 檔案（避免 CORS 問題）：
   ```bash
   # 使用 Python
   python3 -m http.server 8000
   
   # 或使用 Node.js
   npx http-server
   ```

2. 在瀏覽器中開啟：
   - XR Display: `http://localhost:8000/xr-display/index.html`
   - Editor: `http://localhost:8000/editor/index.html`

### PWA 支援

兩個應用都已設定基本的 PWA meta tags，可以：
- 安裝到主畫面
- 離線時顯示快取的資料
- 自動刷新資料

### 快取機制

- **XR Display**: 快取 5 分鐘
- **Editor**: 快取 2 分鐘
- 快取資料儲存在 `localStorage` 中

## 部署

### 靜態網站部署

可以將整個專案部署到任何靜態網站託管服務：

- **GitHub Pages**
- **Netlify**
- **Vercel**
- **Firebase Hosting**
- 或其他靜態網站服務

### 部署步驟

1. 將專案推送到 Git 倉庫
2. 在託管服務中連接倉庫
3. 設定建置目錄為專案根目錄
4. 確保 `xr-display/` 和 `editor/` 目錄可正常存取

## 疑難排解

### API 連線失敗

1. 確認 `API_BASE_URL` 已正確設定
2. 確認 Google Apps Script 已正確部署
3. 檢查瀏覽器 Console 是否有錯誤訊息
4. 確認 Google Apps Script 的權限設定為「所有人」或已正確授權

### CORS 錯誤

Google Apps Script 的程式碼已包含 CORS headers，如果仍有問題：
1. 確認 Apps Script 程式碼已正確部署
2. 檢查瀏覽器 Console 的錯誤訊息

### 資料不同步

1. 清除瀏覽器的 localStorage
2. 重新載入頁面
3. 確認 API 回應正常（檢查 Network 標籤）

## 未來改進（M4+）

- [ ] 加入認證 token 機制
- [ ] 優化快取策略
- [ ] 加入離線編輯佇列
- [ ] 加入推播通知
- [ ] 優化 UI/UX
- [ ] 加入深色/淺色主題切換
- [ ] 加入項目搜尋功能
- [ ] 加入項目編輯功能（目前只能新增和刪除）

## 授權

本專案為個人專案，可自由使用和修改。

## 聯絡方式

如有問題或建議，請透過 GitHub Issues 聯絡。
