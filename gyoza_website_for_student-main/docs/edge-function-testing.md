# Edge Function 測試筆記(create-order / get-order)

使用 Supabase Dashboard → Edge Functions → 選擇函式 → **Test** 面板進行測試。
本筆記不含任何金鑰;授權請在 Test 面板的選單選 **anon**,不要手動輸入金鑰。

## 測試用資料(課堂測試資料庫中已確認存在且啟用)

| 資料表 | id | 名稱 | 價格 |
|---|---|---|---|
| `stores` | `demo-store` | 課堂測試門市 | — |
| `products` | `demo-pork` | 招牌豬肉鍋貼一份 | 80 |
| `products` | `demo-chive` | 韭菜豬肉鍋貼一份 | 90 |

## 共同設定(兩支函式相同)

| 欄位 | 填法 |
|---|---|
| HTTP Method | `POST`(其他方法回 405) |
| Headers | `Content-Type: application/json` |
| Query Parameters | 留空(程式只讀 Body) |
| 授權 | 依各函式 Settings 的 Verify JWT;開啟時在 Test 面板選 anon。兩支函式分開設定 |

---

## 一、create-order(下單)

> ⚠️ 每按一次送出就會建立一筆新訂單,只按一次。

### Request Body

```json
{
  "mode": "take",
  "name": "測試顧客",
  "phone": "0900000000",
  "email": "test@example.com",
  "store_id": "demo-store",
  "address": "",
  "note": "課堂測試,請勿製作或配送",
  "hp": "",
  "items": [
    { "id": "demo-pork", "qty": 1 },
    { "id": "demo-chive", "qty": 1 }
  ]
}
```

- `qty` 必須是數字 `1`,不能是文字 `"1"`。
- `hp` 必須是空字串;有內容會被當成機器人,回傳 `JF-IGNORED` 且不寫入。
- 不要填訂單編號、價格、總額:由資料庫函式 `create_order` 產生與計算。

### 預期回應(HTTP 200)

```json
{ "ok": true, "order_no": "JF<yyMMdd>-<流水號>", "total": 170, "eta": "約 15–20 分鐘", "orderNo": "同 order_no" }
```

- 小計 80 + 90 = 170,外帶費用 0,總額 170。
- **記下 `order_no`**,get-order 測試要用。

### 送出後在 Table Editor 檢查

| 資料表 | 應看到 |
|---|---|
| `orders` | +1 筆:status `new`、fulfillment_type `take`、store_id `demo-store`、subtotal 170、fee 0、total 170 |
| `order_items` | +2 筆:80 元 × 1、90 元 × 1 |
| `order_status_log` | +1 筆:from_status 空白 → to_status `new` |
| `order_counters` | 當天日期那一列,`last_seq` 與編號流水號相同 |

---

## 二、get-order(查單)

> 本次測試的訂單:**JF261119-002**(2026-11-19 建立)。

### 步驟 1:查詢訂單

```json
{ "order_no": "JF261119-002" }
```

預期 HTTP 200,核對:

- [ ] `order_no` 為 `JF261119-002`
- [ ] `items` 兩項:80 × 1、90 × 1(依品名排序,順序可能不同)
- [ ] `subtotal` 170、`fee` 0、`total` 170
- [ ] `status` 為 `new`、`statusLabel` 為「新訂單」
- [ ] 回應**不含**姓名、手機、Email、備註(刻意設計)
- 時間為 UTC(`+00:00`),加 8 小時為台灣時間。記下 `updatedAt`。

### 步驟 2:在 Table Editor 改成「製作中」

允許的狀態碼:`new`、`preparing`、`ready`、`delivering`、`done`、`cancelled`。「製作中」= **`preparing`**。

1. Table Editor → `orders`。
2. 用 Filter:`order_no` equals `JF261119-002`(不要改到其他訂單)。
3. 雙擊 `status`,改成 `preparing`(英文小寫),儲存。

成功時:

- `status` 變成 `preparing`,`updated_at` 自動更新(觸發器)。
- `order_status_log` 多一筆:`new` → `preparing`,changed_by 空白(Dashboard 直接修改)。
- 若出現 `orders_status_check` 錯誤:狀態碼拼錯。

### 步驟 3:重新查詢

送出與步驟 1 相同的 Body,預期:

| 欄位 | 步驟 1 | 步驟 3 |
|---|---|---|
| `status` | `new` | `preparing` |
| `statusLabel` | 新訂單 | 製作中 |
| `updatedAt` | 原時間 | 步驟 2 儲存的時間 |
| `createdAt`、`items`、`total` | — | 不變 |

### 步驟 4:查無訂單

先在 Table Editor 以 Filter 確認 `JF261119-999` 不存在(0 筆),再送出:

```json
{ "order_no": "JF261119-999" }
```

預期 HTTP **404**:

```json
{ "ok": false, "error": "查無此訂單" }
```

Test 面板可能以紅色顯示 404,這是預期結果。
若得到 400「訂單編號格式不正確」,代表格式打錯(正確格式:`JF` + 6 位數字 + `-` + 至少 3 位數字)。

---

## 錯誤判讀

| 結果 | 可能原因 |
|---|---|
| 401 | Verify JWT 開著,但 Test 面板未選授權身分 |
| 405 | Method 不是 POST |
| 400 + 中文訊息 | 資料驗證未通過,訊息說明原因 |
| 404「查無此訂單」 | (get-order)編號不存在 |
| 500「系統忙碌中」 | 到 Edge Functions → 該函式 → **Logs** 看錯誤代碼;`23505` = 訂單編號重複 |

## 已知限制

get-order 只憑訂單編號查詢,沒有驗證查詢者身分。編號有規律,可以被推測(但回應不含個資)。正式營運前應加強,例如加上手機後三碼驗證與流量限制。
