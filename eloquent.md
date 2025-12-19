# Eloquent：入門

- [介紹](#introduction)
- [產生 Model 類別](#generating-model-classes)
- [Eloquent Model 慣例](#eloquent-model-conventions)
    - [資料表名稱](#table-names)
    - [主鍵](#primary-keys)
    - [UUID 與 ULID 鍵](#uuid-and-ulid-keys)
    - [時間戳記](#timestamps)
    - [資料庫連線](#database-connections)
    - [預設屬性值](#default-attribute-values)
    - [配置 Eloquent 嚴謹性](#configuring-eloquent-strictness)
- [擷取 Model](#retrieving-models)
    - [集合](#collections)
    - [分批處理結果](#chunking-results)
    - [使用 Lazy Collection 分批處理](#chunking-using-lazy-collections)
    - [游標](#cursors)
    - [進階子查詢](#advanced-subqueries)
- [擷取單一 Model / 彙總資料](#retrieving-single-models)
    - [擷取或建立 Model](#retrieving-or-creating-models)
    - [擷取彙總資料](#retrieving-aggregates)
- [插入與更新 Model](#inserting-and-updating-models)
    - [插入](#inserts)
    - [更新](#updates)
    - [大量指派](#mass-assignment)
    - [更新插入](#upserts)
- [刪除 Model](#deleting-models)
    - [軟刪除](#soft-deleting)
    - [查詢軟刪除的 Model](#querying-soft-deleted-models)
- [修剪 Model](#pruning-models)
- [複製 Model](#replicating-models)
- [查詢 Scope](#query-scopes)
    - [全域 Scope](#global-scopes)
    - [區域 Scope](#local-scopes)
    - [待定屬性](#pending-attributes)
- [比較 Model](#comparing-models)
- [事件](#events)
    - [使用閉包](#events-using-closures)
    - [觀察者](#observers)
    - [事件靜默](#muting-events)

<a name="introduction"></a>
## 介紹

Laravel 包含 Eloquent，這是一個物件關聯對映 (ORM)，讓您能愉快地與資料庫互動。使用 Eloquent 時，每個資料庫資料表都會有一個對應的「Model」來與該資料表互動。除了從資料庫資料表中擷取記錄之外，Eloquent Model 還允許您從資料表中插入、更新和刪除記錄。

> [!NOTE]
> 開始之前，請務必在應用程式的 `config/database.php` 設定檔中設定資料庫連線。有關設定資料庫的更多資訊，請查閱[資料庫設定文件](/docs/{{version}}/database#configuration)。

<a name="generating-model-classes"></a>
## 產生 Model 類別

首先，讓我們先建立一個 Eloquent Model。Model 通常位於 `app\Models` 目錄中，並會延伸 `Illuminate\Database\Eloquent\Model` 類別。您可以使用 `make:model` [Artisan 命令](/docs/{{version}}/artisan)來產生一個新的 Model：

```shell
php artisan make:model Flight
```

如果您想在產生 Model 時一併產生[資料庫遷移檔](/docs/{{version}}/migrations)，可以使用 `--migration` 或 `-m` 選項：

```shell
php artisan make:model Flight --migration
```

產生 Model 時，您還可以產生其他各種型別的類別，例如 Factory、Seeder、Policy、Controller 和表單請求 (form requests)。此外，這些選項可以組合使用，一次建立多個類別：

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

有時，僅透過瀏覽 Model 的程式碼很難判斷其所有可用的屬性與關聯。您可以改用 `model:show` Artisan 命令，它能提供 Model 所有屬性與關聯的便捷概覽：

```shell
php artisan model:show Flight
```

<a name="eloquent-model-conventions"></a>
## Eloquent Model 慣例

透過 `make:model` 命令產生的 Model 將會放置於 `app/Models` 目錄中。讓我們檢查一個基本的 Model 類別，並討論一些 Eloquent 的主要慣例：

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

在看過上述範例後，您可能已經注意到我們沒有告訴 Eloquent 哪個資料庫資料表對應我們的 `Flight` Model。按照慣例，除非明確指定其他名稱，否則類別的「snake case」複數名稱將會被用作資料表名稱。因此，在此情況下，Eloquent 將會假設 `Flight` Model 將記錄儲存於 `flights` 資料表中，而 `AirTrafficController` Model 將會把記錄儲存於 `air_traffic_controllers` 資料表中。

如果您的 Model 所對應的資料庫資料表不符合此慣例，您可以在 Model 上定義一個 `table` 屬性來手動指定 Model 的資料表名稱：

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

Eloquent 也會假設每個 Model 對應的資料庫資料表都有一個名為 `id` 的主鍵欄位。如果需要，您可以在 Model 上定義一個受保護的 `$primaryKey` 屬性來指定作為 Model 主鍵的不同欄位：

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

此外，Eloquent 假設主鍵是一個遞增的整數值，這表示 Eloquent 會自動將主鍵轉換為整數。如果您希望使用非遞增或非數字的主鍵，您必須在 Model 上定義一個設定為 `false` 的公開 `$incrementing` 屬性：

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

如果您的 Model 的主鍵不是整數，您應該在 Model 上定義一個受保護的 `$keyType` 屬性。此屬性應具有 `string` 值：

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

Eloquent 要求每個 Model 至少有一個可以用作其主鍵的唯一識別「ID」。Eloquent Model 不支援「複合式」主鍵。但是，除了資料表唯一的識別主鍵之外，您可以自由地為資料庫資料表新增其他多欄位、唯一索引。

<a name="uuid-and-ulid-keys"></a>
### UUID 與 ULID 鍵

除了使用自動遞增整數作為 Eloquent Model 的主鍵外，您也可以選擇使用 UUID。UUID 是通用唯一的英數字元識別碼，長度為 36 個字元。

如果您希望 Model 使用 UUID 鍵而非自動遞增整數鍵，您可以在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUuids` Trait。當然，您應該確保 Model 具有一個 [UUID 對應的主鍵欄位](/docs/{{version}}/migrations#column-method-uuid)：

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

預設情況下，`HasUuids` Trait 將為您的 Model 生成 ["有序" UUID](/docs/{{version}}/strings#method-str-ordered-uuid)。這些 UUID 對於索引的資料庫儲存更有效率，因為它們可以按字典序排序。

您可以透過在 Model 上定義 `newUniqueId` 方法來覆寫特定 Model 的 UUID 生成過程。此外，您還可以透過在 Model 上定義 `uniqueIds` 方法來指定哪些欄位應該接收 UUID：

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

如果您願意，您可以選擇使用「ULID」而不是 UUID。ULID 類似於 UUID；但是，它們的長度僅為 26 個字元。與有序 UUID 一樣，ULID 也可以按字典序排序以實現高效的資料庫索引。要使用 ULID，您應該在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUlids` Trait。您還應該確保 Model 具有 [ULID 對應的主鍵欄位](/docs/{{version}}/migrations#column-method-ulid)：

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

預設情況下，Eloquent 預期您的 Model 對應的資料庫資料表上存在 `created_at` 和 `updated_at` 欄位。當 Model 被建立或更新時，Eloquent 將會自動設定這些欄位的值。如果您不希望這些欄位由 Eloquent 自動管理，您應該在 Model 上定義一個值為 `false` 的 `$timestamps` 屬性：

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

如果您需要自訂 Model 時間戳記的格式，請在 Model 上設定 `$dateFormat` 屬性。此屬性決定日期屬性如何儲存於資料庫中，以及當 Model 序列化為陣列或 JSON 時的格式：

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

如果您希望執行 Model 操作時不修改 Model 的 `updated_at` 時間戳記，您可以在給定 `withoutTimestamps` 方法的閉包中對 Model 執行操作：

```php
Model::withoutTimestamps(fn () => $post->increment('reads'));
```

<a name="database-connections"></a>
### 資料庫連線

預設情況下，所有 Eloquent Model 都會使用您應用程式配置的預設資料庫連線。如果您希望指定與特定 Model 互動時應使用的不同連線，您應該在 Model 上定義一個 `$connection` 屬性：

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

預設情況下，新建立的 Model 實例不會包含任何屬性值。如果您想定義 Model 某些屬性的預設值，您可以在 Model 上定義 `$attributes` 屬性。放置在 `$attributes` 陣列中的屬性值應該是原始的、「可儲存」的格式，就像它們剛從資料庫讀取出來一樣：

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
### 配置 Eloquent 嚴謹性

Laravel 提供了幾種方法，讓您可以在各種情況下配置 Eloquent 的行為與「嚴謹性」。

首先，`preventLazyLoading` 方法接受一個選用性的布林參數，指示是否應阻止延遲載入 (lazy loading)。例如，您可能只希望在非正式環境中禁用延遲載入，這樣即使在正式環境的程式碼中不小心出現延遲載入的關係，您的正式環境也能繼續正常運作。通常，此方法應在您應用程式的 `AppServiceProvider` 中的 `boot` 方法裡被調用：

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

此外，您可以透過調用 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填充無法填充的屬性時拋出例外。這有助於在本地開發期間，當嘗試設定尚未新增到 Model 的 `fillable` 陣列中的屬性時，防止發生意外錯誤：

```php
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

<a name="retrieving-models"></a>
## 擷取 Model

一旦你建立了一個 Model 及其[關聯的資料庫資料表](/docs/{{version}}/migrations#generating-migrations)，你就可以開始從資料庫中擷取資料了。你可以將每個 Eloquent Model 視為一個強大的[查詢產生器](/docs/{{version}}/queries)，它讓你可以流暢地查詢與 Model 關聯的資料庫資料表。Model 的 `all` 方法將會擷取 Model 關聯資料庫資料表中的所有記錄：

```php
use App\Models\Flight;

foreach (Flight::all() as $flight) {
    echo $flight->name;
}
```

<a name="building-queries"></a>
#### 建立查詢

Eloquent 的 `all` 方法會回傳 Model 資料表中的所有結果。然而，由於每個 Eloquent Model 都作為一個[查詢產生器](/docs/{{version}}/queries)，你可以為查詢增加額外的限制，然後呼叫 `get` 方法來擷取結果：

```php
$flights = Flight::where('active', 1)
    ->orderBy('name')
    ->limit(10)
    ->get();
```

> [!NOTE]
> 由於 Eloquent Model 也是查詢產生器，你應該檢閱 Laravel [查詢產生器](/docs/{{version}}/queries)提供的所有方法。在撰寫 Eloquent 查詢時，你可以使用其中任何方法。

<a name="refreshing-models"></a>
#### 重新整理 Model

如果你已經有一個從資料庫中擷取的 Eloquent Model 實例，你可以使用 `fresh` 和 `refresh` 方法來「重新整理」Model。`fresh` 方法將會從資料庫中重新擷取 Model。現有的 Model 實例不會受到影響：

```php
$flight = Flight::where('number', 'FR 900')->first();

$freshFlight = $flight->fresh();
```

`refresh` 方法將會使用來自資料庫的最新資料來重新建立現有的 Model。此外，所有已載入的關聯也將會重新整理：

```php
$flight = Flight::where('number', 'FR 900')->first();

$flight->number = 'FR 456';

$flight->refresh();

$flight->number; // "FR 900"
```

<a name="collections"></a>
### 集合

正如我們所見，Eloquent 方法如 `all` 和 `get` 會從資料庫中擷取多條記錄。然而，這些方法並不會回傳一個普通的 PHP 陣列。相反地，它會回傳一個 `Illuminate\Database\Eloquent\Collection` 的實例。

Eloquent `Collection` 類別繼承了 Laravel 的基礎 `Illuminate\Support\Collection` 類別，該類別提供了[多種實用的方法](/docs/{{version}}/collections#available-methods)來與資料集合互動。例如，`reject` 方法可以用來根據所呼叫閉包的結果，從集合中移除 Model：

```php
$flights = Flight::where('destination', 'Paris')->get();

$flights = $flights->reject(function (Flight $flight) {
    return $flight->cancelled;
});
```

除了 Laravel 基礎集合類別提供的方法之外，Eloquent 集合類別還提供了[一些額外的方法](/docs/{{version}}/eloquent-collections#available-methods)，這些方法專門用於與 Eloquent Model 集合互動。

由於 Laravel 的所有集合都實現了 PHP 的可迭代介面，你可以像遍歷陣列一樣遍歷集合：

```php
foreach ($flights as $flight) {
    echo $flight->name;
}
```

<a name="chunking-results"></a>
### 分批處理結果

如果你嘗試透過 `all` 或 `get` 方法載入數萬條 Eloquent 記錄，你的應用程式可能會耗盡記憶體。為了避免使用這些方法，`chunk` 方法可以用來更有效地處理大量的 Model。

`chunk` 方法將會擷取 Eloquent Model 的子集，並將它們傳遞給一個閉包進行處理。由於每次只會擷取當前的 Eloquent Model 區塊，因此在使用大量 Model 時，`chunk` 方法將會顯著減少記憶體使用量：

```php
use App\Models\Flight;
use Illuminate\Database\Eloquent\Collection;

Flight::chunk(200, function (Collection $flights) {
    foreach ($flights as $flight) {
        // ...
    }
});
```

傳遞給 `chunk` 方法的第一個參數是你希望每個「區塊」接收的記錄數。作為第二個參數傳遞的閉包將會針對從資料庫中擷取的每個區塊被呼叫。每次擷取傳遞給閉包的記錄區塊時，都會執行一次資料庫查詢。

如果你正在根據一個欄位過濾 `chunk` 方法的結果，並且在遍歷結果時也會更新該欄位，你應該使用 `chunkById` 方法。在這些情況下使用 `chunk` 方法可能會導致意外和不一致的結果。在內部，`chunkById` 方法將會始終擷取 `id` 欄位大於前一個區塊中最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->chunkById(200, function (Collection $flights) {
        $flights->each->update(['departed' => false]);
    }, column: 'id');
```

由於 `chunkById` 和 `lazyById` 方法會為正在執行的查詢添加自己的「where」條件，你通常應該在閉包中[邏輯分組](/docs/{{version}}/queries#logical-grouping)自己的條件：

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
### 使用 Lazy Collection 分批處理

`lazy` 方法的工作方式與[ `chunk` 方法](#chunking-results)類似，從底層來看，它會分批執行查詢。然而，`lazy` 方法並不會直接將每個區塊傳遞給回呼，而是回傳一個扁平化的 Eloquent Model [LazyCollection](/docs/{{version}}/collections#lazy-collections)，讓你可以像單一串流一樣與結果互動：

```php
use App\Models\Flight;

foreach (Flight::lazy() as $flight) {
    // ...
}
```

如果你正在根據一個欄位過濾 `lazy` 方法的結果，並且在遍歷結果時也會更新該欄位，你應該使用 `lazyById` 方法。在內部，`lazyById` 方法將會始終擷取 `id` 欄位大於前一個區塊中最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->lazyById(200, column: 'id')
    ->each->update(['departed' => false]);
```

你可以使用 `lazyByIdDesc` 方法，根據 `id` 的降序來過濾結果。

<a name="cursors"></a>
### 游標

與 `lazy` 方法類似，`cursor` 方法可用於在迭代數萬筆 Eloquent Model 記錄時，顯著減少應用程式的記憶體消耗。

「`cursor` 方法」只會執行一次資料庫查詢；然而，個別的 Eloquent Model 在實際迭代之前不會被具現化 (hydrated)。因此，在迭代游標時，記憶體中只會保留一個 Eloquent Model。

> [!WARNING]
> 由於 `cursor` 方法在記憶體中一次只會保留一個 Eloquent Model，因此它無法預載入 (eager load) 關聯。如果您需要預載入關聯，請考慮使用[「`lazy` 方法」](#chunking-using-lazy-collections)。

在內部，「`cursor` 方法」使用 PHP [生成器](https://www.php.net/manual/en/language.generators.overview.php)來實現此功能：

```php
use App\Models\Flight;

foreach (Flight::where('destination', 'Zurich')->cursor() as $flight) {
    // ...
}
```

「`cursor`」回傳一個 `Illuminate\Support\LazyCollection` 實例。[Lazy Collection](/docs/{{version}}/collections#lazy-collections) 允許您使用典型 Laravel 集合上的許多集合方法，同時一次只載入一個 Model 到記憶體中：

```php
use App\Models\User;

$users = User::cursor()->filter(function (User $user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```

儘管「`cursor` 方法」比一般查詢使用的記憶體少得多 (因為它一次只在記憶體中保留一個 Eloquent Model)，但它最終仍會耗盡記憶體。這是[由於 PHP 的 PDO 驅動程式內部會將所有原始查詢結果快取在其緩衝區中](https://www.php.net/manual/en/mysqlinfo.concepts.buffering.php)。如果您正在處理大量 Eloquent 記錄，請考慮使用[「`lazy` 方法」](#chunking-using-lazy-collections)來代替。

<a name="advanced-subqueries"></a>
### 進階子查詢

<a name="subquery-selects"></a>
#### 子查詢選擇

Eloquent 還提供進階的子查詢支援，讓您可以透過單一查詢從關聯資料表中提取資訊。例如，假設我們有一個航班目的地 `destinations` 資料表和一個飛往目的地的 `flights` 資料表。「`flights` 資料表」包含一個 `arrived_at` 欄位，表示航班抵達目的地的時間。

利用查詢建構器 (query builder) 的 `select` 和 `addSelect` 方法提供的子查詢功能，我們可以使用單一查詢選取所有 `destinations` (目的地) 和最近抵達該目的地的航班名稱：

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

此外，查詢建構器 (query builder) 的 `orderBy` 函數支援子查詢。繼續使用我們的航班範例，我們可以利用此功能，根據最後一班航班抵達該目的地的時間來排序所有目的地。同樣地，這可以在執行單一資料庫查詢時完成：

```php
return Destination::orderByDesc(
    Flight::select('arrived_at')
        ->whereColumn('destination_id', 'destinations.id')
        ->orderByDesc('arrived_at')
        ->limit(1)
)->get();
```

<a name="retrieving-single-models"></a>
## 擷取單一 Model / 彙總資料

除了擷取符合給定查詢的所有記錄外，您還可以使用 `find`、`first` 或 `firstWhere` 方法來擷取單一記錄。這些方法不會回傳 model 的集合，而是回傳單一 model 實例：

```php
use App\Models\Flight;

// Retrieve a model by its primary key...
$flight = Flight::find(1);

// Retrieve the first model matching the query constraints...
$flight = Flight::where('active', 1)->first();

// Alternative to retrieving the first model matching the query constraints...
$flight = Flight::firstWhere('active', 1);
```

有時您可能希望在沒有找到結果時執行其他操作。`findOr` 和 `firstOr` 方法將回傳單一 model 實例，如果沒有找到結果，則會執行給定的閉包。閉包所回傳的值將被視為該方法的結果：

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

有時，您可能希望在找不到 model 時拋出例外。這在路由或控制器中特別有用。`findOrFail` 和 `firstOrFail` 方法將擷取查詢的第一個結果；然而，如果沒有找到結果，將會拋出 `Illuminate\Database\Eloquent\ModelNotFoundException` 例外：

```php
$flight = Flight::findOrFail(1);

$flight = Flight::where('legs', '>', 3)->firstOrFail();
```

如果 `ModelNotFoundException` 未被捕捉，則會自動向客戶端回傳 404 HTTP 回應：

```php
use App\Models\Flight;

Route::get('/api/flights/{id}', function (string $id) {
    return Flight::findOrFail($id);
});
```


<a name="retrieving-or-creating-models"></a>
### 擷取或建立 Model

`firstOrCreate` 方法將嘗試使用給定的欄位/值對來定位資料庫記錄。如果資料庫中找不到 model，將插入一條記錄，其屬性是將第一個陣列引數與可選的第二個陣列引數合併後的結果。

`firstOrNew` 方法與 `firstOrCreate` 類似，它將嘗試在資料庫中定位符合給定屬性的記錄。然而，如果找不到 model，將回傳一個新的 model 實例。請注意，`firstOrNew` 回傳的 model 尚未被持久化到資料庫中。您需要手動呼叫 `save` 方法來持久化它：

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
### 擷取彙總資料

在與 Eloquent model 互動時，您還可以使用 Laravel [查詢建構器](/docs/{{version}}/queries)提供的 `count`、`sum`、`max` 和其他[彙總方法](/docs/{{version}}/queries#aggregates)。正如您所預期的，這些方法回傳的是一個純量值，而不是 Eloquent model 實例：

```php
$count = Flight::where('active', 1)->count();

$max = Flight::where('active', 1)->max('price');
```

<a name="inserting-and-updating-models"></a>
## 插入與更新 Model


<a name="inserts"></a>
### 插入

當然，在使用 Eloquent 時，我們不只需要從資料庫中擷取 Model。我們也需要插入新的紀錄。值得慶幸的是，Eloquent 讓這一切變得簡單。若要插入一筆新的資料到資料庫中，您應該先實例化一個新的 Model 實例，並設定 Model 的屬性。然後，呼叫該 Model 實例的 `save` 方法：

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

在此範例中，我們將傳入 HTTP 請求的 `name` 欄位指派給 `App\Models\Flight` Model 實例的 `name` 屬性。當我們呼叫 `save` 方法時，一筆紀錄將會被插入到資料庫中。當 `save` 方法被呼叫時，Model 的 `created_at` 和 `updated_at` 時間戳記將會自動設定，因此不需要手動設定它們。

此外，您可以使用 `create` 方法，透過單一 PHP 陳述式來「儲存」一個新的 Model。`create` 方法將會回傳已插入的 Model 實例給您：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，您需要為 Model 類別指定 `fillable` 或 `guarded` 屬性。這些屬性是必要的，因為所有 Eloquent Model 預設都受到大量指派 (mass assignment) 漏洞的保護。若要了解更多關於大量指派的資訊，請參閱 [大量指派文件](#mass-assignment)。


<a name="updates"></a>
### 更新

`save` 方法也可以用於更新資料庫中已存在的 Model。若要更新一個 Model，您應該先擷取它，然後設定您想更新的任何屬性。接著，您應該呼叫 Model 的 `save` 方法。同樣地，`updated_at` 時間戳記將會自動更新，因此不需要手動設定其值：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->name = 'Paris to London';

$flight->save();
```

有時候，您可能需要更新一個已存在的 Model，或者在沒有匹配 Model 時建立一個新的 Model。與 `firstOrCreate` 方法類似，`updateOrCreate` 方法會持久化 Model，因此不需要手動呼叫 `save` 方法。

在以下範例中，如果存在一個 `departure` 地點為 `Oakland` 且 `destination` 地點為 `San Diego` 的航班，其 `price` 和 `discounted` 欄位將會被更新。如果不存在這樣的航班，則會建立一個新的航班，其屬性是將第一個引數陣列與第二個引數陣列合併後的結果：

```php
$flight = Flight::updateOrCreate(
    ['departure' => 'Oakland', 'destination' => 'San Diego'],
    ['price' => 99, 'discounted' => 1]
);
```

當使用 `firstOrCreate` 或 `updateOrCreate` 等方法時，您可能不知道是建立了一個新的 Model 還是更新了一個已存在的 Model。`wasRecentlyCreated` 屬性會指示 Model 是否在其當前生命週期內被建立：

```php
$flight = Flight::updateOrCreate(
    // ...
);

if ($flight->wasRecentlyCreated) {
    // New flight record was inserted...
}
```


<a name="mass-updates"></a>
#### 大量更新

更新也可以針對符合特定查詢的 Model 執行。在此範例中，所有 `active` 且 `destination` 為 `San Diego` 的航班將會被標記為延遲：

```php
Flight::where('active', 1)
    ->where('destination', 'San Diego')
    ->update(['delayed' => 1]);
```

`update` 方法預期一個包含欄位與值對的陣列，代表應更新的欄位。`update` 方法會回傳受影響的行數。

> [!WARNING]
> 當透過 Eloquent 發出大量更新時，`saving`、`saved`、`updating` 和 `updated` 等 Model 事件將不會針對已更新的 Model 觸發。這是因為在執行大量更新時，Model 從未被實際擷取。


<a name="examining-attribute-changes"></a>
#### 檢查屬性變更

Eloquent 提供 `isDirty`、`isClean` 和 `wasChanged` 方法，以檢查 Model 的內部狀態，並判斷其屬性自 Model 最初被擷取以來發生了哪些變化。

`isDirty` 方法會判斷 Model 的任何屬性是否自 Model 擷取以來已被修改。您可以傳遞特定的屬性名稱或屬性陣列給 `isDirty` 方法，以判斷這些屬性中是否有任何一個是「變更過的 (dirty)」。`isClean` 方法會判斷某個屬性是否自 Model 擷取以來保持未變。此方法也接受一個可選的屬性引數：

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

`wasChanged` 方法會判斷在當前請求生命週期內，Model 上次儲存時是否有任何屬性被變更。如有需要，您可以傳遞屬性名稱來查看特定屬性是否已變更：

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

無論 Model 自擷取以來發生了任何變化，`getOriginal` 方法都會回傳一個包含 Model 原始屬性的陣列。如有需要，您可以傳遞特定的屬性名稱來取得特定屬性的原始值：

```php
$user = User::find(1);

$user->name; // John
$user->email; // john@example.com

$user->name = 'Jack';
$user->name; // Jack

$user->getOriginal('name'); // John
$user->getOriginal(); // Array of original attributes...
```

`getChanges` 方法會回傳一個包含 Model 上次儲存時變更的屬性陣列，而 `getPrevious` 方法則會回傳一個包含 Model 上次儲存前的原始屬性值陣列：

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
### 大量指派

您可以使用 `create` 方法，以單一 PHP 敘述「儲存」一個新的 Model。此方法會回傳新插入的 Model 實例：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，您需要先在 Model 類別上指定 `fillable` 或 `guarded` 屬性。這些屬性是必要的，因為所有 Eloquent Model 預設都受到大量指派漏洞的保護。

當使用者傳入非預期的 HTTP 請求欄位，且該欄位更改了您不希望被更改的資料庫欄位時，就會發生大量指派漏洞。舉例來說，惡意使用者可能會透過 HTTP 請求傳送 `is_admin` 參數，然後該參數被傳遞到 Model 的 `create` 方法中，允許該使用者將自己提升為管理員。

因此，首先您應該定義哪些 Model 屬性允許大量指派。您可以使用 Model 上的 `$fillable` 屬性來達成此目的。例如，讓 `Flight` Model 的 `name` 屬性允許大量指派：

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

一旦您指定了哪些屬性允許大量指派，您就可以使用 `create` 方法在資料庫中插入新記錄。`create` 方法會回傳新建立的 Model 實例：

```php
$flight = Flight::create(['name' => 'London to Paris']);
```

如果您已經有一個 Model 實例，您可以使用 `fill` 方法來填充其屬性陣列：

```php
$flight->fill(['name' => 'Amsterdam to Frankfurt']);
```


<a name="mass-assignment-json-columns"></a>
#### 大量指派與 JSON 欄位

在指派 JSON 欄位時，每個欄位的可大量指派鍵必須在 Model 的 `$fillable` 陣列中指定。為了安全性，Laravel 在使用 `guarded` 屬性時不支援更新巢狀 JSON 屬性：

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
#### 允許大量指派

如果您想讓所有屬性都允許大量指派，您可以將 Model 的 `$guarded` 屬性定義為一個空陣列。如果您選擇取消保護 Model，則應該特別注意始終手動建構傳遞給 Eloquent 的 `fill`、`create` 和 `update` 方法的陣列：

```php
/**
 * The attributes that aren't mass assignable.
 *
 * @var array<string>|bool
 */
protected $guarded = [];
```


<a name="mass-assignment-exceptions"></a>
#### 大量指派例外

預設情況下，執行大量指派操作時，未包含在 `$fillable` 陣列中的屬性會被靜默丟棄。在正式環境中，這是預期的行為；然而，在本地開發期間，這可能會導致 Model 更改未生效而產生困惑。

如果您希望，可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填充不可填充的屬性時拋出例外。通常，此方法應在應用程式 `AppServiceProvider` 類別的 `boot` 方法中調用：

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
### 更新插入

Eloquent 的 `upsert` 方法可用於在單一原子操作中更新或建立記錄。該方法的第一個引數包含要插入或更新的值，而第二個引數列出了在關聯資料表中唯一識別記錄的欄位。該方法的第三個也是最後一個引數是一個陣列，包含如果資料庫中已存在匹配記錄時應更新的欄位。如果 Model 上啟用了時間戳記，`upsert` 方法會自動設定 `created_at` 和 `updated_at` 時間戳記：

```php
Flight::upsert([
    ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
    ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
], uniqueBy: ['departure', 'destination'], update: ['price']);
```

> [!WARNING]
> 除 SQL Server 外的所有資料庫都要求 `upsert` 方法第二個引數中的欄位必須具備「主鍵」或「唯一」索引。此外，MariaDB 和 MySQL 資料庫驅動會忽略 `upsert` 方法的第二個引數，並始終使用資料表的「主鍵」和「唯一」索引來檢測現有記錄。

<a name="deleting-models"></a>
## 刪除 Model

要刪除 Model，您可以在 Model 實例上呼叫 `delete` 方法：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->delete();
```


<a name="deleting-an-existing-model-by-its-primary-key"></a>
#### 透過主鍵刪除現有 Model

在上述範例中，我們在呼叫 `delete` 方法之前從資料庫中擷取 Model。然而，如果您知道 Model 的主鍵，您可以透過呼叫 `destroy` 方法，不需明確擷取 Model 即可刪除。除了接受單一主鍵外，`destroy` 方法還可接受多個主鍵、主鍵陣列，或是主鍵的 [collection](/docs/{{version}}/collections)：

```php
Flight::destroy(1);

Flight::destroy(1, 2, 3);

Flight::destroy([1, 2, 3]);

Flight::destroy(collect([1, 2, 3]));
```

如果您正在使用 [軟刪除 Model](#soft-deleting)，您可以透過 `forceDestroy` 方法永久刪除 Model：

```php
Flight::forceDestroy(1);
```

> [!WARNING]
> `destroy` 方法會單獨載入每個 Model 並呼叫 `delete` 方法，以便為每個 Model 正確地派發 `deleting` 和 `deleted` 事件。


<a name="deleting-models-using-queries"></a>
#### 透過查詢刪除 Model

當然，您可以建立一個 Eloquent 查詢來刪除所有符合查詢條件的 Model。在此範例中，我們將刪除所有標記為非活動狀態的航班。如同大量更新，大量刪除將不會為被刪除的 Model 派發 Model 事件：

```php
$deleted = Flight::where('active', 0)->delete();
```

若要刪除資料表中的所有 Model，您應該執行一個不帶任何條件的查詢：

```php
$deleted = Flight::query()->delete();
```

> [!WARNING]
> 當透過 Eloquent 執行大量刪除語句時，`deleting` 和 `deleted` Model 事件將不會為被刪除的 Model 派發。這是因為在執行刪除語句時，Model 從未實際被擷取。


<a name="soft-deleting"></a>
### 軟刪除

除了實際從資料庫中移除記錄之外，Eloquent 還可以「軟刪除」Model。當 Model 被軟刪除時，它們並未實際從資料庫中移除。相反地，Model 上會設定一個 `deleted_at` 屬性，指示 Model 被「刪除」的日期和時間。要為 Model 啟用軟刪除，請將 `Illuminate\Database\Eloquent\SoftDeletes` Trait 加入 Model：

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
> `SoftDeletes` Trait 將自動為您將 `deleted_at` 屬性轉換為 `DateTime` / `Carbon` 實例。

您還應該在資料庫資料表中新增 `deleted_at` 欄位。Laravel 的 [schema builder](/docs/{{version}}/migrations) 包含一個輔助方法來建立此欄位：

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

現在，當您在 Model 上呼叫 `delete` 方法時，`deleted_at` 欄位將被設定為目前的日期和時間。然而，Model 的資料庫記錄將會保留在資料表中。當查詢使用軟刪除的 Model 時，軟刪除的 Model 將自動從所有查詢結果中排除。

若要判斷給定的 Model 實例是否已被軟刪除，您可以使用 `trashed` 方法：

```php
if ($flight->trashed()) {
    // ...
}
```


<a name="restoring-soft-deleted-models"></a>
#### 復原軟刪除的 Model

有時您可能希望「取消刪除」一個軟刪除的 Model。若要復原一個軟刪除的 Model，您可以在 Model 實例上呼叫 `restore` 方法。`restore` 方法會將 Model 的 `deleted_at` 欄位設定為 `null`：

```php
$flight->restore();
```

您也可以在查詢中使用 `restore` 方法來復原多個 Model。同樣地，如同其他「大量」操作，這將不會為被復原的 Model 派發任何 Model 事件：

```php
Flight::withTrashed()
    ->where('airline_id', 1)
    ->restore();
```

`restore` 方法也可以在建構 [關聯](/docs/{{version}}/eloquent-relationships) 查詢時使用：

```php
$flight->history()->restore();
```


<a name="permanently-deleting-models"></a>
#### 永久刪除 Model

有時您可能需要真正地從資料庫中移除 Model。您可以使用 `forceDelete` 方法從資料庫資料表中永久移除軟刪除的 Model：

```php
$flight->forceDelete();
```

您也可以在建構 Eloquent 關聯查詢時使用 `forceDelete` 方法：

```php
$flight->history()->forceDelete();
```


<a name="querying-soft-deleted-models"></a>
### 查詢軟刪除的 Model


<a name="including-soft-deleted-models"></a>
#### 包含軟刪除的 Model

如前所述，軟刪除的 Model 將自動從查詢結果中排除。然而，您可以透過在查詢上呼叫 `withTrashed` 方法，強制將軟刪除的 Model 包含在查詢結果中：

```php
use App\Models\Flight;

$flights = Flight::withTrashed()
    ->where('account_id', 1)
    ->get();
```

`withTrashed` 方法也可以在建構 [關聯](/docs/{{version}}/eloquent-relationships) 查詢時使用：

```php
$flight->history()->withTrashed()->get();
```


<a name="retrieving-only-soft-deleted-models"></a>
#### 只擷取軟刪除的 Model

`onlyTrashed` 方法將**只**擷取軟刪除的 Model：

```php
$flights = Flight::onlyTrashed()
    ->where('airline_id', 1)
    ->get();
```

<a name="pruning-models"></a>
## 修剪 Model

有時候，您可能會想要定期刪除不再需要的 Model。為此，您可以將 `Illuminate\Database\Eloquent\Prunable` 或 `Illuminate\Database\Eloquent\MassPrunable` Trait 加入到您想要定期修剪的 Model 中。將其中一個 Trait 加入 Model 後，實作一個 `prunable` 方法，此方法會回傳一個 Eloquent 查詢建構器，用於解析不再需要的 Model：

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

將 Model 標記為 `Prunable` 時，您也可以在 Model 中定義一個 `pruning` 方法。這個方法會在 Model 刪除之前被呼叫。這個方法對於在 Model 從資料庫中永久移除之前，刪除任何與該 Model 相關聯的額外資源 (例如儲存的檔案) 非常有用：

```php
/**
 * Prepare the model for pruning.
 */
protected function pruning(): void
{
    // ...
}
```

設定好可修剪的 Model 後，您應該在應用程式的 `routes/console.php` 檔案中安排 `model:prune` Artisan 命令。您可以自由選擇此命令應執行的適當間隔：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('model:prune')->daily();
```

在幕後，`model:prune` 命令會自動偵測應用程式 `app/Models` 目錄中的「Prunable」Model。如果您的 Model 位於不同的位置，您可以使用 `--model` 選項來指定 Model 類別名稱：

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

您可以透過執行帶有 `--pretend` 選項的 `model:prune` 命令來測試您的 `prunable` 查詢。在模擬模式下，`model:prune` 命令只會報告如果實際執行該命令會修剪多少筆記錄：

```shell
php artisan model:prune --pretend
```

> [!WARNING]
> 軟刪除的 Model 如果符合修剪查詢的條件，將會被永久刪除 (`forceDelete`)。

<a name="mass-pruning"></a>
#### 批量修剪

當 Model 被標記為 `Illuminate\Database\Eloquent\MassPrunable` Trait 時，Model 會使用批量刪除查詢從資料庫中刪除。因此，`pruning` 方法將不會被呼叫，`deleting` 和 `deleted` Model 事件也不會被分派。這是因為在刪除之前 Model 從未實際被擷取，從而使修剪過程更加高效：

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
## 複製 Model

您可以使用 `replicate` 方法來建立現有 Model 實例的未儲存副本。當您有多個 Model 實例共享許多相同屬性時，這個方法特別有用：

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

全域 Scope 讓您可以針對給定 model 的所有查詢加入限制。Laravel 自帶的 [軟刪除](#soft-deleting) 功能便利用全域 Scope，以便僅從資料庫中擷取「未刪除」的 model。編寫自己的全域 Scope 可以提供一種方便、簡單的方式，確保針對給定 model 的每個查詢都接收到特定的限制。


<a name="generating-scopes"></a>
#### 產生 Scope

若要產生新的全域 Scope，您可以使用 `make:scope` Artisan 命令，該命令會將產生的 Scope 放在應用程式的 `app/Models/Scopes` 目錄中：

```shell
php artisan make:scope AncientScope
```


<a name="writing-global-scopes"></a>
#### 編寫全域 Scope

編寫全域 Scope 很簡單。首先，使用 `make:scope` 命令產生一個實作 `Illuminate\Database\Eloquent\Scope` 介面的類別。`Scope` 介面要求您實作一個方法：`apply`。`apply` 方法可以根據需要，為查詢加入 `where` 限制或其他類型的子句：

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
> 如果您的全域 Scope 正在為查詢的 select 子句加入欄位，您應該使用 `addSelect` 方法而不是 `select`。這將防止查詢現有的 select 子句被意外替換。


<a name="applying-global-scopes"></a>
#### 應用全域 Scope

若要將全域 Scope 指派給 model，您可以簡單地將 `ScopedBy` attribute 放置在 model 上：

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

或者，您可以透過覆寫 model 的 `booted` 方法並呼叫 model 的 `addGlobalScope` 方法來手動註冊全域 Scope。`addGlobalScope` 方法只接受您的 Scope 實例作為其唯一引數：

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

在將上述範例中的 Scope 加入 `App\Models\User` model 後，呼叫 `User::all()` 方法將執行以下 SQL 查詢：

```sql
select * from `users` where `created_at` < 0021-02-18 00:00:00
```


<a name="anonymous-global-scopes"></a>
#### 匿名全域 Scope

Eloquent 也允許您使用閉包定義全域 Scope，這對於不需要單獨類別的簡單 Scope 尤其有用。當使用閉包定義全域 Scope 時，您應該提供一個自訂的 Scope 名稱作為 `addGlobalScope` 方法的第一個引數：

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
#### 移除全域 Scope

如果您想移除針對給定查詢的全域 Scope，您可以使用 `withoutGlobalScope` 方法。此方法只接受全域 Scope 的類別名稱作為其唯一引數：

```php
User::withoutGlobalScope(AncientScope::class)->get();
```

或者，如果您使用閉包定義全域 Scope，您應該傳遞您為全域 Scope 指派的字串名稱：

```php
User::withoutGlobalScope('ancient')->get();
```

如果您想移除查詢中的數個甚至所有全域 Scope，您可以使用 `withoutGlobalScopes` 和 `withoutGlobalScopesExcept` 方法：

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
### 區域 Scope

區域 Scope 允許您定義可在應用程式中輕鬆重複使用的常見查詢限制集合。例如，您可能需要頻繁擷取所有被認為是「熱門」的使用者。若要定義 Scope，請將 `Scope` attribute 加入 Eloquent 方法。

Scope 應始終傳回相同的查詢建構器實例或 `void`：

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
#### 利用區域 Scope

一旦定義了 Scope，您可以在查詢 model 時呼叫 Scope 方法。您甚至可以將對多個 Scope 的呼叫串聯起來：

```php
use App\Models\User;

$users = User::popular()->active()->orderBy('created_at')->get();
```

透過 `or` 查詢運算子結合多個 Eloquent model Scope 時，可能需要使用閉包來實現正確的 [邏輯分組](/docs/{{version}}/queries#logical-grouping)：

```php
$users = User::popular()->orWhere(function (Builder $query) {
    $query->active();
})->get();
```

然而，由於這可能很繁瑣，Laravel 提供了一個「高階」`orWhere` 方法，讓您可以流暢地串聯 Scope，無需使用閉包：

```php
$users = User::popular()->orWhere->active()->get();
```


<a name="dynamic-scopes"></a>
#### 動態 Scope

有時您可能希望定義一個接受參數的 Scope。首先，只需將您的額外參數加入您的 Scope 方法簽章中。Scope 參數應定義在 `$query` 參數之後：

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

一旦預期的引數已加入您的 Scope 方法簽章中，您可以在呼叫 Scope 時傳遞這些引數：

```php
$users = User::ofType('admin')->get();
```

<a name="pending-attributes"></a>
### 待定屬性

如果您想使用 scope 來建立具有與限制該 scope 相同的屬性的 Model，您可以在建構 scope 查詢時使用 `withAttributes` 方法：

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

`withAttributes` 方法會使用給定的屬性向查詢新增 `where` 條件，並且也會將給定的屬性新增到透過 scope 建立的任何 Model 中：

```php
$draft = Post::draft()->create(['title' => 'In Progress']);

$draft->hidden; // true
```

若要指示 `withAttributes` 方法不要向查詢新增 `where` 條件，您可以將 `asConditions` 引數設為 `false`：

```php
$query->withAttributes([
    'hidden' => true,
], asConditions: false);
```

<a name="comparing-models"></a>
## 比較 Model

有時候，您可能需要判斷兩個 Model 是否「相同」。`is` 與 `isNot` 方法可用來快速驗證兩個 Model 是否擁有相同的主鍵、資料表和資料庫連線：

```php
if ($post->is($anotherPost)) {
    // ...
}

if ($post->isNot($anotherPost)) {
    // ...
}
```

`is` 與 `isNot` 方法也可用於 `belongsTo`、`hasOne`、`morphTo` 和 `morphOne` 等 [關聯](/docs/{{version}}/eloquent-relationships)。當您想在不發出查詢來擷取關聯 Model 的情況下比較它們時，這個方法會特別有用：

```php
if ($post->author()->is($user)) {
    // ...
}
```

<a name="events"></a>
## 事件

> [!NOTE]
> 想要將您的 Eloquent 事件直接廣播至客戶端應用程式？請查看 Laravel 的[模型事件廣播](/docs/{{version}}/broadcasting#model-broadcasting)。

Eloquent Model 會分派數個事件，讓您能夠掛接到 Model 生命週期中的以下時刻：`retrieved`、`creating`、`created`、`updating`、`updated`、`saving`、`saved`、`deleting`、`deleted`、`trashed`、`forceDeleting`、`forceDeleted`、`restoring`、`restored` 以及 `replicating`。

`retrieved` 事件會在從資料庫中擷取現有 Model 時分派。當新的 Model 首次儲存時，會分派 `creating` 和 `created` 事件。當現有 Model 被修改並呼叫 `save` 方法時，會分派 `updating` / `updated` 事件。當 Model 建立或更新時，即使 Model 的屬性未變更，也會分派 `saving` / `saved` 事件。以 `-ing` 結尾的事件會在 Model 的任何變更被持久化之前分派，而以 `-ed` 結尾的事件會在 Model 的變更被持久化之後分派。

要開始監聽 Model 事件，請在您的 Eloquent Model 上定義一個 `$dispatchesEvents` 屬性。此屬性將 Eloquent Model 生命週期的各個點對應到您自己的[事件類別](/docs/{{version}}/events)。每個 Model 事件類別都應預期透過其建構函式接收受影響 Model 的實例：

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

在定義並映射您的 Eloquent 事件之後，您可以使用[事件監聽器](/docs/{{version}}/events#defining-listeners)來處理這些事件。

> [!WARNING]
> 當透過 Eloquent 發出大量更新或刪除查詢時，受影響 Model 的 `saved`、`updated`、`deleting` 和 `deleted` 模型事件將不會被分派。這是因為在執行大量更新或刪除時，Model 從未實際被擷取。

<a name="events-using-closures"></a>
### 使用閉包

除了使用自訂事件類別之外，您還可以註冊在分派各種 Model 事件時執行的閉包。通常，您應該在 Model 的 `booted` 方法中註冊這些閉包：

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

如果需要，您可以在註冊 Model 事件時使用[可排入佇列的匿名事件監聽器](/docs/{{version}}/events#queuable-anonymous-event-listeners)。這將指示 Laravel 使用您的應用程式[佇列](/docs/{{version}}/queues)在背景執行 Model 事件監聽器：

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

如果您監聽許多 Model 上的事件，可以使用觀察者 (Observers) 將所有監聽器分組到一個類別中。觀察者類別的方法名稱會反映您希望監聽的 Eloquent 事件。每個方法都會將受影響的 Model 作為其唯一參數。`make:observer` Artisan 命令是建立新觀察者類別最簡單的方法：

```shell
php artisan make:observer UserObserver --model=User
```

此命令會將新的觀察者放置在您的 `app/Observers` 目錄中。如果此目錄不存在，Artisan 會為您建立它。您的全新觀察者會如下所示：

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

要註冊一個觀察者，您可以將 `ObservedBy` 屬性放置在對應的 Model 上：

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
> 觀察者可以監聽額外的事件，例如 `saving` 和 `retrieved`。這些事件在[事件](#events)文件中有所描述。

<a name="observers-and-database-transactions"></a>
#### 觀察者與資料庫交易

當 Model 在資料庫交易中建立時，您可能希望指示觀察者僅在資料庫交易提交後才執行其事件處理器。您可以透過在觀察者上實作 `ShouldHandleEventsAfterCommit` 介面來實現此目的。如果資料庫交易未進行中，事件處理器將立即執行：

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
### 事件靜默

您可能偶爾需要暫時「靜默」Model 觸發的所有事件。您可以使用 `withoutEvents` 方法來實現此目的。`withoutEvents` 方法接受一個閉包作為其唯一參數。在此閉包中執行的任何程式碼都不會分派 Model 事件，並且閉包返回的任何值都將由 `withoutEvents` 方法返回：

```php
use App\Models\User;

$user = User::withoutEvents(function () {
    User::findOrFail(1)->delete();

    return User::find(2);
});
```

<a name="saving-a-single-model-without-events"></a>
#### 不觸發事件來儲存單一 Model

有時您可能希望「儲存」給定的 Model 而不分派任何事件。您可以使用 `saveQuietly` 方法來實現此目的：

```php
$user = User::findOrFail(1);

$user->name = 'Victoria Faith';

$user->saveQuietly();
```

您也可以在不分派任何事件的情況下「更新」、「刪除」、「軟刪除」、「復原」和「複製」給定的 Model：

```php
$user->deleteQuietly();
$user->forceDeleteQuietly();
$user->restoreQuietly();
```