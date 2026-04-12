# Laravel Scout

- [簡介](#introduction)
- [安裝](#installation)
    - [佇列化](#queueing)
- [驅動程式先決條件](#driver-prerequisites)
- [設定](#configuration)
    - [設定可搜尋資料](#configuring-searchable-data)
- [資料庫 / 集合引擎](#database-and-collection-engines)
    - [資料庫引擎](#database-engine)
    - [集合引擎](#collection-engine)
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
    - [條件式可搜尋模型實例](#conditionally-searchable-model-instances)
- [搜尋](#searching)
    - [Where 子句](#where-clauses)
    - [分頁](#pagination)
    - [軟刪除](#soft-deleting)
    - [自定義引擎搜尋](#customizing-engine-searches)
- [自定義引擎](#custom-engines)

<a name="introduction"></a>
## 簡介

[Laravel Scout](https://github.com/laravel/scout) 提供了一個簡單且基於驅動程式的解決方案，可為您的 [Eloquent 模型](/docs/{{version}}/eloquent) 增加全文搜尋功能。透過使用模型觀察者，Scout 會自動將您的搜尋索引與 Eloquent 紀錄保持同步。

Scout 內建了一個 `database` 引擎，該引擎使用 MySQL / PostgreSQL 的全文索引和 `LIKE` 子句來搜尋您現有的資料庫 —— 不需要外部服務。對於大多數應用程式來說，這就足夠了。若要概覽 Laravel 中所有可用的搜尋選項，請參閱 [搜尋文件](/docs/{{version}}/search)。

當您需要容錯搜尋、分面過濾 (faceted filtering) 或大規模地理搜尋等功能時，Scout 還包含 [Algolia](https://www.algolia.com/)、[Meilisearch](https://www.meilisearch.com) 和 [Typesense](https://typesense.org) 的驅動程式。此外，還提供了一個「集合 (collection)」驅動程式可用於本地開發，您也可以自由地編寫 [自定義引擎](#custom-engines)。


<a name="installation"></a>
## 安裝

首先，透過 Composer 套件管理工具安裝 Scout：

```shell
composer require laravel/scout
```

安裝 Scout 後，您應該使用 `vendor:publish` Artisan 命令來發布 Scout 設定檔。此命令會將 `scout.php` 設定檔發布到應用程式的 `config` 目錄中：

```shell
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

最後，將 `Laravel\Scout\Searchable` trait 添加到您想要使其可搜尋的模型中。此 trait 會註冊一個模型觀察者，自動將模型與您的搜尋驅動程式保持同步：

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

當使用非 `database` 或 `collection` 引擎時，建議您在開始使用此函式庫之前先配置 [佇列驅動程式](/docs/{{version}}/queues)。執行佇列工作者將允許 Scout 將所有將模型資訊同步到搜尋索引的操作放入佇列中，從而為您的應用程式網頁介面提供更短的回應時間。

配置好佇列驅動程式後，將 `config/scout.php` 設定檔中的 `queue` 選項值設定為 `true`：

```php
'queue' => true,
```

即使 `queue` 選項設定為 `false`，請記得某些 Scout 驅動程式（如 Algolia 和 Meilisearch）始終非同步地索引紀錄。換句話說，即使索引操作已在您的 Laravel 應用程式中完成，搜尋引擎本身可能不會立即反映新增加或更新的紀錄。

若要指定 Scout 作業所使用的連線和佇列，您可以將 `queue` 設定選項定義為陣列：

```php
'queue' => [
    'connection' => 'redis',
    'queue' => 'scout'
],
```

當然，如果您自定義了 Scout 作業使用的連線和佇列，您應該執行一個佇列工作者來處理該連線和佇列上的作業：

```shell
php artisan queue:work redis --queue=scout
```


<a name="driver-prerequisites"></a>
## 驅動程式先決條件


<a name="algolia"></a>
### Algolia

使用 Algolia 驅動程式時，您應該在 `config/scout.php` 設定檔中配置您的 Algolia `id` 和 `secret` 憑證。憑證配置完成後，您還需要透過 Composer 套件管理工具安裝 Algolia PHP SDK：

```shell
composer require algolia/algoliasearch-client-php
```


<a name="meilisearch"></a>
### Meilisearch

[Meilisearch](https://www.meilisearch.com) 是一個快速的開源搜尋引擎。如果您不確定如何在本地機器上安裝 Meilisearch，可以使用 [Laravel Sail](/docs/{{version}}/sail#meilisearch)，這是 Laravel 官方支援的 Docker 開發環境。

使用 Meilisearch 驅動程式時，您需要透過 Composer 套件管理工具安裝 Meilisearch PHP SDK：

```shell
composer require meilisearch/meilisearch-php http-interop/http-factory-guzzle
```

然後，在應用程式的 `.env` 檔案中設定 `SCOUT_DRIVER` 環境變數以及您的 Meilisearch `host` 和 `key` 憑證：

```ini
SCOUT_DRIVER=meilisearch
MEILISEARCH_HOST=http://127.0.0.1:7700
MEILISEARCH_KEY=masterKey
```

有關 Meilisearch 的更多資訊，請參閱 [Meilisearch 文件](https://docs.meilisearch.com/learn/getting_started/quick_start.html)。

此外，您應該透過查閱 [Meilisearch 關於二進位相容性的文件](https://github.com/meilisearch/meilisearch-php#-compatibility-with-meilisearch)，確保安裝的 `meilisearch/meilisearch-php` 版本與您的 Meilisearch 二進位版本相容。

> [!WARNING]
> 在使用 Meilisearch 的應用程式中升級 Scout 時，您應該始終 [查看 Meilisearch 服務本身的任何額外重大變更](https://github.com/meilisearch/Meilisearch/releases)。


<a name="typesense"></a>
### Typesense

[Typesense](https://typesense.org) 是一個極速的開源搜尋引擎，支援關鍵字搜尋、語義搜尋、地理搜尋和向量搜尋。

您可以 [自行託管](https://typesense.org/docs/guide/install-typesense.html#option-2-local-machine-self-hosting) Typesense 或使用 [Typesense Cloud](https://cloud.typesense.org)。

要開始將 Typesense 與 Scout 搭配使用，請透過 Composer 套件管理工具安裝 Typesense PHP SDK：

```shell
composer require typesense/typesense-php
```

然後，在應用程式的 .env 檔案中設定 `SCOUT_DRIVER` 環境變數以及您的 Typesense 主機和 API 金鑰憑證：

```ini
SCOUT_DRIVER=typesense
TYPESENSE_API_KEY=masterKey
TYPESENSE_HOST=localhost
```

如果您使用 [Laravel Sail](/docs/{{version}}/sail)，您可能需要調整 `TYPESENSE_HOST` 環境變數以匹配 Docker 容器名稱。您也可以選擇性地指定安裝的連接埠、路徑和協定：

```ini
TYPESENSE_PORT=8108
TYPESENSE_PATH=
TYPESENSE_PROTOCOL=http
```

您的 Typesense 集合的額外設定和結構定義 (schema definitions) 可以在應用程式的 `config/scout.php` 設定檔中找到。有關 Typesense 的更多資訊，請參閱 [Typesense 文件](https://typesense.org/docs/guide/#quick-start)。


<a name="configuration"></a>
## 設定


<a name="configuring-searchable-data"></a>
### 設定可搜尋資料

預設情況下，指定模型的整個 `toArray` 形式將被持久化到其搜尋索引中。如果您想自定義同步到搜尋索引的資料，可以在模型上覆寫 `toSearchableArray` 方法：

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

搜尋時，Scout 通常會使用應用程式 `scout` 設定檔中指定的預設搜尋引擎。然而，可以透過在模型上覆寫 `searchableUsing` 方法來更改特定模型的搜尋引擎：

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
> 資料庫引擎目前支援 MySQL 和 PostgreSQL，兩者都提供了快速的全文欄位索引支援。

`database` 引擎使用 MySQL / PostgreSQL 全文索引和 `LIKE` 子句來直接搜尋您現有的資料庫 —— 不需要外部服務。對於大多數應用程式來說，這就是您所需的一切。

要使用資料庫引擎，請將 `SCOUT_DRIVER` 環境變數設定為 `database`：

```ini
SCOUT_DRIVER=database
```

設定完成後，您可以[定義可搜尋資料](#configuring-searchable-data)並開始對您的模型[執行搜尋查詢](#searching)。與第三方引擎不同，資料庫引擎不需要獨立的索引步驟 —— 它直接搜尋您的資料庫資料表。


#### 自定義資料庫搜尋策略

預設情況下，資料庫引擎將對每個您[設定為可搜尋](#configuring-searchable-data)的模型屬性執行 `LIKE` 查詢。然而，您可以為特定欄位指定更高效的搜尋策略。`SearchUsingFullText` 屬性將針對該欄位使用資料庫的全文索引，而 `SearchUsingPrefix` 則僅匹配字串的開頭 (`example%`)，而不是在整個字串中搜尋 (`%example%`)。

要定義此行為，請為模型的 `toSearchableArray` 方法指定 PHP 屬性。任何沒有指定屬性的欄位將繼續使用預設的 `LIKE` 策略：

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
> 在指定欄位應使用全文查詢限制之前，請確保該欄位已分配[全文索引](/docs/{{version}}/migrations#available-index-types)。


<a name="collection-engine"></a>
### 集合引擎

「集合」引擎旨在用於快速原型開發、極小規模的資料集（幾百筆紀錄）或執行測試。它會從資料庫中擷取所有可能的紀錄，並使用 Laravel 的 `Str::is` 輔助函式在 PHP 中對其進行篩選，因此不需要任何索引或資料庫特定的功能。對於任何超出簡單使用案例的情況，您應該改用[資料庫引擎](#database-engine)。

要使用集合引擎，您可以簡單地將 `SCOUT_DRIVER` 環境變數的值設定為 `collection`，或者直接在應用程式的 `scout` 設定檔中指定 `collection` 驅動程式：

```ini
SCOUT_DRIVER=collection
```

一旦您將集合驅動程式指定為首選驅動程式，您就可以開始對您的模型[執行搜尋查詢](#searching)。使用集合引擎時，不需要進行搜尋引擎索引（例如為 Algolia、Meilisearch 或 Typesense 索引填充資料所需的索引作業）。


#### 與資料庫引擎的區別

雖然資料庫引擎使用全文索引和 `LIKE` 子句來高效地尋找匹配的紀錄，但集合引擎則是將所有紀錄取出並在 PHP 中進行篩選。集合引擎是最具可移植性的選項，因為它適用於 Laravel 支援的所有關聯式資料庫（包括 SQLite 和 SQL Server）；然而，它的效率明顯低於資料庫引擎，不應在大型資料集上使用。

<a name="third-party-engine-configuration"></a>
## 第三方引擎設定

以下設定選項僅在使用了第三方搜尋引擎（如 Algolia、Meilisearch 或 Typesense）時才適用。如果您使用的是 [資料庫引擎](#database-engine)，可以跳過此章節。


<a name="configuring-model-indexes"></a>
### 設定模型索引

當使用第三方引擎時，每個 Eloquent 模型會與一個特定的搜尋「索引」同步，該索引包含了該模型所有可搜尋的紀錄。預設情況下，每個模型會被持久化到一個與模型典型的「資料表」名稱相匹配的索引中。通常這是模型名稱的複數形式；不過，您可以透過在模型中覆寫 `searchableAs` 方法來自定義模型的索引：

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
> 當使用資料庫引擎時，`searchableAs` 方法沒有任何效果，因為它總是直接搜尋模型的資料庫資料表。


<a name="configuring-the-model-id"></a>
#### 設定模型 ID

預設情況下，Scout 會使用模型的主鍵作為儲存在搜尋索引中的模型唯一 ID / 鍵。如果您在第三方引擎中使用時需要自定義此行為，可以覆寫模型上的 `getScoutKey` 與 `getScoutKeyName` 方法：

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
> 當使用資料庫引擎時，`getScoutKey` 與 `getScoutKeyName` 方法沒有任何效果，因為它總是使用模型的主鍵。


<a name="algolia-configuration"></a>
### Algolia


<a name="algolia-index-settings"></a>
#### 索引設定

有時您可能想要在 Algolia 索引中設定額外的選項。雖然您可以透過 Algolia UI 管理這些設定，但直接從應用程式的 `config/scout.php` 設定檔中管理索引配置的期望狀態有時會更有效率。

這種方法讓您可以透過應用程式的自動化部署管線來部署這些設定，避免手動配置並確保多個環境之間的一致性。您可以設定可篩選屬性 (filterable attributes)、排序 (ranking)、分面 (faceting) 或 [任何其他支援的設定](https://www.algolia.com/doc/rest-api/search/#tag/Indices/operation/setSettings)。

首先，在應用程式的 `config/scout.php` 設定檔中為每個索引添加設定：

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

如果某個索引對應的模型是可軟刪除的，且被包含在 `index-settings` 陣列中，Scout 會自動為該索引上的軟刪除模型加入分面搜尋 (faceting) 的支援。如果您沒有其他需要為可軟刪除模型索引定義的分面屬性，只需為該模型在 `index-settings` 陣列中添加一個空項目即可：

```php
'index-settings' => [
    Flight::class => []
],
```

設定好應用程式的索引選項後，您必須執行 `scout:sync-index-settings` Artisan 指令。此指令會將您目前配置的索引設定通知 Algolia。為了方便起見，您可能希望將此指令納入您的部署流程中：

```shell
php artisan scout:sync-index-settings
```


<a name="algolia-identifying-users"></a>
#### 識別使用者

當使用 Algolia 時，Scout 允許您自動識別使用者。將認證使用者與搜尋操作關聯，在 Algolia 的儀表板中查看搜尋分析時會非常有幫助。您可以在應用程式的 `.env` 檔案中將 `SCOUT_IDENTIFY` 環境變數定義為 `true` 來啟用使用者識別：

```ini
SCOUT_IDENTIFY=true
```

啟用此功能還會將請求的 IP 位址以及認證使用者的主識別碼傳遞給 Algolia，以便將這些數據與使用者發出的任何搜尋請求關聯起來。


<a name="meilisearch-configuration"></a>
### Meilisearch


<a name="meilisearch-index-settings"></a>
#### 索引設定

Meilisearch 要求您預先定義索引搜尋設定，例如可篩選屬性 (filterable attributes)、可排序屬性 (sortable attributes) 以及 [其他支援的設定欄位](https://docs.meilisearch.com/reference/api/settings.html)。

可篩選屬性是指您在呼叫 Scout 的 `where` 方法時計畫進行篩選的任何屬性，而可排序屬性則是您在呼叫 Scout 的 `orderBy` 方法時計畫進行排序的任何屬性。要定義您的索引設定，請調整應用程式 `scout` 設定檔中 `meilisearch` 設定項目的 `index-settings` 部分：

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

如果某個索引對應的模型是可軟刪除的，且被包含在 `index-settings` 陣列中，Scout 會自動為該索引上的軟刪除模型加入篩選 (filtering) 的支援。如果您沒有其他需要為可軟刪除模型索引定義的可篩選或可排序屬性，只需為該模型在 `index-settings` 陣列中添加一個空項目即可：

```php
'index-settings' => [
    Flight::class => []
],
```

設定好應用程式的索引選項後，您必須執行 `scout:sync-index-settings` Artisan 指令。此指令會將您目前配置的索引設定通知 Meilisearch。為了方便起見，您可能希望將此指令納入您的部署流程中：

```shell
php artisan scout:sync-index-settings
```


<a name="meilisearch-data-types"></a>
#### 可搜尋資料型別

Meilisearch 僅會在正確類型的數據上執行篩選操作（如 `>`、`<` 等）。在自定義可搜尋資料時，您應確保數值被轉換為正確的型別：

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

當使用 Typesense 時，您的可搜尋模型必須定義一個 `toSearchableArray` 方法，將模型的主鍵轉換為字串，以及將建立日期轉換為 UNIX 時間戳記：

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

您還應該在應用程式的 `config/scout.php` 檔案中定義 Typesense 的集合結構 (collection schemas)。集合結構描述了透過 Typesense 搜尋的每個欄位之資料型別。有關所有可用結構選項的更多資訊，請參閱 [Typesense 文件](https://typesense.org/docs/latest/api/collections.html#schema-parameters)。

如果您需要在定義後更改 Typesense 集合的結構，您可以執行 `scout:flush` 和 `scout:import`，這將刪除所有現有的索引資料並重建結構。或者，您可以使用 Typesense 的 API 來修改集合結構，而無需移除任何索引資料。

如果您的可搜尋模型支援軟刪除，您應該在應用程式 `config/scout.php` 設定檔中，在模型對應的 Typesense 結構中定義一個 `__soft_deleted` 欄位：

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

Typesense 允許您在執行搜尋操作時，透過 `options` 方法動態修改您的 [搜尋參數](https://typesense.org/docs/latest/api/search.html#search-parameters)：

```php
use App\Models\Todo;

Todo::search('Groceries')->options([
    'query_by' => 'title, description'
])->get();
```

<a name="indexing"></a>
## 第三方引擎索引

> [!NOTE]
> 本節描述的索引功能主要適用於使用第三方引擎（Algolia, Meilisearch 或 Typesense）時。資料庫引擎會直接搜尋您的資料庫資料表，因此不需要手動管理索引。


<a name="batch-import"></a>
### 批次匯入

如果您將 Scout 安裝到現有的專案中，您可能已經有一些需要匯入到索引中的資料庫紀錄。Scout 提供了一個 `scout:import` Artisan 指令，可用於將所有現有紀錄匯入到您的搜尋索引中：

```shell
php artisan scout:import "App\Models\Post"
```

可以使用 `scout:queue-import` 指令透過 [佇列作業](/docs/{{version}}/queues) 匯入所有現有紀錄：

```shell
php artisan scout:queue-import "App\Models\Post" --chunk=500
```

可以使用 `flush` 指令從搜尋索引中移除某個模型的所有紀錄：

```shell
php artisan scout:flush "App\Models\Post"
```


<a name="modifying-the-import-query"></a>
#### 修改匯入查詢

如果您想要修改用於批次匯入時擷取所有模型的查詢，可以在模型上定義 `makeAllSearchableUsing` 方法。這是添加在匯入模型前可能需要的任何預先載入 (eager loading) 關聯的絕佳位置：

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
> 當使用佇列批次匯入模型時，`makeAllSearchableUsing` 方法可能不適用。因為當模型集合由作業處理時，關聯[不會被恢復](/docs/{{version}}/queues#handling-relationships)。


<a name="adding-records"></a>
### 新增紀錄

一旦您將 `Laravel\Scout\Searchable` trait 添加到模型中，您只需要 `save` 或 `create` 一個模型實例，它就會自動被添加到您的搜尋索引中。如果您已將 Scout 設定為 [使用佇列](#queueing)，此操作將由您的佇列工作者 (queue worker) 在背景執行：

```php
use App\Models\Order;

$order = new Order;

// ...

$order->save();
```


<a name="adding-records-via-query"></a>
#### 透過查詢新增紀錄

如果您想要透過 Eloquent 查詢將一組模型添加到搜尋索引中，可以在 Eloquent 查詢後鏈結 `searchable` 方法。`searchable` 方法會將查詢的[結果分塊 (chunk)](/docs/{{version}}/eloquent#chunking-results) 並將紀錄添加到搜尋索引中。同樣地，如果您已將 Scout 設定為使用佇列，所有分塊都將由佇列工作者在背景匯入：

```php
use App\Models\Order;

Order::where('price', '>', 100)->searchable();
```

您也可以在 Eloquent 關聯實例上呼叫 `searchable` 方法：

```php
$user->orders()->searchable();
```

或者，如果您記憶體中已經有一組 Eloquent 模型集合，可以在集合實例上呼叫 `searchable` 方法，將模型實例添加到其對應的索引中：

```php
$orders->searchable();
```

> [!NOTE]
> `searchable` 方法可以被視為一種 "upsert" (更新或插入) 操作。換句話說，如果模型紀錄已存在於索引中，它將被更新；如果不存在於搜尋索引中，它將被添加到索引中。


<a name="updating-records"></a>
### 更新紀錄

要更新可搜尋模型，您只需要更新模型實例的屬性並將模型 `save` 到資料庫中。Scout 會自動將變更持久化到您的搜尋索引中：

```php
use App\Models\Order;

$order = Order::find(1);

// Update the order...

$order->save();
```

您也可以在 Eloquent 查詢實例上呼叫 `searchable` 方法來更新一組模型。如果模型不存在於搜尋索引中，它們將被建立：

```php
Order::where('price', '>', 100)->searchable();
```

如果您想要更新關聯中所有模型的搜尋索引紀錄，可以在關聯實例上呼叫 `searchable`：

```php
$user->orders()->searchable();
```

或者，如果您記憶體中已經有一組 Eloquent 模型集合，可以在集合實例上呼叫 `searchable` 方法，以更新其對應索引中的模型實例：

```php
$orders->searchable();
```


<a name="modifying-records-before-importing"></a>
#### 在匯入前修改紀錄

有時您可能需要在模型被設為可搜尋之前準備模型集合。例如，您可能想要預先載入一個關聯，以便將關聯資料有效地添加到搜尋索引中。為了實現這一點，請在對應的模型上定義 `makeSearchableUsing` 方法：

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

預設情況下，無論修改了哪些屬性，Scout 都會對更新的模型重新建立索引。如果您想要自定義此行為，可以在模型上定義 `searchIndexShouldBeUpdated` 方法：

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

要從索引中移除紀錄，您只需將模型從資料庫中 `delete` 即可。即使您使用的是 [軟刪除](/docs/{{version}}/eloquent#soft-deleting) 模型，也可以這樣做：

```php
use App\Models\Order;

$order = Order::find(1);

$order->delete();
```

如果您不想在刪除紀錄前先擷取模型，可以在 Eloquent 查詢實例上使用 `unsearchable` 方法：

```php
Order::where('price', '>', 100)->unsearchable();
```

如果您想要移除關聯中所有模型的搜尋索引紀錄，可以在關聯實例上呼叫 `unsearchable`：

```php
$user->orders()->unsearchable();
```

或者，如果您記憶體中已經有一組 Eloquent 模型集合，可以在集合實例上呼叫 `unsearchable` 方法，將模型實例從其對應的索引中移除：

```php
$orders->unsearchable();
```

要從對應索引中移除所有模型紀錄，可以呼叫 `removeAllFromSearch` 方法：

```php
Order::removeAllFromSearch();
```


<a name="pausing-indexing"></a>
### 暫停索引

有時您可能需要對模型執行一批 Eloquent 操作，而不需要將模型資料同步到搜尋索引中。您可以使用 `withoutSyncingToSearch` 方法來實現。此方法接受一個閉包 (closure)，該閉包將立即執行。在閉包中發生的任何模型操作都不會同步到模型的索引中：

```php
use App\Models\Order;

Order::withoutSyncingToSearch(function () {
    // Perform model actions...
});
```

<a name="conditionally-searchable-model-instances"></a>
### 條件式可搜尋模型實例

有時您可能需要僅在特定條件下才讓模型可被搜尋。例如，假設您有一個 `App\Models\Post` 模型，它可能處於兩種狀態之一：「草稿」和「已發佈」。您可能只想允許「已發佈」的貼文可被搜尋。為了實現這一點，您可以在模型上定義 `shouldBeSearchable` 方法：

```php
/**
 * Determine if the model should be searchable.
 */
public function shouldBeSearchable(): bool
{
    return $this->isPublished();
}
```

`shouldBeSearchable` 方法僅在透過 `save` 與 `create` 方法、查詢或關聯來操作模型時才會被套用。直接使用 `searchable` 方法使模型或集合可被搜尋時，將會覆蓋 `shouldBeSearchable` 方法的結果。

> [!WARNING]
> `shouldBeSearchable` 方法在使用了 Scout 的 "database" 引擎時並不適用，因為所有可搜尋的資料一向儲存在資料庫中。若要在使用資料庫引擎時達成類似的行為，您應該改用 [Where 子句](#where-clauses)。

<a name="searching"></a>
## 搜尋

您可以使用 `search` 方法開始搜尋模型。該搜尋方法接受單一字串，用於搜尋您的模型。接著您應該在搜尋查詢後串接 `get` 方法，以取得與給定搜尋查詢相匹配的 Eloquent 模型：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->get();
```

由於 Scout 搜尋會回傳一個 Eloquent 模型的集合，您甚至可以直接從路由或控制器回傳結果，它們將會自動被轉換為 JSON：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/search', function (Request $request) {
    return Order::search($request->search)->get();
});
```

如果您想在結果被轉換為 Eloquent 模型之前取得原始搜尋結果，可以使用 `raw` 方法：

```php
$orders = Order::search('Star Trek')->raw();
```


<a name="custom-indexes"></a>
#### 自定義索引

在使用第三方引擎搜尋時，搜尋查詢通常會在模型 [searchableAs](#configuring-model-indexes) 方法所指定的索引上執行。然而，您可以使用 `within` 方法來指定要搜尋的自定義索引：

```php
$orders = Order::search('Star Trek')
    ->within('tv_shows_popularity_desc')
    ->get();
```


<a name="where-clauses"></a>
### Where 子句

Scout 允許您在搜尋查詢中加入 "where" 子句。例如，基本的等值檢查對於透過擁有者 ID 來限制搜尋查詢範圍非常有用：

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

`whereNotIn` 方法則驗證給定欄位的值是否不包含在給定的陣列中：

```php
$orders = Order::search('Star Trek')->whereNotIn(
    'status', ['closed']
)->get();
```

> [!WARNING]
> 如果您的應用程式使用 Meilisearch，在利用 Scout 的 "where" 子句之前，必須先設定應用程式的 [可篩選屬性](#meilisearch-index-settings)。


<a name="customizing-the-eloquent-results-query"></a>
#### 自定義 Eloquent 結果查詢

在 Scout 從應用程式的搜尋引擎取得匹配的 Eloquent 模型列表後，會使用 Eloquent 透過其主鍵來取得所有匹配的模型。您可以透過呼叫 `query` 方法來自定義此查詢。`query` 方法接受一個閉包，該閉包將接收 Eloquent 查詢建構器實例作為引數：

```php
use App\Models\Order;
use Illuminate\Database\Eloquent\Builder;

$orders = Order::search('Star Trek')
    ->query(fn (Builder $query) => $query->with('invoices'))
    ->get();
```

使用第三方引擎時，此回呼函數在相關模型已從搜尋引擎取出後才會被執行，因此不應將其用於「篩選」結果 —— 請改用 [Scout where 子句](#where-clauses)。然而，使用資料庫引擎時，`query` 方法的限制會直接套用到資料庫查詢中，因此您也可以將其用於篩選。


<a name="pagination"></a>
### 分頁

除了取得模型集合外，您還可以利用 `paginate` 方法對搜尋結果進行分頁。此方法將回傳一個 `Illuminate\Pagination\LengthAwarePaginator` 實例，就像您 [對傳統 Eloquent 查詢進行分頁](/docs/{{version}}/pagination) 一樣：

```php
use App\Models\Order;

$orders = Order::search('Star Trek')->paginate();
```

您可以透過將數量作為 `paginate` 方法的第一個引數，來指定每頁要取得多少個模型：

```php
$orders = Order::search('Star Trek')->paginate(15);
```

使用資料庫引擎時，您也可以使用 `simplePaginate` 方法。與 `paginate`（會取得匹配紀錄的總數以顯示頁碼）不同，`simplePaginate` 僅判斷目前頁面之外是否還有更多結果 —— 這對於僅需要「上一個」與「下一個」連結的大型資料集來說更有效率：

```php
$orders = Order::search('Star Trek')->simplePaginate(15);
```

取得結果後，您可以像對傳統 Eloquent 查詢分頁一樣，使用 [Blade](/docs/{{version}}/blade) 顯示結果並渲染分頁連結：

```html
<div class="container">
    @foreach ($orders as $order)
        {{ $order->price }}
    @endforeach
</div>

{{ $orders->links() }}
```

當然，如果您想將分頁結果以 JSON 形式取得，可以直接從路由或控制器回傳分頁器實例：

```php
use App\Models\Order;
use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    return Order::search($request->input('query'))->paginate(15);
});
```

> [!WARNING]
> 由於搜尋引擎無法得知 Eloquent 模型的全域範圍 (global scope) 定義，因此在利用 Scout 分頁的應用程式中，不應使用全域範圍。或者，您應該在透過 Scout 搜尋時重新建立全域範圍的限制。


<a name="soft-deleting"></a>
### 軟刪除

如果您的索引模型使用 [軟刪除](/docs/{{version}}/eloquent#soft-deleting) 且您需要搜尋被軟刪除的模型，請將 `config/scout.php` 設定檔中的 `soft_delete` 選項設定為 `true`：

```php
'soft_delete' => true,
```

當此設定選項為 `true` 時，Scout 將不會從搜尋索引中移除被軟刪除的模型。相反地，它會在索引紀錄上設定一個隱藏的 `__soft_deleted` 屬性。接著，您可以在搜尋時使用 `withTrashed` 或 `onlyTrashed` 方法來取得被軟刪除的紀錄：

```php
use App\Models\Order;

// Include trashed records when retrieving results...
$orders = Order::search('Star Trek')->withTrashed()->get();

// Only include trashed records when retrieving results...
$orders = Order::search('Star Trek')->onlyTrashed()->get();
```

> [!NOTE]
> 當使用 `forceDelete` 永久刪除被軟刪除的模型時，Scout 會自動將其從搜尋索引中移除。


<a name="customizing-engine-searches"></a>
### 自定義引擎搜尋

如果您需要對引擎的搜尋行為進行進階自定義，可以將閉包作為 `search` 方法的第二個引數。例如，您可以使用此回呼函數在搜尋查詢傳遞給 Algolia 之前，將地理位置資料加入到搜尋選項中：

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

如果內建的 Scout 搜尋引擎無法滿足您的需求，您可以撰寫自己的自定義引擎並將其註冊到 Scout。您的引擎應該繼承 `Laravel\Scout\Engines\Engine` 抽象類別。這個抽象類別包含八個您的自定義引擎必須實作的方法：

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

您可以參考 `Laravel\Scout\Engines\AlgoliaEngine` 類別中這些方法的實作方式。這個類別將為您提供一個很好的起點，讓您學習如何在自己的引擎中實作這些方法。


<a name="registering-the-engine"></a>
#### 註冊引擎

當您撰寫好自定義引擎後，可以使用 Scout 引擎管理器的 `extend` 方法將其註冊到 Scout。Scout 的引擎管理器可以從 Laravel 的服務容器中解析。您應該在 `App\Providers\AppServiceProvider` 類別或應用程式使用的任何其他服務提供者(Service Providers)的 `boot` 方法中呼叫 `extend` 方法：

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

引擎註冊完成後，您可以在應用程式的 `config/scout.php` 設定檔中將其指定為預設的 Scout `driver`：

```php
'driver' => 'mysql',
```