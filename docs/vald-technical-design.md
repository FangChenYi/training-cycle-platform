# VALD 整合 — 技術設計文件

> 適用對象：後端 / 前端工程師、資料庫管理者

---

## 1. 整體架構概覽

VALD 整合透過 OAuth2 Client Credentials 串接 VALD Performance 雲端 API，涵蓋三大服務群：

```
┌──────────────┐        ┌─────────────────────────┐
│              │──────► │  VALD Auth API          │  發放 Bearer Token
│              │        └─────────────────────────┘
│              │        ┌─────────────────────────┐
│    WebAPI    │──────► │  VALD ForceDecks API    │  取得檢測 Tests / Trials
│              │        └─────────────────────────┘
│              │        ┌─────────────────────────┐
│              │──────► │  VALD Profile API       │  取得 Profile / Group
└──────┬───────┘        └─────────────────────────┘
       │
       ▼
┌────────────────────────────┐
│  PostgreSQL                │
│  ─ AspNetUsers             │ ← 以 ValdProfileId 綁定運動員
│  ─ vald_test_metric_records│ ← 快照 Averages/Maxes 供查詢與圖表
└────────────────────────────┘
```

整合採用 **即時呼叫 + 增量快取** 的策略：

- **Profile 同步**：教練觸發時即時打 VALD API，僅針對本地尚未存在的 Profile 建立帳號。
- **Test 同步**：以 `ModifiedFromUtc` 做增量拉取，並在本地保留計算後的指標快照，避免每次查詢都呼叫第三方 API。

---

## 2. 資料流程

### 2.1 Token 取得

```
首次呼叫 → POST {AuthTokenUrl}
             grant_type=client_credentials
             client_id, client_secret, audience
         → 回傳 access_token + expires_in
```

### 2.2 Profile 同步流程

```
教練選 Group → POST /athlete/vald/{groupId}/sync-profiles
  ├─ GET  /profiles?TenantId&groupId           取得 VALD profile 列表
  ├─ 比對 DB.AspNetUsers.ValdProfileId          過濾掉已存在的
  ├─ 並行 GET /profiles/{profileId}             補齊 email / DOB / GroupIds
  ├─ 逐筆建立 Identity User（角色 Athlete）
  └─ 回傳成功 / 失敗筆數
```

### 2.3 Test 同步流程（單一運動員）

```
GET /athlete/vald/{athleteId}/sync-tests
  ├─ 查 User.ValdProfileId
  ├─ GetLatestRecordedDateUtcAsync(profileId)
  │      → 取本地最新 Recorded 日期作為 ModifiedFromUtc
  │      → 若 DB 無任何紀錄 → fallback 2025-01-01
  ├─ ValdTestService.GetValdTestAsync
  │    ├─ GET /tests?TenantId&ModifiedFromUtc&ProfileId
  │    └─ 針對每一筆 Test（僅處理白名單 TestType）:
  │         ├─ Task.Delay(200ms)            簡單限速
  │         ├─ GET /teams/{teamId}/tests/{testId}/trials
  │         ├─ 429 → 指數退避重試 (最多 3 次)
  │         ├─ 過濾 Limb=="Trial" 且在允許 Result 白名單內
  │         ├─ 計算 Averages / Maxes（四捨五入至小數第 2 位）
  │         └─ 彙總進 ValdTestsWithTrialsResponse
  ├─ 計算 DSI = CMJ.PEAK_TAKEOFF_FORCE / IMTP.PEAK_VERTICAL_FORCE (fallback: BW_NET_FORCE_PEAK)
  ├─ 計算 EUR = CMJ.JUMP_HEIGHT_IMP_MOM / SJ.JUMP_HEIGHT_IMP_MOM
  └─ ValdMetricRepository.UpsertTestAsync
       → Upsert 到 vald_test_metric_records（以 TestId + DefinitionResult 為 key）
```

### 2.4 歷史檢測查詢流程

```
GET /athlete/vald/{athleteId}/tests?fromDate&toDate
  ├─ 驗證：FromDate ≤ ToDate、跨度 ≤ 3 年
  ├─ 角色檢查：Athlete 僅能查自己
  ├─ ValdMetricRepository.GetTestsByDateRangeAsync
  │     → 從本地快照組裝 ValdTestWithTrials
  ├─ 以 RecordedDate 分組
  └─ 每組重新計算 DSI / EUR（用本地快照 Averages）
```

---

## 3. Entity 定義

### 3.1 ValdTestMetricRecord（本地快照）

```
Table: vald_test_metric_records
```

| 欄位             | 型別           | 約束             | 說明                       |
| ---------------- | -------------- | ---------------- | -------------------------- |
| Id               | long           | PK               |                            |
| ProfileId        | string         | Required, max 64 | VALD Profile 識別碼        |
| TestId           | string         | Required, max 64 | VALD Test 識別碼           |
| TestType         | string         | Required, max 32 | SJ / CMJ / IMTP / HJ       |
| RecordedDateUtc  | DateTimeOffset | Required         | 檢測發生時間               |
| ModifiedDateUtc  | DateTimeOffset | Required         | VALD 端最後修改時間        |
| Weight           | decimal?       |                  | 檢測時體重                 |
| SyncedAtUtc      | DateTimeOffset | Required         | 最近一次從 VALD 同步的時間 |
| DefinitionResult | string         | Required, max 64 | 指標名稱                   |
| AverageValue     | double         | Required         | 該指標所有 trials 的平均   |
| MaxValue         | double         | Required         | 該指標所有 trials 的最大值 |

**索引：**

- `UX_vald_test_metric_test_def` — `(TestId, DefinitionResult)` 唯一，作為 Upsert key
- `IX_vald_test_metric_profile_recorded` — `(ProfileId, RecordedDateUtc)` 查詢索引

### 3.2 AspNetUsers（擴充欄位）

| 欄位                   | 型別     | 說明                            |
| ---------------------- | -------- | ------------------------------- |
| ValdProfileId          | string   | 綁定 VALD Profile（同步後寫入） |
| ValdGroupId            | string?  | 主要所屬 Group                  |
| GivenName / FamilyName | string   | 由 VALD Profile 同步            |
| DateOfBirth            | DateTime | 由 VALD Profile 同步            |
| Sex                    | string   | 由 VALD Profile 同步            |

---

## 4. 指標白名單

`ValdTestService` 僅處理以下 TestType 與 Result 組合

| TestType                            | 允許 DefinitionResult                                                                                                                                                        |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **SJ** (Squat Jump)                 | JUMP_HEIGHT_IMP_MOM、PEAK_TAKEOFF_POWER、PEAK_TAKEOFF_FORCE、CONCENTRIC_IMPULSE_P1、CONCENTRIC_IMPULSE_P2、CONCENTRIC_RFD_200、RSI_MODIFIED_IMP_MOM                          |
| **CMJ** (Countermovement Jump)      | JUMP_HEIGHT_IMP_MOM、ECCENTRIC_DECEL_IMPULSE、ECCENTRIC_PEAK_VELOCITY、CONCENTRIC_IMPULSE_P1、CONCENTRIC_IMPULSE_P2、PEAK_CONCENTRIC_FORCE、PEAK_TAKEOFF_FORCE、RSI_MODIFIED |
| **IMTP** (Isometric Mid-Thigh Pull) | BW_NET_FORCE_PEAK、PEAK_VERTICAL_FORCE、RFD_AT_250MS、START_TO_PEAK_FORCE、START_TO_80PC_PEAK_FORCE                                                                          |
| **HJ** (Hop Jump)                   | HOP_RSI                                                                                                                                                                      |

過濾條件 (`Limb == "Trial"`) 確保只採雙腳合計的 Trial 結果，排除單腳 Left / Right 細項。

---

## 5. 衍生指標計算

### 5.1 DSI (Dynamic Strength Index)

```
DSI = CMJ.Averages[PEAK_TAKEOFF_FORCE]
    ÷ IMTP.Averages[PEAK_VERTICAL_FORCE]   （若缺則 fallback 到 BW_NET_FORCE_PEAK）
```

反映「爆發力 vs 最大肌力」的比例。缺任一測試或分母為 0 時回傳 null。

### 5.2 EUR (Eccentric Utilization Ratio)

```
EUR = CMJ.Averages[JUMP_HEIGHT_IMP_MOM]
    ÷ SJ.Averages[JUMP_HEIGHT_IMP_MOM]
```

反映「離心蓄力轉換效率」。兩值皆四捨五入至小數第 2 位。

---

## 6. API 端點總覽

### 6.1 Profile / Group

| 方法  | 路徑                                       | 權限  | 說明                                           |
| ----- | ------------------------------------------ | ----- | ---------------------------------------------- |
| GET   | `/vald/groups`                             | Coach | 取得 VALD 所有 Groups                          |
| POST  | `/athlete/vald/{groupId?}/sync-profiles`   | Coach | 同步指定 Group 下尚未建立的 Profile            |
| PATCH | `/athlete/vald/{athleteId}/update-profile` | Coach | 從 VALD 重新抓取單一運動員的基本資料並更新本地 |

### 6.2 Tests

| 方法 | 路徑                                              | 權限           | 說明                               |
| ---- | ------------------------------------------------- | -------------- | ---------------------------------- |
| GET  | `/athlete/vald/{athleteId}/sync-tests`            | Coach          | 增量同步該運動員的檢測數據         |
| GET  | `/athlete/vald/{athleteId}/tests?fromDate&toDate` | Coach, Athlete | 從本地快照查歷史檢測（含 DSI/EUR） |

**授權補充：** 角色為 `Athlete` 時只能查詢自己 (`currentUser.Id == athleteId`)，否則回 403。

---

## 7. 跑步 / 體能數據補充

VALD 僅涵蓋 ForceDecks 類型測試；跑步速度 (30m / 40m / 60m) 與 6 分折返 VO2Max 為教練手動輸入，與 VALD 數據並列於同一運動員的歷史數據圖表：

- `/athlete/{athleteId}/running-data`

---

## 8. 安全性

- **角色邊界**：`Athlete` 僅能查自己的歷史；同步與 Profile 寫入僅限 `Admin` / `Coach`。
- **租戶隔離**：所有對 VALD API 的呼叫一律帶 `TenantId`，不接受 query 外部傳入以防越權讀取其他組織資料。
