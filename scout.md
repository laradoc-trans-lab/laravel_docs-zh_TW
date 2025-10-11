# Laravel Scout

- [簡介](#introduction)
- [安裝](#installation)
    - [佇列](#queueing)
- [驅動程式先決條件](#driver-prerequisites)
    - [Algolia](#algolia)
    - [Meilisearch](#meilisearch)
    - [Typesense](#typesense)
- [設定](#configuration)
    - [設定 Model 索引](#configuring-model-indexes)
    - [設定可搜尋資料](#configuring-searchable-data)
    - [設定 Model ID](#configuring-the-model-id)
    - [設定各 Model 的搜尋引擎](#configuring-search-engines-per-model)
    - [識別使用者](#identifying-users)
- [資料庫 / Collection 引擎](#database-and-collection-engines)
    - [資料庫引擎](#database-engine)
    - [Collection 引擎](#collection-engine)
- [索引](#indexing)
    - [批次匯入](#batch-import)
    - [新增紀錄](#adding-records)
    - [更新紀錄](#updating-records)
    - [移除紀錄](#removing-records)
    - [暫停索引](#pausing-indexing)
    - [條件式可搜尋 Model 實例](#conditionally-searchable-model-instances)
- [搜尋](#searching)
    - [Where 子句](#where-clauses)
    - [分頁](#pagination)
    - [軟刪除](#soft-deleting)
    - [自訂引擎搜尋](#customizing-engine-searches)
- [自訂引擎](#custom-engines)

<a name="introduction"></a>
## 簡介

[Laravel Scout](https://github.com/laravel/scout) 提供了一個簡單、基於驅動程式的解決方案，可以為您的 [Eloquent models](/docs/{{version}}/eloquent) 加入全文搜尋功能。透過 Model 觀察者 (model observers)，Scout 會自動使您的搜尋索引與 Eloquent 紀錄保持同步。

目前，Scout 內建支援 [Algolia](https://www.algolia.com/)、[Meilisearch](https://www.meilisearch.com)、[Typesense](https://typesense.org) 以及 MySQL / PostgreSQL (``database``) 驅動程式。此外，Scout 還包含一個「collection」驅動程式，專為本機開發用途設計，不需要任何外部依賴或第三方服務。而且，編寫自訂驅動程式也很簡單，您可以自由地透過您自己的搜尋實作來擴充 Scout。

<a name="installation"></a>
## 安裝

首先，透過 Composer 套件管理器安裝 Scout：

```shell
composer require laravel/scout
```

安裝 Scout 之後，您應該使用 `vendor:publish` Artisan 命令發布 Scout 設定檔。此命令會將 `scout.php` 設定檔發布到您應用程式的 `config` 目錄：

```shell
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

最後，將 `Laravel\Scout\Searchable` trait 加入到您想使其可搜尋的 Model 中。此 trait 會註冊一個 Model 觀察者，它將自動使 Model 與您的搜尋驅動程式保持同步：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Scout\Searchable;

    class Post extends Model
    {
        use Searchable;
    }

<a name="queueing"></a>
### 佇列

儘管使用 Scout 並非強制要求設定佇列，但您仍應強烈建議您在使用此函式庫之前設定一個 [佇列驅動程式](/docs/{{version}}/queues)。執行佇列工作者 (queue worker) 將允許 Scout 將所有同步 Model 資訊至搜尋索引的操作加入佇列，為您的應用程式網頁介面提供更好的回應時間。

設定好佇列驅動程式後，將您 `config/scout.php` 設定檔中的 `queue` 選項值設定為 `true`：

    'queue' => true,

即使 `queue` 選項設定為 `false`，重要的是要記住，某些 Scout 驅動程式（如 Algolia 和 Meilisearch）總是會非同步地索引紀錄。這表示，即使索引操作已在您的 Laravel 應用程式中完成，搜尋引擎本身可能不會立即反映新的和更新的紀錄。

若要指定 Scout 工作所使用的連線與佇列，您可以將 `queue` 設定選項定義為一個陣列：

    'queue' => [
        'connection' => 'redis',
        'queue' => 'scout'
    ],

當然，如果您自訂了 Scout 工作所使用的連線與佇列，您應該執行一個佇列工作者來處理該連線與佇列上的工作：

    php artisan queue:work redis --queue=scout

<a name="driver-prerequisites"></a>
## 驅動程式先決條件

<a name="algolia"></a>
### Algolia

使用 Algolia 驅動程式時，您應該在 `config/scout.php` 設定檔中設定您的 Algolia `id` 和 `secret` 憑證。設定憑證後，您還需要透過 Composer 套件管理工具安裝 Algolia PHP SDK：

```shell
composer require algolia/algoliasearch-client-php
```

<a name="meilisearch"></a>
### Meilisearch

[Meilisearch](https://www.meilisearch.com) 是一個速度極快且開源的搜尋引擎。如果您不確定如何在您的本機電腦上安裝 Meilisearch，您可以使用 [Laravel Sail](/docs/{{version}}/sail#meilisearch)，這是 Laravel 官方支援的 Docker 開發環境。

使用 Meilisearch 驅動程式時，您需要透過 Composer 套件管理工具安裝 Meilisearch PHP SDK：

```shell
composer require meilisearch/meilisearch-php http-interop/http-factory-guzzle
```

接著，在應用程式的 `.env` 檔案中設定 `SCOUT_DRIVER` 環境變數，以及您的 Meilisearch `host` 和 `key` 憑證：

```ini
SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=masterKey
```

有關 Meilisearch 的更多資訊，請查閱 [Meilisearch 文件](https://docs.meilisearch.com/learn/getting_started/quick_start.html)。

此外，您應該透過查閱 [Meilisearch 關於二進位相容性的文件](https://github.com/meilisearch/meilisearch-php#-compatibility-with-meilisearch)，確保您安裝的 `meilisearch/meilisearch-php` 版本與您的 Meilisearch 二進位版本相容。

> [!WARNING]
> 當在應用程式中升級使用 Meilisearch 的 Scout 時，您應該始終 [查看 Meilisearch 服務本身是否有任何額外的破壞性變更](https://github.com/meilisearch/Meilisearch/releases)。

<a name="typesense"></a>
### Typesense

[Typesense](https://typesense.org) 是一個極速的開源搜尋引擎，支援關鍵字搜尋、語義搜尋、地理搜尋和向量搜尋。

您可以 [自行託管](https://typesense.org/docs/guide/install-typesense.html#option-2-local-machine-self-hosting) Typesense 或使用 [Typesense Cloud](https://cloud.typesense.org)。

要開始使用 Typesense 與 Scout，請透過 Composer 套件管理工具安裝 Typesense PHP SDK：

```shell
composer require typesense/typesense-php
```

然後，在應用程式的 .env 檔案中設定 `SCOUT_DRIVER` 環境變數，以及您的 Typesense host 和 API key 憑證：

```ini
SCOUT_DRIVER=typesense
TYPESENSE_API_KEY=masterKey
TYPESENSE_HOST=localhost
```

如果您正在使用 [Laravel Sail](/docs/{{version}}/sail)，您可能需要調整 `TYPESENSE_HOST` 環境變數以符合 Docker 容器名稱。您也可以選擇性地指定安裝的連接埠、路徑和協定：

```ini
TYPESENSE_PORT=8108
TYPESENSE_PATH=
TYPESENSE_PROTOCOL=http
```

Typesense 集合的其他設定和結構定義可以在應用程式的 `config/scout.php` 設定檔中找到。有關 Typesense 的更多資訊，請查閱 [Typesense 文件](https://typesense.org/docs/guide/#quick-start)。

<a name="preparing-data-for-storage-in-typesense"></a>
#### 準備資料以供儲存在 Typesense

使用 Typesense 時，您的可搜尋 Model 必須定義一個 `toSearchableArray` 方法，將 Model 的主鍵轉換為字串，並將建立日期轉換為 UNIX timestamp：

```php
/**
 * Get the indexable data array for the model.
 *
 * @return array<string, mixed>
 */
public function toSearchableArray()
{
    return array_merge($this->toArray(),[
        'id' => (string) $this->id,
        'created_at' => $this->created_at->timestamp,
    ]);
}
```

您還應該在應用程式的 `config/scout.php` 檔案中定義 Typesense 集合結構。集合結構描述了每個可透過 Typesense 搜尋的欄位的資料型別。有關所有可用結構選項的更多資訊，請查閱 [Typesense 文件](https://typesense.org/docs/latest/api/collections.html#schema-parameters)。

如果您需要更改 Typesense 集合的結構，在定義後，您可以執行 `scout:flush` 和 `scout:import`，這將刪除所有現有的索引資料並重新創建結構。或者，您可以使用 Typesense 的 API 來修改集合的結構，而無需移除任何索引資料。

如果您的可搜尋 Model 可以軟刪除，您應該在應用程式的 `config/scout.php` 設定檔中，於 Model 對應的 Typesense 結構中定義 `__soft_deleted` 欄位：

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

Typesense 允許您在透過 `options` 方法執行搜尋操作時，動態修改您的 [搜尋參數](https://typesense.org/docs/latest/api/search.html#search-parameters)：

```php
use App\Models\Todo;

Todo::search('Groceries')->options([
    'query_by' => 'title, description'
])->get();
```

<a name="configuration"></a>
## 設定

<a name="configuring-model-indexes"></a>
### 設定 Model 索引

每個 Eloquent Model 都會與一個特定的搜尋「索引」同步，此索引包含該 Model 所有可搜尋的紀錄。換句話說，您可以將每個索引視為一個 MySQL 資料表。預設情況下，每個 Model 都會被儲存到與該 Model 典型「資料表」名稱相符的索引中。通常，這會是 Model 名稱的複數形式；然而，您可以在 Model 上覆寫 `searchableAs` 方法，以自訂 Model 的索引名稱：

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

<a name="configuring-searchable-data"></a>
### 設定可搜尋資料

預設情況下，給定 Model 的完整 `toArray` 形式將被儲存到其搜尋索引中。如果您想自訂同步到搜尋索引的資料，您可以覆寫 Model 上的 `toSearchableArray` 方法：

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

某些搜尋引擎（例如 Meilisearch）只會對正確型別的資料執行篩選操作 (`>`、`<` 等)。因此，當您使用這些搜尋引擎並自訂可搜尋資料時，應確保數值被轉換為正確的型別：

    public function toSearchableArray()
    {
        return [
            'id' => (int) $this->id,
            'name' => $this->name,
            'price' => (float) $this->price,
        ];
    }

<a name="configuring-indexes-for-algolia"></a>
#### 設定索引設定 (Algolia)

有時您可能想在 Algolia 索引上設定額外的參數。雖然您可以透過 Algolia UI 管理這些設定，但有時直接從應用程式的 `config/scout.php` 設定檔中管理索引設定的預期狀態會更有效率。

這種方法可讓您透過應用程式的自動化部署流程部署這些設定，避免手動設定並確保多個環境之間的一致性。您可以設定可篩選屬性、排序、分面，或 [任何其他支援的設定](https://www.algolia.com/doc/rest-api/search/#tag/Indices/operation/setSettings)。

首先，在應用程式的 `config/scout.php` 設定檔中為每個索引新增設定：

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

如果給定索引所依據的 Model 是軟刪除的，並且包含在 `index-settings` 陣列中，Scout 將自動為該索引上的軟刪除 Model 包含分面支援。如果對於軟刪除 Model 索引沒有其他要定義的分面屬性，您可以直接為該 Model 在 `index-settings` 陣列中新增一個空項目：

```php
'index-settings' => [
    Flight::class => []
],
```

設定應用程式的索引設定後，您必須執行 `scout:sync-index-settings` Artisan 命令。此命令會將您目前設定的索引設定告知 Algolia。為了方便起見，您可能希望將此命令納入您的部署流程中：

```shell
php artisan scout:sync-index-settings
```

<a name="configuring-filterable-data-for-meilisearch"></a>
#### 設定可篩選資料與索引設定 (Meilisearch)

與 Scout 的其他驅動程式不同，Meilisearch 要求您預先定義索引搜尋設定，例如可篩選屬性、可排序屬性以及 [其他支援的設定欄位](https://docs.meilisearch.com/reference/api/settings.html)。

可篩選屬性是您打算在使用 Scout 的 `where` 方法時進行篩選的任何屬性，而可排序屬性是您打算在使用 Scout 的 `orderBy` 方法時進行排序的任何屬性。若要定義您的索引設定，請調整應用程式 `scout` 設定檔中 `meilisearch` 設定項目中的 `index-settings` 部分：

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

如果給定索引所依據的 Model 是軟刪除的，並且包含在 `index-settings` 陣列中，Scout 將自動為該索引上的軟刪除 Model 包含篩選支援。如果對於軟刪除 Model 索引沒有其他要定義的可篩選或可排序屬性，您可以直接為該 Model 在 `index-settings` 陣列中新增一個空項目：

```php
'index-settings' => [
    Flight::class => []
],
```

設定應用程式的索引設定後，您必須執行 `scout:sync-index-settings` Artisan 命令。此命令會將您目前設定的索引設定告知 Meilisearch。為了方便起見，您可能希望將此命令納入您的部署流程中：

```shell
php artisan scout:sync-index-settings
```

<a name="configuring-the-model-id"></a>
### 設定 Model ID

預設情況下，Scout 會將 Model 的主鍵用作儲存在搜尋索引中的 Model 唯一 ID / 鍵。如果您需要自訂此行為，可以覆寫 Model 上的 `getScoutKey` 和 `getScoutKeyName` 方法：

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

<a name="configuring-search-engines-per-model"></a>
### 設定各 Model 的搜尋引擎

當進行搜尋時，Scout 通常會使用應用程式 `scout` 設定檔中指定的預設搜尋引擎。然而，特定 Model 的搜尋引擎可以透過覆寫該 Model 上的 `searchableUsing` 方法來變更：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Scout\Engines\Engine;
    use Laravel\Scout\EngineManager;
    use Laravel\Scout\Searchable;

    class User extends Model
    {
        use Searchable;

        /**
         * Get the engine used to index the model.
         */
        public function searchableUsing(): Engine
        {
            return app(EngineManager::class)->engine('meilisearch');
        }
    }

<a name="identifying-users"></a>
### 識別使用者

Scout 也允許您在使用 [Algolia](https://algolia.com) 時自動識別使用者。將已驗證的使用者與搜尋操作相關聯，在您查看 Algolia 儀表板中的搜尋分析時可能會很有幫助。您可以透過在應用程式的 `.env` 檔案中將 `SCOUT_IDENTIFY` 環境變數定義為 `true` 來啟用使用者識別功能：

```ini
SCOUT_IDENTIFY=true
```

啟用此功能後，也會將請求的 IP 位址及已驗證使用者的主要識別碼傳遞給 Algolia，以便此資料與使用者發出的任何搜尋請求相關聯。

<a name="database-and-collection-engines"></a>
## 資料庫 / Collection 引擎

<a name="database-engine"></a>
### 資料庫引擎

> [!WARNING]
> 資料庫引擎目前支援 MySQL 和 PostgreSQL。

如果您的應用程式與中小型資料庫互動，或者工作負載較輕，您可能會發現從 Scout 的「database」引擎開始使用會更方便。該資料庫引擎在從現有資料庫過濾結果時，將使用「where like」子句和全文索引來確定適用於您查詢的搜尋結果。

要使用資料庫引擎，您可以直接將 `SCOUT_DRIVER` 環境變數的值設定為 `database`，或在應用程式的 `scout` 設定檔中直接指定 `database` 驅動程式：

```ini
SCOUT_DRIVER=database
```

一旦您將資料庫引擎指定為首選驅動程式，您必須[設定您的可搜尋資料](#configuring-searchable-data)。然後，您可以開始對您的 Model 執行[搜尋查詢](#searching)。使用資料庫引擎時，不需要進行搜尋引擎索引，例如用於填充 Algolia、Meilisearch 或 Typesense 索引的索引。

#### 自訂資料庫搜尋策略

預設情況下，資料庫引擎將對您已[設定為可搜尋](#configuring-searchable-data)的每個 Model 屬性執行「where like」查詢。然而，在某些情況下，這可能會導致效能不佳。因此，可以設定資料庫引擎的搜尋策略，使某些指定欄位使用全文搜尋查詢，或者僅使用「where like」約束來搜尋字串的前綴 (`example%`)，而不是在整個字串中搜尋 (`%example%`)。

若要定義此行為，您可以為 Model 的 `toSearchableArray` 方法指定 PHP 屬性。任何未指定額外搜尋策略行為的欄位將繼續使用預設的「where like」策略：

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
> 在指定某個欄位應使用全文查詢約束之前，請確保該欄位已指定了[全文索引](/docs/{{version}}/migrations#available-index-types)。

<a name="collection-engine"></a>
### Collection 引擎

雖然您可以在本地開發期間自由使用 Algolia、Meilisearch 或 Typesense 搜尋引擎，但您可能會發現從「collection」引擎開始使用更方便。Collection 引擎將使用「where」子句和對現有資料庫結果進行 collection 過濾來確定適用於您查詢的搜尋結果。使用此引擎時，不需要「索引」您的可搜尋 Model，因為它們將直接從您的本地資料庫中檢索。

要使用 collection 引擎，您可以直接將 `SCOUT_DRIVER` 環境變數的值設定為 `collection`，或在應用程式的 `scout` 設定檔中直接指定 `collection` 驅動程式：

```ini
SCOUT_DRIVER=collection
```

一旦您將 collection 驅動程式指定為首選驅動程式，您可以開始對您的 Model 執行[搜尋查詢](#searching)。使用 collection 引擎時，不需要進行搜尋引擎索引，例如用於填充 Algolia、Meilisearch 或 Typesense 索引的索引。

#### 與資料庫引擎的差異

乍看之下，「database」和「collections」引擎相當相似。它們都直接與您的資料庫互動以檢索搜尋結果。然而，collection 引擎不使用全文索引或 `LIKE` 子句來查找匹配的紀錄。相反地，它會提取所有可能的紀錄，並使用 Laravel 的 `Str::is` 輔助函式來判斷搜尋字串是否存在於 Model 屬性值中。

collection 引擎是最具可移植性的搜尋引擎，因為它適用於 Laravel 支援的所有關聯式資料庫（包括 SQLite 和 SQL Server）；但是，它的效率比 Scout 的資料庫引擎低。

<a name="indexing"></a>
## 索引

<a name="batch-import"></a>
### 批次匯入

若您正在現有專案中安裝 Scout，您可能已經有一些需要匯入到索引中的資料庫紀錄。Scout 提供了 `scout:import` Artisan 命令，可用來將所有現有紀錄匯入到您的搜尋索引中：

```shell
php artisan scout:import "App\Models\Post"
```

`flush` 命令可用來從您的搜尋索引中移除 Model 的所有紀錄：

```shell
php artisan scout:flush "App\Models\Post"
```

<a name="modifying-the-import-query"></a>
#### 修改匯入查詢

如果您想修改用於批次匯入所有 Model 的查詢，您可以在您的 Model 上定義 `makeAllSearchableUsing` 方法。這是新增任何在匯入 Model 前可能需要進行的預先載入 (eager relationship loading) 關聯的好地方：

    use Illuminate\Database\Eloquent\Builder;

    /**
     * Modify the query used to retrieve models when making all of the models searchable.
     */
    protected function makeAllSearchableUsing(Builder $query): Builder
    {
        return $query->with('author');
    }

> [!WARNING]
> 當使用佇列進行批次匯入 Model 時，`makeAllSearchableUsing` 方法可能不適用。當 Model 集合由 Job 處理時，關聯 [不會被還原](/docs/{{version}}/queues#handling-relationships)。

<a name="adding-records"></a>
### 新增紀錄

一旦您將 `Laravel\Scout\Searchable` trait 加入到 Model 中，您只需 `save` 或 `create` 一個 Model 實例，它就會自動被新增到您的搜尋索引。如果您已將 Scout 設定為 [使用佇列](#queueing)，此操作將由您的佇列 Worker 在背景執行：

    use App\Models\Order;

    $order = new Order;

    // ...

    $order->save();

<a name="adding-records-via-query"></a>
#### 透過查詢新增紀錄

如果您想透過 Eloquent 查詢將 Model 集合新增至您的搜尋索引，您可以將 `searchable` 方法鏈接到 Eloquent 查詢上。`searchable` 方法將會 [分塊查詢結果](/docs/{{version}}/eloquent#chunking-results)，並將紀錄新增到您的搜尋索引。同樣地，如果您已將 Scout 設定為使用佇列，所有的分塊都將由您的佇列 Worker 在背景匯入：

    use App\Models\Order;

    Order::where('price', '>', 100)->searchable();

您也可以在 Eloquent 關聯實例上呼叫 `searchable` 方法：

    $user->orders()->searchable();

或者，如果您記憶體中已有 Eloquent Model 集合，您可以在集合實例上呼叫 `searchable` 方法，將 Model 實例新增到其對應的索引中：

    $orders->searchable();

> [!NOTE]
> `searchable` 方法可被視為「upsert」操作。換句話說，如果 Model 紀錄已存在於您的索引中，它將被更新。如果它不存在於搜尋索引中，它將被新增到索引。

<a name="updating-records"></a>
### 更新紀錄

要更新一個可搜尋的 Model，您只需更新 Model 實例的屬性並將 Model `save` 到您的資料庫。Scout 將自動將這些變更持久化到您的搜尋索引：

    use App\Models\Order;

    $order = Order::find(1);

    // Update the order...

    $order->save();

您也可以在 Eloquent 查詢實例上呼叫 `searchable` 方法來更新 Model 集合。如果 Model 不存在於您的搜尋索引中，它們將會被建立：

    Order::where('price', '>', 100)->searchable();

如果您想更新關聯中所有 Model 的搜尋索引紀錄，您可以在關聯實例上呼叫 `searchable`：

    $user->orders()->searchable();

或者，如果您記憶體中已有 Eloquent Model 集合，您可以在集合實例上呼叫 `searchable` 方法，以更新其對應索引中的 Model 實例：

    $orders->searchable();

<a name="modifying-records-before-importing"></a>
#### 匯入前修改紀錄

有時候，您可能需要在 Model 集合可搜尋之前準備這些集合。例如，您可能希望預先載入 (eager load) 一個關聯，以便將關聯資料有效率地新增到您的搜尋索引。要做到這一點，請在對應的 Model 上定義 `makeSearchableUsing` 方法：

    use Illuminate\Database\Eloquent\Collection;

    /**
     * Modify the collection of models being made searchable.
     */
    public function makeSearchableUsing(Collection $models): Collection
    {
        return $models->load('author');
    }

<a name="removing-records"></a>
### 移除紀錄

要從您的索引中移除一筆紀錄，您可以簡單地從資料庫中 `delete` Model。即使您正在使用 [軟刪除](/docs/{{version}}/eloquent#soft-deleting) 的 Model，也可以這樣做：

    use App\Models\Order;

    $order = Order::find(1);

    $order->delete();

如果您不想在刪除紀錄前先取出 Model，您可以在 Eloquent 查詢實例上使用 `unsearchable` 方法：

    Order::where('price', '>', 100)->unsearchable();

如果您想移除關聯中所有 Model 的搜尋索引紀錄，您可以在關聯實例上呼叫 `unsearchable`：

    $user->orders()->unsearchable();

或者，如果您記憶體中已有 Eloquent Model 集合，您可以在集合實例上呼叫 `unsearchable` 方法，以從其對應索引中移除 Model 實例：

    $orders->unsearchable();

要從其對應索引中移除所有 Model 紀錄，您可以呼叫 `removeAllFromSearch` 方法：

    Order::removeAllFromSearch();

<a name="pausing-indexing"></a>
### 暫停索引

有時候，您可能需要在 Model 上執行批次 Eloquent 操作，但又不希望將 Model 資料同步到您的搜尋索引。您可以使用 `withoutSyncingToSearch` 方法來做到這一點。此方法接受一個單一的閉包 (closure)，該閉包將會立即執行。在該閉包中發生的任何 Model 操作都不會同步到 Model 的索引：

    use App\Models\Order;

    Order::withoutSyncingToSearch(function () {
        // Perform model actions...
    });

<a name="conditionally-searchable-model-instances"></a>
### 條件式可搜尋 Model 實例

有時候，您可能需要僅在特定條件下才讓 Model 變得可搜尋。例如，想像您有一個 `App\Models\Post` Model，它可能處於兩種狀態之一：「草稿 (draft)」和「已發佈 (published)」。您可能只想讓「已發佈」的貼文可搜尋。為了實現這一點，您可以在您的 Model 上定義 `shouldBeSearchable` 方法：

    /**
     * Determine if the model should be searchable.
     */
    public function shouldBeSearchable(): bool
    {
        return $this->isPublished();
    }

`shouldBeSearchable` 方法僅在透過 `save` 和 `create` 方法、查詢或關聯操作 Model 時應用。直接使用 `searchable` 方法將 Model 或集合設為可搜尋將會覆寫 `shouldBeSearchable` 方法的結果。

> [!WARNING]
> 當使用 Scout 的「資料庫 (database)」引擎時，`shouldBeSearchable` 方法不適用，因為所有可搜尋資料都始終儲存在資料庫中。若要在使用資料庫引擎時實現類似行為，您應該改用 [where 子句](#where-clauses)。

<a name="searching"></a>
## 搜尋

您可以使用 `search` 方法開始搜尋 Model。search 方法接受一個單一字串，此字串將用於搜尋您的 Model。接著您應將 `get` 方法鏈接到搜尋查詢上，以取得與給定搜尋查詢相符的 Eloquent Model。

    use App\Models\Order;

    $orders = Order::search('Star Trek')->get();

由於 Scout 搜尋會回傳一個 Eloquent Model 的集合，您甚至可以直接從路由或控制器中回傳結果，它們將會自動轉換為 JSON：

    use App\Models\Order;
    use Illuminate\Http\Request;

    Route::get('/search', function (Request $request) {
        return Order::search($request->search)->get();
    });

如果您想在將原始搜尋結果轉換為 Eloquent Model 之前取得它們，可以使用 `raw` 方法：

    $orders = Order::search('Star Trek')->raw();


<a name="custom-indexes"></a>
#### 自訂索引

搜尋查詢通常會在使用 Model 的 [`searchableAs`](#configuring-model-indexes) 方法指定的索引上執行。然而，您可以使用 `within` 方法來指定應改為搜尋的自訂索引：

    $orders = Order::search('Star Trek')
        ->within('tv_shows_popularity_desc')
        ->get();


<a name="where-clauses"></a>
### Where 子句

Scout 允許您在搜尋查詢中加入簡單的「where」子句。目前，這些子句僅支援基本的數值相等性檢查，主要用於依據擁有者 ID 來限制搜尋查詢範圍：

    use App\Models\Order;

    $orders = Order::search('Star Trek')->where('user_id', 1)->get();

此外，`whereIn` 方法可用於驗證給定欄位的值是否包含在給定陣列中：

    $orders = Order::search('Star Trek')->whereIn(
        'status', ['open', 'paid']
    )->get();

`whereNotIn` 方法則用於驗證給定欄位的值不包含在給定陣列中：

    $orders = Order::search('Star Trek')->whereNotIn(
        'status', ['closed']
    )->get();

由於搜尋索引並非關聯式資料庫，目前不支援更進階的「where」子句。

> [!WARNING]
> 如果您的應用程式正在使用 Meilisearch，在使用 Scout 的「where」子句之前，您必須配置應用程式的 [可篩選屬性](#configuring-filterable-data-for-meilisearch)。


<a name="pagination"></a>
### 分頁

除了取得 Model 集合之外，您還可以使用 `paginate` 方法來對搜尋結果進行分頁。此方法將回傳一個 `Illuminate\Pagination\LengthAwarePaginator` 實例，就如同您 [對傳統 Eloquent 查詢進行分頁](/docs/{{version}}/pagination) 一樣：

    use App\Models\Order;

    $orders = Order::search('Star Trek')->paginate();

您可以將每頁要取得的 Model 數量作為第一個參數傳遞給 `paginate` 方法來指定：

    $orders = Order::search('Star Trek')->paginate(15);

取得結果後，您可以顯示這些結果並使用 [Blade](/docs/{{version}}/blade) 渲染頁面連結，就如同您對傳統 Eloquent 查詢進行分頁一樣：

```html
<div class="container">
    @foreach ($orders as $order)
        {{ $order->price }}
    @endforeach
</div>

{{ $orders->links() }}
```

當然，如果您想以 JSON 格式取得分頁結果，可以直接從路由或控制器中回傳分頁器實例：

    use App\Models\Order;
    use Illuminate\Http\Request;

    Route::get('/orders', function (Request $request) {
        return Order::search($request->input('query'))->paginate(15);
    });

> [!WARNING]
> 由於搜尋引擎不了解您 Eloquent Model 的全域 Scope 定義，因此在使用 Scout 分頁的應用程式中，不應利用全域 Scope。或者，您應該在使用 Scout 搜尋時重新建立全域 Scope 的約束。


<a name="soft-deleting"></a>
### 軟刪除

如果您的索引 Model 是 [軟刪除](/docs/{{version}}/eloquent#soft-deleting) 的，並且您需要搜尋這些軟刪除的 Model，請將 `config/scout.php` 設定檔中的 `soft_delete` 選項設為 `true`：

    'soft_delete' => true,

當此設定選項為 `true` 時，Scout 將不會從搜尋索引中移除軟刪除的 Model。相反地，它會在索引的紀錄上設定一個隱藏的 `__soft_deleted` 屬性。接著，您可以在搜尋時使用 `withTrashed` 或 `onlyTrashed` 方法來取得這些軟刪除的紀錄：

    use App\Models\Order;

    // Include trashed records when retrieving results...
    $orders = Order::search('Star Trek')->withTrashed()->get();

    // Only include trashed records when retrieving results...
    $orders = Order::search('Star Trek')->onlyTrashed()->get();

> [!NOTE]
> 當使用 `forceDelete` 永久刪除軟刪除的 Model 時，Scout 會自動將其從搜尋索引中移除。


<a name="customizing-engine-searches"></a>
### 自訂引擎搜尋

如果您需要對引擎的搜尋行為進行進階自訂，可以將閉包 (closure) 作為 `search` 方法的第二個參數傳入。例如，您可以使用此回呼函數在搜尋查詢傳遞給 Algolia 之前，向您的搜尋選項添加地理位置資料：

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


<a name="customizing-the-eloquent-results-query"></a>
#### 自訂 Eloquent 結果查詢

在 Scout 從應用程式的搜尋引擎中取得符合的 Eloquent Model 列表後，將使用 Eloquent 根據其主鍵來取得所有符合的 Model。您可以透過呼叫 `query` 方法來自訂此查詢。`query` 方法接受一個閉包 (closure)，該閉包將接收 Eloquent 查詢建構器實例作為參數：

```php
use App\Models\Order;
use Illuminate\Database\Eloquent\Builder;

$orders = Order::search('Star Trek')
    ->query(fn (Builder $query) => $query->with('invoices'))
    ->get();
```

由於此回呼函數是在相關 Model 已從您的應用程式搜尋引擎中取得後才被調用，因此 `query` 方法不應用於「篩選」結果。相反地，您應該使用 [Scout where 子句](#where-clauses)。

<a name="custom-engines"></a>
## 自訂引擎

<a name="writing-the-engine"></a>
#### 編寫引擎

如果內建的 Scout 搜尋引擎不符合您的需求，您可以編寫自己的自訂引擎並向 Scout 註冊。您的引擎應擴展 `Laravel\Scout\Engines\Engine` 抽象類別。此抽象類別包含您的自訂引擎必須實作的八個方法：

    use Laravel\Scout\Builder;

    abstract public function update($models);
    abstract public function delete($models);
    abstract public function search(Builder $builder);
    abstract public function paginate(Builder $builder, $perPage, $page);
    abstract public function mapIds($results);
    abstract public function map(Builder $builder, $results, $model);
    abstract public function getTotalCount($results);
    abstract public function flush($model);

您可能會發現查看 `Laravel\Scout\Engines\AlgoliaEngine` 類別中這些方法的實作會有所幫助。此類別將為您提供一個良好的起點，以學習如何在您自己的引擎中實作這些方法。

<a name="registering-the-engine"></a>
#### 註冊引擎

編寫完您的自訂引擎後，您可以使用 Scout 引擎管理器中的 `extend` 方法向 Scout 註冊它。Scout 的引擎管理器可以從 Laravel 服務容器中解析。您應該在 `App\Providers\AppServiceProvider` 類別或其他應用程式使用的任何服務提供者的 `boot` 方法中呼叫 `extend` 方法：

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

您的引擎註冊後，您可以在應用程式的 `config/scout.php` 設定檔中將其指定為預設的 Scout `driver`：

    'driver' => 'mysql',