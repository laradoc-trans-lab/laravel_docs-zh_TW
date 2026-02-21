# Eloquent：入門

- [簡介](#introduction)
- [生成模型類別](#generating-model-classes)
- [Eloquent 模型慣例](#eloquent-model-conventions)
    - [資料表名稱](#table-names)
    - [主鍵](#primary-keys)
    - [UUID 與 ULID 鍵](#uuid-and-ulid-keys)
    - [時間戳記](#timestamps)
    - [資料庫連接](#database-connections)
    - [預設屬性值](#default-attribute-values)
    - [設定 Eloquent 嚴謹度](#configuring-eloquent-strictness)
- [檢索模型](#retrieving-models)
    - [集合](#collections)
    - [分塊處理結果](#chunking-results)
    - [使用 Lazy 集合分塊](#chunking-using-lazy-collections)
    - [遊標 (Cursors)](#cursors)
    - [進階子查詢](#advanced-subqueries)
- [檢索單一模型 / 聚合](#retrieving-single-models)
    - [檢索或建立模型](#retrieving-or-creating-models)
    - [檢索聚合資料](#retrieving-aggregates)
- [插入與更新模型](#inserting-and-updating-models)
    - [插入](#inserts)
    - [更新](#updates)
    - [批量賦值](#mass-assignment)
    - [更新或插入 (Upserts)](#upserts)
- [刪除模型](#deleting-models)
    - [軟刪除](#soft-deleting)
    - [查詢軟刪除模型](#querying-soft-deleted-models)
- [修剪模型](#pruning-models)
- [複製模型](#replicating-models)
- [查詢範圍](#query-scopes)
    - [全域範圍](#global-scopes)
    - [本地範圍](#local-scopes)
    - [待處理屬性](#pending-attributes)
- [比較模型](#comparing-models)
- [事件](#events)
    - [使用閉包](#events-using-closures)
    - [觀察器](#observers)
    - [靜音事件](#muting-events)

<a name="introduction"></a>
## 簡介

Laravel 包含了 Eloquent，這是一個物件關聯對映 (ORM)，讓你與資料庫的互動變得愉快。使用 Eloquent 時，每個資料表都有一個對應的「模型 (Model)」，用於與該資料表互動。除了從資料表中檢索記錄外，Eloquent 模型還允許你對資料表進行插入、更新和刪除記錄。

> [!NOTE]
> 在開始之前，請務必在應用程式的 `config/database.php` 設定檔中設定資料庫連接。有關設定資料庫的更多資訊，請參閱 [資料庫設定文件](/docs/{{version}}/database#configuration)。


<a name="generating-model-classes"></a>
## 生成模型類別

首先，讓我們建立一個 Eloquent 模型。模型通常存放在 `app\Models` 目錄中，並繼承 `Illuminate\Database\Eloquent\Model` 類別。你可以使用 `make:model` [Artisan 指令](/docs/{{version}}/artisan) 來生成新的模型：

```shell
php artisan make:model Flight
```

如果你想在生成模型的同時生成 [資料庫遷移](/docs/{{version}}/migrations)，可以使用 `--migration` 或 `-m` 選項：

```shell
php artisan make:model Flight --migration
```

在生成模型時，你也可以生成其他各種類型的類別，例如工廠 (factories)、填充器 (seeders)、策略 (policies)、控制器 (controllers) 和表單請求 (form requests)。此外，這些選項可以組合使用，一次建立多個類別：

```shell
# Generate a model and a FlightFactory class...
php artisan make:model Flight --factory
php artisan make:model Flight -f

# Generate a model and a FlightSeeder class...
php artisan make:model Flight --seed
php artisan make:model Flight -s

# Generate a model and a FlightController class...
php artisan make:model Flight --controller
php artisan make:model Flight -c

# Generate a model, FlightController resource class, and form request classes...
php artisan make:model Flight --controller --resource --requests
php artisan make:model Flight -crR

# Generate a model and a FlightPolicy class...
php artisan make:model Flight --policy

# Generate a model and a migration, factory, seeder, and controller...
php artisan make:model Flight -mfsc

# Shortcut to generate a model, migration, factory, seeder, policy, controller, and form requests...
php artisan make:model Flight --all
php artisan make:model Flight -a

# Generate a pivot model...
php artisan make:model Member --pivot
php artisan make:model Member -p
```


<a name="inspecting-models"></a>
#### 檢查模型

有時僅透過瀏覽程式碼很難確定模型所有可用的屬性和關聯。相反地，可以嘗試 `model:show` Artisan 指令，它提供了該模型所有屬性和關聯的便利概覽：

```shell
php artisan model:show Flight
```

<a name="eloquent-model-conventions"></a>
## Eloquent 模型慣例

由 `make:model` 指令生成的模型會被放置在 `app/Models` 目錄中。讓我們來檢視一個基礎的模型類別並討論一些 Eloquent 的核心慣例：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    // ...
}
```


<a name="table-names"></a>
### 資料表名稱

在看過上述範例後，您可能已經注意到我們並沒有告訴 Eloquent 哪個資料庫資料表對應於我們的 `Flight` 模型。依照慣例，除非明確指定了其他名稱，否則類別名稱的「蛇形命名 (snake case)」複數形式將被用作資料表名稱。因此，在此情況下，Eloquent 會假設 `Flight` 模型將記錄儲存在 `flights` 資料表中，而 `AirTrafficController` 模型則會將記錄儲存在 `air_traffic_controllers` 資料表中。

如果您模型對應的資料庫資料表不符合此慣例，您可以透過在模型上定義 `table` 屬性來手動指定模型的資料表名稱：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * The table associated with the model.
     *
     * @var string
     */
    protected $table = 'my_flights';
}
```


<a name="primary-keys"></a>
### 主鍵

Eloquent 也會假設每個模型對應的資料庫資料表都有一個名為 `id` 的主鍵欄位。如有必要，您可以在模型上定義一個受保護的 `$primaryKey` 屬性，以指定一個不同的欄位作為模型的主鍵：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * The primary key associated with the table.
     *
     * @var string
     */
    protected $primaryKey = 'flight_id';
}
```

此外，Eloquent 假設主鍵是一個遞增的整數值，這意味著 Eloquent 會自動將主鍵轉換為整數。如果您希望使用非遞增或非數值的主鍵，您必須在模型上定義一個公開的 `$incrementing` 屬性並將其設為 `false`：

```php
<?php

class Flight extends Model
{
    /**
     * Indicates if the model's ID is auto-incrementing.
     *
     * @var bool
     */
    public $incrementing = false;
}
```

如果您的模型主鍵不是整數，您應該在模型上定義一個受保護的 `$keyType` 屬性。此屬性的值應為 `string`：

```php
<?php

class Flight extends Model
{
    /**
     * The data type of the primary key ID.
     *
     * @var string
     */
    protected $keyType = 'string';
}
```


<a name="composite-primary-keys"></a>
#### 「複合」主鍵

Eloquent 要求每個模型至少具有一個可用作其主鍵的唯一識別「ID」。Eloquent 模型不支援「複合 (Composite)」主鍵。然而，除了資料表的唯一識別主鍵外，您可以自由地向資料庫資料表添加額外的多欄位唯一索引。


<a name="uuid-and-ulid-keys"></a>
### UUID 與 ULID 鍵

除了使用自動遞增整數作為您的 Eloquent 模型主鍵外，您也可以選擇使用 UUID。UUID 是長度為 36 個字元的通用唯一英數識別碼。

如果您想讓模型使用 UUID 鍵而不是自動遞增整數鍵，您可以在模型上使用 `Illuminate\Database\Eloquent\Concerns\HasUuids` trait。當然，您應該確保模型具有一個 [等同於 UUID 的主鍵欄位](/docs/{{version}}/migrations#column-method-uuid)：

```php
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;

class Article extends Model
{
    use HasUuids;

    // ...
}

$article = Article::create(['title' => 'Traveling to Europe']);

$article->id; // "8f8e8478-9035-4d23-b9a7-62f4d2612ce5"
```

預設情況下，`HasUuids` trait 將為您的模型生成 [「有序 (ordered)」UUID](/docs/{{version}}/strings#method-str-ordered-uuid)。這些 UUID 對於索引資料庫儲存更有效率，因為它們可以按字典順序進行排序。

您可以透過在模型上定義 `newUniqueId` 方法來覆蓋給定模型的 UUID 生成過程。此外，您可以透過在模型上定義 `uniqueIds` 方法來指定哪些欄位應該接收 UUID：

```php
use Ramsey\Uuid\Uuid;

/**
 * Generate a new UUID for the model.
 */
public function newUniqueId(): string
{
    return (string) Uuid::uuid4();
}

/**
 * Get the columns that should receive a unique identifier.
 *
 * @return array<int, string>
 */
public function uniqueIds(): array
{
    return ['id', 'discount_code'];
}
```

如果您願意，可以選擇使用「ULID」而不是 UUID。ULID 與 UUID 類似；然而，它們的長度僅為 26 個字元。與有序 UUID 一樣，ULID 可按字典順序排序，以便進行高效的資料庫索引。要使用 ULID，您應該在模型上使用 `Illuminate\Database\Eloquent\Concerns\HasUlids` trait。您還應該確保模型具有 [等同於 ULID 的主鍵欄位](/docs/{{version}}/migrations#column-method-ulid)：

```php
use Illuminate\Database\Eloquent\Concerns\HasUlids;
use Illuminate\Database\Eloquent\Model;

class Article extends Model
{
    use HasUlids;

    // ...
}

$article = Article::create(['title' => 'Traveling to Asia']);

$article->id; // "01gd4d3tgrrfqeda94gdbtdk5c"
```


<a name="timestamps"></a>
### 時間戳記

預設情況下，Eloquent 預期您的模型對應的資料庫資料表中存在 `created_at` 和 `updated_at` 欄位。當建立或更新模型時，Eloquent 會自動設定這些欄位的值。如果您不希望這些欄位由 Eloquent 自動管理，您應該在模型上定義一個值為 `false` 的 `$timestamps` 屬性：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * Indicates if the model should be timestamped.
     *
     * @var bool
     */
    public $timestamps = false;
}
```

如果您需要自訂模型時間戳記的格式，請在模型上設定 `$dateFormat` 屬性。此屬性決定了日期屬性在資料庫中的儲存方式，以及模型序列化為陣列或 JSON 時的格式：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * The storage format of the model's date columns.
     *
     * @var string
     */
    protected $dateFormat = 'U';
}
```

如果您需要自訂用於儲存時間戳記的欄位名稱，您可以在模型上定義 `CREATED_AT` 和 `UPDATED_AT` 常數：

```php
<?php

class Flight extends Model
{
    /**
     * The name of the "created at" column.
     *
     * @var string|null
     */
    public const CREATED_AT = 'creation_date';

    /**
     * The name of the "updated at" column.
     *
     * @var string|null
     */
    public const UPDATED_AT = 'updated_date';
}
```

如果您想在不修改模型的 `updated_at` 時間戳記的情況下執行模型操作，您可以在傳遞給 `withoutTimestamps` 方法的閉包中操作模型：

```php
Model::withoutTimestamps(fn () => $post->increment('reads'));
```


<a name="database-connections"></a>
### 資料庫連接

預設情況下，所有 Eloquent 模型都將使用為您的應用程式設定的預設資料庫連接。如果您想指定與特定模型互動時應使用的不同連接，您應該在模型上定義一個 `$connection` 屬性：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * The database connection that should be used by the model.
     *
     * @var string
     */
    protected $connection = 'mysql';
}
```

<a name="default-attribute-values"></a>
### 預設屬性值

預設情況下，新實例化的模型實例不會包含任何屬性值。如果您想為模型的某些屬性定義預設值，可以在模型上定義 `$attributes` 屬性。放置在 `$attributes` 陣列中的屬性值應該是原始的「可儲存」格式，就像剛從資料庫中讀取出來一樣：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * The model's default values for attributes.
     *
     * @var array
     */
    protected $attributes = [
        'options' => '[]',
        'delayed' => false,
    ];
}
```


<a name="configuring-eloquent-strictness"></a>
### 設定 Eloquent 嚴謹度

Laravel 提供了多種方法，讓您可以在各種情況下設定 Eloquent 的行為和「嚴謹度」。

首先，`preventLazyLoading` 方法接受一個選用的布林值參數，用以指示是否應防止延遲載入。例如，您可能希望僅在非正式環境中停用延遲載入，以便即使在正式環境程式碼中意外出現了延遲載入的關聯，您的正式環境仍能繼續正常運作。通常，此方法應在應用程式 `AppServiceProvider` 的 `boot` 方法中呼叫：

```php
use Illuminate\Database\Eloquent\Model;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Model::preventLazyLoading(! $this->app->isProduction());
}
```

此外，您可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填充不可填充的屬性時拋出例外。這有助於防止在本地開發期間，嘗試設定尚未新增至模型 `fillable` 陣列的屬性時發生非預期的錯誤：

```php
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

<a name="retrieving-models"></a>
## 檢索模型

一旦建立了模型及其[關聯的資料庫資料表](/docs/{{version}}/migrations#generating-migrations)，你就可以開始從資料庫檢索資料。你可以將每個 Eloquent 模型視為一個強大的[查詢產生器](/docs/{{version}}/queries)，讓你能夠流暢地查詢與該模型關聯的資料表。模型的 `all` 方法會從該模型關聯的資料庫資料表中檢索所有紀錄：

```php
use App\Models\Flight;

foreach (Flight::all() as $flight) {
    echo $flight->name;
}
```

<a name="building-queries"></a>
#### 建立查詢

Eloquent 的 `all` 方法會回傳該模型資料表中的所有結果。然而，由於每個 Eloquent 模型都充當[查詢產生器](/docs/{{version}}/queries)，因此你可以為查詢增加額外的約束條件，然後呼叫 `get` 方法來檢索結果：

```php
$flights = Flight::where('active', 1)
    ->orderBy('name')
    ->limit(10)
    ->get();
```

> [!NOTE]
> 由於 Eloquent 模型就是查詢產生器，你應該查看 Laravel 的[查詢產生器](/docs/{{version}}/queries)所提供的所有方法。在撰寫 Eloquent 查詢時，你可以使用其中的任何方法。

<a name="refreshing-models"></a>
#### 重新整理模型

如果你已經有一個從資料庫檢索到的 Eloquent 模型實例，你可以使用 `fresh` 和 `refresh` 方法來「重新整理」模型。`fresh` 方法會從資料庫重新檢索該模型。現有的模型實例將不會受到影響：

```php
$flight = Flight::where('number', 'FR 900')->first();

$freshFlight = $flight->fresh();
```

`refresh` 方法會使用來自資料庫的新資料重新建構 (Re-hydrate) 現有的模型。此外，所有已載入的關聯也都會被重新整理：

```php
$flight = Flight::where('number', 'FR 900')->first();

$flight->number = 'FR 456';

$flight->refresh();

$flight->number; // "FR 900"
```

<a name="collections"></a>
### 集合

正如我們所見，像 `all` 和 `get` 這樣的 Eloquent 方法會從資料庫檢索多條紀錄。然而，這些方法不會回傳單純的 PHP 陣列。相反地，會回傳一個 `Illuminate\Database\Eloquent\Collection` 的實例。

Eloquent 的 `Collection` 類別繼承自 Laravel 的基礎 `Illuminate\Support\Collection` 類別，該類別提供了[多種實用的方法](/docs/{{version}}/collections#available-methods)來與資料集合進行互動。例如，`reject` 方法可以用於根據呼叫的閉包結果從集合中移除模型：

```php
$flights = Flight::where('destination', 'Paris')->get();

$flights = $flights->reject(function (Flight $flight) {
    return $flight->cancelled;
});
```

除了 Laravel 基礎集合類別提供的方法外，Eloquent 集合類別還提供了[一些額外的方法](/docs/{{version}}/eloquent-collections#available-methods)，專門用於與 Eloquent 模型集合進行互動。

由於所有 Laravel 的集合都實作了 PHP 的可迭代 (iterable) 介面，因此你可以像處理陣列一樣對集合進行迴圈處理：

```php
foreach ($flights as $flight) {
    echo $flight->name;
}
```

<a name="chunking-results"></a>
### 分塊處理結果

如果你嘗試透過 `all` 或 `get` 方法載入數萬條 Eloquent 紀錄，你的應用程式可能會耗盡記憶體。與其使用這些方法，不如使用 `chunk` 方法更有效率地處理大量的模型。

`chunk` 方法會檢索 Eloquent 模型的一個子集，並將其傳遞給閉包進行處理。由於每次只會檢索目前的 Eloquent 模型分塊，因此在處理大量模型時，`chunk` 方法將顯著降低記憶體使用量：

```php
use App\Models\Flight;
use Illuminate\Database\Eloquent\Collection;

Flight::chunk(200, function (Collection $flights) {
    foreach ($flights as $flight) {
        // ...
    }
});
```

傳遞給 `chunk` 方法的第一個參數是你希望每個「分塊 (Chunk)」接收的紀錄數量。作為第二個參數傳遞的閉包將針對從資料庫檢索到的每個分塊進行呼叫。將會執行一次資料庫查詢來檢索傳遞給閉包的每個紀錄分塊。

如果你是根據某個欄位過濾 `chunk` 方法的結果，且在迭代結果時也會更新該欄位，則你應該使用 `chunkById` 方法。在這些情境下使用 `chunk` 方法可能會導致非預期且不一致的結果。在內部，`chunkById` 方法始終會檢索 `id` 欄位大於上一個分塊中最後一個模型的模型：

```php
Flight::where('departed', true)
    ->chunkById(200, function (Collection $flights) {
        $flights->each->update(['departed' => false]);
    }, column: 'id');
```

由於 `chunkById` 和 `lazyById` 方法會為正在執行的查詢添加它們自己的「where」條件，因此你通常應該在閉包內[邏輯分組](/docs/{{version}}/queries#logical-grouping)你自己的條件：

```php
Flight::where(function ($query) {
    $query->where('delayed', true)->orWhere('cancelled', true);
})->chunkById(200, function (Collection $flights) {
    $flights->each->update([
        'departed' => false,
        'cancelled' => true
    ]);
}, column: 'id');
```

<a name="chunking-using-lazy-collections"></a>
### 使用 Lazy 集合分塊

`lazy` 方法與 [`chunk` 方法](#chunking-results) 的運作方式類似，因為在後台，它也會分塊執行查詢。然而，`lazy` 方法並非將每個分塊直接按原樣傳遞給回呼，而是回傳一個扁平化的 Eloquent 模型 [LazyCollection](/docs/{{version}}/collections#lazy-collections)，這讓你可以像處理單一串流一樣與結果互動：

```php
use App\Models\Flight;

foreach (Flight::lazy() as $flight) {
    // ...
}
```

如果你是根據某個欄位過濾 `lazy` 方法的結果，且在迭代結果時也會更新該欄位，則你應該使用 `lazyById` 方法。在內部，`lazyById` 方法始終會檢索 `id` 欄位大於上一個分塊中最後一個模型的模型：

```php
Flight::where('departed', true)
    ->lazyById(200, column: 'id')
    ->each->update(['departed' => false]);
```

你可以使用 `lazyByIdDesc` 方法根據 `id` 的降冪排序來過濾結果。

<a name="cursors"></a>
### 遊標 (Cursors)

與 `lazy` 方法類似，`cursor` 方法可用於在迭代數萬條 Eloquent 模型紀錄時，顯著減少應用程式的記憶體消耗。

`cursor` 方法只會執行單一資料庫查詢；然而，個別的 Eloquent 模型在實際被迭代之前不會被填充數據 (Hydrated)。因此，在迭代遊標時，記憶體中一次僅會保留一個 Eloquent 模型。

> [!WARNING]
> 由於 `cursor` 方法一次僅在記憶體中持有一個 Eloquent 模型，因此它無法預載入 (Eager Load) 關聯。如果您需要預載入關聯，請考慮改用 [the `lazy` method](#chunking-using-lazy-collections)。

在內部，`cursor` 方法使用 PHP [產生器 (Generators)](https://www.php.net/manual/en/language.generators.overview.php) 來實作此功能：

```php
use App\Models\Flight;

foreach (Flight::where('destination', 'Zurich')->cursor() as $flight) {
    // ...
}
```

`cursor` 會回傳一個 `Illuminate\Support\LazyCollection` 實例。[Lazy 集合](/docs/{{version}}/collections#lazy-collections) 允許您使用許多在典型 Laravel 集合中可用的集合方法，且一次僅載入單一模型到記憶體中：

```php
use App\Models\User;

$users = User::cursor()->filter(function (User $user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```

雖然 `cursor` 方法比一般查詢節省更多記憶體（因為一次僅在記憶體中持有單一 Eloquent 模型），但最終仍可能耗盡記憶體。這是[由於 PHP 的 PDO 驅動程式內部會將所有原始查詢結果快取在其緩衝區中](https://www.php.net/manual/en/mysqlinfo.concepts.buffering.php)。如果您正在處理極大量數量的 Eloquent 紀錄，請考慮改用 [the `lazy` method](#chunking-using-lazy-collections)。


<a name="advanced-subqueries"></a>
### 進階子查詢


<a name="subquery-selects"></a>
#### 子查詢 Selects

Eloquent 也提供了進階的子查詢支援，允許您在單次查詢中從關聯資料表提取資訊。例如，讓我們想像我們有一張飛行 `destinations` 資料表和一張前往目的地的 `flights` 資料表。`flights` 資料表包含一個 `arrived_at` 欄位，表示航班抵達目的地的時間。

利用查詢產生器 (Query Builder) 的 `select` 與 `addSelect` 方法中可用的子查詢功能，我們可以使用單次查詢來選取所有的 `destinations` 以及最近抵達該目的地的航班名稱：

```php
use App\Models\Destination;
use App\Models\Flight;

return Destination::addSelect(['last_flight' => Flight::select('name')
    ->whereColumn('destination_id', 'destinations.id')
    ->orderByDesc('arrived_at')
    ->limit(1)
])->get();
```


<a name="subquery-ordering"></a>
#### 子查詢排序 (Subquery Ordering)

此外，查詢產生器的 `orderBy` 函式也支援子查詢。繼續使用我們的航班範例，我們可以使用此功能根據最後一班航班抵達該目的地的時間來排序所有目的地。同樣地，這可以在執行單次資料庫查詢時完成：

```php
return Destination::orderByDesc(
    Flight::select('arrived_at')
        ->whereColumn('destination_id', 'destinations.id')
        ->orderByDesc('arrived_at')
        ->limit(1)
)->get();
```

<a name="retrieving-single-models"></a>
## 檢索單一模型 / 聚合

除了檢索符合給定查詢的所有紀錄外，您還可以使用 `find`、`first` 或 `firstWhere` 方法檢索單一紀錄。這些方法會回傳單一模型實例，而非模型集合：

```php
use App\Models\Flight;

// Retrieve a model by its primary key...
$flight = Flight::find(1);

// Retrieve the first model matching the query constraints...
$flight = Flight::where('active', 1)->first();

// Alternative to retrieving the first model matching the query constraints...
$flight = Flight::firstWhere('active', 1);
```

有時您可能希望在找不到結果時執行其他操作。`findOr` 與 `firstOr` 方法將回傳單一模型實例，或者在找不到結果時執行給定的閉包。閉包回傳的值將被視為該方法的結果：

```php
$flight = Flight::findOr(1, function () {
    // ...
});

$flight = Flight::where('legs', '>', 3)->firstOr(function () {
    // ...
});
```

<a name="not-found-exceptions"></a>
#### 找不到例外 (Not Found Exceptions)

有時您可能希望在找不到模型時拋出例外。這在路由或控制器中特別有用。`findOrFail` 和 `firstOrFail` 方法會檢索查詢的第一個結果；然而，如果找不到結果，則會拋出 `Illuminate\Database\Eloquent\ModelNotFoundException` 例外：

```php
$flight = Flight::findOrFail(1);

$flight = Flight::where('legs', '>', 3)->firstOrFail();
```

如果未捕獲 `ModelNotFoundException`，則會自動向客戶端發送 404 HTTP 回應：

```php
use App\Models\Flight;

Route::get('/api/flights/{id}', function (string $id) {
    return Flight::findOrFail($id);
});
```

<a name="retrieving-or-creating-models"></a>
### 檢索或建立模型

`firstOrCreate` 方法將嘗試使用給定的欄位 / 值配對來尋找資料庫紀錄。如果在資料庫中找不到該模型，則會插入一筆紀錄，其屬性由第一個陣列參數與選用的第二個陣列參數合併而成。

`firstOrNew` 方法與 `firstOrCreate` 類似，會嘗試在資料庫中尋找符合給定屬性的紀錄。然而，如果找不到模型，則會回傳一個新的模型實例。請注意，`firstOrNew` 回傳的模型尚未持久化到資料庫中。您需要手動呼叫 `save` 方法來持久化它：

```php
use App\Models\Flight;

// Retrieve flight by name or create it if it doesn't exist...
$flight = Flight::firstOrCreate([
    'name' => 'London to Paris'
]);

// Retrieve flight by name or create it with the name, delayed, and arrival_time attributes...
$flight = Flight::firstOrCreate(
    ['name' => 'London to Paris'],
    ['delayed' => 1, 'arrival_time' => '11:30']
);

// Retrieve flight by name or instantiate a new Flight instance...
$flight = Flight::firstOrNew([
    'name' => 'London to Paris'
]);

// Retrieve flight by name or instantiate with the name, delayed, and arrival_time attributes...
$flight = Flight::firstOrNew(
    ['name' => 'Tokyo to Sydney'],
    ['delayed' => 1, 'arrival_time' => '11:30']
);
```

<a name="retrieving-aggregates"></a>
### 檢索聚合資料

在與 Eloquent 模型互動時，您也可以使用 Laravel [查詢產生器 (query builder)](/docs/{{version}}/queries) 提供的 `count`、`sum`、`max` 和其他 [聚合方法](/docs/{{version}}/queries#aggregates)。如您所料，這些方法回傳的是純量值 (scalar value)，而非 Eloquent 模型實例：

```php
$count = Flight::where('active', 1)->count();

$max = Flight::where('active', 1)->max('price');
```

<a name="inserting-and-updating-models"></a>
## 插入與更新模型


<a name="inserts"></a>
### 插入

當然，使用 Eloquent 時，我們不僅需要從資料庫檢索模型，還需要插入新紀錄。幸運的是，Eloquent 讓這一切變得簡單。要在資料庫中插入一筆新紀錄，您應該實例化一個新的模型實例並設定模型的屬性。接著，在模型實例上呼叫 `save` 方法：

```php
<?php

namespace App\Http\Controllers;

use App\Models\Flight;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class FlightController extends Controller
{
    /**
     * Store a new flight in the database.
     */
    public function store(Request $request): RedirectResponse
    {
        // Validate the request...

        $flight = new Flight;

        $flight->name = $request->name;

        $flight->save();

        return redirect('/flights');
    }
}
```

在此範例中，我們將傳入的 HTTP 請求中的 `name` 欄位賦值給 `App\Models\Flight` 模型實例的 `name` 屬性。當我們呼叫 `save` 方法時，一筆紀錄將被插入到資料庫中。模型的 `created_at` 與 `updated_at` 時間戳記在呼叫 `save` 方法時會自動設定，因此無需手動設定它們。

或者，您可以使用 `create` 方法，僅透過單一 PHP 語句來「儲存」一個新模型。`create` 方法會回傳插入的模型實例：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，您需要在模型類別中指定 `fillable` 或 `guarded` 屬性。這些屬性是必須的，因為預設情況下，所有 Eloquent 模型都受到保護，以防止批量賦值 (Mass Assignment) 漏洞。要瞭解更多關於批量賦值的資訊，請參閱 [批量賦值文件](#mass-assignment)。


<a name="updates"></a>
### 更新

`save` 方法也可用於更新資料庫中已存在的模型。要更新模型，您應該檢索它並設定您想要更新的任何屬性。接著，您應該呼叫模型的 `save` 方法。同樣地，`updated_at` 時間戳記會自動更新，因此無需手動設定它的值：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->name = 'Paris to London';

$flight->save();
```

有時，您可能需要更新現有的模型，或者在沒有匹配模型時建立一個新模型。與 `firstOrCreate` 方法一樣，`updateOrCreate` 方法會持久化模型，因此無需手動呼叫 `save` 方法。

在下面的範例中，如果存在一架出發地 (`departure`) 為 `Oakland` 且目的地 (`destination`) 為 `San Diego` 的航班，則其 `price` 與 `discounted` 欄位將被更新。如果不存在這樣的航班，則會建立一個新航班，其屬性為第一個引數陣列與第二個引數陣列合併後的結果：

```php
$flight = Flight::updateOrCreate(
    ['departure' => 'Oakland', 'destination' => 'San Diego'],
    ['price' => 99, 'discounted' => 1]
);
```

當使用 `firstOrCreate` 或 `updateOrCreate` 等方法時，您可能不知道是建立了一個新模型還是更新了現有的模型。`wasRecentlyCreated` 屬性可指示模型是否是在其當前生命週期內建立的：

```php
$flight = Flight::updateOrCreate(
    // ...
);

if ($flight->wasRecentlyCreated) {
    // New flight record was inserted...
}
```


<a name="mass-updates"></a>
#### 批量更新

更新也可以針對符合給定查詢的模型進行。在此範例中，所有 `active` 且目的地 (`destination`) 為 `San Diego` 的航班都將被標記為延遲：

```php
Flight::where('active', 1)
    ->where('destination', 'San Diego')
    ->update(['delayed' => 1]);
```

`update` 方法預期一個包含欄位與值配對的陣列，代表應該更新的欄位。`update` 方法會回傳受影響的行數。

> [!WARNING]
> 透過 Eloquent 發出批量更新時，受更新的模型將不會觸發 `saving`、`saved`、`updating` 與 `updated` 模型事件。這是因為在發出批量更新時，實際上從未檢索過這些模型。


<a name="examining-attribute-changes"></a>
#### 檢查屬性變動

Eloquent 提供了 `isDirty`、`isClean` 與 `wasChanged` 方法，用於檢查模型的內部狀態，並確定自最初檢索模型以來其屬性發生了哪些變化。

`isDirty` 方法會確定自檢索模型以來，是否更改了模型的任何屬性。您可以將特定的屬性名稱或屬性陣列傳遞給 `isDirty` 方法，以確定是否有任何屬性是「髒的」(Dirty)。`isClean` 方法將確定自檢索模型以來，某個屬性是否保持不變。此方法也接受一個選擇性的屬性引數：

```php
use App\Models\User;

$user = User::create([
    'first_name' => 'Taylor',
    'last_name' => 'Otwell',
    'title' => 'Developer',
]);

$user->title = 'Painter';

$user->isDirty(); // true
$user->isDirty('title'); // true
$user->isDirty('first_name'); // false
$user->isDirty(['first_name', 'title']); // true

$user->isClean(); // false
$user->isClean('title'); // false
$user->isClean('first_name'); // true
$user->isClean(['first_name', 'title']); // false

$user->save();

$user->isDirty(); // false
$user->isClean(); // true
```

`wasChanged` 方法會確定在當前請求週期內上次儲存模型時，是否有任何屬性發生了變動。如果需要，您可以傳遞屬性名稱來查看特定屬性是否發生了變動：

```php
$user = User::create([
    'first_name' => 'Taylor',
    'last_name' => 'Otwell',
    'title' => 'Developer',
]);

$user->title = 'Painter';

$user->save();

$user->wasChanged(); // true
$user->wasChanged('title'); // true
$user->wasChanged(['title', 'slug']); // true
$user->wasChanged('first_name'); // false
$user->wasChanged(['first_name', 'title']); // true
```

`getOriginal` 方法會回傳一個陣列，其中包含模型的原始屬性，而不管自檢索以來模型發生了什麼變化。如果需要，您可以傳遞特定的屬性名稱來取得特定屬性的原始值：

```php
$user = User::find(1);

$user->name; // John
$user->email; // john@example.com

$user->name = 'Jack';
$user->name; // Jack

$user->getOriginal('name'); // John
$user->getOriginal(); // Array of original attributes...
```

`getChanges` 方法回傳一個陣列，其中包含模型上次儲存時變動的屬性，而 `getPrevious` 方法則回傳一個陣列，其中包含模型上次儲存前的原始屬性值：

```php
$user = User::find(1);

$user->name; // John
$user->email; // john@example.com

$user->update([
    'name' => 'Jack',
    'email' => 'jack@example.com',
]);

$user->getChanges();

/*
    [
        'name' => 'Jack',
        'email' => 'jack@example.com',
    ]
*/

$user->getPrevious();

/*
    [
        'name' => 'John',
        'email' => 'john@example.com',
    ]
*/
```

<a name="mass-assignment"></a>
### 批量賦值

您可以使用 `create` 方法，僅透過單一 PHP 語句來「儲存」一個新的模型。該方法將會回傳已插入的模型實例：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在開始使用 `create` 方法之前，您需要在模型類別上指定 `fillable` 或 `guarded` 屬性。這些屬性是必要的，因為預設情況下，所有的 Eloquent 模型都會受到保護，以防止批量賦值漏洞。

批量賦值漏洞發生在當使用者傳遞一個非預期的 HTTP 請求欄位，且該欄位更改了您不希望被更改的資料庫欄位時。例如，惡意使用者可能會透過 HTTP 請求發送 `is_admin` 參數，然後將其傳遞給模型的 `create` 方法，從而讓該使用者將自己提升為管理員。

因此，首先您應該定義哪些模型屬性是您想要開放批量賦值的。您可以透過在模型上使用 `$fillable` 屬性來達成。例如，讓我們將 `Flight` 模型的 `name` 屬性設為可批量賦值：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * The attributes that are mass assignable.
     *
     * @var array<int, string>
     */
    protected $fillable = ['name'];
}
```

一旦您指定了哪些屬性是可批量賦值的，您就可以使用 `create` 方法在資料庫中插入一筆新記錄。`create` 方法會回傳新建立的模型實例：

```php
$flight = Flight::create(['name' => 'London to Paris']);
```

如果您已經有一個模型實例，您可以使用 `fill` 方法以屬性陣列來填充它：

```php
$flight->fill(['name' => 'Amsterdam to Frankfurt']);
```


<a name="mass-assignment-json-columns"></a>
#### 批量賦值與 JSON 欄位

當分配 JSON 欄位時，必須在模型的 `$fillable` 陣列中指定每個欄位的批量賦值鍵。基於安全性考量，當使用 `guarded` 屬性時，Laravel 不支援更新巢狀 JSON 屬性：

```php
/**
 * The attributes that are mass assignable.
 *
 * @var array<int, string>
 */
protected $fillable = [
    'options->enabled',
];
```


<a name="allowing-mass-assignment"></a>
#### 允許批量賦值

如果您想讓所有屬性都可以被批量賦值，可以將模型的 `$guarded` 屬性定義為一個空陣列。如果您選擇取消模型的保護，則應特別注意，務必始終手動建構傳遞給 Eloquent 的 `fill`、`create` 和 `update` 方法的陣列：

```php
/**
 * The attributes that aren't mass assignable.
 *
 * @var array<string>|bool
 */
protected $guarded = [];
```


<a name="mass-assignment-exceptions"></a>
#### 批量賦值例外

預設情況下，執行批量賦值操作時，未包含在 `$fillable` 陣列中的屬性會被靜默丟棄。在正式環境中，這是預期的行為；然而，在本地開發期間，這可能會導致對於模型變更為何未生效感到困惑。

如果您願意，可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填充不可填充的屬性時拋出異常。通常，此方法應在應用程式 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

```php
use Illuminate\Database\Eloquent\Model;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Model::preventSilentlyDiscardingAttributes($this->app->isLocal());
}
```


<a name="upserts"></a>
### 更新或插入 (Upserts)

Eloquent 的 `upsert` 方法可用於在單一原子性操作中更新或建立記錄。該方法的第一個參數包含要插入或更新的值，而第二個參數則列出在相關資料表中唯一識別記錄的欄位。該方法的第三個也是最後一個參數是一個陣列，列出了如果資料庫中已存在匹配記錄時應更新的欄位。如果模型啟用了時間戳記，`upsert` 方法將自動設定 `created_at` 和 `updated_at` 時間戳記：

```php
Flight::upsert([
    ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
    ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
], uniqueBy: ['departure', 'destination'], update: ['price']);
```

> [!WARNING]
> 除了 SQL Server 以外的所有資料庫，都要求 `upsert` 方法第二個參數中的欄位必須具有「主鍵」或「唯一」索引。此外，MariaDB 和 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個參數，並始終使用資料表的「主鍵」和「唯一」索引來偵測現有記錄。

<a name="deleting-models"></a>
## 刪除模型

要刪除模型，您可以在模型實例上呼叫 `delete` 方法：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->delete();
```


<a name="deleting-an-existing-model-by-its-primary-key"></a>
#### 透過主鍵刪除現有模型

在上面的範例中，我們在呼叫 `delete` 方法之前先從資料庫檢索模型。但是，如果您知道模型的主鍵，則可以呼叫 `destroy` 方法來刪除模型，而無需明確地檢索它。除了接受單一主鍵外，`destroy` 方法還接受多個主鍵、主鍵陣列或主鍵[集合](/docs/{{version}}/collections)：

```php
Flight::destroy(1);

Flight::destroy(1, 2, 3);

Flight::destroy([1, 2, 3]);

Flight::destroy(collect([1, 2, 3]));
```

如果您正在使用[軟刪除模型](#soft-deleting)，您可以使用 `forceDestroy` 方法永久刪除模型：

```php
Flight::forceDestroy(1);
```

> [!WARNING]
> `destroy` 方法會單獨載入每個模型並呼叫 `delete` 方法，以便為每個模型正確發送 `deleting` 和 `deleted` 事件。


<a name="deleting-models-using-queries"></a>
#### 使用查詢刪除模型

當然，您可以建立 Eloquent 查詢來刪除符合查詢條件的所有模型。在此範例中，我們將刪除所有標記為非活動狀態的航班。與批量更新一樣，批量刪除不會為被刪除的模型發送模型事件：

```php
$deleted = Flight::where('active', 0)->delete();
```

要刪除資料表中的所有模型，您應該執行不帶任何條件的查詢：

```php
$deleted = Flight::query()->delete();
```

> [!WARNING]
> 在透過 Eloquent 執行批量刪除語句時，不會為被刪除的模型發送 `deleting` 和 `deleted` 模型事件。這是因為在執行刪除語句時，從未實際檢索過這些模型。


<a name="soft-deleting"></a>
### 軟刪除

除了實際從資料庫中移除記錄外，Eloquent 還可以「軟刪除」模型。當模型被軟刪除時，它們不會真的從資料庫中移除。相反地，模型上會設置一個 `deleted_at` 屬性，指示模型被「刪除」的日期和時間。要為模型啟用軟刪除，請將 `Illuminate\Database\Eloquent\SoftDeletes` trait 新增到模型中：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Flight extends Model
{
    use SoftDeletes;
}
```

> [!NOTE]
> `SoftDeletes` trait 會自動為您將 `deleted_at` 屬性轉換為 `DateTime` / `Carbon` 實例。

您還應該將 `deleted_at` 欄位新增到您的資料庫表中。Laravel [結構產生器 (schema builder)](/docs/{{version}}/migrations) 包含一個建立此欄位的輔助方法：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('flights', function (Blueprint $table) {
    $table->softDeletes();
});

Schema::table('flights', function (Blueprint $table) {
    $table->dropSoftDeletes();
});
```

現在，當您在模型上呼叫 `delete` 方法時，`deleted_at` 欄位將被設置為當前日期和時間。但是，模型的資料庫記錄仍會保留在表中。當查詢使用軟刪除的模型時，軟刪除的模型將自動從所有查詢結果中排除。

要確定給定的模型實例是否已被軟刪除，您可以使用 `trashed` 方法：

```php
if ($flight->trashed()) {
    // ...
}
```


<a name="restoring-soft-deleted-models"></a>
#### 還原軟刪除模型

有時您可能希望「取消刪除」軟刪除的模型。要還原軟刪除的模型，您可以在模型實例上呼叫 `restore` 方法。`restore` 方法會將模型的 `deleted_at` 欄位設置為 `null`：

```php
$flight->restore();
```

您也可以在查詢中使用 `restore` 方法來還原多個模型。同樣地，與其他「批量」操作一樣，這不會為還原的模型發送任何模型事件：

```php
Flight::withTrashed()
    ->where('airline_id', 1)
    ->restore();
```

在構建[關聯](/docs/{{version}}/eloquent-relationships)查詢時也可以使用 `restore` 方法：

```php
$flight->history()->restore();
```


<a name="permanently-deleting-models"></a>
#### 永久刪除模型

有時您可能需要真正從資料庫中移除模型。您可以使用 `forceDelete` 方法將軟刪除的模型從資料庫表中永久移除：

```php
$flight->forceDelete();
```

在構建 Eloquent 關聯查詢時也可以使用 `forceDelete` 方法：

```php
$flight->history()->forceDelete();
```


<a name="querying-soft-deleted-models"></a>
### 查詢軟刪除模型


<a name="including-soft-deleted-models"></a>
#### 包含軟刪除模型

如上所述，軟刪除的模型將自動從查詢結果中排除。但是，您可以透過在查詢上呼叫 `withTrashed` 方法來強制軟刪除的模型包含在查詢結果中：

```php
use App\Models\Flight;

$flights = Flight::withTrashed()
    ->where('account_id', 1)
    ->get();
```

在構建[關聯](/docs/{{version}}/eloquent-relationships)查詢時也可以呼叫 `withTrashed` 方法：

```php
$flight->history()->withTrashed()->get();
```


<a name="retrieving-only-soft-deleted-models"></a>
#### 僅檢索軟刪除模型

`onlyTrashed` 方法將**僅**檢索軟刪除的模型：

```php
$flights = Flight::onlyTrashed()
    ->where('airline_id', 1)
    ->get();
```

<a name="pruning-models"></a>
## 修剪模型

有時您可能希望定期刪除不再需要的模型。為此，您可以將 `Illuminate\Database\Eloquent\Prunable` 或 `Illuminate\Database\Eloquent\MassPrunable` trait 加到您想要定期修剪的模型中。在模型中加入其中一個 trait 後，請實作一個 `prunable` 方法，該方法會回傳一個 Eloquent 查詢產生器，用以解析出不再需要的模型：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Prunable;

class Flight extends Model
{
    use Prunable;

    /**
     * Get the prunable model query.
     */
    public function prunable(): Builder
    {
        return static::where('created_at', '<=', now()->minus(months: 1));
    }
}
```

當模型標記為 `Prunable` 時，您也可以在模型中定義一個 `pruning` 方法。此方法會在模型被刪除前呼叫。在模型從資料庫中永久移除之前，此方法對於刪除與模型相關的任何額外資源（例如存儲的檔案）非常有用：

```php
/**
 * Prepare the model for pruning.
 */
protected function pruning(): void
{
    // ...
}
```

設定好可修剪模型後，您應該在應用程式的 `routes/console.php` 檔案中排程執行 `model:prune` Artisan 指令。您可以自由選擇適合執行此指令的時間間隔：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('model:prune')->daily();
```

在背景執行時，`model:prune` 指令會自動偵測應用程式 `app/Models` 目錄下的 「Prunable」 模型。如果您的模型位於不同的位置，您可以使用 `--model` 選項來指定模型類別名稱：

```php
Schedule::command('model:prune', [
    '--model' => [Address::class, Flight::class],
])->daily();
```

如果您希望在修剪所有其他偵測到的模型時排除某些模型，可以使用 `--except` 選項：

```php
Schedule::command('model:prune', [
    '--except' => [Address::class, Flight::class],
])->daily();
```

您可以透過執行帶有 `--pretend` 選項的 `model:prune` 指令來測試您的 `prunable` 查詢。在模擬執行時，`model:prune` 指令只會回報如果實際執行該指令將會修剪多少筆紀錄：

```shell
php artisan model:prune --pretend
```

> [!WARNING]
> 軟刪除模型如果符合修剪查詢，將會被永久刪除 (`forceDelete`)。


<a name="mass-pruning"></a>
#### 批量修剪

當模型被標記為 `Illuminate\Database\Eloquent\MassPrunable` trait 時，會使用批量刪除查詢從資料庫中刪除模型。因此，不會呼叫 `pruning` 方法，也不會發送 `deleting` 和 `deleted` 模型事件。這是因為模型在刪除前從未被實際檢索過，這使得修剪過程更加高效：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\MassPrunable;

class Flight extends Model
{
    use MassPrunable;

    /**
     * Get the prunable model query.
     */
    public function prunable(): Builder
    {
        return static::where('created_at', '<=', now()->minus(months: 1));
    }
}
```


<a name="replicating-models"></a>
## 複製模型

您可以使用 `replicate` 方法為現有的模型實體建立一個尚未儲存的副本。當您的模型實體共享許多相同屬性時，此方法特別有用：

```php
use App\Models\Address;

$shipping = Address::create([
    'type' => 'shipping',
    'line_1' => '123 Example Street',
    'city' => 'Victorville',
    'state' => 'CA',
    'postcode' => '90001',
]);

$billing = $shipping->replicate()->fill([
    'type' => 'billing'
]);

$billing->save();
```

若要排除一或多個屬性不被複製到新模型中，可以將陣列傳遞給 `replicate` 方法：

```php
$flight = Flight::create([
    'destination' => 'LAX',
    'origin' => 'LHR',
    'last_flown' => '2020-03-04 11:00:00',
    'last_pilot_id' => 747,
]);

$flight = $flight->replicate([
    'last_flown',
    'last_pilot_id'
]);
```

<a name="query-scopes"></a>
## 查詢範圍


<a name="global-scopes"></a>
### 全域範圍

全域範圍允許您為給定模型的所有查詢增加約束。Laravel 自有的 [軟刪除](#soft-deleting) 功能利用全域範圍僅從資料庫中檢索「未刪除」的模型。編寫您自己的全域範圍可以提供一種方便、簡單的方法，確保給定模型的每個查詢都收到特定的約束。


<a name="generating-scopes"></a>
#### 生成範圍

若要生成新的全域範圍，您可以調用 `make:scope` Artisan 指令，這會將生成的範圍放在應用程式的 `app/Models/Scopes` 目錄中：

```shell
php artisan make:scope AncientScope
```


<a name="writing-global-scopes"></a>
#### 編寫全域範圍

編寫全域範圍很簡單。首先，使用 `make:scope` 指令生成一個實作 `Illuminate\Database\Eloquent\Scope` 介面的類別。`Scope` 介面要求您實作一個方法：`apply`。`apply` 方法可以根據需要向查詢添加 `where` 約束或其他類型的子句：

```php
<?php

namespace App\Models\Scopes;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class AncientScope implements Scope
{
    /**
     * Apply the scope to a given Eloquent query builder.
     */
    public function apply(Builder $builder, Model $model): void
    {
        $builder->where('created_at', '<', now()->minus(years: 2000));
    }
}
```

> [!NOTE]
> 如果您的全域範圍正在向查詢的 select 子句添加欄位，您應該使用 `addSelect` 方法而不是 `select`。這將防止意外替換查詢現有的 select 子句。


<a name="applying-global-scopes"></a>
#### 套用全域範圍

要為模型分配全域範圍，您只需在模型上放置 `ScopedBy` 屬性即可：

```php
<?php

namespace App\Models;

use App\Models\Scopes\AncientScope;
use Illuminate\Database\Eloquent\Attributes\ScopedBy;

#[ScopedBy([AncientScope::class])]
class User extends Model
{
    //
}
```

或者，您可以透過覆寫模型的 `booted` 方法並調用模型的 `addGlobalScope` 方法來手動註冊全域範圍。`addGlobalScope` 方法接受您的範圍實例作為唯一參數：

```php
<?php

namespace App\Models;

use App\Models\Scopes\AncientScope;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * The "booted" method of the model.
     */
    protected static function booted(): void
    {
        static::addGlobalScope(new AncientScope);
    }
}
```

將上述範例中的範圍添加到 `App\Models\User` 模型後，調用 `User::all()` 方法將執行以下 SQL 查詢：

```sql
select * from `users` where `created_at` < 0021-02-18 00:00:00
```


<a name="anonymous-global-scopes"></a>
#### 匿名全域範圍

Eloquent 還允許您使用閉包定義全域範圍，這對於不需要獨立類別的簡單範圍特別有用。當使用閉包定義全域範圍時，您應該提供一個自選的範圍名稱作為 `addGlobalScope` 方法的第一個參數：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * The "booted" method of the model.
     */
    protected static function booted(): void
    {
        static::addGlobalScope('ancient', function (Builder $builder) {
            $builder->where('created_at', '<', now()->minus(years: 2000));
        });
    }
}
```


<a name="removing-global-scopes"></a>
#### 移除全域範圍

如果您想移除給定查詢的全域範圍，可以使用 `withoutGlobalScope` 方法。此方法接受全域範圍的類別名稱作為其唯一參數：

```php
User::withoutGlobalScope(AncientScope::class)->get();
```

或者，如果您使用閉包定義了全域範圍，則應傳遞您分配給該全域範圍的字串名稱：

```php
User::withoutGlobalScope('ancient')->get();
```

如果您想移除多個甚至所有的全域範圍，可以使用 `withoutGlobalScopes` 和 `withoutGlobalScopesExcept` 方法：

```php
// Remove all of the global scopes...
User::withoutGlobalScopes()->get();

// Remove some of the global scopes...
User::withoutGlobalScopes([
    FirstScope::class, SecondScope::class
])->get();

// Remove all global scopes except the given ones...
User::withoutGlobalScopesExcept([
    SecondScope::class,
])->get();
```


<a name="local-scopes"></a>
### 本地範圍

本地範圍允許您定義常見的查詢約束集，以便在整個應用程式中輕鬆重複使用。例如，您可能需要經常檢索所有被認為是「熱門」的使用者。要定義一個範圍，請將 `Scope` 屬性添加到 Eloquent 方法中。

範圍應始終回傳相同的查詢產生器實例或 `void`：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Scope a query to only include popular users.
     */
    #[Scope]
    protected function popular(Builder $query): void
    {
        $query->where('votes', '>', 100);
    }

    /**
     * Scope a query to only include active users.
     */
    #[Scope]
    protected function active(Builder $query): void
    {
        $query->where('active', 1);
    }
}
```


<a name="utilizing-a-local-scope"></a>
#### 使用本地範圍

定義範圍後，您可以在查詢模型時調用範圍方法。您甚至可以鏈式調用各種範圍：

```php
use App\Models\User;

$users = User::popular()->active()->orderBy('created_at')->get();
```

透過 `or` 查詢運算子組合多個 Eloquent 模型範圍可能需要使用閉包來達成正確的 [邏輯分組](/docs/{{version}}/queries#logical-grouping)：

```php
$users = User::popular()->orWhere(function (Builder $query) {
    $query->active();
})->get();
```

然而，由於這可能很繁瑣，Laravel 提供了一個「高階」的 `orWhere` 方法，允許您流暢地鏈接範圍而無需使用閉包：

```php
$users = User::popular()->orWhere->active()->get();
```


<a name="dynamic-scopes"></a>
#### 動態範圍

有時您可能希望定義一個接受參數的範圍。首先，只需將額外的參數添加到範圍方法的簽章中。範圍參數應定義在 `$query` 參數之後：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Scope a query to only include users of a given type.
     */
    #[Scope]
    protected function ofType(Builder $query, string $type): void
    {
        $query->where('type', $type);
    }
}
```

一旦預期參數添加到您的範圍方法簽章後，您可以在調用範圍時傳遞參數：

```php
$users = User::ofType('admin')->get();
```

<a name="pending-attributes"></a>
### 待處理屬性

如果您想使用查詢範圍來建立具有與用於限制範圍相同屬性的模型，您可以在構建範圍查詢時使用 `withAttributes` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    /**
     * Scope the query to only include drafts.
     */
    #[Scope]
    protected function draft(Builder $query): void
    {
        $query->withAttributes([
            'hidden' => true,
        ]);
    }
}
```

`withAttributes` 方法會使用指定的屬性在查詢中加入 `where` 條件，並且也會將這些指定的屬性加入到透過該範圍建立的任何模型中：

```php
$draft = Post::draft()->create(['title' => 'In Progress']);

$draft->hidden; // true
```

若要指示 `withAttributes` 方法不要在查詢中加入 `where` 條件，您可以將 `asConditions` 參數設為 `false`：

```php
$query->withAttributes([
    'hidden' => true,
], asConditions: false);
```

<a name="comparing-models"></a>
## 比較模型

有時您可能需要判斷兩個模型是否「相同」。`is` 與 `isNot` 方法可用於快速驗證兩個模型是否具有相同的主鍵、資料表及資料庫連接：

```php
if ($post->is($anotherPost)) {
    // ...
}

if ($post->isNot($anotherPost)) {
    // ...
}
```

在使用 `belongsTo`、`hasOne`、`morphTo` 與 `morphOne` [關聯](/docs/{{version}}/eloquent-relationships) 時，也可以使用 `is` 與 `isNot` 方法。當您想要在不執行查詢來檢索該模型的情況下比較相關模型時，此方法特別有用：

```php
if ($post->author()->is($user)) {
    // ...
}
```

<a name="events"></a>
## 事件

> [!NOTE]
> 想要將您的 Eloquent 事件直接廣播到您的客戶端應用程式嗎？請查看 Laravel 的 [模型事件廣播](/docs/{{version}}/broadcasting#model-broadcasting)。

Eloquent 模型會發送多個事件，允許您掛入模型生命週期的以下時刻：`retrieved`、`creating`、`created`、`updating`、`updated`、`saving`、`saved`、`deleting`、`deleted`、`trashed`、`forceDeleting`、`forceDeleted`、`restoring`、`restored` 以及 `replicating`。

`retrieved` 事件會在從資料庫檢索現有模型時發送。當新模型第一次被儲存時，會發送 `creating` 與 `created` 事件。當修改現有模型並呼叫 `save` 方法時，會發送 `updating` / `updated` 事件。無論是建立還是更新模型，都會發送 `saving` / `saved` 事件——即使模型的屬性沒有改變也是如此。以 `-ing` 結尾的事件名稱會在模型的任何變更被持久化之前發送，而以 `-ed` 結尾的事件則是在變更持久化後發送。

要開始監聽模型事件，請在您的 Eloquent 模型上定義一個 `$dispatchesEvents` 屬性。此屬性將 Eloquent 模型生命週期的各個點對應到您自己的 [事件類別](/docs/{{version}}/events)。每個模型事件類別都應該預期透過其建構子接收受影響的模型實例：

```php
<?php

namespace App\Models;

use App\Events\UserDeleted;
use App\Events\UserSaved;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;

class User extends Authenticatable
{
    use Notifiable;

    /**
     * The event map for the model.
     *
     * @var array<string, string>
     */
    protected $dispatchesEvents = [
        'saved' => UserSaved::class,
        'deleted' => UserDeleted::class,
    ];
}
```

在定義並對應您的 Eloquent 事件之後，您可以使用 [事件監聽器](/docs/{{version}}/events#defining-listeners) 來處理這些事件。

> [!WARNING]
> 當透過 Eloquent 發出批量更新或刪除查詢時，受影響的模型將不會發送 `saved`、`updated`、`deleting` 與 `deleted` 模型事件。這是因為在執行批量更新或刪除時，實際上從未檢索過這些模型。


<a name="events-using-closures"></a>
### 使用閉包

除了使用自定義事件類別外，您還可以註冊在各種模型事件發送時執行的閉包。通常，您應該在模型的 `booted` 方法中註冊這些閉包：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * The "booted" method of the model.
     */
    protected static function booted(): void
    {
        static::created(function (User $user) {
            // ...
        });
    }
}
```

如果需要，您可以在註冊模型事件時利用 [可佇列的匿名事件監聽器](/docs/{{version}}/events#queuable-anonymous-event-listeners)。這將指示 Laravel 使用應用程式的 [佇列 (Queue)](/docs/{{version}}/queues) 在背景執行模型事件監聽器：

```php
use function Illuminate\Events\queueable;

static::created(queueable(function (User $user) {
    // ...
}));
```


<a name="observers"></a>
### 觀察器


<a name="defining-observers"></a>
#### 定義觀察器

如果您正在監聽特定模型上的許多事件，您可以使用觀察器將所有監聽器分組到單一類別中。觀察器類別的方法名稱反映了您希望監聽的 Eloquent 事件。這些方法中的每一個都接收受影響的模型作為其唯一的參數。`make:observer` Artisan 指令是建立新觀察器類別最簡單的方法：

```shell
php artisan make:observer UserObserver --model=User
```

此指令會將新的觀察器放在您的 `app/Observers` 目錄中。如果該目錄不存在，Artisan 會為您建立。您剛建立的觀察器看起來會像下面這樣：

```php
<?php

namespace App\Observers;

use App\Models\User;

class UserObserver
{
    /**
     * Handle the User "created" event.
     */
    public function created(User $user): void
    {
        // ...
    }

    /**
     * Handle the User "updated" event.
     */
    public function updated(User $user): void
    {
        // ...
    }

    /**
     * Handle the User "deleted" event.
     */
    public function deleted(User $user): void
    {
        // ...
    }

    /**
     * Handle the User "restored" event.
     */
    public function restored(User $user): void
    {
        // ...
    }

    /**
     * Handle the User "forceDeleted" event.
     */
    public function forceDeleted(User $user): void
    {
        // ...
    }
}
```

要註冊觀察器，您可以在對應的模型上放置 `ObservedBy` 屬性：

```php
use App\Observers\UserObserver;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy([UserObserver::class])]
class User extends Authenticatable
{
    //
}
```

或者，您可以透過在想要觀察的模型上呼叫 `observe` 方法來手動註冊觀察器。您可以在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中註冊觀察器：

```php
use App\Models\User;
use App\Observers\UserObserver;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    User::observe(UserObserver::class);
}
```

> [!NOTE]
> 觀察器還可以監聽其他事件，例如 `saving` 與 `retrieved`。這些事件在 [事件](#events) 文件中有所說明。


<a name="observers-and-database-transactions"></a>
#### 觀察器與資料庫交易

當模型是在資料庫交易中建立時，您可能希望指示觀察器僅在資料庫交易提交 (Commit) 後才執行其事件處理程序。您可以透過在觀察器上實作 `ShouldHandleEventsAfterCommit` 介面來達成此目的。如果資料庫交易不在進行中，事件處理程序將立即執行：

```php
<?php

namespace App\Observers;

use App\Models\User;
use Illuminate\Contracts\Events\ShouldHandleEventsAfterCommit;

class UserObserver implements ShouldHandleEventsAfterCommit
{
    /**
     * Handle the User "created" event.
     */
    public function created(User $user): void
    {
        // ...
    }
}
```


<a name="muting-events"></a>
### 靜音事件

您偶爾可能需要暫時「靜音」模型觸發的所有事件。您可以使用 `withoutEvents` 方法來達成此目的。`withoutEvents` 方法接收一個閉包作為其唯一參數。在此閉包內執行的任何程式碼都不會發送模型事件，且閉包返回的任何值都將由 `withoutEvents` 方法返回：

```php
use App\Models\User;

$user = User::withoutEvents(function () {
    User::findOrFail(1)->delete();

    return User::find(2);
});
```


<a name="saving-a-single-model-without-events"></a>
#### 儲存單一模型而不發送事件

有時您可能希望「儲存」給定的模型而不發送任何事件。您可以使用 `saveQuietly` 方法來達成此目的：

```php
$user = User::findOrFail(1);

$user->name = 'Victoria Faith';

$user->saveQuietly();
```

您也可以在不發送任何事件的情況下「更新 (Update)」、「刪除 (Delete)」、「軟刪除 (Soft Delete)」、「還原 (Restore)」以及「複製 (Replicate)」給定的模型：

```php
$user->deleteQuietly();
$user->forceDeleteQuietly();
$user->restoreQuietly();
```