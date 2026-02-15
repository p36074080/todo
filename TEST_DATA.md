# 測試資料 - 手動貼到 Google Sheet

請將以下資料複製貼到 Google Sheet 的 `items` sheet 中（從第 2 行開始，第 1 行是標題列）

## 測試資料範例

**注意**：現在欄位包含 `assignee`（負責人），請在 Google Sheet 的 L 欄（第 12 欄）新增 `assignee` 標題。

```
it_1737000000001	買牛奶	shopping			high	active	1737000000001	1737000000001			Jerry		熊熊
it_1737000000002	整理下週行程	todo	task		high	active	1737000000002	1737000000002			Jerry		貓貓
it_1737000000003	討論三月旅行地點	todo	discuss		mid	active	1737000000003	1737000000003			Jerry		熊熊＆貓貓
it_1737000000004	繳停車費	todo	task		low	done	1737000000004	1737000000004			Jerry		
it_1737000000005	衛生紙	shopping			high	active	1737000000005	1737000000005			Jerry		熊熊
```

## 欄位說明

| 欄位 | 範例值 | 說明 |
|------|--------|------|
| id | it_1737000000001 | 唯一 ID，格式：it_ + 時間戳 |
| text | 買牛奶 | 項目內容 |
| type | shopping | todo 或 shopping |
| subtype | (空白) | task 或 discuss（保留欄位，向後相容） |
| priority | high | high、mid、low |
| status | active | active、done、archived |
| createdAt | 1737000000001 | 建立時間（數字，epoch milliseconds） |
| updatedAt | 1737000000001 | 更新時間（數字，epoch milliseconds） |
| dueDate | (空白) | 到期日（YYYY-MM-DD 格式，選填） |
| createdBy | Jerry | 建立者（選填） |
| note | (空白) | 備註（選填） |
| assignee | 熊熊 | 負責人：熊熊、貓貓、熊熊＆貓貓（選填） |

## 使用方式

1. 開啟你的 Google Sheet
2. 確認第一個 sheet 名稱是 `items`
3. 確認第 1 行有標題列：`id	text	type	subtype	priority	status	createdAt	updatedAt	dueDate	createdBy	note	assignee`
   - **重要**：如果還沒有 `assignee` 欄位，請在 L 欄（第 12 欄）新增 `assignee` 標題
4. 從第 2 行開始貼上測試資料（用 Tab 分隔）
5. 測試 Editor 頁面是否能正常載入這些資料
