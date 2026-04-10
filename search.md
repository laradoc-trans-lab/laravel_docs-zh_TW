# 搜尋

- [簡介](#introduction)
    - [全文檢索](#introduction-full-text-search)
    - [語義 / 向量搜尋](#introduction-semantic-vector-search)
    - [重新排序](#introduction-reranking)
    - [Scout 搜尋引擎](#introduction-scout-search-engines)
- [全文檢索](#full-text-search)
    - [新增全文檢索索引](#adding-full-text-indexes)
    - [執行全文檢索查詢](#running-full-text-queries)
- [語義 / 向量搜尋](#semantic-vector-search)
    - [產生嵌入向量](#generating-embeddings)
    - [儲存與索引向量](#storing-and-indexing-vectors)
    - [依相似度查詢](#querying-by-similarity)
- [重新排序結果](#reranking-results)
- [Laravel Scout](#laravel-scout)
    - [資料庫引擎](#database-engine)
    - [第三方引擎](#third-party-engines)
- [結合技術](#combining-techniques)

<a name="introduction"></a>
## 簡介

幾乎每個應用程式都需要搜尋功能。無論您的使用者是在知識庫中搜尋相關文件、瀏覽產品目錄，還是針對文件集提出自然語言問題，Laravel 都提供了內建工具來處理這些情境——而且您通常不需要任何外部服務即可實現。

大多數應用程式會發現 Laravel 提供的內建資料庫驅動選項已經綽綽有餘——只有在您需要大規模的拼寫容錯、分面篩選 (Faceted Filtering) 或地理搜尋等功能時，才需要外部搜尋服務。


<a name="introduction-full-text-search"></a>
#### 全文檢索

當您需要關鍵字相關性排序時——即資料庫根據結果與搜尋字詞的匹配程度進行評分與排序——Laravel 的 `whereFullText` 查詢產生器方法利用了 MariaDB、MySQL 和 PostgreSQL 上的原生全文檢索索引。全文檢索能夠辨識詞界與詞幹提取 (Stemming)，因此搜尋 "running" 可以匹配包含 "run" 的紀錄。不需要任何外部服務。


<a name="introduction-semantic-vector-search"></a>
#### 語義 / 向量搜尋

對於透過 *意義* 而非精確關鍵字來匹配結果的 AI 驅動語義搜尋，可以使用 `whereVectorSimilarTo` 查詢產生器方法，它使用儲存在帶有 `pgvector` 擴充功能的 PostgreSQL 中的向量嵌入 (Vector Embeddings)。例如，搜尋 "best wineries in Napa Valley" 可以帶出標題為 "Top Vineyards to Visit" 的文章——即使兩者的單字沒有重疊。向量搜尋需要帶有 `pgvector` 擴充功能的 PostgreSQL 以及 [Laravel AI SDK](/docs/{{version}}/ai-sdk)。


<a name="introduction-reranking"></a>
#### 重新排序

Laravel 的 [AI SDK](/docs/{{version}}/ai-sdk) 提供了重新排序功能，使用 AI 模型根據對查詢的語義相關性來重新排列任何結果集。重新排序作為快速初始檢索步驟（如全文檢索）後的第二階段特別強大——讓您同時兼顧速度與語義準確性。


<a name="introduction-scout-search-engines"></a>
#### Scout 搜尋引擎

對於想要使用 `Searchable` trait 來自動保持搜尋索引與 Eloquent 模型同步的應用程式，[Laravel Scout](/docs/{{version}}/scout) 同時提供了內建的資料庫引擎以及 Algolia、Meilisearch 和 Typesense 等第三方服務的驅動程式。


<a name="full-text-search"></a>
## 全文檢索

雖然 `LIKE` 查詢在簡單的子字串匹配中表現良好，但它們不理解語言。對 "running" 進行 `LIKE` 搜尋將找不到包含 "run" 的紀錄，且結果不會按相關性排序——它們只是按照資料庫找到的任何順序回傳。全文檢索透過使用專門的索引來解決這兩個問題，這些索引能夠理解詞界、詞幹提取和相關性評分，讓資料庫能夠優先回傳最相關的結果。

快速的全文檢索已內建於 MariaDB、MySQL 和 PostgreSQL 中——不需要外部搜尋服務。您只需要為想要搜尋的欄位新增全文檢索索引，然後使用 `whereFullText` 查詢產生器方法對其進行搜尋即可。

> [!WARNING]
> 全文檢索目前由 MariaDB、MySQL 和 PostgreSQL 支援。


<a name="adding-full-text-indexes"></a>
### 新增全文檢索索引

要使用全文檢索，首先要為您想要搜尋的欄位新增全文檢索索引。您可以將索引新增到單一欄位，或傳遞一個欄位陣列來建立一個同時搜尋多個欄位的複合索引：

```php
Schema::create('articles', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('body');
    $table->timestamps();

    $table->fullText(['title', 'body']);
});
```

在 PostgreSQL 上，您可以為索引指定語言配置，這會控制詞幹提取的方式：

```php
$table->fullText('body')->language('english');
```

有關建立索引的更多資訊，請參閱 [遷移文件](/docs/{{version}}/migrations#available-index-types)。


<a name="running-full-text-queries"></a>
### 執行全文檢索查詢

索引就緒後，請使用 `whereFullText` 查詢產生器方法對其進行搜尋。Laravel 將為您的資料庫驅動程式產生適當的 SQL——例如，在 MariaDB 和 MySQL 上為 `MATCH(...) AGAINST(...)`，在 PostgreSQL 上為 `to_tsvector(...) @@ plainto_tsquery(...)`：

```php
$articles = Article::whereFullText('body', 'web developer')->get();
```

使用 MariaDB 和 MySQL 時，結果會自動按相關性分數排序。在 PostgreSQL 上，`whereFullText` 會篩選匹配的紀錄，但不會按相關性對其進行排序——如果您在 PostgreSQL 上需要自動相關性排序，請考慮使用 [Scout 的資料庫引擎](#database-engine)，它會為您處理。

如果您建立了跨多個欄位的複合全文檢索索引，您可以透過將相同的欄位陣列傳遞給 `whereFullText` 來對所有欄位進行搜尋：

```php
$articles = Article::whereFullText(
    ['title', 'body'], 'web developer'
)->get();
```

`orWhereFullText` 方法可用於將全文檢索搜尋子句作為 "or" 條件新增。完整詳細資訊請參閱 [查詢產生器文件](/docs/{{version}}/queries#full-text-where-clauses)。

<a name="semantic-vector-search"></a>
## 語義 / 向量搜尋

全文檢索依賴於關鍵字比對——查詢中的字詞必須（以某種形式）出現在資料中。語義搜尋則採取了截然不同的方法：它使用 AI 產生的向量嵌入 (Vector Embeddings) 將文字的「意義」表示為數字陣列，然後找出其意義與查詢最相似的結果。例如，搜尋 「best wineries in Napa Valley」 可能會出現標題為 「Top Vineyards to Visit」 的文章——即便字面上沒有任何重疊。

向量搜尋的基本工作流程是：為每一段內容產生一個嵌入向量（數字陣列）並將其與資料一起儲存，接著在搜尋時，為使用者的查詢產生一個嵌入向量，並在向量空間中找出與其最接近的已儲存嵌入向量。

> [!NOTE]
> 向量搜尋需要使用帶有 `pgvector` 擴充功能的 PostgreSQL 資料庫以及 [Laravel AI SDK](/docs/{{version}}/ai-sdk)。所有 [Laravel Cloud](https://cloud.laravel.com) Serverless Postgres 資料庫都已內建 `pgvector`。

<a name="generating-embeddings"></a>
### 產生嵌入向量

嵌入向量是一個高維度的數字陣列（通常包含數百或數千個數字），代表了一段文字的語義意義。您可以使用 Laravel 的 `Stringable` 類別中提供的 `toEmbeddings` 方法來為字串產生嵌入向量：

```php
use Illuminate\Support\Str;

$embedding = Str::of('Napa Valley has great wine.')->toEmbeddings();
```

若要一次為多個輸入產生嵌入向量——這比逐一產生更有效率，因為它只需要對嵌入提供者進行一次 API 呼叫——請使用 `Embeddings` 類別：

```php
use Laravel\Ai\Embeddings;

$response = Embeddings::for([
    'Napa Valley has great wine.',
    'Laravel is a PHP framework.',
])->generate();

$response->embeddings; // [[0.123, 0.456, ...], [0.789, 0.012, ...]]
```

有關設定嵌入提供者、自定義維度與快取的更多細節，請參閱 [AI SDK 文件](/docs/{{version}}/ai-sdk#embeddings)。

<a name="storing-and-indexing-vectors"></a>
### 儲存與索引向量

要儲存向量嵌入，請在遷移 (Migration) 中定義一個 `vector` 欄位，並指定與您的嵌入提供者輸出相符的維度數量（例如，OpenAI 的 `text-embedding-3-small` 模型為 1536）。您還應該在該欄位上呼叫 `index` 以建立 HNSW (Hierarchical Navigable Small World) 索引，這能顯著加快大型資料集上的相似度搜尋：

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

在您的 Eloquent 模型上，將向量欄位轉換為 `array`，以便 Laravel 自動處理 PHP 陣列與資料庫向量格式之間的轉換：

```php
protected function casts(): array
{
    return [
        'embedding' => 'array',
    ];
}
```

有關向量欄位與索引的更多細節，請參閱 [遷移文件](/docs/{{version}}/migrations#available-column-types)。

<a name="querying-by-similarity"></a>
### 依相似度查詢

儲存內容的嵌入向量後，您可以使用 `whereVectorSimilarTo` 方法搜尋相似的紀錄。此方法使用餘弦相似度 (Cosine Similarity) 將指定的嵌入向量與儲存的向量進行比較，過濾掉低於 `minSimilarity` 閾值的結果，並自動按相關性排序——最相似的紀錄排在最前面。閾值應介於 `0.0` 到 `1.0` 之間，其中 `1.0` 代表向量完全相同：

```php
$documents = Document::query()
    ->whereVectorSimilarTo('embedding', $queryEmbedding, minSimilarity: 0.4)
    ->limit(10)
    ->get();
```

為了方便起見，當提供純字串而非嵌入陣列時，Laravel 會自動使用您設定的嵌入提供者為您產生嵌入向量。這意味著您可以直接傳入使用者的搜尋查詢，而無需手動將其轉換為嵌入向量：

```php
$documents = Document::query()
    ->whereVectorSimilarTo('embedding', 'best wineries in Napa Valley')
    ->limit(10)
    ->get();
```

若要對向量查詢進行更低階的控制，還可以使用 `whereVectorDistanceLessThan`、`selectVectorDistance` 與 `orderByVectorDistance` 方法。這些方法讓您可以直接處理距離值而非相似度分數，將計算出的距離作為結果中的一個欄位選取，或是手動控制排序。有關完整細節，請參閱 [查詢建構器文件](/docs/{{version}}/queries#vector-similarity-clauses) 與 [AI SDK 文件](/docs/{{version}}/ai-sdk#querying-embeddings)。

<a name="reranking-results"></a>
## 重新排序結果

重新排序 (Reranking) 是一項技術，由 AI 模型根據每個結果與給定查詢的語義相關程度，對一組結果進行重新排序。與向量搜尋不同（向量搜尋需要您預先計算並儲存嵌入向量），重新排序適用於任何文字集合——它將原始內容和查詢作為輸入，並回傳按相關性排序後的項目。

重新排序作為快速初步檢索後的第二階段特別強大。例如，您可以使用全文檢索將數千條紀錄快速縮小到前 50 個候選項目，然後使用重新排序將最相關的結果置於頂部。這種「先檢索後重新排序」的模式兼顧了速度與語義準確性。

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

Laravel 集合 (Collections) 也有一個 `rerank` 巨集，它接受欄位名稱（或閉包）和一個查詢，讓您可以輕鬆地對 Eloquent 結果進行重新排序：

```php
$articles = Article::all()
    ->rerank('body', 'Laravel tutorials');
```

有關設定重新排序提供者與可用選項的完整細節，請參閱 [AI SDK 文件](/docs/{{version}}/ai-sdk#reranking)。

<a name="laravel-scout"></a>
## Laravel Scout

上述描述的搜尋技術都是您在程式碼中直接呼叫的查詢建構器 (Query Builder) 方法。[Laravel Scout](/docs/{{version}}/scout) 採取了不同的方法：它提供了一個您加入到 Eloquent 模型中的 `Searchable` trait，Scout 會在紀錄建立、更新及刪除時自動保持搜尋索引同步。當您希望模型始終可供搜尋而無需手動管理索引更新時，這特別方便。


<a name="database-engine"></a>
### 資料庫引擎

Scout 內建的資料庫引擎對您現有的資料庫執行全文檢索與 `LIKE` 搜尋 — 無需外部服務或額外的基礎設施。只需將 `Searchable` trait 加入到您的模型中，並定義一個回傳您希望可搜尋欄位的 `toSearchableArray` 方法即可。

您可以使用 PHP 屬性 (Attributes) 來控制每個欄位的搜尋策略。`SearchUsingFullText` 將使用您資料庫的全文檢索索引，`SearchUsingPrefix` 將僅從字串開頭進行比對 (`example%`)，而任何不帶屬性的欄位則使用預設的 `LIKE` 策略，兩側皆帶有萬用字元 (`%example%`)：

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
> 在指定欄位應使用全文檢索查詢限制之前，請確保該欄位已被分配了一個 [全文檢索索引](/docs/{{version}}/migrations#available-index-types)。

加入 trait 後，您可以使用 Scout 的 `search` 方法來搜尋您的模型。Scout 的資料庫引擎將自動按相關性排序結果，即使在 PostgreSQL 上也是如此：

```php
$articles = Article::search('Laravel')->get();
```

當您的搜尋需求適中，且您希望在不部署外部服務的情況下享受 Scout 自動索引同步的便利時，資料庫引擎是一個絕佳的選擇。它能很好地處理最常見的搜尋使用案例，包括過濾、分頁和軟刪除紀錄處理。如需完整詳細資訊，請參閱 [Scout 文件](/docs/{{version}}/scout#database-engine)。


<a name="third-party-engines"></a>
### 第三方引擎

Scout 還支援第三方搜尋引擎，例如 [Algolia](https://www.algolia.com/)、[Meilisearch](https://www.meilisearch.com) 與 [Typesense](https://typesense.org)。這些專用的搜尋服務提供進階功能，如拼字糾錯 (Typo Tolerance)、分面篩選 (Faceted Filtering)、地理位置搜尋以及自定義排序規則 — 當規模非常大或是您需要高度精緻的「隨打即搜 (Search-as-you-type)」體驗時，這些功能會變得非常重要。

由於 Scout 在其所有驅動程式中提供了統一的 API，因此日後從資料庫引擎切換到第三方引擎只需要極少的程式碼變更。您可以先從資料庫引擎開始，僅在應用程式的需求超出資料庫所能提供的範圍時，再遷移到第三方服務。

關於設定第三方引擎的完整詳細資訊，請參閱 [Scout 文件](/docs/{{version}}/scout)。

> [!NOTE]
> 許多應用程式永遠不需要外部搜尋引擎。本頁面描述的內建技術涵蓋了絕大多數的使用案例。


<a name="combining-techniques"></a>
## 結合技術

本頁面描述的搜尋技術並非互斥的 — 將它們結合使用通常會產生最佳效果。以下是展示這些工具如何協作的兩種常見模式。

**全文檢索擷取 + 重新排序**

使用全文檢索搜尋快速將大型資料集縮小到候選集合中，然後應用重新排序根據語義相關性對這些候選者進行排序。這讓您既能擁有資料庫原生全文檢索的速度，又能擁有 AI 驅動的相關性評分準確性：

```php
$articles = Article::query()
    ->whereFullText('body', $request->input('query'))
    ->limit(50)
    ->get()
    ->rerank('body', $request->input('query'), limit: 10);
```

**向量搜尋 + 傳統過濾器**

將向量相似度與標準的 `where` 子句結合，將語義搜尋限制在紀錄的子集中。這在您想要基於意義進行搜尋，但需要按所有權、類別或任何其他屬性限制結果時非常有用：

```php
$documents = Document::query()
    ->where('team_id', $user->team_id)
    ->whereVectorSimilarTo('embedding', $request->input('query'))
    ->limit(10)
    ->get();
```