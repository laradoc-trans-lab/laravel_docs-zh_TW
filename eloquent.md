# Eloquent：入門

- [簡介](#introduction)
- [產生模型類別](#generating-model-classes)
- [Eloquent 模型慣例](#eloquent-model-conventions)
    - [資料表名稱](#table-names)
    - [主鍵](#primary-keys)
    - [UUID 與 ULID 金鑰](#uuid-and-ulid-keys)
    - [時間戳記](#timestamps)
    - [資料庫連線](#database-connections)
    - [屬性預設值](#default-attribute-values)
    - [設定 Eloquent 嚴格模式](#configuring-eloquent-strictness)
- [擷取模型](#retrieving-models)
    - [集合](#collections)
    - [分塊處理結果](#chunking-results)
    - [使用延遲集合進行分塊](#chunking-using-lazy-collections)
    - [游標](#cursors)
    - [進階子查詢](#advanced-subqueries)
- [擷取單一模型 / 聚合函數](#retrieving-single-models)
    - [擷取或建立模型](#retrieving-or-creating-models)
    - [擷取聚合結果](#retrieving-aggregates)
- [新增與更新模型](#inserting-and-updating-models)
    - [新增](#inserts)
    - [更新](#updates)
    - [大量賦值](#mass-assignment)
    - [Upserts](#upserts)
- [刪除模型](#deleting-models)
    - [軟刪除](#soft-deleting)
    - [查詢軟刪除的模型](#querying-soft-deleted-models)
- [修剪模型](#pruning-models)
- [複製模型](#replicating-models)
- [查詢範圍 (Query Scopes)](#query-scopes)
    - [全域範圍](#global-scopes)
    - [區域範圍](#local-scopes)
    - [待處理屬性](#pending-attributes)
- [比較模型](#comparing-models)
- [事件](#events)
    - [使用閉包](#events-using-closures)
    - [觀察者](#observers)
    - [靜音事件](#muting-events)

<a name="introduction"></a>
## 簡介

Laravel 包含了 Eloquent，這是一個物件關係對應器 (ORM)，讓與資料庫的互動變得愉快。使用 Eloquent 時，每個資料庫資料表都有一個對應的「Model」，用於與該資料表進行互動。除了從資料庫資料表中擷取記錄外，Eloquent 模型還允許您在資料表中新增、更新以及刪除記錄。

> [!NOTE]
> 在開始之前，請確保已在應用程式的 `config/database.php` 設定檔中配置資料庫連線。如需更多關於配置資料庫的資訊，請參閱 [資料庫設定文件](/docs/{{version}}/database#configuration)。


<a name="generating-model-classes"></a>
## 產生模型類別

首先，讓我們建立一個 Eloquent 模型。模型通常位於 `app\Models` 目錄中，並繼承 `Illuminate\Database\Eloquent\Model` 類別。您可以使用 `make:model` [Artisan 命令](/docs/{{version}}/artisan) 來產生一個新模型：

```shell
php artisan make:model Flight
```

如果您想在產生模型時同時產生 [資料庫遷移](/docs/{{version}}/migrations)，可以使用 `--migration` 或 `-m` 選項：

```shell
php artisan make:model Flight --migration
```

在產生模型時，您還可以產生其他各種類型的類別，例如 factories、seeders、policies、controllers 以及表單請求(Form request)。此外，您可以將這些選項組合在一起，一次建立多個類別：

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

有時候，僅僅透過瀏覽程式碼很難確定模型所有可用的屬性與關聯。相反地，請嘗試使用 `model:show` Artisan 命令，它能提供模型所有屬性與關聯的便捷概覽：

```shell
php artisan model:show Flight
```

<a name="eloquent-model-conventions"></a>
## Eloquent 模型慣例

由 `make:model` 指令產生的模型將被放置在 `app/Models` 目錄中。讓我們來檢查一個基本的模型類別，並討論一些 Eloquent 的關鍵慣例：

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

在看了上面的範例後，您可能會注意到我們並沒有告訴 Eloquent 哪個資料庫資料表對應到我們的 `Flight` 模型。根據慣例，除非明確指定其他名稱，否則將使用類別名稱的「snake case」複數形式作為資料表名稱。因此，在這種情況下，Eloquent 會假設 `Flight` 模型將記錄儲存在 `flights` 資料表中，而 `AirTrafficController` 模型則會將記錄儲存在 `air_traffic_controllers` 資料表中。

如果您的模型對應的資料庫資料表不符合此慣例，您可以使用 `Table` 屬性手動指定模型的資料表名稱：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Model;

#[Table('my_flights')]
class Flight extends Model
{
    // ...
}
```



<a name="primary-keys"></a>
### 主鍵

Eloquent 也會假設每個模型對應的資料庫資料表都有一個名為 `id` 的主鍵欄位。如有必要，您可以使用 `Table` 屬性上的 `key` 引數來指定另一個作為模型主鍵的欄位：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Model;

#[Table(key: 'flight_id')]
class Flight extends Model
{
    // ...
}
```

此外，Eloquent 假設主鍵是一個遞增的整數值，這意味著 Eloquent 將自動將主鍵轉換為整數。如果您希望使用非遞增或非數字的主鍵，您應該在 `Table` 屬性上指定 `keyType` 和 `incrementing` 引數：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Model;

#[Table(key: 'uuid', keyType: 'string', incrementing: false)]
class Flight extends Model
{
    // ...
}
```

如果您只需要禁用自動遞增 ID，可以使用 `WithoutIncrementing` 屬性：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\WithoutIncrementing;
use Illuminate\Database\Eloquent\Model;

#[WithoutIncrementing]
class Flight extends Model
{
    // ...
}
```


<a name="composite-primary-keys"></a>
#### 「複合」主鍵

Eloquent 要求每個模型至少要有一個唯一識別的 「ID」 可以作為其主鍵。「複合」主鍵並不被 Eloquent 模型支援。不過，除了資料表唯一識別的主鍵之外，您可以自由地在資料庫資料表中添加額外的多欄位唯一索引。


<a name="uuid-and-ulid-keys"></a>
### UUID 與 ULID 金鑰

您可以選擇使用 UUID 而不是自動遞增整數作為 Eloquent 模型的主鍵。UUID 是長度為 36 個字元的通用唯一英數識別碼。

如果您希望模型使用 UUID 金鑰而非自動遞增整數金鑰，可以在模型上使用 `Illuminate\Database\Eloquent\Concerns\HasUuids` trait。當然，您應該確保模型具有 [UUID 等效的主鍵欄位](/docs/{{version}}/migrations#column-method-uuid)：

```php
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Model;

class Article extends Model
{
    use HasUuids;

    // ...
}

$article = Article::create(['title' => 'Traveling to Europe']);

$article->id; // "018f2b5c-6a7f-7b12-9d6f-2f8a4e0c9c11"
```

預設情況下，`HasUuids` trait 會為您的模型產生 [UUIDv7](/docs/{{version}}/strings#method-str-uuid7) 識別碼。這些 UUID 對於索引化的資料庫儲存更有效率，因為它們可以按字典順序排序。

您可以在模型上定義 `newUniqueId` 方法來覆蓋特定模型的 UUID 產生過程。此外，您還可以透過在模型上定義 `uniqueIds` 方法來指定哪些欄位應該接收 UUID：

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

如果您願意，也可以選擇使用 「ULID」 而不是 UUID。ULID 與 UUID 類似，但長度僅為 26 個字元。與有序 UUID 一樣，ULID 可以按字典順序排序，以實現高效的資料庫索引。要使用 ULID，您應該在模型上使用 `Illuminate\Database\Eloquent\Concerns\HasUlids` trait。您還應該確保模型具有 [ULID 等效的主鍵欄位](/docs/{{version}}/migrations#column-method-ulid)：

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

預設情況下，Eloquent 期望您的模型對應的資料庫資料表中存在 `created_at` 和 `updated_at` 欄位。當模型被建立或更新時，Eloquent 會自動設定這些欄位的值。如果您不希望這些欄位由 Eloquent 自動管理，可以在模型的 `Table` 屬性上將 `timestamps` 設定為 `false`：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Model;

#[Table(timestamps: false)]
class Flight extends Model
{
    // ...
}
```

如果您只需要禁用時間戳記，可以使用 `WithoutTimestamps` 屬性：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\WithoutTimestamps;
use Illuminate\Database\Eloquent\Model;

#[WithoutTimestamps]
class Flight extends Model
{
    // ...
}
```

如果您需要自訂模型時間戳記的格式，可以使用 `Table` 屬性上的 `dateFormat` 引數。這決定了日期屬性如何在資料庫中儲存，以及當模型被序列化為陣列或 JSON 時的格式：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Model;

#[Table(dateFormat: 'U')]
class Flight extends Model
{
    // ...
}
```

如果您只需要定義日期格式，可以使用 `DateFormat` 屬性：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\DateFormat;
use Illuminate\Database\Eloquent\Model;

#[DateFormat('U')]
class Flight extends Model
{
    // ...
}
```

如果您需要自訂用於儲存時間戳記的欄位名稱，可以在模型上定義 `CREATED_AT` 和 `UPDATED_AT` 常數：

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

如果您希望在執行模型操作時不修改模型的 `updated_at` 時間戳記，可以在傳遞給 `withoutTimestamps` 方法的閉包中對模型進行操作：

```php
Model::withoutTimestamps(fn () => $post->increment('reads'));
```

<a name="database-connections"></a>
### 資料庫連線

預設情況下，所有 Eloquent 模型都會使用為您的應用程式設定的預設資料庫連線。如果您想指定在與特定模型互動時應使用的不同連線，可以使用 `Connection` 屬性：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Connection;
use Illuminate\Database\Eloquent\Model;

#[Connection('mysql')]
class Flight extends Model
{
    // ...
}
```


<a name="default-attribute-values"></a>
### 屬性預設值

預設情況下，新實例化的模型實例將不包含任何屬性值。如果您想為模型的某些屬性定義預設值，可以在模型上定義 `$attributes` 屬性。放入 `$attributes` 陣列中的屬性值應採用其原始的、「可儲存」格式，就如同剛從資料庫讀取出來一樣：

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
### 設定 Eloquent 嚴格模式

Laravel 提供了幾種方法，讓您可以在各種情況下設定 Eloquent 的行為與「嚴格程度」。

首先，`preventLazyLoading` 方法接受一個選填的布林引數，用以指示是否應防止延遲載入 (lazy loading)。例如，您可能希望僅在非正式環境中禁用延遲載入，以便即使正式環境的程式碼中不小心出現了延遲載入的關係，您的正式環境仍能繼續正常運作。通常，此方法應在應用程式 `AppServiceProvider` 的 `boot` 方法中呼叫：

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

此外，您也可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填充不可填充的屬性時拋出例外。這有助於在本地開發過程中，避免在嘗試設定尚未添加到模型 `fillable` 陣列中的屬性時發生意外錯誤：

```php
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

<a name="retrieving-models"></a>
## 擷取模型

一旦您建立了模型以及 [其關聯的資料庫資料表](/docs/{{version}}/migrations#generating-migrations)，您就可以開始從資料庫中擷取資料了。您可以將每個 Eloquent 模型視為一個強大的 [查詢產生器](/docs/{{version}}/queries)，讓您能流暢地查詢與模型關聯的資料庫資料表。模型的 `all` 方法將從模型關聯的資料庫資料表中擷取所有的記錄：

```php
use App\Models\Flight;

foreach (Flight::all() as $flight) {
    echo $flight->name;
}
```


<a name="building-queries"></a>
#### 構建查詢

Eloquent 的 `all` 方法將回傳模型資料表中的所有結果。然而，由於每個 Eloquent 模型都充當 [查詢產生器](/docs/{{version}}/queries)，您可以為查詢增加額外的限制條件，然後呼叫 `get` 方法來擷取結果：

```php
$flights = Flight::where('active', 1)
    ->orderBy('name')
    ->limit(10)
    ->get();
```

> [!NOTE]
> 由於 Eloquent 模型是查詢產生器，您應該查看 Laravel [查詢產生器](/docs/{{version}}/queries) 提供的所有方法。在撰寫 Eloquent 查詢時，您可以使用其中任何方法。


<a name="refreshing-models"></a>
#### 重新整理模型

如果您已經有一個從資料庫擷取的 Eloquent 模型實例，您可以使用 `fresh` 和 `refresh` 方法來「重新整理」該模型。`fresh` 方法會重新從資料庫擷取該模型。現有的模型實例將不會受到影響：

```php
$flight = Flight::where('number', 'FR 900')->first();

$freshFlight = $flight->fresh();
```

`refresh` 方法會使用來自資料庫的最新資料來重新填充現有的模型。此外，所有已載入的關聯也會一併被重新整理：

```php
$flight = Flight::where('number', 'FR 900')->first();

$flight->number = 'FR 456';

$flight->refresh();

$flight->number; // "FR 900"
```


<a name="collections"></a>
### 集合

正如我們所見，像 `all` 和 `get` 這樣的 Eloquent 方法會從資料庫擷取多筆記錄。然而，這些方法回傳的不是一般的 PHP 陣列，而是一個 `Illuminate\Database\Eloquent\Collection` 的實例。

Eloquent 的 `Collection` 類別繼承了 Laravel 的基礎 `Illuminate\Support\Collection` 類別，該類別提供了 [多種實用的方法](/docs/{{version}}/collections#available-methods) 來與資料集合進行互動。例如，可以使用 `reject` 方法根據呼叫的閉包結果從集合中移除模型：

```php
$flights = Flight::where('destination', 'Paris')->get();

$flights = $flights->reject(function (Flight $flight) {
    return $flight->cancelled;
});
```

除了 Laravel 基礎集合類別提供的方法外，Eloquent 集合類別還提供 [一些額外的方法](/docs/{{version}}/eloquent-collections#available-methods)，專門用於與 Eloquent 模型集合進行互動。

由於 Laravel 的所有集合都實作了 PHP 的可迭代介面 (iterable interfaces)，您可以像處理陣列一樣對集合進行迴圈操作：

```php
foreach ($flights as $flight) {
    echo $flight->name;
}
```


<a name="chunking-results"></a>
### 分塊處理結果

如果您嘗試透過 `all` 或 `get` 方法載入數萬筆 Eloquent 記錄，您的應用程式可能會耗盡記憶體。與其使用這些方法，不如使用 `chunk` 方法更有效地處理大量模型。

`chunk` 方法會擷取 Eloquent 模型的子集，並將其傳遞給閉包進行處理。由於一次只擷取目前的一塊 (chunk) Eloquent 模型，因此在處理大量模型時，`chunk` 方法能顯著降低記憶體使用量：

```php
use App\Models\Flight;
use Illuminate\Database\Eloquent\Collection;

Flight::chunk(200, function (Collection $flights) {
    foreach ($flights as $flight) {
        // ...
    }
});
```

傳遞給 `chunk` 方法的第一個引數是您希望每「塊」接收的記錄數量。作為第二個引數傳遞的閉包將在從資料庫擷取每一塊資料時被呼叫。系統將執行資料庫查詢來擷取傳遞給閉包的每一塊記錄。

如果您根據某個欄位來篩選 `chunk` 方法的結果，且在迭代結果的過程中也會更新該欄位，您應該使用 `chunkById` 方法。在這些情境中使用 `chunk` 方法可能會導致不可預期且不一致的結果。在內部，`chunkById` 方法將始終擷取 `id` 欄位大於前一塊中最後一個模型的模型：

```php
Flight::where('departed', true)
    ->chunkById(200, function (Collection $flights) {
        $flights->each->update(['departed' => false]);
    }, column: 'id');
```

由於 `chunkById` 和 `lazyById` 方法會為正在執行的查詢增加自己的 "where" 條件，因此您通常應該在閉包中將自己的條件進行 [邏輯分組](/docs/{{version}}/queries#logical-grouping)：

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
### 使用延遲集合進行分塊

`lazy` 方法的運作方式與 [ `chunk` 方法](#chunking-results) 類似，也就是在後台以分塊方式執行查詢。然而，`lazy` 方法不是直接將每一塊資料傳遞到回呼函數中，而是回傳一個扁平化的 Eloquent 模型 [LazyCollection](/docs/{{version}}/collections#lazy-collections)，讓您可以將結果視為單一串流 (stream) 進行互動：

```php
use App\Models\Flight;

foreach (Flight::lazy() as $flight) {
    // ...
}
```

如果您根據某個欄位來篩選 `lazy` 方法的結果，且在迭代結果的過程中也會更新該欄位，您應該使用 `lazyById` 方法。在內部，`lazyById` 方法將始終擷取 `id` 欄位大於前一塊中最後一個模型的模型：

```php
Flight::where('departed', true)
    ->lazyById(200, column: 'id')
    ->each->update(['departed' => false]);
```

您可以使用 `lazyByIdDesc` 方法根據 `id` 的降序來篩選結果。

<a name="cursors"></a>
### 游標

與 `lazy` 方法類似，當在迭代數以萬計的 Eloquent 模型紀錄時，可以使用 `cursor` 方法來顯著降低應用程式的記憶體消耗。

`cursor` 方法只會執行單次資料庫查詢；然而，個別的 Eloquent 模型在實際被迭代之前不會被填充 (hydrated)。因此，在迭代游標時，任何時間點記憶體中都只會保留一個 Eloquent 模型。

> [!WARNING]
> 由於 `cursor` 方法一次只會在記憶體中保留單個 Eloquent 模型，因此它無法預先載入 (eager load) 關聯。如果您需要預先載入關聯，請考慮改用 [`lazy` 方法](#chunking-using-lazy-collections)。

在內部，`cursor` 方法使用 PHP [generators](https://www.php.net/manual/en/language.generators.overview.php) 來實現此功能：

```php
use App\Models\Flight;

foreach (Flight::where('destination', 'Zurich')->cursor() as $flight) {
    // ...
}
```

`cursor` 會回傳一個 `Illuminate\Support\LazyCollection` 實例。[Lazy collections](/docs/{{version}}/collections#lazy-collections) 讓您在一次僅將單個模型載入記憶體的情況下，仍能使用典型的 Laravel 集合所提供的許多集合方法：

```php
use App\Models\User;

$users = User::cursor()->filter(function (User $user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```

雖然 `cursor` 方法比一般查詢使用的記憶體少得多（因為一次只在記憶體中保留一個 Eloquent 模型），但最終仍可能會耗盡記憶體。這是 [因為 PHP 的 PDO 驅動程式會在內部將所有原始查詢結果快取在緩衝區 (buffer) 中](https://www.php.net/manual/en/mysqlinfo.concepts.buffering.php)。如果您需要處理極大量的 Eloquent 紀錄，請考慮改用 [`lazy` 方法](#chunking-using-lazy-collections)。


<a name="advanced-subqueries"></a>
### 進階子查詢


<a name="subquery-selects"></a>
#### 子查詢選取

Eloquent 還提供了進階的子查詢支援，讓您能在單次查詢中從相關的資料表中提取資訊。例如，假設我們有一個航班目的地 `destinations` 表和一個航班 `flights` 表。`flights` 表包含一個 `arrived_at` 欄位，用來表示航班到達目的地的時間。

利用查詢產生器的 `select` 與 `addSelect` 方法提供的子查詢功能，我們可以用單次查詢選取所有的 `destinations` 以及最近到達該目的地的航班名稱：

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

此外，查詢產生器的 `orderBy` 函式也支援子查詢。延續航班的範例，我們可以使用此功能根據最後一次航班到達的時間來對所有目的地進行排序。同樣地，這可以在執行單次資料庫查詢時完成：

```php
return Destination::orderByDesc(
    Flight::select('arrived_at')
        ->whereColumn('destination_id', 'destinations.id')
        ->orderByDesc('arrived_at')
        ->limit(1)
)->get();
```

<a name="retrieving-single-models"></a>
## 擷取單一模型 / 聚合函數

除了擷取所有符合給定查詢的紀錄外，您也可以使用 `find`、`first` 或 `firstWhere` 方法來擷取單一紀錄。這些方法會回傳單一模型實例，而不是模型集合：

```php
use App\Models\Flight;

// Retrieve a model by its primary key...
$flight = Flight::find(1);

// Retrieve the first model matching the query constraints...
$flight = Flight::where('active', 1)->first();

// Alternative to retrieving the first model matching the query constraints...
$flight = Flight::firstWhere('active', 1);
```

有時如果找不到結果，您可能希望執行其他操作。`findOr` 和 `firstOr` 方法會回傳單一模型實例，或者在找不到結果時執行給定的閉包。閉包回傳的值將被視為該方法的結果：

```php
$flight = Flight::findOr(1, function () {
    // ...
});

$flight = Flight::where('legs', '>', 3)->firstOr(function () {
    // ...
});
```


<a name="not-found-exceptions"></a>
#### 找不到模型的例外

有時如果找不到模型，您可能希望拋出一個例外。這在路由或控制器中特別有用。`findOrFail` 和 `firstOrFail` 方法會擷取查詢的第一個結果；然而，如果找不到結果，將會拋出 `Illuminate\Database\Eloquent\ModelNotFoundException`：

```php
$flight = Flight::findOrFail(1);

$flight = Flight::where('legs', '>', 3)->firstOrFail();
```

如果 `ModelNotFoundException` 沒有被捕捉，將會自動向客戶端傳回 404 HTTP 回應：

```php
use App\Models\Flight;

Route::get('/api/flights/{id}', function (string $id) {
    return Flight::findOrFail($id);
});
```


<a name="retrieving-or-creating-models"></a>
### 擷取或建立模型

`firstOrCreate` 方法會嘗試使用給定的欄位/值對來定位資料庫紀錄。如果在資料庫中找不到該模型，將會插入一筆紀錄，其屬性是由第一個陣列引數與可選的第二個陣列引數合併而成的。

`firstOrNew` 方法與 `firstOrCreate` 類似，會嘗試在資料庫中定位符合給定屬性的紀錄。然而，如果找不到模型，將會回傳一個新的模型實例。請注意，`firstOrNew` 回傳的模型尚未持久化到資料庫中。您需要手動呼叫 `save` 方法來將其持久化：

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

在與 Eloquent 模型互動時，您也可以使用 Laravel [查詢建構器](/docs/{{version}}/queries) 提供的 `count`、`sum`、`max` 以及其他 [聚合函數](/docs/{{version}}/queries#aggregates)。正如您所預期的，這些方法會回傳一個純量值而非 Eloquent 模型實例：

```php
$count = Flight::where('active', 1)->count();

$max = Flight::where('active', 1)->max('price');
```

<a name="inserting-and-updating-models"></a>
## 新增與更新模型


<a name="inserts"></a>
### 新增

當然，在使用 Eloquent 時，我們不只需要從資料庫擷取模型，還需要新增新紀錄。幸運的是，Eloquent 讓這件事變得非常簡單。要將新紀錄新增到資料庫中，您應該實例化一個新模型實例並設定模型上的屬性，接著在模型實例上呼叫 `save` 方法：

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

在這個範例中，我們將來自 HTTP 請求的 `name` 欄位賦值給 `App\Models\Flight` 模型實例的 `name` 屬性。當我們呼叫 `save` 方法時，一筆紀錄將被新增到資料庫中。模型 的 `created_at` 與 `updated_at` 時間戳記會在呼叫 `save` 方法時自動設定，因此不需要手動設定它們。

如果您想在資料庫交易中儲存模型，可以使用 `saveOrFail` 方法。如果在儲存過程中拋出異常，交易將會自動回滾：

```php
$flight->saveOrFail();
```

或者，您可以使用 `create` 方法，僅用一行 PHP 語句來「儲存」一個新模型。`create` 方法會將新增的模型實例回傳給您：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，您需要在模型類別中指定 `Fillable` 或 `Guarded` 屬性。這些屬性是必要的，因為所有 Eloquent 模型預設都受到大量賦值漏洞的保護。要了解更多關於大量賦值的資訊，請參閱 [大量賦值文件](#mass-assignment)。


<a name="updates"></a>
### 更新

`save` 方法也可以用來更新已存在於資料庫中的模型。要更新模型，您應該先擷取該模型並設定任何您想要更新的屬性，接著呼叫模型的 `save` 方法。同樣地，`updated_at` 時間戳記會自動更新，因此不需要手動設定其值：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->name = 'Paris to London';

$flight->save();
```

如果您想在資料庫交易中更新模型，可以使用 `updateOrFail` 方法。如果在更新過程中拋出異常，交易將會自動回滾：

```php
$flight->updateOrFail(['name' => 'Paris to London']);
```

有時候，您可能需要更新現有模型，或者在沒有匹配模型時建立一個新模型。就像 `firstOrCreate` 方法一樣，`updateOrCreate` 方法會將模型持久化到資料庫，因此不需要手動呼叫 `save` 方法。

在下方的範例中，如果存在一個出發地 (`departure`) 為 `Oakland` 且目的地 (`destination`) 為 `San Diego` 的航班，其 `price` 和 `discounted` 欄位將被更新。如果不存在這樣的航班，則會建立一個新航班，其屬性是由第一個引數陣列與第二個引數陣列合併而成的結果：

```php
$flight = Flight::updateOrCreate(
    ['departure' => 'Oakland', 'destination' => 'San Diego'],
    ['price' => 99, 'discounted' => 1]
);
```

在使用 `firstOrCreate` 或 `updateOrCreate` 等方法時，您可能不知道是建立了一個新模型還是更新了現有模型。`wasRecentlyCreated` 屬性可用來指示該模型是否在目前的生命週期中被建立：

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

也可以對符合特定查詢的模型進行更新。在這個範例中，所有 `active` 為 1 且 `destination` 為 `San Diego` 的航班都將被標記為延遲：

```php
Flight::where('active', 1)
    ->where('destination', 'San Diego')
    ->update(['delayed' => 1]);
```

`update` 方法需要一個包含欄位與值配對的陣列，代表應該被更新的欄位。`update` 方法會回傳受影響的行數。

> [!WARNING]
> 當透過 Eloquent 執行大量更新時，更新的模型將不會觸發 `saving`、`saved`、`updating` 和 `updated` 模型事件。這是因為在執行大量更新時，模型實際上從未被擷取。


<a name="examining-attribute-changes"></a>
#### 檢查屬性變更

Eloquent 提供了 `isDirty`、`isClean` 和 `wasChanged` 方法，用來檢查模型的內部狀態，並確定其屬性自最初擷取以來發生了哪些變更。

`isDirty` 方法會判斷模型屬性自擷取以來是否已被變更。您可以將特定的屬性名稱或屬性陣列傳遞給 `isDirty` 方法，以確定是否有任何屬性是「髒的 (dirty)」。`isClean` 方法則會判斷屬性自擷取以來是否保持不變。此方法同樣接受一個選填的屬性引數：

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

`wasChanged` 方法會判斷在目前的請求週期中，模型上次儲存時是否有任何屬性被變更。如果需要，您可以傳遞屬性名稱來查看特定屬性是否被變更：

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

`getOriginal` 方法會回傳一個包含模型原始屬性的陣列，無論模型自擷取以來發生了哪些變更。如果需要，您可以傳遞特定的屬性名稱來獲取該屬性的原始值：

```php
$user = User::find(1);

$user->name; // John
$user->email; // john@example.com

$user->name = 'Jack';
$user->name; // Jack

$user->getOriginal('name'); // John
$user->getOriginal(); // Array of original attributes...
```

`getChanges` 方法會回傳一個包含模型上次儲存時變更之屬性的陣列，而 `getPrevious` 方法則會回傳模型上次儲存之前的原始屬性值陣列：

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
### 大量賦值

您可以使用 `create` 方法透過單一 PHP 敘述來「儲存」一個新模型。該方法會將新增的模型實例回傳給您：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在利用 `create` 方法之前，您必須在模型類別中指定 `Fillable` 或 `Guarded` 屬性。由於所有 Eloquent 模型預設都受到大量賦值漏洞的保護，因此需要這些屬性。

大量賦值漏洞發生在使用者傳遞了非預期的 HTTP 請求欄位，且該欄位修改了您未預期的資料庫欄位時。例如，惡意使用者可能會透過 HTTP 請求發送 `is_admin` 參數，接著該參數被傳遞到模型的 `create` 方法中，使該使用者能將自己的權限提升為管理員。

因此，在開始之前，您應該定義哪些模型屬性可以被大量賦值。您可以使用模型上的 `Fillable` 屬性來達成此目的。例如，讓我們將 `Flight` 模型的 `name` 屬性設為可大量賦值：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Model;

#[Fillable(['name'])]
class Flight extends Model
{
    // ...
}
```

一旦您指定了哪些屬性可被大量賦值，您就可以使用 `create` 方法在資料庫中新增一筆記錄。`create` 方法會回傳新建立的模型實例：

```php
$flight = Flight::create(['name' => 'London to Paris']);
```

如果您已經有一個模型實例，可以使用 `fill` 方法利用屬性陣列來填充它：

```php
$flight->fill(['name' => 'Amsterdam to Frankfurt']);
```


<a name="mass-assignment-json-columns"></a>
#### 大量賦值與 JSON 欄位

在賦值 JSON 欄位時，每個欄位的可大量賦值鍵值必須在模型的 `Fillable` 屬性中指定。基於安全性考量，在使用 `Guarded` 屬性時，Laravel 不支援更新巢狀 JSON 屬性：

```php
use Illuminate\Database\Eloquent\Attributes\Fillable;

#[Fillable(['options->enabled'])]
class Flight extends Model
{
    // ...
}
```


<a name="allowing-mass-assignment"></a>
#### 允許大量賦值

如果您希望將所有屬性都設為可大量賦值，可以使用模型上的 `Unguarded` 屬性。如果您選擇取消對模型的保護，請務必小心，始終手動建構傳遞給 Eloquent `fill`、`create` 與 `update` 方法的陣列：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Unguarded;
use Illuminate\Database\Eloquent\Model;

#[Unguarded]
class Flight extends Model
{
    // ...
}
```


<a name="mass-assignment-exceptions"></a>
#### 大量賦值例外

預設情況下，執行大量賦值操作時，未包含在 `Fillable` 屬性中的屬性會被靜默捨棄。在生產環境中，這是預期的行為；然而，在本地開發期間，這可能會導致您困惑為什麼模型的變更沒有生效。

如果您希望在嘗試填充不可賦值屬性時讓 Laravel 拋出例外，可以呼叫 `preventSilentlyDiscardingAttributes` 方法。通常，此方法應該在應用程式 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
### Upserts

Eloquent 的 `upsert` 方法可以用於在單一的原子操作中更新或建立記錄。該方法的第一個引數包含要新增或更新的值，而第二個引數則是列出在關聯資料表中可用於唯一識別記錄的欄位。第三個也是最後一個引數是一個陣列，包含在資料庫中已存在匹配記錄時應更新的欄位。如果模型啟用了時間戳記，`upsert` 方法會自動設定 `created_at` 與 `updated_at` 時間戳記：

```php
Flight::upsert([
    ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
    ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
], uniqueBy: ['departure', 'destination'], update: ['price']);
```

> [!WARNING]
> 除 SQL Server 外，所有資料庫都要求 `upsert` 方法第二個引數中的欄位必須具有「主鍵 (primary)」或「唯一 (unique)」索引。此外，MariaDB 與 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個引數，並始終使用資料表的「主鍵」與「唯一」索引來偵測既有記錄。

<a name="deleting-models"></a>
## 刪除模型

若要刪除模型，您可以呼叫模型實例上的 `delete` 方法：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->delete();
```

如果您想在資料庫交易中刪除模型，可以使用 `deleteOrFail` 方法。如果在刪除過程中拋出異常，交易將會自動回滾：

```php
$flight->deleteOrFail();
```


<a name="deleting-an-existing-model-by-its-primary-key"></a>
#### 透過主鍵刪除現有模型

在上述範例中，我們是在呼叫 `delete` 方法之前先從資料庫擷取模型。然而，如果您知道模型的主鍵，您可以透過呼叫 `destroy` 方法，在不需要顯式擷取模型的情況下直接刪除模型。除了接受單一主鍵外，`destroy` 方法還接受多個主鍵、主鍵陣列或主鍵的 [collection](/docs/{{version}}/collections)：

```php
Flight::destroy(1);

Flight::destroy(1, 2, 3);

Flight::destroy([1, 2, 3]);

Flight::destroy(collect([1, 2, 3]));
```

如果您正在使用 [軟刪除模型](#soft-deleting)，您可以透過 `forceDestroy` 方法永久刪除模型：

```php
Flight::forceDestroy(1);
```

> [!WARNING]
> `destroy` 方法會單獨載入每個模型並呼叫 `delete` 方法，以便為每個模型正確地發送 `deleting` 和 `deleted` 事件。


<a name="deleting-models-using-queries"></a>
#### 使用查詢刪除模型

當然，您可以建立一個 Eloquent 查詢來刪除所有符合查詢條件的模型。在本範例中，我們將刪除所有被標記為不活用的航班。與大量更新一樣，大量刪除不會為被刪除的模型發送模型事件：

```php
$deleted = Flight::where('active', 0)->delete();
```

若要刪除資料表中的所有模型，您應該執行一個不帶任何條件的查詢：

```php
$deleted = Flight::query()->delete();
```

> [!WARNING]
> 當透過 Eloquent 執行大量刪除陳述式時，不會為被刪除的模型發送 `deleting` 和 `deleted` 模型事件。這是因為在執行刪除陳述式時，模型實際上從未被擷取。


<a name="soft-deleting"></a>
### 軟刪除

除了真正地從資料庫中移除紀錄外，Eloquent 還可以「軟刪除」模型。當模型被軟刪除時，它們實際上並沒有從資料庫中移除。相反地，模型上會設定一個 `deleted_at` 屬性，用以表示模型被「刪除」的日期和時間。若要為模型啟用軟刪除，請將 `Illuminate\Database\Eloquent\SoftDeletes` trait 添加到模型中：

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

您還應該在資料庫資料表中添加 `deleted_at` 欄位。Laravel 的 [schema builder](/docs/{{version}}/migrations) 包含了一個用於建立此欄位的輔助方法：

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

現在，當您在模型上呼叫 `delete` 方法時，`deleted_at` 欄位將被設定為目前的日期和時間。然而，模型的資料庫紀錄仍會保留在資料表中。在查詢使用軟刪除的模型時，軟刪除的模型將自動從所有查詢結果中排除。

若要判斷給定的模型實例是否已被軟刪除，您可以使用 `trashed` 方法：

```php
if ($flight->trashed()) {
    // ...
}
```


<a name="restoring-soft-deleted-models"></a>
#### 還原軟刪除的模型

有時您可能希望「取消刪除」一個被軟刪除的模型。若要還原軟刪除的模型，您可以呼叫模型實例上的 `restore` 方法。`restore` 方法會將模型的 `deleted_at` 欄位設定為 `null`：

```php
$flight->restore();
```

您也可以在查詢中使用 `restore` 方法來還原多個模型。同樣地，與其他「大量」操作一樣，這不會為被還原的模型發送任何模型事件：

```php
Flight::withTrashed()
    ->where('airline_id', 1)
    ->restore();
```

`restore` 方法也可以在建立 [relationship](/docs/{{version}}/eloquent-relationships) 查詢時使用：

```php
$flight->history()->restore();
```


<a name="permanently-deleting-models"></a>
#### 永久刪除模型

有時您可能需要真正地從資料庫中移除模型。您可以使用 `forceDelete` 方法將軟刪除的模型永久從資料庫資料表中移除：

```php
$flight->forceDelete();
```

您也可以在建立 Eloquent 關係查詢時使用 `forceDelete` 方法：

```php
$flight->history()->forceDelete();
```


<a name="querying-soft-deleted-models"></a>
### 查詢軟刪除的模型


<a name="including-soft-deleted-models"></a>
#### 包含軟刪除的模型

如前所述，軟刪除的模型將自動從查詢結果中排除。然而，您可以透過在查詢上呼叫 `withTrashed` 方法，強制將軟刪除的模型包含在查詢結果中：

```php
use App\Models\Flight;

$flights = Flight::withTrashed()
    ->where('account_id', 1)
    ->get();
```

`withTrashed` 方法也可以在建立 [relationship](/docs/{{version}}/eloquent-relationships) 查詢時呼叫：

```php
$flight->history()->withTrashed()->get();
```


<a name="retrieving-only-soft-deleted-models"></a>
#### 僅擷取軟刪除的模型

`onlyTrashed` 方法將**僅**擷取軟刪除的模型：

```php
$flights = Flight::onlyTrashed()
    ->where('airline_id', 1)
    ->get();
```

<a name="pruning-models"></a>
## 修剪模型

有時候您可能想要定期刪除不再需要的模型。為了達成這個目的，您可以將 `Illuminate\Database\Eloquent\Prunable` 或 `Illuminate\Database\Eloquent\MassPrunable` trait 新增至您想要定期修剪的模型中。在模型中新增其中一個 trait 後，請實作一個 `prunable` 方法，該方法需回傳一個 Eloquent 查詢建構器，用以找出不再需要的模型：

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

當將模型標記為 `Prunable` 時，您也可以在模型上定義一個 `pruning` 方法。此方法會在模型被刪除之前被呼叫。在模型從資料庫中被永久移除之前，此方法可用於刪除與該模型相關的任何額外資源（例如儲存的文件）：

```php
/**
 * Prepare the model for pruning.
 */
protected function pruning(): void
{
    // ...
}
```

在設定好可修剪的模型後，您應該在應用程式的 `routes/console.php` 檔案中排程 `model:prune` Artisan 命令。您可以自由選擇此命令執行的適當間隔時間：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('model:prune')->daily();
```

在幕後，`model:prune` 命令會自動偵測應用程式 `app/Models` 目錄中的 "Prunable" 模型。如果您的模型位於其他位置，您可以使用 `--model` 選項來指定模型類別名稱：

```php
Schedule::command('model:prune', [
    '--model' => [Address::class, Flight::class],
])->daily();
```

如果您希望在修剪所有偵測到的模型時，排除某些特定的模型，可以使用 `--except` 選項：

```php
Schedule::command('model:prune', [
    '--except' => [Address::class, Flight::class],
])->daily();
```

您可以透過執行帶有 `--pretend` 選項的 `model:prune` 命令來測試您的 `prunable` 查詢。在模擬執行（pretending）時，`model:prune` 命令僅會報告如果該命令實際執行時，將有多少筆紀錄被修剪：

```shell
php artisan model:prune --pretend
```

> [!WARNING]
> 如果軟刪除的模型符合修剪查詢，它們將被永久刪除 (`forceDelete`)。


<a name="mass-pruning"></a>
#### 大量修剪

當模型被標記為 `Illuminate\Database\Eloquent\MassPrunable` trait 時，模型將使用大量刪除查詢從資料庫中刪除。因此，`pruning` 方法不會被呼叫，`deleting` 與 `deleted` 模型事件也不會被觸發。這是因為模型在刪除前從未被實際擷取，因此使修剪過程更加高效：

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

您可以使用 `replicate` 方法來建立一個現有模型實體的未儲存副本。當您有許多共用相同屬性的模型實體時，此方法特別有用：

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

若要排除一個或多個屬性不被複製到新模型中，您可以將陣列傳遞給 `replicate` 方法：

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
## 查詢範圍 (Query Scopes)


<a name="global-scopes"></a>
### 全域範圍

全域範圍允許您為給定模型的所有查詢添加限制。Laravel 自身的 [軟刪除](#soft-deleting) 功能就是利用全域範圍來確保僅從資料庫中擷取「未刪除」的模型。編寫自己的全域範圍可以提供一種方便且簡單的方法，確保針對給定模型的所有查詢都包含特定的限制。


<a name="generating-scopes"></a>
#### 產生範圍

若要產生新的全域範圍，您可以使用 `make:scope` Artisan 指令，該指令會將產生的範圍放置在應用程式的 `app/Models/Scopes` 目錄中：

```shell
php artisan make:scope AncientScope
```


<a name="writing-global-scopes"></a>
#### 編寫全域範圍

編寫全域範圍非常簡單。首先，使用 `make:scope` 指令產生一個實作 `Illuminate\Database\Eloquent\Scope` 介面的類別。`Scope` 介面要求您實作一個方法：`apply`。`apply` 方法可根據需要向查詢添加 `where` 限制或其他類型的子句：

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
> 如果您的全域範圍正在向查詢的 select 子句添加欄位，您應該使用 `addSelect` 方法而非 `select`。這可以防止意外替換查詢中現有的 select 子句。


<a name="applying-global-scopes"></a>
#### 套用全域範圍

若要將全域範圍分配給模型，您只需在模型上放置 `ScopedBy` 屬性：

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

或者，您可以透過覆寫模型的 `booted` 方法並呼叫模型的 `addGlobalScope` 方法來手動註冊全域範圍。`addGlobalScope` 方法僅接受一個範圍實例作為其引數：

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

在將上述範例中的範圍添加到 `App\Models\User` 模型後，呼叫 `User::all()` 方法將執行以下 SQL 查詢：

```sql
select * from `users` where `created_at` < 0021-02-18 00:00:00
```


<a name="anonymous-global-scopes"></a>
#### 匿名全域範圍

Eloquent 還允許您使用閉包定義全域範圍，這對於不需要獨立類別的簡單範圍特別有用。使用閉包定義全域範圍時，您應該在 `addGlobalScope` 方法的第一個引數中提供一個您自定義的範圍名稱：

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

如果您想在給定查詢中移除全域範圍，可以使用 `withoutGlobalScope` 方法。此方法僅接受全域範圍的類別名稱作為其引數：

```php
User::withoutGlobalScope(AncientScope::class)->get();
```

或者，如果您是使用閉包定義全域範圍，則應傳遞您分配給該全域範圍的字串名稱：

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
### 區域範圍

區域範圍允許您定義一組常用的查詢限制，以便在整個應用程式中輕鬆重複使用。例如，您可能需要頻繁地擷取所有被視為「熱門」的使用者。若要定義範圍，請將 `Scope` 屬性添加到 Eloquent 方法中。

範圍應始終返回相同的查詢建立者 (query builder) 實例或 `void`：

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
#### 使用區域範圍

定義好範圍後，您可以在查詢模型時呼叫範圍方法。您甚至可以鏈接呼叫多個不同的範圍：

```php
use App\Models\User;

$users = User::popular()->active()->orderBy('created_at')->get();
```

透過 `or` 查詢運算子組合多個 Eloquent 模型範圍時，可能需要使用閉包以實現正確的 [邏輯分組](/docs/{{version}}/queries#logical-grouping)：

```php
$users = User::popular()->orWhere(function (Builder $query) {
    $query->active();
})->get();
```

然而，由於這樣做可能較為繁瑣，Laravel 提供了一個「高階」的 `orWhere` 方法，允許您流暢地鏈接範圍而無需使用閉包：

```php
$users = User::popular()->orWhere->active()->get();
```


<a name="dynamic-scopes"></a>
#### 動態範圍

有時您可能希望定義一個可以接受參數的範圍。首先，只需將額外的參數添加到範圍方法的簽章中。範圍參數應定義在 `$query` 參數之後：

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

將預期引數添加到範圍方法的簽章後，您可以在呼叫範圍時傳遞這些引數：

```php
$users = User::ofType('admin')->get();
```

<a name="pending-attributes"></a>
### 待處理屬性

如果您想使用範圍 (scopes) 來建立與限制該範圍時所使用的屬性相同的模型，您可以在建構範圍查詢時使用 `withAttributes` 方法：

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

`withAttributes` 方法會使用給定的屬性將 `where` 條件添加到查詢中，同時也會將這些屬性添加到透過該範圍建立的任何模型中：

```php
$draft = Post::draft()->create(['title' => 'In Progress']);

$draft->hidden; // true
```

若要指示 `withAttributes` 方法不要將 `where` 條件添加到查詢中，您可以將 `asConditions` 引數設定為 `false`：

```php
$query->withAttributes([
    'hidden' => true,
], asConditions: false);
```

<a name="comparing-models"></a>
## 比較模型

有時候您可能需要判斷兩個模型是否為「相同」的模型。您可以使用 `is` 與 `isNot` 方法來快速驗證兩個模型是否具有相同的主鍵、資料表以及資料庫連線：

```php
if ($post->is($anotherPost)) {
    // ...
}

if ($post->isNot($anotherPost)) {
    // ...
}
```

在使用 `belongsTo`、`hasOne`、`morphTo` 與 `morphOne` [關聯](/docs/{{version}}/eloquent-relationships) 時，也可以使用 `is` 與 `isNot` 方法。當您想要比較一個關聯模型，且不想發出查詢來擷取該模型時，這個方法特別有用：

```php
if ($post->author()->is($user)) {
    // ...
}
```

<a name="events"></a>
## 事件

> [!NOTE]
> 想要將您的 Eloquent 事件直接廣播到您的客戶端應用程式嗎？請參閱 Laravel 的 [模型事件廣播](/docs/{{version}}/broadcasting#model-broadcasting)。

Eloquent 模型會發出數個事件，讓您能夠在模型的生命週期中於以下時刻掛鉤：`retrieved`、`creating`、`created`、`updating`、`updated`、`saving`、`saved`、`deleting`、`deleted`、`trashed`、`forceDeleting`、`forceDeleted`、`restoring`、`restored` 以及 `replicating`。

當從資料庫擷取現有模型時，會發出 `retrieved` 事件。當新模型第一次被儲存時，會發出 `creating` 與 `created` 事件。當現有模型被修改並呼叫 `save` 方法時，會發出 `updating` / `updated` 事件。當模型被建立或更新時，會發出 `saving` / `saved` 事件 —— 即使模型的屬性沒有被改變。以 `-ing` 結尾的事件名稱在模型變更被持久化 (persisted) 之前發出，而以 `-ed` 結尾的事件則在變更被持久化之後發出。

若要開始監聽模型事件，請在您的 Eloquent 模型中定義一個 `$dispatchesEvents` 屬性。此屬性將 Eloquent 模型生命週期的各個時間點映射到您自定義的 [事件類別](/docs/{{version}}/events)。每個模型事件類別應該預期透過其建構子接收到受影響模型的實例：

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

在定義並映射您的 Eloquent 事件後，您可以使用 [事件監聽器](/docs/{{version}}/events#defining-listeners) 來處理這些事件。

> [!WARNING]
> 當透過 Eloquent 執行大量更新或刪除查詢時，受影響的模型不會發出 `saved`、`updated`、`deleting` 與 `deleted` 模型事件。這是因為在執行大量更新或刪除時，模型實際上從未被擷取。


<a name="events-using-closures"></a>
### 使用閉包

您可以使用註冊閉包來替代自定義事件類別，使閉包在各種模型事件發出時執行。通常，您應該在模型的 `booted` 方法中註冊這些閉包：

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

如果有需要，您可以在註冊模型事件時利用 [可佇列的匿名事件監聽器](/docs/{{version}}/events#queueable-anonymous-event-listeners)。這將指示 Laravel 使用您應用程式的 [佇列](/docs/{{version}}/queues) 在背景執行模型事件監聽器：

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

如果您在監聽某個模型的許多事件，您可以使用觀察者將所有監聽器分組到單一類別中。觀察者類別的方法名稱反映了您希望監聽的 Eloquent 事件。這些方法中的每一個都會接收受影響的模型作為唯一的引數。`make:observer` Artisan 指令是建立新觀察者類別最簡單的方式：

```shell
php artisan make:observer UserObserver --model=User
```

此指令會將新的觀察者放置在您的 `app/Observers` 目錄中。如果此目錄不存在，Artisan 會為您建立。您新建立的觀察者將如下所示：

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

若要註冊觀察者，您可以將 `ObservedBy` 屬性放置在對應的模型上：

```php
use App\Observers\UserObserver;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy([UserObserver::class])]
class User extends Authenticatable
{
    //
}
```

或者，您也可以透過在您想要觀察的模型上呼叫 `observe` 方法來手動註冊觀察者。您可以在應用程式 `AppServiceProvider` 類別的 `boot` 方法中註冊觀察者：

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
> 觀察者還可以監聽其他事件，例如 `saving` 與 `retrieved`。這些事件在 [事件](#events) 文件中有詳細說明。


<a name="observers-and-database-transactions"></a>
#### 觀察者與資料庫交易

當模型在資料庫交易 (transaction) 中被建立時，您可能希望指示觀察者僅在資料庫交易提交 (committed) 之後才執行其事件處理程式。您可以透過在觀察者上實作 `ShouldHandleEventsAfterCommit` 介面來達成此目的。如果目前沒有資料庫交易在進行，事件處理程式將立即執行：

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

您偶爾可能需要暫時「靜音」模型發出的所有事件。您可以使用 `withoutEvents` 方法來達成。`withoutEvents` 方法僅接收一個閉包作為引數。在此閉包中執行的任何程式碼都不會發出模型事件，且閉包回傳的任何值都將由 `withoutEvents` 方法回傳：

```php
use App\Models\User;

$user = User::withoutEvents(function () {
    User::findOrFail(1)->delete();

    return User::find(2);
});
```


<a name="saving-a-single-model-without-events"></a>
#### 在不發出事件的情況下儲存單一模型

有時您可能希望在不發出任何事件的情況下「儲存」特定模型。您可以使用 `saveQuietly` 方法來達成：

```php
$user = User::findOrFail(1);

$user->name = 'Victoria Faith';

$user->saveQuietly();
```

您也可以在不發出任何事件的情況下對特定模型進行「更新」、「刪除」、「軟刪除」、「還原」以及「複製」：

```php
$user->deleteQuietly();
$user->forceDeleteQuietly();
$user->restoreQuietly();
```