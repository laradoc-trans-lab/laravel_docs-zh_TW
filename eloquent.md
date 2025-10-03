# Eloquent：入門

- [簡介](#introduction)
- [產生 Model 類別](#generating-model-classes)
- [Eloquent Model 慣例](#eloquent-model-conventions)
    - [資料表名稱](#table-names)
    - [主鍵](#primary-keys)
    - [UUID 與 ULID 鍵](#uuid-and-ulid-keys)
    - [時間戳](#timestamps)
    - [資料庫連線](#database-connections)
    - [預設屬性值](#default-attribute-values)
    - [設定 Eloquent 嚴格模式](#configuring-eloquent-strictness)
- [取出 Model](#retrieving-models)
    - [Collection](#collections)
    - [分塊取出結果](#chunking-results)
    - [使用 Lazy Collection 來分塊](#chunking-using-lazy-collections)
    - [Cursor](#cursors)
    - [進階子查詢](#advanced-subqueries)
- [取出單一 Model / 彙總](#retrieving-single-models)
    - [取出或建立 Model](#retrieving-or-creating-models)
    - [取出彙總值](#retrieving-aggregates)
- [新增與更新 Model](#inserting-and-updating-models)
    - [新增](#inserts)
    - [更新](#updates)
    - [Mass Assignment](#mass-assignment)
    - [Upsert](#upserts)
- [刪除 Model](#deleting-models)
    - [軟刪除](#soft-deleting)
    - [查詢軟刪除的 Model](#querying-soft-deleted-models)
- [修剪 Model](#pruning-models)
- [複製 Model](#replicating-models)
- [查詢作用域](#query-scopes)
    - [全域作用域](#global-scopes)
    - [區域作用域](#local-scopes)
    - [待定屬性](#pending-attributes)
- [比較 Model](#comparing-models)
- [事件](#events)
    - [使用閉包](#events-using-closures)
    - [Observer](#observers)
    - [靜音事件](#muting-events)

<a name="introduction"></a>
## 簡介

Laravel 內建了 Eloquent，這是一個物件關聯對應 (Object-Relational Mapper, ORM)，能讓開發者愉快地與資料庫互動。使用 Eloquent 時，每個資料表都會有對應的「Model」來與該資料表互動。除了從資料表取出紀錄外，Eloquent models 也允許你新增、更新、與刪除資料表中的紀錄。

> [!NOTE]
> 在開始前，請務必在應用程式的 `config/database.php` 設定檔中設定好資料庫連線。想了解更多有關設定資料庫的資訊，請參考[資料庫設定文件](/docs/{{version}}/database#configuration)。

<a name="generating-model-classes"></a>
## 產生 Model 類別

首先，我們先來建立一個 Eloquent model。Models 通常會放在 `app\Models` 目錄下，並繼承 `Illuminate\Database\Eloquent\Model` 類別。你可以使用 `make:model` [Artisan 指令](/docs/{{version}}/artisan) 來產生新的 model：

```shell
php artisan make:model Flight
```

若想在產生 model 時也產生[資料庫遷移](/docs/{{version}}/migrations)，可使用 `--migration` 或 `-m` 選項：

```shell
php artisan make:model Flight --migration
```

產生 model 時，你也可以產生各種類型的類別，如 factories、seeders、policies、controllers、以及 form requests。此外，這些選項也可以組合起來，一次建立多個類別：

```shell

# 產生一個 Model 與一個 FlightFactory 類別...
php artisan make:model Flight --factory
php artisan make:model Flight -f


# 產生一個 Model 與一個 FlightSeeder 類別...
php artisan make:model Flight --seed
php artisan make:model Flight -s


# 產生一個 Model 與一個 FlightController 類別...
php artisan make:model Flight --controller
php artisan make:model Flight -c


# 產生一個 Model、一個 FlightController Resource 類別、以及 Form Request 類別...
php artisan make:model Flight --controller --resource --requests
php artisan make:model Flight -crR


# 產生一個 Model 與一個 FlightPolicy 類別...
php artisan make:model Flight --policy


# 產生一個 Model、一個 Migration、Factory、Seeder、與 Controller...
php artisan make:model Flight -mfsc


# 快捷指令，用以產生 Model、Migration、Factory、Seeder、Policy、Controller、與 Form Request...
php artisan make:model Flight --all
php artisan make:model Flight -a

# 產生 Pivot Model...
php artisan make:model Member --pivot
php artisan make:model Member -p
```

<a name="inspecting-models"></a>
#### 檢查 Model

有時候，光是瀏覽程式碼可能很難判斷出一個 Model 所有可用的屬性與關聯。不妨改用 `model:show` 這個 Artisan 指令，該指令會提供一個方便的總覽，其中包含該 Model 的所有屬性與關聯：

```shell
php artisan model:show Flight
```

<a name="eloquent-model-conventions"></a>
## Eloquent Model 慣例

由 `make:model` 指令產生的 Model 會被放在 `app/Models` 目錄下。讓我們來看看一個基本的 Model 類別，並討論一些 Eloquent 的主要慣例：

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

看過上面的範例後，你可能已經注意到了，我們並沒有告訴 Eloquent 我們的 `Flight` Model 對應到哪個資料表。依照慣例，除非有另外明確指定，否則會使用類別名稱的「蛇形命名法 (snake case)」、複數形式的名稱來當作資料表名稱。因此，在這個例子中，Eloquent 會假設 `Flight` Model 將記錄儲存在 `flights` 資料表中，而 `AirTrafficController` Model 則會將記錄儲存在 `air_traffic_controllers` 資料表中。

若你的 Model 所對應的資料表不符合這個慣例，可以手動在 Model 上定義一個 `table` 屬性來指定 Model 的資料表名稱：

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

Eloquent 也會假設每個 Model 對應的資料表都有一個名為 `id` 的主鍵欄位。如有需要，你也可以在 Model 上定義一個 Protected 的 `$primaryKey` 屬性來指定另一個欄位作為 Model 的主鍵：

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

此外，Eloquent 假設主鍵是一個遞增的整數值，這代表 Eloquent 會自動將主鍵轉型為一個整數。若想使用非遞增或非數字的主鍵，則必須在 Model 上定義一個 Public 的 `$incrementing` 屬性，並將其設為 `false`：

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

若你的 Model 的主鍵不是整數，則應在 Model 上定義一個 Protected 的 `$keyType` 屬性。該屬性的值應為 `string`：

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
#### 「複合 (Composite)」主鍵

Eloquent 要求每個 Model 至少要有一個可作為其主鍵的唯一識別「ID」。Eloquent Model 不支援「複合 (Composite)」主鍵。不過，除了資料表的唯一識別主鍵外，你也可以自由地在資料表上新增額外的多欄位唯一索引。


<a name="uuid-and-ulid-keys"></a>
### UUID 與 ULID 鍵

除了使用自動遞增的整數作為 Eloquent Model 的主鍵外，你也可以選擇改用 UUID。UUID 是通用唯一識別碼 (universally unique identifier)，為長度 36 個字元的英數識別碼。

若想讓 Model 使用 UUID 鍵而非自動遞增的整數鍵，可以在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUuids` Trait。當然，請務必確保該 Model 有一個 [UUID 相容的主鍵欄位](/docs/{{version}}/migrations#column-method-uuid)：

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

預設情況下，`HasUuids` Trait 會為你的 Model 產生 [「有序的 (ordered)」UUID](/docs/{{version}}/strings#method-str-ordered-uuid)。這些 UUID 在索引資料庫儲存上更有效率，因為它們可以按字典順序排序。

我們可以為給定的 Model 定義一個 `newUniqueId` 方法來覆寫 UUID 的產生過程。此外，也可以在 Model 上定義一個 `uniqueIds` 方法來指定哪些欄位應使用 UUID：

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

若有需要，也可以選擇使用「ULID」而非 UUID。ULID 與 UUID 類似；不過，ULID 只有 26 個字元長。與有序 UUID 一樣，ULID 也可依字典順序排序，以提供有效率的資料庫索引。若要使用 ULID，應在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUlids` Trait。此外，也請務必確保該 Model 有個 [ULID 相容的主鍵欄位](/docs/{{version}}/migrations#column-method-ulid)：

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
### 時間戳

預設情況下，Eloquent 會預期 Model 對應的資料表中有 `created_at` 與 `updated_at` 欄位。在建立或更新 Model 時，Eloquent 會自動設定這些欄位的值。若不希望 Eloquent 自動管理這些欄位，則應在 Model 上定義一個 `$timestamps` 屬性，並將其值設為 `false`：

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

若需要自訂 Model 時間戳的格式，請在 Model 上設定 `$dateFormat` 屬性。此屬性會決定日期屬性儲存在資料庫中的格式，以及在將 Model 序列化為陣列或 JSON 時的格式：

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

若需要自訂用來儲存時間戳的欄位名稱，可以在 Model 上定義 `CREATED_AT` 與 `UPDATED_AT` 常數：

```php
<?php

class Flight extends Model
{
    const CREATED_AT = 'creation_date';
    const UPDATED_AT = 'updated_date';
}
```

若想在執行 Model 操作時不修改 Model 的 `updated_at` 時間戳，可以在傳給 `withoutTimestamps` 方法的閉包中操作 Model：

```php
Model::withoutTimestamps(fn () => $post->increment('reads'));
```


<a name="database-connections"></a>
### 資料庫連線

預設情況下，所有的 Eloquent Model 都會使用應用程式中設定的預設資料庫連線。若想在與特定 Model 互動時指定一個不同的連線，則應在該 Model 上定義一個 `$connection` 屬性：

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

預設情況下，一個新實體化的 Model 實體不會包含任何屬性值。若想為 Model 的某些屬性定義預設值，可以在 Model 上定義一個 `$attributes` 屬性。放在 `$attributes` 陣列中的屬性值應為其原始、「可儲存」的格式，就好像剛從資料庫中讀取出來一樣：

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

Laravel 提供了數種方法，能讓你在不同情況下設定 Eloquent 的行為與「嚴格度」。

首先，`preventLazyLoading` 方法可接受一個選用的布林引數，用來指出是否應防止 Lazy Loading (延遲載入)。舉例來說，你可能只希望在非正式上線環境中停用 Lazy Loading，這樣即使在正式上線的程式碼中不小心出現了 Lazy Loading 的關聯，正式上線環境也能繼續正常運作。一般來說，這個方法應在應用程式 `AppServiceProvider` 的 `boot` 方法中叫用：

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

此外，我們也可以叫用 `preventSilentlyDiscardingAttributes` 方法，來讓 Laravel 在嘗試填充 (fill) 無法填充的屬性時擲回例外。這有助於防止在本地開發時，因嘗試設定未被加入 Model `fillable` 陣列的屬性而產生非預期的錯誤：

```php
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

<a name="retrieving-models"></a>
## 取出 Model

建立好 Model 與其[對應的資料庫資料表](/docs/{{version}}/migrations#generating-migrations)後，就可以開始從資料庫中取出資料了。可以把每個 Eloquent Model 都想成一個功能強大的[查詢產生器](/docs/{{version}}/queries)，能讓開發者流暢地查詢與該 Model 關聯的資料庫資料表。Model 的 `all` 方法會取出該 Model 對應資料表中的所有紀錄：

```php
use App\Models\Flight;

foreach (Flight::all() as $flight) {
    echo $flight->name;
}
```


<a name="building-queries"></a>
#### 建立查詢

Eloquent 的 `all` 方法會回傳該 Model 資料表中的所有結果。不過，因為每個 Eloquent Model 也都是個[查詢產生器](/docs/{{version}}/queries)，所以我們也可以在查詢上加上額外的限制，然後再呼叫 `get` 方法來取得結果：

```php
$flights = Flight::where('active', 1)
    ->orderBy('name')
    ->limit(10)
    ->get();
```

> [!NOTE]
> 由於 Eloquent Model 就是查詢產生器，建議各位讀者閱讀 Laravel [查詢產生器](/docs/{{version}}/queries)所提供的所有方法。在撰寫 Eloquent 查詢時，可以使用這些方法中的任何一個。


<a name="refreshing-models"></a>
#### 重新整理 Model

若手上已經有一個從資料庫中取出的 Eloquent Model 實體，可以使用 `fresh` 與 `refresh` 方法來「重新整理 (Refresh)」該 Model。`fresh` 方法會從資料庫中重新取出該 Model。原有的 Model 實體則不會受影響：

```php
$flight = Flight::where('number', 'FR 900')->first();

$freshFlight = $flight->fresh();
```

`refresh` 方法會使用資料庫中最新的資料來重新 Hydrate (填充) 現有的 Model。此外，所有已載入的關聯也都會被一併重新整理：

```php
$flight = Flight::where('number', 'FR 900')->first();

$flight->number = 'FR 456';

$flight->refresh();

$flight->number; // "FR 900"
```


<a name="collections"></a>
### Collection

如我們所見，像 `all` 與 `get` 等 Eloquent 方法會從資料庫中取出多筆紀錄。不過，這些方法不會回傳純 PHP 陣列，而是會回傳 `Illuminate\Database\Eloquent\Collection` 的實體。

Eloquent 的 `Collection` 類別繼承了 Laravel 基礎的 `Illuminate\Support\Collection` 類別，該基礎類別提供了[多種實用的方法](/docs/{{version}}/collections#available-methods)來操作資料 Collection。舉例來說，`reject` 方法可用來根據叫用閉包的結果來從 Collection 中移除 Model：

```php
$flights = Flight::where('destination', 'Paris')->get();

$flights = $flights->reject(function (Flight $flight) {
    return $flight->cancelled;
});
```

除了 Laravel 基礎 Collection 類別提供的方法外，Eloquent Collection 類別還提供了[一些額外的方法](/docs/{{version}}/eloquent-collections#available-methods)，這些方法是專門用來操作 Eloquent Model Collection 的。

由於 Laravel 所有的 Collection 都有實作 PHP 的可迭代 (iterable) 介面，因此可以像陣列一樣疊代 Collection：

```php
foreach ($flights as $flight) {
    echo $flight->name;
}
```


<a name="chunking-results"></a>
### 分塊取出結果

若試圖透過 `all` 或 `get` 方法載入數萬筆 Eloquent 紀錄，應用程式可能會耗盡記憶體。相較於使用這些方法，`chunk` 方法可用來更有效率地處理大量的 Model。

`chunk` 方法會取出一部分的 Eloquent Model，並將其傳入閉包中進行處理。由於一次只會取出目前的 Model 區塊，因此在使用大量 Model 時，`chunk` 方法可大幅降低記憶體用量：

```php
use App\Models\Flight;
use Illuminate\Database\Eloquent\Collection;

Flight::chunk(200, function (Collection $flights) {
    foreach ($flights as $flight) {
        // ...
    }
});
```

傳入 `chunk` 方法的第一個引數為每個「區塊 (Chunk)」想收到的紀錄數量。第二個引數傳入的閉包，則會對從資料庫中取出的每個區塊都叫用一次。每次要將紀錄區塊傳給閉包時，都會執行一次資料庫查詢。

若要根據某個欄位來篩選 `chunk` 方法的結果，且該欄位又會在疊代結果時被更新，則應改用 `chunkById` 方法。在這種情況下使用 `chunk` 方法可能會導致非預期且不一致的結果。在內部，`chunkById` 方法會一直取出 `id` 欄位大於前一個區塊中最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->chunkById(200, function (Collection $flights) {
        $flights->each->update(['departed' => false]);
    }, column: 'id');
```

由於 `chunkById` 與 `lazyById` 方法會將自己的 "where" 條件式加到要執行的查詢中，因此，通常應將自己的條件式[以邏輯分組](/docs/{{version}}/queries#logical-grouping)到閉包內：

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
### 使用 Lazy Collection 來分塊

`lazy` 方法的運作方式與 [`chunk` 方法](#chunking-results)類似，都是在底層將查詢分成多個區塊執行。不過，`lazy` 方法並不會將每個區塊都直接傳入回呼，而是會回傳一個攤平的 (Flattened) Eloquent Model 的 [LazyCollection](/docs/{{version}}/collections#lazy-collections)，讓你能像操作單一流 (Stream) 一樣操作結果：

```php
use App\Models\Flight;

foreach (Flight::lazy() as $flight) {
    // ...
}
```

若要根據某個欄位來篩選 `lazy` 方法的結果，且該欄位又會在疊代結果時被更新，則應改用 `lazyById` 方法。在內部，`lazyById` 方法會一直取出 `id` 欄位大於前一個區塊中最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->lazyById(200, column: 'id')
    ->each->update(['departed' => false]);
```

可以使用 `lazyByIdDesc` 方法來根據 `id` 的降冪順序來篩選結果。


<a name="cursors"></a>
### Cursor

與 `lazy` 方法類似，`cursor` 方法可用來在疊代數萬筆 Eloquent Model 紀錄時大幅降低應用程式的記憶體消耗。

`cursor` 方法只會執行一次資料庫查詢；不過，單一的 Eloquent Model 只有在實際被疊代時才會被 Hydrate (填充)。因此，在疊代 Cursor 時，在任何時間點都只會有一個 Eloquent Model 保存在記憶體中。

> [!WARNING]
> 由於 `cursor` 方法在任何時間點都只會在記憶體中保留一個 Eloquent Model，因此無法預先載入 (Eager Load) 關聯。若需要預先載入關聯，請考慮改用 [`lazy` 方法](#chunking-using-lazy-collections)。

在內部，`cursor` 方法使用 PHP 的[產生器 (Generator)](https://www.php.net/manual/en/language.generators.overview.php) 來實作此功能：

```php
use App\Models\Flight;

foreach (Flight::where('destination', 'Zurich')->cursor() as $flight) {
    // ...
}
```

`cursor` 會回傳一個 `Illuminate\Support\LazyCollection` 實體。[Lazy Collection](/docs/{{version}}/collections#lazy-collections) 可讓讀者使用許多一般 Laravel Collection 上可用的 Collection 方法，同時一次只在記憶體中載入一個 Model：

```php
use App\Models\User;

$users = User::cursor()->filter(function (User $user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```

雖然 `cursor` 方法比一般的查詢使用更少的記憶體 (因為在任何時間點都只會在記憶體中保留一個 Eloquent Model)，但最終還是有可能會耗盡記憶體。這是[因為 PHP 的 PDO 驅動程式會在內部將所有原始的查詢結果快取在其緩衝區中](https://www.php.net/manual/en/mysqlinfo.concepts.buffering.php)。若要處理大量的 Eloquent 紀錄，請考慮改用 [`lazy` 方法](#chunking-using-lazy-collections)。


<a name="advanced-subqueries"></a>
### 進階子查詢


<a name="subquery-selects"></a>
#### 子查詢 Select

Eloquent 也提供了進階的子查詢支援，讓我們能在單一查詢中從關聯的資料表裡拉取資訊。舉例來說，假設我們有一個航班 `destinations` (目的地) 資料表與一個 `flights` (航班) 資料表。`flights` 資料表包含了一個 `arrived_at` 欄位，用來指出航班抵達目的地的時間。

使用查詢產生器 `select` 與 `addSelect` 方法上可用的子查詢功能，我們就可以在單一查詢中選取所有的 `destinations` 與最近抵達該目的地的航班名稱：

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

此外，查詢產生器的 `orderBy` 函式也支援子查詢。繼續使用剛才的航班範例，我們可以使用此功能來將所有目的地根據最後一班航班抵達該目的地的時間進行排序。同樣地，這也可以在執行單一資料庫查詢時完成：

```php
return Destination::orderByDesc(
    Flight::select('arrived_at')
        ->whereColumn('destination_id', 'destinations.id')
        ->orderByDesc('arrived_at')
        ->limit(1)
)->get();
```

<a name="retrieving-single-models"></a>
## 取出單一 Model / 彙總

除了可取出符合指定查詢的所有記錄外，我們也可使用 `find`, `first`, `firstWhere` 等方法來取出單筆記錄。這些方法不會回傳 Model 的 Collection，而是回傳單一 Model 的實體：

```php
use App\Models\Flight;

// Retrieve a model by its primary key...
$flight = Flight::find(1);

// Retrieve the first model matching the query constraints...
$flight = Flight::where('active', 1)->first();

// Alternative to retrieving the first model matching the query constraints...
$flight = Flight::firstWhere('active', 1);
```

有時候，找不到結果時我們可能會想執行其他動作。`findOr` 與 `firstOr` 方法會回傳單一 Model 實體，或是在找不到結果時執行給定的閉包。閉包的回傳值會被視為該方法的回傳值：

```php
$flight = Flight::findOr(1, function () {
    // ...
});

$flight = Flight::where('legs', '>', 3)->firstOr(function () {
    // ...
});
```


<a name="not-found-exceptions"></a>
#### 「找不到」例外

有時候，找不到 Model 時我們會想擲回例外。這個功能在 Route 或 Controller 中特別有用。`findOrFail` 與 `firstOrFail` 方法會取回查詢的第一筆結果；不過，若找不到結果，則會擲回 `Illuminate\Database\Eloquent\ModelNotFoundException`。

```php
$flight = Flight::findOrFail(1);

$flight = Flight::where('legs', '>', 3)->firstOrFail();
```

若沒有捕捉到 `ModelNotFoundException`，則會自動回傳 404 HTTP 回應給用戶端：

```php
use App\Models\Flight;

Route::get('/api/flights/{id}', function (string $id) {
    return Flight::findOrFail($id);
});
```


<a name="retrieving-or-creating-models"></a>
### 取出或建立 Model

`firstOrCreate` 方法會嘗試使用給定的欄位／值組合來尋找一筆資料庫記錄。若在資料庫中找不到該 Model，則會插入一筆記錄，其中包含由第一個陣列引數與可選的第二個陣列引數合併而成的屬性。

`firstOrNew` 方法與 `firstOrCreate` 類似，會嘗試在資料庫中尋找符合給定屬性的記錄。不過，若找不到 Model，則會回傳一個新的 Model 實體。請注意，`firstOrNew` 所回傳的 Model 還尚未被保存到資料庫。我們需要手動呼叫 `save` 方法來將其保存：

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
### 取出彙總值

與 Eloquent Model 互動時，我們也可以使用 `count`, `sum`, `max` 與其他由 Laravel [查詢產生器](/docs/{{version}}/queries)所提供的[彙總方法](/docs/{{version}}/queries#aggregates)。正如我們所想的，這些方法會回傳純量值，而不是 Eloquent Model 實體：

```php
$count = Flight::where('active', 1)->count();

$max = Flight::where('active', 1)->max('price');
```

<a name="inserting-and-updating-models"></a>
## 新增與更新 Model


<a name="inserts"></a>
### 新增

當然，在使用 Eloquent 時，我們不只需要從資料庫中取出 Model。我們也需要新增新的資料。幸運的是，Eloquent 讓這件事變得很簡單。若要新增一筆新的資料到資料庫中，應先具現化一個新的 Model 實體，並為該 Model 設定屬性。接著，在該 Model 實體上呼叫 `save` 方法：

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

在這個範例中，我們將傳入的 HTTP Request 中的 `name` 欄位指派給 `App\Models\Flight` Model 實體的 `name` 屬性。當我們呼叫 `save` 方法時，就會將一筆資料插入資料庫。呼叫 `save` 方法時，Model 的 `created_at` 與 `updated_at` 時間戳會自動被設定，所以不需要手動設定。

或者，也可以使用 `create` 方法，只用一個 PHP 陳述式就「儲存」一個新的 Model。`create` 方法會回傳已插入的 Model 實體：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

不過，在使用 `create` 方法前，需要在 Model 類別上指定 `fillable` 或 `guarded` 屬性。需要這些屬性，是因為所有的 Eloquent Model 預設都會防範 Mass Assignment 漏洞。想了解更多有關 Mass Assignment 的資訊，請參考 [Mass Assignment 的說明文件](#mass-assignment)。


<a name="updates"></a>
### 更新

`save` 方法也可用來更新已存在於資料庫中的 Model。若要更新 Model，應先取出 Model，並設定想更新的屬性。接著，應呼叫該 Model 的 `save` 方法。`updated_at` 時間戳一樣會自動更新，所以不需要手動設定其值：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->name = 'Paris to London';

$flight->save();
```

有時候，我們可能會需要在沒有符合的 Model 時，更新現有的 Model 或建立新的 Model。與 `firstOrCreate` 方法類似，`updateOrCreate` 方法也會將 Model 保存到資料庫，所以不需要手動呼叫 `save` 方法。

在下列範例中，若有一筆 `departure` 地點為 `Oakland` 且 `destination` 地點為 `San Diego` 的航班，則會更新其 `price` 與 `discounted` 欄位。若沒有這樣的航班，則會建立一個新的航班，其屬性為第一個引數陣列與第二個引數陣列合併後的結果：

```php
$flight = Flight::updateOrCreate(
    ['departure' => 'Oakland', 'destination' => 'San Diego'],
    ['price' => 99, 'discounted' => 1]
);
```


<a name="mass-updates"></a>
#### Mass Update

也可以對符合查詢條件的 Model 進行更新。在這個範例中，所有 `active` 且 `destination` 為 `San Diego` 的航班都會被標示為延誤：

```php
Flight::where('active', 1)
    ->where('destination', 'San Diego')
    ->update(['delayed' => 1]);
```

`update` 方法應傳入一個由欄位與值組成的鍵值對陣列，用來代表應更新的欄位。`update` 方法會回傳受影響的資料列數。

> [!WARNING]
> 當透過 Eloquent 執行 Mass Update 時，更新的 Model 將不會觸發 `saving`、`saved`、`updating`、與 `updated` 等 Model 事件。這是因為在執行 Mass Update 時從未實際取出過這些 Model。


<a name="examining-attribute-changes"></a>
#### 檢查屬性變更

Eloquent 提供了 `isDirty`、`isClean`、與 `wasChanged` 方法來檢查 Model 的內部狀態，並判斷其屬性與最初取出 Model 時相比有哪些變更。

`isDirty` 方法可判斷 Model 取出後是否有任何屬性被變更。可以傳入一個特定的屬性名稱或一個屬性陣列給 `isDirty` 方法，來判斷是否有任何給定的屬性是「Dirty (變更過的)」。`isClean` 方法可判斷某個屬性在 Model 取出後是否保持未變更。這個方法也接受一個可選的屬性引數：

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

`wasChanged` 方法可判斷在目前這個 Request 生命週期中，上次儲存 Model 時是否有任何屬性被變更。若有需要，可以傳入一個屬性名稱，來查看特定的屬性是否有被變更：

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

`getOriginal` 方法會回傳一個包含 Model 原始屬性的陣列，不論 Model 取出後做了哪些變更。若有需要，可以傳入一個特定的屬性名稱，來取得特定屬性的原始值：

```php
$user = User::find(1);

$user->name; // John
$user->email; // john@example.com

$user->name = 'Jack';
$user->name; // Jack

$user->getOriginal('name'); // John
$user->getOriginal(); // Array of original attributes...
```

`getChanges` 方法會回傳一個陣列，其中包含上次儲存 Model 時變更的屬性；而 `getPrevious` 方法會回傳一個陣列，其中包含上次儲存 Model 前的原始屬性值：

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
### Mass Assignment

可以使用 `create` 方法，只用一個 PHP 陳述式就「儲存」一個新的 Model。這個方法會回傳已插入的 Model 實體：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

不過，在使用 `create` 方法前，需要在 Model 類別上指定 `fillable` 或 `guarded` 屬性。需要這些屬性，是因為所有的 Eloquent Model 預設都會防範 Mass Assignment 漏洞。

當使用者傳入未預期的 HTTP Request 欄位，且該欄位更改了資料庫中未預期更改的欄位時，就會發生 Mass Assignment 漏洞。舉例來說，惡意使用者可能會透過 HTTP Request 傳送一個 `is_admin` 參數，然後這個參數被傳給了 Model 的 `create` 方法，進而讓該使用者將自己提升為管理員。

所以，首先，我們應定義哪些 Model 屬性是可進行 Mass Assignment 的。可以使用 Model 上的 `$fillable` 屬性來做到這點。舉例來說，讓我們的 `Flight` Model 的 `name` 屬性可進行 Mass Assignment：

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

指定好哪些屬性可進行 Mass Assignment 後，就可以使用 `create` 方法來將新的資料插入資料庫。`create` 方法會回傳新建的 Model 實體：

```php
$flight = Flight::create(['name' => 'London to Paris']);
```

若已有一個 Model 實體，可以使用 `fill` 方法來為其填入一個屬性陣列：

```php
$flight->fill(['name' => 'Amsterdam to Frankfurt']);
```


<a name="mass-assignment-json-columns"></a>
#### Mass Assignment 與 JSON 欄位

在指派 JSON 欄位時，每個欄位中可進行 Mass Assignment 的鍵都必須在 Model 的 `$fillable` 陣列中指定。為了安全起見，在使用 `guarded` 屬性時，Laravel 不支援更新巢狀的 JSON 屬性：

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
#### 允許 Mass Assignment

若想讓所有屬性都可進行 Mass Assignment，可以將 Model 的 `$guarded` 屬性定義為一個空陣列。若選擇不防範 Model，則應特別小心，務必手動處理傳給 Eloquent `fill`、`create`、與 `update` 方法的陣列：

```php
/**
 * The attributes that aren't mass assignable.
 *
 * @var array<string>|bool
 */
protected $guarded = [];
```


<a name="mass-assignment-exceptions"></a>
#### Mass Assignment 例外

預設情況下，在執行 Mass Assignment 操作時，未包含在 `$fillable` 陣列中的屬性會被默默地捨棄。在生產環境中，這是預期的行為；然而，在本地開發期間，這可能會導致對於為什麼 Model 的變更沒有生效感到困惑。

若有需要，可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，來讓 Laravel 在嘗試填入不可填入的屬性時拋出例外。通常，這個方法應在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

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

Eloquent 的 `upsert` 方法可用於在單一、不可分割的操作中更新或建立資料。此方法的第一個引數包含要插入或更新的值，而第二個引數則列出在關聯資料表中唯一識別記錄的欄位。此方法的第三個也是最後一個引數是一個陣列，其中包含如果資料庫中已存在相符記錄時應更新的欄位。如果 Model 上啟用了時間戳，`upsert` 方法會自動設定 `created_at` 和 `updated_at` 時間戳：

```php
Flight::upsert([
    ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
    ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
], uniqueBy: ['departure', 'destination'], update: ['price']);
```

> [!WARNING]
> 除了 SQL Server 以外的所有資料庫都要求 `upsert` 方法第二個引數中的欄位具有「主鍵」或「唯一」索引。此外，MariaDB 和 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個引數，並始終使用資料表的「主鍵」和「唯一」索引來偵測現有記錄。

<a name="deleting-models"></a>
## 刪除 Model

若要刪除 Model，可在 Model 實體上呼叫 `delete` 方法：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->delete();
```


<a name="deleting-an-existing-model-by-its-primary-key"></a>
#### 透過主鍵刪除現有 Model

在上面的範例中，我們在呼叫 `delete` 方法前先從資料庫中取出 Model。不過，若知道 Model 的主鍵，則不須明確取出 Model 即可透過呼叫 `destroy` 方法來刪除 Model。除了接受單一主鍵外，`destroy` 方法還可接受多個主鍵、一組主鍵陣列、或一組主鍵 [Collection](/docs/{{version}}/collections)：

```php
Flight::destroy(1);

Flight::destroy(1, 2, 3);

Flight::destroy([1, 2, 3]);

Flight::destroy(collect([1, 2, 3]));
```

若有使用[軟刪除 Model](#soft-deleting)，可透過 `forceDestroy` 方法來永久刪除 Model：

```php
Flight::forceDestroy(1);
```

> [!WARNING]
> `destroy` 方法會個別載入每個 Model 並呼叫其 `delete` 方法，這樣才能為每個 Model 都正常派送 `deleting` 與 `deleted` 事件。


<a name="deleting-models-using-queries"></a>
#### 使用查詢來刪除 Model

當然，我們也可以建立 Eloquent 查詢來刪除所有符合查詢條件的 Model。在這個範例中，我們會刪除所有標記為非作用中的航班。與 Mass Update (大量更新) 類似，Mass Delete (大量刪除) 也不會為被刪除的 Model 派送任何 Model 事件：

```php
$deleted = Flight::where('active', 0)->delete();
```

若要刪除資料表中的所有 Model，應執行一個沒有任何條件的查詢：

```php
$deleted = Flight::query()->delete();
```

> [!WARNING]
> 透過 Eloquent 執行 Mass Delete 陳述式時，不會為被刪除的 Model 派送 `deleting` 與 `deleted` 等 Model 事件。這是因為在執行刪除陳述式時，並不會實際去取出這些 Model。


<a name="soft-deleting"></a>
### 軟刪除

除了實際從資料庫中移除紀錄外，Eloquent 也能「軟刪除」Model。當 Model 被軟刪除時，並不會實際從資料庫中被移除。反之，Model 上會設定一個 `deleted_at` 屬性，用來標記該 Model 被「刪除」的日期與時間。若要為 Model 啟用軟刪除，請將 `Illuminate\Database\Eloquent\SoftDeletes` Trait 加到該 Model 上：

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
> `SoftDeletes` Trait 會自動為你將 `deleted_at` 屬性轉型為 `DateTime` / `Carbon` 實體。

我們也應該將 `deleted_at` 欄位加到資料表中。Laravel 的 [Schema Builder](/docs/{{version}}/migrations) 中包含了一個可建立此欄位的輔助方法：

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

現在，當在 Model 上呼叫 `delete` 方法時，`deleted_at` 欄位會被設為目前的日期與時間。不過，該 Model 的資料庫紀錄仍會留在資料表內。當查詢使用軟刪除的 Model 時，所有被軟刪除了的 Model 都會自動從查詢結果中排除。

若要判斷某個 Model 實體是否已被軟刪除，可使用 `trashed` 方法：

```php
if ($flight->trashed()) {
    // ...
}
```


<a name="restoring-soft-deleted-models"></a>
#### 回復軟刪除的 Model

有時候，我們可能會想「取消刪除」某個被軟刪除的 Model。若要回復軟刪除的 Model，可在 Model 實體上呼叫 `restore` 方法。`restore` 方法會將 Model 的 `deleted_at` 欄位設為 `null`：

```php
$flight->restore();
```

我們也可以在查詢中使用 `restore` 方法來回復多個 Model。同樣地，與其他「大量」操作一樣，此操作不會為被回復的 Model 派送任何 Model 事件：

```php
Flight::withTrashed()
    ->where('airline_id', 1)
    ->restore();
```

`restore` 方法也可用在建立[關聯](/docs/{{version}}/eloquent-relationships)查詢上：

```php
$flight->history()->restore();
```


<a name="permanently-deleting-models"></a>
#### 永久刪除 Model

有時候，我們可能會需要真的從資料庫中移除 Model。我們可以使用 `forceDelete` 方法來從資料庫資料表中永久移除軟刪除的 Model：

```php
$flight->forceDelete();
```

`forceDelete` 方法也可用在建立 Eloquent 關聯查詢上：

```php
$flight->history()->forceDelete();
```


<a name="querying-soft-deleted-models"></a>
### 查詢軟刪除的 Model


<a name="including-soft-deleted-models"></a>
#### 包含軟刪除的 Model

如上所述，軟刪除的 Model 會自動從查詢結果中排除。不過，我們可以在查詢上呼叫 `withTrashed` 方法來強制讓軟刪除的 Model 也包含在查詢結果中：

```php
use App\Models\Flight;

$flights = Flight::withTrashed()
    ->where('account_id', 1)
    ->get();
```

`withTrashed` 方法也可用在建立[關聯](/docs/{{version}}/eloquent-relationships)查詢上：

```php
$flight->history()->withTrashed()->get();
```


<a name="retrieving-only-soft-deleted-models"></a>
#### 只取出軟刪除的 Model

`onlyTrashed` 方法則會 **只** 取出軟刪除的 Model：

```php
$flights = Flight::onlyTrashed()
    ->where('airline_id', 1)
    ->get();
```

<a name="pruning-models"></a>
## 修剪 Model

有時候，我們可能會想定期刪除不再需要的 Model。為此，我們可以在想定期修剪的 Model 上加上 `Illuminate\Database\Eloquent\Prunable` 或 `Illuminate\Database\Eloquent\MassPrunable` Trait。在 Model 上加上其中一個 Trait 後，請實作一個 `prunable` 方法。該方法應回傳一個 Eloquent 查詢產生器，用以解析出不再需要的 Model：

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

將 Model 標記為 `Prunable` 時，也可以在 Model 上定義一個 `pruning` 方法。這個方法會在 Model 被刪除前呼叫。這個方法可用於在 Model 從資料庫中永久移除前，刪除任何與該 Model 關聯的額外資源 (如儲存的檔案)：

```php
/**
 * Prepare the model for pruning.
 */
protected function pruning(): void
{
    // ...
}
```

設定好可修剪的 Model 後，應在應用程式的 `routes/console.php` 檔中排程 `model:prune` Artisan 指令。你可以自由選擇執行此指令的適當間隔：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('model:prune')->daily();
```

在背景中，`model:prune` 指令會自動偵測應用程式 `app/Models` 目錄內可「修剪 (Prunable)」的 Model。若你的 Model 放在別的位置，則可使用 `--model` 選項來指定 Model 的類別名稱：

```php
Schedule::command('model:prune', [
    '--model' => [Address::class, Flight::class],
])->daily();
```

若想在修剪所有其他偵測到的 Model 時排除某些特定的 Model，可使用 `--except` 選項：

```php
Schedule::command('model:prune', [
    '--except' => [Address::class, Flight::class],
])->daily();
```

我們可以執行帶有 `--pretend` 選項的 `model:prune` 指令來測試 `prunable` 查詢。模擬執行時，`model:prune` 指令只會回報若實際執行指令時會修剪掉多少筆資料：

```shell
php artisan model:prune --pretend
```

> [!WARNING]
> 若軟刪除的 Model 符合修剪查詢，則會被永久刪除 (`forceDelete`)。

<a name="mass-pruning"></a>
#### 大量修剪

當 Model 被標記 `Illuminate\Database\Eloquent\MassPrunable` Trait 時，這些 Model 會使用大量刪除查詢從資料庫中刪除。因此，並不會叫用 `pruning` 方法，也不會分派 `deleting` 與 `deleted` 的 Model 事件。這是因為在刪除前並不會實際上去取出這些 Model，因此能讓修剪過程更有效率：

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

我們可以使用 `replicate` 方法來建立一個現有 Model Instance 的未儲存複本。當有多個 Model Instance 共用許多相同屬性時，這個方法特別好用：

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

若要讓一或多個屬性不要被複製到新的 Model，可以傳遞一個陣列給 `replicate` 方法：

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
## 查詢作用域


<a name="global-scopes"></a>
### 全域作用域

全域作用域 (Global Scope) 能讓我們為給定 Model 的所有查詢加上限制。Laravel 內建的[軟刪除](#soft-deleting)功能就是利用全域作用域來只從資料庫中取出「未被刪除」的 Model。編寫自己的全域作用域是個很方便、簡單的方法，能確保某個 Model 的每個查詢都有加上特定的限制。


<a name="generating-scopes"></a>
#### 產生作用域

若要產生新的全域作用域，可叫用 `make:scope` Artisan 指令。產生的作用域會被放在專案的 `app/Models/Scopes` 目錄下：

```shell
php artisan make:scope AncientScope
```


<a name="writing-global-scopes"></a>
#### 編寫全域作用域

編寫全域作用域很簡單。首先，使用 `make:scope` 指令來產生一個實作 `Illuminate\Database\Eloquent\Scope` 介面的類別。`Scope` 介面會要求我們實作一個 `apply` 方法。`apply` 方法可視情況在查詢上加上 `where` 限制或其他類型的子句：

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
> 若全域作用域有在查詢的 Select 子句上新增欄位，請改用 `addSelect` 方法而不是 `select`。這樣才能避免不小心蓋掉查詢中已存在的 Select 子句。


<a name="applying-global-scopes"></a>
#### 套用全域作用域

若要將全域作用域指派給 Model，只要在 Model 上放上 `ScopedBy` 屬性 (Attribute) 即可：

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

或者，我們也可以手動註冊全域作用域。只要覆寫 Model 的 `booted` 方法並在其中叫用 `addGlobalScope` 方法即可。`addGlobalScope` 方法只接受一個引數，為作用域的實體：

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

在上述範例中，將該作用域加到 `App\Models\User` Model 後，`User::all()` 方法的呼叫就會執行下列 SQL 查詢：

```sql
select * from `users` where `created_at` < 0021-02-18 00:00:00
```


<a name="anonymous-global-scopes"></a>
#### 匿名全域作用域

Eloquent 也允許我們使用閉包 (Closure) 來定義全域作用域。對於不需要獨立出一個類別的簡單作用域來說特別有用。使用閉包來定義全域作用域時，應在 `addGlobalScope` 方法的第一個引數中提供一個自訂的作用域名稱：

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
#### 移除全域作用域

若想在某個查詢中移除某個全域作用域，可使用 `withoutGlobalScope` 方法。此方法只接受一個引數，為全域作用域的類別名稱：

```php
User::withoutGlobalScope(AncientScope::class)->get();
```

或者，若作用域是使用閉包定義的，則應傳入我們指派給該全域作用域的字串名稱：

```php
User::withoutGlobalScope('ancient')->get();
```

若想移除查詢中的多個、甚至所有的全域作用域，可使用 `withoutGlobalScopes` 與 `withoutGlobalScopesExcept` 方法：

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
### 區域作用域

區域作用域 (Local Scope) 可讓我們定義一組能在專案內輕鬆重複使用的查詢限制。舉例來說，我們可能需要頻繁地取出所有被視為「熱門」的使用者。若要定義作用域，請將 `Scope` 屬性 (Attribute) 加到 Eloquent 方法上。

作用域應一律回傳同一個查詢產生器實體，或回傳 `void`：

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
#### 使用區域作用域

定義好作用域後，就可以在查詢 Model 時呼叫該作用域方法。我們甚至可以將多個作用域的呼叫串連起來：

```php
use App\Models\User;

$users = User::popular()->active()->orderBy('created_at')->get();
```

若要透過 `or` 查詢運算子來結合多個 Eloquent Model 作用域，可能需要使用閉包來達成正確的[邏輯分組](/docs/{{version}}/queries#logical-grouping)：

```php
$users = User::popular()->orWhere(function (Builder $query) {
    $query->active();
})->get();
```

不過，由於這樣做可能有點麻煩，Laravel 提供了一個「高階 (Higher Order)」的 `orWhere` 方法，能讓我們流暢地將作用域串連在一起，而不需要使用閉包：

```php
$users = User::popular()->orWhere->active()->get();
```


<a name="dynamic-scopes"></a>
#### 動態作用域

有時候我們可能會想定義一個可接受參數的作用域。若要這麼做，只要在作用域方法的簽章 (Signature) 中加上額外的參數即可。作用域參數應在 `$query` 參數後方定義：

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

在作用域方法的簽章中加上預期的引數後，就可以在呼叫作用域時傳入引數：

```php
$users = User::ofType('admin')->get();
```


<a name="pending-attributes"></a>
### 待定屬性

若想使用作用域來建立具有與作用域限制相同屬性的 Model，可以在建立作用域查詢時使用 `withAttributes` 方法：

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

`withAttributes` 方法會使用給定的屬性在查詢上加上 `where` 條件，並且也會將給定的屬性加到透過該作用域建立的任何 Model 上：

```php
$draft = Post::draft()->create(['title' => 'In Progress']);

$draft->hidden; // true
```

若要讓 `withAttributes` 方法不要在查詢上加上 `where` 條件，可將 `asConditions` 引數設為 `false`：

```php
$query->withAttributes([
    'hidden' => true,
], asConditions: false);
```

<a name="comparing-models"></a>
## 比較 Model

有時候，我們可能需要判斷兩個 Model 是否「相同」。可使用 `is` 與 `isNot` 方法來快速驗證兩個 Model 是否有相同的主鍵、資料表、與資料庫連線：

```php
if ($post->is($anotherPost)) {
    // ...
}

if ($post->isNot($anotherPost)) {
    // ...
}
```

在使用 `belongsTo`、`hasOne`、`morphTo`、`morphOne` [關聯](/docs/{{version}}/eloquent-relationships)時，也可以使用 `is` 與 `isNot` 方法。當我們想在不發出查詢來取得關聯 Model 的情況下比較關聯 Model 時，這個方法特別好用：

```php
if ($post->author()->is($user)) {
    // ...
}
```

<a name="events"></a>
## 事件

> [!NOTE]
> 想直接將 Eloquent 事件廣播到前端應用程式嗎？請參考 Laravel 的 [Model 事件廣播](/docs/{{version}}/broadcasting#model-broadcasting) 功能。

Eloquent Model 會分派 (dispatch) 數個事件，可讓開發者掛載 (hook) 到 Model 生命週期的下列幾個時間點：`retrieved`、`creating`、`created`、`updating`、`updated`、`saving`、`saved`、`deleting`、`deleted`、`trashed`、`forceDeleting`、`forceDeleted`、`restoring`、`restored`、`replicating`。

當從資料庫中取出已存在的 Model 時，會分派 `retrieved` 事件。當第一次儲存新的 Model 時，會分派 `creating` 與 `created` 事件。當修改已存在的 Model 並呼叫 `save` 方法時，則會分派 `updating` / `updated` 事件。當建立或更新 Model 時，則會分派 `saving` / `saved` 事件 —— 即便 Model 的屬性沒有變更。以 `-ing` 結尾的事件會在 Model 的任何變更被保存 (Persist) 前分派，而以 `-ed` 結尾的事件則會在 Model 變更被保存後才分派。

若要開始監聽 Model 事件，請在 Eloquent Model 上定義 `$dispatchesEvents` 屬性。此屬性可將 Eloquent Model 生命週期的各個時間點對應到你自己的[事件類別](/docs/{{version}}/events)。每個 Model 事件類別都應預期能透過其建構函式收到一個受影響的 Model 的 Instance：

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

在定義並對應好 Eloquent 事件後，就可以使用[事件監聽器](/docs/{{version}}/events#defining-listeners)來處理這些事件。

> [!WARNING]
> 當透過 Eloquent 發出 Mass Update 或 Mass Delete 查詢時，`saved`、`updated`、`deleting`、`deleted` 等 Model 事件將**不會**為受影響的 Model 分派。這是因為在執行 Mass Update 或 Mass Delete 時，並不會真的將這些 Model 取出。

<a name="events-using-closures"></a>
### 使用閉包

除了使用自訂事件類別外，我們也可以註冊閉包 (Closure)，並讓這些閉包在分派各種 Model 事件時執行。一般來說，我們應在 Model 的 `booted` 方法中註冊這些閉包：

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

若有需要，也可以在註冊 Model 事件時使用[可佇列的匿名事件監聽器](/docs/{{version}}/events#queuable-anonymous-event-listeners)。這麼做會讓 Laravel 使用應用程式的[佇列](/docs/{{version}}/queues)來在背景執行 Model 事件監聽器：

```php
use function Illuminate\Events\queueable;

static::created(queueable(function (User $user) {
    // ...
}));
```

<a name="observers"></a>
### Observer

<a name="defining-observers"></a>
#### 定義 Observer

若要監聽某個特定 Model 的多個事件，可以使用 Observer 來將所有的監聽器群組到單一類別中。Observer 類別的方法名稱會反映我們想監聽的 Eloquent 事件。這些方法的唯一引數都是受影響的 Model。`make:observer` Artisan 指令是建立新 Observer 類別最簡單的方法：

```shell
php artisan make:observer UserObserver --model=User
```

此指令會將新的 Observer 放置在 `app/Observers` 目錄下。若該目錄不存在，Artisan 會為你建立該目錄。剛建立好的 Observer 會像這樣：

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

若要註冊 Observer，可在對應的 Model 上放置 `ObservedBy` 屬性：

```php
use App\Observers\UserObserver;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy([UserObserver::class])]
class User extends Authenticatable
{
    //
}
```

或者，我們也可以在想觀察的 Model 上叫用 `observe` 方法來手動註冊 Observer。我們可以在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中註冊 Observer：

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
> Observer 還可以監聽其他的事件，如 `saving` 與 `retrieved`。這些事件都記載在[事件](#events)說明文件中。

<a name="observers-and-database-transactions"></a>
#### Observer 與資料庫交易

當 Model 在資料庫交易 (Transaction) 中建立時，有時我們可能會想讓 Observer 只在資料庫交易被 Commit 後才執行其事件處理函式。若要這麼做，可在 Observer 上實作 `ShouldHandleEventsAfterCommit` 介面。若目前沒有在資料庫交易中，則事件處理函式會立即執行：

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

有時候，我們可能會需要暫時「靜音」某個 Model 觸發的所有事件。若要這麼做，可使用 `withoutEvents` 方法。`withoutEvents` 方法的唯一引數是一個閉包。所有在閉包中執行的程式碼都不會分派 Model 事件，而該閉包的回傳值會被 `withoutEvents` 方法回傳：

```php
use App\Models\User;

$user = User::withoutEvents(function () {
    User::findOrFail(1)->delete();

    return User::find(2);
});
```

<a name="saving-a-single-model-without-events"></a>
#### 不觸發事件來儲存單一 Model

有時候，我們可能會想「儲存」某個特定的 Model 但不分派任何事件。若要這麼做，可使用 `saveQuietly` 方法：

```php
$user = User::findOrFail(1);

$user->name = 'Victoria Faith';

$user->saveQuietly();
```

我們也可以「更新」、「刪除」、「軟刪除」、「還原」、並「複製」某個特定的 Model 而不分派任何事件：

```php
$user->deleteQuietly();
$user->forceDeleteQuietly();
$user->restoreQuietly();
```