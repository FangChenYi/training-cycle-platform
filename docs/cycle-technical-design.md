# 週期課表 — 技術設計文件

> 適用對象：前端 / 後端工程師

---

## 1. 整體架構概覽

週期課表系統用於管理教練為學員設計的訓練計畫。核心概念分為三層：

```
TrainingCycle (課表)
 └─ TrainingCycleWeek (週次)
     └─ TrainingExerciseGroup (動作群組)
         └─ TrainingExerciseItem (動作項目)
              └─ TrainingExercise (動作庫)
```

---

## 2. 兩種維度的區分

### 2.1 模板 vs 學員課表（UserId 維度）

|          | 模板 (Template)          | 學員課表 (User-specific) |
| -------- | ------------------------ | ------------------------ |
| `UserId` | `null`                   | 具體的 userId            |
| 用途     | 教練預先設計，可重複套用 | 屬於特定學員，獨立運作   |
| 週次管理 | 手動新增/刪除            | 依課表日期區間自動計算   |

### 2.2 固定課表 vs 週期課表（WeekNumber 維度）

|              | 固定課表 (Fixed)       | 週期課表 (Periodic)         |
| ------------ | ---------------------- | --------------------------- |
| `WeekNumber` | `0`（僅一筆 Week）     | `1, 2, 3, ...`（多筆 Week） |
| 概念         | 每次訓練都做一樣的內容 | 多週循環，每週內容可不同    |
| UI 呈現      | 單一平面列表           | Tab 分頁，每頁一週          |
| 使用場景     | 暖身、活動度等固定流程 | 肌力週期化、多週訓練計畫    |

---

## 3. Entity 定義

### 3.1 TrainingCycle

```
Table: training_cycles
```

| 欄位        | 型別                   | 約束                 | 說明        |
| ----------- | ---------------------- | -------------------- | ----------- |
| Id          | int                    | PK                   |             |
| UserId      | int?                   | FK → users, nullable | null = 模板 |
| Name        | string                 | Required, max 100    | 課表名稱    |
| SectionType | TrainingSection (enum) | Required             | 訓練分類    |
| StartDate   | DateOnly               | Required             | 課表起始日  |
| EndDate     | DateOnly               | Required             | 課表結束日  |

**驗證規則：**

- `StartDate <= EndDate`
- 當 UserId 為 null（即模板）時，(SectionType, Name) 組合需唯一

**關聯：**

- `HasMany(TrainingCycleWeeks)` → Cascade Delete

---

### 3.2 TrainingCycleWeek

```
Table: training_cycle_weeks
```

| 欄位            | 型別    | 約束                 | 說明                              |
| --------------- | ------- | -------------------- | --------------------------------- |
| Id              | int     | PK                   | **不可變識別碼，Schedule 綁定用** |
| TrainingCycleId | int     | FK → training_cycles |                                   |
| WeekNumber      | int     | Required             | 0 = 固定, 1+ = 週期               |
| Name            | string? | max 50               | 週次名稱，可選                    |

**唯一約束：** `(TrainingCycleId, WeekNumber)` — `UX_training_cycle_weeks_cycleid_weeknumber`

**關聯：**

- `HasOne(TrainingCycle)` → Cascade Delete
- `HasMany(TrainingExerciseGroups)` → Cascade Delete

---

### 3.3 TrainingExerciseGroup

```
Table: training_exercise_groups
```

| 欄位                     | 型別    | 約束     | 說明         |
| ------------------------ | ------- | -------- | ------------ |
| Id                       | int     | PK       |              |
| TrainingCycleWeekId      | int     | FK       |              |
| Name                     | string? | max 50   | 群組名稱     |
| Order                    | int     | Required | 群組排序     |
| Rounds                   | int?    |          | 循環組次數   |
| RestBetweenRoundsSeconds | int?    |          | 組間休息秒數 |

**唯一約束：** `(TrainingCycleWeekId, Order)`

---

### 3.4 TrainingExerciseItem

```
Table: training_exercise_items
```

| 欄位            | 型別    | 約束                              | 說明                          |
| --------------- | ------- | --------------------------------- | ----------------------------- |
| Id              | int     | PK                                |                               |
| ExerciseGroupId | int     | FK                                |                               |
| ExerciseId      | int?    | FK → training_exercises, nullable | 參照動作庫，SetNull on delete |
| Name            | string  | Required                          | 動作名稱                      |
| Sets            | int?    |                                   | 組數                          |
| Reps            | int?    |                                   | 次數                          |
| Seconds         | int?    |                                   | 秒數（計時型動作）            |
| Intensity       | string? | max 100                           | 強度描述 (RPE / %1RM)         |
| Notes           | string? | max 1000                          | 備註                          |
| Order           | int     | Required                          | 動作排序                      |

**唯一約束：** `(ExerciseGroupId, Order)`

**刪除行為：** 動作庫項目被刪除時，此處 `ExerciseId` 設為 NULL，Item 本身保留。項目失去動作庫連結後仍可正常顯示。

---

### 3.5 TrainingExercise（動作庫）

```
Table: training_exercises
```

| 欄位               | 型別            | 約束                          | 說明      |
| ------------------ | --------------- | ----------------------------- | --------- |
| Id                 | int             | PK                            |           |
| Name               | string          | Required, max 200, **Unique** | 動作名稱  |
| PrimaryMuscleGroup | string?         | max 100                       | 主要肌群  |
| Equipment          | string?         | max 100                       | 器材      |
| SectionType        | TrainingSection | Required                      | 分類      |
| VideoUrl           | string?         | max 500                       | 教學影片  |
| Description        | string?         | max 2000                      | 描述      |
| Status             | ExerciseStatus  | Default = Active              | 啟用/停用 |

---

### 3.6 TrainingSection（訓練類型）

```csharp
Strength, Core, Mobility,
WarmUp, Conditioning, SmallMuscle, Skill, Other
```

---

### 3.7 AthleteSchedule

```
Table: athlete_schedules
```

| 欄位                | 型別     | 約束                      | 說明           |
| ------------------- | -------- | ------------------------- | -------------- |
| Id                  | int      | PK                        |                |
| UserId              | int      | Required                  | 學員 ID        |
| TrainingCycleId     | int      | FK → training_cycles      | Cascade Delete |
| TrainingCycleWeekId | int      | FK → training_cycle_weeks | Cascade Delete |
| Date                | DateOnly | Required                  | 排程日期       |
| Order (sort_order)  | int      | Required                  | 同日排序       |

**索引：**

- `IX_athlete_schedules_userid_date` — 查詢用
- `IX_athlete_schedules_userid_date_order` — 排序查詢用
- `UX_athlete_schedules_unique_assignment` — `(UserId, Date, TrainingCycleId, TrainingCycleWeekId)` 唯一，防止同日重複安排

**關鍵概念：** Schedule 是「哪一天要練哪個 Week」的連結，不儲存動作內容本身。查詢時透過 WeekId join 取得完整內容。

---

## 4. ER 關係圖

```
┌──────────────────┐
│  TrainingCycle   │
│──────────────────│
│ Id (PK)          │
│ UserId? (FK)     │◄─── null = 模板
│ Name             │
│ SectionType      │
│ StartDate        │
│ EndDate          │
└────────┬─────────┘
         │ 1:N (Cascade)
         ▼
┌──────────────────┐       ┌────────────────────┐
│ TrainingCycleWeek│       │  AthleteSchedule   │
│──────────────────│       │────────────────────│
│ Id (PK)          │◄──────│ TrainingCycleWeekId│
│ TrainingCycleId  │       │ TrainingCycleId    │──► TrainingCycle
│ WeekNumber       │       │ UserId             │
│ Name?            │       │ Date               │
└────────┬─────────┘       │ Order              │
         │ 1:N             └────────────────────┘
         ▼
┌──────────────────────┐
│ TrainingExerciseGroup│
│──────────────────────│
│ Id (PK)              │
│ TrainingCycleWeekId  │
│ Name?                │
│ Order                │
│ Rounds?              │
│ RestBetweenRounds?   │
└────────┬─────────────┘
         │ 1:N
         ▼
┌──────────────────────┐     ┌──────────────────┐
│ TrainingExerciseItem │     │ TrainingExercise │
│──────────────────────│     │  (動作庫)         │
│ Id (PK)              │     │──────────────────│
│ ExerciseGroupId      │     │ Id (PK)          │
│ ExerciseId? ─────────│────►│ Name (Unique)    │
│ Name                 │     │ SectionType      │
│ Sets?, Reps?, Secs?  │     │ VideoUrl?        │
│ Intensity?, Notes?   │     │ Status           │
│ Order                │     └──────────────────┘
└──────────────────────┘
```

---

## 5. API 端點總覽

### 5.1 Cycle CRUD（教練）

| 方法   | 路徑                                    | 說明                                  |
| ------ | --------------------------------------- | ------------------------------------- |
| POST   | `/training/cycle/query`                 | 查詢課表列表（含完整 Detail）         |
| GET    | `/training/cycle/query/{id}/detail`     | 查詢單一課表完整巢狀結構              |
| POST   | `/training/cycle/create-full`           | 建立完整課表（含 Weeks/Groups/Items） |
| POST   | `/training/cycle/update-full/{cycleId}` | 更新完整課表（含結構重建）            |
| DELETE | `/training/cycle/delete/{id}`           | 刪除課表（Cascade）                   |

### 5.2 學員唯讀（學員端）

| 方法 | 路徑                                  | 說明               |
| ---- | ------------------------------------- | ------------------ |
| POST | `/athlete/my-cycle/query`             | 查詢自己的課表     |
| GET  | `/athlete/my-cycle/query/{id}/detail` | 查詢自己的課表細節 |
| POST | `/athlete/my-schedule/query`          | 查詢自己的排程     |

### 5.3 Schedule（排程）

| 方法   | 路徑                        | 說明               |
| ------ | --------------------------- | ------------------ |
| POST   | `/athlete/schedule/query`   | 查詢排程           |
| POST   | `/athlete/schedule/create`  | 建立排程（可批次） |
| POST   | `/athlete/schedule/reorder` | 調整同日排程順序   |
| DELETE | `/athlete/schedule/delete`  | 刪除排程（可批次） |

### 5.4 其他子項 CRUD

Cycle、CycleWeek、ExerciseGroup、ExerciseItem、Exercise 各有獨立的 query/create/update/delete 端點。
