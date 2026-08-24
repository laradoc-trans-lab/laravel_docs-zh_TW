# 搜尋

- [簡介](#introduction)
    - [全文檢索](#introduction-full-text-search)
    - [語意 / 向量搜尋](#introduction-semantic-vector-search)
    - [重新排序](#introduction-reranking)
    - [Scout 搜尋引擎](#introduction-scout-search-engines)
- [全文檢索](#full-text-search)
    - [新增全文索引](#adding-full-text-indexes)
    - [執行全文檢索查詢](#running-full-text-queries)
- [語意 / 向量搜尋](#semantic-vector-search)
    - [產生 Embedding](#generating-embeddings)
    - [儲存與建立向量索引](#storing-and-indexing-vectors)
    - [依相似度查詢](#querying-by-similarity)
- [重新排序結果](#reranking-results)
- [Laravel Scout](#laravel-scout)
    - [資料庫引擎](#database-engine)
    - [第三方引擎](#third-party-engines)
- [結合多種技巧](#combining-techniques)

<a name="introduction"></a>
## 簡介

幾乎每個應用程式都需要搜尋功能。無論您的使用者是在知識庫中搜尋相關文章、探索產品目錄，還是在文件語料庫中提出自然語言問題，Laravel 都提供了內建工具來處理這些情境——而且您通常不需要任何外部服務就能達成目的。

大多數應用程式會發現 Laravel 提供的內建資料庫搜尋選項就已經非常充裕——只有當您在大規模情境下需要打字容錯（typo tolerance）、分面篩選（faceted filtering）或地理位置搜尋（geo-search）等功能時，才需要外部搜尋服務。


<a name="introduction-full-text-search"></a>
#### 全文檢索

當您需要關鍵字相關性排序——由資料庫根據結果與搜尋詞的匹配程度進行評分與排序時——Laravel 的 `whereFullText` 查詢建構器方法利用了 MariaDB、MySQL 和 PostgreSQL 的原生全文索引。全文檢索能理解詞彙邊界與詞幹提取（stemming），因此搜尋 "running" 可以匹配到包含 "run" 的紀錄。完全不需要外部服務。


<a name="introduction-semantic-vector-search"></a>
#### 語意 / 向量搜尋

對於透過 *意義* 而非精確關鍵字來匹配結果的 AI 驅動語意搜尋，`whereVectorSimilarTo` 查詢建構器方法使用了儲存在帶有 `pgvector` 擴充套件之 PostgreSQL 中的向量 Embedding。例如，搜尋 "best wineries in Napa Valley" 可以找出標題為 "Top Vineyards to Visit" 的文章——即使文字完全沒有重疊。向量搜尋需要帶有 `pgvector` 擴充套件的 PostgreSQL 以及 [Laravel AI SDK](/docs/{{version}}/ai-sdk)。


<a name="introduction-reranking"></a>
#### 重新排序

Laravel 的 [AI SDK](/docs/{{version}}/ai-sdk) 提供了重新排序功能，利用 AI 模型根據與查詢的語意相關性來重新排序任何結果集。重新排序特別適合放在快速的初始檢索步驟（如全文檢索）之後作為第二階段使用——同時為您提供速度與語意精準度。


<a name="introduction-scout-search-engines"></a>
#### Scout 搜尋引擎

對於希望透過 `Searchable` Trait 來自動保持搜尋索引與 Eloquent 模型同步的應用程式，[Laravel Scout](/docs/{{version}}/scout) 同時提供了內建的資料庫引擎以及 Algolia、Meilisearch 和 Typesense 等第三方服務的驅動程式。


<a name="full-text-search"></a>
## 全文檢索

雖然 `LIKE` 查詢在簡單的子字串匹配上表現良好，但它們無法理解語言。對 "running" 進行 `LIKE` 搜尋找不到包含 "run" 的紀錄，而且結果不會依據相關性進行排序——資料庫找到什麼順序就直接回傳什麼順序。全文檢索透過使用了解詞彙邊界、詞幹提取與相關性評分的專用索引解決了這兩個問題，讓資料庫能夠優先回傳最相關的結果。

快速的全文檢索已內建於 MariaDB、MySQL 和 PostgreSQL 中——不需要外部搜尋服務。您只需要在想要搜尋的欄位上新增全文索引，然後使用 `whereFullText` 查詢建構器方法對其進行搜尋即可。

> [!WARNING]
> 全文檢索目前支援 MariaDB、MySQL 和 PostgreSQL。


<a name="adding-full-text-indexes"></a>
### 新增全文索引

若要使用全文檢索，首先請在您想要搜尋的欄位新增全文索引。您可以將索引新增至單一欄位，或是傳入欄位陣列以建立能一次搜尋多個欄位的複合索引：

```php
Schema::create('articles', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('body');
    $table->timestamps();

    $table->fullText(['title', 'body']);
});
```

在 PostgreSQL 上，您可以為索引指定語言設定，這會控制詞彙如何進行詞幹提取：

```php
$table->fullText('body')->language('english');
```

關於建立索引的更多資訊，請參閱 [Migration 文件](/docs/{{version}}/migrations#available-index-types)。


<a name="running-full-text-queries"></a>
### 執行全文檢索查詢

當索引設定完畢後，即可使用 `whereFullText` 查詢建構器方法對其進行搜尋。Laravel 會為您的資料庫驅動程式產生適當的 SQL——例如在 MariaDB 與 MySQL 上產生 `MATCH(...) AGAINST(...)`，以及在 PostgreSQL 上產生 `to_tsvector(...) @@ plainto_tsquery(...)`：

```php
$articles = Article::whereFullText('body', 'web developer')->get();
```

在使用 MariaDB 和 MySQL 時，結果會自動依相關性分數進行排序。在 PostgreSQL 上，`whereFullText` 會篩選匹配的紀錄，但不會依相關性為其排序——如果您在 PostgreSQL 上需要自動依相關性排序，可以考慮使用 [Scout 的資料庫引擎](#database-engine)，它會為您處理這個問題。

如果您跨多個欄位建立了複合全文索引，可以藉由向 `whereFullText` 傳入相同的欄位陣列來同時搜尋所有欄位：

```php
$articles = Article::whereFullText(
    ['title', 'body'], 'web developer'
)->get();
```

`orWhereFullText` 方法可用於新增全文檢索子句作為「or」條件。完整細節請參閱[查詢建構器文件](/docs/{{version}}/queries#full-text-where-clauses)。

<a name="semantic-vector-search"></a>
## 語意 / 向量搜尋

全文檢索依賴關鍵字比對——查詢中的字詞必須（以某種形式）出現在資料中。語意搜尋則採用截然不同的方法：它使用 AI 產生的向量 Embedding 來將文字的*意義*表示為數字陣列，接著尋找意義與查詢最相似的結果。例如，搜尋「best wineries in Napa Valley」可以找出標題為「Top Vineyards to Visit」的文章——即使這些字詞完全沒有重疊。

向量搜尋的基本流程為：為每段內容產生 Embedding（數字陣列）並與資料一併儲存，接著在搜尋時，為使用者的查詢產生 Embedding，並找出在向量空間中與其最接近的已儲存 Embedding。

> [!NOTE]
> 向量搜尋需要 [Laravel AI SDK](/docs/{{version}}/ai-sdk)，並由 PostgreSQL（需要 `pgvector` 擴充功能）和 MongoDB（需要 [Laravel MongoDB 套件](https://laravel.com/docs/13.x/mongodb)）所支援。所有在 [Laravel Cloud](https://laravel.com/cloud) 上的 Postgres 資料庫均已預先安裝 `pgvector`。

<a name="generating-embeddings"></a>
### 產生 Embedding

Embedding 是高維度的數字陣列（通常有數百或數千個數字），代表了一段文字的語意。您可以使用 Laravel `Stringable` 類別提供的 `toEmbeddings` 方法來為字串產生 Embedding：

```php
use Illuminate\Support\Str;

$embedding = Str::of('Napa Valley has great wine.')->toEmbeddings();
```

若要一次為多個輸入產生 Embedding（這比逐一產生更有效率，因為只需要對 Embedding 提供者進行單次 API 呼叫），請使用 `Embeddings` 類別：

```php
use Laravel\Ai\Embeddings;

$response = Embeddings::for([
    'Napa Valley has great wine.',
    'Laravel is a PHP framework.',
])->generate();

$response->embeddings; // [[0.123, 0.456, ...], [0.789, 0.012, ...]]
```

有關設定 Embedding 提供者、自訂維度與快取的更多詳細資訊，請參閱 [AI SDK 文件](/docs/{{version}}/ai-sdk#embeddings)。

<a name="storing-and-indexing-vectors"></a>
### 儲存與建立向量索引

若要儲存向量 Embedding，請在 Migration 中定義一個 `vector` 欄位，並指定符合 Embedding 提供者產出的維度數量（例如 OpenAI 的 `text-embedding-3-small` 模型為 1536）。您還應該在該欄位上呼叫 `index` 來建立 HNSW（Hierarchical Navigable Small World）索引，這能顯著提升大型資料集上的相似度搜尋速度：

```php
Schema::ensureVectorExtensionExists();

Schema::create('documents', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('content');
    $table->vector('embedding', dimensions: 1536)->index();
    $table->timestamps();
});
```

`Schema::ensureVectorExtensionExists` 方法可確保在建立資料表之前，您的 PostgreSQL 資料庫已啟用 `pgvector` 擴充功能。

在您的 Eloquent 模型上，將向量欄位型別轉換（Cast）為 `array`，這樣 Laravel 就會自動處理 PHP 陣列與資料庫向量格式之間的轉換：

```php
protected function casts(): array
{
    return [
        'embedding' => 'array',
    ];
}
```

有關向量欄位與索引的更多詳細資訊，請參閱 [Migration 文件](/docs/{{version}}/migrations#available-column-types)。

<a name="querying-by-similarity"></a>
### 依相似度查詢

當您為內容儲存 Embedding 後，就可以使用 `whereVectorSimilarTo` 方法搜尋相似的紀錄。此方法使用餘弦相似度（Cosine similarity）將指定的 Embedding 與已儲存的向量進行比較，過濾掉低於 `minSimilarity` 門檻值的結果，並自動按相關性對結果進行排序——最相似的紀錄排在最前面。門檻值應介於 `0.0` 到 `1.0` 之間，其中 `1.0` 表示向量完全相同：

```php
$documents = Document::query()
    ->whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4)
    ->limit(10)
    ->get();
```

為提供便利性，當傳入純字串而非 Embedding 陣列時，Laravel 會自動使用您設定的 Embedding 提供者為您產生 Embedding。這意味著您可以直接傳入使用者的搜尋查詢，而無需先手動將其轉換為 Embedding：

```php
$documents = Document::query()
    ->whereVectorSimilarTo('embedding', 'best wineries in Napa Valley')
    ->limit(10)
    ->get();
```

若要對向量查詢進行更低階的控制，也可以使用 `whereVectorDistanceLessThan`、`selectVectorDistance` 和 `orderByVectorDistance` 方法。這些方法讓您可以直接操作距離值而非相似度分數、將計算出的距離選取為結果中的欄位，或是手動控制排序。有關完整詳細資訊，請參閱[查詢建構器文件](/docs/{{version}}/queries#vector-similarity-clauses)與 [AI SDK 文件](/docs/{{version}}/ai-sdk#querying-embeddings)。

<a name="reranking-results"></a>
## 重新排序結果

重新排序（Reranking）是一種由 AI 模型根據每個結果與指定查詢的語意相關性，重新對一組結果進行排序的技巧。與需要預先計算並儲存 Embedding 的向量搜尋不同，重新排序適用於任何文字集合——它接收原始內容與查詢作為輸入，並回傳按相關性排序後的項目。

重新排序在快速的初始檢索步驟之後作為第二階段特別強大。例如，您可以先使用全文檢索快速將數千筆紀錄縮減為前 50 個候選項目，然後使用重新排序將最相關的結果置頂。這種「先檢索後重排（Retrieve then rerank）」的模式兼具速度與語意精準度。

您可以使用 `Reranking` 類別對字串陣列進行重新排序：

```php
use Laravel\Ai\Reranking;

$response = Reranking::of([
    'Django is a Python web framework.',
    'Laravel is a PHP web application framework.',
    'React is a JavaScript library for building user interfaces.',
])->rerank('PHP frameworks');

$response->first()->document; // "Laravel is a PHP web application framework."
```

Laravel Collection 也提供了一個 `rerank` 巨集（Macro），可以接收欄位名稱（或 Closure）與查詢，讓重新排序 Eloquent 結果變得非常簡單：

```php
$articles = Article::all()
    ->rerank('body', 'Laravel tutorials');
```

有關設定重新排序提供者與可用選項的完整詳細資訊，請參閱 [AI SDK 文件](/docs/{{version}}/ai-sdk#reranking)。

<a name="laravel-scout"></a>
## Laravel Scout

上述說明的搜尋技巧都是您在程式碼中直接呼叫的查詢建構器（Query Builder）方法。[Laravel Scout](/docs/{{version}}/scout) 則採取了不同的做法：它提供了一個 `Searchable` Trait 讓您加到 Eloquent Model 中，而且 Scout 會在建立、更新和刪除紀錄時，自動將您的搜尋索引保持同步。當您希望 Model 始終能被搜尋，又不想手動管理索引更新時，這項功能特別方便。

<a name="database-engine"></a>
### 資料庫引擎

Scout 內建的資料庫引擎能對您現有的資料庫執行全文檢索與 `LIKE` 搜尋 — 無需任何外部服務或額外基礎架構。只需將 `Searchable` Trait 加至您的 Model，並定義一個回傳您想要開放搜尋之欄位的 `toSearchableArray` 方法即可。

您可以使用 PHP Attribute 來控制每個欄位的搜尋策略。`SearchUsingFullText` 會使用資料庫的全文索引，`SearchUsingPrefix` 僅會從字串開頭進行比對（`example%`），而任何沒有設定 Attribute 的欄位則會使用預設的 `LIKE` 策略，並在兩側加上通配符（`%example%`）：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Attributes\SearchUsingFullText;
use Laravel\Scout\Attributes\SearchUsingPrefix;
use Laravel\Scout\Searchable;

class Article extends Model
{
    use Searchable;

    #[SearchUsingPrefix(['id'])]
    #[SearchUsingFullText(['title', 'body'])]
    public function toSearchableArray(): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'body' => $this->body,
        ];
    }
}
```

> [!WARNING]
> 在指定欄位應使用全文檢索查詢約束之前，請確保該欄位已被指派[全文索引](/docs/{{version}}/migrations#available-index-types)。

加入 Trait 後，您就可以使用 Scout 的 `search` 方法來搜尋 Model。Scout 的資料庫引擎會自動按相關性對結果排序，即使在 PostgreSQL 上也是如此：

```php
$articles = Article::search('Laravel')->get();
```

當您的搜尋需求屬於中等程度，且希望能有 Scout 自動同步索引的便利性又不想部署外部服務時，資料庫引擎是一個絕佳的選擇。它能妥善處理最常見的搜尋使用情境，包括篩選、分頁以及軟刪除（Soft-deleted）紀錄的處理。如需完整的詳細資訊，請參考 [Scout 文件](/docs/{{version}}/scout#database-engine)。

<a name="third-party-engines"></a>
### 第三方引擎

Scout 也支援第三方搜尋引擎，例如 [Algolia](https://www.algolia.com/)、[Meilisearch](https://www.meilisearch.com) 以及 [Typesense](https://typesense.org)。這些專用的搜尋服務提供了諸如錯字容忍（Typo tolerance）、分面篩選（Faceted filtering）、地理位置搜尋（Geo-search）以及自訂排序規則等進階功能 — 這些功能在極大規模或需要高度順暢的邊打字邊搜尋（Search-as-you-type）體驗時會變得非常重要。

由於 Scout 在其所有驅動程式之間提供了統一的 API，因此日後從資料庫引擎切換到第三方引擎時，只需要修改極少量的程式碼。您可以先從資料庫引擎開始，僅在您的應用程式需求超出資料庫所能提供的範疇時，再遷移到第三方服務。

如需設定第三方引擎的完整詳細資訊，請參考 [Scout 文件](/docs/{{version}}/scout)。

> [!NOTE]
> 許多應用程式其實從不需要外部搜尋引擎。本頁說明的內建技巧已涵蓋了絕大多數的使用情境。

<a name="combining-techniques"></a>
## 結合多種技巧

本頁說明的搜尋技巧並非相互排斥 — 將它們結合使用通常能獲得最佳效果。以下是展示這些工具如何協同工作的兩種常見模式。

**全文檢索擷取 + 重新排序**

使用全文檢索快速將大量資料集縮減為候選集合，接著套用重新排序來按語意相關性排序這些候選項目。這能讓您同時擁有資料庫原生全文檢索的速度，以及 AI 驅動相關性計分的精確度：

```php
$articles = Article::query()
    ->whereFullText('body', $request->input('query'))
    ->limit(50)
    ->get()
    ->rerank('body', $request->input('query'), limit: 10);
```

**向量搜尋 + 傳統篩選條件**

將向量相似度與標準的 `where` 子句相結合，以將語意搜尋限制在紀錄的特定子集中。當您想要基於文意進行搜尋，但又需要依據擁有者權限、分類或任何其他屬性來限制結果時，這非常實用：

```php
$documents = Document::query()
    ->where('team_id', $user->team_id)
    ->whereVectorSimilarTo('embedding', $request->input('query'))
    ->limit(10)
    ->get();
```