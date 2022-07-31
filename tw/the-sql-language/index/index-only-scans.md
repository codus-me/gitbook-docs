# 11.11. 索引限定查詢（Index-only scan）

PostgreSQL 中的所有索引都是二級索引，這意味著每個索引都與資料的主資料區域（在 PostgreSQL 術語中稱為資料表的 heap）分開儲存。這意味著在普通索引掃描中，每個資料列檢索都需要從索引和堆中獲取資料。此外，雖然與給予可索引 WHERE 條件匹配的索引項目通常在索引中靠近在一起，但它們引用的資料列可能在 heap 中的任何位置。因此，索引掃描 heap 部分涉及大量隨機存取，這可能很慢，特別是在傳統的磁碟媒體上。（如第 11.5 節所述，bitmap 掃描嘗試透過按排序循序執行 heap 存取來減輕此成本，但這只是到目前為止而已。）

為了解決這個效能問題，PostgreSQL 支援索引限定掃描，它可以單獨回答索引中的查詢而毋須任何 heap 存取。基本思維是直接從每個索引項目中回傳值，而不是查詢相關的 heap 項目。何時可以使用此方法有兩個基本限制：

1. 索引類型必須支援僅索引掃描。B-tree 索引能做到，GiST 和 SP-GiST 索引支援某些運算子類的僅索引掃描，但不支持其他運算子類。其他索引類型則沒有支援。 基本要求是索引必須物理儲存或能夠重建每個索引項目的原始資料值。作為一個反例，GIN 索引不支援索引限定掃描，因為每個索引項目通常只包含原始資料值的一部分。
2. 查詢必須僅引用儲存在索引中的欄位。例如，給予一個也有欄位 z 的資料表欄位 x 和 y 的索引，這些查詢可以使用索引限定掃描：

   ```text
   SELECT x, y FROM tab WHERE x = 'key';
   SELECT x FROM tab WHERE x = 'key' AND y < 42;
   ```

   但這些查詢不能：

   ```text
   SELECT x, z FROM tab WHERE x = 'key';
   SELECT x FROM tab WHERE x = 'key' AND z < 42;
   ```

   （表示式索引和部分索引使此規則複雜化，如下所述。）

只要滿足這兩個基本要求，那麼查詢所需的所有資料值都可以從索引中獲得，因此就能只進行索引掃描。但是對 PostgreSQL 中的任何資料表掃描還有一個額外的要求：它必須驗證每個檢索到的資料列對查詢的 MVCC 快照是「可見的」，如[第 13 章](../concurrency-control/)所述。可見性訊息不儲存在索引項目中，僅儲存在 heap 項目中；所以乍看之下似乎每個資料列檢索都需要存取 heap。如果資料列最近被修改，情況確實如此。但是，對於很少變化的資料，可以解決這個問題。對於資料表 heap 中的每個頁面，PostgreSQL 追踪儲存在該頁面中的所有資料表是否足夠大以使所有目前和未來的事務都可見。此訊息儲存在資料表的可見性映射表中。在找到候選索引項目之後，索引限定掃描會檢查相應 heap 頁面的可見性映射表。如果已設定，則該資料列已知可見，因此可以回傳資料而毋須進一步的作業。如果未設定，則必須存取 heap 項目以查明它是否可見，因此與標準索引掃描相比沒有效能優勢。即使在成功的情況下，這種方法也會對 heap 存取進行可見性映射表存取；但由於可見性圖比它描述的 heap 小四個數量級，因此存取它所需的物理 I/O 要少得多。在大多數情況下，可見性映射表始終在暫存在記憶體中。

簡而言之，雖然在給予兩個基本要求的情況下可以進行索引限定掃描，但只有當資料表的 heap 頁面的很大一部分設定了全部可見的映射位元時，才會獲勝。但是大部分資料列不變的資料通常足以使這種類型的掃描成立，這在實作中非常有用。

要有效使用索引限定掃描功能，您可以選擇建立僅在前導欄位中匹配 WHERE 子句的索引，而尾隨列包含要由查詢回傳的“payload”資料。例如，如果你經常執行像這樣的查詢

```text
SELECT y FROM tab WHERE x = 'key';
```

加速此類查詢的傳統方法是僅在 x 上建立索引。但是，\(x, y\) 上的索引將提供將此查詢實作為索引限定掃描的可能性。 如前所述，這樣的索引會更大，因此比單獨使用 x 的索引更昂貴，所以只有在知道該資表大部分是靜態的情況下才有吸引力。 請注意，在 \(x, y\) 而不是 \(y, x\) 上宣告索引很重要，因為大多數索引類型（特別是B-tree）不限制前導索引欄位的搜尋效率不高。

原則上，索引限定掃描可以與表示式索引一起使用。例如，給予 f\(x\) 上的索引，其中 x 是資料表欄位，應該可以執行

```text
SELECT f(x) FROM tab WHERE f(x) < 1;
```

作為索引限定掃描；如果 f\(\) 是一個昂貴的計算函數，這是非常有吸引力的。但是，PostgreSQL 的規劃程序目前對這種情況並不十分聰明。只有當查詢所需的所有欄位都可以從索引獲得時，它才會將查詢視為可能透過索引限定進行掃描。在此範例中，除了裡面的 f\(x\) 之外，不需要 x，但是規劃程序沒有注意到這一點，並得出結論進行索引限定掃描。如果索引限定掃描似乎足夠值得，可以通過宣告索引打開 \(f\(x\), x\) 來解決這個問題，其中第二欄位預計不會在實作中使用，而只是說服了規劃程序可以進行索引限定掃描。如果目標是避免重新計算 f\(x\)，則另一個警告是規劃程序不一定要匹配不在索引欄位的可索引 WHERE 子句中的 f\(x\) 使用。它通常會在如上所示的簡單查詢中得到正確的結果，但在涉及 JOIN 的查詢中則不會。這些缺陷也許會在 PostgreSQL 的未來版本中得到改善。

部分索引還與索引限定掃描具有有趣的交互運作。考慮[範例 11.3 ](partial-indexes.md)中顯示的部分索引：

```text
CREATE UNIQUE INDEX tests_success_constraint ON tests (subject, target)
    WHERE success;
```

原則上，我們可以對此索引執行索引限定掃描就能滿足查詢

```text
SELECT target FROM tests WHERE subject = 'some-subject' AND success;
```

但是存在一個問題：WHERE 子句參考的 success，它不能作為索引的結果欄位。 儘管如此，仍能進行索引限定掃描，因為計劃不需要在執行時重新檢查 WHERE 子句的那一部分：索引中找到的所有項目必須具有 success = true，因此毋須在計劃中明確檢查。 PostgreSQL 版本 9.6 及更高版本能識別此類情況並允許産生索引限定掃描，但舊版本不會。
