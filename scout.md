# Laravel Scout

- [簡介](#introduction)
- [安裝](#installation)
    - [佇列](#queueing)
- [驅動程式先決條件](#driver-prerequisites)
    - [Algolia](#algolia)
    - [Meilisearch](#meilisearch)
    - [Typesense](#typesense)
- [設定](#configuration)
    - [設定模型索引](#configuring-model-indexes)
    - [設定可搜尋資料](#configuring-searchable-data)
    - [設定模型 ID](#configuring-the-model-id)
    - [依模型設定搜尋引擎](#configuring-search-engines-per-model)
    - [識別使用者](#identifying-users)
- [資料庫 / Collection 引擎](#database-and-collection-engines)
    - [資料庫引擎](#database-engine)
    - [Collection 引擎](#collection-engine)
- [索引](#indexing)
    - [批次匯入](#batch-import)
    - [新增記錄](#adding-records)
    - [更新記錄](#updating-records)
    - [移除記錄](#removing-records)
    - [暫停索引](#pausing-indexing)
    - [條件式可搜尋模型實例](#conditionally-searchable-model-instances)
- [搜尋](#searching)
    - [Where 子句](#where-clauses)
    - [分頁](#pagination)
    - [軟刪除](#soft-deleting)
    - [自訂引擎搜尋](#customizing-engine-searches)
- [自訂引擎](#custom-engines)

<a name="introduction"></a>
## 簡介

[Laravel Scout](https://github.com/laravel/scout) 提供了一個簡單、基於驅動程式的解決方案，可為您的 [Eloquent 模型](/docs/{{version}}/eloquent)新增全文搜尋功能。透過模型觀察器，Scout 會自動讓您的搜尋索引與 Eloquent 記錄保持同步。

目前，Scout 隨附 [Algolia](https://www.algolia.com/)、[Meilisearch](https://www.meilisearch.com)、[Typesense](https://typesense.org) 以及 MySQL / PostgreSQL (`database`) 驅動程式。此外，Scout 還包含一個「collection」驅動程式，專為本地開發使用而設計，不需要任何外部依賴或第三方服務。此外，編寫自訂驅動程式很簡單，您可以自由地使用自己的搜尋實作來擴展 Scout。

<a name="installation"></a>
## 安裝

首先，透過 Composer 套件管理器安裝 Scout：

```shell
composer require laravel/scout
```

安裝 Scout 後，您應該使用 `vendor:publish` Artisan 命令發布 Scout 設定檔。此命令會將 `scout.php` 設定檔發布到您應用程式的 `config` 目錄：

```shell
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

最後，將 `Laravel\Scout\Searchable` Trait 新增到您想要使其可搜尋的模型。此 Trait 會註冊一個模型觀察器，該觀察器會自動讓模型與您的搜尋驅動程式保持同步：

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
### 佇列

雖然使用 Scout 並非嚴格要求，但您在使用此函式庫之前，應強烈考慮設定一個 [佇列驅動程式](/docs/{{version}}/queues)。執行佇列工作者將允許 Scout 將所有同步模型資訊到搜尋索引的操作排入佇列，從而為您應用程式的網頁介面提供更好的回應時間。

設定佇列驅動程式後，將 `config/scout.php` 設定檔中的 `queue` 選項值設定為 `true`：

```php
'queue' => true,
```

即使 `queue` 選項設定為 `false`，請務必記住，某些 Scout 驅動程式 (例如 Algolia 和 Meilisearch) 始終會非同步地索引記錄。這表示，即使索引操作已在您的 Laravel 應用程式中完成，搜尋引擎本身可能不會立即反映新的和更新的記錄。

若要指定您的 Scout 工作使用的連線和佇列，您可以將 `queue` 設定選項定義為陣列：

```php
'queue' => [
    'connection' => 'redis',
    'queue' => 'scout'
],
```

當然，如果您自訂了 Scout 工作使用的連線和佇列，您應該執行一個佇列工作者來處理該連線和佇列上的工作：

```shell
php artisan queue:work redis --queue=scout
```

<a name="driver-prerequisites"></a>
## 驅動程式先決條件

<a name="algolia"></a>
### Algolia

使用 Algolia 驅動程式時，您應該在 `config/scout.php` 設定檔中設定您的 Algolia `id` 和 `secret` 憑證。設定憑證後，您還需要透過 Composer 套件管理器安裝 Algolia PHP SDK：

```shell
composer require algolia/algoliasearch-client-php
```

<a name="meilisearch"></a>
### Meilisearch

[Meilisearch](https://www.meilisearch.com) 是一個速度極快且開源的搜尋引擎。如果您不確定如何在本地機器上安裝 Meilisearch，您可以使用 [Laravel Sail](/docs/{{version}}/sail#meilisearch)，這是 Laravel 官方支援的 Docker 開發環境。

使用 Meilisearch 驅動程式時，您需要透過 Composer 套件管理器安裝 Meilisearch PHP SDK：

```shell
composer require meilisearch/meilisearch-php http-interop/http-factory-guzzle
```

然後，在您應用程式的 `.env` 檔案中設定 `SCOUT_DRIVER` 環境變數以及您的 Meilisearch `host` 和 `key` 憑證：

```ini
SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=masterKey
```

有關 Meilisearch 的更多資訊，請參閱 [Meilisearch 文件](https://docs.meilisearch.com/learn/getting_started/quick_start.html)。

此外，您應該透過查閱 [Meilisearch 關於二進位相容性的文件](https://github.com/meilisearch/meilisearch-php#-compatibility-with-meilisearch)，確保您安裝的 `meilisearch/meilisearch-php` 版本與您的 Meilisearch 二進位版本相容。

> [!WARNING]
> 在使用 Meilisearch 的應用程式上升級 Scout 時，您應該始終 [檢閱 Meilisearch 服務本身的任何額外重大變更](https://github.com/meilisearch/Meilisearch/releases)。

<a name="typesense"></a>
### Typesense

[Typesense](https://typesense.org) 是一個閃電般快速的開源搜尋引擎，支援關鍵字搜尋、語義搜尋、地理搜尋和向量搜尋。

您可以 [自行託管](https://typesense.org/docs/guide/install-typesense.html#option-2-local-machine-self-hosting) Typesense 或使用 [Typesense Cloud](https://cloud.typesense.org)。

若要開始將 Typesense 與 Scout 搭配使用，請透過 Composer 套件管理器安裝 Typesense PHP SDK：

```shell
composer require typesense/typesense-php
```

然後，在您應用程式的 `.env` 檔案中設定 `SCOUT_DRIVER` 環境變數以及您的 Typesense 主機和 API 金鑰憑證：

```ini
SCOUT_DRIVER=typesense
TYPESENSE_API_KEY=masterKey
TYPESENSE_HOST=localhost
```

如果您使用 [Laravel Sail](/docs/{{version}}/sail)，您可能需要調整 `TYPESENSE_HOST` 環境變數以符合 Docker 容器名稱。您也可以選擇性地指定安裝的連接埠、路徑和協定：

```ini
TYPESENSE_PORT=8108
TYPESENSE_PATH=
TYPESENSE_PROTOCOL=http
```

您的 Typesense Collection 的其他設定和 Schema 定義可以在您應用程式的 `config/scout.php` 設定檔中找到。有關 Typesense 的更多資訊，請參閱 [Typesense 文件](https://typesense.org/docs/guide/#quick-start)。

<a name="preparing-data-for-storage-in-typesense"></a>
#### 準備資料以儲存在 Typesense 中

使用 Typesense 時，您的可搜尋模型必須定義一個 `toSearchableArray` 方法，該方法將模型的主鍵轉換為字串，並將建立日期轉換為 UNIX 時間戳記：

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

您還應該在您應用程式的 `config/scout.php` 檔案中定義您的 Typesense Collection Schema。Collection Schema 描述了每個可透過 Typesense 搜尋的欄位的資料類型。有關所有可用 Schema 選項的更多資訊，請參閱 [Typesense 文件](https://typesense.org/docs/latest/api/collections.html#schema-parameters)。

如果您需要在定義 Typesense Collection 的 Schema 後進行更改，您可以執行 `scout:flush` 和 `scout:import`，這將刪除所有現有的索引資料並重新建立 Schema。或者，您可以使用 Typesense 的 API 來修改 Collection 的 Schema，而無需移除任何索引資料。

如果您的可搜尋模型是軟刪除的，您應該在您應用程式的 `config/scout.php` 設定檔中，在模型對應的 Typesense Schema 中定義一個 `__soft_deleted` 欄位：

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
### 設定模型索引

每個 Eloquent 模型都與一個給定的搜尋「索引」同步，該索引包含該模型的所有可搜尋記錄。換句話說，您可以將每個索引視為一個 MySQL 資料表。預設情況下，每個模型都會持久化到與模型典型「資料表」名稱相符的索引中。通常，這是模型名稱的複數形式；但是，您可以透過覆寫模型上的 `searchableAs` 方法來自訂模型的索引：

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

<a name="configuring-searchable-data"></a>
### 設定可搜尋資料

預設情況下，給定模型的整個 `toArray` 形式將持久化到其搜尋索引中。如果您想自訂同步到搜尋索引的資料，您可以覆寫模型上的 `toSearchableArray` 方法：

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

某些搜尋引擎 (例如 Meilisearch) 只會對正確類型的資料執行篩選操作 (`>`, `<`, 等)。因此，當使用這些搜尋引擎並自訂您的可搜尋資料時，您應該確保數值被轉換為其正確的類型：

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

<a name="configuring-indexes-for-algolia"></a>
#### 設定索引設定 (Algolia)

有時您可能希望在 Algolia 索引上設定其他設定。雖然您可以透過 Algolia UI 管理這些設定，但有時直接從您應用程式的 `config/scout.php` 設定檔管理索引設定的所需狀態會更有效率。

這種方法允許您透過應用程式的自動化部署流程部署這些設定，避免手動設定並確保多個環境之間的一致性。您可以設定可篩選屬性、排名、分面或 [任何其他支援的設定](https://www.algolia.com/doc/rest-api/search/#tag/Indices/operation/setSettings)。

首先，在您應用程式的 `config/scout.php` 設定檔中為每個索引新增設定：

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

如果給定索引所依據的模型是軟刪除的，並且包含在 `index-settings` 陣列中，Scout 將自動包含對該索引上軟刪除模型進行分面的支援。如果您沒有其他分面屬性要為軟刪除模型索引定義，您可以簡單地為該模型在 `index-settings` 陣列中新增一個空項目：

```php
'index-settings' => [
    Flight::class => []
],
```

設定應用程式的索引設定後，您必須呼叫 `scout:sync-index-settings` Artisan 命令。此命令會通知 Algolia 您目前設定的索引設定。為了方便起見，您可能希望將此命令作為部署流程的一部分：

```shell
php artisan scout:sync-index-settings
```

<a name="configuring-filterable-data-for-meilisearch"></a>
#### 設定可篩選資料和索引設定 (Meilisearch)

與 Scout 的其他驅動程式不同，Meilisearch 要求您預先定義索引搜尋設定，例如可篩選屬性、可排序屬性以及 [其他支援的設定欄位](https://docs.meilisearch.com/reference/api/settings.html)。

可篩選屬性是您在呼叫 Scout 的 `where` 方法時計畫篩選的任何屬性，而可排序屬性是您在呼叫 Scout 的 `orderBy` 方法時計畫排序的任何屬性。若要定義您的索引設定，請調整您應用程式的 `scout` 設定檔中 `meilisearch` 設定項目中的 `index-settings` 部分：

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

如果給定索引所依據的模型是軟刪除的，並且包含在 `index-settings` 陣列中，Scout 將自動包含對該索引上軟刪除模型進行篩選的支援。如果您沒有其他可篩選或可排序屬性要為軟刪除模型索引定義，您可以簡單地為該模型在 `index-settings` 陣列中新增一個空項目：

```php
'index-settings' => [
    Flight::class => []
],
```

設定應用程式的索引設定後，您必須呼叫 `scout:sync-index-settings` Artisan 命令。此命令會通知 Meilisearch 您目前設定的索引設定。為了方便起見，您可能希望將此命令作為部署流程的一部分：

```shell
php artisan scout:sync-index-settings
```

<a name="configuring-the-model-id"></a>
### 設定模型 ID

預設情況下，Scout 會使用模型的主鍵作為儲存在搜尋索引中的模型唯一 ID / 鍵。如果您需要自訂此行為，您可以覆寫模型上的 `getScoutKey` 和 `getScoutKeyName` 方法：

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

<a name="configuring-search-engines-per-model"></a>
### 依模型設定搜尋引擎

搜尋時，Scout 通常會使用您應用程式的 `scout` 設定檔中指定的預設搜尋引擎。但是，可以透過覆寫模型上的 `searchableUsing` 方法來更改特定模型的搜尋引擎：

```php
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
```

<a name="identifying-users"></a>
### 識別使用者

Scout 還允許您在使用 [Algolia](https://algolia.com) 時自動識別使用者。將已驗證的使用者與搜尋操作關聯起來，在 Algolia 的儀表板中查看搜尋分析時可能會很有幫助。您可以透過在應用程式的 `.env` 檔案中將 `SCOUT_IDENTIFY` 環境變數定義為 `true` 來啟用使用者識別：

```ini
SCOUT_IDENTIFY=true
```

啟用此功能還會將請求的 IP 位址和您已驗證使用者的主要識別碼傳遞給 Algolia，以便此資料與使用者發出的任何搜尋請求相關聯。

<a name="database-and-collection-engines"></a>
## 資料庫 / Collection 引擎

<a name="database-engine"></a>
### 資料庫引擎

> [!WARNING]
> 資料庫引擎目前支援 MySQL 和 PostgreSQL。

如果您的應用程式與中小型資料庫互動或工作負載較輕，您可能會發現使用 Scout 的「database」引擎更方便。資料庫引擎將使用「where like」子句和全文索引來篩選現有資料庫中的結果，以確定查詢的適用搜尋結果。

若要使用資料庫引擎，您可以簡單地將 `SCOUT_DRIVER` 環境變數的值設定為 `database`，或直接在您應用程式的 `scout` 設定檔中指定 `database` 驅動程式：

```ini
SCOUT_DRIVER=database
```

一旦您將資料庫引擎指定為您的首選驅動程式，您必須 [設定您的可搜尋資料](#configuring-searchable-data)。然後，您可以開始對您的模型 [執行搜尋查詢](#searching)。使用資料庫引擎時，不需要搜尋引擎索引，例如為 Algolia、Meilisearch 或 Typesense 索引建立種子所需的索引。

#### 自訂資料庫搜尋策略

預設情況下，資料庫引擎將對您已 [設定為可搜尋](#configuring-searchable-data) 的每個模型屬性執行「where like」查詢。但是，在某些情況下，這可能會導致效能不佳。因此，可以設定資料庫引擎的搜尋策略，以便某些指定的欄位利用全文搜尋查詢，或者只使用「where like」約束來搜尋字串的前綴 (`example%`)，而不是搜尋整個字串 (`%example%`)。

若要定義此行為，您可以將 PHP 屬性指派給模型的 `toSearchableArray` 方法。任何未指派額外搜尋策略行為的欄位將繼續使用預設的「where like」策略：

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
> 在指定欄位應使用全文查詢約束之前，請確保該欄位已指派 [全文索引](/docs/{{version}}/migrations#available-index-types)。

<a name="collection-engine"></a>
### Collection 引擎

雖然您可以在本地開發期間自由使用 Algolia、Meilisearch 或 Typesense 搜尋引擎，但您可能會發現使用「collection」引擎更方便。Collection 引擎將使用「where」子句和 Collection 篩選來篩選現有資料庫中的結果，以確定查詢的適用搜尋結果。使用此引擎時，無需「索引」您的可搜尋模型，因為它們將簡單地從您的本地資料庫中檢索。

若要使用 Collection 引擎，您可以簡單地將 `SCOUT_DRIVER` 環境變數的值設定為 `collection`，或直接在您應用程式的 `scout` 設定檔中指定 `collection` 驅動程式：

```ini
SCOUT_DRIVER=collection
```

一旦您將 Collection 驅動程式指定為您的首選驅動程式，您可以開始對您的模型 [執行搜尋查詢](#searching)。使用 Collection 引擎時，不需要搜尋引擎索引，例如為 Algolia、Meilisearch 或 Typesense 索引建立種子所需的索引。

#### 與資料庫引擎的差異

乍看之下，「database」和「collections」引擎非常相似。它們都直接與您的資料庫互動以檢索搜尋結果。但是，Collection 引擎不使用全文索引或 `LIKE` 子句來尋找匹配的記錄。相反，它會提取所有可能的記錄，並使用 Laravel 的 `Str::is` 輔助函數來判斷搜尋字串是否存在於模型屬性值中。

Collection 引擎是最具可攜性的搜尋引擎，因為它適用於 Laravel 支援的所有關聯式資料庫 (包括 SQLite 和 SQL Server)；但是，它的效率不如 Scout 的資料庫引擎。

<a name="indexing"></a>
## 索引

<a name="batch-import"></a>
### 批次匯入

如果您將 Scout 安裝到現有專案中，您可能已經有需要匯入到索引中的資料庫記錄。Scout 提供了一個 `scout:import` Artisan 命令，您可以使用它將所有現有記錄匯入到您的搜尋索引中：

```shell
php artisan scout:import "App\Models\Post"
```

`scout:queue-import` 命令可用於使用 [佇列工作](/docs/{{version}}/queues) 匯入所有現有記錄：

```shell
php artisan scout:queue-import "App\Models\Post" --chunk=500
```

`flush` 命令可用於從您的搜尋索引中移除模型的所有記錄：

```shell
php artisan scout:flush "App\Models\Post"
```

<a name="modifying-the-import-query"></a>
#### 修改匯入查詢

如果您想修改用於檢索所有模型以進行批次匯入的查詢，您可以在模型上定義一個 `makeAllSearchableUsing` 方法。這是新增任何在匯入模型之前可能需要的預先載入關聯的好地方：

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
> 當使用佇列批次匯入模型時，`makeAllSearchableUsing` 方法可能不適用。當模型 Collection 由工作處理時，關聯 [不會還原](/docs/{{version}}/queues#handling-relationships)。

<a name="adding-records"></a>
### 新增記錄

一旦您將 `Laravel\Scout\Searchable` Trait 新增到模型中，您只需 `save` 或 `create` 一個模型實例，它就會自動新增到您的搜尋索引中。如果您已設定 Scout [使用佇列](#queueing)，此操作將由您的佇列工作者在背景執行：

```php
use App\Models\Order;

$order = new Order;

// ...

$order->save();
```

<a name="adding-records-via-query"></a>
#### 透過查詢新增記錄

如果您想透過 Eloquent 查詢將模型 Collection 新增到您的搜尋索引中，您可以將 `searchable` 方法鏈接到 Eloquent 查詢。`searchable` 方法將 [分塊查詢結果](/docs/{{version}}/eloquent#chunking-results) 並將記錄新增到您的搜尋索引中。同樣，如果您已設定 Scout 使用佇列，所有分塊都將由您的佇列工作者在背景匯入：

```php
use App\Models\Order;

Order::where('price', '>', 100)->searchable();
```

您也可以在 Eloquent 關聯實例上呼叫 `searchable` 方法：

```php
$user->orders()->searchable();
```

或者，如果您記憶體中已經有一個 Eloquent 模型 Collection，您可以在 Collection 實例上呼叫 `searchable` 方法，將模型實例新增到其對應的索引中：

```php
$orders->searchable();
```

> [!NOTE]
> `searchable` 方法可以被視為一個「upsert」操作。換句話說，如果模型記錄已在您的索引中，它將被更新。如果它不存在於搜尋索引中，它將被新增到索引中。

<a name="updating-records"></a>
### 更新記錄

若要更新可搜尋模型，您只需更新模型實例的屬性並將模型 `save` 到您的資料庫。Scout 會自動將變更持久化到您的搜尋索引中：

```php
use App\Models\Order;

$order = Order::find(1);

// Update the order...

$order->save();
```

您也可以在 Eloquent 查詢實例上呼叫 `searchable` 方法來更新模型 Collection。如果模型不存在於您的搜尋索引中，它們將被建立：

```php
Order::where('price', '>', 100)->searchable();
```

如果您想更新關聯中所有模型的搜尋索引記錄，您可以在關聯實例上呼叫 `searchable`：

```php
$user->orders()->searchable();
```

或者，如果您記憶體中已經有一個 Eloquent 模型 Collection，您可以在 Collection 實例上呼叫 `searchable` 方法，以更新其對應索引中的模型實例：

```php
$orders->searchable();
```

<a name="modifying-records-before-importing"></a>
#### 匯入前修改記錄

有時您可能需要在模型可搜尋之前準備模型 Collection。例如，您可能希望預先載入關聯，以便將關聯資料有效地新增到您的搜尋索引中。若要實現此目的，請在對應的模型上定義一個 `makeSearchableUsing` 方法：

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

<a name="removing-records"></a>
### 移除記錄

若要從索引中移除記錄，您只需從資料庫中 `delete` 模型即可。即使您使用 [軟刪除](/docs/{{version}}/eloquent#soft-deleting) 模型，也可以這樣做：

```php
use App\Models\Order;

$order = Order::find(1);

$order->delete();
```

如果您不想在刪除記錄之前檢索模型，您可以在 Eloquent 查詢實例上使用 `unsearchable` 方法：

```php
Order::where('price', '>', 100)->unsearchable();
```

如果您想移除關聯中所有模型的搜尋索引記錄，您可以在關聯實例上呼叫 `unsearchable`：

```php
$user->orders()->unsearchable();
```

或者，如果您記憶體中已經有一個 Eloquent 模型 Collection，您可以在 Collection 實例上呼叫 `unsearchable` 方法，以從其對應索引中移除模型實例：

```php
$orders->unsearchable();
```

若要從其對應索引中移除所有模型記錄，您可以呼叫 `removeAllFromSearch` 方法：

```php
Order::removeAllFromSearch();
```

<a name="pausing-indexing"></a>
### 暫停索引

有時您可能需要在模型上執行批次 Eloquent 操作，而無需將模型資料同步到您的搜尋索引。您可以使用 `withoutSyncingToSearch` 方法來執行此操作。此方法接受一個單一的閉包，該閉包將立即執行。在閉包中發生的任何模型操作都不會同步到模型的索引：

```php
use App\Models\Order;

Order::withoutSyncingToSearch(function () {
    // Perform model actions...
});
```

<a name="conditionally-searchable-model-instances"></a>
### 條件式可搜尋模型實例

有時您可能只需要在特定條件下使模型可搜尋。例如，假設您有一個 `App\Models\Post` 模型，它可能處於兩種狀態之一：「草稿」和「已發布」。您可能只希望允許「已發布」的貼文可搜尋。若要實現此目的，您可以在模型上定義一個 `shouldBeSearchable` 方法：

```php
/**
 * Determine if the model should be searchable.
 */
public function shouldBeSearchable(): bool
{
    return $this->isPublished();
}
```

`shouldBeSearchable` 方法僅在透過 `save` 和 `create` 方法、查詢或關聯操作模型時才適用。直接使用 `searchable` 方法使模型或 Collection 可搜尋將覆寫 `shouldBeSearchable` 方法的結果。

> [!WARNING]
> `shouldBeSearchable` 方法不適用於使用 Scout 的「database」引擎，因為所有可搜尋資料始終儲存在資料庫中。若要在使用資料庫引擎時實現類似的行為，您應該改用 [where 子句](#where-clauses)。

<a name="searching"></a>
## 搜尋

您可以使用 `search` 方法開始搜尋模型。`search` 方法接受一個字串，該字串將用於搜尋您的模型。然後，您應該將 `get` 方法鏈接到搜尋查詢，以檢索與給定搜尋查詢匹配的 Eloquent 模型：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->get();
```

由於 Scout 搜尋會傳回 Eloquent 模型 Collection，您甚至可以直接從路由或控制器傳回結果，它們將自動轉換為 JSON：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/search', function (Request $request) {
    return Order::search($request->search)->get();
});
```

如果您想在將原始搜尋結果轉換為 Eloquent 模型之前取得它們，您可以使用 `raw` 方法：

```php
$orders = Order::search('Star Trek')->raw();
```

<a name="custom-indexes"></a>
#### 自訂索引

搜尋查詢通常會在模型 [searchableAs](#configuring-model-indexes) 方法指定的索引上執行。但是，您可以使用 `within` 方法來指定應改為搜尋的自訂索引：

```php
$orders = Order::search('Star Trek')
    ->within('tv_shows_popularity_desc')
    ->get();
```

<a name="where-clauses"></a>
### Where 子句

Scout 允許您在搜尋查詢中新增簡單的「where」子句。目前，這些子句僅支援基本的數值相等檢查，主要用於透過擁有者 ID 限制搜尋查詢的範圍：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->where('user_id', 1)->get();
```

此外，`whereIn` 方法可用於驗證給定欄位的值是否包含在給定陣列中：

```php
$orders = Order::search('Star Trek')->whereIn(
    'status', ['open', 'paid']
)->get();
```

`whereNotIn` 方法驗證給定欄位的值是否不包含在給定陣列中：

```php
$orders = Order::search('Star Trek')->whereNotIn(
    'status', ['closed']
)->get();
```

由於搜尋索引不是關聯式資料庫，因此目前不支援更進階的「where」子句。

> [!WARNING]
> 如果您的應用程式使用 Meilisearch，您必須在利用 Scout 的「where」子句之前設定應用程式的 [可篩選屬性](#configuring-filterable-data-for-meilisearch)。

<a name="pagination"></a>
### 分頁

除了檢索模型 Collection 之外，您還可以使用 `paginate` 方法對搜尋結果進行分頁。此方法將傳回一個 `Illuminate\Pagination\LengthAwarePaginator` 實例，就像您 [對傳統 Eloquent 查詢進行分頁](/docs/{{version}}/pagination) 一樣：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->paginate();
```

您可以透過將數量作為 `paginate` 方法的第一個參數傳遞，來指定每頁要檢索的模型數量：

```php
$orders = Order::search('Star Trek')->paginate(15);
```

一旦您檢索到結果，您可以使用 [Blade](/docs/{{version}}/blade) 顯示結果並呈現頁面連結，就像您對傳統 Eloquent 查詢進行分頁一樣：

```html
<div class="container">
    @foreach ($orders as $order)
        {{ $order->price }}
    @endforeach
</div>

{{ $orders->links() }}
```

當然，如果您想將分頁結果作為 JSON 檢索，您可以直接從路由或控制器傳回分頁器實例：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    return Order::search($request->input('query'))->paginate(15);
});
```

> [!WARNING]
> 由於搜尋引擎不了解您的 Eloquent 模型的全域範圍定義，因此在使用 Scout 分頁的應用程式中不應使用全域範圍。或者，您應該在使用 Scout 搜尋時重新建立全域範圍的約束。

<a name="soft-deleting"></a>
### 軟刪除

如果您的索引模型是 [軟刪除](/docs/{{version}}/eloquent#soft-deleting) 的，並且您需要搜尋您的軟刪除模型，請將 `config/scout.php` 設定檔的 `soft_delete` 選項設定為 `true`：

```php
'soft_delete' => true,
```

當此設定選項為 `true` 時，Scout 不會從搜尋索引中移除軟刪除模型。相反，它會在索引記錄上設定一個隱藏的 `__soft_deleted` 屬性。然後，您可以在搜尋時使用 `withTrashed` 或 `onlyTrashed` 方法來檢索軟刪除記錄：

```php
use App\Models\Order;

// Include trashed records when retrieving results...
$orders = Order::search('Star Trek')->withTrashed()->get();

// Only include trashed records when retrieving results...
$orders = Order::search('Star Trek')->onlyTrashed()->get();
```

> [!NOTE]
> 當使用 `forceDelete` 永久刪除軟刪除模型時，Scout 會自動將其從搜尋索引中移除。

<a name="customizing-engine-searches"></a>
### 自訂引擎搜尋

如果您需要對引擎的搜尋行為進行進階自訂，您可以將閉包作為 `search` 方法的第二個參數傳遞。例如，您可以使用此回呼在將搜尋查詢傳遞給 Algolia 之前，將地理位置資料新增到您的搜尋選項中：

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

<a name="customizing-the-eloquent-results-query"></a>
#### 自訂 Eloquent 結果查詢

在 Scout 從您應用程式的搜尋引擎檢索到匹配的 Eloquent 模型列表後，Eloquent 會使用它們的主鍵檢索所有匹配的模型。您可以透過呼叫 `query` 方法來自訂此查詢。`query` 方法接受一個閉包，該閉包將接收 Eloquent 查詢建構器實例作為參數：

```php
use App\Models\Order;
use Illuminate\Database\Eloquent\Builder;

$orders = Order::search('Star Trek')
    ->query(fn (Builder $query) => $query->with('invoices'))
    ->get();
```

由於此回呼是在相關模型已從您應用程式的搜尋引擎檢索後呼叫的，因此不應使用 `query` 方法來「篩選」結果。相反，您應該使用 [Scout where 子句](#where-clauses)。

<a name="custom-engines"></a>
## 自訂引擎

<a name="writing-the-engine"></a>
#### 編寫引擎

如果內建的 Scout 搜尋引擎不符合您的需求，您可以編寫自己的自訂引擎並將其註冊到 Scout。您的引擎應該擴展 `Laravel\Scout\Engines\Engine` 抽象類別。此抽象類別包含您的自訂引擎必須實作的八個方法：

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

您可能會發現檢閱 `Laravel\Scout\Engines\AlgoliaEngine` 類別上這些方法的實作很有幫助。此類別將為您提供一個很好的起點，以了解如何在自己的引擎中實作這些方法。

<a name="registering-the-engine"></a>
#### 註冊引擎

一旦您編寫了自訂引擎，您可以使用 Scout 引擎管理器的 `extend` 方法將其註冊到 Scout。Scout 的引擎管理器可以從 Laravel 服務容器中解析。您應該從 `App\Providers\AppServiceProvider` 類別或您應用程式使用的任何其他服務提供者的 `boot` 方法中呼叫 `extend` 方法：

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

一旦您的引擎註冊完成，您可以在您應用程式的 `config/scout.php` 設定檔中將其指定為您的預設 Scout `driver`：

```php
'driver' => 'mysql',
```
