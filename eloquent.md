# Eloquent：入門

- [簡介](#introduction)
- [產生 Model 類別](#generating-model-classes)
- [Eloquent Model 慣例](#eloquent-model-conventions)
    - [資料表名稱](#table-names)
    - [主鍵](#primary-keys)
    - [UUID 與 ULID 鍵](#uuid-and-ulid-keys)
    - [時間戳記](#timestamps)
    - [資料庫連線](#database-connections)
    - [預設屬性值](#default-attribute-values)
    - [設定 Eloquent 的嚴謹度](#configuring-eloquent-strictness)
- [擷取 Model](#retrieving-models)
    - [集合](#collections)
    - [分批處理結果](#chunking-results)
    - [使用惰性集合分批處理](#chunking-using-lazy-collections)
    - [遊標](#cursors)
    - [進階子查詢](#advanced-subqueries)
- [擷取單一 Model / 聚合結果](#retrieving-single-models)
    - [擷取或建立 Model](#retrieving-or-creating-models)
    - [擷取聚合結果](#retrieving-aggregates)
- [新增與更新 Model](#inserting-and-updating-models)
    - [新增](#inserts)
    - [更新](#updates)
    - [批量指派](#mass-assignment)
    - [Upsert](#upserts)
- [刪除 Model](#deleting-models)
    - [軟刪除](#soft-deleting)
    - [查詢軟刪除的 Model](#querying-soft-deleted-models)
- [修剪 Model](#pruning-models)
- [複製 Model](#replicating-models)
- [查詢 Scope](#query-scopes)
    - [全域 Scope](#global-scopes)
    - [本地 Scope](#local-scopes)
    - [待處理屬性](#pending-attributes)
- [比較 Model](#comparing-models)
- [事件](#events)
    - [使用閉包](#events-using-closures)
    - [觀察者](#observers)
    - [靜音事件](#muting-events)

<a name="introduction"></a>
## 簡介

Laravel 內建了 Eloquent，這是一個物件關聯對映 (ORM)，讓與資料庫的互動變得更加愉快。使用 Eloquent 時，每個資料庫資料表都有一個對應的「Model」，用於與該資料表進行互動。除了從資料庫資料表擷取記錄外，Eloquent Model 也允許你對資料表新增、更新及刪除記錄。

> [!NOTE]
> 在開始之前，請務必在應用程式的 `config/database.php` 設定檔中設定資料庫連線。有關設定資料庫的更多資訊，請查閱[資料庫設定文件](/docs/{{version}}/database#configuration)。

<a name="generating-model-classes"></a>
## 產生 Model 類別

首先，讓我們先來建立一個 Eloquent Model。Model 通常位於 `app\Models` 目錄中，並繼承 `Illuminate\Database\Eloquent\Model` 類別。你可以使用 `make:model` [Artisan 指令](/docs/{{version}}/artisan)來產生一個新的 Model：

```shell
php artisan make:model Flight
```

如果你想在產生 Model 時一併產生[資料庫遷移](/docs/{{version}}/migrations)檔案，可以使用 `--migration` 或 `-m` 選項：

```shell
php artisan make:model Flight --migration
```

在產生 Model 時，你可以產生各種其他類型的類別，例如 factories、seeders、policies、controllers 及 form requests。此外，這些選項可以組合使用，一次建立多個類別：

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
#### 檢查 Model

有時候，單憑瀏覽 Model 的程式碼很難判斷其所有可用的屬性與關聯。你可以嘗試使用 `model:show` Artisan 指令，它會提供 Model 所有屬性與關聯的方便總覽：

```shell
php artisan model:show Flight
```

<a name="eloquent-model-conventions"></a>
## Eloquent Model 慣例

由 `make:model` 命令產生的 Model 會被放置在 `app/Models` 目錄中。讓我們來檢視一個基本的 Model 類別，並討論一些 Eloquent 的主要慣例：

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

看過上面的範例後，您可能已經注意到我們沒有告訴 Eloquent 哪個資料庫資料表對應到我們的 `Flight` Model。依照慣例，類別的「snake case」複數名稱會被用作資料表名稱，除非明確指定其他名稱。因此，在這種情況下，Eloquent 會假設 `Flight` Model 將紀錄儲存在 `flights` 資料表中，而 `AirTrafficController` Model 會將紀錄儲存在 `air_traffic_controllers` 資料表中。

如果您的 Model 對應的資料庫資料表不符合此慣例，您可以透過在 Model 上定義一個 `table` 屬性來手動指定 Model 的資料表名稱：

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

Eloquent 也會假設每個 Model 對應的資料庫資料表都有一個名為 `id` 的主鍵欄位。如有必要，您可以在 Model 上定義一個保護的 `$primaryKey` 屬性，以指定作為 Model 主鍵的不同欄位：

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

此外，Eloquent 假設主鍵是一個遞增的整數值，這表示 Eloquent 會自動將主鍵轉換為整數。如果您希望使用非遞增或非數值的主鍵，則必須在 Model 上定義一個設定為 `false` 的公共 `$incrementing` 屬性：

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

如果您的 Model 的主鍵不是整數，您應該在 Model 上定義一個保護的 `$keyType` 屬性。此屬性應具有 `string` 值：

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
#### 「複合式」主鍵

Eloquent 要求每個 Model 至少有一個唯一的識別碼「ID」作為其主鍵。Eloquent Model 不支援「複合式」主鍵。但是，除了資料表唯一的識別主鍵之外，您可以自由地在資料庫資料表中新增多欄位的唯一索引。

<a name="uuid-and-ulid-keys"></a>
### UUID 與 ULID 鍵

您可以選擇使用 UUIDs 作為 Eloquent Model 的主鍵，而不是使用自動遞增的整數。UUIDs 是通用唯一字母數字識別碼，長度為 36 個字元。

如果您希望 Model 使用 UUID 鍵而非自動遞增的整數鍵，您可以在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUuids` trait。當然，您應該確保 Model 具有 [UUID 對應的主鍵欄位](/docs/{{version}}/migrations#column-method-uuid)：

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

預設情況下，`HasUuids` trait 會為您的 Model 產生「有序」的 [UUIDs](/docs/{{version}}/strings#method-str-ordered-uuid)。這些 UUIDs 對於索引資料庫儲存更有效率，因為它們可以按字典順序排序。

您可以透過在 Model 上定義 `newUniqueId` 方法來覆寫給定 Model 的 UUID 產生過程。此外，您可以透過在 Model 上定義 `uniqueIds` 方法來指定哪些欄位應該接收 UUIDs：

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

如果您願意，您可以選擇使用「ULIDs」而不是 UUIDs。ULIDs 與 UUIDs 相似；但是，它們的長度僅為 26 個字元。如同有序 UUIDs，ULIDs 可以按字典順序排序，以實現高效的資料庫索引。要使用 ULIDs，您應該在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUlids` trait。您還應該確保 Model 具有 [ULID 對應的主鍵欄位](/docs/{{version}}/migrations#column-method-ulid)：

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

預設情況下，Eloquent 預期 Model 對應的資料庫資料表上存在 `created_at` 和 `updated_at` 欄位。當 Model 建立或更新時，Eloquent 會自動設定這些欄位的值。如果您不希望這些欄位由 Eloquent 自動管理，您應該在 Model 上定義一個 `$timestamps` 屬性並將其值設定為 `false`：

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

如果您需要自訂 Model 時間戳記的格式，請在 Model 上設定 `$dateFormat` 屬性。此屬性決定了日期屬性在資料庫中如何儲存，以及當 Model 序列化為陣列或 JSON 時的格式：

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

如果您需要自訂用於儲存時間戳記的欄位名稱，您可以在 Model 上定義 `CREATED_AT` 和 `UPDATED_AT` 常數：

```php
<?php

class Flight extends Model
{
    const CREATED_AT = 'creation_date';
    const UPDATED_AT = 'updated_date';
}
```

如果您希望在不修改 Model 的 `updated_at` 時間戳記的情況下執行 Model 操作，您可以在 `withoutTimestamps` 方法給定的閉包中對 Model 進行操作：

```php
Model::withoutTimestamps(fn () => $post->increment('reads'));
```

<a name="database-connections"></a>
### 資料庫連線

預設情況下，所有 Eloquent Model 都會使用為您的應用程式設定的預設資料庫連線。如果您希望指定與特定 Model 互動時應使用的不同連線，您應該在 Model 上定義 `$connection` 屬性：

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

預設情況下，新實例化的 Model 實例將不包含任何屬性值。如果您想為 Model 的一些屬性定義預設值，可以在 Model 上定義 `$attributes` 屬性。放置在 `$attributes` 陣列中的屬性值，應以其原始的「可儲存」格式呈現，就像它們是剛從資料庫中讀取出來一樣：

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
### 設定 Eloquent 的嚴謹度

Laravel 提供了多種方法，讓您可以在各種情況下設定 Eloquent 的行為和「嚴謹度」。

首先，`preventLazyLoading` 方法接受一個可選的布林引數，用於指示是否應阻止延遲載入 (lazy loading)。例如，您可能只希望在非生產環境中停用延遲載入，這樣即使生產程式碼中意外存在延遲載入的關聯，您的生產環境也能繼續正常運作。通常，此方法應在應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫：

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

此外，您可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填充一個不可填充 (unfillable) 屬性時拋出例外。這有助於在本地開發期間，當嘗試設定一個尚未新增到 Model 的 `fillable` 陣列中的屬性時，防止發生意外錯誤。

```php
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

<a name="retrieving-models"></a>
## 擷取 Model

一旦您建立了 Model 及其[關聯的資料庫資料表](/docs/{{version}}/migrations#generating-migrations)，就可以開始從資料庫中擷取資料了。您可以將每個 Eloquent Model 視為一個強大的[查詢產生器](/docs/{{version}}/queries)，讓您能夠流暢地查詢與 Model 相關聯的資料庫資料表。Model 的 `all` 方法將會擷取 Model 關聯資料庫資料表中的所有記錄：

```php
use App\Models\Flight;

foreach (Flight::all() as $flight) {
    echo $flight->name;
}
```


<a name="building-queries"></a>
#### 建構查詢

Eloquent 的 `all` 方法會回傳 Model 資料表中的所有結果。然而，由於每個 Eloquent Model 都作為一個[查詢產生器](/docs/{{version}}/queries)，您可以為查詢添加額外的約束，然後調用 `get` 方法來擷取結果：

```php
$flights = Flight::where('active', 1)
    ->orderBy('name')
    ->limit(10)
    ->get();
```

> [!NOTE]
> 由於 Eloquent Model 是查詢產生器，您應該查閱 Laravel [查詢產生器](/docs/{{version}}/queries)提供的所有方法。在撰寫 Eloquent 查詢時，您可以使用其中任何方法。


<a name="refreshing-models"></a>
#### 重新整理 Model

如果您已經有一個從資料庫中擷取到的 Eloquent Model 實例，您可以使用 `fresh` 和 `refresh` 方法來「重新整理」Model。`fresh` 方法將會從資料庫中重新擷取 Model。現有的 Model 實例將不會受到影響：

```php
$flight = Flight::where('number', 'FR 900')->first();

$freshFlight = $flight->fresh();
```

`refresh` 方法將會使用來自資料庫的新資料重新載入現有的 Model。此外，其所有已載入的關係也會一併重新整理：

```php
$flight = Flight::where('number', 'FR 900')->first();

$flight->number = 'FR 456';

$flight->refresh();

$flight->number; // "FR 900"
```


<a name="collections"></a>
### 集合

如我們所見，Eloquent 方法如 `all` 和 `get` 會從資料庫中擷取多筆記錄。然而，這些方法並不會回傳普通的 PHP 陣列。相反地，它會回傳 `Illuminate\Database\Eloquent\Collection` 的實例。

Eloquent 的 `Collection` 類別擴展了 Laravel 的基礎 `Illuminate\Support\Collection` 類別，該類別提供了[多種有用的方法](/docs/{{version}}/collections#available-methods)來與資料集合互動。例如，`reject` 方法可用於根據調用的閉包結果從集合中移除 Model：

```php
$flights = Flight::where('destination', 'Paris')->get();

$flights = $flights->reject(function (Flight $flight) {
    return $flight->cancelled;
});
```

除了 Laravel 基礎集合類別提供的方法之外，Eloquent 集合類別還提供了[一些額外的方法](/docs/{{version}}/eloquent-collections#available-methods)，這些方法專門用於與 Eloquent Model 集合互動。

由於 Laravel 的所有集合都實作了 PHP 的可迭代介面，您可以像遍歷陣列一樣遍歷集合：

```php
foreach ($flights as $flight) {
    echo $flight->name;
}
```


<a name="chunking-results"></a>
### 分批處理結果

如果嘗試透過 `all` 或 `get` 方法載入數萬筆 Eloquent 記錄，您的應用程式可能會耗盡記憶體。為了替代這些方法，可以使用 `chunk` 方法來更有效地處理大量 Model。

`chunk` 方法會擷取 Eloquent Model 的子集，並將其傳遞給閉包進行處理。由於每次只擷取當前批次的 Eloquent Model，因此在使用大量 Model 時，`chunk` 方法將顯著減少記憶體使用量：

```php
use App\Models\Flight;
use Illuminate\Database\Eloquent\Collection;

Flight::chunk(200, function (Collection $flights) {
    foreach ($flights as $flight) {
        // ...
    }
});
```

傳遞給 `chunk` 方法的第一個參數是您希望每個「批次」接收的記錄數。作為第二個參數傳遞的閉包將會對從資料庫中擷取的每個批次執行。每次擷取記錄批次時，都會執行一次資料庫查詢。

如果您正在根據一個在遍歷結果時也會更新的欄位來過濾 `chunk` 方法的結果，則應該使用 `chunkById` 方法。在這些情況下使用 `chunk` 方法可能會導致意外且不一致的結果。在內部，`chunkById` 方法始終會擷取 `id` 欄位大於前一個批次中最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->chunkById(200, function (Collection $flights) {
        $flights->each->update(['departed' => false]);
    }, column: 'id');
```

由於 `chunkById` 和 `lazyById` 方法會將其自己的「where」條件添加到正在執行的查詢中，因此您通常應該在閉包中[邏輯分組](/docs/{{version}}/queries#logical-grouping)您自己的條件：

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
### 使用惰性集合分批處理

`lazy` 方法與[ `chunk` 方法](#chunking-results)的工作方式相似，即在幕後它會分批執行查詢。然而，`lazy` 方法不會將每個批次直接傳遞給回調，而是回傳一個扁平化的 Eloquent Model [LazyCollection](/docs/{{version}}/collections#lazy-collections)，這讓您可以將結果作為單一流進行互動：

```php
use App\Models\Flight;

foreach (Flight::lazy() as $flight) {
    // ...
}
```

如果您正在根據一個在遍歷結果時也會更新的欄位來過濾 `lazy` 方法的結果，則應該使用 `lazyById` 方法。在內部，`lazyById` 方法始終會擷取 `id` 欄位大於前一個批次中最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->lazyById(200, column: 'id')
    ->each->update(['departed' => false]);
```

您可以使用 `lazyByIdDesc` 方法根據 `id` 的降序來過濾結果。

<a name="cursors"></a>
### 遊標

與 `lazy` 方法類似，`cursor` 方法可用於在遍歷數萬個 Eloquent Model 記錄時，顯著降低應用程式的記憶體消耗。

`cursor` 方法只會執行單一資料庫查詢；然而，個別的 Eloquent Model 在實際遍歷之前不會被實例化 (hydrated)。因此，在遍歷遊標時，記憶體中一次只會保留一個 Eloquent Model。

> [!WARNING]
> 由於 `cursor` 方法在記憶體中一次只保留一個 Eloquent Model，因此它無法預先載入 (eager load) 關聯。如果您需要預先載入關聯，請考慮改用 [the `lazy` method](#chunking-using-lazy-collections)。

在內部，`cursor` 方法使用 PHP 的 [生成器 (generators)](https://www.php.net/manual/en/language.generators.overview.php) 來實作此功能：

```php
use App\Models\Flight;

foreach (Flight::where('destination', 'Zurich')->cursor() as $flight) {
    // ...
}
```

`cursor` 會回傳一個 `Illuminate\Support\LazyCollection` 實例。[惰性集合 (Lazy collections)](/docs/{{version}}/collections#lazy-collections) 允許您使用典型 Laravel 集合上的許多集合方法，同時一次只載入一個 Model 到記憶體中：

```php
use App\Models\User;

$users = User::cursor()->filter(function (User $user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```

儘管 `cursor` 方法比常規查詢使用的記憶體少得多（因為它一次只在記憶體中保留一個 Eloquent Model），但它最終仍會耗盡記憶體。這是由於 [PHP 的 PDO 驅動程式會在其緩衝區中內部快取所有原始查詢結果](https://www.php.net/manual/en/mysqlinfo.concepts.buffering.php)。如果您正在處理大量的 Eloquent 記錄，請考慮改用 [the `lazy` method](#chunking-using-lazy-collections)。

<a name="advanced-subqueries"></a>
### 進階子查詢

<a name="subquery-selects"></a>
#### 子查詢選取

Eloquent 也提供進階子查詢 (subquery) 支援，讓您可以透過單一查詢從相關資料表中提取資訊。例如，假設我們有一個航班的 `destinations` (目的地) 資料表和一個前往目的地的 `flights` (航班) 資料表。`flights` 資料表包含一個 `arrived_at` 欄位，表示航班抵達目的地的時間。

透過查詢產生器 (query builder) 的 `select` 與 `addSelect` 方法提供的子查詢功能，我們可以使用單一查詢選取所有 `destinations` 以及最近抵達該目的地的航班名稱：

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
#### 子查詢排序

此外，查詢產生器 (query builder) 的 `orderBy` 函數支援子查詢。繼續以我們的航班範例為例，我們可以利用此功能，根據上次航班抵達目的地的時間來排序所有目的地。同樣地，這可以在執行單一資料庫查詢時完成：

```php
return Destination::orderByDesc(
    Flight::select('arrived_at')
        ->whereColumn('destination_id', 'destinations.id')
        ->orderByDesc('arrived_at')
        ->limit(1)
)->get();
```

<a name="retrieving-single-models"></a>
## 擷取單一 Model / 聚合結果

除了擷取符合指定查詢的所有紀錄之外，您也可以使用 `find`、`first` 或 `firstWhere` 方法來擷取單一紀錄。這些方法並非回傳 Model 集合，而是回傳單一 Model 實例：

```php
use App\Models\Flight;

// Retrieve a model by its primary key...
$flight = Flight::find(1);

// Retrieve the first model matching the query constraints...
$flight = Flight::where('active', 1)->first();

// Alternative to retrieving the first model matching the query constraints...
$flight = Flight::firstWhere('active', 1);
```

有時候，如果找不到任何結果，您可能希望執行其他動作。`findOr` 和 `firstOr` 方法將會回傳單一 Model 實例，如果找不到任何結果，則會執行給定的閉包。閉包回傳的值將被視為該方法的結果：

```php
$flight = Flight::findOr(1, function () {
    // ...
});

$flight = Flight::where('legs', '>', 3)->firstOr(function () {
    // ...
});
```


<a name="not-found-exceptions"></a>
#### 找不到例外

有時候，如果找不到 Model，您可能希望拋出例外。這在路由或控制器中特別有用。`findOrFail` 和 `firstOrFail` 方法會擷取查詢的第一個結果；然而，如果找不到結果，則會拋出 `Illuminate\Database\Eloquent\ModelNotFoundException` 例外：

```php
$flight = Flight::findOrFail(1);

$flight = Flight::where('legs', '>', 3)->firstOrFail();
```

如果未捕捉到 `ModelNotFoundException`，則會自動向用戶端傳送 404 HTTP 回應：

```php
use App\Models\Flight;

Route::get('/api/flights/{id}', function (string $id) {
    return Flight::findOrFail($id);
});
```


<a name="retrieving-or-creating-models"></a>
### 擷取或建立 Model

`firstOrCreate` 方法會嘗試使用給定的欄位/值對來尋找資料庫紀錄。如果在資料庫中找不到該 Model，則會插入一條新紀錄，其屬性會合併第一個陣列引數與選用的第二個陣列引數。

`firstOrNew` 方法，與 `firstOrCreate` 類似，會嘗試在資料庫中尋找符合給定屬性的紀錄。然而，如果找不到 Model，則會回傳一個新的 Model 實例。請注意，`firstOrNew` 回傳的 Model 尚未持久化到資料庫。您需要手動呼叫 `save` 方法來將其持久化：

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
### 擷取聚合結果

在與 Eloquent Model 互動時，您也可以使用 Laravel [查詢產生器](/docs/{{version}}/queries) 提供的 `count`、`sum`、`max` 和其他 [聚合方法](/docs/{{version}}/queries#aggregates)。如您所料，這些方法會回傳純量值，而非 Eloquent Model 實例：

```php
$count = Flight::where('active', 1)->count();

$max = Flight::where('active', 1)->max('price');
```

<a name="inserting-and-updating-models"></a>
## 新增與更新 Model


<a name="inserts"></a>
### 新增

當然，使用 Eloquent 時，我們不只從資料庫擷取 Model。我們還需要新增記錄。幸運的是，Eloquent 讓這一切變得簡單。若要新增記錄到資料庫，您應該實例化一個新的 Model 實例並設定 Model 的屬性。然後，呼叫 Model 實例的 `save` 方法：

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

在此範例中，我們將傳入 HTTP 請求的 `name` 欄位指派給 `App\Models\Flight` Model 實例的 `name` 屬性。當我們呼叫 `save` 方法時，記錄將會新增到資料庫。Model 的 `created_at` 和 `updated_at` 時間戳記在呼叫 `save` 方法時將會自動設定，因此無需手動設定。

或者，您可以使用 `create` 方法，以單一 PHP 語句來「儲存」一個新的 Model。`create` 方法會傳回新增的 Model 實例給您：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，您需要為您的 Model 類別指定 `fillable` 或 `guarded` 屬性。這些屬性是必需的，因為所有 Eloquent Model 預設都受到批量指派 (mass assignment) 漏洞的保護。若要深入了解批量指派，請參閱[批量指派文件](#mass-assignment)。


<a name="updates"></a>
### 更新

`save` 方法也可以用於更新資料庫中已存在的 Model。若要更新 Model，您應該擷取它並設定您希望更新的任何屬性。然後，您應該呼叫 Model 的 `save` 方法。同樣地，`updated_at` 時間戳記將會自動更新，因此無需手動設定其值：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->name = 'Paris to London';

$flight->save();
```

有時，您可能需要更新現有的 Model，或者在沒有匹配 Model 存在時建立一個新的 Model。與 `firstOrCreate` 方法一樣，`updateOrCreate` 方法會持久化 Model，因此無需手動呼叫 `save` 方法。

在下面的範例中，如果存在一個 `departure` 位置為 `Oakland` 且 `destination` 位置為 `San Diego` 的 Flight，其 `price` 和 `discounted` 欄位將會更新。如果不存在這樣的 Flight，則會建立一個新的 Flight，其屬性是將第一個陣列引數與第二個陣列引數合併後的結果：

```php
$flight = Flight::updateOrCreate(
    ['departure' => 'Oakland', 'destination' => 'San Diego'],
    ['price' => 99, 'discounted' => 1]
);
```

當使用諸如 `firstOrCreate` 或 `updateOrCreate` 等方法時，您可能不知道是建立了新的 Model 還是更新了現有的 Model。`wasRecentlyCreated` 屬性會指示 Model 是否在其當前生命週期中被建立：

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

更新也可以針對符合給定查詢的 Model 進行。在此範例中，所有 `active` 且 `destination` 為 `San Diego` 的 Flight 都將被標記為延誤：

```php
Flight::where('active', 1)
    ->where('destination', 'San Diego')
    ->update(['delayed' => 1]);
```

`update` 方法預期一個由欄位與值組成的陣列，代表應更新的欄位。`update` 方法會傳回受影響的列數。

> [!WARNING]
> 當透過 Eloquent 發出批量更新時，`saving`、`saved`、`updating` 和 `updated` Model 事件將不會針對更新後的 Model 觸發。這是因為在發出批量更新時，Model 從未實際被擷取。


<a name="examining-attribute-changes"></a>
#### 檢查屬性變更

Eloquent 提供了 `isDirty`、`isClean` 和 `wasChanged` 方法來檢查 Model 的內部狀態，並判斷其屬性自 Model 原始擷取以來是否發生了變化。

`isDirty` 方法判斷 Model 的任何屬性自擷取以來是否已更改。您可以將特定的屬性名稱或屬性陣列傳遞給 `isDirty` 方法，以判斷任何屬性是否「髒」。`isClean` 方法將判斷屬性自 Model 擷取以來是否保持不變。此方法也接受一個可選的屬性引數：

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

`wasChanged` 方法判斷在當前請求週期中，Model 最後一次儲存時是否有任何屬性被更改。如果需要，您可以傳遞一個屬性名稱來查看特定屬性是否被更改：

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

`getOriginal` 方法傳回一個陣列，其中包含 Model 的原始屬性，無論 Model 自擷取以來發生了什麼變化。如果需要，您可以傳遞一個特定的屬性名稱來獲取該屬性的原始值：

```php
$user = User::find(1);

$user->name; // John
$user->email; // john@example.com

$user->name = 'Jack';
$user->name; // Jack

$user->getOriginal('name'); // John
$user->getOriginal(); // Array of original attributes...
```

`getChanges` 方法傳回一個陣列，其中包含 Model 最後一次儲存時更改的屬性，而 `getPrevious` 方法傳回一個陣列，其中包含 Model 最後一次儲存之前的原始屬性值：

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
### 批量指派

您可以使用 `create` 方法，以單一 PHP 語句「儲存」新的 Model。此方法會回傳已插入的 Model 實例：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，您需要在您的 Model 類別上指定 `fillable` 或 `guarded` 屬性。這些屬性是必需的，因為所有 Eloquent Model 預設都受到批量指派漏洞的保護。

當使用者傳入非預期的 HTTP 請求欄位，並且該欄位改變了資料庫中您不希望被改變的欄位時，就會發生批量指派漏洞。例如，惡意使用者可能會透過 HTTP 請求傳送 `is_admin` 參數，然後將該參數傳遞給您 Model 的 `create` 方法，從而允許該使用者將自己提升為管理員。

因此，首先您應該定義哪些 Model 屬性可以批量指派。您可以透過 Model 上的 `$fillable` 屬性來實現。例如，讓我們將 `Flight` Model 的 `name` 屬性設定為可批量指派：

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

一旦您指定了哪些屬性可以批量指派，您就可以使用 `create` 方法在資料庫中插入新紀錄。`create` 方法會回傳新建立的 Model 實例：

```php
$flight = Flight::create(['name' => 'London to Paris']);
```

如果您已經有一個 Model 實例，您可以使用 `fill` 方法以屬性陣列填充它：

```php
$flight->fill(['name' => 'Amsterdam to Frankfurt']);
```

<a name="mass-assignment-json-columns"></a>
#### 批量指派與 JSON 欄位

在指派 JSON 欄位時，每個欄位的可批量指派鍵都必須在 Model 的 `$fillable` 陣列中指定。為了安全性，Laravel 不支援在使用 `guarded` 屬性時更新巢狀 JSON 屬性：

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
#### 允許批量指派

如果您想讓所有屬性都可以批量指派，您可以將 Model 的 `$guarded` 屬性定義為一個空陣列。如果您選擇解除 Model 的保護，則應該特別注意始終手動建立傳遞給 Eloquent 的 `fill`、`create` 和 `update` 方法的陣列：

```php
/**
 * The attributes that aren't mass assignable.
 *
 * @var array<string>|bool
 */
protected $guarded = [];
```

<a name="mass-assignment-exceptions"></a>
#### 批量指派例外

預設情況下，在執行批量指派操作時，未包含在 `$fillable` 陣列中的屬性會被靜默地丟棄。在正式環境中，這是預期的行為；然而，在本地開發期間，這可能會導致 Model 變更未生效的困惑。

如果您願意，可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填充不可填充的屬性時拋出例外。通常，此方法應在應用程式 `AppServiceProvider` 類別的 `boot` 方法中調用：

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
### Upsert

Eloquent 的 `upsert` 方法可用於在單一原子操作中更新或建立記錄。該方法的第一個參數包含要插入或更新的值，第二個參數列出在關聯資料表中唯一識別記錄的欄位。該方法的第三個也是最後一個參數是一個欄位陣列，如果資料庫中已經存在匹配的記錄，則這些欄位將被更新。如果 Model 上啟用時間戳記，`upsert` 方法會自動設定 `created_at` 和 `updated_at` 時間戳記：

```php
Flight::upsert([
    ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
    ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
], uniqueBy: ['departure', 'destination'], update: ['price']);
```

> [!WARNING]
> 除 SQL Server 外，所有資料庫都要求 `upsert` 方法的第二個參數中的欄位具有「主鍵」或「唯一」索引。此外，MariaDB 和 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個參數，並始終使用資料表的主鍵和唯一索引來偵測現有記錄。

<a name="deleting-models"></a>
## 刪除 Model

若要刪除 Model，您可以對 Model 實例呼叫 `delete` 方法：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->delete();
```


<a name="deleting-an-existing-model-by-its-primary-key"></a>
#### 依主鍵刪除現有 Model

在上面的範例中，我們在呼叫 `delete` 方法之前，先從資料庫中擷取 Model。然而，如果您知道 Model 的主鍵，您可以透過呼叫 `destroy` 方法，而無需明確擷取 Model 即可刪除 Model。除了接受單一主鍵之外，`destroy` 方法還可以接受多個主鍵、一個主鍵陣列或一個 [集合](/docs/{{version}}/collections) 的主鍵：

```php
Flight::destroy(1);

Flight::destroy(1, 2, 3);

Flight::destroy([1, 2, 3]);

Flight::destroy(collect([1, 2, 3]));
```

如果您正在使用 [軟刪除的 Model](#soft-deleting)，您可以透過 `forceDestroy` 方法永久刪除 Model：

```php
Flight::forceDestroy(1);
```

> [!WARNING]
> `destroy` 方法會單獨載入每個 Model 並呼叫 `delete` 方法，以便正確地為每個 Model 觸發 `deleting` 和 `deleted` 事件。


<a name="deleting-models-using-queries"></a>
#### 使用查詢來刪除 Model

當然，您可以建立一個 Eloquent 查詢來刪除所有符合您查詢條件的 Model。在此範例中，我們將刪除所有標記為非活動狀態的航班。與批量更新一樣，批量刪除也不會為已刪除的 Model 觸發 Model 事件：

```php
$deleted = Flight::where('active', 0)->delete();
```

若要刪除資料表中的所有 Model，您應該在不添加任何條件的情況下執行查詢：

```php
$deleted = Flight::query()->delete();
```

> [!WARNING]
> 當透過 Eloquent 執行批量刪除語句時，`deleting` 和 `deleted` Model 事件將不會針對已刪除的 Model 觸發。這是因為在執行刪除語句時，Model 從未實際被擷取。


<a name="soft-deleting"></a>
### 軟刪除

除了實際從資料庫中刪除記錄之外，Eloquent 還可以「軟刪除」Model。當 Model 被軟刪除時，它們並不會實際從資料庫中移除。相反地，Model 上會設定一個 `deleted_at` 屬性，指示 Model 被「刪除」的日期和時間。若要為 Model 啟用軟刪除，請將 `Illuminate\Database\Eloquent\SoftDeletes` trait 新增至 Model：

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
> `SoftDeletes` trait 會自動將 `deleted_at` 屬性轉換為 `DateTime` / `Carbon` 實例。

您也應該將 `deleted_at` 欄位新增至您的資料庫資料表。Laravel [資料表結構產生器](/docs/{{version}}/migrations) 包含一個協助方法來建立此欄位：

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

現在，當您在 Model 上呼叫 `delete` 方法時，`deleted_at` 欄位將會設定為目前的日期和時間。然而，Model 的資料庫記錄將會保留在資料表中。當查詢使用軟刪除的 Model 時，軟刪除的 Model 將會自動從所有查詢結果中排除。

若要判斷給定的 Model 實例是否已被軟刪除，您可以使用 `trashed` 方法：

```php
if ($flight->trashed()) {
    // ...
}
```


<a name="restoring-soft-deleted-models"></a>
#### 復原軟刪除的 Model

有時您可能希望「復原」一個軟刪除的 Model。若要復原軟刪除的 Model，您可以對 Model 實例呼叫 `restore` 方法。`restore` 方法會將 Model 的 `deleted_at` 欄位設定為 `null`：

```php
$flight->restore();
```

您也可以在查詢中使用 `restore` 方法來復原多個 Model。同樣地，與其他「批量」操作一樣，這不會為已復原的 Model 觸發任何 Model 事件：

```php
Flight::withTrashed()
    ->where('airline_id', 1)
    ->restore();
```

`restore` 方法也可以在建立 [關聯](/docs/{{version}}/eloquent-relationships) 查詢時使用：

```php
$flight->history()->restore();
```


<a name="permanently-deleting-models"></a>
#### 永久刪除 Model

有時您可能需要真正從資料庫中移除 Model。您可以使用 `forceDelete` 方法來永久從資料庫資料表中移除軟刪除的 Model：

```php
$flight->forceDelete();
```

您也可以在建立 Eloquent 關聯查詢時使用 `forceDelete` 方法：

```php
$flight->history()->forceDelete();
```


<a name="querying-soft-deleted-models"></a>
### 查詢軟刪除的 Model


<a name="including-soft-deleted-models"></a>
#### 包含軟刪除的 Model

如上所述，軟刪除的 Model 將會自動從查詢結果中排除。然而，您可以透過在查詢上呼叫 `withTrashed` 方法，強制將軟刪除的 Model 包含在查詢結果中：

```php
use App\Models\Flight;

$flights = Flight::withTrashed()
    ->where('account_id', 1)
    ->get();
```

`withTrashed` 方法也可以在建立 [關聯](/docs/{{version}}/eloquent-relationships) 查詢時呼叫：

```php
$flight->history()->withTrashed()->get();
```


<a name="retrieving-only-soft-deleted-models"></a>
#### 僅擷取軟刪除的 Model

`onlyTrashed` 方法將**僅限**擷取軟刪除的 Model：

```php
$flights = Flight::onlyTrashed()
    ->where('airline_id', 1)
    ->get();
```

<a name="pruning-models"></a>
## 修剪 Model

有時您可能希望定期刪除不再需要的 Model。為此，您可以將 `Illuminate\Database\Eloquent\Prunable` 或 `Illuminate\Database\Eloquent\MassPrunable` Trait 新增至您希望定期修剪的 Model 中。將其中一個 Trait 新增至 Model 後，實作一個 `prunable` 方法，此方法會回傳一個 Eloquent 查詢建構器，用來解析不再需要的 Model：

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
        return static::where('created_at', '<=', now()->subMonth());
    }
}
```

當將 Model 標記為 `Prunable` 時，您也可以在 Model 上定義一個 `pruning` 方法。此方法將在 Model 刪除之前被呼叫。此方法可用於在 Model 永久從資料庫中移除之前，刪除與 Model 相關聯的任何額外資源，例如儲存的檔案：

```php
/**
 * Prepare the model for pruning.
 */
protected function pruning(): void
{
    // ...
}
```

設定完可修剪的 Model 後，您應該在應用程式的 `routes/console.php` 檔案中排程 `model:prune` Artisan 指令。您可以自由選擇此指令執行的適當時間間隔：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('model:prune')->daily();
```

在幕後，`model:prune` 指令將自動偵測應用程式 `app/Models` 目錄中的「Prunable」Model。如果您的 Model 位於不同的位置，您可以使用 `--model` 選項來指定 Model 的類別名稱：

```php
Schedule::command('model:prune', [
    '--model' => [Address::class, Flight::class],
])->daily();
```

如果您希望在修剪所有其他偵測到的 Model 時，排除某些 Model 不被修剪，您可以使用 `--except` 選項：

```php
Schedule::command('model:prune', [
    '--except' => [Address::class, Flight::class],
])->daily();
```

您可以使用 `--pretend` 選項執行 `model:prune` 指令來測試您的 `prunable` 查詢。在模擬時，`model:prune` 指令只會報告如果實際執行該指令將會修剪多少筆記錄：

```shell
php artisan model:prune --pretend
```

> [!WARNING]
> 軟刪除的 Model 如果符合修剪查詢的條件，將會被永久刪除 (`forceDelete`)。

<a name="mass-pruning"></a>
#### 批量修剪

當 Model 被標記為 `Illuminate\Database\Eloquent\MassPrunable` Trait 時，Model 會使用批量刪除查詢從資料庫中刪除。因此，`pruning` 方法將不會被呼叫，`deleting` 和 `deleted` Model 事件也不會被派發。這是因為在刪除之前 Model 從未實際被擷取，從而使修剪過程更加高效：

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
        return static::where('created_at', '<=', now()->subMonth());
    }
}
```

<a name="replicating-models"></a>
## 複製 Model

您可以使用 `replicate` 方法建立現有 Model 實例的未儲存副本。當您擁有多個共用許多相同屬性的 Model 實例時，此方法特別有用：

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

若要排除一個或多個屬性不被複製到新的 Model，您可以將一個陣列傳遞給 `replicate` 方法：

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
## 查詢 Scope


<a name="global-scopes"></a>
### 全域 Scope

全域 Scope 讓您可以為指定 Model 的所有查詢新增限制。Laravel 內建的 [軟刪除](#soft-deleting) 功能就利用全域 Scope 來僅從資料庫中擷取「未刪除」的 Model。撰寫自己的全域 Scope 可以提供一個方便、簡單的方式，來確保 Model 的每個查詢都收到特定的限制。


<a name="generating-scopes"></a>
#### 產生 Scope

若要產生新的全域 Scope，您可以呼叫 `make:scope` Artisan 指令，這會將產生的 Scope 放置在您應用程式的 `app/Models/Scopes` 目錄中：

```shell
php artisan make:scope AncientScope
```


<a name="writing-global-scopes"></a>
#### 撰寫全域 Scope

撰寫全域 Scope 很簡單。首先，使用 `make:scope` 指令產生一個實作 `Illuminate\Database\Eloquent\Scope` 介面的類別。`Scope` 介面要求您實作一個方法：`apply`。`apply` 方法可以視需要向查詢新增 `where` 限制或其他類型的子句：

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
        $builder->where('created_at', '<', now()->subYears(2000));
    }
}
```

> [!NOTE]
> 如果您的全域 Scope 要為查詢的 select 子句新增欄位，您應該使用 `addSelect` 方法而不是 `select`。這將防止意外替換查詢中現有的 select 子句。


<a name="applying-global-scopes"></a>
#### 應用全域 Scope

若要將全域 Scope 指派給 Model，您可以在 Model 上放置 `ScopedBy` 屬性：

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

或者，您可以透過覆寫 Model 的 `booted` 方法並呼叫 Model 的 `addGlobalScope` 方法來手動註冊全域 Scope。`addGlobalScope` 方法接受一個 Scope 實例作為其唯一引數：

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

在上方範例中將 Scope 新增到 `App\Models\User` Model 後，呼叫 `User::all()` 方法將執行以下 SQL 查詢：

```sql
select * from `users` where `created_at` < 0021-02-18 00:00:00
```


<a name="anonymous-global-scopes"></a>
#### 匿名全域 Scope

Eloquent 也允許您使用閉包定義全域 Scope，這對於不需要單獨類別的簡單 Scope 特別有用。當使用閉包定義全域 Scope 時，您應該提供一個自選的 Scope 名稱作為 `addGlobalScope` 方法的第一個引數：

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
            $builder->where('created_at', '<', now()->subYears(2000));
        });
    }
}
```


<a name="removing-global-scopes"></a>
#### 移除全域 Scope

如果您想為指定查詢移除全域 Scope，您可以使用 `withoutGlobalScope` 方法。此方法接受全域 Scope 的類別名稱作為其唯一引數：

```php
User::withoutGlobalScope(AncientScope::class)->get();
```

或者，如果您是使用閉包定義全域 Scope，您應該傳遞您指派給全域 Scope 的字串名稱：

```php
User::withoutGlobalScope('ancient')->get();
```

如果您想移除查詢中的多個甚至所有全域 Scope，您可以使用 `withoutGlobalScopes` 和 `withoutGlobalScopesExcept` 方法：

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
### 本地 Scope

本地 Scope 允許您定義一組常見的查詢限制，您可以在整個應用程式中輕鬆重複使用這些限制。例如，您可能需要經常擷取所有被視為「熱門」的使用者。若要定義 Scope，請將 `Scope` 屬性新增到 Eloquent 方法中。

Scope 應始終傳回相同的查詢 Builder 實例或 `void`：

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
#### 利用本地 Scope

一旦 Scope 被定義，您可以在查詢 Model 時呼叫 Scope 方法。您甚至可以鏈結呼叫各種 Scope：

```php
use App\Models\User;

$users = User::popular()->active()->orderBy('created_at')->get();
```

透過 `or` 查詢運算子組合多個 Eloquent Model Scope 可能需要使用閉包來實現正確的 [邏輯分組](/docs/{{version}}/queries#logical-grouping)：

```php
$users = User::popular()->orWhere(function (Builder $query) {
    $query->active();
})->get();
```

然而，由於這可能很繁瑣，Laravel 提供了一個「高階」`orWhere` 方法，讓您可以無需使用閉包即可流暢地鏈結 Scope：

```php
$users = User::popular()->orWhere->active()->get();
```


<a name="dynamic-scopes"></a>
#### 動態 Scope

有時您可能希望定義一個接受參數的 Scope。首先，只需將您的額外參數新增到 Scope 方法的簽章中。Scope 參數應在 `$query` 參數之後定義：

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

一旦預期的引數已新增到 Scope 方法的簽章中，您就可以在呼叫 Scope 時傳遞這些引數：

```php
$users = User::ofType('admin')->get();
```

<a name="pending-attributes"></a>
### 待處理屬性

如果您想使用 Scope 來建立與用來約束該 Scope 的 Model 擁有相同屬性的 Model，您可以在建構 Scope 查詢時使用 `withAttributes` 方法：

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

`withAttributes` 方法會使用給定的屬性將 `where` 條件新增到查詢中，並且還會將這些給定的屬性新增到透過該 Scope 建立的任何 Model 中：

```php
$draft = Post::draft()->create(['title' => 'In Progress']);

$draft->hidden; // true
```

若要指示 `withAttributes` 方法不要將 `where` 條件新增到查詢中，您可以將 `asConditions` 引數設定為 `false`：

```php
$query->withAttributes([
    'hidden' => true,
], asConditions: false);
```

<a name="comparing-models"></a>
## 比較 Model

有時候您可能需要判斷兩個 Model 是否「相同」。`is` 與 `isNot` 方法可用來快速驗證兩個 Model 是否擁有相同的主鍵、資料表及資料庫連線：

```php
if ($post->is($anotherPost)) {
    // ...
}

if ($post->isNot($anotherPost)) {
    // ...
}
```

`is` 與 `isNot` 方法在運用 `belongsTo`、`hasOne`、`morphTo` 和 `morphOne` [關聯](/docs/{{version}}/eloquent-relationships) 時也同樣適用。此方法在您想比較關聯 Model 而不發出查詢來擷取該 Model 時特別有用：

```php
if ($post->author()->is($user)) {
    // ...
}
```

<a name="events"></a>
## 事件

> [!NOTE]
> 想要將您的 Eloquent 事件直接廣播至您的用戶端應用程式嗎？請參考 Laravel 的 [Model 事件廣播](/docs/{{version}}/broadcasting#model-broadcasting)。

Eloquent Model 會派送多個事件，讓您可以介入 Model 生命週期中的以下時刻：`retrieved`、`creating`、`created`、`updating`、`updated`、`saving`、`saved`、`deleting`、`deleted`、`trashed`、`forceDeleting`、`forceDeleted`、`restoring`、`restored` 以及 `replicating`。

當現有 Model 從資料庫中擷取出來時，會派送 `retrieved` 事件。當新 Model 第一次儲存時，會派送 `creating` 和 `created` 事件。當現有 Model 被修改並呼叫 `save` 方法時，會派送 `updating` / `updated` 事件。當 Model 被建立或更新時，即使 Model 的屬性未改變，也會派送 `saving` / `saved` 事件。以 `-ing` 結尾的事件名稱會在 Model 的任何變更被永久儲存之前派送，而以 `-ed` 結尾的事件則在 Model 的變更被永久儲存之後派送。

要開始監聽 Model 事件，請在您的 Eloquent Model 上定義 `$dispatchesEvents` 屬性。此屬性會將 Eloquent Model 生命週期中的各個點對應到您自己的[事件類別](/docs/{{version}}/events)。每個 Model 事件類別都應預期透過其建構子接收受影響 Model 的實例：

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

在定義並對應您的 Eloquent 事件之後，您可以使用[事件監聽器](/docs/{{version}}/events#defining-listeners)來處理這些事件。

> [!WARNING]
> 當透過 Eloquent 發出批量更新或刪除查詢時，`saved`、`updated`、`deleting` 和 `deleted` Model 事件將不會為受影響的 Model 派送。這是因為在執行批量更新或刪除時，Model 從未實際被擷取。

<a name="events-using-closures"></a>
### 使用閉包

除了使用自訂事件類別外，您還可以註冊閉包，讓這些閉包在各種 Model 事件派送時執行。通常，您應該在 Model 的 `booted` 方法中註冊這些閉包：

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

如果需要，您可以在註冊 Model 事件時利用[可佇列的匿名事件監聽器](/docs/{{version}}/events#queuable-anonymous-event-listeners)。這將指示 Laravel 使用您應用程式的[佇列](/docs/{{version}}/queues)在背景中執行 Model 事件監聽器：

```php
use function Illuminate\Events\queueable;

static::created(queueable(function (User $user) {
    // ...
}));
```

<a name="observers"></a>
### 觀察者

<a name="defining-observers"></a>
#### 定義觀察者

如果您正在監聽一個 Model 上的許多事件，您可以使用觀察者 (observer) 將所有監聽器分組到單一類別中。觀察者類別的方法名稱會反映您希望監聽的 Eloquent 事件。每個方法都會接收受影響的 Model 作為其唯一引數。`make:observer` Artisan 指令是建立新觀察者類別最簡單的方法：

```shell
php artisan make:observer UserObserver --model=User
```

此指令會將新觀察者放置在您的 `app/Observers` 目錄中。如果此目錄不存在，Artisan 會為您建立它。您的新觀察者會如下所示：

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

要註冊觀察者，您可以將 `ObservedBy` 屬性放置在對應的 Model 上：

```php
use App\Observers\UserObserver;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy([UserObserver::class])]
class User extends Authenticatable
{
    //
}
```

或者，您可以透過呼叫您希望觀察的 Model 上的 `observe` 方法來手動註冊觀察者。您可以在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中註冊觀察者：

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
> 觀察者可以監聽額外的事件，例如 `saving` 和 `retrieved`。這些事件在[事件](#events)文件中有所說明。

<a name="observers-and-database-transactions"></a>
#### 觀察者與資料庫交易

當 Model 在資料庫交易中建立時，您可能希望指示觀察者只在資料庫交易提交之後才執行其事件處理器。您可以透過在觀察者上實作 `ShouldHandleEventsAfterCommit` 介面來達成此目的。如果資料庫交易未在進行中，事件處理器將會立即執行：

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

您有時可能需要暫時「靜音」由 Model 派送的所有事件。您可以透過使用 `withoutEvents` 方法來達成此目的。`withoutEvents` 方法接受一個閉包作為其唯一引數。在此閉包內執行的任何程式碼將不會派送 Model 事件，並且閉包返回的任何值都將由 `withoutEvents` 方法返回：

```php
use App\Models\User;

$user = User::withoutEvents(function () {
    User::findOrFail(1)->delete();

    return User::find(2);
});
```

<a name="saving-a-single-model-without-events"></a>
#### 不派送事件地儲存單一 Model

有時您可能希望在不派送任何事件的情況下「儲存」給定 Model。您可以透過使用 `saveQuietly` 方法來達成此目的：

```php
$user = User::findOrFail(1);

$user->name = 'Victoria Faith';

$user->saveQuietly();
```

您也可以在不派送任何事件的情況下「更新」、「刪除」、「軟刪除」、「還原」以及「複製」給定 Model：

```php
$user->deleteQuietly();
$user->forceDeleteQuietly();
$user->restoreQuietly();
```