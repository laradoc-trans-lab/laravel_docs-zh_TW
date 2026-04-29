# Eloquent：入門

- [簡介](#introduction)
- [產生 Model 類別](#generating-model-classes)
- [Eloquent Model 慣例](#eloquent-model-conventions)
    - [資料表名稱](#table-names)
    - [主鍵](#primary-keys)
    - [UUID 與 ULID 鍵](#uuid-and-ulid-keys)
    - [時間戳記](#timestamps)
    - [資料庫連線](#database-connections)
    - [屬性預設值](#default-attribute-values)
    - [設定 Eloquent 嚴謹度](#configuring-eloquent-strictness)
- [取得 Model](#retrieving-models)
    - [集合](#collections)
    - [分塊處理結果](#chunking-results)
    - [使用 Lazy Collections 分塊](#chunking-using-lazy-collections)
    - [指標 (Cursors)](#cursors)
    - [進階子查詢](#advanced-subqueries)
- [取得單一 Model / 聚合結果](#retrieving-single-models)
    - [取得或建立 Model](#retrieving-or-creating-models)
    - [取得聚合結果](#retrieving-aggregates)
- [新增與更新 Model](#inserting-and-updating-models)
    - [新增](#inserts)
    - [更新](#updates)
    - [批量賦值 (Mass Assignment)](#mass-assignment)
    - [更新或新增 (Upserts)](#upserts)
- [刪除 Model](#deleting-models)
    - [軟刪除](#soft-deleting)
    - [查詢軟刪除的 Model](#querying-soft-deleted-models)
- [修剪 Model](#pruning-models)
- [複製 Model](#replicating-models)
- [查詢範圍 (Query Scopes)](#query-scopes)
    - [全域範圍](#global-scopes)
    - [區域範圍](#local-scopes)
    - [待定屬性](#pending-attributes)
- [比較 Model](#comparing-models)
- [事件](#events)
    - [使用閉包](#events-using-closures)
    - [觀察者](#observers)
    - [靜音事件](#muting-events)

<a name="introduction"></a>
## 簡介

Laravel 包含了 Eloquent，這是一個物件關聯映射 (ORM)，讓與資料庫的互動變得愉快。使用 Eloquent 時，每個資料庫資料表都有一個對應的「Model」，用於與該資料表互動。除了從資料庫資料表中取得記錄外，Eloquent Model 還允許你在資料表中新增、更新與刪除記錄。

> [!NOTE]
> 在開始之前，請務必在應用程式的 `config/database.php` 設定檔中設定資料庫連線。有關設定資料庫的更多資訊，請參閱[資料庫設定文件](/docs/{{version}}/database#configuration)。


<a name="generating-model-classes"></a>
## 產生 Model 類別

首先，讓我們來建立一個 Eloquent Model。Model 通常位於 `app\Models` 目錄下，並繼承 `Illuminate\Database\Eloquent\Model` 類別。你可以使用 `make:model` [Artisan 指令](/docs/{{version}}/artisan) 來產生一個新的 Model：

```shell
php artisan make:model Flight
```

如果你希望在產生 Model 的同時也產生[資料庫遷移](/docs/{{version}}/migrations)，可以使用 `--migration` 或 `-m` 選項：

```shell
php artisan make:model Flight --migration
```

在產生 Model 時，你也可以同時產生其他各種類型的類別，例如 Factory、Seeder、Policy、Controller 以及表單請求(Form request)。此外，這些選項可以組合在一起，一次建立多個類別：

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

有時僅透過瀏覽程式碼，很難確定 Model 的所有可用屬性與關聯。相反地，可以嘗試使用 `model:show` Artisan 指令，它提供了該 Model 所有屬性與關聯的便利概覽：

```shell
php artisan model:show Flight
```

<a name="eloquent-model-conventions"></a>
## Eloquent Model 慣例

使用 `make:model` 指令產生的 Model 會被放置在 `app/Models` 目錄中。讓我們來看一個基本的 Model 類別，並討論一些 Eloquent 的核心慣例：

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

看了上面的範例後，您可能已經注意到我們並沒有告訴 Eloquent 哪個資料表對應到我們的 `Flight` Model。按照慣例，除非明確指定其他名稱，否則類別名稱的「蛇形命名 (snake case)」複數形式將被用作資料表名稱。因此，在這種情況下，Eloquent 會假設 `Flight` Model 將紀錄儲存在 `flights` 資料表中，而 `AirTrafficController` Model 則會將紀錄儲存在 `air_traffic_controllers` 資料表中。

如果您 Model 對應的資料表不符合此慣例，您可以使用 `Table` 屬性手動指定 Model 的資料表名稱：

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

Eloquent 也會假設每個 Model 對應的資料表都有一個名為 `id` 的主鍵欄位。如有必要，您可以使用 `Table` 屬性上的 `key` 引數來指定一個不同的欄位作為 Model 的主鍵：

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

此外，Eloquent 假設主鍵是一個自動遞增的整數值，這意味著 Eloquent 會自動將主鍵轉換為整數型別。如果您希望使用非遞增或非數值的主鍵，您應該在 `Table` 屬性上指定 `keyType` 和 `incrementing` 引數：

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

如果您只需要停用自動遞增 ID，可以使用 `WithoutIncrementing` 屬性：

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

Eloquent 要求每個 Model 至少有一個可以用作其主鍵的唯一識別「ID」。Eloquent Model 不支援「複合 (Composite)」主鍵。然而，除了資料表的唯一識別主鍵外，您仍可以自由地在資料表中添加額外的多欄位唯一索引。


<a name="uuid-and-ulid-keys"></a>
### UUID 與 ULID 鍵

您可以選擇使用 UUID，而不是使用自動遞增的整數作為 Eloquent Model 的主鍵。UUID 是長度為 36 個字元的通用唯一英數識別碼。

如果您希望 Model 使用 UUID 鍵而不是自動遞增的整數鍵，可以在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUuids` trait。當然，您應該確保 Model 具有一個[等同於 UUID 的主鍵欄位](/docs/{{version}}/migrations#column-method-uuid)：

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

預設情況下，`HasUuids` trait 會為您的 Model 產生 [UUIDv7](/docs/{{version}}/strings#method-str-uuid7) 識別碼。這些 UUID 對於索引資料庫儲存更有效率，因為它們可以按字典順序排序。

您可以透過在 Model 上定義 `newUniqueId` 方法來覆寫特定 Model 的 UUID 產生過程。此外，您可以透過在 Model 上定義 `uniqueIds` 方法來指定哪些欄位應該接收 UUID：

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

如果您願意，也可以選擇使用「ULID」而不是 UUID。ULID 與 UUID 類似；但是它們的長度只有 26 個字元。就像排序後的 UUID 一樣，ULID 可以按字典順序排序，以便進行高效的資料庫索引。要使用 ULID，您應該在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUlids` trait。您還應該確保 Model 有一個[等同於 ULID 的主鍵欄位](/docs/{{version}}/migrations#column-method-ulid)：

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

預設情況下，Eloquent 預期 `created_at` 和 `updated_at` 欄位存在於您 Model 對應的資料表中。當建立或更新 Model 時，Eloquent 會自動設定這些欄位的值。如果您不希望這些欄位由 Eloquent 自動管理，可以在 Model 的 `Table` 屬性上將 `timestamps` 設定為 `false`：

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

如果您只需要停用時間戳記，可以使用 `WithoutTimestamps` 屬性：

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

如果您需要自訂 Model 時間戳記的格式，可以使用 `Table` 屬性上的 `dateFormat` 引數。這決定了日期屬性在資料庫中的儲存方式，以及 Model 被序列化為陣列或 JSON 時的格式：

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

如果您需要自訂用於儲存時間戳記的欄位名稱，可以在 Model 中定義 `CREATED_AT` 和 `UPDATED_AT` 常數：

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

如果您希望在不修改 Model 的 `updated_at` 時間戳記的情況下執行 Model 操作，可以在傳遞給 `withoutTimestamps` 方法的閉包內操作該 Model：

```php
Model::withoutTimestamps(fn () => $post->increment('reads'));
```

<a name="database-connections"></a>
### 資料庫連線

預設情況下，所有 Eloquent model 都會使用應用程式所設定的預設資料庫連線。如果你想為特定 model 指定不同的連線，可以使用 `Connection` 屬性：

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

預設情況下，新建立的 model 實例不會包含任何屬性值。如果你想為某些 model 屬性定義預設值，可以在 model 中定義 `$attributes` 屬性。放在 `$attributes` 陣列中的屬性值應該採用原始的「可儲存 (Storable)」格式，就像剛從資料庫讀取出來時一樣：

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flight extends Model
{
    /**
     * The model's default values for attributes.
     *
     * @var array<string, mixed>
     */
    protected $attributes = [
        'options' => '[]',
        'delayed' => false,
    ];
}
```

<a name="configuring-eloquent-strictness"></a>
### 設定 Eloquent 嚴謹度

Laravel 提供了一些方法，讓你可以在各種情況下設定 Eloquent 的行為與「嚴謹度」。

首先，`preventLazyLoading` 方法接受一個選填的布林值引數，用來指示是否應防止延遲載入 (Lazy Loading)。例如，你可能只希望在非正式環境中停用延遲載入，這樣即使正式環境的程式碼中不小心出現了延遲載入的關聯，正式環境仍能正常運作。通常，這個方法應該在你應用程式的 `AppServiceProvider` 中的 `boot` 方法內呼叫：

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

此外，你也可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填充 (Fill) 不可填充的屬性時拋出例外。這有助於在本地開發期間，防止因嘗試設定未被加入 model `fillable` 陣列的屬性而導致的非預期錯誤：

```php
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

<a name="retrieving-models"></a>
## 取得 Model

當你建立好 Model 及其[關聯的資料表](/docs/{{version}}/migrations#generating-migrations)後，就可以開始從資料庫中取得資料。你可以將每個 Eloquent Model 視為一個強大的[查詢產生器 (Query Builder)](/docs/{{version}}/queries)，讓你能夠流暢地查詢與該 Model 關聯的資料表。Model 的 `all` 方法會取得該 Model 關聯資料表中的所有紀錄：

```php
use App\Models\Flight;

foreach (Flight::all() as $flight) {
    echo $flight->name;
}
```


<a name="building-queries"></a>
#### 建立查詢

Eloquent 的 `all` 方法會回傳該 Model 資料表中的所有結果。然而，由於每個 Eloquent Model 都可以作為[查詢產生器](/docs/{{version}}/queries)使用，你可以為查詢增加額外的約束條件，然後呼叫 `get` 方法來取得結果：

```php
$flights = Flight::where('active', 1)
    ->orderBy('name')
    ->limit(10)
    ->get();
```

> [!NOTE]
> 由於 Eloquent Model 就是查詢產生器，你應該查看 Laravel [查詢產生器](/docs/{{version}}/queries)提供的所有方法。在撰寫 Eloquent 查詢時，你可以使用其中的任何方法。


<a name="refreshing-models"></a>
#### 重整 Model

如果你已經有一個從資料庫取得的 Eloquent Model 實例，你可以使用 `fresh` 和 `refresh` 方法來「重整」該 Model。`fresh` 方法會從資料庫重新取得該 Model，而原本的 Model 實例不會受到影響：

```php
$flight = Flight::where('number', 'FR 900')->first();

$freshFlight = $flight->fresh();
```

`refresh` 方法會使用來自資料庫的新資料重新載入 (Re-hydrate) 現有的 Model 實例。此外，所有已載入的關聯也都會被重整：

```php
$flight = Flight::where('number', 'FR 900')->first();

$flight->number = 'FR 456';

$flight->refresh();

$flight->number; // "FR 900"
```


<a name="collections"></a>
### 集合

如我們所見，像 `all` 和 `get` 這樣的 Eloquent 方法會從資料庫中取得多筆紀錄。然而，這些方法回傳的並非普通的 PHP 陣列，而是一個 `Illuminate\Database\Eloquent\Collection` 的實例。

Eloquent 的 `Collection` 類別繼承了 Laravel 的基礎 `Illuminate\Support\Collection` 類別，該類別提供了[多種實用的方法](/docs/{{version}}/collections#available-methods)來操作資料集合。例如，`reject` 方法可用於根據閉包的執行結果從集合中移除 Model：

```php
$flights = Flight::where('destination', 'Paris')->get();

$flights = $flights->reject(function (Flight $flight) {
    return $flight->cancelled;
});
```

除了 Laravel 基礎集合類別提供的方法外，Eloquent 集合類別還提供了[一些額外的方法](/docs/{{version}}/eloquent-collections#available-methods)，專門用於操作 Eloquent Model 的集合。

由於所有的 Laravel 集合都實作了 PHP 的可迭代 (Iterable) 介面，你可以像操作陣列一樣對集合進行迴圈走訪：

```php
foreach ($flights as $flight) {
    echo $flight->name;
}
```


<a name="chunking-results"></a>
### 分塊處理結果

如果你嘗試透過 `all` 或 `get` 方法載入數萬筆 Eloquent 紀錄，你的應用程式可能會耗盡記憶體。與其使用這些方法，不如使用 `chunk` 方法來更有效率地處理大量 Model。

`chunk` 方法會取得 Eloquent Model 的子集，並將它們傳遞給閉包進行處理。由於一次只會取得當前區塊的 Eloquent Model，因此在處理大量 Model 時，`chunk` 方法能顯著減少記憶體使用量：

```php
use App\Models\Flight;
use Illuminate\Database\Eloquent\Collection;

Flight::chunk(200, function (Collection $flights) {
    foreach ($flights as $flight) {
        // ...
    }
});
```

傳遞給 `chunk` 方法的第一個引數是每個「分塊 (Chunk)」你希望接收的紀錄數量。作為第二個引數傳遞的閉包將針對從資料庫取得的每個分塊進行呼叫。每次為了取得傳遞給閉包的分塊紀錄時，都會執行一次資料庫查詢。

如果你在過濾 `chunk` 方法的結果時，所依據的欄位同時也會在迭代結果時被更新，則你應該使用 `chunkById` 方法。在這些情境下使用 `chunk` 方法可能會導致意外且不一致的結果。在內部，`chunkById` 方法始終會取得 `id` 欄位大於前一個分塊中最後一個 Model 的紀錄：

```php
Flight::where('departed', true)
    ->chunkById(200, function (Collection $flights) {
        $flights->each->update(['departed' => false]);
    }, column: 'id');
```

由於 `chunkById` 和 `lazyById` 方法會在其執行的查詢中加入自己的 "where" 條件，因此你通常應該在閉包內對你自己的條件進行[邏輯分組](/docs/{{version}}/queries#logical-grouping)：

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
### 使用 Lazy Collections 分塊

`lazy` 方法運作方式與 [`chunk` 方法](#chunking-results)類似，在幕後它也會分塊執行查詢。然而，`lazy` 方法並非將每個分塊直接傳遞給回呼函數，而是回傳一個扁平化的 Eloquent Model [LazyCollection](/docs/{{version}}/collections#lazy-collections)，這讓你能夠將結果視為單一串流進行操作：

```php
use App\Models\Flight;

foreach (Flight::lazy() as $flight) {
    // ...
}
```

如果你在過濾 `lazy` 方法的結果時，所依據的欄位同時也會在迭代結果時被更新，則你應該使用 `lazyById` 方法。在內部，`lazyById` 方法始終會取得 `id` 欄位大於前一個分塊中最後一個 Model 的紀錄：

```php
Flight::where('departed', true)
    ->lazyById(200, column: 'id')
    ->each->update(['departed' => false]);
```

你可以使用 `lazyByIdDesc` 方法根據 `id` 的降冪順序來過濾結果。

<a name="cursors"></a>
### 指標 (Cursors)

與 `lazy` 方法類似，當您需要疊代數萬條 Eloquent 模型紀錄時，可以使用 `cursor` 方法來顯著降低應用程式的記憶體消耗。

`cursor` 方法只會執行單一資料庫查詢；然而，個別的 Eloquent 模型在實際被疊代之前不會被實例化 (Hydrated)。因此，在疊代指標的過程中，任何時間點都只會有一個 Eloquent 模型保留在記憶體中。

> [!WARNING]
> 由於 `cursor` 方法在記憶體中一次只會保留一個 Eloquent 模型，因此它無法預載 (Eager Load) 關聯。如果您需要預載關聯，請考慮改用 [`lazy` 方法](#chunking-using-lazy-collections)。

在內部，`cursor` 方法使用 PHP [產生器 (Generators)](https://www.php.net/manual/en/language.generators.overview.php) 來實作此功能：

```php
use App\Models\Flight;

foreach (Flight::where('destination', 'Zurich')->cursor() as $flight) {
    // ...
}
```

`cursor` 會回傳一個 `Illuminate\Support\LazyCollection` 執行個體。[Lazy collections](/docs/{{version}}/collections#lazy-collections) 讓您可以使用許多典型 Laravel 集合中可用的集合方法，同時每次只將一個模型載入記憶體：

```php
use App\Models\User;

$users = User::cursor()->filter(function (User $user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```

雖然 `cursor` 方法使用的記憶體遠少於一般查詢（因為一次只在記憶體中保留一個 Eloquent 模型），但最終仍可能耗盡記憶體。這是 [由於 PHP 的 PDO 驅動程式在內部將所有原始查詢結果快取在其緩衝區中](https://www.php.net/manual/en/mysqlinfo.concepts.buffering.php)。如果您正在處理極大量的 Eloquent 紀錄，請考慮改用 [`lazy` 方法](#chunking-using-lazy-collections) 來代替。


<a name="advanced-subqueries"></a>
### 進階子查詢


<a name="subquery-selects"></a>
#### 子查詢 Select

Eloquent 也提供了進階的子查詢支援，這讓您可以在單次查詢中從關聯資料表取得資訊。舉例來說，假設我們有一個航班目的地 `destinations` 資料表，以及一個前往目的地的 `flights` 航班資料表。`flights` 資料表中包含一個 `arrived_at` 欄位，用來表示航班抵達目的地的時間。

利用查詢產生器的 `select` 與 `addSelect` 方法所提供的子查詢功能，我們可以在單次查詢中選取所有的 `destinations`，以及最近抵達該目的地的航班名稱：

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

此外，查詢產生器的 `orderBy` 函式也支援子查詢。延續我們的航班範例，我們可以使用此功能，根據最後一班航班抵達該目的地的時間來排序所有目的地。同樣地，這可以透過執行單次資料庫查詢來完成：

```php
return Destination::orderByDesc(
    Flight::select('arrived_at')
        ->whereColumn('destination_id', 'destinations.id')
        ->orderByDesc('arrived_at')
        ->limit(1)
)->get();
```

<a name="retrieving-single-models"></a>
## 取得單一 Model / 聚合結果

除了取得符合給定查詢的所有紀錄外，您還可以使用 `find`、`first` 或 `firstWhere` 方法來取得單一紀錄。這些方法會回傳單一 Model 實例，而不是回傳一個 Model 的集合 (Collection)：

```php
use App\Models\Flight;

// Retrieve a model by its primary key...
$flight = Flight::find(1);

// Retrieve the first model matching the query constraints...
$flight = Flight::where('active', 1)->first();

// Alternative to retrieving the first model matching the query constraints...
$flight = Flight::firstWhere('active', 1);
```

有時您可能希望在找不到結果時執行其他操作。`findOr` 與 `firstOr` 方法會回傳單一 Model 實例，或者在找不到結果時執行給定的閉包 (Closure)。閉包回傳的值將被視為該方法的結果：

```php
$flight = Flight::findOr(1, function () {
    // ...
});

$flight = Flight::where('legs', '>', 3)->firstOr(function () {
    // ...
});
```

<a name="not-found-exceptions"></a>
#### 找不到結果的異常

有時您可能希望在找不到 Model 時拋出異常。這在路由或控制器中特別有用。`findOrFail` 與 `firstOrFail` 方法會取得查詢的第一個結果；然而，如果找不到結果，則會拋出 `Illuminate\Database\Eloquent\ModelNotFoundException` 異常：

```php
$flight = Flight::findOrFail(1);

$flight = Flight::where('legs', '>', 3)->firstOrFail();
```

如果沒有捕捉到 `ModelNotFoundException`，系統會自動向客戶端發送 404 HTTP 回應：

```php
use App\Models\Flight;

Route::get('/api/flights/{id}', function (string $id) {
    return Flight::findOrFail($id);
});
```

<a name="retrieving-or-creating-models"></a>
### 取得或建立 Model

`firstOrCreate` 方法會嘗試使用給定的欄位 / 值配對來定位資料庫紀錄。如果在資料庫中找不到該 Model，則會插入一筆紀錄，其屬性是將第一個陣列引數與選用的第二個陣列引數合併後的結果。

`firstOrNew` 方法與 `firstOrCreate` 類似，會嘗試在資料庫中尋找符合給定屬性的紀錄。然而，如果找不到 Model，則會回傳一個新的 Model 實例。請注意，由 `firstOrNew` 回傳的 Model 尚未持久化到資料庫中。您需要手動呼叫 `save` 方法來將其儲存：

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
### 取得聚合結果

當與 Eloquent Model 互動時，您還可以使用 Laravel [查詢產生器 (Query Builder)](/docs/{{version}}/queries) 提供的 `count`、`sum`、`max` 以及其他[聚合方法](/docs/{{version}}/queries#aggregates)。正如您所預期的，這些方法會回傳一個純量 (Scalar) 值，而不是一個 Eloquent Model 實例：

```php
$count = Flight::where('active', 1)->count();

$max = Flight::where('active', 1)->max('price');
```

<a name="inserting-and-updating-models"></a>
## 新增與更新 Model


<a name="inserts"></a>
### 新增

當然，使用 Eloquent 時，我們不只需要從資料庫取得 Model，還需要新增紀錄。幸好， Eloquent 讓這件事變得非常簡單。要將新紀錄寫入資料庫，您應該實例化一個新的 Model 實例並設定屬性。接著，在 Model 實例上呼叫 `save` 方法：

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

在此範例中，我們將傳入的 HTTP 請求中的 `name` 欄位指定給 `App\Models\Flight` Model 實例的 `name` 屬性。當我們呼叫 `save` 方法時，紀錄就會被寫入資料庫。當 `save` 方法被呼叫時，Model 的 `created_at` 與 `updated_at` 時間戳記會自動設定，因此不需要手動設定。

若您想在資料庫交易 (Transaction) 中儲存 Model，可以使用 `saveOrFail` 方法。如果在儲存過程中拋出例外，交易將會自動回滾 (Roll back)：

```php
$flight->saveOrFail();
```

此外，您也可以使用 `create` 方法，僅透過一行 PHP 語句即可「儲存」新的 Model。`create` 方法會回傳已新增的 Model 實例：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，您需要在 Model 類別中指定 `Fillable` 或 `Guarded` 屬性。這些屬性是必須的，因為預設情況下所有 Eloquent Model 都受到保護，以防止批量賦值 (Mass Assignment) 漏洞。要進一步瞭解批量賦值，請參閱 [批量賦值文件](#mass-assignment)。


<a name="updates"></a>
### 更新

`save` 方法也可以用來更新資料庫中已存在的 Model。要更新 Model，您應該先取得它，並設定您想要更新的屬性。然後呼叫 Model 的 `save` 方法。同樣地，`updated_at` 時間戳記會自動更新，因此不需要手動設定其值：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->name = 'Paris to London';

$flight->save();
```

若您想在資料庫交易中更新 Model，可以使用 `updateOrFail` 方法。如果在更新過程中拋出例外，交易將會自動回滾：

```php
$flight->updateOrFail(['name' => 'Paris to London']);
```

有時候，您可能需要更新現有的 Model，或是如果沒有相符的 Model 則建立一個新的。就像 `firstOrCreate` 方法一樣，`updateOrCreate` 方法會直接將 Model 持久化到資料庫，因此不需要手動呼叫 `save` 方法。

在下方的範例中，如果已存在起飛地 (`departure`) 為 `Oakland` 且目的地 (`destination`) 為 `San Diego` 的航班，則會更新其 `price` 與 `discounted` 欄位。如果不存在這樣的航班，則會建立一個新的航班，其屬性為第一個引數陣列與第二個引數陣列合併後的結果：

```php
$flight = Flight::updateOrCreate(
    ['departure' => 'Oakland', 'destination' => 'San Diego'],
    ['price' => 99, 'discounted' => 1]
);
```

當使用 `firstOrCreate` 或 `updateOrCreate` 等方法時，您可能不知道是建立了一個新的 Model 還是更新了一個現有的 Model。`wasRecentlyCreated` 屬性可用來指示該 Model 是否是在當前的生命週期中建立的：

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

也可以對符合給定查詢條件的 Model 進行批量更新。在此範例中，所有 `active` 且 `destination` 為 `San Diego` 的航班都將被標記為延遲：

```php
Flight::where('active', 1)
    ->where('destination', 'San Diego')
    ->update(['delayed' => 1]);
```

`update` 方法接受一個包含欄位與值配對的陣列，代表要更新的欄位。`update` 方法會回傳受影響的資料列數量。

> [!WARNING]
> 當透過 Eloquent 執行批量更新時，受更新的 Model 不會觸發 `saving`、`saved`、`updating` 與 `updated` 的 Model 事件。這是因為在執行批量更新時，實際上從未取得過這些 Model。


<a name="examining-attribute-changes"></a>
#### 檢查屬性變動

Eloquent 提供了 `isDirty`、`isClean` 與 `wasChanged` 方法來檢查 Model 的內部狀態，並判斷其屬性自最初取得後發生了哪些變化。

`isDirty` 方法用來判斷自取得 Model 以來，是否有任何屬性被更改過。您可以將特定的屬性名稱或屬性陣列傳遞給 `isDirty` 方法，以判斷特定屬性是否為「dirty」。`isClean` 方法則會判斷自取得 Model 以來，屬性是否保持不變。此方法同樣接受一個選用的屬性引數：

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

`wasChanged` 方法用來判斷在當前的請求週期內，Model 最後一次儲存時是否有任何屬性發生過變動。如果需要，您可以傳入屬性名稱來查看特定屬性是否已被更改：

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

`getOriginal` 方法會回傳一個包含 Model 原始屬性的陣列，不論 Model 取得後發生過什麼變動。如果需要，您可以傳入特定的屬性名稱來取得該屬性的原始值：

```php
$user = User::find(1);

$user->name; // John
$user->email; // john@example.com

$user->name = 'Jack';
$user->name; // Jack

$user->getOriginal('name'); // John
$user->getOriginal(); // Array of original attributes...
```

`getChanges` 方法會回傳一個陣列，包含上次儲存 Model 時發生變動的屬性；而 `getPrevious` 方法會回傳一個陣列，包含 Model 上次儲存前的原始屬性值：

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
### 批量賦值 (Mass Assignment)

您可以使用 `create` 方法，僅透過單一 PHP 敘述來「儲存」一個新的 Model。該方法會回傳被新增的 Model 執行個體：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，您需要在 Model 類別中指定 `Fillable` 或 `Guarded` 屬性。這些屬性是必要的，因為預設情況下，所有的 Eloquent Model 都有防止批量賦值 (Mass Assignment) 安全漏洞的保護機制。

批量賦值漏洞發生在使用者傳送了非預期的 HTTP 請求欄位，而該欄位更改了您不希望被變動的資料庫欄位。例如：惡意使用者可能會透過 HTTP 請求傳送 `is_admin` 參數，接著該參數被傳遞到 Model 的 `create` 方法中，讓該使用者能將自己提升為管理員權限。

因此，在開始之前，您應該定義哪些 Model 屬性是可以被批量賦值的。您可以使用 Model 上的 `Fillable` 屬性來完成。例如，讓我們將 `Flight` Model 的 `name` 屬性設為可批量賦值：

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

一旦您指定了哪些屬性是可批量賦值的，您就可以使用 `create` 方法在資料庫中插入新紀錄。`create` 方法會回傳新建立的 Model 執行個體：

```php
$flight = Flight::create(['name' => 'London to Paris']);
```

如果您已經有一個 Model 執行個體，您可以使用 `fill` 方法，以屬性陣列來填充它：

```php
$flight->fill(['name' => 'Amsterdam to Frankfurt']);
```


<a name="mass-assignment-json-columns"></a>
#### 批量賦值與 JSON 欄位

當賦值給 JSON 欄位時，必須在 Model 的 `Fillable` 屬性中指定每個欄位可批量賦值的鍵。為了安全起見，當使用 `Guarded` 屬性時，Laravel 不支援更新巢狀的 JSON 屬性：

```php
use Illuminate\Database\Eloquent\Attributes\Fillable;

#[Fillable(['options->enabled'])]
class Flight extends Model
{
    // ...
}
```


<a name="allowing-mass-assignment"></a>
#### 允許批量賦值

如果您希望讓所有屬性都能被批量賦值，可以在 Model 上使用 `Unguarded` 屬性。如果您選擇不對 Model 進行保護 (Unguard)，則應特別小心，務必始終手動建構傳遞給 Eloquent 的 `fill`、`create` 和 `update` 方法的陣列：

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
#### 批量賦值例外

預設情況下，執行批量賦值操作時，未包含在 `Fillable` 屬性中的屬性會被自動忽略。在正式環境中，這是預期行為；然而，在本地開發期間，這可能會導致對 Model 變更為何未生效感到困惑。

如果您願意，可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填充不可賦值的屬性時拋出例外。通常，此方法應該在應用程式 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

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
### 更新或新增 (Upserts)

Eloquent 的 `upsert` 方法可用於在單一原子操作中更新或建立紀錄。該方法的第一個引數包含要插入或更新的值，而第二個引數則列出用於唯一識別關聯表中紀錄的欄位。第三個也是最後一個引數是一個陣列，包含如果資料庫中已存在匹配紀錄時應該更新的欄位。如果 Model 開啟了時間戳記，`upsert` 方法將自動設定 `created_at` 與 `updated_at` 時間戳記：

```php
Flight::upsert([
    ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
    ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
], uniqueBy: ['departure', 'destination'], update: ['price']);
```

> [!WARNING]
> 除了 SQL Server 之外，所有資料庫都要求 `upsert` 方法第二個引數中的欄位必須具備「主鍵 (Primary)」或「唯一 (Unique)」索引。此外，MariaDB 和 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個引數，並始終使用資料表的主鍵和唯一索引來偵測現有紀錄。

<a name="deleting-models"></a>
## 刪除 Model

要刪除一個 Model，您可以呼叫 Model 實例上的 `delete` 方法：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->delete();
```

如果您想在資料庫交易中刪除 Model，可以使用 `deleteOrFail` 方法。如果在刪除過程中發生例外，交易將自動回滾：

```php
$flight->deleteOrFail();
```

<a name="deleting-an-existing-model-by-its-primary-key"></a>
#### 透過主鍵刪除現有的 Model

在上面的範例中，我們在呼叫 `delete` 方法之前先從資料庫取得了 Model。然而，如果您已知 Model 的主鍵，則可以呼叫 `destroy` 方法直接刪除 Model，而無需明確地取得它。`destroy` 方法除了接受單一主鍵外，還接受多個主鍵、主鍵陣列或[集合](/docs/{{version}}/collections)：

```php
Flight::destroy(1);

Flight::destroy(1, 2, 3);

Flight::destroy([1, 2, 3]);

Flight::destroy(collect([1, 2, 3]));
```

如果您正在使用[軟刪除 Model](#soft-deleting)，可以使用 `forceDestroy` 方法來永久刪除 Model：

```php
Flight::forceDestroy(1);
```

> [!WARNING]
> `destroy` 方法會個別載入每個 Model 並呼叫 `delete` 方法，以便為每個 Model 正確發送 `deleting` 和 `deleted` 事件。

<a name="deleting-models-using-queries"></a>
#### 使用查詢刪除 Model

當然，您也可以建立一個 Eloquent 查詢來刪除所有符合查詢條件的 Model。在此範例中，我們將刪除所有被標記為不活躍 (inactive) 的航班。與批量更新一樣，批量刪除不會為被刪除的 Model 發送 Model 事件：

```php
$deleted = Flight::where('active', 0)->delete();
```

要刪除資料表中的所有 Model，您應該執行一個不帶任何條件的查詢：

```php
$deleted = Flight::query()->delete();
```

> [!WARNING]
> 當透過 Eloquent 執行批量刪除語句時，將不會為被刪除的 Model 發送 `deleting` 和 `deleted` Model 事件。這是因為在執行刪除語句時，實際上從未取得過這些 Model。

<a name="soft-deleting"></a>
### 軟刪除

除了實際從資料庫中移除紀錄外，Eloquent 還可以「軟刪除」Model。當 Model 被軟刪除時，它們不會真的從資料庫中移除。相反地，會在 Model 上設定一個 `deleted_at` 屬性，標示該 Model 是在何時被「刪除」的。要為 Model 啟用軟刪除，請在 Model 中加入 `Illuminate\Database\Eloquent\SoftDeletes` trait：

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
> `SoftDeletes` trait 會自動幫您將 `deleted_at` 屬性轉換為 `DateTime` / `Carbon` 實例。

您也應該將 `deleted_at` 欄位加入您的資料表。Laravel [結構產生器](/docs/{{version}}/migrations) 包含一個輔助方法來建立此欄位：

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

現在，當您呼叫 Model 的 `delete` 方法時，`deleted_at` 欄位將被設定為目前的日期與時間。然而，該 Model 的資料庫紀錄仍會留在資料表中。當查詢使用軟刪除的 Model 時，軟刪除的 Model 會自動從所有查詢結果中排除。

要判斷給定的 Model 實例是否已被軟刪除，可以使用 `trashed` 方法：

```php
if ($flight->trashed()) {
    // ...
}
```

<a name="restoring-soft-deleted-models"></a>
#### 還原軟刪除的 Model

有時候您可能想要「取消刪除」一個軟刪除的 Model。要還原軟刪除的 Model，您可以在 Model 實例上呼叫 `restore` 方法。`restore` 方法會將 Model 的 `deleted_at` 欄位設定為 `null`：

```php
$flight->restore();
```

您也可以在查詢中使用 `restore` 方法來一次還原多個 Model。同樣地，與其他「批量」操作一樣，這不會為還原的 Model 發送任何 Model 事件：

```php
Flight::withTrashed()
    ->where('airline_id', 1)
    ->restore();
```

在建立[關聯](/docs/{{version}}/eloquent-relationships)查詢時也可以使用 `restore` 方法：

```php
$flight->history()->restore();
```

<a name="permanently-deleting-models"></a>
#### 永久刪除 Model

有時候您可能需要真正地從資料庫中移除 Model。您可以使用 `forceDelete` 方法將軟刪除的 Model 從資料表中永久移除：

```php
$flight->forceDelete();
```

在建立 Eloquent 關聯查詢時也可以使用 `forceDelete` 方法：

```php
$flight->history()->forceDelete();
```

<a name="querying-soft-deleted-models"></a>
### 查詢軟刪除的 Model

<a name="including-soft-deleted-models"></a>
#### 包含軟刪除的 Model

如上所述，軟刪除的 Model 會自動從查詢結果中排除。然而，您可以透過在查詢上呼叫 `withTrashed` 方法，強制將軟刪除的 Model 包含在查詢結果中：

```php
use App\Models\Flight;

$flights = Flight::withTrashed()
    ->where('account_id', 1)
    ->get();
```

在建立[關聯](/docs/{{version}}/eloquent-relationships)查詢時也可以呼叫 `withTrashed` 方法：

```php
$flight->history()->withTrashed()->get();
```

<a name="retrieving-only-soft-deleted-models"></a>
#### 僅取得軟刪除的 Model

`onlyTrashed` 方法將**僅**取得軟刪除的 Model：

```php
$flights = Flight::onlyTrashed()
    ->where('airline_id', 1)
    ->get();
```

<a name="pruning-models"></a>
## 修剪 Model

有時候你可能想要定期刪除不再需要的 Model。為了實現這一點，你可以將 `Illuminate\Database\Eloquent\Prunable` 或 `Illuminate\Database\Eloquent\MassPrunable` trait 新增到你想要定期修剪的 Model 中。在將其中一個 trait 新增到 Model 之後，請實作一個 `prunable` 方法，該方法會回傳一個 Eloquent 查詢生成器 (Query Builder)，用以解析出不再需要的 Model：

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

當將 Model 標記為 `Prunable` 時，你也可以在 Model 中定義一個 `pruning` 方法。此方法將在 Model 被刪除之前呼叫。此方法對於在 Model 從資料庫永久移除之前，刪除與該 Model 關聯的任何額外資源（例如儲存的檔案）非常有用：

```php
/**
 * Prepare the model for pruning.
 */
protected function pruning(): void
{
    // ...
}
```

設定好可修剪的 Model 後，你應該在應用程式的 `routes/console.php` 檔案中排程執行 `model:prune` Artisan 指令。你可以自由選擇執行此指令的適當間隔時間：

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('model:prune')->daily();
```

在後台，`model:prune` 指令會自動偵測應用程式 `app/Models` 目錄下的「Prunable」Model。如果你的 Model 位於不同位置，你可以使用 `--model` 選項來指定 Model 的類別名稱：

```php
Schedule::command('model:prune', [
    '--model' => [Address::class, Flight::class],
])->daily();
```

如果你希望在修剪所有其他偵測到的 Model 時排除特定 Model，可以使用 `--except` 選項：

```php
Schedule::command('model:prune', [
    '--except' => [Address::class, Flight::class],
])->daily();
```

你可以透過執行帶有 `--pretend` 選項的 `model:prune` 指令來測試你的 `prunable` 查詢。在使用預覽 (Pretend) 模式時，`model:prune` 指令只會報告如果實際執行指令將會修剪多少條紀錄：

```shell
php artisan model:prune --pretend
```

> [!WARNING]
> 軟刪除 (Soft deleting) 的 Model 如果符合修剪查詢，將會被永久刪除 (`forceDelete`)。

<a name="mass-pruning"></a>
#### 批量修剪

當 Model 被標記為 `Illuminate\Database\Eloquent\MassPrunable` trait 時，會使用批量刪除查詢從資料庫中刪除 Model。因此，`pruning` 方法將不會被呼叫，也不會發送 `deleting` 和 `deleted` 的 Model 事件。這是因為 Model 在刪除之前從未被實際取出，從而使修剪過程更加高效：

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

你可以使用 `replicate` 方法來建立一個現有 Model 實例的未儲存副本。當你有許多共享相同屬性的 Model 實例時，此方法特別有用：

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

若要從複製到新 Model 的屬性中排除一個或多個屬性，你可以將一個陣列傳遞給 `replicate` 方法：

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

全域範圍 (Global Scopes) 允許您為指定 Model 的所有查詢增加約束條件。Laravel 本身的 [軟刪除](#soft-deleting) 功能就是利用全域範圍來僅從資料庫中取得「未刪除」的 Model。撰寫您自己的全域範圍可以提供一種方便且簡單的方法，確保指定 Model 的每個查詢都能套用特定的約束條件。


<a name="generating-scopes"></a>
#### 產生範圍

要產生一個新的全域範圍，您可以執行 `make:scope` Artisan 指令，該指令會將產生的範圍類別放置在應用程式的 `app/Models/Scopes` 目錄中：

```shell
php artisan make:scope AncientScope
```


<a name="writing-global-scopes"></a>
#### 撰寫全域範圍

撰寫全域範圍很簡單。首先，使用 `make:scope` 指令產生一個實作 `Illuminate\Database\Eloquent\Scope` 介面的類別。`Scope` 介面要求您實作一個 `apply` 方法。`apply` 方法可以根據需要向查詢添加 `where` 約束或其他類型的子句：

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
> 如果您的全域範圍是在查詢的 select 子句中增加欄位，則應該使用 `addSelect` 方法而不是 `select`。這將防止無意中替換掉查詢現有的 select 子句。


<a name="applying-global-scopes"></a>
#### 套用全域範圍

若要將全域範圍指派給 Model，您只需在 Model 上放置 `ScopedBy` 屬性即可：

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

或者，您可以透過覆寫 Model 的 `booted` 方法來手動註冊全域範圍，並呼叫 Model 的 `addGlobalScope` 方法。`addGlobalScope` 方法接受您的範圍實例作為其唯一參數：

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

在上面的範例中將範圍添加到 `App\Models\User` Model 後，呼叫 `User::all()` 方法將執行以下 SQL 查詢：

```sql
select * from `users` where `created_at` < 0021-02-18 00:00:00
```


<a name="anonymous-global-scopes"></a>
#### 匿名全域範圍

Eloquent 也允許您使用閉包來定義全域範圍，這對於不需要獨立類別的簡單範圍特別有用。使用閉包定義全域範圍時，您應該提供一個自訂的範圍名稱作為 `addGlobalScope` 方法的第一個參數：

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

如果您想在特定查詢中移除全域範圍，可以使用 `withoutGlobalScope` 方法。此方法接受全域範圍的類別名稱作為其唯一參數：

```php
User::withoutGlobalScope(AncientScope::class)->get();
```

或者，如果您是使用閉包定義全域範圍，則應傳入指派給該全域範圍的字串名稱：

```php
User::withoutGlobalScope('ancient')->get();
```

如果您想移除查詢中的多個甚至全部全域範圍，可以使用 `withoutGlobalScopes` 和 `withoutGlobalScopesExcept` 方法：

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

區域範圍 (Local Scopes) 允許您定義通用的查詢約束集合，以便在應用程式中輕鬆重複使用。例如，您可能需要頻繁地取得所有被認為是「熱門 (popular)」的使用者。要定義範圍，請在 Eloquent 方法中加上 `Scope` 屬性。

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
#### 使用區域範圍

定義好範圍後，您可以在查詢 Model 時呼叫這些範圍方法。您甚至可以鏈式呼叫多個範圍：

```php
use App\Models\User;

$users = User::popular()->active()->orderBy('created_at')->get();
```

透過 `or` 查詢運算子組合多個 Eloquent Model 範圍時，可能需要使用閉包來達成正確的 [邏輯分組](/docs/{{version}}/queries#logical-grouping)：

```php
$users = User::popular()->orWhere(function (Builder $query) {
    $query->active();
})->get();
```

然而，由於這樣寫可能很繁瑣，Laravel 提供了一個「高階 (higher order)」的 `orWhere` 方法，讓您可以流暢地鏈接範圍而無需使用閉包：

```php
$users = User::popular()->orWhere->active()->get();
```


<a name="dynamic-scopes"></a>
#### 動態範圍

有時您可能希望定義一個接受參數的範圍。首先，只需將額外的參數添加到範圍方法的參數列中。範圍參數應定義在 `$query` 參數之後：

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

一旦將預期的引數添加到範圍方法的參數列中，您就可以在呼叫範圍時傳遞這些引數：

```php
$users = User::ofType('admin')->get();
```

<a name="pending-attributes"></a>
### 待定屬性

如果您想利用範圍 (Scope) 來建立具有與該範圍限制條件相同屬性的 Model，您可以在建構範圍查詢時使用 `withAttributes` 方法：

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

`withAttributes` 方法會使用給定的屬性為查詢加入 `where` 條件，同時也會將這些屬性加入至任何透過該範圍所建立的 Model 中：

```php
$draft = Post::draft()->create(['title' => 'In Progress']);

$draft->hidden; // true
```

若要指示 `withAttributes` 方法不要將 `where` 條件加入至查詢，您可以將 `asConditions` 引數設為 `false`：

```php
$query->withAttributes([
    'hidden' => true,
], asConditions: false);
```

<a name="comparing-models"></a>
## 比較 Model

有時候你可能需要判斷兩個 Model 是否「相同」。`is` 與 `isNot` 方法可用於快速驗證兩個 Model 是否具有相同的主鍵、資料表以及資料庫連線：

```php
if ($post->is($anotherPost)) {
    // ...
}

if ($post->isNot($anotherPost)) {
    // ...
}
```

當使用 `belongsTo`、`hasOne`、`morphTo` 與 `morphOne` [關聯(relationships)](/docs/{{version}}/eloquent-relationships) 時，也可以使用 `is` 與 `isNot` 方法。當你想要在不執行查詢來取得該 Model 的情況下比較關聯 Model 時，這個方法特別有用：

```php
if ($post->author()->is($user)) {
    // ...
}
```

<a name="events"></a>
## 事件

> [!NOTE]
> 想要將您的 Eloquent 事件直接廣播到前端應用程式嗎？請參考 Laravel 的 [Model 事件廣播](/docs/{{version}}/broadcasting#model-broadcasting)。

Eloquent Model 會發送多種事件，讓您可以介入 Model 生命週期中的以下時間點：`retrieved`、`creating`、`created`、`updating`、`updated`、`saving`、`saved`、`deleting`、`deleted`、`trashed`、`forceDeleting`、`forceDeleted`、`restoring`、`restored` 以及 `replicating`。

當從資料庫取得現有的 Model 時，會觸發 `retrieved` 事件。當第一次儲存一個新的 Model 時，會觸發 `creating` 與 `created` 事件。當修改一個現有的 Model 並呼叫 `save` 方法時，會觸發 `updating` / `updated` 事件。無論是建立還是更新 Model，都會觸發 `saving` / `saved` 事件——即使 Model 的屬性沒有變動也是如此。以 `-ing` 結尾的事件名稱會在 Model 的任何變動被持久化 (Persisted) 之前發送，而以 `-ed` 結尾的事件則是在變動被持久化之後發送。

若要開始監聽 Model 事件，請在您的 Eloquent Model 中定義 `$dispatchesEvents` 屬性。此屬性將 Eloquent Model 生命週期的各個點對應到您自己的 [事件類別 (Event Classes)](/docs/{{version}}/events)。每個 Model 事件類別都應預期透過其建構子接收受影響的 Model 實例：

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

定義並對應好 Eloquent 事件後，您可以使用 [事件監聽器 (Event Listeners)](/docs/{{version}}/events#defining-listeners) 來處理這些事件。

> [!WARNING]
> 當透過 Eloquent 進行批量更新或刪除查詢時，受影響的 Model 將不會發送 `saved`、`updated`、`deleting` 與 `deleted` 事件。這是因為在執行批量更新或刪除時，實際上從未取得過這些 Model。


<a name="events-using-closures"></a>
### 使用閉包

除了使用自定義事件類別外，您還可以註冊在各種 Model 事件發生時執行的閉包。通常，您應該在 Model 的 `booted` 方法中註冊這些閉包：

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

如果需要，您可以在註冊 Model 事件時利用 [可進入佇列的匿名事件監聽器](/docs/{{version}}/events#queueable-anonymous-event-listeners)。這將指示 Laravel 使用應用程式的 [佇列](/docs/{{version}}/queues) 在背景執行 Model 事件監聽器：

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

如果您正在監聽某個 Model 的多個事件，可以使用觀察者 (Observers) 將所有的監聽器組合到單一類別中。觀察者類別的方法名稱對應於您想要監聽的 Eloquent 事件。每個方法都接收受影響的 Model 作為其唯一引數。使用 `make:observer` Artisan 指令是建立新觀察者類別最簡單的方法：

```shell
php artisan make:observer UserObserver --model=User
```

此指令會將新的觀察者放置在您的 `app/Observers` 目錄中。如果此目錄不存在，Artisan 會為您建立。您剛建立的觀察者看起來會像這樣：

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

若要註冊觀察者，您可以在對應的 Model 上加上 `ObservedBy` 屬性：

```php
use App\Observers\UserObserver;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy([UserObserver::class])]
class User extends Authenticatable
{
    //
}
```

或者，您可以透過在想要觀察的 Model 上呼叫 `observe` 方法來手動註冊觀察者。您可以在應用程式 `AppServiceProvider` 類別的 `boot` 方法中註冊觀察者：

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
> 觀察者還可以監聽其他事件，例如 `saving` 和 `retrieved`。這些事件在 [事件](#events) 文件中皆有說明。


<a name="observers-and-database-transactions"></a>
#### 觀察者與資料庫交易

當 Model 在資料庫交易中建立時，您可能希望指示觀察者僅在資料庫交易提交 (Commit) 後才執行其事件處理常式。您可以透過在觀察者中實作 `ShouldHandleEventsAfterCommit` 介面來達成此目的。如果目前沒有進行中的資料庫交易，事件處理常式將會立即執行：

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

您偶爾可能需要暫時「靜音」Model 觸發的所有事件。您可以使用 `withoutEvents` 方法來達成此目的。`withoutEvents` 方法接受一個閉包作為其唯一引數。在此閉包中執行的任何程式碼都不會發送 Model 事件，且閉包回傳的任何值都將由 `withoutEvents` 方法回傳：

```php
use App\Models\User;

$user = User::withoutEvents(function () {
    User::findOrFail(1)->delete();

    return User::find(2);
});
```


<a name="saving-a-single-model-without-events"></a>
#### 在不發送事件的情況下儲存單一 Model

有時您可能希望在不發送任何事件的情況下「儲存」給定的 Model。您可以使用 `saveQuietly` 方法來達成此目的：

```php
$user = User::findOrFail(1);

$user->name = 'Victoria Faith';

$user->saveQuietly();
```

您也可以在不發送任何事件的情況下對給定 Model 進行「更新 (Update)」、「刪除 (Delete)」、「軟刪除 (Soft delete)」、「還原 (Restore)」和「複製 (Replicate)」：

```php
$user->deleteQuietly();
$user->forceDeleteQuietly();
$user->restoreQuietly();
```