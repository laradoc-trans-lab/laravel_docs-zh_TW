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
    - [設定每個模型的搜尋引擎](#configuring-search-engines-per-model)
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
    - [Where 語句](#where-clauses)
    - [分頁](#pagination)
    - [軟刪除](#soft-deleting)
    - [自訂引擎搜尋](#customizing-engine-searches)
- [自訂引擎](#custom-engines)

<a name="introduction"></a>
## 簡介

[Laravel Scout](https://github.com/laravel/scout) 提供了一個簡單、基於驅動程式的解決方案，可為您的 [Eloquent models](/docs/{{version}}/eloquent) 添加全文搜尋功能。透過模型觀察器 (model observers)，Scout 會自動讓您的搜尋索引與 Eloquent 記錄保持同步。

目前，Scout 隨附 [Algolia](https://www.algolia.com/)、[Meilisearch](https://www.meilisearch.com)、[Typesense](https://typesense.org) 以及 MySQL / PostgreSQL (`database`) 驅動程式。此外，Scout 還包含一個專為本地開發用途設計的「collection」驅動程式，它不需要任何外部依賴或第三方服務。更棒的是，撰寫自訂驅動程式非常簡單，您可以自由地使用自己的搜尋實作來擴展 Scout。

<a name="installation"></a>
## 安裝

首先，透過 Composer 套件管理器安裝 Scout：

```shell
composer require laravel/scout
```

安裝 Scout 後，您應該使用 `vendor:publish` Artisan 指令發布 Scout 設定檔。此指令會將 `scout.php` 設定檔發布到您應用程式的 `config` 目錄中：

```shell
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

最後，將 `Laravel\Scout\Searchable` trait 添加到您想要設為可搜尋的模型中。此 trait 會註冊一個模型觀察器，該觀察器會自動使模型與您的搜尋驅動程式保持同步：

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

當使用的引擎不是 `database` 或 `collection` 引擎時，您在使用此函式庫之前，應該強烈考慮設定一個 [佇列驅動程式](/docs/{{version}}/queues)。運行一個佇列 worker 將允許 Scout 將所有將模型資訊同步到搜尋索引的操作加入佇列，為您應用程式的網頁介面提供更好的響應時間。

設定好佇列驅動程式後，將您 `config/scout.php` 設定檔中的 `queue` 選項值設定為 `true`：

```php
'queue' => true,
```

即使 `queue` 選項設定為 `false`，也請務必記住，某些 Scout 驅動程式 (如 Algolia 和 Meilisearch) 總是會非同步索引記錄。這表示即使索引操作在您的 Laravel 應用程式中已完成，搜尋引擎本身可能不會立即反映新增和更新的記錄。

要指定 Scout Job 所使用的連線和佇列，您可以將 `queue` 設定選項定義為一個陣列：

```php
'queue' => [
    'connection' => 'redis',
    'queue' => 'scout'
],
```

當然，如果您自訂了 Scout Job 所使用的連線和佇列，您應該運行一個佇列 worker 來處理該連線和佇列上的工作：

```shell
php artisan queue:work redis --queue=scout
```

<a name="driver-prerequisites"></a>
## 驅動程式先決條件

<a name="algolia"></a>
### Algolia

使用 Algolia 驅動程式時，您應該在應用程式的 `config/scout.php` 設定檔中設定您的 Algolia `id` 和 `secret` 憑證。一旦憑證設定完成，您還需要透過 Composer 套件管理器安裝 Algolia PHP SDK：

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

然後，在應用程式的 `.env` 檔案中設定 `SCOUT_DRIVER` 環境變數以及您的 Meilisearch `host` 和 `key` 憑證：

```ini
SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=masterKey
```

有關 Meilisearch 的更多資訊，請查閱 [Meilisearch 文件](https://docs.meilisearch.com/learn/getting_started/quick_start.html)。

此外，您應該透過查閱 [Meilisearch 關於二進制相容性的文件](https://github.com/meilisearch/meilisearch-php#-compatibility-with-meilisearch)，確保您安裝的 `meilisearch/meilisearch-php` 版本與您的 Meilisearch 二進制版本相容。

> [!WARNING]
> 在利用 Meilisearch 的應用程式上升級 Scout 時，您應該始終 [查閱 Meilisearch 服務本身的任何額外破壞性變更](https://github.com/meilisearch/Meilisearch/releases)。

<a name="typesense"></a>
### Typesense

[Typesense](https://typesense.org) 是一個閃電般快速、開源的搜尋引擎，支援關鍵字搜尋、語義搜尋、地理搜尋和向量搜尋。

您可以 [自託管](https://typesense.org/docs/guide/install-typesense.html#option-2-local-machine-self-hosting) Typesense 或使用 [Typesense Cloud](https://cloud.typesense.org)。

要開始使用 Typesense 與 Scout，請透過 Composer 套件管理器安裝 Typesense PHP SDK：

```shell
composer require typesense/typesense-php
```

然後，在應用程式的 .env 檔案中設定 `SCOUT_DRIVER` 環境變數以及您的 Typesense host 和 API key 憑證：

```ini
SCOUT_DRIVER=typesense
TYPESENSE_API_KEY=masterKey
TYPESENSE_HOST=localhost
```

如果您正在使用 [Laravel Sail](/docs/{{version}}/sail)，您可能需要調整 `TYPESENSE_HOST` 環境變數以符合 Docker 容器名稱。您也可以選擇性地指定安裝的 port、path 和 protocol：

```ini
TYPESENSE_PORT=8108
TYPESENSE_PATH=
TYPESENSE_PROTOCOL=http
```

您可以在應用程式的 `config/scout.php` 設定檔中找到 Typesense collections 的其他設定和結構定義。有關 Typesense 的更多資訊，請查閱 [Typesense 文件](https://typesense.org/docs/guide/#quick-start)。

<a name="preparing-data-for-storage-in-typesense"></a>
#### 準備資料以儲存至 Typesense

當使用 Typesense 時，您的可搜尋模型必須定義一個 `toSearchableArray` 方法，該方法將模型的主鍵轉換為字串，並將建立日期轉換為 UNIX 時間戳記：

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

您還應該在應用程式的 `config/scout.php` 檔案中定義您的 Typesense collection 結構。collection 結構描述了每個可透過 Typesense 搜尋的欄位的資料型別。有關所有可用結構選項的更多資訊，請查閱 [Typesense 文件](https://typesense.org/docs/latest/api/collections.html#schema-parameters)。

如果您需要在定義 Typesense collection 結構後進行變更，您可以執行 `scout:flush` 和 `scout:import`，這將刪除所有現有的索引資料並重新建立結構。或者，您可以使用 Typesense 的 API 來修改 collection 的結構，而無需移除任何索引資料。

如果您的可搜尋模型是軟刪除的，您應該在應用程式的 `config/scout.php` 設定檔中，於模型對應的 Typesense 結構中定義一個 `__soft_deleted` 欄位：

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

Typesense 允許您在透過 `options` 方法執行搜尋操作時，動態修改您的 [search parameters](https://typesense.org/docs/latest/api/search.html#search-parameters)：

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

每個 Eloquent 模型都會與一個特定的搜尋「索引」同步，該索引包含該模型所有可搜尋的記錄。換句話說，您可以將每個索引視為一個 MySQL 資料表。預設情況下，每個模型都會被持久化到一個與模型典型「資料表」名稱相符的索引中。通常，這是模型名稱的複數形式；但是，您可以透過在模型上覆寫 `searchableAs` 方法來客製化模型的索引：

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

預設情況下，模型的整個 `toArray` 形式將被持久化到其搜尋索引中。如果您想客製化同步到搜尋索引的資料，可以在模型上覆寫 `toSearchableArray` 方法：

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

某些搜尋引擎（例如 Meilisearch）只會對正確型別的資料執行篩選操作（`>`、`<` 等）。因此，當使用這些搜尋引擎並客製化您的可搜尋資料時，您應該確保數字值被轉換為其正確的型別：

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

有時候您可能想在 Algolia 索引上配置額外的設定。儘管您可以透過 Algolia UI 管理這些設定，但有時直接從應用程式的 `config/scout.php` 設定檔中管理索引設定的期望狀態會更有效率。

這種方法允許您透過應用程式的自動化部署流程來部署這些設定，避免手動配置並確保多個環境之間的一致性。您可以配置可篩選屬性、排序、分面或[任何其他支援的設定](https://www.algolia.com/doc/rest-api/search/#tag/Indices/operation/setSettings)。

要開始使用，請在應用程式的 `config/scout.php` 設定檔中為每個索引新增設定：

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

如果特定索引所屬的模型是可軟刪除的，並且包含在 `index-settings` 陣列中，Scout 將自動為該索引上的軟刪除模型包含分面支援。如果您沒有其他分面屬性要為可軟刪除的模型索引定義，您只需為該模型新增一個空的 `index-settings` 條目：

```php
'index-settings' => [
    Flight::class => []
],
```

配置完應用程式的索引設定後，您必須呼叫 `scout:sync-index-settings` Artisan 指令。此指令將通知 Algolia 您當前配置的索引設定。為了方便起見，您可能希望將此指令作為部署流程的一部分：

```shell
php artisan scout:sync-index-settings
```

<a name="configuring-filterable-data-for-meilisearch"></a>
#### 設定可篩選資料與索引設定 (Meilisearch)

與 Scout 的其他驅動程式不同，Meilisearch 要求您預先定義索引搜尋設定，例如可篩選屬性、可排序屬性以及[其他支援的設定欄位](https://docs.meilisearch.com/reference/api/settings.html)。

可篩選屬性是您計劃在使用 Scout 的 `where` 方法時進行篩選的任何屬性，而可排序屬性是您計劃在使用 Scout 的 `orderBy` 方法時進行排序的任何屬性。要定義您的索引設定，請調整應用程式 `scout` 設定檔中 `meilisearch` 設定項目中的 `index-settings` 部分：

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

如果特定索引所屬的模型是可軟刪除的，並且包含在 `index-settings` 陣列中，Scout 將自動為該索引上的軟刪除模型包含篩選支援。如果您沒有其他可篩選或可排序的屬性要為可軟刪除的模型索引定義，您只需為該模型新增一個空的 `index-settings` 條目：

```php
'index-settings' => [
    Flight::class => []
],
```

配置完應用程式的索引設定後，您必須呼叫 `scout:sync-index-settings` Artisan 指令。此指令將通知 Meilisearch 您當前配置的索引設定。為了方便起見，您可能希望將此指令作為部署流程的一部分：

```shell
php artisan scout:sync-index-settings
```

<a name="configuring-the-model-id"></a>
### 設定模型 ID

預設情況下，Scout 會使用模型的主鍵作為儲存在搜尋索引中的模型唯一 ID / 鍵。如果您需要客製化此行為，可以在模型上覆寫 `getScoutKey` 和 `getScoutKeyName` 方法：

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
### 設定每個模型的搜尋引擎

搜尋時，Scout 通常會使用應用程式 `scout` 設定檔中指定的預設搜尋引擎。但是，可以透過在模型上覆寫 `searchableUsing` 方法來變更特定模型的搜尋引擎：

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

<a name="identifying-users"></a>
### 識別使用者

Scout 也允許您在使用 [Algolia](https://algolia.com) 時自動識別使用者。將已驗證的使用者與搜尋操作關聯起來，在 Algolia 的儀表板中查看搜尋分析時可能會很有幫助。您可以透過在應用程式的 `.env` 檔案中將 `SCOUT_IDENTIFY` 環境變數定義為 `true` 來啟用使用者識別功能：

```ini
SCOUT_IDENTIFY=true
```

啟用此功能也會將請求的 IP 位址以及您已驗證使用者的主要識別符傳遞給 Algolia，如此一來，這些資料就會與使用者發出的任何搜尋請求相關聯。

<a name="database-and-collection-engines"></a>
## 資料庫 / Collection 引擎


<a name="database-engine"></a>
### 資料庫引擎

> [!WARNING]
> 資料庫引擎目前支援 MySQL 和 PostgreSQL。

`database` 引擎是開始使用 Laravel Scout 最快的方式，它使用 MySQL / PostgreSQL 的全文索引 (full-text indexes) 和「where like」語句，從現有的資料庫中篩選結果，以確定查詢適用的搜尋結果。

要使用資料庫引擎，您只需將 `SCOUT_DRIVER` 環境變數的值設定為 `database`，或者在應用程式的 `scout` 設定檔中直接指定 `database` 驅動程式：

```ini
SCOUT_DRIVER=database
```

一旦您將資料庫引擎指定為首選驅動程式，您就必須[設定您的可搜尋資料](#configuring-searchable-data)。然後，您可以開始針對您的模型[執行搜尋查詢](#searching)。使用資料庫引擎時，不需要搜尋引擎索引，例如為 Algolia、Meilisearch 或 Typesense 索引建立種子資料所需的索引。


#### 自訂資料庫搜尋策略

預設情況下，資料庫引擎會針對您[設定為可搜尋](#configuring-searchable-data)的每個模型屬性執行「where like」查詢。然而，在某些情況下，這可能會導致效能不佳。因此，資料庫引擎的搜尋策略可以進行配置，讓某些指定的欄位使用全文搜尋查詢，或者只使用「where like」限制來搜尋字串的前綴 (例如 `example%`)，而不是在整個字串中搜尋 (例如 `%example%`)。

若要定義此行為，您可以為模型的 `toSearchableArray` 方法指定 PHP 屬性。任何未指定額外搜尋策略行為的欄位將繼續使用預設的「where like」策略：

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
> 在指定欄位應使用全文查詢限制之前，請確保該欄位已指定[全文索引](/docs/{{version}}/migrations#available-index-types)。


<a name="collection-engine"></a>
### Collection 引擎

雖然您可以在本機開發期間自由使用 Algolia、Meilisearch 或 Typesense 搜尋引擎，但您可能會發現使用「collection」引擎更方便。Collection 引擎將使用「where」子句和集合篩選來處理現有資料庫中的結果，以確定查詢適用的搜尋結果。使用此引擎時，不需要「索引」您的可搜尋模型，因為它們將直接從您的本機資料庫中擷取。

要使用 collection 引擎，您可以簡單地將 `SCOUT_DRIVER` 環境變數的值設定為 `collection`，或者在應用程式的 `scout` 設定檔中直接指定 `collection` 驅動程式：

```ini
SCOUT_DRIVER=collection
```

一旦您將 collection 驅動程式指定為首選驅動程式，您就可以開始針對您的模型[執行搜尋查詢](#searching)了。使用 collection 引擎時，不需要搜尋引擎索引，例如為 Algolia、Meilisearch 或 Typesense 索引建立種子資料所需的索引。


#### 與資料庫引擎的差異

乍看之下，「database」和「collections」引擎相當相似。它們都直接與您的資料庫互動以檢索搜尋結果。然而，collection 引擎不使用全文索引或 `LIKE` 子句來尋找匹配的記錄。相反地，它會拉取所有可能的記錄，並使用 Laravel 的 `Str::is` 輔助函式來判斷搜尋字串是否存在於模型屬性值中。

collection 引擎是最具可攜性的搜尋引擎，因為它適用於 Laravel 支援的所有關聯式資料庫 (包括 SQLite 和 SQL Server)；然而，它的效率遠低於 Scout 的資料庫引擎。

<a name="indexing"></a>
## 索引


<a name="batch-import"></a>
### 批次匯入

若您將 Scout 安裝到現有專案中，可能已經有需要匯入到索引的資料庫記錄。Scout 提供了一個 `scout:import` Artisan 指令，可用來將所有現有記錄匯入到搜尋索引中：

```shell
php artisan scout:import "App\Models\Post"
```

`scout:queue-import` 指令可用來使用[佇列作業](/docs/{{version}}/queues)匯入所有現有記錄：

```shell
php artisan scout:queue-import "App\Models\Post" --chunk=500
```

`flush` 指令可用來從搜尋索引中移除模型的所有記錄：

```shell
php artisan scout:flush "App\Models\Post"
```


<a name="modifying-the-import-query"></a>
#### 修改匯入查詢

如果您想修改用於批次匯入所有模型的查詢，可以在模型上定義一個 `makeAllSearchableUsing` 方法。這是加入任何在匯入模型之前可能需要的預先載入關聯的好地方：

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
> `makeAllSearchableUsing` 方法在使用佇列批次匯入模型時可能不適用。在佇列作業處理模型集合時，關聯[不會被還原](/docs/{{version}}/queues#handling-relationships)。


<a name="adding-records"></a>
### 新增記錄

一旦您將 `Laravel\Scout\Searchable` trait 加入到模型中，您所需要做的就是 `save` 或 `create` 一個模型實例，它將自動被新增到您的搜尋索引中。如果您已設定 Scout [使用佇列](#queueing)，此操作將由佇列工作者在背景執行：

```php
use App\Models\Order;

$order = new Order;

// ...

$order->save();
```


<a name="adding-records-via-query"></a>
#### 透過查詢新增記錄

如果您想透過 Eloquent 查詢將模型集合新增到您的搜尋索引中，可以將 `searchable` 方法串接到 Eloquent 查詢上。`searchable` 方法會將查詢結果[分塊](/docs/{{version}}/eloquent#chunking-results)並將記錄新增到您的搜尋索引中。同樣地，如果您已設定 Scout 使用佇列，所有分塊都將由佇列工作者在背景匯入：

```php
use App\Models\Order;

Order::where('price', '>', 100)->searchable();
```

您也可以在 Eloquent 關聯實例上呼叫 `searchable` 方法：

```php
$user->orders()->searchable();
```

或者，如果您記憶體中已經有一個 Eloquent 模型集合，可以在該集合實例上呼叫 `searchable` 方法，將模型實例新增到其對應的索引中：

```php
$orders->searchable();
```

> [!NOTE]
> `searchable` 方法可視為一個「upsert (更新/插入)」操作。換句話說，如果模型記錄已存在於您的索引中，它將被更新。如果它不存在於搜尋索引中，它將被新增到索引中。


<a name="updating-records"></a>
### 更新記錄

要更新可搜尋模型，您只需更新模型實例的屬性並將模型 `save` 到資料庫即可。Scout 將自動將變更持久化到您的搜尋索引中：

```php
use App\Models\Order;

$order = Order::find(1);

// Update the order...

$order->save();
```

您也可以在 Eloquent 查詢實例上呼叫 `searchable` 方法來更新模型集合。如果模型不存在於您的搜尋索引中，它們將被建立：

```php
Order::where('price', '>', 100)->searchable();
```

如果您想更新關聯中所有模型的搜尋索引記錄，可以在關聯實例上呼叫 `searchable`：

```php
$user->orders()->searchable();
```

或者，如果您記憶體中已經有一個 Eloquent 模型集合，可以在該集合實例上呼叫 `searchable` 方法，以更新其對應索引中的模型實例：

```php
$orders->searchable();
```


<a name="modifying-records-before-importing"></a>
#### 匯入前修改記錄

有時您可能需要在模型變得可搜尋之前準備模型集合。例如，您可能希望預先載入一個關聯，以便將關聯資料有效地新增到您的搜尋索引中。為此，請在相應的模型上定義一個 `makeSearchableUsing` 方法：

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

要從索引中移除記錄，您只需從資料庫中 `delete` 模型即可。即使您正在使用[軟刪除](/docs/{{version}}/eloquent#soft-deleting)模型，這也可以完成：

```php
use App\Models\Order;

$order = Order::find(1);

$order->delete();
```

如果您不想在刪除記錄之前先擷取模型，可以在 Eloquent 查詢實例上使用 `unsearchable` 方法：

```php
Order::where('price', '>', 100)->unsearchable();
```

如果您想移除關聯中所有模型的搜尋索引記錄，可以在關聯實例上呼叫 `unsearchable`：

```php
$user->orders()->unsearchable();
```

或者，如果您記憶體中已經有一個 Eloquent 模型集合，可以在該集合實例上呼叫 `unsearchable` 方法，以從其對應索引中移除模型實例：

```php
$orders->unsearchable();
```

要從其對應索引中移除所有模型記錄，您可以呼叫 `removeAllFromSearch` 方法：

```php
Order::removeAllFromSearch();
```


<a name="pausing-indexing"></a>
### 暫停索引

有時您可能需要在模型上執行批次 Eloquent 操作，而不同步模型資料到您的搜尋索引。您可以使用 `withoutSyncingToSearch` 方法來執行此操作。此方法接受一個將立即執行的閉包。在該閉包中發生的任何模型操作都不會同步到模型的索引中：

```php
use App\Models\Order;

Order::withoutSyncingToSearch(function () {
    // Perform model actions...
});
```


<a name="conditionally-searchable-model-instances"></a>
### 條件式可搜尋模型實例

有時您可能需要僅在特定條件下才使模型可搜尋。例如，假設您有一個 `App\Models\Post` 模型，它可能處於兩種狀態之一：「草稿 (draft)」和「已發布 (published)」。您可能只想允許「已發布 (published)」的貼文可搜尋。為此，您可以在模型上定義一個 `shouldBeSearchable` 方法：

```php
/**
 * Determine if the model should be searchable.
 */
public function shouldBeSearchable(): bool
{
    return $this->isPublished();
}
```

`shouldBeSearchable` 方法僅在透過 `save` 和 `create` 方法、查詢或關聯操作模型時應用。直接使用 `searchable` 方法使模型或集合可搜尋將會覆寫 `shouldBeSearchable` 方法的結果。

> [!WARNING]
> 當使用 Scout 的「資料庫 (database)」引擎時，`shouldBeSearchable` 方法不適用，因為所有可搜尋資料都始終儲存在資料庫中。若要在使用資料庫引擎時實現類似的行為，您應該改用 [where 語句](#where-clauses)。

<a name="searching"></a>
## 搜尋

您可以使用 `search` 方法來開始搜尋模型。該 search 方法接受一個字串，用於搜尋您的模型。然後，您應該將 `get` 方法鏈接到搜尋查詢上，以檢索符合給定搜尋查詢的 Eloquent 模型：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->get();
```

由於 Scout 搜尋會返回 Eloquent 模型的集合，您甚至可以直接從路由或控制器返回結果，它們將自動轉換為 JSON：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/search', function (Request $request) {
    return Order::search($request->search)->get();
});
```

如果您想在將原始搜尋結果轉換為 Eloquent 模型之前取得它們，可以使用 `raw` 方法：

```php
$orders = Order::search('Star Trek')->raw();
```


<a name="custom-indexes"></a>
#### 自訂索引

搜尋查詢通常會針對模型 [searchableAs](#configuring-model-indexes) 方法所指定的索引執行。但是，您可以使用 `within` 方法來指定應改為搜尋的自訂索引：

```php
$orders = Order::search('Star Trek')
    ->within('tv_shows_popularity_desc')
    ->get();
```


<a name="where-clauses"></a>
### Where 語句

Scout 允許您在搜尋查詢中加入簡單的「where」語句。目前，這些語句僅支援基本的數字相等檢查，主要用於根據擁有者 ID 限制搜尋查詢的範圍：

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

`whereNotIn` 方法則用於驗證給定欄位的值是否不包含在給定陣列中：

```php
$orders = Order::search('Star Trek')->whereNotIn(
    'status', ['closed']
)->get();
```

由於搜尋索引不是關聯式資料庫，因此目前不支援更進階的「where」語句。

> [!WARNING]
> 如果您的應用程式正在使用 Meilisearch，您必須先設定應用程式的[可篩選屬性](#configuring-filterable-data-for-meilisearch)，然後才能使用 Scout 的「where」語句。


<a name="pagination"></a>
### 分頁

除了檢索模型集合之外，您還可以使用 `paginate` 方法對搜尋結果進行分頁。此方法將返回一個 `Illuminate\Pagination\LengthAwarePaginator` 實例，就像您[對傳統 Eloquent 查詢進行分頁](/docs/{{version}}/pagination)一樣：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->paginate();
```

您可以透過將數量作為 `paginate` 方法的第一個參數傳遞，來指定每頁要檢索多少個模型：

```php
$orders = Order::search('Star Trek')->paginate(15);
```

檢索結果後，您可以顯示結果並使用 [Blade](/docs/{{version}}/blade) 渲染頁面連結，就像您對傳統 Eloquent 查詢進行分頁一樣：

```html
<div class="container">
    @foreach ($orders as $order)
        {{ $order->price }}
    @endforeach
</div>

{{ $orders->links() }}
```

當然，如果您想以 JSON 格式檢索分頁結果，可以直接從路由或控制器返回 paginator 實例：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    return Order::search($request->input('query'))->paginate(15);
});
```

> [!WARNING]
> 由於搜尋引擎無法感知您的 Eloquent 模型全域範圍定義，因此在使用 Scout 分頁的應用程式中，不應使用全域範圍。或者，您應該在使用 Scout 搜尋時重新建立全域範圍的限制。


<a name="soft-deleting"></a>
### 軟刪除

如果您的索引模型正在[軟刪除](/docs/{{version}}/eloquent#soft-deleting)，並且您需要搜尋已軟刪除的模型，請將 `config/scout.php` 設定檔中的 `soft_delete` 選項設為 `true`：

```php
'soft_delete' => true,
```

當此設定選項為 `true` 時，Scout 將不會從搜尋索引中移除軟刪除的模型。相反地，它會在索引記錄上設定一個隱藏的 `__soft_deleted` 屬性。然後，您可以在搜尋時使用 `withTrashed` 或 `onlyTrashed` 方法來檢索軟刪除的記錄：

```php
use App\Models\Order;

// Include trashed records when retrieving results...
$orders = Order::search('Star Trek')->withTrashed()->get();

// Only include trashed records when retrieving results...
$orders = Order::search('Star Trek')->onlyTrashed()->get();
```

> [!NOTE]
> 當使用 `forceDelete` 永久刪除軟刪除的模型時，Scout 將自動從搜尋索引中移除它。


<a name="customizing-engine-searches"></a>
### 自訂引擎搜尋

如果您需要對引擎的搜尋行為進行進階自訂，您可以將閉包作為 `search` 方法的第二個參數傳遞。例如，您可以使用此回呼在搜尋查詢傳遞給 Algolia 之前，向搜尋選項中加入地理位置資料：

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

在 Scout 從應用程式的搜尋引擎檢索到匹配的 Eloquent 模型列表後，Eloquent 將用於透過其主鍵檢索所有匹配的模型。您可以透過調用 `query` 方法來自訂此查詢。該 `query` 方法接受一個閉包，該閉包將接收 Eloquent 查詢建構器實例作為參數：

```php
use App\Models\Order;
use Illuminate\Database\Eloquent\Builder;

$orders = Order::search('Star Trek')
    ->query(fn (Builder $query) => $query->with('invoices'))
    ->get();
```

由於此回呼是在相關模型已從應用程式的搜尋引擎中檢索之後調用的，因此 `query` 方法不應用於「篩選」結果。相反地，您應該使用 [Scout where 語句](#where-clauses)。

<a name="custom-engines"></a>
## 自訂引擎


<a name="writing-the-engine"></a>
#### 撰寫引擎

如果內建的 Scout 搜尋引擎無法滿足您的需求，您可以撰寫自己的自訂引擎並向 Scout 註冊。您的引擎應該繼承 `Laravel\Scout\Engines\Engine` 抽象類別。這個抽象類別包含八個您必須實作的自訂引擎方法：

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

您可以參考 `Laravel\Scout\Engines\AlgoliaEngine` 類別中這些方法的實作。這將為您提供一個良好的起點，以了解如何在自己的引擎中實作這些方法。


<a name="registering-the-engine"></a>
#### 註冊引擎

撰寫完自訂引擎後，您可以透過 Scout 引擎管理器的 `extend` 方法向 Scout 註冊它。Scout 的引擎管理器可以從 Laravel 服務容器中解析。您應該在 `App\Providers\AppServiceProvider` 類別或其他應用程式使用的任何服務提供者的 `boot` 方法中呼叫 `extend` 方法：

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

一旦您的引擎註冊完成，您可以在應用程式的 `config/scout.php` 設定檔中將其指定為預設的 Scout `driver`：

```php
'driver' => 'mysql',
```