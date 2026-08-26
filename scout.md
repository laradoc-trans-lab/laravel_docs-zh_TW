# Laravel Scout

- [簡介](#introduction)
- [安裝](#installation)
    - [佇列化](#queueing)
- [驅動程式前置需求](#driver-prerequisites)
    - [Algolia](#algolia)
    - [Meilisearch](#meilisearch)
    - [Typesense](#typesense)
- [設定](#configuration)
    - [設定可搜尋資料](#configuring-searchable-data)
- [資料庫 / 集合引擎](#database-and-collection-engines)
    - [資料庫引擎](#database-engine)
    - [集合引擎](#collection-engine)
- [第三方引擎設定](#third-party-engine-configuration)
    - [設定 Model 索引](#configuring-model-indexes)
    - [Algolia](#algolia-configuration)
    - [Meilisearch](#meilisearch-configuration)
    - [Typesense](#typesense-configuration)
- [第三方引擎索引](#indexing)
    - [整批匯入](#batch-import)
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

[Laravel Scout](https://github.com/laravel/scout) 為您的 [Eloquent Model](/docs/{{version}}/eloquent) 提供了一個簡單、基於驅動程式的全圖文搜尋（全文搜尋）解決方案。透過使用 Model Observer，Scout 會自動將您的搜尋索引與 Eloquent 紀錄保持同步。

Scout 內建了一個 `database` 引擎，使用 MySQL / PostgreSQL 全文索引與 `LIKE` 子句來直接搜尋您現有的資料庫——不需要外部服務。對大多數應用程式來說，這就足夠了。若要總覽 Laravel 中所有可用的搜尋選項，請參考[搜尋文件](/docs/{{version}}/search)。

當您需要錯字容忍（typo tolerance）、分面過濾（faceted filtering）或大規模地理位置搜尋（geo-search）等功能時，Scout 還包含了 [Algolia](https://www.algolia.com/)、[Meilisearch](https://www.meilisearch.com) 與 [Typesense](https://typesense.org) 的驅動程式。本地端開發時也有 "collection" 驅動程式可用，而且您也可以自由撰寫[自訂引擎](#custom-engines)。


<a name="installation"></a>
## 安裝

首先，透過 Composer 套件管理器安裝 Scout：

```shell
composer require laravel/scout
```

安裝 Scout 後，您應該使用 `vendor:publish` Artisan 指令發布 Scout 的設定檔。此指令會將 `scout.php` 設定檔發布到您應用程式的 `config` 目錄：

```shell
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

最後，將 `Laravel\Scout\Searchable` Trait 新增到您希望提供搜尋功能的 Model 中。此 Trait 會註冊一個 Model Observer，自動讓該 Model 與您的搜尋驅動程式保持同步：

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
### 佇列化

當使用的引擎不是 `database` 或 `collection` 引擎時，強烈建議您在使用此函式庫之前先設定好[佇列驅動程式](/docs/{{version}}/queues)。執行 Queue Worker 能讓 Scout 將所有把 Model 資訊同步至搜尋索引的操作進行佇列化，從而大幅提升您應用程式 Web 介面的回應速度。

設定好佇列驅動程式後，請將 `config/scout.php` 設定檔中的 `queue` 選項值設為 `true`：

```php
'queue' => true,
```

即使 `queue` 選項設置為 `false`，也必須記住某些 Scout 驅動程式（例如 Algolia 與 Meilisearch）永遠都是非同步地對紀錄進行索引。換句話說，即使索引操作已在您的 Laravel 應用程式內完成，搜尋引擎本身也可能不會立即反映出新增和更新的紀錄。

若要指定 Scout 任務（Job）所使用的連線與佇列，您可以將 `queue` 設定選項定義為陣列：

```php
'queue' => [
    'connection' => 'redis',
    'queue' => 'scout'
],
```

當然，如果您自訂了 Scout 任務所使用的連線與佇列，就應該執行 Queue Worker 來處理該連線與佇列上的任務：

```shell
php artisan queue:work redis --queue=scout
```


<a name="unique-jobs"></a>
#### 唯一任務

在寫入密集型（write-heavy）的應用程式中，您可能希望防止 Scout 針對相同的 Model 紀錄將重複的任務排入佇列。您可以透過註冊 `MakeSearchableUniquely` 與 `RemoveFromSearchUniquely` 任務類別來啟用唯一索引任務，這通常會在服務提供者的 `boot` 方法中進行：

```php
use Laravel\Scout\Jobs\MakeSearchableUniquely;
use Laravel\Scout\Jobs\RemoveFromSearchUniquely;
use Laravel\Scout\Scout;

Scout::makeSearchableUsing(MakeSearchableUniquely::class);
Scout::removeFromSearchUsing(RemoveFromSearchUniquely::class);
```

這些任務會使用 Laravel 的[唯一任務鎖](/docs/{{version}}/queues#unique-jobs)，以避免在相同的可搜尋 Model 紀錄已經有匹配的任務排入佇列時，又分派重複的佇列化索引操作。


<a name="driver-prerequisites"></a>
## 驅動程式前置需求


<a name="algolia"></a>
### Algolia

使用 Algolia 驅動程式時，您應該在 `config/scout.php` 設定檔中設定您的 Algolia `id` 與 `secret` 憑證。設定好憑證後，您還需要透過 Composer 套件管理器安裝 Algolia PHP SDK：

```shell
composer require algolia/algoliasearch-client-php
```


<a name="meilisearch"></a>
### Meilisearch

[Meilisearch](https://www.meilisearch.com) 是一個快速且開源的搜尋引擎。如果您不確定如何在本機電腦上安裝 Meilisearch，可以使用 [Laravel Sail](/docs/{{version}}/sail#meilisearch)，這是 Laravel 官方支援的 Docker 開發環境。

使用 Meilisearch 驅動程式時，您需要透過 Composer 套件管理器安裝 Meilisearch PHP SDK：

```shell
composer require meilisearch/meilisearch-php http-interop/http-factory-guzzle
```

然後，在應用程式的 `.env` 檔案中設定 `SCOUT_DRIVER` 環境變數以及您的 Meilisearch `host` 與 `key` 憑證：

```ini
SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=masterKey
```

有關 Meilisearch 的更多資訊，請參考 [Meilisearch 文件](https://docs.meilisearch.com/learn/getting_started/quick_start.html)。

此外，您應該透過參閱 [Meilisearch 關於執行檔相容性的文件](https://github.com/meilisearch/meilisearch-php#-compatibility-with-meilisearch)，確保您安裝的 `meilisearch/meilisearch-php` 版本與您的 Meilisearch 執行檔版本相容。

> [!WARNING]
> 在使用 Meilisearch 的應用程式上升級 Scout 時，您應該始終[查閱 Meilisearch 服務本身的任何其他重大變更](https://github.com/meilisearch/Meilisearch/releases)。


<a name="typesense"></a>
### Typesense

[Typesense](https://typesense.org) 是一個極速且開源的搜尋引擎，支援關鍵字搜尋、語意搜尋、地理位置搜尋與向量搜尋。

您可以[自行託管](https://typesense.org/docs/guide/install-typesense.html#option-2-local-machine-self-hosting) Typesense，或是使用 [Typesense Cloud](https://cloud.typesense.org)。

要開始在 Scout 中使用 Typesense，請透過 Composer 套件管理器安裝 Typesense PHP SDK：

```shell
composer require typesense/typesense-php
```

然後，在應用程式的 .env 檔案中設定 `SCOUT_DRIVER` 環境變數以及您的 Typesense 主機與 API 金鑰憑證：

```ini
SCOUT_DRIVER=typesense
TYPESENSE_API_KEY=masterKey
TYPESENSE_HOST=localhost
```

如果您正在使用 [Laravel Sail](/docs/{{version}}/sail)，您可能需要調整 `TYPESENSE_HOST` 環境變數以匹配 Docker 容器名稱。您也可以選擇性地指定安裝的連接埠、路徑與協定：

```ini
TYPESENSE_PORT=8108
TYPESENSE_PATH=
TYPESENSE_PROTOCOL=http
```

您 Typesense 集合的額外設定與 Schema 定義可以在應用程式的 `config/scout.php` 設定檔中找到。有關 Typesense 的更多資訊，請參考 [Typesense 文件](https://typesense.org/docs/guide/#quick-start)。

<a name="configuration"></a>
## 設定


<a name="configuring-searchable-data"></a>
### 設定可搜尋資料

預設情況下，給定 Model 的整個 `toArray` 形式都會被持久化儲存到其搜尋索引中。若您想要自訂同步到搜尋索引的資料，可以在 Model 上覆寫 `toSearchableArray` 方法：

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
#### 設定 Model 引擎

搜尋時，Scout 通常會使用您應用程式 `scout` 設定檔中所指定的預設搜尋引擎。然而，特定 Model 的搜尋引擎可以透過覆寫該 Model 上的 `searchableUsing` 方法來變更：

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
## 資料庫 / 集合引擎


<a name="database-engine"></a>
### 資料庫引擎

> [!WARNING]
> 資料庫引擎目前支援 MySQL 和 PostgreSQL，兩者皆提供對快速全文欄位索引的支援。

`database` 引擎使用 MySQL / PostgreSQL 的全文索引與 `LIKE` 子句直接搜尋您現有的資料庫。對大多數應用程式來說，這是新增搜尋功能最簡單且最實用的方式——不需要任何外部服務或額外的基礎設施。

若要使用資料庫引擎，請將 `SCOUT_DRIVER` 環境變數設定為 `database`：

```ini
SCOUT_DRIVER=database
```

設定完成後，您即可[設定可搜尋資料](#configuring-searchable-data)並開始針對您的 Model [執行搜尋查詢](#searching)。與第三方引擎不同，資料庫引擎不需要獨立的索引步驟——它會直接搜尋您的資料庫資料表。


#### 自訂資料庫搜尋策略

預設情況下，資料庫引擎會針對您[設定為可搜尋](#configuring-searchable-data)的每個 Model 屬性執行 `LIKE` 查詢。但是，您可以為特定欄位指派更高效率的搜尋策略。`SearchUsingFullText` 屬性（Attribute）將會對該欄位使用資料庫的全文索引，而 `SearchUsingPrefix` 則只會比對字串開頭（`example%`），而非搜尋整段字串（`%example%`）。

若要定義此行為，請將 PHP 屬性（Attribute）指定給您 Model 的 `toSearchableArray` 方法。任何沒有指定屬性的欄位將繼續使用預設的 `LIKE` 策略：

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
> 在指定欄位應使用全文查詢限制條件之前，請確保該欄位已被指派[全文索引](/docs/{{version}}/migrations#available-index-types)。


<a name="collection-engine"></a>
### 集合引擎

「collection」引擎適用於快速原型開發、極小規模的資料集（數百筆紀錄）或執行測試。它會從您的資料庫中取出所有可能的紀錄，並使用 Laravel 的 `Str::is` 輔助函式在 PHP 中進行過濾，因此不需要任何索引或特定於資料庫的功能。對於任何超出微型範疇的實務情境，您都應該改為使用[資料庫引擎](#database-engine)。

若要使用集合引擎，您只需將 `SCOUT_DRIVER` 環境變數的值設定為 `collection`，或者直接在應用程式的 `scout` 設定檔中指定 `collection` 驅動程式：

```ini
SCOUT_DRIVER=collection
```

將集合驅動程式指定為您偏好的驅動程式後，您就可以開始針對您的 Model [執行搜尋查詢](#searching)。當使用集合引擎時，並不需要進行搜尋引擎索引（例如填入 Algolia、Meilisearch 或 Typesense 索引所需的索引作業）。


#### 與資料庫引擎的差異

資料庫引擎使用全文索引與 `LIKE` 子句來高效地尋找符合的紀錄，而集合引擎則會拉取所有紀錄並在 PHP 中進行過濾。集合引擎是可攜性最高的選項，因為它可以跨 Laravel 支援的所有關聯式資料庫（包含 SQLite 和 SQL Server）運作；然而，它的效率顯著低於資料庫引擎，不應將其用於大型資料集。

<a name="third-party-engine-configuration"></a>
## 第三方引擎設定

以下的設定選項僅適用於使用第三方搜尋引擎（如 Algolia、Meilisearch 或 Typesense）。若您使用的是[資料庫引擎](#database-engine)，可以跳過此章節。

<a name="configuring-model-indexes"></a>
### 設定 Model 索引

當使用第三方引擎時，每個 Eloquent Model 都會與特定的搜尋「索引 (index)」同步，該索引包含該 Model 的所有可搜尋紀錄。預設情況下，每個 Model 都會被儲存到與該 Model 慣用的「資料表 (table)」名稱相符的索引中。通常這是 Model 名稱的複數形式；不過，您可以透過覆寫 Model 上的 `searchableAs` 方法來自由自訂 Model 的索引：

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
> 當使用資料庫引擎時，`searchableAs` 方法沒有任何效果，因為資料庫引擎總是直接搜尋 Model 的資料庫資料表。

<a name="configuring-the-model-id"></a>
#### 設定 Model ID

預設情況下，Scout 會使用 Model 的主鍵 (primary key) 作為儲存在搜尋索引中的 Model 唯一 ID / 鍵值。若您在使用第三方引擎時需要自訂此行為，可以在 Model 上覆寫 `getScoutKey` 與 `getScoutKeyName` 方法：

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
> 當使用資料庫引擎時，`getScoutKey` 與 `getScoutKeyName` 方法沒有任何效果，因為資料庫引擎總是使用 Model 的主鍵。

<a name="algolia-configuration"></a>
### Algolia

<a name="algolia-index-settings"></a>
#### 索引設定

有時候您可能想為 Algolia 索引設定額外設定值。雖然您可以透過 Algolia 使用者介面管理這些設定，但有時候直接從應用程式的 `config/scout.php` 設定檔管理索引設定的期望狀態會更有效率。

這種做法讓您可以透過應用程式的自動化部署流程來部署這些設定，避免手動設定並確保多個環境之間的一致性。您可以設定可過濾屬性 (filterable attributes)、排序 (ranking)、分面 (faceting) 或[任何其他支援的設定](https://www.algolia.com/doc/rest-api/search/#tag/Indices/operation/setSettings)。

首先，請在應用程式的 `config/scout.php` 設定檔中為每個索引新增設定：

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

若給定索引所對應的 Model 支援軟刪除 (soft deletable) 且已包含在 `index-settings` 陣列中，Scout 將會自動對該索引上的軟刪除 Model 提供分面 (faceting) 支援。若您沒有其他要為軟刪除 Model 索引定義的分面屬性，可以簡單地為該 Model 在 `index-settings` 陣列中新增一個空項目：

```php
'index-settings' => [
    Flight::class => []
],
```

在設定好應用程式的索引設定後，您必須執行 `scout:sync-index-settings` Artisan 指令。此指令會將您目前設定的索引設定通知 Algolia。為了方便起見，您可以將此指令作為部署流程的一部分：

```shell
php artisan scout:sync-index-settings
```

<a name="algolia-identifying-users"></a>
#### 辨識使用者

Scout 允許您在使用 Algolia 時自動辨識使用者。在 Algolia 的控制面板中查看搜尋分析時，將已通過認證的使用者與搜尋操作關聯會非常有幫助。您可以透過在應用程式的 `.env` 檔案中將 `SCOUT_IDENTIFY` 環境變數定義為 `true` 來啟用使用者辨識：

```ini
SCOUT_IDENTIFY=true
```

啟用此功能還會將請求的 IP 位址以及您已認證使用者的主要識別碼傳送給 Algolia，以便將這些資料與使用者發出的任何搜尋請求進行關聯。

<a name="meilisearch-configuration"></a>
### Meilisearch

<a name="meilisearch-index-settings"></a>
#### 索引設定

Meilisearch 需要您預先定義索引搜尋設定，例如可過濾屬性 (filterable attributes)、可排序屬性 (sortable attributes) 以及[其他支援的設定欄位](https://docs.meilisearch.com/reference/api/settings.html)。

可過濾屬性是指您在呼叫 Scout 的 `where` 方法時打算用來過濾的任何屬性，而可排序屬性是指您在呼叫 Scout 的 `orderBy` 方法時打算用來排序的任何屬性。要定義您的索引設定，請調整應用程式 `scout` 設定檔中 `meilisearch` 設定條目的 `index-settings` 部分：

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

若給定索引所對應的 Model 支援軟刪除且已包含在 `index-settings` 陣列中，Scout 將會自動對該索引上的軟刪除 Model 提供過濾支援。若您沒有其他要為軟刪除 Model 索引定義的可過濾或可排序屬性，可以簡單地為該 Model 在 `index-settings` 陣列中新增一個空項目：

```php
'index-settings' => [
    Flight::class => []
],
```

在設定好應用程式的索引設定後，您必須執行 `scout:sync-index-settings` Artisan 指令。此指令會將您目前設定的索引設定通知 Meilisearch。為了方便起見，您可以將此指令作為部署流程的一部分：

```shell
php artisan scout:sync-index-settings
```

<a name="meilisearch-data-types"></a>
#### 可搜尋資料型別

Meilisearch 僅會對正確型別的資料執行過濾操作（例如 `>`、`<` 等）。在自訂可搜尋資料時，您應確保數值已被轉型 (cast) 為正確的型別：

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

使用 Typesense 時，您的可搜尋 Model 必須定義一個 `toSearchableArray` 方法，將 Model 的主鍵轉型為字串，並將建立日期轉型為 UNIX 時間戳記：

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

您還應該在應用程式的 `config/scout.php` 檔案中定義 Typesense 集合架構（Collection schema）。集合架構描述了可透過 Typesense 搜尋的每個欄位資料型態。關於所有可用的架構選項詳細資訊，請參閱 [Typesense 文件](https://typesense.org/docs/latest/api/collections.html#schema-parameters)。

若您在定義 Typesense 集合架構後需要進行修改，您可以執行 `scout:flush` 與 `scout:import`，這會刪除所有現有的索引資料並重新建立架構。或者，您也可以使用 Typesense 的 API 來修改集合架構，而不需要移除任何已索引的資料。

如果您的可搜尋 Model 支援軟刪除，您應該在應用程式的 `config/scout.php` 設定檔中，為該 Model 對應的 Typesense 架構定義一個 `__soft_deleted` 欄位：

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

Typesense 允許您在透過 `options` 方法進行搜尋操作時，動態修改您的[搜尋參數](https://typesense.org/docs/latest/api/search.html#search-parameters)：

```php
use App\Models\Todo;

Todo::search('Groceries')->options([
    'query_by' => 'title, description'
])->get();
```

<a name="indexing"></a>
## 第三方引擎索引

> [!NOTE]
> 本節所描述的索引功能主要適用於使用第三方引擎（Algolia、Meilisearch 或 Typesense）時。資料庫引擎直接搜尋您的資料庫資料表，因此不需要手動進行索引管理。

<a name="batch-import"></a>
### 整批匯入

如果您是在既有專案中安裝 Scout，可能已經有需要匯入至索引中的資料庫紀錄。Scout 提供了一個 `scout:import` Artisan 指令，讓您可以將所有既有紀錄匯入至搜尋索引中：

```shell
php artisan scout:import "App\Models\Post"
```

`scout:queue-import` 指令可用於透過[佇列任務](/docs/{{version}}/queues)匯入您所有的既有紀錄：

```shell
php artisan scout:queue-import "App\Models\Post" --chunk=500
```

`flush` 指令可用於從搜尋索引中移除某個 Model 的所有紀錄：

```shell
php artisan scout:flush "App\Models\Post"
```

<a name="modifying-the-import-query"></a>
#### 修改匯入查詢

如果您想要修改整批匯入時用於取得所有 Model 的查詢，可以在 Model 上定義 `makeAllSearchableUsing` 方法。這是在匯入 Model 之前新增任何必要預先載入關聯的好地方：

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
> 當使用佇列整批匯入 Model 時，可能不適用 `makeAllSearchableUsing` 方法。因為當任務處理 Model 集合時，關聯是[不會被恢復](/docs/{{version}}/queues#handling-relationships)的。

<a name="adding-records"></a>
### 新增紀錄

當您將 `Laravel\Scout\Searchable` Trait 新增至 Model 後，只需 `save` 或 `create` Model 實例，它就會自動被新增至您的搜尋索引中。如果您已將 Scout 設定為[使用佇列](#queueing)，此操作將由您的佇列 Worker 在背景執行：

```php
use App\Models\Order;

$order = new Order;

// ...

$order->save();
```

<a name="adding-records-via-query"></a>
#### 透過查詢新增紀錄

如果您想透過 Eloquent 查詢將 Model 集合新增至搜尋索引，可以在 Eloquent 查詢鏈結 `searchable` 方法。`searchable` 方法會[分塊處理查詢結果](/docs/{{version}}/eloquent#chunking-results)並將紀錄新增至搜尋索引中。同樣地，如果您已將 Scout 設定為使用佇列，所有的分塊都將由您的佇列 Worker 在背景匯入：

```php
use App\Models\Order;

Order::where('price', '>', 100)->searchable();
```

您也可以在 Eloquent 關聯實例上呼叫 `searchable` 方法：

```php
$user->orders()->searchable();
```

或者，如果您在記憶體中已經有一個 Eloquent Model 集合，您可以在該集合實例上呼叫 `searchable` 方法，將這些 Model 實例新增至對應的索引中：

```php
$orders->searchable();
```

> [!NOTE]
> `searchable` 方法可以被視為一種「Upsert (新增或更新)」操作。換句話說，如果該 Model 紀錄已經在索引中，它將會被更新。如果它不存在於搜尋索引中，則會被新增至索引。

<a name="updating-records"></a>
### 更新紀錄

若要更新可搜尋的 Model，您只需要更新 Model 實例的屬性並將 Model `save` 至資料庫即可。Scout 會自動將變更持久化儲存至您的搜尋索引中：

```php
use App\Models\Order;

$order = Order::find(1);

// Update the order...

$order->save();
```

您也可以在 Eloquent 查詢實例上呼叫 `searchable` 方法來更新 Model 集合。如果 Model 不存在於您的搜尋索引中，將會被建立：

```php
Order::where('price', '>', 100)->searchable();
```

如果您想更新關聯中所有 Model 的搜尋索引紀錄，可以在該關聯實例上呼叫 `searchable`：

```php
$user->orders()->searchable();
```

或者，如果您在記憶體中已經有一個 Eloquent Model 集合，可以在該集合實例上呼叫 `searchable` 方法，以更新對應索引中的 Model 實例：

```php
$orders->searchable();
```

<a name="modifying-records-before-importing"></a>
#### 匯入前修改紀錄

有時您可能需要在使 Model 變為可搜尋之前先準備好 Model 集合。例如，您可能想要預先載入關聯，以便有效率地將關聯資料新增至搜尋索引中。若要達到此目的，可以在對應的 Model 上定義 `makeSearchableUsing` 方法：

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
#### 條件式更新搜尋索引

預設情況下，無論修改了哪些屬性，Scout 都會重新為已更新的 Model 建立索引。如果您想要自訂此行為，可以在 Model 上定義 `searchIndexShouldBeUpdated` 方法：

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

要從索引中移除紀錄，您只需從資料庫中 `delete` 該 Model 即可。即使您使用的是[軟刪除](/docs/{{version}}/eloquent#soft-deleting) Model，也可以這樣操作：

```php
use App\Models\Order;

$order = Order::find(1);

$order->delete();
```

如果您不想在刪除紀錄前取得 Model，可以在 Eloquent 查詢實例上使用 `unsearchable` 方法：

```php
Order::where('price', '>', 100)->unsearchable();
```

如果您想移除關聯中所有 Model 的搜尋索引紀錄，可以在該關聯實例上呼叫 `unsearchable`：

```php
$user->orders()->unsearchable();
```

或者，如果您在記憶體中已經有一個 Eloquent Model 集合，可以在該集合實例上呼叫 `unsearchable` 方法，從對應的索引中移除這些 Model 實例：

```php
$orders->unsearchable();
```

要從對應的索引中移除該 Model 的所有紀錄，您可以呼叫 `removeAllFromSearch` 方法：

```php
Order::removeAllFromSearch();
```

<a name="pausing-indexing"></a>
### 暫停索引

有時您可能需要對 Model 執行一連串 Eloquent 操作，但又不希望將 Model 資料同步至搜尋索引。您可以使用 `withoutSyncingToSearch` 方法來達到此目的。該方法接收一個單一的 Closure（閉包），並會立即執行該 Closure。在 Closure 內發生的任何 Model 操作都不會同步至該 Model 的索引：

```php
use App\Models\Order;

Order::withoutSyncingToSearch(function () {
    // Perform model actions...
});
```

<a name="conditionally-searchable-model-instances"></a>
### 條件式可搜尋 Model 實例

有時候，您可能只需要在特定條件下將 Model 設為可搜尋。例如，假設您有一個 `App\Models\Post` Model，它可能處於兩種狀態之一："draft"（草稿）和 "published"（已發布）。您可能只希望允許搜尋「已發布」的文章。若要達成此目的，可以在您的 Model 上定義一個 `shouldBeSearchable` 方法：

```php
/**
 * Determine if the model should be searchable.
 */
public function shouldBeSearchable(): bool
{
    return $this->isPublished();
}
```

`shouldBeSearchable` 方法僅在透過 `save` 和 `create` 方法、查詢或關聯操作 Model 時才會套用。如果直接使用 `searchable` 方法將 Model 或集合設為可搜尋，則會覆蓋 `shouldBeSearchable` 方法的結果。

> [!WARNING]
> 使用 Scout 的 "database" 引擎時不適用 `shouldBeSearchable` 方法，因為所有可搜尋的資料總是儲存在資料庫中。若要在使用資料庫引擎時達成類似行為，您應該改用 [where 子句](#where-clauses)。

<a name="searching"></a>
## 搜尋

您可以使用 `search` 方法開始搜尋 Model。搜尋方法接收一個字串，該字串將用於搜尋您的 Model。接著，您應該在搜尋查詢後鏈結 `get` 方法，以取得符合給定搜尋查詢的 Eloquent Model：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->get();
```

由於 Scout 搜尋會回傳 Eloquent Model 集合，您甚至可以直接從路由或控制器回傳結果，這些結果會自動轉換為 JSON：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/search', function (Request $request) {
    return Order::search($request->search)->get();
});
```

如果您想在將原始搜尋結果轉換為 Eloquent Model 之前取得它們，可以使用 `raw` 方法：

```php
$orders = Order::search('Star Trek')->raw();
```


<a name="custom-indexes"></a>
#### 自訂索引

當使用第三方引擎進行搜尋時，搜尋查詢通常會在 Model 的 [searchableAs](#configuring-model-indexes) 方法所指定的索引上執行。不過，您可以使用 `within` 方法來指定要搜尋的自訂索引：

```php
$orders = Order::search('Star Trek')
    ->within('tv_shows_popularity_desc')
    ->get();
```


<a name="where-clauses"></a>
### Where 子句

Scout 允許您在搜尋查詢中加入 "where" 子句。例如，基本的相等檢查對於依擁有者 ID 限定搜尋查詢範圍非常有用：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->where('user_id', 1)->get();
```

您也可以使用 `=`, `!=`, `<`, `>`, `>=`, `<=` 比較運算子來建立更進階的查詢：

```php
Order::search('Star Trek')
  ->where('status', '=', 'completed')
  ->where('is_refunded', '!=', true)
  ->where('total_price', '>', 100)
  ->where('shipping_cost', '<', 20)
  ->where('discount_percent', '>=', 10)
  ->where('item_count', '<=', 5)
  ->get();
```

此外，`whereIn` 方法可用於驗證給定欄位的值是否包含在給定的陣列中：

```php
$orders = Order::search('Star Trek')->whereIn(
    'status', ['open', 'paid']
)->get();
```

`whereNotIn` 方法可驗證給定欄位的值是否不包含在給定的陣列中：

```php
$orders = Order::search('Star Trek')->whereNotIn(
    'status', ['closed']
)->get();
```

> [!WARNING]
> 若您的應用程式正在使用 Meilisearch，在使用 Scout 的 "where" 子句之前，您必須先設定應用程式的[可過濾屬性](#meilisearch-index-settings)。


<a name="customizing-the-eloquent-results-query"></a>
#### 自訂 Eloquent 結果查詢

在 Scout 從您應用程式的搜尋引擎取得符合的 Eloquent Model 列表後，會透過 Eloquent 根據主鍵取得所有符合的 Model。您可以透過呼叫 `query` 方法來自訂此查詢。`query` 方法接收一個閉包，該閉包將收到 Eloquent 查詢建構器實例作為引數：

```php
use App\Models\Order;
use Illuminate\Database\Eloquent\Builder;

$orders = Order::search('Star Trek')
    ->query(fn (Builder $query) => $query->with('invoices'))
    ->get();
```

當使用第三方引擎時，此回呼會在從搜尋引擎取得相關 Model 之後才被調用，因此它不應該用於「過濾」結果 — 請改用 [Scout where 子句](#where-clauses)。不過，當使用資料庫引擎時，`query` 方法的約束條件會直接套用到資料庫查詢，因此您也可以將其用於過濾。


<a name="pagination"></a>
### 分頁

除了取得 Model 集合之外，您還可以使用 `paginate` 方法對搜尋結果進行分頁。此方法將回傳一個 `Illuminate\Pagination\LengthAwarePaginator` 實例，就像您對[傳統的 Eloquent 查詢進行分頁](/docs/{{version}}/pagination)一樣：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->paginate();
```

您可以將數量作為第一個引數傳遞給 `paginate` 方法，來指定每頁要取得多少個 Model：

```php
$orders = Order::search('Star Trek')->paginate(15);
```

使用資料庫引擎時，您還可以使用 `simplePaginate` 方法。與取得符合紀錄總數以顯示頁碼的 `paginate` 不同，`simplePaginate` 僅確定當前頁面之外是否還有更多結果 — 這使得它在只需要「上一頁」和「下一頁」連結的大型資料集中更有效率：

```php
$orders = Order::search('Star Trek')->simplePaginate(15);
```

取得結果後，您可以如同對傳統 Eloquent 查詢進行分頁一樣，使用 [Blade](/docs/{{version}}/blade) 顯示結果並渲染頁面連結：

```html
<div class="container">
    @foreach ($orders as $order)
        {{ $order->price }}
    @endforeach
</div>

{{ $orders->links() }}
```

當然，如果您想以 JSON 形式取得分頁結果，可以直接從路由或控制器回傳分頁器實例：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    return Order::search($request->input('query'))->paginate(15);
});
```

> [!WARNING]
> 由於搜尋引擎無法感知 Eloquent Model 的全域範疇定義，因此在使用 Scout 分頁的應用程式中，不應使用全域範疇。或者，您應該在透過 Scout 搜尋時重新建立全域範疇的約束條件。


<a name="soft-deleting"></a>
### 軟刪除

如果您的索引 Model 支援[軟刪除](/docs/{{version}}/eloquent#soft-deleting)，且您需要搜尋已軟刪除的 Model，請將 `config/scout.php` 設定檔中的 `soft_delete` 選項設定為 `true`：

```php
'soft_delete' => true,
```

當此設定選項為 `true` 時，Scout 不會從搜尋索引中移除已軟刪除的 Model。相反地，它會在索引紀錄上設定一個隱藏的 `__soft_deleted` 屬性。接著，您可以使用 `withTrashed` 或 `onlyTrashed` 方法在搜尋時取得已軟刪除的紀錄：

```php
use App\Models\Order;

// Include trashed records when retrieving results...
$orders = Order::search('Star Trek')->withTrashed()->get();

// Only include trashed records when retrieving results...
$orders = Order::search('Star Trek')->onlyTrashed()->get();
```

> [!NOTE]
> 當使用 `forceDelete` 永久刪除軟刪除的 Model 時，Scout 會自動將其從搜尋索引中移除。


<a name="customizing-engine-searches"></a>
### 自訂引擎搜尋

如果您需要對引擎的搜尋行為進行進階自訂，可以傳遞一個閉包作為 `search` 方法的第二個引數。例如，您可以使用此回呼在搜尋查詢傳遞給 Algolia 之前，將地理位置資料加入至搜尋選項中：

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
## 自訂引擎


<a name="writing-the-engine"></a>
#### 撰寫引擎

如果內建的 Scout 搜尋引擎無法滿足您的需求，您可以撰寫自己的自訂引擎並將其註冊到 Scout。您的引擎應該繼承 `Laravel\Scout\Engines\Engine` 抽象類別。這個抽象類別包含八個您的自訂引擎必須實作的方法：

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

您可能會發現參考 `Laravel\Scout\Engines\AlgoliaEngine` 類別中這些方法的實作會很有幫助。該類別將為您提供一個良好的起點，以瞭解如何在您自己的引擎中實作每個方法。


<a name="registering-the-engine"></a>
#### 註冊引擎

撰寫好自訂引擎後，您可以使用 Scout 引擎管理者的 `extend` 方法將其註冊到 Scout 中。Scout 的引擎管理者可以從 Laravel 服務容器中解析。您應該在 `App\Providers\AppServiceProvider` 類別或您應用程式使用的任何其他服務提供者(Service Providers)的 `boot` 方法中呼叫 `extend` 方法：

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

註冊引擎後，您可以在應用程式的 `config/scout.php` 設定檔中將其指定為預設的 Scout `driver`：

```php
'driver' => 'mysql',
```