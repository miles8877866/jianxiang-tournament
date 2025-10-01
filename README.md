# 健祥盃賽程表 – 部署與使用說明

這份文件說明如何將 Google Sheet 中的比賽資料，透過 Google Apps Script 匯出 JSON，並利用 GitHub Pages 安全地展示網頁。

---

## 🔧 前置需求
- 已建立一份 Google Sheet（賽程/名次資料）。  
- 已撰寫好 Apps Script，可將工作表輸出為 JSON。  
- GitHub 帳號 + Repo（要啟用 GitHub Pages）。  

---

## 🚀 Step 1：部署 Google Apps Script
1. 在 [Google Apps Script 編輯器](https://script.google.com/) 打開專案。  
2. 在 `doGet(e)` 加上「密鑰檢查」：
   ```js
   function doGet(e) {
     var key = e && e.parameter && e.parameter.key;
     if (key !== 'YOUR_SECRET') {
       return ContentService.createTextOutput(JSON.stringify({ error: 'Forbidden' }))
         .setMimeType(ContentService.MimeType.JSON);
     }
     // 其餘程式碼：輸出 JSON
   }
   ```
3. 部署為 Web App → 「任何人」皆可存取。  
4. 複製 exec URL，例如：
   ```
   https://script.google.com/macros/s/xxxxxx/exec?key=YOUR_SECRET
   ```

---

## 🗄 Step 2：建立 GitHub Repo 與 Pages
1. 建立 Repo，例如 `go-cup-page`。  
2. 新增：
   - `index.html`（前端頁面）  
   - `data.json`（先放 `[]` 空陣列）  
3. 到 **Settings → Pages** 啟用 GitHub Pages。  
   - 完成後會得到公開網址，例如 `https://帳號.github.io/go-cup-page/`。

---

## 🔑 Step 3：設定 GitHub Secrets
1. 到 **Settings → Secrets and variables → Actions → New repository secret**。  
2. 新增一個 Secret：  
   - **Name**：`GAS_URL`  
   - **Value**：`https://script.google.com/macros/s/xxxxxx/exec?key=YOUR_SECRET`

---

## ⚡ Step 4：新增 GitHub Actions Workflow
在 repo 建立 `.github/workflows/export-gas-to-json.yml`：

```yaml
name: Export GAS to data.json
on:
  schedule:
    - cron: '0 1 * * *'   # 每天台灣早上 9 點（UTC 1 點）
  workflow_dispatch: {}    # 手動觸發

permissions:
  contents: write

jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Fetch GAS JSON
        run: |
          curl -fsSL "${{ secrets.GAS_URL }}" -o data.json
          if command -v jq >/dev/null 2>&1; then jq . data.json > /tmp/tidy.json && mv /tmp/tidy.json data.json; fi

      - name: Commit & Push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add data.json
          git commit -m "Update data.json" || echo "no changes"
          git push
```

---

## 🖥 Step 5：修改前端 index.html
在 HTML 中，把資料來源改成讀本地的 `data.json`：

```html
<script>
const DATA_URL = "./data.json";

async function loadData(){
  try {
    const res = await fetch(DATA_URL, { cache: "no-store" });
    const data = await res.json();
    renderTable(Array.isArray(data) ? data : []);
    document.getElementById("lastUpdated").textContent = new Date().toLocaleString("zh-TW");
  } catch (err) {
    console.error(err);
    document.getElementById("statusMsg").textContent = "讀取失敗";
  }
}
document.addEventListener("DOMContentLoaded", loadData);
</script>
```

---

## ✅ Step 6：驗證流程
1. Push 到 main 分支 → Pages 更新。  
2. 到 Repo → Actions → 手動觸發 workflow → 更新 `data.json`。  
3. 打開 Pages 網址，確認表格顯示最新資料。  

---

## 📌 注意事項
- Apps Script URL **不會出現在前端**，只存在 GitHub Secret 中。  
- `data.json` 是公開的，任何人可存取 → 請確保其中不含敏感資訊。  
- `cron` 時間為 UTC；台灣時間 = UTC+8。  
- 可以依需求調整排程，例如每 30 分更新一次：  
  ```yaml
  - cron: '*/30 * * * *'
  ```

---

✦ 到這裡，你就完成了從 Google Sheet → Apps Script → GitHub Actions → GitHub Pages 的安全自動化流程！
