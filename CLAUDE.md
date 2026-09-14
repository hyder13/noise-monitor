# noise-monitor — 專案事實卡

住宅低頻噪音與衝擊音的事件記錄工具。單檔網頁（`index.html`），無建置流程。

## 目的（會影響取捨）

hyder 家中疑似有兩種噪音，要蒐證跟建商／設計公司溝通：

1. **水錘／管路共振** — 懷疑樓上用水造成，低頻持續型
2. **客廳「趴」一聲** — 單次衝擊型，疑似系統櫃或木作熱脹冷縮

因此**取捨一律偏向舉證強度**：寧可多存原始資料（WAV 不壓縮、頻譜快照），
也不要為了省空間而失去證據。所有判定都要標明依據與限制，
因為這份資料會被建商檢視。

## 部署

| 目標 | 網址 | 備註 |
|---|---|---|
| 主要 | https://hyder13.github.io/noise-monitor/ | GitHub Pages，手機用這個 |
| Repo | https://github.com/hyder13/noise-monitor | public（Pages 免費方案需要） |
| 備用 | Artifact v5 | 需登入 Claude 帳號 |

同一份 `index.html` 同時服務兩邊，所以**不能有 `<html>`／`<head>`／`<body>` 標籤**
（Artifact 格式限制），meta 標籤直接寫在檔案開頭，瀏覽器會自動解析進 head。

## 雷區（實測踩過）

1. **【實測 2026-09-14】缺 viewport meta → 手機上全部字體極小。**
   Artifact 環境會自動補這個標籤，所以只有 GitHub Pages 版中招。
   症狀是 `document.documentElement.clientWidth` 回 **981**（桌面預設寬度）而非裝置寬度。
   量到 981 就是這個原因，不要誤判成瀏覽器模擬失效（我犯過）。

2. **【實測 2026-09-14】canvas 尺寸會逐幀放大。**
   `fitCanvas` 若讀 `getAttribute('height')` 當目標高度，而設定 `cv.height` 會改寫同一屬性
   → 每次重繪再乘一次 dpr。dpr=1 的桌面看不出來，手機 dpr=2 會崩潰。
   目標高度一律存 `data-h`，JS 永不寫入該屬性。

3. **【實測 2026-09-14】GitHub Pages 的 build 等待不能只看 `status=built`。**
   push 後立即查到的是**上一次**建置的狀態，迴圈會馬上結束。
   正確做法：比對 `pages/builds/latest` 的 `.commit` 與 `git rev-parse HEAD`，
   兩者相同且 status=built 才算完成。之後 CDN 還有 `max-age=600`，
   驗證線上內容要用 cache-buster 參數。

4. **Artifact 沙箱不給下載權限。**
   `<a download>` 與 script 觸發的儲存對觀看者完全無效。
   必須宣告 `capabilities: {downloads: true}` 並走 `window.claude.use('downloads')`。
   允許的副檔名不含 **wav**，所以音檔一律包進 zip 再給（`makeZip`，store 模式）。
   非 Artifact 環境（GitHub Pages、日後的 App）會自動退回 `<a download>`。

5. **`text-transform: uppercase` 會毀掉聲學單位**（dB→DB、Hz→HZ）。整份檔案不使用。

## 量測邏輯的關鍵決定（別隨意改）

- **絕對值不可信，相對量可信**。手機麥克風未經聲學校準，且低頻（<80 Hz）有滾降。
  但 **C−A 差值、窄峰位置、峰值因數、事件時間分布**不受校準影響——
  這幾樣才是這個工具的價值所在。任何 UI 改動都不能弱化這個訊息。
- **A 加權會把水錘共振壓掉近 30 dB**。主讀數一律用 dB(Z)，不要改成 dB(A)。
- **衝擊音必須獨立偵測**。「趴」一聲只有 20–50 ms，撐不到持續時間門檻。
  峰值追蹤在 PCM 層逐取樣進行（`onPcm`），不能只靠每 80 ms 一次的 FFT 窗。
- **錄音要前置緩衝**。等偵測到才錄就錄不到起始那一下，`ring` 保留 8 秒滾動緩衝。
- **窄峰判定門檻依頻率而異**（NSW 準則）：25–125 Hz 需高出鄰帶 15 dB、160–400 Hz 8 dB。
- **60 Hz 及其倍頻要標假陽性警告**。台灣電網 60 Hz，冰箱與加壓馬達是頭號誤判來源。

## 驗證方式

無測試框架。改動後至少做：

```bash
# 抽出 JS 做語法檢查
python -c "import re,io; s=io.open('index.html',encoding='utf-8').read(); \
io.open('check.js','w',encoding='utf-8').write(re.search(r'<script>(.*)</script>',s,re.S).group(1))"
node --check check.js
```

介面改動要在瀏覽器實看（`mcp__Claude_Browser__`），並確認：
`document.documentElement.clientWidth` 等於裝置寬度、canvas 連續重繪後尺寸穩定。

## 相關文件

- `README.md` — 功能、量測前提、對照實驗設計
- `docs/舉證指南.md` — 法條、求償流程、證據位階、SOP

## 待辦

- [ ] 加速度計同步記錄結構震動（證明結構傳導而非空氣傳音）
- [ ] DEFRA NANR45 低頻準則曲線疊在事件頻譜上
- [ ] Capacitor 打包 APK／iOS，取得真正的背景執行能力
