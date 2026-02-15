# Google Calendar 串接設定指南

本文檔說明如何設定 XR Display 的行事曆功能，讓它能夠顯示你的 Google Calendar 事件。

## 概述

XR Display 的行事曆功能會自動從你的 Google Calendar 讀取未來 3 天的事件，並顯示在右側面板中。這個功能已經整合在 Google Apps Script 中，只需要正確設定權限即可。

## 工作原理

1. **前端（XR Display）**：發送 API 請求到 Google Apps Script
2. **後端（Google Apps Script）**：使用 `CalendarApp.getDefaultCalendar()` 讀取你的預設 Google Calendar
3. **資料處理**：將事件按照日期分組，格式化後回傳給前端
4. **顯示**：XR Display 在右側面板顯示這些事件

## 設定步驟

### 步驟 1: 確認 Google Apps Script 程式碼

確認你的 Google Apps Script 中已經包含 `handleGetCalendar` 函數。這個函數應該已經在 `GOOGLE_SHEET_SETUP.md` 的程式碼中。

如果沒有，請確認以下函數存在：

```javascript
// GET /calendar
function handleGetCalendar(e) {
  const days = parseInt(e.parameter.days || '3', 10);
  const calendar = CalendarApp.getDefaultCalendar();
  const now = new Date();
  const endDate = new Date(now.getTime() + days * 24 * 60 * 60 * 1000);
  
  const events = calendar.getEvents(now, endDate);
  
  const eventsByDate = {};
  const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
  
  events.forEach(event => {
    const eventDate = new Date(event.getStartTime());
    const dateKey = formatDate(eventDate);
    const dateLabel = getDateLabel(eventDate, today);
    
    if (!eventsByDate[dateKey]) {
      eventsByDate[dateKey] = {
        date: dateKey,
        label: dateLabel,
        events: []
      };
    }
    
    eventsByDate[dateKey].events.push({
      start: formatTime(event.getStartTime()),
      end: formatTime(event.getEndTime()),
      title: event.getTitle(),
      description: event.getDescription() || ''
    });
  });
  
  const daysArray = Object.values(eventsByDate).sort((a, b) => {
    return a.date.localeCompare(b.date);
  });
  
  return createResponse({ days: daysArray });
}
```

### 步驟 2: 授權 Google Calendar 權限

當你首次部署 Google Apps Script Web App 時，系統會要求授權。請確保授權包含 Google Calendar 的讀取權限。

**授權流程：**

1. 在 Google Apps Script 編輯器中，點擊「部署」→「新增部署作業」
2. 選擇「網頁應用程式」
3. 點擊「部署」
4. 系統會顯示授權畫面
5. 點擊「授權存取權限」
6. 選擇你的 Google 帳號
7. 點擊「進階」→「前往 [專案名稱]（不安全）」
8. 點擊「允許」
9. **重要**：確認授權範圍包含：
   - ✅ Google Sheets（讀寫）
   - ✅ Google Calendar（讀取）

### 步驟 3: 確認預設日曆

Google Apps Script 使用 `CalendarApp.getDefaultCalendar()` 來讀取你的**預設 Google Calendar**。

**如何確認預設日曆：**

1. 前往 [Google Calendar](https://calendar.google.com)
2. 點擊左側的「設定」（齒輪圖示）
3. 查看「我的日曆」區塊
4. 確認你的主要日曆（通常是你的 Google 帳號名稱）是啟用狀態
5. 這個日曆就是「預設日曆」

**注意事項：**
- 系統只會讀取**預設日曆**的事件
- 如果你有多個日曆，只有預設日曆的事件會顯示
- 如果你想要顯示其他日曆，需要修改 Google Apps Script 程式碼（見下方「進階設定」）

### 步驟 4: 測試行事曆 API

部署完成後，測試行事曆 API 是否正常運作：

**方法 1: 使用瀏覽器測試**

在瀏覽器中開啟以下 URL（將 `YOUR_API_URL` 替換為你的實際 API URL）：

```
YOUR_API_URL?action=calendar&days=3
```

應該會看到類似以下的 JSON 回應：

```json
{
  "days": [
    {
      "date": "2026-02-09",
      "label": "Today",
      "events": [
        {
          "start": "19:30",
          "end": "21:00",
          "title": "晚餐",
          "description": ""
        }
      ]
    },
    {
      "date": "2026-02-10",
      "label": "Tomorrow",
      "events": []
    }
  ]
}
```

**方法 2: 在 XR Display 中測試**

1. 開啟 XR Display 頁面
2. 檢查右側的「行事曆」面板
3. 應該會顯示未來 3 天的事件
4. 如果沒有事件，確認你的 Google Calendar 中是否有未來 3 天的事件

### 步驟 5: 檢查執行記錄（如果無法運作）

如果行事曆無法顯示，檢查 Google Apps Script 的執行記錄：

1. 在 Google Apps Script 編輯器中，點擊「執行」→「檢視執行記錄」
2. 查看是否有錯誤訊息
3. 常見錯誤：
   - **權限錯誤**：需要重新授權 Calendar 權限
   - **找不到日曆**：確認 Google 帳號有預設日曆

## API 端點說明

### GET /calendar

**URL 格式：**
```
{API_BASE_URL}?action=calendar&days=3
```

**查詢參數：**
- `action=calendar`（必填）
- `days`（選填，預設為 3，表示未來 N 天）

**回應格式：**
```json
{
  "days": [
    {
      "date": "2026-02-09",
      "label": "Today",
      "events": [
        {
          "start": "19:30",
          "end": "21:00",
          "title": "晚餐",
          "description": "和女友吃晚餐"
        }
      ]
    },
    {
      "date": "2026-02-10",
      "label": "Tomorrow",
      "events": [
        {
          "start": "09:00",
          "end": "18:00",
          "title": "上班",
          "description": ""
        }
      ]
    },
    {
      "date": "2026-02-11",
      "label": "Day+2",
      "events": []
    }
  ]
}
```

**日期標籤說明：**
- `Today`：今天
- `Tomorrow`：明天
- `Day+2`：後天
- 其他日期：顯示日期（YYYY-MM-DD）

## 顯示共享日曆

### 方法 1: 將共享日曆加入「我的日曆」（推薦）

**步驟：**

1. 請對方在 Google Calendar 中：
   - 點擊左側的日曆名稱旁邊的「⋮」（三個點）
   - 選擇「設定和共用」
   - 在「與特定使用者共用」區塊中，輸入你的 Google 帳號 email
   - 設定權限為「查看所有活動詳情」或「進行變更」
   - 點擊「傳送」

2. 你會在 Gmail 收到共享邀請，點擊「是，新增這個日曆」

3. 確認共享日曆已出現在 Google Calendar 左側的「我的日曆」區塊中

4. 修改 Google Apps Script 程式碼（見下方「讀取多個日曆」）

### 方法 2: 使用日曆 ID

如果你知道共享日曆的 ID，可以直接使用日曆 ID 來讀取。

## 進階設定

### 選擇特定日曆（而非預設日曆）

如果你想要讀取特定的日曆，而不是預設日曆，可以修改 `handleGetCalendar` 函數：

```javascript
// GET /calendar
function handleGetCalendar(e) {
  const days = parseInt(e.parameter.days || '3', 10);
  
  // 方法 1: 使用日曆名稱
  const calendars = CalendarApp.getCalendarsByName('你的日曆名稱');
  const calendar = calendars[0] || CalendarApp.getDefaultCalendar();
  
  // 方法 2: 使用日曆 ID（可以在 Google Calendar 設定中找到）
  // const calendar = CalendarApp.getCalendarById('your-calendar-id@group.calendar.google.com');
  
  const now = new Date();
  const endDate = new Date(now.getTime() + days * 24 * 60 * 60 * 1000);
  
  const events = calendar.getEvents(now, endDate);
  
  // ... 其餘程式碼相同
}
```

**如何取得日曆 ID：**

1. 前往 [Google Calendar](https://calendar.google.com)
2. 點擊左側的日曆名稱旁邊的「⋮」（三個點）
3. 選擇「設定和共用」
4. 在「整合日曆」區塊中，找到「日曆 ID」
5. 複製日曆 ID（格式類似：`xxxxx@group.calendar.google.com`）

### 讀取多個日曆（包含共享日曆）

如果你想要合併多個日曆的事件（包括共享日曆），可以修改 `handleGetCalendar` 函數：

**方法 A: 使用日曆名稱**

```javascript
// GET /calendar
function handleGetCalendar(e) {
  const days = parseInt(e.parameter.days || '3', 10);
  const now = new Date();
  const endDate = new Date(now.getTime() + days * 24 * 60 * 60 * 1000);
  
  // 取得多個日曆
  const defaultCalendar = CalendarApp.getDefaultCalendar();
  const sharedCalendar = CalendarApp.getCalendarsByName('共享日曆名稱')[0]; // 替換為實際的日曆名稱
  
  // 合併所有事件
  const allEvents = [];
  if (defaultCalendar) {
    allEvents.push(...defaultCalendar.getEvents(now, endDate));
  }
  if (sharedCalendar) {
    allEvents.push(...sharedCalendar.getEvents(now, endDate));
  }
  
  // 按照開始時間排序
  allEvents.sort((a, b) => a.getStartTime() - b.getStartTime());
  
  // 按照日期分組
  const eventsByDate = {};
  const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
  
  allEvents.forEach(event => {
    const eventDate = new Date(event.getStartTime());
    const dateKey = formatDate(eventDate);
    const dateLabel = getDateLabel(eventDate, today);
    
    if (!eventsByDate[dateKey]) {
      eventsByDate[dateKey] = {
        date: dateKey,
        label: dateLabel,
        events: []
      };
    }
    
    eventsByDate[dateKey].events.push({
      start: formatTime(event.getStartTime()),
      end: formatTime(event.getEndTime()),
      title: event.getTitle(),
      description: event.getDescription() || ''
    });
  });
  
  const daysArray = Object.values(eventsByDate).sort((a, b) => {
    return a.date.localeCompare(b.date);
  });
  
  return createResponse({ days: daysArray });
}
```

**方法 B: 讀取所有可存取的日曆（自動包含共享日曆）**

```javascript
// GET /calendar
function handleGetCalendar(e) {
  const days = parseInt(e.parameter.days || '3', 10);
  const now = new Date();
  const endDate = new Date(now.getTime() + days * 24 * 60 * 60 * 1000);
  
  // 取得所有可存取的日曆（包括共享日曆）
  const allCalendars = CalendarApp.getAllCalendars();
  
  // 合併所有日曆的事件
  const allEvents = [];
  allCalendars.forEach(calendar => {
    try {
      const events = calendar.getEvents(now, endDate);
      allEvents.push(...events);
    } catch (error) {
      // 如果某個日曆無法存取，跳過它
      Logger.log('無法讀取日曆: ' + calendar.getName() + ', 錯誤: ' + error);
    }
  });
  
  // 按照開始時間排序
  allEvents.sort((a, b) => a.getStartTime() - b.getStartTime());
  
  // 按照日期分組（其餘程式碼相同）
  const eventsByDate = {};
  const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
  
  allEvents.forEach(event => {
    const eventDate = new Date(event.getStartTime());
    const dateKey = formatDate(eventDate);
    const dateLabel = getDateLabel(eventDate, today);
    
    if (!eventsByDate[dateKey]) {
      eventsByDate[dateKey] = {
        date: dateKey,
        label: dateLabel,
        events: []
      };
    }
    
    eventsByDate[dateKey].events.push({
      start: formatTime(event.getStartTime()),
      end: formatTime(event.getEndTime()),
      title: event.getTitle(),
      description: event.getDescription() || ''
    });
  });
  
  const daysArray = Object.values(eventsByDate).sort((a, b) => {
    return a.date.localeCompare(b.date);
  });
  
  return createResponse({ days: daysArray });
}
```

**推薦使用方法 B**，因為它會自動包含：
- 你的預設日曆
- 所有共享給你的日曆
- 你建立的其他日曆

不需要手動指定每個日曆名稱。

## 疑難排解

### 問題 1: 行事曆顯示「未來三天沒有活動」

**可能原因：**
- Google Calendar 中確實沒有未來 3 天的事件
- 預設日曆被停用
- 權限設定問題

**解決方法：**
1. 確認 Google Calendar 中有未來 3 天的事件
2. 確認預設日曆是啟用狀態
3. 測試 API：`YOUR_API_URL?action=calendar&days=3`
4. 檢查執行記錄是否有錯誤

### 問題 2: 出現權限錯誤

**錯誤訊息：**
```
Authorization required
```

**解決方法：**
1. 重新部署 Google Apps Script
2. 重新授權，確保包含 Calendar 權限
3. 確認 Google 帳號有 Calendar 存取權限

### 問題 3: 只顯示部分事件

**可能原因：**
- 事件在非預設日曆中
- 事件是全天事件，時間顯示可能不同

**解決方法：**
- 確認事件在預設日曆中
- 或修改程式碼讀取特定日曆（見「進階設定」）

### 問題 4: 時間顯示不正確

**可能原因：**
- 時區設定問題

**解決方法：**
- Google Apps Script 會自動使用你的 Google 帳號時區
- 確認 Google Calendar 的時區設定正確

## 資料格式說明

### 事件物件結構

每個事件包含以下欄位：

```javascript
{
  "start": "19:30",        // 開始時間（HH:MM 格式）
  "end": "21:00",          // 結束時間（HH:MM 格式）
  "title": "晚餐",         // 事件標題
  "description": ""        // 事件描述（選填）
}
```

### 日期分組

事件會按照日期分組，每天包含：

```javascript
{
  "date": "2026-02-09",    // 日期（YYYY-MM-DD 格式）
  "label": "Today",        // 顯示標籤（Today/Tomorrow/Day+2/日期）
  "events": [...]          // 該日期的事件陣列
}
```

## 自動刷新

XR Display 會自動：
- 每 7 分鐘重新抓取行事曆資料
- 每 30 分鐘重新載入整個頁面

這確保行事曆資料保持最新狀態。

## 注意事項

1. **隱私**：行事曆資料只會在你的 Google Apps Script 和前端之間傳遞，不會儲存在其他地方
2. **權限**：Google Apps Script 只需要**讀取**權限，不會修改你的日曆
3. **效能**：每次請求會讀取未來 3 天的事件，如果事件很多可能會稍微影響載入速度
4. **快取**：前端會快取行事曆資料 5 分鐘，減少 API 請求次數

## 測試檢查清單

- [ ] Google Apps Script 已部署並取得 API URL
- [ ] 已授權 Calendar 讀取權限
- [ ] Google Calendar 中有未來 3 天的事件
- [ ] 測試 API：`YOUR_API_URL?action=calendar&days=3` 有正確回應
- [ ] XR Display 右側面板顯示行事曆事件
- [ ] 事件時間和標題正確顯示

完成以上檢查後，行事曆功能應該就能正常運作了！
