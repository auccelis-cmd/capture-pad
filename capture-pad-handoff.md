# 插單 Capture Pad — 專案交接文件

> 給協作 AI 的說明：這是一個已上線使用中的個人工作追蹤 PWA。請在**不破壞現有資料與功能**的前提下修改。每次修改後請：
> 1. 回傳完整的 `index.html`（單一檔案，CSS/JS 都內嵌）；
> 2. 把 `sw.js` 的 `CACHE` 版本號 +1，並更新頁尾版本號；
> 3. 若需要新欄位，提供可重複執行的 SQL（`add column if not exists`），並在最後加 `notify pgrst, 'reload schema';`。
>
> 目前版本：**v2.0**（`sw.js` 的 CACHE = `capture-pad-v200`）

---

## 1. 專案概要

- **用途**：人力仲介公司（外籍移工引進）文件管理師的工作追蹤工具。核心理念是「先記下來，不要現在處理」。
- **使用者**：單一使用者，電腦與 iPhone 同時使用（iPhone 以「加入主畫面」當 PWA）。
- **網址**：https://auccelis-cmd.github.io/capture-pad/
- **程式碼**：GitHub repo `auccelis-cmd/capture-pad`，GitHub Pages 部署（main 分支根目錄）。
- **後端**：Supabase（Postgres + Auth + Realtime）。
  - Project URL：`https://xdymhrblfpwrnertsbre.supabase.co`
  - SQL Editor：https://supabase.com/dashboard/project/xdymhrblfpwrnertsbre/sql/new
  - 前端只使用 **Publishable key**（使用者第一次開啟時自行貼上，存在 localStorage），絕不使用 Secret key。
  - 登入方式：Email + 密碼（帳號在 Supabase Authentication 建立）。

### 檔案

| 檔案 | 說明 |
|---|---|
| `index.html` | 整個 App（約 70 KB），HTML + CSS + JS 全部內嵌 |
| `sw.js` | Service Worker，navigate 請求採 network-first；改版時只需改 `CACHE` 版本字串 |
| `manifest.webmanifest` | PWA 設定，`background_color` / `theme_color` = `#141817` |
| `icon-192.png` 等 | App 圖示 |

外部資源：`@supabase/supabase-js@2`（jsDelivr CDN）、Google Fonts（LXGW WenKai TC、Noto Sans TC）。

---

## 2. 版面與功能

### 整體版面
- **電腦（>980px）**：上方「正在做」列 + 下方三欄（一般｜案件｜近期）。
- **手機（≤980px）**：一次顯示一欄，底部固定分頁列（一般／案件／近期），分頁上顯示件數；「近期」有逾期或今天到期時數字變紅。

### 正在做（多件並行）
- 最多同時 5 件（`MAXF=5`）。按任何事項的 ▶ 加入／移出（已加入時 ▶ 亮綠底）。
- 第一件是「主要」，大字顯示；其他以小標籤列在下方，點標籤文字可設為主要。
- 每件有 ✓（直接把事項標為完成並移出）與 ×（只移出）。兩件以上時出現「全部清空」。
- 資料存在 `capture_focus` 表（跨裝置同步）；表不存在時退回 localStorage。
- 存的是**參照**而不是文字：`{k:'item',id}`、`{k:'sub',id,cid}`、`{k:'text',t}`（舊版遷移用）。事項改名會自動反映；完成或刪除的會自動移出（`pruneFocus`）。

### 一般（快速插單）
- 輸入內容 + 預計日期（今天／明天／不指定）→ 記下。
- 「尚未完成」排序：釘選 → 預計日期近 → 最近更新。
- **點事項名稱**展開編輯區：可改內容與預計日期。
- 每筆有 ▶（正在做）、⌖（釘選）、×（刪除，會確認）。
- 「今天完成」只列今天完成的；「清掉完成項」刪除所有已完成紀錄（會確認，並顯示較早完成的筆數）。

### 案件
- 案件卡可摺疊（展開狀態存 localStorage）。
- **案件類型**：`entry` 入境案件（預設，`case_type` 為 null 也視為入境）／`takeover` 承接案件／`general` 一般案件。
  - 入境案件：有進度軸、國外進度追蹤、15 項流程勾選器。
  - 承接案件：沒有進度軸。案件上方有「承接方式」`takeover_mode`：
    - `two` 雙方合意：需要「終止聘僱許可函」；日期欄位＝承接日 `takeover_date`。
    - `three` 三方合意：不需要終止聘僱許可函；日期欄位＝承接日。
    - `expiry` 期滿轉換：外國人期滿隔日到新公司上班；日期欄位＝原雇主期滿日 `contract_end_date`、轉換合意日 `agree_date`；並顯示提示框（上班日＝期滿日+1、接續通報最晚日、居留效期約在期滿日前一天或當天屆滿，送通報時要一起送一站式居留證）。
    - 舊欄位 `expiry_transfer=true` 且沒有 mode 時視為 expiry。
    - 提示就服站時程（每週二回報確認單、每週四可承接，顯示最近的週四）。摺疊標題顯示「承接方式・期滿 m/d／預定承接／已承接 m/d」。
  - 承接流程 `TK_FLOW`（依序）：求才登記、求才送審、無違反登記｜勞動部函、終止聘僱許可函（提示：雙方合意才需要；非雙方合意時勾選器預設不勾）、承接登記（提示：沒有工業局函時才需要）｜承接日確認、接續通報、接續聘僱、接續居留展延。流程中的「勞動部函」新增時寫入 `doc_type='tkmol'`。
  - 一般案件：沒有進度軸與流程。
  - 每種類型的流程定義在 `FLOWS`（`list`／`groups`／`idx` 排序函式），用 `flowOf(c)` 取得。
- **入境案件的進度軸**：挑工 → 國外作業 → 送簽 → 領簽 → 入境。
  - 四個日期欄位：挑工日 `pick_date`、送簽日 `visa_submit_date`、領簽日 `visa_get_date`、安排入境日 `arrival_date`（改變即自動儲存）。
  - 節點狀態：已發生＝實心、未來日期＝虛線（預定）、未填＝空心。
  - 「國外作業」天數：未送簽時＝挑工日到今天；已送簽＝挑工到送簽的天數。
  - 摺疊時標題旁顯示階段標籤：待挑工／預計挑工 m/d／國外作業中・N 天／送簽中・N 天／已領簽・待排入境／入境 m/d／已入境 m/d。
- **國外進度追蹤**：日期（預設今天）+ 一句話，新的在上，存在 `progress_log`（jsonb 陣列 `{id,d,t}`）。
- **案件備註**：離開輸入框自動儲存。
- **子任務**：
  - 點名稱展開編輯區：名稱、類型、狀態、依類型出現的日期欄位、備註。
  - ▶ 加入正在做、× 刪除（會確認）、勾選完成。
  - 摺疊時顯示：類型小標籤、狀態、日期、效期／期限（14 天內琥珀色、過期紅色，並顯示「N 天後／逾期 N 天」）、備註摘要兩行。
- **新增流程項目（勾選器）**：列出標準流程 15 項，分「申請階段／國外作業／入境」三段，可整段勾選、全選、全不選；已存在的項目灰掉標「已有」。新增入境案件時勾「建立後選擇流程項目」會自動展開勾選器。
- 新增單一子任務：常用按鈕（帶入名稱＋類型）、名稱、類型、狀態、依類型的日期欄位。子任務名稱與狀態欄按 Enter 可新增（已處理中文輸入法選字不誤送）。
- 子任務顯示順序：依標準流程順序（`flowIdx`），非流程項目排在最後。

### 近期
- 彙整所有有日期的未完成事項，分組：逾期／今天／七天內／之後（每組最多 40 筆）。
- 來源：一般事項的預計日期、子任務效期／期限、DHL 寄國外日（未來的才列）、舊的辦理日 `task_date`、入境案件的安排入境日（未來的才列）、「國外作業已 N 天・該追進度了」提醒。
- 跨年度的日期會顯示年份。

---

## 3. 業務規則（最重要，修改時務必保留）

### 3.1 標準入境流程（`FLOW`，依序）

1. 求才登記
2. 無違反法令申請
3. 求才送審
4. 無違反證明書
5. 求才證明書
6. 勞動部函
7. 入簽函
8. 驗證文件
9. 機場關懷
10. 接機安排
11. 入國通報
12. 入境體檢
13. 初次居留證
14. 聘僱許可
15. 卡式居留證

分段（`FLOW_GROUPS`）：申請階段＝1–7、國外作業＝8、入境＝9–15。

### 3.2 子任務類型（`TYPES`）與日期計算

| 類型 key | 名稱 | 名稱比對（自動判斷） | 起算欄位 | 效期／期限 |
|---|---|---|---|---|
| `general` | 一般（不需日期） | — | — | 無 |
| `jc` | 求才證明書 | `/求才證明/` | 發文日 | 發文日 **+90 天** |
| `nv` | 無違反證明書 | `/無違[反法](法令)?證明/` | 發文日 | 發文日 **+90 天** |
| `mol` | 勞動部函（初招） | `/勞動部函\|招募函/` | 發文日 | 發文日起 **1 年**（對應日當天） |
| `entry` | 入簽函（初招） | `/入簽\|引進/` | 發文日 | 發文日起 **9 個月**（對應日當天） |
| `reentry` | 重入簽函（重招） | `/重入簽/` | 發文日 | **手動填寫**（規則待使用者提供） |
| `takeover` | 承接登記 | `/承接登記/` | 登記日 | 登記日 **+59**（登記日當天算第 1 天，60 天內；欄位名「登記到期日」） |
| `tkmol` | 勞動部函（承接） | `/承接函/`（承接流程新增時直接指定） | 發文日 | 無效期（隨時可承接） |
| `tknotify` | 接續通報 | `/接續通報/` | 承接日／轉換合意日 | 雙方、三方：承接日 **+2**（當天算第 1 天，3 日內）。期滿轉換：轉換合意日 +2，但**不晚於原雇主期滿日**（例：10/1 合意、10/2 期滿 → 10/2） |
| `tkhire` | 接續聘僱 | `/接續聘僱/` | 承接日／轉換合意日 | **+14**（15 日內；例：10/1 合意 → 10/15） |
| `tkres` | 接續居留展延 | `/居留展延\|接續居留/` | 承接日／轉換合意日 | 雙方、三方：跟接續聘僱一起 **+14**，留意居留效期。期滿轉換：跟接續通報同一天，且若「外國人居留效期」更早則以居留效期為期限。居留效期存在 `expiry_date`，早於期限時標紅，並列入近期 |
| `tkend` | 終止聘僱許可函 | `/終止聘僱/` | 發文日 | 無效期（雙方合意才需要） |
| `verify` | 驗證文件 | `/驗證/` | — | 無效期；有「DHL 寄國外日」欄位 |
| `care` | 機場關懷 | `/機場關懷/` | 案件入境日 | 入境日 **−3 天** |
| `pickup` | 接機安排 | `/接機/` | 案件入境日 | 入境日 **−3 天**（與機場關懷一起做） |
| `notify` | 入國通報 | `/入國通報/` | 案件入境日 | 入境日 **+3**（入境隔日起 3 天內） |
| `exam` | 入境體檢 | `/體檢/` | 案件入境日 | 入境日 **+3** |
| `permit` | 聘僱許可 | `/聘僱許可/` | 案件入境日 | 入境日 **+15** |
| `arc` | 初次居留證 | `/初次居留/` | 案件入境日 | 入境日 **+30** |

- **自動判斷順序**（`guessType`）：tknotify → tkhire → tkres → tkend → tkmol → takeover → reentry → notify → care → pickup → exam → permit → arc → entry → jc → nv → mol → verify（順序有意義，例如「重入簽」要先於「入簽」、「入國通報」不能被判成入簽函）。
- `doc_type` 只在使用者選的類型與自動判斷不同時才寫入；null 代表用名稱判斷。
- 「求才登記」「求才送審」「無違反法令申請」是申請流程，屬於 `general`，**沒有效期**。
- **月份計算**（`termEnd`）：到期日為對應日當天（例：2026/9/17 起算 1 年 → 2027/9/17；9 個月 → 2027/6/17）；該月沒有對應日時取月底（5/31 + 9 個月 → 2/28）。
- **天數計算**：直接加天數（2026/9/17 + 90 → 2026/12/16）。
- **入境類期限**不存在子任務上，而是即時由 `cases.arrival_date` 計算（`dueOf`）；沒填入境日時顯示「填入境日後自動算期限」。
- 「國外作業」提醒：挑工後超過 `ABROAD_ALERT = 30` 天仍未送簽就提醒；第 23 天起先出現在「七天內」。

### 3.3 兩種「幾日內」的算法（使用者已確認）
- **入境**：入境日「隔日」起算。例：9/21 入境，3 日內 = 9/22–9/24 → 期限 = 入境日 +3（15 日 → +15、30 日 → +30）。
- **承接**：承接日「當天」起算。例：9/21 承接，3 日內 = 9/21–9/23 → 期限 = 承接日 +2（15 日 → +14）。

### 3.4 名稱容錯
`flowIdx` 會把「違法法令」視為「違反法令」，並依類型對應流程項目（例：「勞動部函申請」算「勞動部函」、「越辦驗證」算「驗證文件」），避免新增流程時重複。

---

## 4. 資料庫結構（Supabase / Postgres）

> 原始三張表（`capture_items`、`capture_cases`、`capture_case_subtasks`）是早期建立的，欄位由程式碼推得；後續新增的欄位與 `capture_focus` 表如下方 SQL。所有表都以 `user_id` 做 RLS（只能存取自己的資料）。

### capture_items（一般事項）
| 欄位 | 型別 | 說明 |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | 擁有者 |
| content | text | 內容 |
| planned_date | date | 預計日期 |
| is_pinned | bool | 釘選 |
| is_completed | bool | 完成 |
| is_active | bool | 舊欄位，目前未使用 |
| created_at / updated_at | timestamptz | 「今天完成」以 updated_at 判斷 |

### capture_cases（案件）
| 欄位 | 型別 | 說明 |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | |
| name | text | 案件名稱 |
| notes | text | 案件備註 |
| created_at | timestamptz | 案件依此倒序排列 |
| case_type | text | `entry`／`general`；null 視為 entry |
| pick_date | date | 挑工日 |
| visa_submit_date | date | 送簽日 |
| visa_get_date | date | 領簽日 |
| arrival_date | date | 安排入境日 |
| takeover_date | date | 承接日（承接案件） |
| expiry_transfer | bool | 舊欄位（v1.9），已由 takeover_mode 取代 |
| takeover_mode | text | 承接方式 two／three／expiry |
| contract_end_date | date | 原雇主期滿日（期滿轉換） |
| agree_date | date | 轉換合意日（期滿轉換） |
| progress_log | jsonb | 國外進度追蹤 `[{id,d,t}]`，預設 `[]` |

### capture_case_subtasks（子任務）
| 欄位 | 型別 | 說明 |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | |
| case_id | uuid | 所屬案件（刪案件時需一併刪除，依賴 DB cascade） |
| content | text | 名稱 |
| status | text | 狀態文字 |
| task_date | date | **舊欄位**（原「辦理日／預計日」，已停用；編輯區可清除） |
| issue_date | date | 發文日／登記日 |
| expiry_date | date | 效期／到期日 |
| dhl_date | date | DHL 寄國外日 |
| doc_type | text | 類型 key，null＝依名稱判斷 |
| notes | text | 子任務備註 |
| is_completed | bool | |
| created_at | timestamptz | 同順序時的次要排序；批次新增時每筆 +1 秒以保持順序 |

### capture_focus（正在做）
| 欄位 | 型別 | 說明 |
|---|---|---|
| user_id | uuid | PK，預設 auth.uid() |
| entries | jsonb | 正在做清單，第一筆為主要 |
| updated_at | timestamptz | |

### 累積執行過的 SQL（可重複執行）

```sql
-- 子任務欄位
alter table capture_case_subtasks
  add column if not exists notes text,
  add column if not exists issue_date date,
  add column if not exists doc_type text,
  add column if not exists dhl_date date;

-- 案件欄位
alter table capture_cases
  add column if not exists arrival_date date,
  add column if not exists case_type text,
  add column if not exists pick_date date,
  add column if not exists visa_submit_date date,
  add column if not exists visa_get_date date,
  add column if not exists progress_log jsonb not null default '[]',
  add column if not exists takeover_date date,
  add column if not exists expiry_transfer boolean not null default false,
  add column if not exists takeover_mode text,
  add column if not exists contract_end_date date,
  add column if not exists agree_date date;

-- 正在做（跨裝置同步）
create table if not exists capture_focus (
  user_id uuid primary key default auth.uid() references auth.users(id) on delete cascade,
  entries jsonb not null default '[]',
  updated_at timestamptz not null default now()
);
alter table capture_focus enable row level security;
-- 以下兩行只在第一次建立時執行（重複執行會報「已存在」）
-- create policy "own focus" on capture_focus for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
-- alter publication supabase_realtime add table capture_focus;

notify pgrst, 'reload schema';
```

---

## 5. 程式結構（index.html 內的 JS）

- **狀態**：`items`、`cases`、`subs`（`{case_id: [subtask]}`）、`focus`、`openCases`、`openSubs`、`openItems`、`pickOpen`。
- **同步**：`refresh(which)`（`'items'`／`'cases'`／`'all'`）→ `loadItems` / `loadCases`（子任務一次用 `.in()` 抓）/ `loadFocus` → `renderAll()`。Realtime 訂閱四張表，事件 300ms 防抖後 `refresh('all')`；App 回到前景或恢復網路時也會 refresh。
- **寫入**：`run(promise)` 統一處理錯誤與同步狀態；錯誤訊息含新欄位名時會跳提示請使用者跑 SQL。多數操作是「先更新畫面、再寫入、最後 refresh」。
- **渲染**：`renderFocus`、`renderGeneral`、`renderCases`（含 `subRowHTML`、`stepperHTML`、`pickerHTML`）、`renderUpcoming`。重繪前會保存輸入框的值、勾選狀態、焦點與游標位置並還原，避免同步時打到一半被清掉。
- **所有按鈕**用事件委派：元素加 `data-act="…"`，在 `document` 的 click 監聽器裡 `switch(act)` 處理。
- **子任務表單欄位**以 `key` 分組：編輯區 key＝`se-<subId>`，新增表單 key＝`c-<caseId>`；欄位 id＝`${key}-name|type|status|from|exp|dhl|notes|dates`，由 `onField` 處理類型切換、自動判斷、自動計算效期。
- **日期工具**：`parseD`、`iso`、`todayISO`、`dayDiff`、`addDays`、`termEnd`、`dueLabel`（今天／明天／後天／昨天／m/d（週）；跨年加年份）、`ymd`、`md`。

### localStorage keys
| key | 用途 |
|---|---|
| `capturePad.v04.connection` | Supabase URL + Publishable key |
| `capturePad.tab` | 手機目前分頁 |
| `capturePad.openCases` | 展開中的案件 |
| `capturePad.focus` | 正在做的本機備份 |
| `capturePad.bg` | 背景模式 mist／stars／off |
| `capturePad.v01.items` / `capturePad.v01.now` / `capturePadPrevNow` | 舊版資料，啟動時自動遷移後清除 |

---

## 6. 視覺設計

- **深色、沉穩**。色彩 token（`:root`）：
  - 底 `--bg #141817`、卡片 `--surface #1c211f`、浮起 `--raised #232927`、線 `--line #2f3633`
  - 文字 `--ink #e7e3d9`、次要 `--muted #9a9e96`、更淡 `--faint #6f746d`
  - 強調（鼠尾草綠）`--accent #8fb5a1`、逾期（乾燥玫瑰）`--danger #d99a8e`、快到期（琥珀）`--warn #d8b06a`
- **字體**：標題類（欄位標題、案件名、正在做、小標題、日期）用霞鶩文楷 `LXGW WenKai TC`；內文用 `Noto Sans TC`。
- 卡片用細邊框分層、不用陰影；主要按鈕實心綠、次要按鈕細框。
- **動態背景**（v1.7）：頁尾可切換「霧光流動」（預設，三團緩慢漂移的柔光，純 CSS transform 動畫）／「星塵漂浮」（canvas 粒子緩慢上飄，約 30fps，分頁隱藏時暫停）／「靜止」。設定存在 localStorage `capturePad.bg`。尊重系統「減少動態效果」設定。卡片底色為 90% 不透明，讓背景微微透出。
- 手機輸入框字級維持 16px（避免 iOS 自動放大）；電腦 14px。
- 已處理 iPhone 瀏海／底部安全區（`env(safe-area-inset-*)`）。

---

## 7. 部署步驟（每次更新）

1. 若有 SQL，先到 Supabase SQL Editor 執行。
2. 到 GitHub repo 用新檔案取代 `index.html`（與 `sw.js`，版本號已 +1）。
3. 等 GitHub Pages 更新（約 1 分鐘），重新整理；iPhone PWA 需完全關閉再開。

---

## 8. 待確認／之後可能的需求

- **重入簽函（重招）** 的效期規則尚未提供，目前手動填寫。
- 「入境隔日起 3 天內」目前算成入境日 +3（例：10/5 入境 → 10/8），使用者尚未明確確認。
- 「國外作業」提醒天數 `ABROAD_ALERT` 暫定 30 天，使用者試用後可能調整。
- 曾提議但尚未做：「已完成」歷史紀錄頁。

## 9. 修改時的注意事項

- 使用者的操作語言是**繁體中文（台灣用語）**，介面文字請維持一致。
- 不要改動資料表既有欄位的意義；新增欄位一律 `add column if not exists`，並讓程式在欄位不存在時盡量不壞（例如只在有值時才把新欄位放進 insert）。
- 不要使用 Secret key；不要把 Publishable key 寫死在程式裡。
- 中文輸入法：按 Enter 送出的地方都要檢查 `e.isComposing` 與 `keyCode 229`。
- 修改後請用手機寬度（390px）與電腦寬度都檢查過版面。
