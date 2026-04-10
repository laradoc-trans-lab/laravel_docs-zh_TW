# Laravel Scout

- [簡介](#introduction)
- [安裝](#installation)
    - [隊列](#queueing)
- [驅動器前置需求](#driver-prerequisites)
- [設定](#configuration)
    - [設定可搜尋資料](#configuring-searchable-data)
- [資料庫 / Collection 引擎](#database-and-collection-engines)
    - [資料庫引擎](#database-engine)
    - [Collection 引擎](#collection-engine)
- [第三方引擎設定](#third-party-engine-configuration)
    - [設定模型索引](#configuring-model-indexes)
    - [Algolia](#algolia-configuration)
    - [Meilisearch](#meilisearch-configuration)
    - [Typesense](#typesense-configuration)
- [第三方引擎索引](#indexing)
    - [批次匯入](#batch-import)
    - [新增紀錄](#adding-records)
    - [更新紀錄](#updating-records)
    - [移除紀錄](#removing-records)
    - [暫停索引](#pausing-indexing)
    - [有條件的可搜尋模型實體](#conditionally-searchable-model-instances)
- [搜尋](#searching)
    - [Where 子句](#where-clauses)
    - [分頁](#pagination)
    - [軟刪除](#soft-deleting)
    - [自定義引擎搜尋](#customizing-engine-searches)
    - [自定義引擎](#custom-engines)

<a name="introduction"></a>
## 簡介

[Laravel Scout](https://github.com/laravel/scout) 為您的 [Eloquent models](/docs/{{version}}/eloquent) 提供了一個簡單的、基於驅動器的全文搜尋方案。藉由模型觀察者，Scout 會自動將您的搜尋索引與您的 Eloquent 紀錄保持同步。

Scout 內建了一個 `database` 引擎，它使用 MySQL / PostgreSQL 的全文索引與 `LIKE` 子句來搜尋您現有的資料庫 —— 不需要外部服務。對於大多數應用程式來說，這就足夠了。如需瞭解 Laravel 中所有可用的搜尋選項，請參閱 [搜尋文件](/docs/{{version}}/search)。

當您在大規模環境下需要誤字容許 (Typo Tolerance)、分面篩選 (Faceted Filtering) 或地理搜尋 (Geo-search) 等功能時，Scout 也包含了 [Algolia](https://www.algolia.com/)、[Meilisearch](https://www.meilisearch.com) 與 [Typesense](https://typesense.org) 的驅動器。此外也提供了一個「collection」驅動器用於本地開發，您也可以自由地編寫 [自定義引擎](#custom-engines)。


<a name="installation"></a>
## 安裝

首先，經由 Composer 套件管理器安裝 Scout：

```shell
composer require laravel/scout
```

安裝 Scout 後，您應該使用 `vendor:publish` Artisan 指令來發布 Scout 的設定檔。此指令會將 `scout.php` 設定檔發布到您應用程式的 `config` 目錄中：

```shell
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

最後，將 `Laravel\Scout\Searchable` trait 加入到您想要使其可搜尋的模型中。此 trait 會註冊一個模型觀察者，該觀察者會自動讓模型與您的搜尋驅動器保持同步：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class Post extends Model
{
    use Searchable;
}
```


<a name="queueing"></a>
### 隊列

當使用的引擎不是 `database` 或 `collection` 引擎時，在使用此函式庫之前，您應該強烈考慮設定一個 [隊列驅動器](/docs/{{version}}/queues)。執行隊列工作者 (Queue Worker) 將允許 Scout 將所有同步模型資訊到搜尋索引的操作排入隊列，從而為您的應用程式 Web 介面提供更好的回應速度。

設定好隊列驅動器後，請將 `config/scout.php` 設定檔中的 `queue` 選項值設定為 `true`：

```php
'queue' => true,
```

即使 `queue` 選項設定為 `false`，請務必記住，某些 Scout 驅動器（如 Algolia 和 Meilisearch）始終是非同步地對紀錄建立索引。換句話說，即使索引操作已在您的 Laravel 應用程式內完成，搜尋引擎本身可能不會立即反映新增和更新的紀錄。

要指定您的 Scout 任務所使用的連線與隊列，您可以將 `queue` 設定選項定義為一個陣列：

```php
'queue' => [
    'connection' => 'redis',
    'queue' => 'scout'
],
```

當然，如果您自定義了 Scout 任務使用的連線與隊列，您應該執行隊列工作者來處理該連線與隊列上的任務：

```shell
php artisan queue:work redis --queue=scout
```


<a name="driver-prerequisites"></a>
## 驅動器前置需求


<a name="algolia"></a>
### Algolia

使用 Algolia 驅動器時，您應該在 `config/scout.php` 設定檔中設定您的 Algolia `id` 和 `secret` 憑證。設定好憑證後，您還需要經由 Composer 套件管理器安裝 Algolia PHP SDK：

```shell
composer require algolia/algoliasearch-client-php
```


<a name="meilisearch"></a>
### Meilisearch

[Meilisearch](https://www.meilisearch.com) 是一款快速且開源的搜尋引擎。如果您不確定如何在本地機器上安裝 Meilisearch，您可以使用 [Laravel Sail](/docs/{{version}}/sail#meilisearch)，這是 Laravel 官方支援的 Docker 開發環境。

使用 Meilisearch 驅動器時，您需要經由 Composer 套件管理器安裝 Meilisearch PHP SDK：

```shell
composer require meilisearch/meilisearch-php http-interop/http-factory-guzzle
```

接著，在您應用程式的 `.env` 檔案中設定 `SCOUT_DRIVER` 環境變數，以及您的 Meilisearch `host` 與 `key` 憑證：

```ini
SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=masterKey
```

如需更多關於 Meilisearch 的資訊，請參閱 [Meilisearch 文件](https://docs.meilisearch.com/learn/getting_started/quick_start.html)。

此外，您應該參閱 [Meilisearch 關於二進位版本相容性的文件](https://github.com/meilisearch/meilisearch-php#-compatibility-with-meilisearch)，確保安裝與您的 Meilisearch 二進位版本相容的 `meilisearch/meilisearch-php` 版本。

> [!WARNING]
> 在使用了 Meilisearch 的應用程式上更新 Scout 時，您應該始終[查看 Meilisearch 服務本身的任何額外重大變更](https://github.com/meilisearch/Meilisearch/releases)。


<a name="typesense"></a>
### Typesense

[Typesense](https://typesense.org) 是一款閃電般快速且開源的搜尋引擎，支援關鍵字搜尋、語義搜尋、地理搜尋與向量搜尋。

您可以[自行託管](https://typesense.org/docs/guide/install-typesense.html#option-2-local-machine-self-hosting) Typesense，或使用 [Typesense Cloud](https://cloud.typesense.org)。

要開始將 Typesense 與 Scout 搭配使用，請經由 Composer 套件管理器安裝 Typesense PHP SDK：

```shell
composer require typesense/typesense-php
```

接著，在您應用程式的 .env 檔案中設定 `SCOUT_DRIVER` 環境變數，以及您的 Typesense 主機與 API 金鑰憑證：

```ini
SCOUT_DRIVER=typesense
TYPESENSE_API_KEY=masterKey
TYPESENSE_HOST=localhost
```

如果您使用的是 [Laravel Sail](/docs/{{version}}/sail)，您可能需要調整 `TYPESENSE_HOST` 環境變數以匹配 Docker 容器名稱。您也可以選擇性地指定安裝的埠號、路徑與通訊協定：

```ini
TYPESENSE_PORT=8108
TYPESENSE_PATH=
TYPESENSE_PROTOCOL=http
```

您可以在應用程式的 `config/scout.php` 設定檔中找到 Typesense Collection 的額外設定與 Schema 定義。如需更多關於 Typesense 的資訊，請參閱 [Typesense 文件](https://typesense.org/docs/guide/#quick-start)。


<a name="configuration"></a>
## 設定


<a name="configuring-searchable-data"></a>
### 設定可搜尋資料

預設情況下，給定模型的整個 `toArray` 形式將會被持久化到其搜尋索引中。如果您想自定義同步到搜尋索引的資料，可以覆蓋模型上的 `toSearchableArray` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class Post extends Model
{
    use Searchable;

    /**
     * Get the indexable data array for the model.
     *
     * @return array<string, mixed>
     */
    public function toSearchableArray(): array
    {
        $array = $this->toArray();

        // Customize the data array...

        return $array;
    }
}
```


<a name="configuring-search-engines-per-model"></a>
#### 設定模型引擎

進行搜尋時，Scout 通常會使用應用程式 `scout` 設定檔中指定的預設搜尋引擎。但是，可以藉由覆蓋模型上的 `searchableUsing` 方法來更改特定模型的搜尋引擎：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Engines\Engine;
use Laravel\Scout\Scout;
use Laravel\Scout\Searchable;

class User extends Model
{
    use Searchable;

    /**
     * Get the engine used to index the model.
     */
    public function searchableUsing(): Engine
    {
        return Scout::engine('meilisearch');
    }
}
```

<a name="database-and-collection-engines"></a>
## 資料庫 / Collection 引擎

<a name="database-engine"></a>
### 資料庫引擎

> [!WARNING]
> 資料庫引擎目前支援 MySQL 和 PostgreSQL，兩者都提供了快速的全文本欄位索引支援。

`database` 引擎使用 MySQL / PostgreSQL 的全文本索引與 `LIKE` 子句直接搜尋現有的資料庫。對於許多應用程式來說，這是增加搜尋功能最簡單且最實用的方式 —— 不需要外部服務或額外的基礎設施。

要使用資料庫引擎，請將 `SCOUT_DRIVER` 環境變數設置為 `database`：

```ini
SCOUT_DRIVER=database
```

設定完成後，您可以[定義可搜尋資料](#configuring-searchable-data)並開始針對模型[執行搜尋查詢](#searching)。與第三方引擎不同，資料庫引擎不需要獨立的索引步驟 —— 它會直接搜尋您的資料庫資料表。

#### 自定義資料庫搜尋策略

預設情況下，資料庫引擎會對您[設定為可搜尋](#configuring-searchable-data)的每個模型屬性執行 `LIKE` 查詢。然而，您可以為特定欄位分配更有效率的搜尋策略。`SearchUsingFullText` 屬性將對該欄位使用資料庫的全文本索引，而 `SearchUsingPrefix` 則僅匹配字串的開頭 (`example%`)，而不是搜尋整個字串 (`%example%`)。

若要定義此行為，請將 PHP 屬性分配給模型的 `toSearchableArray` 方法。任何沒有屬性的欄位將繼續使用預設的 `LIKE` 策略：

```php
use Laravel\Scout\Attributes\SearchUsingFullText;
use Laravel\Scout\Attributes\SearchUsingPrefix;

/**
 * Get the indexable data array for the model.
 *
 * @return array<string, mixed>
 */
#[SearchUsingPrefix(['id', 'email'])]
#[SearchUsingFullText(['bio'])]
public function toSearchableArray(): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->email,
        'bio' => $this->bio,
    ];
}
```

> [!WARNING]
> 在指定欄位應使用全文本查詢約束之前，請確保該欄位已被分配了 [全文本索引](/docs/{{version}}/migrations#available-index-types)。

<a name="collection-engine"></a>
### Collection 引擎

"collection" 引擎適用於快速原型開發、極小規模的資料集（數百筆記錄）或執行測試。它從資料庫中檢索所有可能的記錄，並使用 Laravel 的 `Str::is` 輔助函式在 PHP 中進行過濾，因此它不需要任何索引或特定的資料庫功能。對於任何超出微不足道的案例，您應該改用 [資料庫引擎](#database-engine)。

要使用 collection 引擎，您可以簡單地將 `SCOUT_DRIVER` 環境變數的值設置為 `collection`，或直接在應用程式的 `scout` 設定檔中指定 `collection` 驅動器：

```ini
SCOUT_DRIVER=collection
```

當您將 collection 驅動器指定為偏好驅動器後，即可開始針對模型[執行搜尋查詢](#searching)。在使用 collection 引擎時，不需要進行搜尋引擎索引，例如填充 Algolia、Meilisearch 或 Typesense 索引所需的索引。

#### 與資料庫引擎的差異

雖然資料庫引擎使用全文本索引和 `LIKE` 子句來高效地尋找匹配記錄，但 collection 引擎會抓取所有記錄並在 PHP 中過濾。collection 引擎是最具移植性的選項，因為它適用於 Laravel 支援的所有關聯式資料庫（包括 SQLite 和 SQL Server）；然而，它的效率明顯低於資料庫引擎，且不應在大型資料集上使用。

<a name="third-party-engine-configuration"></a>
## 第三方引擎設定

以下設定選項僅在使用了 Algolia、Meilisearch 或 Typesense 等第三方搜尋引擎時才相關。如果您使用的是 [資料庫引擎](#database-engine)，則可以跳過本章節。


<a name="configuring-model-indexes"></a>
### 設定模型索引

使用第三方引擎時，每個 Eloquent 模型都會與指定的搜尋「索引 (index)」同步，該索引包含該模型的所有可搜尋紀錄。預設情況下，每個模型都會被持久化到與模型典型的「資料表」名稱相匹配的索引中。通常，這是模型名稱的複數形式；但是，您可以透過在模型上覆寫 `searchableAs` 方法來過自定義模型的索引：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class Post extends Model
{
    use Searchable;

    /**
     * Get the name of the index associated with the model.
     */
    public function searchableAs(): string
    {
        return 'posts_index';
    }
}
```

> [!NOTE]
> `searchableAs` 方法在編譯資料庫引擎時沒有作用，因為資料庫引擎總是直接搜尋模型的資料庫資料表。


<a name="configuring-the-model-id"></a>
#### 設定模型 ID

預設情況下，Scout 會將模型的主鍵作為存儲在搜尋索引中的模型唯一 ID / 鍵。如果您在使用第三方引擎時需要自定義此行為，可以覆寫模型上的 `getScoutKey` 和 `getScoutKeyName` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class User extends Model
{
    use Searchable;

    /**
     * Get the value used to index the model.
     */
    public function getScoutKey(): mixed
    {
        return $this->email;
    }

    /**
     * Get the key name used to index the model.
     */
    public function getScoutKeyName(): mixed
    {
        return 'email';
    }
}
```

> [!NOTE]
> `getScoutKey` 與 `getScoutKeyName` 方法在編譯資料庫引擎時沒有作用，因為資料庫引擎總是使用模型的主鍵。


<a name="algolia-configuration"></a>
### Algolia


<a name="algolia-index-settings"></a>
#### 索引設定

有時您可能希望在您的 Algolia 索引上設定額外的設定。雖然您可以透過 Algolia 的使用者介面 (UI) 管理這些設定，但直接在應用程式的 `config/scout.php` 設定檔中管理索引配置的期望狀態，有時會更有效率。

此方法允許您透過應用程式的自動化部署流程部署這些設定，避免手動配置並確保多個環境之間的一致性。您可以設定可過濾屬性 (filterable attributes)、排名 (ranking)、分面 (faceting) 或 [任何其他支援的設定](https://www.algolia.com/doc/rest-api/search/#tag/Indices/operation/setSettings)。

首先，在您的應用程式 `config/scout.php` 設定檔中為每個索引添加設定：

```php
use App\Models\User;
use App\Models\Flight;

'algolia' => [
    'id' => env('ALGOLIA_APP_ID', ''),
    'secret' => env('ALGOLIA_SECRET', ''),
    'index-settings' => [
        User::class => [
            'searchableAttributes' => ['id', 'name', 'email'],
            'attributesForFaceting'=> ['filterOnly(email)'],
            // Other settings fields...
        ],
        Flight::class => [
            'searchableAttributes'=> ['id', 'destination'],
        ],
    ],
],
```

如果指定索引底層的模型是可以軟刪除的，並且被包含在 `index-settings` 陣列中，Scout 將自動在該索引上包含對軟刪除模型的分面支援。如果您沒有為軟刪除模型的索引定義其他分面屬性，您只需在該模型的 `index-settings` 陣列中添加一個空項目即可：

```php
'index-settings' => [
    Flight::class => []
],
```

在設定好應用程式的索引設定後，您必須執行 `scout:sync-index-settings` Artisan 指令。此指令會將您目前配置的索引設定告知 Algolia。為了方便起見，您可能希望將此指令作為部署過程的一部分：

```shell
php artisan scout:sync-index-settings
```


<a name="algolia-identifying-users"></a>
#### 識別使用者

Scout 允許您在使用 Algolia 時自動識別使用者。將通過驗證的使用者與搜尋操作相關聯，在 Algolia 的儀表板中查看搜尋分析時可能會很有幫助。您可以透過在應用程式的 `.env` 檔案中將 `SCOUT_IDENTIFY` 環境變數定義為 `true` 來啟用使用者識別：

```ini
SCOUT_IDENTIFY=true
```

啟用此功能還會將請求的 IP 位址和通過驗證的使用者的主要識別碼傳遞給 Algolia，以便此資料與使用者發出的任何搜尋請求相關聯。


<a name="meilisearch-configuration"></a>
### Meilisearch


<a name="meilisearch-index-settings"></a>
#### 索引設定

Meilisearch 要求您預先定義索引搜尋設定，例如可過濾屬性 (filterable attributes)、可排序屬性 (sortable attributes) 以及 [其他支援的設定欄位](https://docs.meilisearch.com/reference/api/settings.html)。

可過濾屬性是您在呼叫 Scout 的 `where` 方法時打算用來過濾的任何屬性，而可排序屬性是您在呼叫 Scout 的 `orderBy` 方法時打算用來排序的任何屬性。要定義您的索引設定，請調整應用程式 `scout` 設定檔中 `meilisearch` 設定項目下的 `index-settings` 部分：

```php
use App\Models\User;
use App\Models\Flight;

'meilisearch' => [
    'host' => env('MEILISEARCH_HOST', 'http://localhost:7700'),
    'key' => env('MEILISEARCH_KEY', null),
    'index-settings' => [
        User::class => [
            'filterableAttributes'=> ['id', 'name', 'email'],
            'sortableAttributes' => ['created_at'],
            // Other settings fields...
        ],
        Flight::class => [
            'filterableAttributes'=> ['id', 'destination'],
            'sortableAttributes' => ['updated_at'],
        ],
    ],
],
```

如果指定索引底層的模型是可以軟刪除的，並且被包含在 `index-settings` 陣列中，Scout 將自動在該索引上包含對軟刪除模型的過濾支援。如果您沒有為軟刪除模型的索引定義其他可過濾或可排序屬性，您只需在該模型的 `index-settings` 陣列中添加一個空項目即可：

```php
'index-settings' => [
    Flight::class => []
],
```

在設定好應用程式的索引設定後，您必須執行 `scout:sync-index-settings` Artisan 指令。此指令會將您目前配置的索引設定告知 Meilisearch。為了方便起見，您可能希望將此指令作為部署過程的一部分：

```shell
php artisan scout:sync-index-settings
```


<a name="meilisearch-data-types"></a>
#### 可搜尋資料類型

Meilisearch 僅對正確類型的資料執行過濾操作（`>`、`<` 等）。在自定義可搜尋資料時，您應確保將數值轉換 (cast) 為正確的類型：

```php
public function toSearchableArray()
{
    return [
        'id' => (int) $this->id,
        'name' => $this->name,
        'price' => (float) $this->price,
    ];
}
```

<a name="typesense-configuration"></a>
### Typesense


<a name="typesense-searchable-data"></a>
#### 準備可搜尋資料

當使用 Typesense 時，您的可搜尋模型必須定義一個 `toSearchableArray` 方法，該方法會將模型的主鍵轉型為字串，並將建立日期轉型為 UNIX 時間戳記：

```php
/**
 * Get the indexable data array for the model.
 *
 * @return array<string, mixed>
 */
public function toSearchableArray(): array
{
    return array_merge($this->toArray(),[
        'id' => (string) $this->id,
        'created_at' => $this->created_at->timestamp,
    ]);
}
```

您還應該在應用程式的 `config/scout.php` 檔案中定義您的 Typesense Collection Schema。Collection Schema 描述了每個可透過 Typesense 搜尋的欄位的資料類型。有關所有可用 Schema 選項的更多資訊，請參閱 [Typesense 說明文件](https://typesense.org/docs/latest/api/collections.html#schema-parameters)。

如果您在定義 Typesense Collection 的 Schema 後需要變更它，您可以執行 `scout:flush` 和 `scout:import`，這將刪除所有現有的索引資料並重新建立 Schema。或者，您可以使用 Typesense 的 API 來修改 Collection 的 Schema，而無需刪除任何已索引的資料。

如果您的可搜尋模型具有軟刪除功能，您應該在應用程式的 `config/scout.php` 設定檔中，為該模型對應的 Typesense Schema 定義一個 `__soft_deleted` 欄位：

```php
User::class => [
    'collection-schema' => [
        'fields' => [
            // ...
            [
                'name' => '__soft_deleted',
                'type' => 'int32',
                'optional' => true,
            ],
        ],
    ],
],
```


<a name="typesense-dynamic-search-parameters"></a>
#### 動態搜尋參數

Typesense 允許您在透過 `options` 方法執行搜尋操作時，動態地修改您的 [搜尋參數](https://typesense.org/docs/latest/api/search.html#search-parameters)：

```php
use App\Models\Todo;

Todo::search('Groceries')->options([
    'query_by' => 'title, description'
])->get();
```

<a name="indexing"></a>
## 第三方引擎索引

> [!NOTE]
> 本節描述的索引功能主要適用於使用第三方引擎（Algolia、Meilisearch 或 Typesense）的情況。資料庫引擎直接搜尋您的資料庫資料表，因此不需要手動管理索引。


<a name="batch-import"></a>
### 批次匯入

如果您正在將 Scout 安裝到現有的專案中，可能已經有需要匯入到索引中的資料庫紀錄。Scout 提供了 `scout:import` Artisan 指令，您可以用它將所有現有紀錄匯入到搜尋索引中：

```shell
php artisan scout:import "App\Models\Post"
```

`scout:queue-import` 指令可用於使用[隊列任務](/docs/{{version}}/queues)匯入所有現有紀錄：

```shell
php artisan scout:queue-import "App\Models\Post" --chunk=500
```

`flush` 指令可用於從搜尋索引中移除模型的所有紀錄：

```shell
php artisan scout:flush "App\Models\Post"
```


<a name="modifying-the-import-query"></a>
#### 修改匯入查詢

如果您想修改用於檢索所有模型以進行批次匯入的查詢，您可以在模型上定義 `makeAllSearchableUsing` 方法。這是加入在匯入模型之前可能需要的任何關聯預載 (eager loading) 的絕佳位置：

```php
use Illuminate\Database\Eloquent\Builder;

/**
 * Modify the query used to retrieve models when making all of the models searchable.
 */
protected function makeAllSearchableUsing(Builder $query): Builder
{
    return $query->with('author');
}
```

> [!WARNING]
> 使用隊列批次匯入模型時，`makeAllSearchableUsing` 方法可能不適用。當模型集合由任務處理時，關聯[不會被恢復](/docs/{{version}}/queues#handling-relationships)。


<a name="adding-records"></a>
### 新增紀錄

一旦您將 `Laravel\Scout\Searchable` trait 加入到模型中，您只需要 `save` 或 `create` 一個模型實體，它就會自動被加入到您的搜尋索引中。如果您已將 Scout 設定為[使用隊列](#queueing)，此操作將由您的隊列工作者在背景執行：

```php
use App\Models\Order;

$order = new Order;

// ...

$order->save();
```


<a name="adding-records-via-query"></a>
#### 透過查詢新增紀錄

如果您想透過 Eloquent 查詢將模型集合加入到搜尋索引中，可以在 Eloquent 查詢鏈結 `searchable` 方法。`searchable` 方法會將查詢結果[分塊 (chunk)](/docs/{{version}}/eloquent#chunking-results) 並將紀錄加入到搜尋索引中。同樣地，如果您已將 Scout 設定為使用隊列，所有的分塊都將由您的隊列工作者在背景匯入：

```php
use App\Models\Order;

Order::where('price', '>', 100)->searchable();
```

您也可以在 Eloquent 關聯實體上呼叫 `searchable` 方法：

```php
$user->orders()->searchable();
```

或者，如果您記憶體中已經有一個 Eloquent 模型集合，可以在該集合實體上呼叫 `searchable` 方法，將模型實體加入到其對應的索引中：

```php
$orders->searchable();
```

> [!NOTE]
> `searchable` 方法可以被視為「upsert」操作。換句話說，如果模型紀錄已存在於索引中，它將被更新。如果它在搜尋索引中不存在，則會被加入到索引中。


<a name="updating-records"></a>
### 更新紀錄

要更新一個可搜尋的模型，您只需要更新模型實體的屬性並將模型 `save` 到資料庫。Scout 會自動將變更同步到您的搜尋索引中：

```php
use App\Models\Order;

$order = Order::find(1);

// Update the order...

$order->save();
```

您也可以在 Eloquent 查詢實體上呼叫 `searchable` 方法來更新模型集合。如果模型不存在於您的搜尋索引中，則會被建立：

```php
Order::where('price', '>', 100)->searchable();
```

如果您想更新關聯中所有模型的搜尋索引紀錄，可以在關聯實體上呼叫 `searchable`：

```php
$user->orders()->searchable();
```

或者，如果您記憶體中已經有一個 Eloquent 模型集合，可以在集合實體上呼叫 `searchable` 方法來更新其對應索引中的模型實體：

```php
$orders->searchable();
```


<a name="modifying-records-before-importing"></a>
#### 在匯入前修改紀錄

有時您可能需要在模型集合變為可搜尋之前對其進行準備。例如，您可能想要預載 (eager load) 一個關聯，以便將關聯資料有效地加入到您的搜尋索引中。若要實現此目的，請在相應的模型上定義 `makeSearchableUsing` 方法：

```php
use Illuminate\Database\Eloquent\Collection;

/**
 * Modify the collection of models being made searchable.
 */
public function makeSearchableUsing(Collection $models): Collection
{
    return $models->load('author');
}
```


<a name="conditionally-updating-the-search-index"></a>
#### 有條件地更新搜尋索引

預設情況下，Scout 會重新索引更新後的模型，無論修改了哪些屬性。如果您想自定義此行為，可以在模型上定義 `searchIndexShouldBeUpdated` 方法：

```php
/**
 * Determine if the search index should be updated.
 */
public function searchIndexShouldBeUpdated(): bool
{
    return $this->wasRecentlyCreated || $this->wasChanged(['title', 'body']);
}
```


<a name="removing-records"></a>
### 移除紀錄

要從索引中移除紀錄，您只需從資料庫中 `delete` 模型。即使您使用的是[軟刪除 (soft deleted)](/docs/{{version}}/eloquent#soft-deleting) 模型，也可以這樣做：

```php
use App\Models\Order;

$order = Order::find(1);

$order->delete();
```

如果您不想在刪除紀錄前先檢索模型，可以在 Eloquent 查詢實體上使用 `unsearchable` 方法：

```php
Order::where('price', '>', 100)->unsearchable();
```

如果您想移除關聯中所有模型的搜尋索引紀錄，可以在關聯實體上呼叫 `unsearchable`：

```php
$user->orders()->unsearchable();
```

或者，如果您記憶體中已經有一個 Eloquent 模型集合，可以在集合實體上呼叫 `unsearchable` 方法，從其對應的索引中移除模型實體：

```php
$orders->unsearchable();
```

若要從其對應的索引中移除所有模型紀錄，可以呼叫 `removeAllFromSearch` 方法：

```php
Order::removeAllFromSearch();
```


<a name="pausing-indexing"></a>
### 暫停索引

有時您可能需要對模型執行一批 Eloquent 操作，而不將模型資料同步到搜尋索引。您可以使用 `withoutSyncingToSearch` 方法來做到這一點。此方法接受一個閉包 (closure)，該閉包將立即被執行。閉包內發生的任何模型操作都不會同步到模型的索引：

```php
use App\Models\Order;

Order::withoutSyncingToSearch(function () {
    // Perform model actions...
});
```

<a name="conditionally-searchable-model-instances"></a>
### 有條件的可搜尋模型實體

有時您可能只需要在特定條件下讓模型可被搜尋。例如，假設您有一個 `App\Models\Post` 模型，它可能處於「草稿 (draft)」與「已發佈 (published)」這兩種狀態之一。您可能只想讓「已發佈」的文章可以被搜尋。要達成此目的，您可以在模型中定義一個 `shouldBeSearchable` 方法：

```php
/**
 * Determine if the model should be searchable.
 */
public function shouldBeSearchable(): bool
{
    return $this->isPublished();
}
```

`shouldBeSearchable` 方法僅在使用 `save` 與 `create` 方法、查詢或關聯性操作模型時才會套用。直接使用 `searchable` 方法使模型或集合變為可搜尋，將會覆蓋 `shouldBeSearchable` 方法的結果。

> [!WARNING]
> 當使用 Scout 的「database」引擎時，`shouldBeSearchable` 方法並不適用，因為所有可搜尋的資料始終存儲在資料庫中。若要在使用資料庫引擎時達成類似行為，您應該改用 [where 子句](#where-clauses)。

<a name="searching"></a>
## 搜尋

您可以使用 `search` 方法開始搜尋模型。搜尋方法接受一個用於搜尋模型的字串。接著您應該在搜尋查詢後串接 `get` 方法，以取得與給定搜尋查詢相符的 Eloquent 模型：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->get();
```

由於 Scout 搜尋會回傳 Eloquent 模型的集合，因此您甚至可以直接從路由或控制器回傳結果，它們將自動轉換為 JSON：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/search', function (Request $request) {
    return Order::search($request->search)->get();
});
```

如果您想在轉換為 Eloquent 模型之前取得原始搜尋結果，可以使用 `raw` 方法：

```php
$orders = Order::search('Star Trek')->raw();
```


<a name="custom-indexes"></a>
#### 自定義索引

使用第三方引擎進行搜尋時，搜尋查詢通常會在模型的 [searchableAs](#configuring-model-indexes) 方法所指定的索引上執行。但是，您可以使用 `within` 方法來指定應該搜尋的自定義索引：

```php
$orders = Order::search('Star Trek')
    ->within('tv_shows_popularity_desc')
    ->get();
```


<a name="where-clauses"></a>
### Where 子句

Scout 允許您在搜尋查詢中加入簡單的 "where" 子句。目前這些子句僅支援基本的相等檢查，主要用於依據擁有者 ID 來限制搜尋查詢的範圍：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->where('user_id', 1)->get();
```

此外，`whereIn` 方法可用於驗證給定欄位的值是否包含在給定的陣列中：

```php
$orders = Order::search('Star Trek')->whereIn(
    'status', ['open', 'paid']
)->get();
```

`whereNotIn` 方法則驗證給定欄位的值是否不包含在給定的陣列中：

```php
$orders = Order::search('Star Trek')->whereNotIn(
    'status', ['closed']
)->get();
```

> [!WARNING]
> 如果您的應用程式正在使用 Meilisearch，您必須在利用 Scout 的 "where" 子句之前先設定應用程式的 [可過濾屬性](#meilisearch-index-settings)。


<a name="customizing-the-eloquent-results-query"></a>
#### 自定義 Eloquent 結果查詢

在 Scout 從應用程式的搜尋引擎取得符合的 Eloquent 模型清單後，會接著使用 Eloquent 透過主鍵取得所有符合的模型。您可以透過呼叫 `query` 方法來對此查詢進行自定義。`query` 方法接受一個閉包，該閉包將接收 Eloquent 查詢建構器實體作為參數：

```php
use App\Models\Order;
use Illuminate\Database\Eloquent\Builder;

$orders = Order::search('Star Trek')
    ->query(fn (Builder $query) => $query->with('invoices'))
    ->get();
```

使用第三方引擎時，此回呼是在相關模型已經從搜尋引擎取得後才被呼叫，因此不應用於「過濾」結果 — 請改用 [Scout where 子句](#where-clauses)。然而，使用資料庫引擎時，`query` 方法的約束會直接套用於資料庫查詢，因此您也可以將其用於過濾。


<a name="pagination"></a>
### 分頁

除了取得模型集合之外，您還可以使用 `paginate` 方法對搜尋結果進行分頁。此方法將回傳一個 `Illuminate\Pagination\LengthAwarePaginator` 實體，就像您對 [傳統的 Eloquent 查詢進行分頁](/docs/{{version}}/pagination) 一樣：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->paginate();
```

您可以透過將數量作為第一個參數傳遞給 `paginate` 方法來指定每頁要取得多少模型：

```php
$orders = Order::search('Star Trek')->paginate(15);
```

使用資料庫引擎時，您還可以使用 `simplePaginate` 方法。與 `paginate` 不同，`paginate` 會取得相符紀錄的總數以便顯示頁碼，而 simplePaginate 僅判斷目前頁面之後是否還有更多結果 — 這對於您只需要「上一頁」和「下一頁」連結的大型資料集來說更有效率：

```php
$orders = Order::search('Star Trek')->simplePaginate(15);
```

取得結果後，您可以使用 [Blade](/docs/{{version}}/blade) 顯示結果並渲染頁面連結，就像您對傳統的 Eloquent 查詢進行分頁一樣：

```html
<div class="container">
    @foreach ($orders as $order)
        {{ $order->price }}
    @endforeach
</div>

{{ $orders->links() }}
```

當然，如果您想以 JSON 格式取得分頁結果，可以直接從路由或控制器回傳分頁器實體：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    return Order::search($request->input('query'))->paginate(15);
});
```

> [!WARNING]
> 由於搜尋引擎不知道您的 Eloquent 模型的全域範圍 (Global Scope) 定義，因此在利用 Scout 分頁的應用程式中，您不應該使用全域範圍。或者，您應該在透過 Scout 搜尋時重新建立全域範圍的約束。


<a name="soft-deleting"></a>
### 軟刪除

如果您索引的模型支援 [軟刪除](/docs/{{version}}/eloquent#soft-deleting)，且您需要搜尋已軟刪除的模型，請將 `config/scout.php` 設定檔中的 `soft_delete` 選項設定為 `true`：

```php
'soft_delete' => true,
```

當此設定選項為 `true` 時，Scout 不會從搜尋索引中移除已軟刪除的模型。相反地，它會在索引紀錄上設定一個隱藏的 `__soft_deleted` 屬性。接著，您可以在搜尋時使用 `withTrashed` 或 `onlyTrashed` 方法來取得已軟刪除的紀錄：

```php
use App\Models\Order;

// Include trashed records when retrieving results...
$orders = Order::search('Star Trek')->withTrashed()->get();

// Only include trashed records when retrieving results...
$orders = Order::search('Star Trek')->onlyTrashed()->get();
```

> [!NOTE]
> 當一個已軟刪除的模型使用 `forceDelete` 被永久刪除時，Scout 會自動將其從搜尋索引中移除。


<a name="customizing-engine-searches"></a>
### 自定義引擎搜尋

如果您需要對引擎的搜尋行為進行進階自定義，可以將一個閉包作為第二個參數傳遞給 `search` 方法。例如，您可以使用此回呼在搜尋查詢傳遞給 Algolia 之前，將地理位置資料加入搜尋選項中：

```php
use Algolia\AlgoliaSearch\SearchIndex;
use App\Models\Order;

Order::search(
    'Star Trek',
    function (SearchIndex $algolia, string $query, array $options) {
        $options['body']['query']['bool']['filter']['geo_distance'] = [
            'distance' => '1000km',
            'location' => ['lat' => 36, 'lon' => 111],
        ];

        return $algolia->search($query, $options);
    }
)->get();
```

<a name="custom-engines"></a>
## 自定義引擎


<a name="writing-the-engine"></a>
#### 撰寫引擎

如果內建的 Scout 搜尋引擎不符合您的需求，您可以撰寫自己的自定義引擎並將其註冊到 Scout。您的引擎應繼承 `Laravel\Scout\Engines\Engine` 抽象類別。這個抽象類別包含了八個您的自定義引擎必須實作的方法：

```php
use Laravel\Scout\Builder;

abstract public function update($models);
abstract public function delete($models);
abstract public function search(Builder $builder);
abstract public function paginate(Builder $builder, $perPage, $page);
abstract public function mapIds($results);
abstract public function map(Builder $builder, $results, $model);
abstract public function getTotalCount($results);
abstract public function flush($model);
```

您可能會發現查看 `Laravel\Scout\Engines\AlgoliaEngine` 類別中這些方法的實作會有所幫助。該類別將為您學習如何在自己的引擎中實作這些方法提供一個良好的起點。


<a name="registering-the-engine"></a>
#### 註冊引擎

一旦撰寫好您的自定義引擎，您可以使用 Scout 引擎管理器的 `extend` 方法將其註冊到 Scout。Scout 的引擎管理器可以從 Laravel 服務容器中解析出來。您應該在 `App\Providers\AppServiceProvider` 類別或應用程式使用的任何其他服務提供者的 `boot` 方法中呼叫 `extend` 方法：

```php
use App\ScoutExtensions\MySqlSearchEngine;
use Laravel\Scout\EngineManager;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    resolve(EngineManager::class)->extend('mysql', function () {
        return new MySqlSearchEngine;
    });
}
```

一旦註冊了您的引擎，您就可以在應用程式的 `config/scout.php` 設定檔中將其指定為預設的 Scout `driver`：

```php
'driver' => 'mysql',
```