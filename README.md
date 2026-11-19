# 🎫 火車選位分票系統（Google 登入版）

一個給「團體購票」使用的協作小工具：由一個人把整批車票訂好之後，成員各自用 Google 帳號登入，**自己挑座位、取得該座位的訂票代碼與取票驗證碼**，再到各自的手機 App 完成取票。

整個系統是**單一 `index.html`**，沒有打包工具、沒有自己的伺服器，後端完全交給 Firebase（Authentication + Cloud Firestore）。

> 這是 [`train-ticket-system`](https://github.com/Twilight-Strider/train-ticket-system) 的後續版本。舊版以「選名字」的方式辨識身分；本版改為 **Google 登入 + Email 白名單**，並搭配對應的 Firestore 安全規則。

---

## 目錄

- [功能與使用流程](#功能與使用流程)
- [運作原理](#運作原理)
- [技術架構](#技術架構)
- [建立你自己的版本](#建立你自己的版本)
- [CONFIG 設定參考](#config-設定參考)
- [管理與維運](#管理與維運)
- [安全性與隱私（請務必閱讀）](#安全性與隱私請務必閱讀)
- [已知限制](#已知限制)
- [疑難排解](#疑難排解)
- [專案結構](#專案結構)
- [讓 AI Agent 幫你部署](#讓-ai-agent-幫你部署)
- [第三方素材與商標聲明](#第三方素材與商標聲明)
- [授權](#授權)

---

## 功能與使用流程

使用者開啟網頁後，依序經過五個步驟（進度會存在資料庫，重新整理或換裝置登入後會回到上次的步驟）：

| 步驟 | 畫面 | 說明 |
|------|------|------|
| 0 | Google 登入 | 以 Google 帳號登入；Email 不在白名單內會立即被登出，並在登入頁顯示「此帳號不在參與者名單內」與該 Email |
| 1 | 安裝 App | 提供 Google Play / App Store 連結，安裝台鐵 e 訂通 |
| 2 | 去程選位 | 視覺化座位圖，點選座位後即時鎖定；下一頁顯示該座位的訂票代碼與取票驗證碼（可一鍵複製） |
| 3 | 返程選位 | 同上，針對返程車次 |
| 4 | 上傳截圖 | 上傳 App 取票完成的截圖，作為確認；過大的圖片會自動壓縮 |
| 5 | 完成 | 內嵌顯示行程表 PDF（Google Drive），並提供下載按鈕 |

其他行為：

- **即時同步**：任何人選位，其他人的畫面立刻更新（已被他人選走的座位會變綠色且不可點）。
- **搶位保護**：選位使用 Firestore Transaction，兩個人同時點同一個座位時，只有一個人會成功。
- **可換位**：同一個人再選另一個座位時，舊座位會自動釋放。
- **斷點續填**：步驟、已選座位都存在 `users/{顯示名稱}`，隨時可回來繼續。

---

## 運作原理

### 資料模型（Cloud Firestore）

```
config/seats                      ← 全體共用的座位認領表
  outbound: { "12-34": "使用者 A", "12-36": "使用者 B", ... }
  return:   { "4-37":  "使用者 A", ... }

users/{顯示名稱}                   ← 每位參與者一份（文件 ID = CONFIG.users 裡的顯示名稱）
  name, email, step
  outboundSelection, returnSelection        例："12-34"
  screenshotBase64, screenshotName,
  screenshotOriginalSize, screenshotFinalSize, uploadedAt
  createdAt
```

- 座位識別碼的格式固定為 **`車廂號-座位號`**（例如 `12-34`），`CONFIG.*.tickets` 的 key 也必須使用同樣格式。
- 截圖以 **Base64 字串直接存在 Firestore 文件裡**（沒有使用 Firebase Storage），因此不需要額外開啟 Storage。

### 身分驗證與授權

1. 前端使用 `signInWithPopup` + Google Provider，每次都會要求選擇帳號。
2. 登入後，前端用 `CONFIG.users`（Email → 顯示名稱）檢查白名單；不在名單內就立刻 `signOut()`。
3. **真正的防線是 Firestore 安全規則**（見下方），前端白名單只負責畫面流程。

---

## 技術架構

| 技術 | 用途 |
|------|------|
| React 18（CDN, UMD） | 前端 UI |
| Babel Standalone | 在瀏覽器內即時編譯 JSX |
| Tailwind CSS（CDN） | 樣式 |
| Firebase JS SDK 9.22.0（compat） | Authentication、Cloud Firestore |
| Google Drive 預覽連結 | 行程表 PDF 顯示與下載 |

零建置、零伺服器：任何能提供靜態檔案的主機都能部署。

---

## 建立你自己的版本

整個流程大約 20～30 分鐘。你需要：一個 Google 帳號、（建議）Git、以及一個能放靜態檔案的主機（Firebase Hosting / GitHub Pages / Cloudflare Pages 皆可）。

### 步驟 1：取得程式碼

```bash
git clone https://github.com/Twilight-Strider/train-ticket-system_Google-login-version.git
cd train-ticket-system_Google-login-version
```

也可以直接在 GitHub 上 Fork 這個 repo。

### 步驟 2：建立 Firebase 專案

1. 前往 [Firebase Console](https://console.firebase.google.com/)，點「建立專案」。
2. 名稱自訂；Google Analytics 可以不啟用（本專案沒有使用）。

### 步驟 3：啟用 Google 登入

1. Console 左側 **Build → Authentication → Get started**（或「開始使用」）。
2. 切到 **Sign-in method**，選 **Google**，啟用並選擇專案的支援電子郵件，儲存。
3. 切到 **Settings → Authorized domains（已授權網域）**，確認之後會用來開啟網頁的網域都在清單裡：
   - `localhost`、`<專案ID>.firebaseapp.com`、`<專案ID>.web.app` 通常預設就有。
   - 如果你用 GitHub Pages（`xxx.github.io`）、Cloudflare Pages（`xxx.pages.dev`）或自己的網域，**必須手動新增**，否則登入視窗會報錯 `auth/unauthorized-domain`。

### 步驟 4：建立 Firestore 並設定安全規則

1. **Build → Firestore Database → Create database**，選一個離使用者近的區域（台灣可選 `asia-east1`），模式選 **Production mode（正式版模式）**。
2. 進入 **Rules** 分頁，貼上下面的規則，並把 `whitelist()` 裡的內容換成**和 `CONFIG.users` 完全相同**的 Email → 顯示名稱（Email 一律小寫），按「發布」。

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ⚠️ 必須與 index.html 的 CONFIG.users 保持一致（Email 請用小寫）
    function whitelist() {
      return {
        "alice@gmail.com": "使用者 A",
        "bob@gmail.com":   "使用者 B",
        "carol@gmail.com": "使用者 C"
      };
    }

    // 已登入、Email 已驗證、且在白名單內
    function isMember() {
      return request.auth != null
          && request.auth.token.email_verified == true
          && request.auth.token.email.lower() in whitelist();
    }

    // 目前登入者對應的顯示名稱
    function myName() {
      return whitelist()[request.auth.token.email.lower()];
    }

    // 座位認領表：任何人可讀（只含座位與顯示名稱），只有白名單成員可寫，禁止刪除
    match /config/seats {
      allow read: if true;
      allow create, update: if isMember();
      allow delete: if false;
    }

    // 個人資料：只有本人可讀寫；截圖字串限制在 1,000,000 字元內
    match /users/{userId} {
      allow read: if isMember() && myName() == userId;
      allow create, update: if isMember() && myName() == userId
        && (!("screenshotBase64" in request.resource.data)
            || request.resource.data.screenshotBase64.size() < 1000000);
      allow delete: if false;
    }
  }
}
```

> **為什麼 `config/seats` 是公開可讀？** 前端在登入**之前**就會開始監聽這份文件；若讀取需要登入，監聽會在登入前就被拒絕並中止，登入後不會自動重試。這份文件只包含「座位編號 → 顯示名稱」，不含票券代碼與 Email。如果你介意，可以自行調整前端改為登入後才監聽，再把 `read` 改成 `isMember()`。
>
> 這份規則是依程式碼的讀寫行為推導出來的，**請在部署後依[驗收清單](#部署後驗收清單)實際測過**。

### 步驟 5：註冊 Web App 並取得設定

1. **專案設定（齒輪）→ 一般 → 你的應用程式 → `</>`（Web）**，註冊一個 Web App。
2. 複製畫面上的 `firebaseConfig`，下一步會貼進 `index.html`。
   （`measurementId` 只有啟用 Analytics 才會出現；本專案沒有使用，沒有也可以。）

### 步驟 6：編輯 `index.html` 的 `CONFIG`

打開 `index.html`，找到檔案上方標示「🔧 每次複製使用前，只需修改這個區塊」的 `CONFIG`，依 [CONFIG 設定參考](#config-設定參考) 填寫：參與者 Email 白名單、去程／返程的車次資訊與座位圖、每個座位的訂票代碼與驗證碼、行程表 PDF 連結、Firebase 設定。

### 步驟 7：修改內容安全策略（CSP）裡寫死的網域 ⚠️

`index.html` 最上方有一段 CSP `<meta>`，其中 `frame-src` 寫死了**原作者專案**的登入網域：

```
https://ticket-system--login.firebaseapp.com
```

請把它換成**你自己的 `authDomain`**（也就是 `CONFIG.firebase.authDomain` 的值，例如 `my-trip-2027.firebaseapp.com`）。**沒改的話，Google 登入視窗無法正常運作。**

> 若你之後在 CSP 之外又引入新的外部資源（字型、圖片 CDN、其他腳本），也要記得到這段 CSP 補上對應網域。

### 步驟 8：準備圖片

程式會讀取三張圖片，請放在與 `index.html` 同層的 `images/` 資料夾：

| 檔案 | 用在哪裡 |
|------|----------|
| `images/Google_Play.png` | 步驟 1 的 Google Play 連結圖 |
| `images/App_Store.png` | 步驟 1 的 App Store 連結圖 |
| `images/upload_info.jpg` | 步驟 4「上傳說明」彈窗裡的說明圖（自己製作） |

商店徽章請從 [Google Play 品牌資源](https://partnerdash.google.com/apps/brand-resources)、[Apple Marketing Resources](https://developer.apple.com/app-store/marketing/guidelines/) 取得並遵守其使用規範。圖片也可以改放在 Cloudflare R2 等 CDN，把程式中對應的 `src` 換成完整網址即可（網域需同時加進 CSP 的 `img-src`；目前 `img-src` 已允許任何 `https:`）。

另外，步驟 2、3 的「App 取票教學」圖目前是**直接引用第三方網站的圖片網址**（見 `helpImageUrl`），建議換成你自己製作的圖片，原因見[第三方素材與商標聲明](#第三方素材與商標聲明)。

### 步驟 9：本機測試

⚠️ **不能直接雙擊開啟 `index.html`**（`file://`）。Firebase 的 Google 彈出式登入不支援 `file://`，會出現 `auth/operation-not-supported-in-this-environment`。請用本機伺服器：

```bash
python3 -m http.server 8000
# 瀏覽器開啟 http://localhost:8000
```

`localhost` 預設在授權網域清單內，所以不需另外設定。

### 步驟 10：部署

任選一種靜態主機：

**Firebase Hosting（最省事，登入網域自動授權）**

```bash
npm install -g firebase-tools
firebase login
firebase init hosting     # public 目錄填 . ；不要覆寫 index.html；不用設為單頁應用
firebase deploy --only hosting
```

> 部署時請確認 `README.md`、`AGENTS.md`、`LICENSE` 等不想公開的檔案已列在 `firebase.json` 的 `hosting.ignore`。

**GitHub Pages / Cloudflare Pages**：照平台指示發布，並回到步驟 3 把網站網域加入 Authorized domains。

### 部署後驗收清單

- [ ] 用白名單內的帳號登入成功，看得到自己的顯示名稱。
- [ ] 用**不在**白名單的 Google 帳號登入，會被立刻登出，登入頁出現「此帳號不在參與者名單內」的提示，無法進入。
- [ ] 用兩個帳號（或兩台裝置）同時點同一個座位，只有一個人成功，另一人收到「此座位已被 XXX 選取」。
- [ ] 選位後，另一個帳號的畫面即時看到該座位變綠色。
- [ ] 去程與返程的訂票代碼與驗證碼和你填的一致（若顯示「未設定」，代表 `tickets` 的 key 與座位 `車廂-座位號` 對不上）。
- [ ] 上傳一張截圖成功並進入完成頁（建議用一張大於 1 MB 的圖，確認自動壓縮正常）。
- [ ] **規則測試**：未登入狀態下直接呼叫 Firestore REST API 寫入或讀取 `users/…`，應該回傳 403：

  ```bash
  curl -s -o /dev/null -w "%{http_code}\n" \
    "https://firestore.googleapis.com/v1/projects/<專案ID>/databases/(default)/documents/users/test?key=<apiKey>"
  # 預期輸出 403
  ```

---

## CONFIG 設定參考

### 1. `users` — 參與者名單（Email 白名單）

```javascript
users: {
    "alice@gmail.com": "使用者 A",   // key：Google 帳號 Email（小寫）；value：顯示名稱
    "bob@gmail.com":   "使用者 B",
},
```

- 比對時會把登入者的 Email 轉成小寫，所以 **key 必須是小寫**。
- 顯示名稱會當作 Firestore `users/{名稱}` 的**文件 ID** 與座位表裡的擁有者名稱：**不可重複、不可包含 `/`**。
- 修改名單時，**記得同步更新 Firestore 規則裡的 `whitelist()`**。

### 2. `outbound` / `return` — 去程與返程

```javascript
outbound: {
    info: ["車次：自強 XXX", "車型：EMU3000", "乘車時間：09:00 - 11:30"],  // 顯示在選位頁左上角

    seats: [
        {
            id: 'row1',
            left:  [{ isGhost: true },       { no: '40', car: '12' }],
            right: [{ no: '39', car: '12' }, { no: '37', car: '12' }],
        },
        {
            id: 'row2',
            left:  [{ no: '34', car: '12' }, { no: '36', car: '12' }],
            right: [null, null],
        },
    ],

    tickets: {
        "12-34": { code: "訂票代碼", verify: "取票驗證碼" },
        "12-36": { code: "訂票代碼", verify: "取票驗證碼" },
        // ……每個實際座位一筆
    },
},
```

座位圖每一列有左側 2 格、右側 2 格，每一格可以是：

| 值 | 意義 |
|----|------|
| `{ no: '座位號', car: '車廂號' }` | 實際座位，可點選 |
| `{ isGhost: true }` | 空白佔位框（用來在視覺上補齊排列） |
| `null` | 完全空白（沒有座位） |

規則：

- 每個實際座位在 `tickets` 裡必須有一筆，key 為 `車廂號-座位號`（兩邊的字串要完全一致，包含前導零）。
- `return`（返程）的格式與 `outbound` 完全相同。

### 3. `schedule` — 行程表 PDF

```javascript
schedule: {
    pdfUrl:      "https://drive.google.com/file/d/<檔案ID>/preview",
    downloadUrl: "https://drive.google.com/u/0/uc?id=<檔案ID>&export=download",
},
```

把 PDF 上傳到 Google Drive，共用設定改為「知道連結的任何人可檢視」，再把檔案 ID 填入兩個連結。

### 4. `firebase` — 步驟 5 取得的設定

`apiKey`、`authDomain`、`projectId`、`storageBucket`、`messagingSenderId`、`appId`（`measurementId` 選填）。

---

## 管理與維運

- **查看誰選了哪個座位**：Console → Firestore → `config/seats`。
- **重置全部座位**：在 Console 刪除 `config/seats` 整份文件（規則禁止從前端刪除，Console 不受限）；第一次有人選位時會自動重建。
- **重置某人的進度**：在 Console 刪除 `users/{顯示名稱}` 文件。
- **檢視截圖**：`users/{名稱}.screenshotBase64` 已經是完整的 `data:image/jpeg;base64,…` 字串，可貼到瀏覽器網址列或轉成圖片檢視。
- **活動結束後**：請刪除 `users` 集合與 `config/seats`，並停用或刪除整個 Firebase 專案／網站。

---

## 安全性與隱私（請務必閱讀）

這個專案是為「一小群互相認識的人、短期使用」設計的，**不是**高安全等級的系統。請了解以下事實：

1. **票券代碼寫在前端檔案裡。** 訂票代碼與取票驗證碼放在 `index.html` 的 `CONFIG.*.tickets`，**任何能打開你部署網址的人，都可以用「檢視原始碼」看到全部票券資料**，不需要登入。Email 白名單與 Firestore 規則保護的是資料庫，**保護不了寫死在網頁裡的內容**。
   - 請**不要把填入真實票券資料的 `index.html` 推到公開的 repo**。
   - 部署網址只分享給參與者；活動結束後立刻下架。
   - 需要更高保護，可以考慮放在有存取控制的主機（例如 Cloudflare Access），或自行改為把票券資料存進受規則保護的 Firestore（需要修改程式）。
2. **前端的「防止篡改」只是嚇阻，不是安全機制。** 頁面包含凍結 `console`／`JSON`、定時 `debugger` 的干擾除錯程式碼，懂的人可以輕易繞過；它也會讓開發者工具一開就被中斷。開發階段可以自行移除那兩段 `<script>`。
3. **Firebase `apiKey` 不是機密**（Firebase 官方文件如此說明），可以存在前端；真正的防護來自 Authentication 與 Firestore 規則，所以**規則一定要上線**，不要使用「測試模式」。
4. **白名單成員彼此信任。** 規則允許任何白名單成員修改 `config/seats`，因此惡意的成員理論上能覆寫別人的座位認領。若成員間不互信，需要更嚴格的規則或後端驗證。
5. **截圖含個人購票資訊**，且儲存在 Firestore 中；規則限制只有本人可以讀取，但你自己（專案擁有者）在 Console 看得到。

---

## 已知限制

- **截圖有大小上限。** 截圖以 Base64 存在 Firestore，單一文件上限為 1 MiB。程式會在原檔超過 500 KB 時自動壓縮成 JPEG（必要時再縮小尺寸），把結果控制在約 700,000 字元以內；若仍壓不下來（極端大圖）會提示更換圖片。上限數值定義在 `Step4_Upload` 的 `MAX_BASE64_CHARS` 與 `NO_COMPRESS_BYTES`，改動時請同步檢查 Firestore 規則裡的 1,000,000 字元限制。
- 只支援兩段行程（去程＋返程），且每列固定為「左 2 格＋右 2 格」的座位配置。
- 一個人在去程與返程各只能選一個座位。
- React／Babel 以 CDN 載入並在瀏覽器內即時編譯，首次載入較慢，且 CSP 需要允許 `unsafe-eval`。
- 程式載入了 Firebase App Check 的 SDK，但並沒有啟用它。

---

## 疑難排解

| 現象 | 可能原因與解法 |
|------|----------------|
| 點「使用 Google 帳號登入」後視窗空白或登入失敗 | 1) CSP `frame-src` 沒改成你自己的 `authDomain`（步驟 7）；2) 目前網址不在 Authorized domains（步驟 3）；3) 用 `file://` 開啟（步驟 9） |
| `auth/unauthorized-domain` | 把網站網域加到 Authentication → Settings → Authorized domains |
| `auth/popup-blocked` | 瀏覽器封鎖彈出視窗，允許後重試 |
| 登入後馬上被登出，登入頁顯示「此帳號不在參與者名單內」 | Email 不在 `CONFIG.users`，或 key 含大寫（必須全小寫）；請確認對方用的是名單上的那個 Google 帳號 |
| 主控台出現 `Missing or insufficient permissions` | Firestore 規則的 `whitelist()` 與 `CONFIG.users` 不一致，或規則尚未發布 |
| 座位沒有即時更新 | 檢查 `config/seats` 的讀取規則；檢查 `firebase` 設定是否填對 |
| 票券顯示「未設定」 | `tickets` 的 key 與座位的 `車廂號-座位號` 不一致 |
| 上傳截圖失敗 | 網路中斷，或圖片極大壓不下來（見[已知限制](#已知限制)），換一張較小的截圖 |
| 圖片破圖 | `images/` 資料夾缺檔，或沒有一起部署 |
| 行程表 PDF 打不開 | Drive 檔案沒有開放「知道連結的任何人可檢視」，或連結中的檔案 ID 錯誤 |

---

## 專案結構

```
.
├── index.html     # 主程式（單一檔案，設定都在上方的 CONFIG）
├── images/
│   ├── Google_Play.png
│   ├── App_Store.png
│   └── upload_info.jpg
├── AGENTS.md      # 給 AI Agent 的自動部署說明
├── LICENSE
└── README.md
```

---

## 讓 AI Agent 幫你部署

如果你使用 Claude Code、Codex 等能操作終端機的 AI Agent，可以請它先閱讀 [`AGENTS.md`](AGENTS.md)，它會說明需要向你蒐集哪些資訊、哪些步驟必須由你親自在瀏覽器完成（例如登入 Google、啟用 Google 登入方式），以及如何驗收。

---

## 第三方素材與商標聲明

- 本專案與台灣鐵路公司（臺鐵）、Google、Apple **沒有任何關聯**。「台鐵 e 訂通」、Google Play、App Store 等名稱與標誌為其各自所有者的商標或著作。
- MIT 授權只涵蓋本 repo 中**本專案自己的程式碼與文件**。商店徽章圖片、截圖、以及程式中引用的第三方圖片（例如 `helpImageUrl` 所指向的外部教學圖）**不在授權範圍內**，其著作權屬於原作者；散布自己的版本前，請替換成你自己製作或有權使用的素材。
- 程式使用的開源函式庫以 CDN 方式載入、不隨本 repo 散布：React（MIT）、Babel（MIT）、Tailwind CSS（MIT）、Firebase JS SDK（Apache-2.0）。

---

## 授權

本專案以 [MIT License](LICENSE) 釋出，你可以自由使用、修改、散布，包含商業用途，只需保留版權與授權聲明。軟體依「現狀」提供，不附任何擔保。
