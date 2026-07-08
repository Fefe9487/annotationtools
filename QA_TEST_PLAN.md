# 標註系統 QA 測試計畫（annotation_tools/index.html）

> 目的：交給 Auto 模式（或另一位工程師）執行的可操作測試與修正清單。
> 主檔案：`annotation_tools/index.html`（Vue 3 + Tailwind CDN 單機版批註系統，資料存於 IndexedDB `ZhianAnnotationLargeDB`）。
> 使用者回報：（1）手機上操作可能不好用；（2）JS 好像有 error。

---

## ✅ 已執行測試結果（2026-07-08，手機模擬 390×844，demo 前 3 筆）

| 項目 | 結果 |
|------|------|
| 初次載入 / 從 IndexedDB 還原 | **0 error**（通過） |
| Page 1→4 逐頁前進、上一頁 | **0 error**（通過） |
| Page 2「部分法規不適當」勾選 + 未填建議前進 | 正確跳驗證 alert、擋下前進、**0 error**（通過） |
| Page 2「所有法規均不適當」（案件**有** processed_result） | **0 error**（通過） |
| **Page 2「所有法規均不適當」（案件缺 processed_result）** | ❌ **確認丟錯**（見下方 CONFIRMED BUG） |
| Page 3 表格手機版 | ⚠️ 無水平溢出但欄位被壓縮、標題文字直排、非常擁擠（需改善，見 FIX-1） |
| sticky footer 遮擋 | ⚠️ Page 1 底部 radio 點擊被 footer 攔截（需修，見 FIX-2） |

### 🔴 CONFIRMED BUG（最可能就是使用者說的「JS error」）
- **重現**：當前案件物件**沒有 `processed_result` 欄位**時，在 Page 2 點選「所有法規均不適當」。
- **實測 console 輸出**：
  ```
  [Vue warn]: Unhandled error during execution of native event handler  at <App>
  Script error.
  ```
- **根因**：`handleLawStatusChange()`（約 957 行）`currentCase.value.processed_result.map(...)` 未防呆；`syncCurrentPage2Answers()`（約 701 行）與 `watch(currentCaseIndex)`（約 793 行 `oldCase.processed_result.map`）同樣寫法。
- **重現用 JS（在正常載入且進到 Page 2 後執行）**：
  ```js
  const ss=document.querySelector('#app').__vue_app__._instance.setupState;
  delete ss.cases[ss.currentCaseIndex].processed_result;   // 模擬缺欄位的真實資料
  [...document.querySelectorAll('input[type=radio]')].find(x=>x.value==='所有法規均不適當').click();
  // → 觸發 Unhandled error
  ```
- **修法**：三處都改成 `(currentCase.value.processed_result || [])` / `(oldCase.processed_result || [])`。**（原 §3.1 由「疑似」升級為「已確認」，對應 FIX-3。）**

---

## 0. 背景與目前結論（前一輪已查到的事實）

- 本次調查在**手機模擬（390×844）**下進行。
- **已驗證：頁面「初次載入」與「從 IndexedDB 還原既有進度」時，console 皆 0 錯誤。**
  - 亦即最初懷疑的「`answers` computed 在 `restoreFromDB` 期間短暫為 `null` 導致渲染錯誤」**未重現**，先不要據此修改。
- **手機可用性問題（已觀察到 2 項，需修正）：**
  1. **Page 3「危害特徵標籤」使用固定寬度 `<table>`**（`w-28 / w-64 / w-48`，外層僅 `overflow-x-auto`）。手機上被迫水平捲動、欄位擁擠、radio 與輸入框難操作。
  2. **sticky footer（上一頁 / 下一頁）在窄視窗會蓋住底部內容**。實測在 Page 1 點選靠近底部的 radio 時，點擊被 footer 攔截（"Click intercepted by `<footer>`"）。代表 `<main>` 底部未預留 footer 高度的 padding，最後幾個選項會被壓在 footer 下方點不到。
- **尚未驗證（因環境中斷）：** Page 2（法規勾選、部分/全部不適當路徑）、Page 3、Page 4（PPE）、匯出 `exportAllData` 的實際互動是否有 JS error。**這是本 QA 的重點。**
- `outputeditor.html` 有前一輪的本機修改，**但它不是本次目標檔案**；提交時**不要**包含它（或先 `git checkout -- outputeditor.html` 還原）。

---

## 1. 環境準備

1. 用本機 http server 提供檔案（**不可用 `file://`**，瀏覽器 MCP 會擋）：
   ```bash
   # 在 repo 根目錄 D:\RiskIdentification
   python -m http.server 8123 --bind 127.0.0.1
   ```
   Windows sandbox 下若失敗，Shell 需以 `required_permissions: ["all"]` 執行。
2. 瀏覽器開啟：`http://127.0.0.1:8123/annotation_tools/index.html`
   - 換版驗證時加 query 破快取，例如 `?v=2`（http.server 會走瀏覽器快取）。
3. 手機模擬（CDP）：
   ```
   Emulation.setDeviceMetricsOverride {"width":390,"height":844,"deviceScaleFactor":3,"mobile":true}
   ```

### 1.1 錯誤攔截器（重要：在頁面載入前注入，才能抓到 Vue render error / console.error）
用 CDP `Page.addScriptToEvaluateOnNewDocument`，source：
```js
window.__errs=[];
window.addEventListener('error',e=>window.__errs.push('ERR: '+(e.message||e.error)));
window.addEventListener('unhandledrejection',e=>window.__errs.push('REJECT: '+e.reason));
(function(){var oe=console.error;console.error=function(){try{window.__errs.push('CONSOLE.ERROR: '+Array.from(arguments).map(a=>(a&&a.message)?a.message:String(a)).join(' '));}catch(_){}return oe.apply(console,arguments);};var ow=console.warn;console.warn=function(){try{window.__errs.push('CONSOLE.WARN: '+Array.from(arguments).map(a=>(a&&a.message)?a.message:String(a)).join(' '));}catch(_){}return ow.apply(console,arguments);};})();
```
之後每個步驟結束都讀 `window.__errs` 檢查。

### 1.2 灌測試資料（免手動上傳）
先抓 demo 案件寫進 IndexedDB `cases_store`，再重載觸發 `restoreFromDB`：
```js
// Runtime.evaluate, awaitPromise:true
(async () => {
  const arr = await (await fetch('/annotation_tools/annotation_demo_cases.json')).json();
  const cases = arr.slice(0,3);
  const db = await new Promise((res,rej)=>{const q=indexedDB.open('ZhianAnnotationLargeDB',2);
    q.onupgradeneeded=e=>{const d=e.target.result;
      if(!d.objectStoreNames.contains('cases_store'))d.createObjectStore('cases_store',{keyPath:'id'});
      if(!d.objectStoreNames.contains('answers_store'))d.createObjectStore('answers_store',{keyPath:'case_title'});
      if(!d.objectStoreNames.contains('meta_store'))d.createObjectStore('meta_store',{keyPath:'key'});};
    q.onsuccess=e=>res(e.target.result); q.onerror=rej;});
  await new Promise((res,rej)=>{const tx=db.transaction('cases_store','readwrite');const st=tx.objectStore('cases_store');
    st.clear(); cases.forEach((c,i)=>st.put({id:i,data:c})); tx.oncomplete=res; tx.onerror=rej;});
  localStorage.setItem('zhian_annotator_id','A01');
  return cases.length;
})();
```
若要一次填滿 3 筆「完整答案」以便測 Page2/3/4 與匯出（跳過逐格填寫），寫入 `answers_store`（keyPath = `case_title`）：
```js
(async () => {
  const arr = await (await fetch('/annotation_tools/annotation_demo_cases.json')).json();
  const cases = arr.slice(0,3);
  const dims=['危害類型','危害媒介','作業型態','空間特徵','防護設施缺失','不安全狀態','不安全行為'];
  const mk = c => { const p3={}; dims.forEach(d=>p3[d]={status:'適當',correction:''});
    return {case_title:c.標題敘述,
      page0_test:{hazard_confirm:'x',casualty_confirm:'x'},
      page1_logic:{hallucination_score:4,hallucination_reason:'',consistency_score:4,consistency_reason:'',is_structural_complete:'是',missing_structures:[],has_privacy_issue:'否',privacy_issue_details:''},
      page2_laws:{evaluation_status:'所有引用法規均適當',inappropriate_laws:[]},
      page3_tags:p3,
      page4_improvements:{action_score:4,action_reason:'',summary_score:4,summary_reason:'',ppe_evaluation:(c.個人防護具的應用||[]).map(it=>({item:it,result:'正確'})),missing_ppe_added:[]}};};
  const db = await new Promise((res,rej)=>{const q=indexedDB.open('ZhianAnnotationLargeDB',2);q.onsuccess=e=>res(e.target.result);q.onerror=rej;});
  await new Promise((res,rej)=>{const tx=db.transaction('answers_store','readwrite');const st=tx.objectStore('answers_store');
    st.clear(); cases.forEach(c=>st.put(mk(c))); tx.oncomplete=res; tx.onerror=rej;});
  return true;
})();
```
> 前進「下一頁」建議用 JS 觸發，避開手機視窗 footer 遮擋：`document.querySelectorAll('footer button')[1].click()`。
> 注意：`currentPage===4` 再按下一頁會 `alert()`，CDP evaluate 會被 alert 卡住；測到 Page 4 就停，匯出/換案另外用真實點擊或先處理 dialog。

---

## 2. 測試案例（TC）

每個 TC 執行後都要：讀 `window.__errs`（應為空）、截圖、記錄實際結果。

| TC | 場景 | 步驟 | 預期結果 |
|----|------|------|----------|
| TC1 | 初次載入（無資料） | 清空 IndexedDB → 開頁 | 顯示「尚未載入…」空狀態；`__errs` 為空 |
| TC2 | 還原既有進度 | 依 1.2 灌 cases → 重載 | 正常渲染 Page 1、快速切換有 3 筆；`__errs` 為空（**已驗證通過**，回歸用） |
| TC3 | 逐頁前進 1→4 | 灌完整答案(1.2) → 連按下一頁 | 每頁正常渲染、可回上一頁；`__errs` 為空 |
| TC4 | **Page 3 表格手機版** | 手機模擬 → 進 Page 3 | 不需水平捲動即可看到「維度/標籤/評估/補充」；radio 與輸入框好點；截圖佐證（**目前失敗，需修**） |
| TC5 | **sticky footer 遮擋** | 手機模擬 → Page 1 捲到底 → 點最後一個 radio（隱私「否」） | 能點到、不被 footer 攔截（**目前失敗，需修**） |
| TC6 | 匯出（全部完成） | 灌完整答案 → 點「匯出批註結果」 | 觸發下載 `職安系統批註結果_A01_…json`；`__errs` 為空；JSON 內每筆含 `_annotator_id` 等欄位 |
| TC7 | 匯出（未完成阻擋） | 只灌 cases 不灌答案 → 點匯出 | 跳 alert 列出待補項目、不下載；無 JS error |
| TC8 | 法規「部分不適當」 | Page 2 選「部分法規不適當」→ 勾選 1 條法規卡 → 填建議 → 前進 | 勾選同步、可通過驗證；`__errs` 為空 |
| TC9 | 法規「全部不適當」 | Page 2 選「所有法規均不適當」→ 填建議 → 前進 | 不得因 `currentCase.processed_result` 為空而丟 `TypeError`（見 §3 風險）；`__errs` 為空 |
| TC10 | 切換案件保存 | 填一筆 → 用「快速切換」跳別案再跳回 | Page 2 勾選/建議正確還原；`__errs` 為空 |
| TC11 | 清空本地快取 | 點「清空本地快取」→ 確認 | 回空狀態、annotator 代號清掉、meta_store 清掉；`__errs` 為空 |
| TC12 | 標注者代號 Modal | 匯入資料後 | 跳出輸入代號 Modal；空值/非法字元(非 `[A-Za-z0-9_-]{1,32}`) 有錯誤提示；正確後關閉 |
| TC13 | 重複匯入同檔 | 匯入 → 再匯入同一檔 | `<input type=file>` 需能重觸發（見 §3 風險：目前未清 `event.target.value`） |

---

## 3. 需重點檢查的程式風險點（靜態分析，尚未實測）

1. **`currentCase.processed_result.map(...)` 未防呆** — 出現在 `handleLawStatusChange`、`syncCurrentPage2Answers`、`watch(currentCaseIndex)`。若某案件缺 `processed_result`，選「所有法規均不適當」或切換案件時會丟 `TypeError: Cannot read properties of undefined (reading 'map')`。demo 資料都有此欄位，故 TC9 用 demo 不會炸；**請另用「缺 processed_result 的案件」測**。建議修為 `(currentCase.value.processed_result||[]).map(...)`。
2. **`handleJsonImport` 未重置 `event.target.value`** — 連續選同一檔第二次不會觸發 change（TC13）。建議在 `reader.onload` 後 `event.target.value=''`。
3. **Page 4 `currentCase.關鍵改善動作 / 改善建議摘要`** — 缺欄位時 `v-for` 空跑不報錯，但畫面空白；確認是否要 fallback 文案。
4. **型別/必填一致性**：`v-model.number` 的分數為 number；`is_structural_complete`、`has_privacy_issue` 為字串 `'是'/'否'`。匯出時型別是否符合下游需求，請對照後端 schema。
5. **自動存檔 debounce 為 30 秒**（`triggerAutoSave`）。測試「編輯後馬上關頁」是否遺失：屬設計取捨，記錄即可。

---

## 4. 要實作的修正（含驗收）

### FIX-1 Page 3 表格手機版（對應 TC4）
- 作法（擇一）：
  - **建議**：手機斷點下把 `<table>` 改為卡片式（每個維度一張卡：標籤、評估 radio、補充輸入直向堆疊）。可用 Tailwind：桌面維持 table，手機 `block`/隱藏 `thead` + 每列 `flex-col`。
  - 或最低限度：移除固定 `w-28/w-64/w-48`、給 `min-w-[…]`，確保 `overflow-x-auto` 捲動順、cell 內容不重疊。
- 驗收：390px 下 Page 3 不需水平捲動即可完成 7 個維度的評估與補充填寫；截圖佐證；`__errs` 空。

### FIX-2 sticky footer 遮擋（對應 TC5）
- 作法：`<main>` 加底部安全間距（footer 約 70px），例如 `class="… pb-24"`，或改 footer 定位策略確保不覆蓋內容。
- 驗收：手機捲到任一頁底部，最後一個 radio/按鈕都可點且不被 footer 攔截。

### FIX-3 ✅【已確認必修】processed_result 防呆
- `(currentCase.value.processed_result || [])` 全面加防呆。
- 驗收：用缺 `processed_result` 的案件跑 TC9、TC10 無 `TypeError`。

### FIX-4（低風險，可選）重複匯入
- `handleJsonImport` 內加 `event.target.value=''`。
- 驗收：TC13 連續匯入同檔可正常觸發。

> 若實測發現使用者所說的「JS error」是其他具體錯誤，先在此文件補記 repro，再修。

---

## 5. 提交 GitHub

1. 只加目標檔案（**排除 `outputeditor.html`**）：
   ```bash
   git checkout -- outputeditor.html   # 還原前一輪非目標的改動（若不想保留）
   git add annotation_tools/index.html annotation_tools/QA_TEST_PLAN.md
   git status --short                  # 確認只含預期檔案
   ```
2. Commit（訊息示例）：
   ```bash
   git commit -m "fix(annotation): 手機版 Page3 表格 RWD 與 sticky footer 遮擋修正"
   ```
3. Push：
   ```bash
   git push
   ```
4. 提交前再跑一次 TC1–TC13，全部 `__errs` 為空、TC4/TC5 截圖通過才推。

---

## 6. 交接備註
- 前一輪環境問題：Shell 指令回傳「no exit status」無法執行、瀏覽器 MCP `cursor-ide-browser` 一度消失。若重現，需重啟終端／agent 環境再繼續。
- 已驗證通過：TC2（還原無 error）。已觀察失敗：TC4、TC5。其餘 TC 尚未執行。
