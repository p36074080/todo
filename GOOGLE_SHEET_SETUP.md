# Google Apps Script 設定指南

## 步驟 1: 設定 Google Sheet

1. 建立新的 Google Sheet
2. 將第一個 sheet 重新命名為 `items`
3. 在第 1 行貼上標題列：

```
id	text	type	subtype	priority	status	createdAt	updatedAt	dueDate	createdBy	note	assignee
```

**注意**：`subtype` 欄位保留以維持向後相容，但新功能使用 `assignee`（負責人）欄位。

## 步驟 2: 貼上測試資料（先測試 GET）

請參考 `TEST_DATA.md` 檔案，手動貼上測試資料到 Google Sheet 第 2 行開始。

## 步驟 3: 建立 Google Apps Script

1. 在 Google Sheet 中：**擴充功能** → **Apps Script**
2. 刪除預設的 `Code.gs` 內容
3. 貼上以下完整程式碼：

```javascript
// Sheet configuration
const SHEET_NAME = 'items';
const HEADER_ROW = 1;

// Column indices (0-based)
const COL_ID = 0;
const COL_TEXT = 1;
const COL_TYPE = 2;
const COL_SUBTYPE = 3;
const COL_PRIORITY = 4;
const COL_STATUS = 5;
const COL_CREATED_AT = 6;
const COL_UPDATED_AT = 7;
const COL_DUE_DATE = 8;
const COL_CREATED_BY = 9;
const COL_NOTE = 10;
const COL_ASSIGNEE = 11;

// GET handler
function doGet(e) {
  try {
    const action = e.parameter.action || '';
    
    if (action === 'items') {
      return handleGetItems(e);
    } else if (action === 'calendar') {
      return handleGetCalendar(e);
    } else {
      return createResponse({ error: 'Invalid action' }, 400);
    }
  } catch (error) {
    return createResponse({ error: error.toString() }, 500);
  }
}

// POST handler
function doPost(e) {
  try {
    let postData;
    
    // Handle FormData
    if (e.parameter && e.parameter.payload) {
      postData = JSON.parse(e.parameter.payload);
    }
    // Handle JSON body
    else if (e.postData && e.postData.contents) {
      postData = JSON.parse(e.postData.contents);
    } else {
      return createResponse({ error: 'No POST data received' }, 400);
    }
    
    const action = postData.action || '';
    
    if (action === 'create') {
      return handleCreateItem(postData);
    } else if (action === 'update') {
      return handleUpdateItem(postData);
    } else if (action === 'archive') {
      return handleArchiveItem(postData);
    } else {
      return createResponse({ error: 'Invalid action: ' + action }, 400);
    }
  } catch (error) {
    return createResponse({ error: error.toString() }, 500);
  }
}

// GET /items
function handleGetItems(e) {
  const sheet = getSheet();
  const data = sheet.getDataRange().getValues();
  const rows = data.slice(HEADER_ROW);
  
  let items = rows.map((row) => {
    return {
      id: row[COL_ID],
      text: row[COL_TEXT],
      type: row[COL_TYPE],
      subtype: row[COL_SUBTYPE] || '',
      priority: row[COL_PRIORITY],
      status: row[COL_STATUS],
      createdAt: row[COL_CREATED_AT],
      updatedAt: row[COL_UPDATED_AT],
      dueDate: row[COL_DUE_DATE] || '',
      createdBy: row[COL_CREATED_BY] || '',
      note: row[COL_NOTE] || '',
      assignee: (row[COL_ASSIGNEE] !== undefined && row[COL_ASSIGNEE] !== null) ? row[COL_ASSIGNEE] : ''
    };
  });
  
  // Filter by status
  const status = e.parameter.status || 'active';
  items = items.filter(item => item.status === status);
  
  // Filter by type if specified
  if (e.parameter.type) {
    items = items.filter(item => item.type === e.parameter.type);
  }
  
  // Sort: priority (high -> mid -> low), then updatedAt desc
  const priorityOrder = { high: 1, mid: 2, low: 3 };
  items.sort((a, b) => {
    const priorityDiff = (priorityOrder[a.priority] || 99) - (priorityOrder[b.priority] || 99);
    if (priorityDiff !== 0) return priorityDiff;
    return (b.updatedAt || 0) - (a.updatedAt || 0);
  });
  
  return createResponse({ items: items });
}

// POST /items (create)
function handleCreateItem(data) {
  const sheet = getSheet();
  const now = Date.now();
  const id = 'it_' + now;
  
  const row = [
    id,
    data.text || '',
    data.type || 'todo',
    data.subtype || '',
    data.priority || 'mid',
    'active',
    now,
    now,
    data.dueDate || '',
    data.createdBy || '',
    data.note || '',
    data.assignee || ''
  ];
  
  sheet.appendRow(row);
  
  return createResponse({
    success: true,
    item: {
      id: id,
      text: data.text,
      type: data.type,
      subtype: data.subtype || '',
      priority: data.priority,
      status: 'active',
      createdAt: now,
      updatedAt: now,
      dueDate: data.dueDate || '',
      createdBy: data.createdBy || '',
      note: data.note || '',
      assignee: data.assignee || ''
    }
  });
}

// PATCH /items/:id (update)
function handleUpdateItem(data) {
  const sheet = getSheet();
  const dataRange = sheet.getDataRange();
  const values = dataRange.getValues();
  
  let rowIndex = -1;
  for (let i = HEADER_ROW; i < values.length; i++) {
    if (values[i][COL_ID] === data.id) {
      rowIndex = i;
      break;
    }
  }
  
  if (rowIndex === -1) {
    return createResponse({ error: 'Item not found' }, 404);
  }
  
  const row = values[rowIndex];
  const now = Date.now();
  
  // Ensure row has enough columns (for backward compatibility)
  while (row.length <= COL_ASSIGNEE) {
    row.push('');
  }
  
  if (data.status !== undefined) row[COL_STATUS] = data.status;
  if (data.text !== undefined) row[COL_TEXT] = data.text;
  if (data.priority !== undefined) row[COL_PRIORITY] = data.priority;
  if (data.subtype !== undefined) row[COL_SUBTYPE] = data.subtype;
  if (data.assignee !== undefined) row[COL_ASSIGNEE] = data.assignee;
  if (data.dueDate !== undefined) row[COL_DUE_DATE] = data.dueDate;
  if (data.note !== undefined) row[COL_NOTE] = data.note;
  
  row[COL_UPDATED_AT] = now;
  
  const range = sheet.getRange(rowIndex + 1, 1, 1, row.length);
  range.setValues([row]);
  
  return createResponse({
    success: true,
    item: {
      id: row[COL_ID],
      text: row[COL_TEXT],
      type: row[COL_TYPE],
      subtype: row[COL_SUBTYPE] || '',
      priority: row[COL_PRIORITY],
      status: row[COL_STATUS],
      createdAt: row[COL_CREATED_AT],
      updatedAt: row[COL_UPDATED_AT],
      dueDate: row[COL_DUE_DATE] || '',
      createdBy: row[COL_CREATED_BY] || '',
      note: row[COL_NOTE] || '',
      assignee: (row[COL_ASSIGNEE] !== undefined && row[COL_ASSIGNEE] !== null) ? row[COL_ASSIGNEE] : ''
    }
  });
}

// PATCH /items/:id/archive
function handleArchiveItem(data) {
  const sheet = getSheet();
  const dataRange = sheet.getDataRange();
  const values = dataRange.getValues();
  
  let rowIndex = -1;
  for (let i = HEADER_ROW; i < values.length; i++) {
    if (values[i][COL_ID] === data.id) {
      rowIndex = i;
      break;
    }
  }
  
  if (rowIndex === -1) {
    return createResponse({ error: 'Item not found' }, 404);
  }
  
  const row = values[rowIndex];
  row[COL_STATUS] = 'archived';
  row[COL_UPDATED_AT] = Date.now();
  
  const range = sheet.getRange(rowIndex + 1, 1, 1, row.length);
  range.setValues([row]);
  
  return createResponse({ success: true });
}

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

// Helper: Get sheet
function getSheet() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let sheet = ss.getSheetByName(SHEET_NAME);
  
  if (!sheet) {
    sheet = ss.insertSheet(SHEET_NAME);
    sheet.getRange(HEADER_ROW, 1, 1, 12).setValues([[
      'id', 'text', 'type', 'subtype', 'priority', 'status',
      'createdAt', 'updatedAt', 'dueDate', 'createdBy', 'note', 'assignee'
    ]]);
  }
  
  return sheet;
}

// Helper: Create response
function createResponse(data, statusCode = 200) {
  const output = ContentService.createTextOutput(JSON.stringify(data));
  output.setMimeType(ContentService.MimeType.JSON);
  return output;
}

// Helper: Format date
function formatDate(date) {
  const year = date.getFullYear();
  const month = String(date.getMonth() + 1).padStart(2, '0');
  const day = String(date.getDate()).padStart(2, '0');
  return `${year}-${month}-${day}`;
}

// Helper: Format time
function formatTime(date) {
  const hours = String(date.getHours()).padStart(2, '0');
  const minutes = String(date.getMinutes()).padStart(2, '0');
  return `${hours}:${minutes}`;
}

// Helper: Get date label
function getDateLabel(date, today) {
  const diffDays = Math.floor((date - today) / (1000 * 60 * 60 * 24));
  
  if (diffDays === 0) return 'Today';
  if (diffDays === 1) return 'Tomorrow';
  if (diffDays === 2) return 'Day+2';
  
  return formatDate(date);
}
```

4. 點擊**儲存**

## 步驟 4: 部署 Web App

1. 點擊**部署** → **新增部署作業**
2. 點擊**選取類型**旁的齒輪 → 選擇**網頁應用程式**
3. 設定：
   - **執行身分**：我
   - **具有存取權的使用者**：所有人
4. 點擊**部署**
5. **複製 Web App URL**（格式：`https://script.google.com/macros/s/.../exec`）

## 步驟 5: 測試 GET 請求

1. 在瀏覽器開啟：`你的URL?action=items&status=active`
2. 應該會看到 JSON 回應，包含你手動貼上的測試資料

## 步驟 6: 更新前端 API URL

在 `editor/index.html` 和 `xr-display/index.html` 中找到：

```javascript
const API_BASE_URL = 'YOUR_GOOGLE_APPS_SCRIPT_URL_HERE';
```

替換為你的 Web App URL。

## API 端點

### GET /items
```
{URL}?action=items&status=active&type=todo
```

### POST /items (create)
使用 FormData：
```javascript
const formData = new FormData();
formData.append('payload', JSON.stringify({
  action: 'create',
  text: '買牛奶',
  type: 'shopping',
  priority: 'high'
}));
fetch(URL, { method: 'POST', body: formData });
```

### POST /items (update)
```javascript
formData.append('payload', JSON.stringify({
  action: 'update',
  id: 'it_xxx',
  status: 'done'
}));
```

### POST /items (archive)
```javascript
formData.append('payload', JSON.stringify({
  action: 'archive',
  id: 'it_xxx'
}));
```
