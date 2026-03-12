# TenColor 網站 SSL 憑證安裝 SOP

## 環境資訊

| 項目 | 內容 |
|------|------|
| 伺服器 FQDN | `ec2-43-202-225-47.ap-northeast-2.compute.amazonaws.com` |
| SSH 金鑰 | `D:\.ssh\ten-color.pem` |
| 網站 HTML 路徑 | `/data/tencolor_web` |
| Docker Volume | `/data/tencolor_web:/usr/share/nginx/html` |
| SSL 憑證路徑 | `/data/nginx/conf.d/www.tpmitest.tw.crt` |
| SSL 金鑰路徑 | `/data/nginx/conf.d/www.tpmitest.tw.key` |

---

## 查詢現有憑證到期日

```bash
openssl x509 -in /data/nginx/conf.d/www.tpmitest.tw.crt -noout -dates
```

---

## 申請新憑證流程

### Step 1：取得文件驗證資訊

從憑證申請頁面取得驗證碼與發行商資訊，例如：

- 驗證碼：`<驗證碼>`
- 發行商網域：`comodoca.com`

---

### Step 2：建立驗證資料夾

```bash
mkdir -p /data/tencolor_web/.well-known/pki-validation
```

---

### Step 3：建立驗證檔案

```bash
vi /data/tencolor_web/.well-known/pki-validation/<驗證碼>.txt
```

檔案內容：

```
<驗證碼>
comodoca.com
```

---

### Step 4：調整 Nginx 設定

編輯 `default.conf`，確保 HTTP 80 port 可正確提供驗證路徑：

```nginx
server {
    listen 80;
    server_name www.tpmitest.tw;

    location ^~ /.well-known/pki-validation/ {
        root /usr/share/nginx/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

---

### Step 5：Reload Nginx

```bash
docker exec nginx nginx -s reload
```

---

### Step 6：測試驗證檔案可讀取

```bash
curl http://www.tpmitest.tw/.well-known/pki-validation/<驗證碼>.txt
```

**成功回應範例：**

```
<驗證碼>
comodoca.com
```

---

### Step 7：向憑證機構請求驗證

回到憑證申請頁面，點擊「**請求驗證（Request Validation）**」。

---

## 安裝新憑證

### Step 8：下載憑證檔案

從憑證機構下載以下兩個檔案：

- `www.tpmitest.tw.crt`
- `www.tpmitest.tw.ca-bundle`

---

### Step 9：合併為 Fullchain 憑證

```bash
cat www.tpmitest.tw.crt www.tpmitest.tw.ca-bundle > fullchain.crt
```

---

### Step 10：將憑證放入 Nginx 設定目錄

| 用途 | 來源檔案 | 目標路徑 |
|------|----------|----------|
| SSL 憑證 | `fullchain.crt` | `/data/nginx/conf.d/www.tpmitest.tw.crt` |
| SSL 私鑰 | CSR 產生的 `.key` | `/data/nginx/conf.d/www.tpmitest.tw.key` |

Nginx 設定參考：

```nginx
ssl_certificate     /etc/nginx/conf.d/www.tpmitest.tw.crt;
ssl_certificate_key /etc/nginx/conf.d/www.tpmitest.tw.key;
```

---

### Step 11：Reload Nginx

```bash
docker exec nginx nginx -s reload
```

---

### Step 12：測試 SSL 憑證

```bash
curl -I https://www.tpmitest.tw
```

回應包含 `HTTP/2 200` 或 `HTTP/1.1 200 OK` 即表示安裝成功。

---

## 流程摘要

```
取得驗證碼
    ↓
建立驗證資料夾與檔案
    ↓
調整 Nginx 設定 → Reload Nginx
    ↓
curl 測試驗證檔案可讀取
    ↓
回到憑證頁按「請求驗證」
    ↓
下載 .crt + .ca-bundle → 合併為 fullchain.crt
    ↓
放入 Nginx → Reload Nginx
    ↓
curl -I 測試 HTTPS
```