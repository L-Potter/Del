在 Oracle SQL 中，`LIKE + OR` 和 `IN` 的效能比較取決於以下因素：

### **1. 如果 `LIKE` 能使用索引**
當 `LIKE` 條件能匹配索引時，其效能通常會優於不使用索引的條件查詢，甚至可以在特定情況下比 `IN` 快。這是因為索引的使用可以顯著減少掃描的資料量。

#### **條件：`LIKE` 可以使用索引的情況**
- 範圍匹配的前綴有固定字符（如 `LIKE 'abc%'`）。
- 相關索引類型適合支持前綴匹配（如 B-tree 索引）。

範例：
```sql
SELECT * 
FROM my_table
WHERE column1 LIKE 'abc%' OR column1 LIKE 'xyz%';
```
如果索引適用於 `column1`，這種查詢的效能會很好。

---

### **2. 使用 `IN` 的效能考量**
`IN` 一般適用於完全匹配條件（等於比較），並且在以下情況下表現優秀：
- 列值是離散的。
- 適配 B-tree 索引。

範例：
```sql
SELECT * 
FROM my_table
WHERE column1 IN ('abc', 'xyz');
```
這種情況下，`IN` 通常比 `LIKE + OR` 更高效，因為 `IN` 可以直接利用索引執行高效的查詢計劃，而不需要進行字串匹配操作。

---

### **3. LIKE + OR vs IN 的效能對比**
#### **若 `LIKE` 的條件能使用索引**
- `LIKE + OR` 效能可能與 `IN` 接近，具體取決於匹配條件的複雜度。
- 若有多個 `LIKE`，索引命中率高，`LIKE + OR` 表現可以與 `IN` 一致。

#### **若 `LIKE` 的條件無法使用索引**
- `LIKE + OR` 的效能會急劇下降，因為需要逐行掃描來匹配條件。
- 此時，`IN` 更具優勢。

---

### **效能優化建議**
1. **確保索引有效**：
   - 為查詢中使用的列建立適當的索引，例如 B-tree 或 Text 索引。
   - 使用 `EXPLAIN PLAN` 檢查執行計劃，確認索引是否被使用。

2. **替代方案**：
   - 若 `LIKE` 無法直接使用索引，可考慮建立全文檢索索引（Oracle Text 索引）。
   - 將多個 `LIKE` 替換為正規表達式匹配：
     ```sql
     SELECT *
     FROM my_table
     WHERE REGEXP_LIKE(column1, '^abc|^xyz');
     ```
     注意這通常無法使用傳統索引，但在特定需求下可提升靈活性。

3. **測試與調整**：
   - 實際測試不同條件組合的查詢效能，特別是在大型資料集上。
   - 利用 SQL Trace 或 Oracle 的 SQL Tuning Advisor 分析瓶頸。

---

### **結論**
- **`LIKE + OR`** 效能在 `LIKE` 能使用索引時表現良好，但匹配字符過多或無索引支持時可能下降。
- **`IN`** 在需要完全匹配的情況下更適合，且效能穩定。

`IN` 是否需要進行 **full table scan**（全表掃描）取決於查詢條件是否能夠使用索引。如果 `IN` 的條件與索引匹配（如列上有合適的 B-tree 索引），則 Oracle 可以利用索引來進行高效查詢，而不需要掃描整個表。

然而，當 `IN` 的條件無法利用索引時，也會導致全表掃描，這與 `LIKE` 的情況類似。

---

### **`IN` 和 `INDEX` 的關係**
1. **能使用索引的情況**
   - 如果查詢列上有索引，且 `IN` 中的值是離散的常量匹配，Oracle 通常會選擇使用索引來尋找匹配的行。
   - 範例：
     ```sql
     SELECT * 
     FROM my_table
     WHERE column1 IN ('abc', 'xyz', 'def');
     ```
     假設 `column1` 上有 B-tree 索引，Oracle 可以通過索引快速定位 `'abc'`、`'xyz'` 和 `'def'`，避免全表掃描。

2. **無法使用索引的情況**
   - 如果列上沒有索引，或者 `IN` 的條件過於複雜，Oracle 可能選擇全表掃描。
   - 範例：
     - 查詢列沒有索引：
       ```sql
       SELECT * 
       FROM my_table
       WHERE column1 IN ('abc', 'xyz', 'def');
       ```
     - 查詢列是函數運算結果（無法直接使用索引）：
       ```sql
       SELECT * 
       FROM my_table
       WHERE LOWER(column1) IN ('abc', 'xyz');
       ```

### **`LIKE` 和 `IN` 效能比較（無法使用索引時）**
- **`LIKE`**：
  - 必須逐行掃描來匹配模式（如 `%abc%`）開頭為任意字元集％, *。
  
- **`IN`**：
  - 如果沒有索引，效能與全表掃描相似，但匹配過程是直接比對，通常比逐行模式匹配的 `LIKE` 快。

---


在處理多層式的 JOIN 或子查詢時，查詢效能和資料處理順序至關重要。確保內層查詢的過濾條件盡量減少資料量是一個重要的優化策略，但資料處理的順序需要依據具體的情況以及資料庫查詢優化器的執行計劃來調整。

以下是一些指南來確定多層查詢中資料過濾的順序和策略：

---

### **原則 1: 優先減少資料量**
**WHERE** 條件越靠近資料來源（如表或子查詢），資料庫可以越早過濾掉不需要的資料，從而降低後續處理的資料量和計算開銷。

- **優化策略**：
  1. 在子查詢或 JOIN 的內層優先添加有效的過濾條件。
  2. 利用索引，讓過濾條件盡可能執行在能使用索引的列上。

**範例：**
```sql
-- 未優化
SELECT * 
FROM large_table l
JOIN (
    SELECT * FROM filter_table WHERE condition = 'value'
) f ON l.id = f.id;

-- 優化：提早過濾 假設condition 有index 那效能好的變成小表 join 表示為inner join 基於on 的條件 然後資料庫inner join 會依照小表作為驅動表來查詢
SELECT * 
FROM large_table l
JOIN (
    SELECT id FROM filter_table WHERE condition = 'value'
) f ON l.id = f.id;

-- 像是上述只是要看large_Table 是否有存在數值id 沒有要看filter_table的數值 可以用where exists 來應用
-- 高效使用 WHERE EXISTS
SELECT *
FROM large_table l
WHERE EXISTS (
    SELECT 1 
    FROM filter_table f
    WHERE f.condition = 'value' AND l.id = f.id
);

```

### **3. 效能比較**
| 查詢方式                      | 優點                                                                                 | 潛在問題                                                                                |
|-------------------------------|-------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| **JOIN + 子查詢（整表）**     | 簡單直觀，適合需要子查詢返回多欄位的情況。                                             | 如果子查詢返回不必要的欄位，可能增加資料量和計算成本。                                   |
| **JOIN + 精簡子查詢（單欄位）**| 減少資料量，適合只需要用來比對的欄位（如 `id`）。                                     | 如果 `large_table` 和 `filter_table` 都很大，可能導致較高的中間結果集計算成本。          |
| **WHERE EXISTS**              | 高效，因為一旦找到匹配行就停止，且無需額外傳輸資料。                                   | 如果子查詢內條件過於複雜或索引缺失，可能導致效能不如預期。                               |

---

### **原則 2: 過濾條件的執行順序**
1. **資料量小的表或子查詢優先過濾**：
   - 如果某個表或子查詢的資料量較小，將其過濾後再與其他表或子查詢進行 JOIN，可以減少資料處理量。

2. **過濾條件範圍小的優先執行**：
   - WHERE 條件應按選擇性（Selectivity）優先排序，範圍小的條件能更有效地減少資料量。

**範例：**
```sql
-- 範圍大的條件先過濾（效能低）
SELECT * 
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE c.region = 'East' AND o.amount > 100;

-- 範圍小的條件先過濾（效能高）
SELECT * 
FROM orders o
JOIN (
    SELECT customer_id FROM customers WHERE region = 'East'
) c ON o.customer_id = c.customer_id
WHERE o.amount > 100;
```

---

### **原則 3: JOIN 順序的考量**
1. **小表優先**：
   - 如果一方的資料量明顯較少，將其作為驅動表進行 JOIN，可以減少中間結果的大小。

2. **過濾條件多的表優先**：
   - 先過濾條件更多的表，確保減少資料後再進行 JOIN。

**範例：**
```sql
-- 未優化：JOIN 順序處理大表
SELECT * 
FROM big_table b
JOIN small_table s ON b.id = s.id
WHERE b.condition = 'value';

-- 優化：以小表為驅動表
SELECT * 
FROM small_table s
JOIN big_table b ON b.id = s.id
WHERE b.condition = 'value';
```

假設：

big_table 有 1,000,000 行，small_table 有 1,000 行。
b.id 和 s.id 均有`索引 big O(log(n))`。
未優化的執行邏輯：

big_table（大表）作為驅動表，逐行處理每一行。
對於 big_table 的每一行，資料庫需要掃描 small_table。
匹配操作的次數接近 1,000,000 × log(1,000)（假設索引生效）。
優化後的執行邏輯：

small_table（小表）作為驅動表，逐行處理每一行。
對於 small_table 的每一行，資料庫檢索 big_table 中的匹配行。
匹配操作的次數接近 1,000 × log(1,000,000)。
結果：匹配次數和效能明顯不同。

---

### **原則 4: 減少中間結果**
- 避免返回過多未過濾的中間結果，這會增加臨時資料存儲和處理的開銷。
- 使用精確的過濾條件避免產生不必要的 Cartesian Join（笛卡爾積）。

**範例：**
```sql
-- 未優化：產生多餘中間結果
SELECT * 
FROM (SELECT * FROM large_table WHERE column_a = 'value') t1
JOIN another_large_table t2 ON t1.id = t2.id;

-- 優化：減少中間結果的大小
SELECT * 
FROM (SELECT id FROM large_table WHERE column_a = 'value') t1
JOIN another_large_table t2 ON t1.id = t2.id;
```

---
**判斷資料是否存在**，使用 `EXISTS` 是更高效且推薦的方法，而不是使用 `JOIN` 或 `COUNT` 的方式。這是因為 `EXISTS` 在找到第一筆符合條件的記錄後就會立即返回，避免了無謂的額外計算或資料掃描。

---

### **`EXISTS` 的優點**
1. **快速返回**：
   - 一旦找到符合條件的資料，查詢就會停止，無需處理整個結果集。
2. **避免多餘計算**：
   - 比 `COUNT` 更高效，因為 `COUNT` 需要計算整個結果集的大小，而 `EXISTS` 不需要。
3. **避免不必要的 JOIN**：
   - 相較於使用 `JOIN` 判斷存在性，`EXISTS` 只需檢查條件而不產生額外的 Cartesian Join（笛卡爾積）。

---

### **使用 `EXISTS` 判斷資料是否存在**

#### 範例 1: 基本用法
```sql
SELECT 1
FROM dual
WHERE EXISTS (
    SELECT 1
    FROM my_table
    WHERE column1 = 'value'
);
```
- **邏輯**：只要 `my_table` 中有至少一筆符合 `column1 = 'value'` 的資料，`EXISTS` 條件就會返回 `TRUE`。
- **執行效率**：一旦找到第一筆匹配資料，查詢就停止。

---

#### 範例 2: 多表關聯時使用 `EXISTS`
如果你需要檢查一個表是否有符合條件的相關記錄，使用 `EXISTS` 也是推薦的。

```sql
SELECT *
FROM main_table m
WHERE EXISTS (
    SELECT 1
    FROM related_table r
    WHERE r.main_id = m.id
      AND r.status = 'active'
);
```
- **邏輯**：只要 `related_table` 中有一筆 `main_id = m.id 且 status = 'active'` 的資料，則 `EXISTS` 條件為 `TRUE`。
- **優化**：這比用 `JOIN` 篩選後再判斷結果集更有效率。

---

#### 範例 3: 與 `COUNT` 的比較
如果只需判斷是否存在，不用 `COUNT > 0`，直接用 `EXISTS`：

```sql
-- 不推薦：使用 COUNT
SELECT *
FROM main_table
WHERE (
    SELECT COUNT(*)
    FROM related_table
    WHERE related_table.main_id = main_table.id
) > 0;

-- 推薦：使用 EXISTS
SELECT *
FROM main_table
WHERE EXISTS (
    SELECT 1
    FROM related_table
    WHERE related_table.main_id = main_table.id
);
```
- **原因**：`COUNT` 會計算所有匹配記錄，即使只需要知道是否存在，這會浪費資源。
- **`EXISTS` 更快**：因為它在第一個匹配結果時就結束查詢。

---

### **`EXISTS` 和索引的關係**
- 如果內層查詢中的條件可以利用索引，`EXISTS` 的效能會非常高，因為資料庫會通過索引快速找到匹配的記錄。
- 如果內層查詢無法使用索引，則仍需要進行全表掃描，但 `EXISTS` 仍然優於 `COUNT` 或不必要的 `JOIN`。


---
分頁查詢範例的目的是從資料表中分批檢索資料（例如：每次取出第 51 到第 100 筆資料）。
更好的解法：使用 OFFSET 和 FETCH
如果使用 Oracle 12c 或更新版本，可以改用 OFFSET 和 FETCH 語法，這更直觀且高效。
```sql
SELECT *
FROM employees
ORDER BY employee_id
OFFSET 50 ROWS FETCH NEXT 50 ROWS ONLY;
```

OFFSET 50 ROWS：跳過前 50 筆資料。
FETCH NEXT 50 ROWS ONLY：取接下來的 50 筆資料。

1. ROWNUM 的特性
Oracle 專屬：ROWNUM 是一個虛擬列，用來標識每一行的序列號，從 1 開始遞增。

Oracle 傳統上採用的是 ROWNUM，而不是直接提供 LIMIT。直到 Oracle 12c，才新增了類似的功能，使用 FETCH FIRST 和 OFFSET 關鍵字實現類似行為。


### ROWID
描述：
ROWID 是 Oracle 為每一行分配的一個唯一識別碼，用於標識該行在資料庫中的物理位置。
它是 Oracle 中的內建列，每行都有一個固定的 ROWID。
特性：
表示該行在資料文件中的存儲位置（檔案號、區塊號和行號）。
非常快，用於直接定位資料。
在特定條件下，可以通過 ROWID 執行精確刪除、更新操作。
即使資料內容相同，行的 ROWID 也不同。

---
在 Oracle 和許多其他資料庫中，當使用 `LEFT JOIN` 時，左邊的表通常被視為驅動表（**Driving Table**）。驅動表的選擇會影響執行計劃和效能，特別是在表的大小差異明顯的情況下。

---

### **1. 驅動表與 JOIN 的行為**

#### **LEFT JOIN**
```sql
SELECT *
FROM table_a a
LEFT JOIN table_b b
ON a.id = b.id;
```
- **左表 (table_a)**：
  - 被視為驅動表，資料庫首先會讀取左表中的資料。
  - 驅動表中的每一行會與右表 (`table_b`) 進行匹配。
  - 如果右表中沒有匹配的行，結果中右表的欄位將為 `NULL`。

#### **INNER JOIN**
```sql
SELECT *
FROM table_a a
JOIN table_b b
ON a.id = b.id;
```
- 在 `INNER JOIN` 中，通常沒有明確的驅動表，資料庫會根據統計資訊（如表大小、索引）動態決定哪個表作為驅動表。
- 使用提示（Hints）可以強制指定驅動表，例如 `/*+ LEADING(a) */`。

---

### **2. 驅動表的選擇原則**

#### **小表優先**
- **為什麼小表應該是驅動表？**
  - 驅動表的每一行都需要與另一個表中的行進行匹配。
  - 如果驅動表較小，會減少匹配操作的次數，從而提高效能。
  - 驅動表的大小直接影響中間結果集的大小，較小的結果集有助於減少後續計算成本。

#### **索引優先**
- 如果一方表有高效索引，且匹配的列可以用索引加速，該表應考慮作為被驅動表。

#### **根據查詢語句的邏輯**
- 在業務邏輯上，哪個表的篩選條件更多、更嚴格，應該考慮作為驅動表，以減少不必要的資料處理。

---

### **3. 控制驅動表的方式**

#### **默認行為**
- `LEFT JOIN`：左表為驅動表。
- `RIGHT JOIN`：右表為驅動表。

#### **使用提示（Hints）**
在 Oracle 中，可以使用 `LEADING` 提示強制指定驅動表。例如：
```sql
SELECT /*+ LEADING(a) */ *
FROM table_a a
JOIN table_b b
ON a.id = b.id;
```
- 強制指定 `table_a` 為驅動表。

#### **控制表的順序**
- 在一些資料庫（如 MySQL）中，表的順序會影響驅動表的選擇。
- 在 Oracle 中，資料庫的優化器可能會忽略表的物理順序，根據成本選擇最佳執行計劃。

---

### **4. 效能測試的建議**

- **檢查執行計劃**：
  使用 `EXPLAIN PLAN` 或 `AUTOTRACE` 查看執行計劃，確保資料庫選擇了合適的驅動表。
  ```sql
  EXPLAIN PLAN FOR
  SELECT *
  FROM table_a a
  LEFT JOIN table_b b
  ON a.id = b.id;
  ```

- **根據表大小進行調整**：
  如果資料庫未選擇最優驅動表，可以嘗試：
  - 增加索引。
  - 調整查詢結構（如使用子查詢提早過濾）。
  - 添加提示（Hints）。

---



```sql
CREATE OR REPLACE PROCEDURE manage_flag_updates
IS
BEGIN
  -- 將上一筆紀錄的 flag 更新為 'N'
  UPDATE your_table
  SET is_active = 0
  WHERE is_active = 1;

  -- 插入當前時間的新紀錄，flag 設為 'Y'
  INSERT INTO your_table (datetime_column, is_active)
  VALUES (SYSDATE, 1);

  -- 提交更改
  COMMIT;
END;
```

linux scheduler

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_JOB (
    job_name        => 'manage_flag_updates_job',
    job_type        => 'PLSQL_BLOCK',
    job_action      => 'BEGIN manage_flag_updates; END;',
    start_date      => SYSTIMESTAMP,
    repeat_interval => 'FREQ=MINUTELY; INTERVAL=5',
    enabled         => TRUE
  );
END;
/
```


```sql
BEGIN
   DBMS_SCHEDULER.create_job (
      job_name        => 'my_job',
      job_type        => 'PLSQL_BLOCK',
      job_action      => 'BEGIN your_procedure; END;',
      start_date      => TO_TIMESTAMP('2025-01-13 09:00:00', 'YYYY-MM-DD HH24:MI:SS'),
      repeat_interval => 'FREQ=MINUTELY;INTERVAL=5',
      enabled         => TRUE
   );
END;
```




https://medium.com/jimmy-wang/oracle基本修練-pl-sql-stored-procedures-and-functions-73b235631880

Schema可擁有資料庫中所有的型態的Object例如：表格（Table）、索引（Index）、視觀表（View）、循序項（Sequence）、同義字（Synonym）等等。



在 PL/SQL 或 Oracle 資料庫中，出現需要指定 `xxx.tablename` 的原因，主要與 Oracle 的結構和命名空間管理方式有關。以下是詳細解釋：  

---

### 1. **Oracle 的 Schema 概念**
   - 在 Oracle 中，**每個使用者**（User）實際上對應一個**Schema**。
   - **Schema** 是用來組織資料庫物件（如表格、索引、視圖等）的邏輯結構，每個表格都屬於某個 Schema。
   - 當你執行 `CREATE TABLE xxx.tablename` 時：
     - `xxx` 表示你在該 Schema 中建立表格。
     - 如果不指定 Schema（直接寫 `CREATE TABLE tablename`），則表格會被建立在你當前的 Schema 中，這取決於你連線到的使用者帳號。

---

### 2. **為什麼 Query 時可以不加 Schema**
   - 當你執行查詢（如 `SELECT * FROM tablename`）時，如果不指定 `xxx`：
     1. **默認 Schema**：Oracle 會嘗試在當前連線的使用者（即默認 Schema）中查找該表。
     2. **公用物件**：如果該表是一個 Public Synonym（公用同義詞），即一種全局可訪問的對象引用，Oracle 會根據同義詞自動解析到正確的 Schema 和表格。

---

### 3. **為什麼有時需要指定 Schema**
   - 如果你嘗試存取非當前 Schema 的表，則必須指定該 Schema 名稱，例如：`xxx.tablename`。
   - **原因可能是：**
     1. **不同使用者的 Schema**：你的連線使用者和表格屬於不同的 Schema，因此需要手動指定。
     2. **多重命名空間**：如果不同 Schema 中存在同名的表格（如 `xxx.tablename` 和 `yyy.tablename`），Oracle 無法確定要存取哪個，必須顯式指明。
     3. **授權限制**：只有在你擁有對其他 Schema 中表格的存取權限（如 `SELECT` 或 `INSERT` 權限）時，才能查詢該表格。

---

### 4. **與 MySQL 的比較**
   - 在 MySQL 中，`Database` 是資料庫的命名空間，所有表格都屬於某個 Database。
   - 你需要通過 `USE database_name` 切換工作空間，或者在表名前加上 `database_name.tablename` 明確指定。
   - 在 Oracle 中，沒有 `USE database` 這樣的命令，因為它的命名空間基於 Schema，而非 Database。Schema 與連線使用者密切相關。

---

### 5. **如何減少手動指定 Schema 的麻煩**
   - 如果你經常需要存取其他 Schema 的表，可以使用 **同義詞（Synonym）**：
     ```sql
     CREATE SYNONYM tablename FOR xxx.tablename;
     ```
     - 這樣在查詢時，只需要寫 `SELECT * FROM tablename` 即可。

   - 或者直接切換連線到目標 Schema 的使用者帳號。

---

### 總結
1. **建立表時需要指定 Schema**：因為 Oracle 需要明確知道要將表存在哪個命名空間中。
2. **查詢時不一定需要指定 Schema**：如果你連線的使用者正是表的擁有者，或存在公用同義詞，則可以省略 Schema。
3. **與 MySQL 的對比**：MySQL 使用 `Database` 作為命名空間，Oracle 使用 Schema，設計理念不同。


---

在 PL/SQL 下，你可以使用以下方法來了解你的身份以及擁有的資料表和權限：
在 Oracle 中，每個使用者帳號自動對應一個 Schema。

---

### 1. **確認你是誰（使用者資訊）**
   - 使用內建函數 `USER` 或 `SELECT` 查詢你的使用者名稱：
     ```sql
     SELECT USER FROM DUAL;
     ```
     **輸出**會顯示當前的 Oracle 使用者名稱，這對應到你的 Schema。

---

### 2. **查詢你擁有哪些表格**
   - 查詢當前 Schema 下你擁有的所有表格：
     ```sql
     SELECT TABLE_NAME 
     FROM USER_TABLES;
     ```
     **`USER_TABLES`** 是一個數據字典視圖，列出了當前使用者（Schema）擁有的表。

   - 如果你有權限存取其他 Schema 的表，則可以使用：
     ```sql
     SELECT OWNER, TABLE_NAME 
     FROM ALL_TABLES;
     ```
     **`ALL_TABLES`** 列出了當前使用者可以存取的所有表（包括其他 Schema 中授權的表）。

   - 如果你想查看資料庫中的所有表（需要 DBA 權限）：
     ```sql
     SELECT OWNER, TABLE_NAME 
     FROM DBA_TABLES;
     ```

---

### 3. **查詢你的權限**
   - 查詢當前使用者的系統權限（如建立表、刪除表等）：
     ```sql
     SELECT * 
     FROM USER_SYS_PRIVS;
     ```
     **`USER_SYS_PRIVS`** 列出了授予你的系統權限。

   - 查詢當前使用者的物件權限（如對哪些表有 SELECT、INSERT 等權限）：
     ```sql
     SELECT * 
     FROM USER_TAB_PRIVS;
     ```
     **`USER_TAB_PRIVS`** 列出了你對特定物件（如表、視圖等）的權限。

   - 查詢當前使用者的角色：
     ```sql
     SELECT * 
     FROM USER_ROLE_PRIVS;
     ```
     **`USER_ROLE_PRIVS`** 列出了授予你的角色，角色通常包含一組權限。

---

### 4. **範例分析**
假設你執行以下指令：
```sql
SELECT * 
FROM ALL_TABLES 
WHERE OWNER = 'HR';
```
這將顯示 Schema `HR` 中的所有表格，但前提是你對 `HR` 中的表具備存取權限。

---

### 5. **動態查看當前連線的資料庫與 Schema**
   - 確認當前連線的資料庫名稱：
     ```sql
     SELECT SYS_CONTEXT('USERENV', 'DB_NAME') AS DATABASE_NAME 
     FROM DUAL;
     ```
   - 確認當前 Schema 名稱：
     ```sql
     SELECT SYS_CONTEXT('USERENV', 'CURRENT_SCHEMA') AS SCHEMA_NAME 
     FROM DUAL;
     ```

---

**快速檢查你的存取權限**
   如果你只關注自己能存取哪些表，直接使用 `ALL_TAB_PRIVS` 就能滿足需求：
   ```sql
   SELECT TABLE_NAME, PRIVILEGE, OWNER
   FROM ALL_TAB_PRIVS
   WHERE GRANTEE = USER; -- 當前使用者是授權對象
   ```
   **輸出範例：**
   | TABLE_NAME | PRIVILEGE | OWNER  |
   |------------|-----------|--------|
   | EMPLOYEES  | SELECT    | HR     |
   | DEPARTMENTS| INSERT    | ADMIN  |

---

**`NLS_DATABASE_PARAMETERS`** 這個視圖並不是指某個特定 Schema 的設定，而是指整個 **Oracle 資料庫的 NLS（National Language Support）設定**。  

---

### 1. **`NLS_DATABASE_PARAMETERS` 的用途**
   - 這個視圖列出了與 **Oracle 資料庫層級的語言、區域設置** 和 **字符集設置** 相關的參數。它不針對單一的使用者或 Schema，而是整體資料庫的設定。
   - 你可以查詢資料庫的語言、區域設置、日期格式、字符集等參數。

---

### 2. **`NLS_DATABASE_PARAMETERS` 中的欄位**
   - **`PARAMETER`**：NLS 參數的名稱，例如 `NLS_LANGUAGE`, `NLS_TERRITORY`, `NLS_CHARACTERSET` 等。
   - **`VALUE`**：對應的設置值，例如 `AMERICAN`, `AMERICA`, `AL32UTF8` 等。

---

### 3. **常見的 NLS 設定**
以下是一些常見的 NLS 設定，會在 `NLS_DATABASE_PARAMETERS` 視圖中列出：

- **`NLS_LANGUAGE`**：資料庫使用的語言（例如 `AMERICAN`, `SIMPLIFIED CHINESE`）。
- **`NLS_TERRITORY`**：資料庫使用的地區（例如 `AMERICA`, `CHINA`）。
- **`NLS_DATE_FORMAT`**：資料庫的日期格式（例如 `YYYY-MM-DD`）。
- **`NLS_NUMERIC_CHARACTERS`**：資料庫的數字符號（例如 `.,` 表示小數點是 `.`，千分位符是 `,`）。
- **`NLS_CHARACTERSET`**：資料庫的字符集（例如 `AL32UTF8`、`WE8ISO8859P1` 等）。

---

### 4. **查詢範例**
若你希望檢視資料庫的 NLS 設定，可以執行以下 SQL 查詢：
```sql
SELECT * 
FROM NLS_DATABASE_PARAMETERS;
```

**範例結果：**
| PARAMETER           | VALUE          |
|---------------------|----------------|
| NLS_LANGUAGE        | AMERICAN       |
| NLS_TERRITORY       | AMERICA        |
| NLS_CHARACTERSET    | AL32UTF8       |
| NLS_DATE_FORMAT     | YYYY-MM-DD     |
| NLS_NUMERIC_CHARACTERS | . ,           |

---

### 5. **與 `NLS_SESSION_PARAMETERS` 的區別**
- **`NLS_SESSION_PARAMETERS`** 是針對當前 **會話** 的 NLS 設定，而不是整個資料庫。
- **`NLS_DATABASE_PARAMETERS`** 是針對 **資料庫級別** 的 NLS 設定。
  
如果你想要查看當前會話的設定，可以查詢 `NLS_SESSION_PARAMETERS`：
```sql
SELECT * 
FROM NLS_SESSION_PARAMETERS;
```

---

### 6. **設定層級**
- **資料庫級別**：由 `NLS_DATABASE_PARAMETERS` 顯示，這是 Oracle 資料庫啟動時的預設設定。
- **會話級別**：由 `NLS_SESSION_PARAMETERS` 顯示，這是當前連線的會話級別設置，通常可以在會話中修改這些設置。
- **使用者級別**：通常由 `NLS_LANG` 環境變數來決定，用於客戶端連線。

---

### 總結
- **`NLS_DATABASE_PARAMETERS`** 顯示的是資料庫層級的語言和區域設置。
- 如果你想查詢的是當前會話或使用者的設置，應該查詢 **`NLS_SESSION_PARAMETERS`** 或 **`USERENV`** 相關的參數。

如果還有其他問題，隨時可以問我！



上課完成 有一些概念想要用一些簡單題目來實作                  

1. schema 概念mfgdev3 ? 如果是有account直接連線到db ?
2. pl sql 能直接連線到db ? 
3. grant 權限
4. c# 的 app identify 要如何設定 先用hard core 方式
5. delphi 當中dm   bridge (dmon), datatable 這些元件能要如何使用寫一個簡單的範例 來連接db 
6. sql 小站 暫時先不用