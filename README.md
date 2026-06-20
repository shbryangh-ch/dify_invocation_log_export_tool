# dify_invocation_log_export_tool

把自架 (self-hosted) Dify 後台「App Logs」頁面看到的對話紀錄,匯出成 CSV,方便離線分析、保存或交給其他系統處理。


## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.


## 為什麼不直接用官方 API?

這個工具最後選擇直接連資料庫,是因為前面兩條路都走不通:

- **Service API (`/v1/conversations`)**:官方文件明確說明這個端點不會回傳 WebApp / 公開頁面建立的對話,只會回傳透過 API 呼叫(帶自訂 `user` 參數)建立的對話。如果你的對話主要來自 WebApp,這條路抓不到資料。
- **Console (後台) API**:需要先登入拿到 token,但新版 Dify 為了防止帳號列舉攻擊,登入密碼改成前端用 RSA 公鑰加密後再送出,後端用對應的私鑰解密驗證。這個機制沒有公開文件,且不同版本可能會調整,逆向成本高、也不穩定。

因此最終做法是**直接連 Dify 使用的 PostgreSQL 資料庫查詢**,資料來源跟後台顯示的完全一致,也不需要處理登入或加密問題。


## 需求

- Python 3.8+
- `psycopg2-binary`
  ```bash
  pip install psycopg2-binary
  ```
- 能連線到 Dify 用的 PostgreSQL(預設 docker-compose 設定不會對外開放,需要手動開啟,見下方說明)
- Dify 資料庫帳密(`.env` 裡的 `DB_USERNAME` / `DB_PASSWORD`;)
- 要匯出的 App 的 `app_id`(在後台網址 `/app/<APP_ID>/...` 可以看到)

## 開放資料庫連接埠

自架 Dify 的 docker-compose 預設只讓 Postgres 在 docker 內網互通,不會對外開放連接埠。打開 `docker-compose.yaml`,找到 Postgres 服務的定義區塊(視版本可能叫 `db` 或 `db_postgres`,特徵是底下會接著 `image: postgres...`),加上:

```yaml
    ports:
      - "5432:5432"
```

存檔後在同一個資料夾執行:

```bash
docker compose up -d
```

這只會重建 Postgres 這一個容器,資料(存在 named volume 裡)不會受影響,其他服務也不會被中斷。

## 設定

打開 `dify_export_logs_v3.py`,修改最上面這幾個變數:

```python
DB_HOST = "localhost"
DB_PORT = 5432
DB_NAME = "dify"
DB_USER = "postgres"
DB_PASSWORD = "你的密碼"
APP_ID = "你的 app_id"
```

## 使用方式

```bash
python dify_export_logs_v3.py
```

執行後會在同個資料夾產生兩個檔案:

- **`dify_logs_export.csv`**:對話清單,對應後台 App Logs 頁面看到的那張表。欄位包含標題、使用者或帳戶、狀態、訊息數、使用者反饋、管理員反饋、建立時間、更新時間。
- **`dify_messages_export.csv`**:每一輪問答的完整內容(提問、回覆、狀態、錯誤訊息)。一個對話會展開成多列,用「標題」或「使用者或帳戶」可以對回上面那張對話清單。

## 欄位對不上怎麼辦

這個工具是直接讀資料庫的 schema,不同 Dify 版本欄位可能略有差異。如果執行時跳出 `column ... does not exist` 之類的錯誤,把 `__main__` 區塊改成先跑欄位檢查:

```python
if __name__ == "__main__":
    step1_inspect_schema()
```

印出 `conversations`、`messages`、`message_feedbacks` 實際的欄位清單後,對照修改檔案裡的 `EXPORT_QUERY` 跟 `EXPORT_MESSAGES_QUERY` 即可。

## 已知限制

- 只匯出未被軟刪除的對話(`is_deleted = FALSE`)。
- 「狀態」欄位是依據對話內是否有任一則訊息 `status = 'error'` 來判斷 SUCCESS / FAILURE,不是直接讀 `conversations.status`(那個欄位的意義不同)。
- 「使用者或帳戶」顯示的是原始的 `from_end_user_id` 或 `from_account_id`(UUID 格式),沒有額外查表轉換成顯示名稱。

## 安全提醒

- 不要把資料庫密碼寫死後直接 commit 上傳到公開 repo。建議改用環境變數讀取,或把帳密相關設定加進 `.gitignore`。
- 這是只讀查詢,理論上不會修改任何資料,但仍建議申請一個只有 `SELECT` 權限的資料庫帳號來跑這個腳本,而不是直接用 superuser 帳密。
- 如果完成匯出後不再需要對外連線,記得把 `docker-compose.yaml` 裡加的 `ports` 設定拿掉、`docker compose up -d` 還原,避免資料庫長期暴露在主機網路上。
