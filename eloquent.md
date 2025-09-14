# Eloquent: 入門

- [簡介](#introduction)
- [產生 Model 類別](#generating-model-classes)
- [Eloquent Model 慣例](#eloquent-model-conventions)
    - [資料表名稱](#table-names)
    - [主鍵](#primary-keys)
    - [UUID 與 ULID 鍵](#uuid-and-ulid-keys)
    - [時間戳記](#timestamps)
    - [資料庫連線](#database-connections)
    - [預設屬性值](#default-attribute-values)
    - [設定 Eloquent 的嚴格模式](#configuring-eloquent-strictness)
- [擷取 Model](#retrieving-models)
    - [集合](#collections)
    - [分批處理結果](#chunking-results)
    - [使用 Lazy Collections 分批處理](#chunking-using-lazy-collections)
    - [游標](#cursors)
    - [進階子查詢](#advanced-subqueries)
- [擷取單一 Model / 聚合](#retrieving-single-models)
    - [擷取或建立 Model](#retrieving-or-creating-models)
    - [擷取聚合](#retrieving-aggregates)
- [插入與更新 Model](#inserting-and-updating-models)
    - [插入](#inserts)
    - [更新](#updates)
    - [大量賦值](#mass-assignment)
    - [Upserts](#upserts)
- [刪除 Model](#deleting-models)
    - [軟刪除](#soft-deleting)
    - [查詢軟刪除的 Model](#querying-soft-deleted-models)
- [修剪 Model](#pruning-models)
- [複製 Model](#replicating-models)
- [查詢 Scope](#query-scopes)
    - [全域 Scope](#global-scopes)
    - [區域 Scope](#local-scopes)
    - [待處理屬性](#pending-attributes)
- [比較 Model](#comparing-models)
- [事件](#events)
    - [使用 Closure](#events-using-closures)
    - [觀察者](#observers)
    - [靜音事件](#muting-events)

<a name="introduction"></a>
## 簡介

Laravel 包含了 Eloquent，這是一個物件關聯對映 (ORM)，讓您能愉快地與資料庫互動。使用 Eloquent 時，每個資料庫資料表都有一個對應的「Model」，用於與該資料表互動。除了從資料庫資料表擷取記錄外，Eloquent Model 也允許您插入、更新和刪除資料表中的記錄。

> [!NOTE]
> 在開始之前，請務必在應用程式的 `config/database.php` 設定檔中設定資料庫連線。有關設定資料庫的更多資訊，請參閱[資料庫設定說明文件](/docs/{{version}}/database#configuration)。

<a name="generating-model-classes"></a>
## 產生 Model 類別

首先，讓我們建立一個 Eloquent Model。Model 通常位於 `app\Models` 目錄中，並繼承 `Illuminate\Database\Eloquent\Model` 類別。您可以使用 `make:model` [Artisan 命令](/docs/{{version}}/artisan)來產生一個新的 Model：

```shell
php artisan make:model Flight
```

如果您想在產生 Model 時同時產生[資料庫遷移](/docs/{{version}}/migrations)，可以使用 `--migration` 或 `-m` 選項：

```shell
php artisan make:model Flight --migration
```

在產生 Model 時，您可以產生各種其他類型的類別，例如 factories、seeders、policies、controllers 和 form requests。此外，這些選項可以組合起來一次建立多個類別：

```shell
# 產生一個 Model 和一個 FlightFactory 類別...
php artisan make:model Flight --factory
php artisan make:model Flight -f

# 產生一個 Model 和一個 FlightSeeder 類別...
php artisan make:model Flight --seed
php artisan make:model Flight -s

# 產生一個 Model 和一個 FlightController 類別...
php artisan make:model Flight --controller
php artisan make:model Flight -c

# 產生一個 Model、FlightController 資源類別和 form request 類別...
php artisan make:model Flight --controller --resource --requests
php artisan make:model Flight -crR

# 產生一個 Model 和一個 FlightPolicy 類別...
php artisan make:model Flight --policy

# 產生一個 Model、migration、factory、seeder 和 controller...
php artisan make:model Flight -mfsc

# 產生一個 Model、migration、factory、seeder、policy、controller 和 form requests 的捷徑...
php artisan make:model Flight --all
php artisan make:model Flight -a

# 產生一個 pivot Model...
php artisan make:model Member --pivot
php artisan make:model Member -p
```

<a name="inspecting-models"></a>
#### 檢查 Model

有時僅透過瀏覽程式碼很難確定 Model 的所有可用屬性和關聯。您可以嘗試 `model:show` Artisan 命令，它提供了 Model 所有屬性和關聯的便捷概覽：

```shell
php artisan model:show Flight
```

<a name="eloquent-model-conventions"></a>
## Eloquent Model 慣例

由 `make:model` 命令產生的 Model 將會被放置在 `app/Models` 目錄中。讓我們來檢視一個基本的 Model 類別，並討論一些 Eloquent 的關鍵慣例：

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

看過上面的範例後，您可能已經注意到我們沒有告訴 Eloquent 哪個資料庫資料表對應到我們的 `Flight` Model。按照慣例，除非明確指定其他名稱，否則類別的「snake case」、複數名稱將用作資料表名稱。因此，在這種情況下，Eloquent 將假定 `Flight` Model 將記錄儲存在 `flights` 資料表中，而 `AirTrafficController` Model 將記錄儲存在 `air_traffic_controllers` 資料表中。

如果您的 Model 對應的資料庫資料表不符合此慣例，您可以透過在 Model 上定義 `table` 屬性來手動指定 Model 的資料表名稱：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * 與 Model 相關聯的資料表。
     *
     * @var string
     */
    protected $table = 'my_flights';
}
```

<a name="primary-keys"></a>
### 主鍵

Eloquent 也會假定每個 Model 對應的資料庫資料表都有一個名為 `id` 的主鍵欄位。如有必要，您可以在 Model 上定義一個受保護的 `$primaryKey` 屬性，以指定作為 Model 主鍵的不同欄位：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * 與資料表相關聯的主鍵。
     *
     * @var string
     */
    protected $primaryKey = 'flight_id';
}
```

此外，Eloquent 假定主鍵是一個遞增的整數值，這表示 Eloquent 會自動將主鍵轉換為整數。如果您希望使用非遞增或非數值的主鍵，您必須在 Model 上定義一個公開的 `$incrementing` 屬性並將其設定為 `false`：

```php
<?php

class Flight extends Model
{
    /**
     * 指示 Model 的 ID 是否自動遞增。
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
     * 主鍵 ID 的資料型別。
     *
     * @var string
     */
    protected $keyType = 'string';
}
```

<a name="composite-primary-keys"></a>
#### 「複合」主鍵

Eloquent 要求每個 Model 至少有一個唯一識別的「ID」作為其主鍵。Eloquent Model 不支援「複合」主鍵。但是，除了資料表的唯一識別主鍵之外，您可以自由地為資料庫資料表新增額外的多欄位唯一索引。

<a name="uuid-and-ulid-keys"></a>
### UUID 與 ULID 鍵

您可以選擇使用 UUID 而不是自動遞增的整數作為 Eloquent Model 的主鍵。UUID 是通用唯一的英數字元識別碼，長度為 36 個字元。

如果您希望 Model 使用 UUID 鍵而不是自動遞增的整數鍵，您可以在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUuids` trait。當然，您應該確保 Model 具有[等效的 UUID 主鍵欄位](/docs/{{version}}/migrations#column-method-uuid)：

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

預設情況下，`HasUuids` trait 將為您的 Model 產生[「有序」UUID](/docs/{{version}}/strings#method-str-ordered-uuid)。這些 UUID 對於索引資料庫儲存更有效率，因為它們可以按字典順序排序。

您可以透過在 Model 上定義 `newUniqueId` 方法來覆寫給定 Model 的 UUID 產生過程。此外，您可以透過在 Model 上定義 `uniqueIds` 方法來指定哪些欄位應該接收 UUID：

```php
use Ramsey\Uuid\Uuid;

/**
 * 為 Model 產生一個新的 UUID。
 */
public function newUniqueId(): string
{
    return (string) Uuid::uuid4();
}

/**
 * 取得應該接收唯一識別碼的欄位。
 *
 * @return array<int, string>
 */
public function uniqueIds(): array
{
    return ['id', 'discount_code'];
}
```

如果您願意，您可以選擇使用「ULID」而不是 UUID。ULID 與 UUID 相似；但是，它們的長度只有 26 個字元。與有序 UUID 一樣，ULID 可以按字典順序排序，以實現高效的資料庫索引。要使用 ULID，您應該在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUlids` trait。您還應該確保 Model 具有[等效的 ULID 主鍵欄位](/docs/{{version}}/migrations#column-method-ulid)：

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

預設情況下，Eloquent 預期您的 Model 對應的資料庫資料表上存在 `created_at` 和 `updated_at` 欄位。當 Model 被建立或更新時，Eloquent 會自動設定這些欄位的值。如果您不希望這些欄位由 Eloquent 自動管理，您應該在 Model 上定義一個 `$timestamps` 屬性並將其值設定為 `false`：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * 指示 Model 是否應帶有時間戳記。
     *
     * @var bool
     */
    public $timestamps = false;
}
```

如果您需要自訂 Model 時間戳記的格式，請在 Model 上設定 `$dateFormat` 屬性。此屬性決定了日期屬性在資料庫中的儲存方式，以及當 Model 序列化為陣列或 JSON 時的格式：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * Model 日期欄位的儲存格式。
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

如果您想在不修改 Model 的 `updated_at` 時間戳記的情況下執行 Model 操作，您可以在傳遞給 `withoutTimestamps` 方法的 Closure 中操作 Model：

```php
Model::withoutTimestamps(fn () => $post->increment('reads'));
```

<a name="database-connections"></a>
### 資料庫連線

預設情況下，所有 Eloquent Model 都將使用為您的應用程式設定的預設資料庫連線。如果您想指定在與特定 Model 互動時應使用的不同連線，您應該在 Model 上定義一個 `$connection` 屬性：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * Model 應使用的資料庫連線。
     *
     * @var string
     */
    protected $connection = 'mysql';
}
```

<a name="default-attribute-values"></a>
### 預設屬性值

預設情況下，新實例化的 Model 實例將不包含任何屬性值。如果您想為 Model 的某些屬性定義預設值，您可以在 Model 上定義一個 `$attributes` 屬性。放置在 `$attributes` 陣列中的屬性值應以其原始的「可儲存」格式呈現，就像它們剛從資料庫中讀取一樣：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * Model 屬性的預設值。
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
### 設定 Eloquent 的嚴格模式

Laravel 提供了幾種方法，讓您可以在各種情況下設定 Eloquent 的行為和「嚴格性」。

首先，`preventLazyLoading` 方法接受一個可選的布林參數，指示是否應阻止 Lazy Loading。例如，您可能希望只在非生產環境中禁用 Lazy Loading，以便即使生產程式碼中意外存在 Lazy Loading 關聯，您的生產環境也能繼續正常運作。通常，此方法應在應用程式 `AppServiceProvider` 的 `boot` 方法中呼叫：

```php
use Illuminate\Database\Eloquent\Model;

/**
 * 啟動任何應用程式服務。
 */
public function boot(): void
{
    Model::preventLazyLoading(! $this->app->isProduction());
}
```

此外，您可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填寫不可填寫的屬性時拋出例外。這有助於防止在本地開發期間，嘗試設定尚未新增到 Model 的 `fillable` 陣列中的屬性時發生意外錯誤：

```php
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

<a name="retrieving-models"></a>
## 擷取 Model

一旦您建立了一個 Model 和[其相關聯的資料庫資料表](/docs/{{version}}/migrations#generating-migrations)，您就可以開始從資料庫中擷取資料了。您可以將每個 Eloquent Model 視為一個強大的[查詢建構器](/docs/{{version}}/queries)，讓您能夠流暢地查詢與 Model 相關聯的資料庫資料表。Model 的 `all` 方法將擷取 Model 相關聯資料庫資料表中的所有記錄：

```php
use App\Models\Flight;

foreach (Flight::all() as $flight) {
    echo $flight->name;
}
```

<a name="building-queries"></a>
#### 建構查詢

Eloquent 的 `all` 方法將返回 Model 資料表中所有結果。然而，由於每個 Eloquent Model 都作為一個[查詢建構器](/docs/{{version}}/queries)，您可以向查詢添加額外的約束，然後呼叫 `get` 方法來擷取結果：

```php
$flights = Flight::where('active', 1)
    ->orderBy('name')
    ->limit(10)
    ->get();
```

> [!NOTE]
> 由於 Eloquent Model 是查詢建構器，您應該檢閱 Laravel [查詢建構器](/docs/{{version}}/queries)提供的所有方法。您可以在編寫 Eloquent 查詢時使用這些方法中的任何一個。

<a name="refreshing-models"></a>
#### 重新整理 Model

如果您已經有一個從資料庫中擷取的 Eloquent Model 實例，您可以使用 `fresh` 和 `refresh` 方法「重新整理」該 Model。`fresh` 方法將從資料庫中重新擷取 Model。現有的 Model 實例將不受影響：

```php
$flight = Flight::where('number', 'FR 900')->first();

$freshFlight = $flight->fresh();
```

`refresh` 方法將使用資料庫中的新資料重新填充現有的 Model。此外，其所有已載入的關聯也會被重新整理：

```php
$flight = Flight::where('number', 'FR 900')->first();

$flight->number = 'FR 456';

$flight->refresh();

$flight->number; // "FR 900"
```

<a name="collections"></a>
### 集合

如我們所見，Eloquent 的 `all` 和 `get` 等方法從資料庫中擷取多條記錄。然而，這些方法不會返回一個普通的 PHP 陣列。相反，它會返回一個 `Illuminate\Database\Eloquent\Collection` 實例。

Eloquent 的 `Collection` 類別擴展了 Laravel 的基礎 `Illuminate\Support\Collection` 類別，該類別提供了[各種有用的方法](/docs/{{version}}/collections#available-methods)用於與資料集合互動。例如，`reject` 方法可用於根據呼叫的 Closure 結果從集合中移除 Model：

```php
$flights = Flight::where('destination', 'Paris')->get();

$flights = $flights->reject(function (Flight $flight) {
    return $flight->cancelled;
});
```

除了 Laravel 基礎 Collection 類別提供的方法外，Eloquent Collection 類別還提供了[一些額外的方法](/docs/{{version}}/eloquent-collections#available-methods)，這些方法專門用於與 Eloquent Model 集合互動。

由於 Laravel 的所有 Collection 都實作了 PHP 的可迭代介面，您可以像處理陣列一樣遍歷 Collection：

```php
foreach ($flights as $flight) {
    echo $flight->name;
}
```

<a name="chunking-results"></a>
### 分批處理結果

如果您的應用程式嘗試透過 `all` 或 `get` 方法載入數萬條 Eloquent 記錄，可能會耗盡記憶體。與其使用這些方法，不如使用 `chunk` 方法來更有效地處理大量 Model。

`chunk` 方法將擷取 Eloquent Model 的子集，並將它們傳遞給一個 Closure 進行處理。由於每次只擷取當前批次的 Eloquent Model，因此當處理大量 Model 時，`chunk` 方法將顯著減少記憶體使用量：

```php
use App\Models\Flight;
use Illuminate\Database\Eloquent\Collection;

Flight::chunk(200, function (Collection $flights) {
    foreach ($flights as $flight) {
        // ...
    }
});
```

傳遞給 `chunk` 方法的第一個參數是您希望每個「批次」接收的記錄數。作為第二個參數傳遞的 Closure 將針對從資料庫中擷取的每個批次被呼叫。每次擷取傳遞給 Closure 的記錄批次時，都會執行一次資料庫查詢。

如果您根據一個欄位來篩選 `chunk` 方法的結果，並且您在遍歷結果時也會更新該欄位，那麼您應該使用 `chunkById` 方法。在這些情況下使用 `chunk` 方法可能會導致意外且不一致的結果。在內部，`chunkById` 方法將始終擷取 `id` 欄位大於前一個批次中最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->chunkById(200, function (Collection $flights) {
        $flights->each->update(['departed' => false]);
    }, column: 'id');
```

由於 `chunkById` 和 `lazyById` 方法會向正在執行的查詢添加自己的「where」條件，因此您通常應該在 Closure 中[邏輯分組](/docs/{{version}}/queries#logical-grouping)您自己的條件：

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
### 使用 Lazy Collections 分批處理

`lazy` 方法的工作方式與[ `chunk` 方法](#chunking-results)類似，它在幕後以批次執行查詢。然而，`lazy` 方法不是直接將每個批次傳遞給回呼，而是返回一個扁平化的 [LazyCollection](/docs/{{version}}/collections#lazy-collections) 的 Eloquent Model，讓您可以將結果作為單一串流進行互動：

```php
use App\Models\Flight;

foreach (Flight::lazy() as $flight) {
    // ...
}
```

如果您根據一個欄位來篩選 `lazy` 方法的結果，並且您在遍歷結果時也會更新該欄位，那麼您應該使用 `lazyById` 方法。在內部，`lazyById` 方法將始終擷取 `id` 欄位大於前一個批次中最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->lazyById(200, column: 'id')
    ->each->update(['departed' => false]);
```

您可以使用 `lazyByIdDesc` 方法根據 `id` 的降序來篩選結果。

<a name="cursors"></a>
### 游標

與 `lazy` 方法類似，`cursor` 方法可用於在遍歷數萬條 Eloquent Model 記錄時顯著減少應用程式的記憶體消耗。

`cursor` 方法只會執行單一資料庫查詢；然而，個別的 Eloquent Model 直到實際遍歷時才會被填充。因此，在遍歷游標時，任何給定時間只有一個 Eloquent Model 保留在記憶體中。

> [!WARNING]
> 由於 `cursor` 方法一次只在記憶體中保留一個 Eloquent Model，因此它無法預先載入關聯。如果您需要預先載入關聯，請考慮使用[ `lazy` 方法](#chunking-using-lazy-collections)。

在內部，`cursor` 方法使用 PHP [生成器](https://www.php.net/manual/en/language.generators.overview.php)來實作此功能：

```php
use App\Models\Flight;

foreach (Flight::where('destination', 'Zurich')->cursor() as $flight) {
    // ...
}
```

`cursor` 返回一個 `Illuminate\Support\LazyCollection` 實例。[Lazy collections](/docs/{{version}}/collections#lazy-collections) 允許您使用典型 Laravel Collection 上可用的許多 Collection 方法，同時一次只將一個 Model 載入記憶體中：

```php
use App\Models\User;

$users = User::cursor()->filter(function (User $user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```

儘管 `cursor` 方法比常規查詢使用更少的記憶體 (一次只在記憶體中保留一個 Eloquent Model)，但它最終仍會耗盡記憶體。這是[由於 PHP 的 PDO 驅動程式在內部緩存其緩衝區中的所有原始查詢結果](https://www.php.net/manual/en/mysqlinfo.concepts.buffering.php)。如果您正在處理非常大量的 Eloquent 記錄，請考慮使用[ `lazy` 方法](#chunking-using-lazy-collections)。

<a name="advanced-subqueries"></a>
### 進階子查詢

<a name="subquery-selects"></a>
#### 子查詢 Select

Eloquent 也提供進階子查詢支援，讓您可以在單一查詢中從相關資料表提取資訊。例如，假設我們有一個航班 `destinations` 資料表和一個飛往目的地的 `flights` 資料表。`flights` 資料表包含一個 `arrived_at` 欄位，表示航班抵達目的地的時間。

使用查詢建構器 `select` 和 `addSelect` 方法可用的子查詢功能，我們可以透過單一查詢選取所有 `destinations` 以及最近抵達該目的地的航班名稱：

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

此外，查詢建構器的 `orderBy` 函數支援子查詢。繼續使用我們的航班範例，我們可以使用此功能根據上次航班抵達目的地的時間對所有目的地進行排序。同樣，這可以在執行單一資料庫查詢時完成：

```php
return Destination::orderByDesc(
    Flight::select('arrived_at')
        ->whereColumn('destination_id', 'destinations.id')
        ->orderByDesc('arrived_at')
        ->limit(1)
)->get();
```

<a name="retrieving-single-models"></a>
## 擷取單一 Model / 聚合

除了擷取符合給定查詢的所有記錄外，您還可以使用 `find`、`first` 或 `firstWhere` 方法擷取單一記錄。這些方法不會返回 Model 集合，而是返回單一 Model 實例：

```php
use App\Models\Flight;

// 透過主鍵擷取 Model...
$flight = Flight::find(1);

// 擷取符合查詢約束的第一個 Model...
$flight = Flight::where('active', 1)->first();

// 擷取符合查詢約束的第一個 Model 的替代方法...
$flight = Flight::firstWhere('active', 1);
```

有時您可能希望在找不到結果時執行其他操作。`findOr` 和 `firstOr` 方法將返回單一 Model 實例，或者，如果找不到結果，則執行給定的 Closure。Closure 返回的值將被視為方法的結果：

```php
$flight = Flight::findOr(1, function () {
    // ...
});

$flight = Flight::where('legs', '>', 3)->firstOr(function () {
    // ...
});
```

<a name="not-found-exceptions"></a>
#### 未找到例外

有時您可能希望在找不到 Model 時拋出例外。這在路由或控制器中特別有用。`findOrFail` 和 `firstOrFail` 方法將擷取查詢的第一個結果；但是，如果找不到結果，將拋出 `Illuminate\Database\Eloquent\ModelNotFoundException`：

```php
$flight = Flight::findOrFail(1);

$flight = Flight::where('legs', '>', 3)->firstOrFail();
```

如果未捕獲 `ModelNotFoundException`，則會自動將 404 HTTP 回應傳回給客戶端：

```php
use App\Models\Flight;

Route::get('/api/flights/{id}', function (string $id) {
    return Flight::findOrFail($id);
});
```

<a name="retrieving-or-creating-models"></a>
### 擷取或建立 Model

`firstOrCreate` 方法將嘗試使用給定的欄位/值對來定位資料庫記錄。如果在資料庫中找不到 Model，則會插入一條記錄，其屬性是將第一個陣列參數與可選的第二個陣列參數合併後的結果。

`firstOrNew` 方法與 `firstOrCreate` 類似，它將嘗試在資料庫中定位與給定屬性匹配的記錄。但是，如果找不到 Model，則會返回一個新的 Model 實例。請注意，`firstOrNew` 返回的 Model 尚未持久化到資料庫中。您需要手動呼叫 `save` 方法才能將其持久化：

```php
use App\Models\Flight;

// 透過名稱擷取航班，如果不存在則建立...
$flight = Flight::firstOrCreate([
    'name' => 'London to Paris'
]);

// 透過名稱擷取航班，如果不存在則使用名稱、delayed 和 arrival_time 屬性建立...
$flight = Flight::firstOrCreate(
    ['name' => 'London to Paris'],
    ['delayed' => 1, 'arrival_time' => '11:30']
);

// 透過名稱擷取航班，如果不存在則實例化一個新的 Flight 實例...
$flight = Flight::firstOrNew([
    'name' => 'London to Paris'
]);

// 透過名稱擷取航班，如果不存在則使用名稱、delayed 和 arrival_time 屬性實例化...
$flight = Flight::firstOrNew(
    ['name' => 'Tokyo to Sydney'],
    ['delayed' => 1, 'arrival_time' => '11:30']
);
```

<a name="retrieving-aggregates"></a>
### 擷取聚合

與 Eloquent Model 互動時，您也可以使用 Laravel [查詢建構器](/docs/{{version}}/queries)提供的 `count`、`sum`、`max` 和其他[聚合方法](/docs/{{version}}/queries#aggregates)。正如您所預期的，這些方法返回一個純量值而不是 Eloquent Model 實例：

```php
$count = Flight::where('active', 1)->count();

$max = Flight::where('active', 1)->max('price');
```

<a name="inserting-and-updating-models"></a>
## 插入與更新 Model

<a name="inserts"></a>
### 插入

當然，使用 Eloquent 時，我們不僅需要從資料庫中擷取 Model。我們還需要插入新記錄。幸運的是，Eloquent 使其變得簡單。要將新記錄插入資料庫，您應該實例化一個新的 Model 實例並設定 Model 上的屬性。然後，呼叫 Model 實例上的 `save` 方法：

```php
<?php

namespace App\Http\Controllers;

use App\Models\Flight;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class FlightController extends Controller
{
    /**
     * 將新航班儲存到資料庫中。
     */
    public function store(Request $request): RedirectResponse
    {
        // 驗證請求...

        $flight = new Flight;

        $flight->name = $request->name;

        $flight->save();

        return redirect('/flights');
    }
}
```

在此範例中，我們將傳入 HTTP 請求中的 `name` 欄位賦值給 `App\Models\Flight` Model 實例的 `name` 屬性。當我們呼叫 `save` 方法時，一條記錄將被插入資料庫。Model 的 `created_at` 和 `updated_at` 時間戳記將在呼叫 `save` 方法時自動設定，因此無需手動設定它們。

或者，您可以使用 `create` 方法透過單一 PHP 語句「儲存」一個新的 Model。`create` 方法將返回插入的 Model 實例給您：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，您需要在 Model 類別上指定 `fillable` 或 `guarded` 屬性。這些屬性是必需的，因為所有 Eloquent Model 預設都受到保護，以防止大量賦值漏洞。要了解更多關於大量賦值的資訊，請參閱[大量賦值說明文件](#mass-assignment)。

<a name="updates"></a>
### 更新

`save` 方法也可以用於更新資料庫中已存在的 Model。要更新 Model，您應該擷取它並設定您希望更新的任何屬性。然後，您應該呼叫 Model 的 `save` 方法。同樣，`updated_at` 時間戳記將自動更新，因此無需手動設定其值：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->name = 'Paris to London';

$flight->save();
```

有時，您可能需要更新現有 Model，或者在沒有匹配 Model 存在時建立一個新 Model。與 `firstOrCreate` 方法一樣，`updateOrCreate` 方法會持久化 Model，因此無需手動呼叫 `save` 方法。

在下面的範例中，如果存在一個 `departure` 地點為 `Oakland` 且 `destination` 地點為 `San Diego` 的航班，其 `price` 和 `discounted` 欄位將被更新。如果不存在這樣的航班，則會建立一個新航班，其屬性是將第一個參數陣列與第二個參數陣列合併後的結果：

```php
$flight = Flight::updateOrCreate(
    ['departure' => 'Oakland', 'destination' => 'San Diego'],
    ['price' => 99, 'discounted' => 1]
);
```

<a name="mass-updates"></a>
#### 大量更新

更新也可以針對符合給定查詢的 Model 執行。在此範例中，所有 `active` 且 `destination` 為 `San Diego` 的航班都將被標記為延遲：

```php
Flight::where('active', 1)
    ->where('destination', 'San Diego')
    ->update(['delayed' => 1]);
```

`update` 方法預期一個欄位和值對的陣列，表示應該更新的欄位。`update` 方法返回受影響的行數。

> [!WARNING]
> 當透過 Eloquent 發出大量更新時，`saving`、`saved`、`updating` 和 `updated` Model 事件將不會針對更新的 Model 觸發。這是因為在發出大量更新時，Model 從未實際被擷取。

<a name="examining-attribute-changes"></a>
#### 檢查屬性變更

Eloquent 提供了 `isDirty`、`isClean` 和 `wasChanged` 方法來檢查 Model 的內部狀態，並確定其屬性自 Model 最初擷取以來如何變化。

`isDirty` 方法判斷 Model 的任何屬性自 Model 擷取以來是否已更改。您可以將特定的屬性名稱或屬性陣列傳遞給 `isDirty` 方法，以判斷任何屬性是否「髒」。`isClean` 方法將判斷屬性自 Model 擷取以來是否保持不變。此方法也接受一個可選的屬性參數：

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

`wasChanged` 方法判斷在當前請求週期中，Model 上次儲存時是否有任何屬性發生變化。如果需要，您可以傳遞屬性名稱以查看特定屬性是否發生變化：

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

`getOriginal` 方法返回一個陣列，其中包含 Model 的原始屬性，無論 Model 自擷取以來是否有任何更改。如果需要，您可以傳遞特定的屬性名稱以獲取特定屬性的原始值：

```php
$user = User::find(1);

$user->name; // John
$user->email; // john @example.com

$user->name = 'Jack';
$user->name; // Jack

$user->getOriginal('name'); // John
$user->getOriginal(); // 原始屬性陣列...
```

`getChanges` 方法返回一個陣列，其中包含 Model 上次儲存時更改的屬性，而 `getPrevious` 方法返回一個陣列，其中包含 Model 上次儲存之前的原始屬性值：

```php
$user = User::find(1);

$user->name; // John
$user->email; // john @example.com

$user->update([
    'name' => 'Jack',
    'email' => 'jack @example.com',
]);

$user->getChanges();

/*
    [
        'name' => 'Jack',
        'email' => 'jack @example.com',
    ]
*/

$user->getPrevious();

/*
    [
        'name' => 'John',
        'email' => 'john @example.com',
    ]
*/
```

<a name="mass-assignment"></a>
### 大量賦值

您可以使用 `create` 方法透過單一 PHP 語句「儲存」一個新的 Model。該方法將返回插入的 Model 實例給您：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，您需要在 Model 類別上指定 `fillable` 或 `guarded` 屬性。這些屬性是必需的，因為所有 Eloquent Model 預設都受到保護，以防止大量賦值漏洞。

當使用者傳遞意外的 HTTP 請求欄位，並且該欄位更改了您未預期的資料庫中的欄位時，就會發生大量賦值漏洞。例如，惡意使用者可能會透過 HTTP 請求發送 `is_admin` 參數，然後將該參數傳遞給 Model 的 `create` 方法，從而允許使用者將自己提升為管理員。

因此，首先，您應該定義您希望哪些 Model 屬性可以進行大量賦值。您可以使用 Model 上的 `$fillable` 屬性來完成此操作。例如，讓我們將 `Flight` Model 的 `name` 屬性設定為可大量賦值：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * 可大量賦值的屬性。
     *
     * @var array<int, string>
     */
    protected $fillable = ['name'];
}
```

一旦您指定了哪些屬性可以進行大量賦值，您就可以使用 `create` 方法在資料庫中插入一條新記錄。`create` 方法返回新建立的 Model 實例：

```php
$flight = Flight::create(['name' => 'London to Paris']);
```

如果您已經有一個 Model 實例，您可以使用 `fill` 方法用屬性陣列填充它：

```php
$flight->fill(['name' => 'Amsterdam to Frankfurt']);
```

<a name="mass-assignment-json-columns"></a>
#### 大量賦值與 JSON 欄位

在賦值 JSON 欄位時，每個欄位的大量賦值鍵必須在 Model 的 `$fillable` 陣列中指定。為了安全起見，Laravel 在使用 `guarded` 屬性時不支援更新巢狀 JSON 屬性：

```php
/**
 * 可大量賦值的屬性。
 *
 * @var array<int, string>
 */
protected $fillable = [
    'options->enabled',
];
```

<a name="allowing-mass-assignment"></a>
#### 允許大量賦值

如果您想讓所有屬性都可以大量賦值，您可以將 Model 的 `$guarded` 屬性定義為一個空陣列。如果您選擇解除 Model 的保護，您應該特別注意始終手動建立傳遞給 Eloquent 的 `fill`、`create` 和 `update` 方法的陣列：

```php
/**
 * 不可大量賦值的屬性。
 *
 * @var array<string>|bool
 */
protected $guarded = [];
```

<a name="mass-assignment-exceptions"></a>
#### 大量賦值例外

預設情況下，在執行大量賦值操作時，未包含在 `$fillable` 陣列中的屬性會被靜默丟棄。在生產環境中，這是預期的行為；然而，在本地開發期間，這可能會導致混淆，因為 Model 更改未生效。

如果您願意，您可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填寫不可填寫的屬性時拋出例外。通常，此方法應在應用程式 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

```php
use Illuminate\Database\Eloquent\Model;

/**
 * 啟動任何應用程式服務。
 */
public function boot(): void
{
    Model::preventSilentlyDiscardingAttributes($this->app->isLocal());
}
```

<a name="upserts"></a>
### Upserts

Eloquent 的 `upsert` 方法可用於在單一原子操作中更新或建立記錄。該方法的第一個參數包含要插入或更新的值，而第二個參數列出了在相關資料表中唯一識別記錄的欄位。該方法的第三個也是最後一個參數是一個陣列，其中包含如果資料庫中已存在匹配記錄則應更新的欄位。如果 Model 上啟用了時間戳記，`upsert` 方法將自動設定 `created_at` 和 `updated_at` 時間戳記：

```php
Flight::upsert([
    ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
    ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
], uniqueBy: ['departure', 'destination'], update: ['price']);
```

> [!WARNING]
> 除 SQL Server 外，所有資料庫都要求 `upsert` 方法的第二個參數中的欄位具有「主鍵」或「唯一」索引。此外，MariaDB 和 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個參數，並始終使用資料表的「主鍵」和「唯一」索引來偵測現有記錄。

<a name="deleting-models"></a>
## 刪除 Model

要刪除 Model，您可以呼叫 Model 實例上的 `delete` 方法：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->delete();
```

<a name="deleting-an-existing-model-by-its-primary-key"></a>
#### 透過主鍵刪除現有 Model

在上面的範例中，我們在呼叫 `delete` 方法之前從資料庫中擷取 Model。但是，如果您知道 Model 的主鍵，您可以透過呼叫 `destroy` 方法來刪除 Model，而無需明確擷取它。除了接受單一主鍵外，`destroy` 方法還將接受多個主鍵、主鍵陣列或主鍵[集合](/docs/{{version}}/collections)：

```php
Flight::destroy(1);

Flight::destroy(1, 2, 3);

Flight::destroy([1, 2, 3]);

Flight::destroy(collect([1, 2, 3]));
```

如果您正在使用[軟刪除 Model](#soft-deleting)，您可以透過 `forceDestroy` 方法永久刪除 Model：

```php
Flight::forceDestroy(1);
```

> [!WARNING]
> `destroy` 方法會單獨載入每個 Model 並呼叫 `delete` 方法，以便為每個 Model 正確分派 `deleting` 和 `deleted` 事件。

<a name="deleting-models-using-queries"></a>
#### 使用查詢刪除 Model

當然，您可以建構一個 Eloquent 查詢來刪除所有符合您查詢條件的 Model。在此範例中，我們將刪除所有標記為非活動的航班。與大量更新一樣，大量刪除不會為被刪除的 Model 分派 Model 事件：

```php
$deleted = Flight::where('active', 0)->delete();
```

要刪除資料表中所有 Model，您應該執行一個不帶任何條件的查詢：

```php
$deleted = Flight::query()->delete();
```

> [!WARNING]
> 當透過 Eloquent 執行大量刪除語句時，`deleting` 和 `deleted` Model 事件將不會針對被刪除的 Model 分派。這是因為在執行刪除語句時，Model 從未實際被擷取。

<a name="soft-deleting"></a>
### 軟刪除

除了實際從資料庫中移除記錄外，Eloquent 還可以「軟刪除」Model。當 Model 被軟刪除時，它們實際上並未從資料庫中移除。相反，Model 上會設定一個 `deleted_at` 屬性，指示 Model 被「刪除」的日期和時間。要為 Model 啟用軟刪除，請將 `Illuminate\Database\Eloquent\SoftDeletes` trait 新增到 Model：

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

您還應該將 `deleted_at` 欄位新增到您的資料庫資料表。Laravel [schema builder](/docs/{{version}}/migrations) 包含一個輔助方法來建立此欄位：

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

現在，當您在 Model 上呼叫 `delete` 方法時，`deleted_at` 欄位將被設定為當前日期和時間。但是，Model 的資料庫記錄將保留在資料表中。當查詢使用軟刪除的 Model 時，軟刪除的 Model 將自動從所有查詢結果中排除。

要判斷給定的 Model 實例是否已軟刪除，您可以使用 `trashed` 方法：

```php
if ($flight->trashed()) {
    // ...
}
```

<a name="restoring-soft-deleted-models"></a>
#### 還原軟刪除的 Model

有時您可能希望「取消刪除」一個軟刪除的 Model。要還原一個軟刪除的 Model，您可以在 Model 實例上呼叫 `restore` 方法。`restore` 方法會將 Model 的 `deleted_at` 欄位設定為 `null`：

```php
$flight->restore();
```

您也可以在查詢中使用 `restore` 方法來還原多個 Model。同樣，與其他「大量」操作一樣，這不會為被還原的 Model 分派任何 Model 事件：

```php
Flight::withTrashed()
    ->where('airline_id', 1)
    ->restore();
```

`restore` 方法也可以在建構[關聯](/docs/{{version}}/eloquent-relationships)查詢時使用：

```php
$flight->history()->restore();
```

<a name="permanently-deleting-models"></a>
#### 永久刪除 Model

有時您可能需要真正從資料庫中移除 Model。您可以使用 `forceDelete` 方法從資料庫資料表中永久移除軟刪除的 Model：

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

如上所述，軟刪除的 Model 將自動從查詢結果中排除。但是，您可以透過在查詢上呼叫 `withTrashed` 方法，強制將軟刪除的 Model 包含在查詢結果中：

```php
use App\Models\Flight;

$flights = Flight::withTrashed()
    ->where('account_id', 1)
    ->get();
```

`withTrashed` 方法也可以在建構[關聯](/docs/{{version}}/eloquent-relationships)查詢時呼叫：

```php
$flight->history()->withTrashed()->get();
```

<a name="retrieving-only-soft-deleted-models"></a>
#### 僅擷取軟刪除的 Model

`onlyTrashed` 方法將**僅**擷取軟刪除的 Model：

```php
$flights = Flight::onlyTrashed()
    ->where('airline_id', 1)
    ->get();
```

<a name="pruning-models"></a>
## 修剪 Model

有時您可能希望定期刪除不再需要的 Model。為此，您可以將 `Illuminate\Database\Eloquent\Prunable` 或 `Illuminate\Database\Eloquent\MassPrunable` trait 新增到您希望定期修剪的 Model 中。將其中一個 trait 新增到 Model 後，實作一個 `prunable` 方法，該方法返回一個 Eloquent 查詢建構器，用於解析不再需要的 Model：

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
     * 取得可修剪的 Model 查詢。
     */
    public function prunable(): Builder
    {
        return static::where('created_at', '<=', now()->subMonth());
    }
}
```

當將 Model 標記為 `Prunable` 時，您也可以在 Model 上定義 `pruning` 方法。此方法將在 Model 刪除之前呼叫。此方法對於刪除與 Model 相關聯的任何額外資源（例如儲存的檔案）非常有用，然後 Model 才會從資料庫中永久移除：

```php
/**
 * 準備 Model 進行修剪。
 */
protected function pruning(): void
{
    // ...
}
```

設定好可修剪的 Model 後，您應該在應用程式的 `routes/console.php` 檔案中排程 `model:prune` Artisan 命令。您可以自由選擇此命令應執行的適當間隔：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('model:prune')->daily();
```

在幕後，`model:prune` 命令將自動偵測應用程式 `app/Models` 目錄中的「Prunable」Model。如果您的 Model 位於不同的位置，您可以使用 `--model` 選項來指定 Model 類別名稱：

```php
Schedule::command('model:prune', [
    '--model' => [Address::class, Flight::class],
])->daily();
```

如果您希望在修剪所有其他偵測到的 Model 時排除某些 Model，您可以使用 `--except` 選項：

```php
Schedule::command('model:prune', [
    '--except' => [Address::class, Flight::class],
])->daily();
```

您可以透過執行帶有 `--pretend` 選項的 `model:prune` 命令來測試您的 `prunable` 查詢。在模擬時，`model:prune` 命令將只報告如果命令實際執行將修剪多少條記錄：

```shell
php artisan model:prune --pretend
```

> [!WARNING]
> 軟刪除的 Model 如果符合可修剪查詢，將會被永久刪除 (`forceDelete`)。

<a name="mass-pruning"></a>
#### 大量修剪

當 Model 標記有 `Illuminate\Database\Eloquent\MassPrunable` trait 時，Model 會使用大量刪除查詢從資料庫中刪除。因此，`pruning` 方法將不會被呼叫，`deleting` 和 `deleted` Model 事件也不會被分派。這是因為在刪除之前 Model 從未實際被擷取，因此修剪過程效率更高：

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
     * 取得可修剪的 Model 查詢。
     */
    public function prunable(): Builder
    {
        return static::where('created_at', '<=', now()->subMonth());
    }
}
```

<a name="replicating-models"></a>
## 複製 Model

您可以使用 `replicate` 方法建立現有 Model 實例的未儲存副本。當您擁有多個具有許多相同屬性的 Model 實例時，此方法特別有用：

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

要從複製到新 Model 的屬性中排除一個或多個屬性，您可以將一個陣列傳遞給 `replicate` 方法：

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

全域 Scope 允許您為給定 Model 的所有查詢添加約束。Laravel 自己的[軟刪除](#soft-deleting)功能利用全域 Scope 僅從資料庫中擷取「未刪除」的 Model。編寫自己的全域 Scope 可以提供一種方便、簡單的方法來確保給定 Model 的每個查詢都接收到某些約束。

<a name="generating-scopes"></a>
#### 產生 Scope

要產生一個新的全域 Scope，您可以呼叫 `make:scope` Artisan 命令，它會將產生的 Scope 放置在應用程式的 `app/Models/Scopes` 目錄中：

```shell
php artisan make:scope AncientScope
```

<a name="writing-global-scopes"></a>
#### 編寫全域 Scope

編寫全域 Scope 很簡單。首先，使用 `make:scope` 命令產生一個實作 `Illuminate\Database\Eloquent\Scope` 介面的類別。`Scope` 介面要求您實作一個方法：`apply`。`apply` 方法可以根據需要向查詢添加 `where` 約束或其他類型的子句：

```php
<?php

namespace App\Models\Scopes;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class AncientScope implements Scope
{
    /**
     * 將 Scope 應用於給定的 Eloquent 查詢建構器。
     */
    public function apply(Builder $builder, Model $model): void
    {
        $builder->where('created_at', '<', now()->subYears(2000));
    }
}
```

> [!NOTE]
> 如果您的全域 Scope 正在向查詢的 select 子句添加欄位，您應該使用 `addSelect` 方法而不是 `select`。這將防止意外替換查詢現有的 select 子句。

<a name="applying-global-scopes"></a>
#### 應用全域 Scope

要將全域 Scope 分配給 Model，您只需將 `ScopedBy` 屬性放置在 Model 上：

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

或者，您可以透過覆寫 Model 的 `booted` 方法並呼叫 Model 的 `addGlobalScope` 方法來手動註冊全域 Scope。`addGlobalScope` 方法接受您的 Scope 實例作為其唯一參數：

```php
<?php

namespace App\Models;

use App\Models\Scopes\AncientScope;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Model 的「booted」方法。
     */
    protected static function booted(): void
    {
        static::addGlobalScope(new AncientScope);
    }
}
```

在將上述範例中的 Scope 添加到 `App\Models\User` Model 後，呼叫 `User::all()` 方法將執行以下 SQL 查詢：

```sql
select * from `users` where `created_at` < 0021-02-18 00:00:00
```

<a name="anonymous-global-scopes"></a>
#### 匿名全域 Scope

Eloquent 也允許您使用 Closure 定義全域 Scope，這對於不需要單獨類別的簡單 Scope 特別有用。當使用 Closure 定義全域 Scope 時，您應該提供一個您自己選擇的 Scope 名稱作為 `addGlobalScope` 方法的第一個參數：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Model 的「booted」方法。
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

如果您想為給定查詢移除全域 Scope，您可以使用 `withoutGlobalScope` 方法。此方法接受全域 Scope 的類別名稱作為其唯一參數：

```php
User::withoutGlobalScope(AncientScope::class)->get();
```

或者，如果您使用 Closure 定義了全域 Scope，您應該傳遞您分配給全域 Scope 的字串名稱：

```php
User::withoutGlobalScope('ancient')->get();
```

如果您想移除查詢的幾個甚至所有全域 Scope，您可以使用 `withoutGlobalScopes` 和 `withoutGlobalScopesExcept` 方法：

```php
// 移除所有全域 Scope...
User::withoutGlobalScopes()->get();

// 移除部分全域 Scope...
User::withoutGlobalScopes([
    FirstScope::class, SecondScope::class
])->get();

// 移除除給定 Scope 之外的所有全域 Scope...
User::withoutGlobalScopesExcept([
    SecondScope::class,
])->get();
```

<a name="local-scopes"></a>
### 區域 Scope

區域 Scope 允許您定義一組常見的查詢約束，您可以輕鬆地在整個應用程式中重複使用。例如，您可能需要經常擷取所有被認為「受歡迎」的使用者。要定義 Scope，請將 `Scope` 屬性添加到 Eloquent 方法中。

Scope 應始終返回相同的查詢建構器實例或 `void`：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * 將查詢範圍限定為僅包含受歡迎的使用者。
     */
    #[Scope]
    protected function popular(Builder $query): void
    {
        $query->where('votes', '>', 100);
    }

    /**
     * 將查詢範圍限定為僅包含活躍的使用者。
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

一旦定義了 Scope，您就可以在查詢 Model 時呼叫 Scope 方法。您甚至可以鏈接對各種 Scope 的呼叫：

```php
use App\Models\User;

$users = User::popular()->active()->orderBy('created_at')->get();
```

透過 `or` 查詢運算子組合多個 Eloquent Model Scope 可能需要使用 Closure 來實現正確的[邏輯分組](/docs/{{version}}/queries#logical-grouping)：

```php
$users = User::popular()->orWhere(function (Builder $query) {
    $query->active();
})->get();
```

然而，由於這可能很麻煩，Laravel 提供了一個「高階」`orWhere` 方法，讓您可以流暢地鏈接 Scope，而無需使用 Closure：

```php
$users = User::popular()->orWhere->active()->get();
```

<a name="dynamic-scopes"></a>
#### 動態 Scope

有時您可能希望定義一個接受參數的 Scope。首先，只需將您的額外參數添加到您的 Scope 方法的簽名中。Scope 參數應在 `$query` 參數之後定義：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * 將查詢範圍限定為僅包含給定類型的使用者。
     */
    #[Scope]
    protected function ofType(Builder $query, string $type): void
    {
        $query->where('type', $type);
    }
}
```

一旦預期的參數已添加到您的 Scope 方法的簽名中，您就可以在呼叫 Scope 時傳遞參數：

```php
$users = User::ofType('admin')->get();
```

<a name="pending-attributes"></a>
### 待處理屬性

如果您想使用 Scope 建立具有與用於約束 Scope 的屬性相同的 Model，您可以在建構 Scope 查詢時使用 `withAttributes` 方法：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Scope;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    /**
     * 將查詢範圍限定為僅包含草稿。
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

`withAttributes` 方法將使用給定的屬性向查詢添加 `where` 條件，它還會將給定的屬性添加到透過 Scope 建立的任何 Model 中：

```php
$draft = Post::draft()->create(['title' => 'In Progress']);

$draft->hidden; // true
```

要指示 `withAttributes` 方法不向查詢添加 `where` 條件，您可以將 `asConditions` 參數設定為 `false`：

```php
$query->withAttributes([
    'hidden' => true,
], asConditions: false);
```

<a name="comparing-models"></a>
## 比較 Model

有時您可能需要判斷兩個 Model 是否「相同」。`is` 和 `isNot` 方法可用於快速驗證兩個 Model 是否具有相同的主鍵、資料表和資料庫連線：

```php
if ($post->is($anotherPost)) {
    // ...
}

if ($post->isNot($anotherPost)) {
    // ...
}
```

`is` 和 `isNot` 方法在使用 `belongsTo`、`hasOne`、`morphTo` 和 `morphOne` [關聯](/docs/{{version}}/eloquent-relationships)時也可用。當您想比較相關 Model 而不發出查詢來擷取該 Model 時，此方法特別有用：

```php
if ($post->author()->is($user)) {
    // ...
}
```

<a name="events"></a>
## 事件

> [!NOTE]
> 想要將您的 Eloquent 事件直接廣播到您的客戶端應用程式嗎？請查看 Laravel 的[Model 事件廣播](/docs/{{version}}/broadcasting#model-broadcasting)。

Eloquent Model 會分派多個事件，讓您可以在 Model 生命週期的以下時刻掛鉤：`retrieved`、`creating`、`created`、`updating`、`updated`、`saving`、`saved`、`deleting`、`deleted`、`trashed`、`forceDeleting`、`forceDeleted`、`restoring`、`restored` 和 `replicating`。

`retrieved` 事件將在從資料庫中擷取現有 Model 時分派。當新 Model 首次儲存時，`creating` 和 `created` 事件將分派。當現有 Model 被修改並呼叫 `save` 方法時，`updating` / `updated` 事件將分派。當 Model 被建立或更新時，即使 Model 的屬性沒有改變，`saving` / `saved` 事件也會分派。以 `-ing` 結尾的事件名稱在 Model 的任何更改持久化之前分派，而以 `-ed` 結尾的事件在 Model 的更改持久化之後分派。

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
     * Model 的事件對映。
     *
     * @var array<string, string>
     */
    protected $dispatchesEvents = [
        'saved' => UserSaved::class,
        'deleted' => UserDeleted::class,
    ];
}
```

定義並對映您的 Eloquent 事件後，您可以使用[事件監聽器](/docs/{{version}}/events#defining-listeners)來處理事件。

> [!WARNING]
> 當透過 Eloquent 發出大量更新或刪除查詢時，`saved`、`updated`、`deleting` 和 `deleted` Model 事件將不會針對受影響的 Model 分派。這是因為在執行大量更新或刪除時，Model 從未實際被擷取。

<a name="events-using-closures"></a>
### 使用 Closure

除了使用自訂事件類別外，您還可以註冊在分派各種 Model 事件時執行的 Closure。通常，您應該在 Model 的 `booted` 方法中註冊這些 Closure：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    /**
     * Model 的「booted」方法。
     */
    protected static function booted(): void
    {
        static::created(function (User $user) {
            // ...
        });
    }
}
```

如果需要，您可以在註冊 Model 事件時利用[可排隊的匿名事件監聽器](/docs/{{version}}/events#queuable-anonymous-event-listeners)。這將指示 Laravel 使用應用程式的[佇列](/docs/{{version}}/queues)在背景執行 Model 事件監聽器：

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

如果您正在監聽給定 Model 上的許多事件，您可以使用觀察者將所有監聽器分組到一個類別中。觀察者類別的方法名稱反映了您希望監聽的 Eloquent 事件。這些方法中的每一個都將受影響的 Model 作為其唯一參數。`make:observer` Artisan 命令是建立新觀察者類別的最簡單方法：

```shell
php artisan make:observer UserObserver --model=User
```

此命令會將新的觀察者放置在您的 `app/Observers` 目錄中。如果此目錄不存在，Artisan 會為您建立它。您新的觀察者將如下所示：

```php
<?php

namespace App\Observers;

use App\Models\User;

class UserObserver
{
    /**
     * 處理 User 的「created」事件。
     */
    public function created(User $user): void
    {
        // ...
    }

    /**
     * 處理 User 的「updated」事件。
     */
    public function updated(User $user): void
    {
        // ...
    }

    /**
     * 處理 User 的「deleted」事件。
     */
    public function deleted(User $user): void
    {
        // ...
    }

    /**
     * 處理 User 的「restored」事件。
     */
    public function restored(User $user): void
    {
        // ...
    }

    /**
     * 處理 User 的「forceDeleted」事件。
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

或者，您可以透過呼叫您希望觀察的 Model 上的 `observe` 方法來手動註冊觀察者。您可以在應用程式 `AppServiceProvider` 類別的 `boot` 方法中註冊觀察者：

```php
use App\Models\User;
use App\Observers\UserObserver;

/**
 * 啟動任何應用程式服務。
 */
public function boot(): void
{
    User::observe(UserObserver::class);
}
```

> [!NOTE]
> 觀察者可以監聽其他事件，例如 `saving` 和 `retrieved`。這些事件在[事件](#events)說明文件中有所描述。

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
     * 處理 User 的「created」事件。
     */
    public function created(User $user): void
    {
        // ...
    }
}
```

<a name="muting-events"></a>
### 靜音事件

您可能偶爾需要暫時「靜音」Model 觸發的所有事件。您可以使用 `withoutEvents` 方法來實現此目的。`withoutEvents` 方法接受一個 Closure 作為其唯一參數。在此 Closure 中執行的任何程式碼都不會分派 Model 事件，並且 Closure 返回的任何值都將由 `withoutEvents` 方法返回：

```php
use App\Models\User;

$user = User::withoutEvents(function () {
    User::findOrFail(1)->delete();

    return User::find(2);
});
```

<a name="saving-a-single-model-without-events"></a>
#### 在沒有事件的情況下儲存單一 Model

有時您可能希望「儲存」給定的 Model 而不分派任何事件。您可以使用 `saveQuietly` 方法來實現此目的：

```php
$user = User::findOrFail(1);

$user->name = 'Victoria Faith';

$user->saveQuietly();
```

您也可以在不分派任何事件的情況下「更新」、「刪除」、「軟刪除」、「還原」和「複製」給定的 Model：

```php
$user->deleteQuietly();
$user->forceDeleteQuietly();
$user->restoreQuietly();
```
