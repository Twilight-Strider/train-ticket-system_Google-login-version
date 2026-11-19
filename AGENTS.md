# AGENTS.md — 給 AI Agent 的部署指南

本文件寫給能操作終端機與檔案的 AI Agent（Claude Code、Codex 等）。目標：**協助使用者從這個 repo 部署出一份屬於他們自己的「火車選位分票系統（Google 登入版）」**。人類讀者請看 [README.md](README.md)。

## 0. 專案一句話

單一 `index.html`（React 18 + Babel + Tailwind，全部走 CDN）＋ Firebase（Authentication 的 Google 登入、Cloud Firestore）。所有客製化都集中在 `index.html` 上方的 `const CONFIG = { ... }`，其下方標示「以下程式碼不需要修改」的部分**除了 CSP 網域外不要改**。

## 1. 鐵則（先讀）

1. **不要索取或儲存使用者的 Google 密碼、Token、服務帳戶金鑰。** 登入一律由使用者自己在瀏覽器／`firebase login` 的互動流程完成。
2. **不要把填了真實票券代碼、驗證碼的 `index.html` 推到公開 repo。** 票券資料寫在前端，部署後任何知道網址的人都看得到（見 README「安全性與隱私」），要明確告知使用者這一點。
3. **Firestore 規則必須是 Production 規則**（本文件第 5 節產生的版本）。**不得**使用測試模式或 `allow read, write: if true`。
4. `CONFIG.users` 的 Email 一律**小寫**；顯示名稱**不可重複、不可含 `/`**。
5. **修改 `CONFIG.users` 後，必須重新產生並部署 Firestore 規則**（兩者要一致）。
6. 刪除資料、覆寫使用者既有檔案、`git push`、`firebase deploy` 到使用者已有的專案之前，先取得使用者明確同意。
7. 不要自行編造票券代碼、Email、車次；缺資料就問。
8. 不要執行本文件沒有寫、且會產生費用或影響其他專案的指令；一律用 `--project <ID>` 明確指定專案，避免誤改別的專案。

## 2. 分工：哪些你能做、哪些必須請人類做

| 步驟 | 誰做 | 備註 |
|------|------|------|
| 安裝 `node`、`firebase-tools`、`git` | Agent | `npm install -g firebase-tools` |
| `firebase login` | **人類** | 互動式；無瀏覽器環境用 `firebase login --no-localhost` |
| 建立 Firebase 專案、Web App、Firestore | Agent（CLI） | 也可請人類在 Console 做 |
| **啟用 Google 登入方式（Sign-in method → Google）** | **人類** | 需在 Console 操作，這份文件沒有可靠的 CLI 替代方案 |
| **新增 Authorized domains（非 Firebase Hosting 網域時）** | **人類** | Console → Authentication → Settings |
| 編輯 `CONFIG`、CSP、準備 `images/` | Agent | 資料由人類提供 |
| 產生並部署 Firestore 規則 | Agent | 第 5 節 |
| 部署 Hosting | Agent | 需人類同意 |
| 真實登入測試、多人搶位測試 | **人類** | Agent 無法代為登入 Google |

遇到「人類」步驟時，**停下來**，給出明確的操作指引與（可用的）Console 直達連結，等使用者回覆完成再繼續。

## 3. 需要向使用者蒐集的資訊

一次問齊，不要零碎地問：

| 項目 | 格式／說明 |
|------|------------|
| 參與者名單 | 每人一組：Google 帳號 Email ＋ 顯示名稱（例：`alice@gmail.com` → `小明`）。**務必是對方登入用的 Google 帳號 Email** |
| 去程資訊 | 車次、車型、乘車時間（三行文字顯示在選位頁） |
| 去程座位圖 | 每個座位的車廂號與座位號，以及它們在「左 2 格＋右 2 格」座位圖中的排列。若使用者只給座位清單，請先與他確認排列方式 |
| 去程票券 | 每個座位的「訂票代碼」「取票驗證碼」 |
| 返程資訊／座位圖／票券 | 同上 |
| 行程表 PDF | Google Drive 檔案 ID（且已設為「知道連結的任何人可檢視」）。沒有的話確認是否要移除行程表頁 |
| Firebase 專案 ID | 新專案：全域唯一，6～30 字元，小寫英文、數字、連字號。或使用者既有專案 ID |
| Firestore 區域 | 預設建議 `asia-east1`（台灣）；可用 `firebase firestore:locations` 列出 |
| 部署目標 | Firebase Hosting（預設建議）／GitHub Pages／Cloudflare Pages／其他，以及網站的最終網域 |
| 圖片 | `images/Google_Play.png`、`images/App_Store.png`、`images/upload_info.jpg`；以及是否要替換 `helpImageUrl` 的取票教學圖 |

## 4. 前置檢查

```bash
node --version        # 建議 18 以上
git --version
npm install -g firebase-tools   # 或使用 npx firebase-tools
firebase --version
```

工作目錄：這個 repo 的根目錄（含 `index.html`）。若 `index.html` 不存在，停止並告知使用者。

## 5. 部署步驟

以下以 `<ID>` 代表 Firebase 專案 ID。

### 5.1 建立專案與 Web App

```bash
firebase login                                   # 人類完成互動登入
firebase projects:create <ID> --display-name "火車分票"
firebase apps:create WEB "train-ticket" --project <ID>      # 輸出會包含 App ID
firebase apps:sdkconfig WEB <APP_ID> --project <ID>         # 輸出 firebaseConfig
```

把 `apps:sdkconfig` 輸出的欄位填進 `CONFIG.firebase`：`apiKey`、`authDomain`、`projectId`、`storageBucket`、`messagingSenderId`、`appId`。`measurementId` 若沒有輸出就**移除該欄位或留空**（程式沒有使用它）。

> 如果使用者要用既有專案，跳過 `projects:create`。

### 5.2 建立 Firestore

```bash
firebase firestore:locations --project <ID>                  # 可選：列出可用區域
firebase firestore:databases:create '(default)' --location=asia-east1 --project <ID>
```

（以上指令已對照 firebase-tools 15.x 的 `--help` 確認存在。若使用者的 CLI 版本太舊或指令失敗，改請人類在 Console：Build → Firestore Database → Create database → Production mode。）

### 5.3 啟用 Google 登入（人類步驟）

請使用者開啟：

```
https://console.firebase.google.com/project/<ID>/authentication/providers
```

→ 點 **Google** → 啟用 → 選擇專案支援電子郵件 → 儲存。**等使用者確認後再繼續。**

### 5.4 編輯 `index.html`

1. 填寫 `CONFIG`：`users`、`outbound`、`return`、`schedule`、`firebase`（欄位格式見 README「CONFIG 設定參考」）。
   - `tickets` 的 key 必須是 `車廂號-座位號`，且與座位圖中每個實際座位一一對應。
2. **修改 CSP**：檔案最上方 `<meta http-equiv="Content-Security-Policy">` 的 `frame-src` 內，把
   `https://ticket-system--login.firebaseapp.com`
   **替換為** `https://<CONFIG.firebase.authDomain>`。不改的話 Google 登入彈出視窗無法運作。
3. （選用）替換頁面標題 `<title>`、`og:title`、`og:description`。
4. （建議）請使用者決定是否移除「干擾除錯」那段 `setInterval(... debugger ...)` 的 `<script>`；它只會擾亂開發者工具，不提供真正的安全性。

### 5.5 檢查設定並產生 Firestore 規則

把第 8 節的腳本存成 `/tmp/check-config.mjs`（**不要放進 repo**），然後：

```bash
node /tmp/check-config.mjs index.html --write-rules firestore.rules
```

- 腳本會檢查：座位與票券一一對應、座位無重複、Email 小寫、顯示名稱不重複／不含 `/`，並警告仍是預留值（`YOUR_...`）的欄位。
- **有任何 `✗` 錯誤就先修正，不要部署。** 預留值警告（`!`）在填完前可忽略，但**部署前必須全部消失**。
- 通過後會依 `CONFIG.users` 產生 `firestore.rules`。

建立 `firebase.json`（如果還沒有）：

```json
{
  "firestore": { "rules": "firestore.rules" },
  "hosting": {
    "public": ".",
    "ignore": [
      "firebase.json", "firestore.rules", "**/.*", "**/node_modules/**",
      "README.md", "AGENTS.md", "LICENSE"
    ]
  }
}
```

部署規則：

```bash
firebase deploy --only firestore:rules --project <ID>
```

### 5.6 本機測試

```bash
python3 -m http.server 8000
```

請使用者在 `http://localhost:8000` 測試登入。**不可用 `file://`**（Google 彈出式登入不支援）。`localhost` 預設在 Authorized domains 內。

### 5.7 部署網站

- **Firebase Hosting**：
  ```bash
  firebase deploy --only hosting --project <ID>
  ```
  網址通常為 `https://<ID>.web.app` 與 `https://<ID>.firebaseapp.com`，預設已在 Authorized domains。
- **GitHub Pages / Cloudflare Pages / 自訂網域**：照該平台流程發布，並請**使用者**把網站網域（例：`xxx.github.io`、`xxx.pages.dev`）加到 Authentication → Settings → Authorized domains。

部署前再次確認：這份要公開的 repo／網站**是否含有真實票券資料**（鐵則 2）。

## 6. 驗收

自動可做的：

```bash
# 未登入不得讀取個人資料（預期 403）
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://firestore.googleapis.com/v1/projects/<ID>/databases/(default)/documents/users/test?key=<API_KEY>"

# 未登入不得寫入座位表（預期 403）
curl -s -o /dev/null -w "%{http_code}\n" -X PATCH -H "Content-Type: application/json" -d '{"fields":{}}' \
  "https://firestore.googleapis.com/v1/projects/<ID>/databases/(default)/documents/config/seats?key=<API_KEY>"

# 網站可連線（預期 200）
curl -s -o /dev/null -w "%{http_code}\n" https://<網站網域>/
```

> 這些預期結果依規則邏輯推得；若得到非 403／200，請照實回報並排查，不要宣稱通過。

必須請使用者手動確認的：

- [ ] 白名單內帳號可登入並看到自己的顯示名稱
- [ ] 不在白名單的 Google 帳號會被登出，且登入頁顯示「此帳號不在參與者名單內」與該 Email
- [ ] 兩人同時選同一座位，只有一人成功；另一人的畫面即時更新
- [ ] 去程、返程的票券代碼與驗證碼正確（不是「未設定」）
- [ ] 上傳一張截圖成功並進入完成頁（建議用大於 1 MB 的圖，確認自動壓縮正常；上限說明見 README「已知限制」）
- [ ] 行程表 PDF 可預覽與下載

全部通過才算完成；回報時請列出：網站網址、Firebase 專案 ID、做了哪些人類步驟、哪些項目尚未驗證。

## 7. 疑難排解對照

| 現象 | 處理 |
|------|------|
| 登入視窗空白／失敗 | 檢查 CSP `frame-src` 是否為使用者自己的 `authDomain`；網域是否在 Authorized domains；是否用 `file://` |
| `auth/unauthorized-domain` | 請使用者新增 Authorized domain |
| `auth/operation-not-supported-in-this-environment` | 用 `file://` 開啟；改用 http(s) |
| 登入後立刻被登出（登入頁顯示「此帳號不在參與者名單內」） | Email 不在 `CONFIG.users`，或含大寫 |
| `Missing or insufficient permissions` | 規則的白名單與 `CONFIG.users` 不一致 → 重跑 5.5 並重新部署規則 |
| 票券顯示「未設定」 | `tickets` key 與座位 `車廂號-座位號` 不一致 |
| `firebase firestore:databases:create` 失敗 | 資料庫可能已存在，或 CLI 版本太舊；改用 Console |
| `projects:create` 失敗 | 專案 ID 已被使用或不合規；換一個 |

## 8. 設定檢查／規則產生腳本

存成 `/tmp/check-config.mjs`。用法：`node /tmp/check-config.mjs index.html [--write-rules firestore.rules]`。

```js
// 檢查 index.html 的 CONFIG：座位 ↔ 票券對應、Email 白名單、Firebase 設定是否仍為預留值
import { readFileSync, writeFileSync } from 'node:fs';
const html = readFileSync(process.argv[2] ?? 'index.html', 'utf8');
const m = html.match(/const CONFIG = (\{[\s\S]*?\n        \});\s*\n\s*\/\/ ╔/);
if (!m) { console.error('找不到 CONFIG 區塊'); process.exit(2); }
const CONFIG = new Function(`return (${m[1]})`)();
let errors = 0;
const err = (s) => { console.error('✗ ' + s); errors++; };
for (const trip of ['outbound', 'return']) {
  const seatIds = new Set();
  for (const row of CONFIG[trip].seats)
    for (const s of [...row.left, ...row.right])
      if (s && !s.isGhost) {
        const id = `${s.car}-${s.no}`;
        if (seatIds.has(id)) err(`${trip} 座位重複：${id}`);
        seatIds.add(id);
      }
  const ticketIds = new Set(Object.keys(CONFIG[trip].tickets));
  for (const id of seatIds) if (!ticketIds.has(id)) err(`${trip} 座位 ${id} 沒有對應的票券`);
  for (const id of ticketIds) if (!seatIds.has(id)) err(`${trip} 票券 ${id} 沒有對應的座位`);
  console.log(`${trip}: ${seatIds.size} 個座位, ${ticketIds.size} 筆票券`);
}
const emails = Object.keys(CONFIG.users);
for (const e of emails) if (e !== e.toLowerCase()) err(`Email 必須全小寫：${e}`);
const names = Object.values(CONFIG.users);
if (new Set(names).size !== names.length) err('顯示名稱不可重複（Firestore 文件 ID 使用顯示名稱）');
for (const n of names) if (n.includes('/')) err(`顯示名稱不可含「/」：${n}`);
console.log(`users: ${emails.length} 位`);
const fb = CONFIG.firebase;
for (const [k, v] of Object.entries(fb)) if (/YOUR_/.test(v)) console.warn(`! firebase.${k} 仍為預留值`);
if (/YOUR_FILE_ID/.test(CONFIG.schedule.pdfUrl)) console.warn('! schedule 仍為預留值');
console.log(errors ? `\n發現 ${errors} 個問題` : '\n結構檢查通過');

// 選用：--write-rules <檔案>  依 CONFIG.users 產生 Firestore 安全規則
const wi = process.argv.indexOf('--write-rules');
if (wi > -1 && !errors) {
  const out = process.argv[wi + 1] ?? 'firestore.rules';
  const entries = Object.entries(CONFIG.users)
    .map(([e, n]) => `        ${JSON.stringify(e.toLowerCase())}: ${JSON.stringify(n)}`).join(',\n');
  writeFileSync(out, `rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function whitelist() {
      return {
${entries}
      };
    }
    function isMember() {
      return request.auth != null
          && request.auth.token.email_verified == true
          && request.auth.token.email.lower() in whitelist();
    }
    function myName() {
      return whitelist()[request.auth.token.email.lower()];
    }

    match /config/seats {
      allow read: if true;
      allow create, update: if isMember();
      allow delete: if false;
    }

    match /users/{userId} {
      allow read: if isMember() && myName() == userId;
      allow create, update: if isMember() && myName() == userId
        && (!("screenshotBase64" in request.resource.data)
            || request.resource.data.screenshotBase64.size() < 1000000);
      allow delete: if false;
    }
  }
}
`);
  console.log('已寫入 ' + out);
}
process.exit(errors ? 1 : 0);
```

## 9. 不要做的事（再次提醒）

- 不要放寬 Firestore 規則來「讓它先動起來」。
- 不要修改 `CONFIG` 以外的邏輯來「修正」使用者的資料問題（例如替他補不存在的票券）。
- 不要在對話、commit 訊息或日誌中輸出使用者的票券代碼與 Email 清單。
- 不要在沒有同意的情況下刪除 Firestore 資料或 Firebase 專案。

