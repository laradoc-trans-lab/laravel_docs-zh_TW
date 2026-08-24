# Eloquent：入門指南

- [介紹](#introduction)
- [生成 Model 類別](#generating-model-classes)
- [Eloquent Model 慣例](#eloquent-model-conventions)
    - [資料表名稱](#table-names)
    - [主鍵](#primary-keys)
    - [UUID 與 ULID 鍵](#uuid-and-ulid-keys)
    - [時間戳記](#timestamps)
    - [資料庫連線](#database-connections)
    - [預設屬性值](#default-attribute-values)
    - [設定 Eloquent 嚴格程度](#configuring-eloquent-strictness)
- [取得 Model](#retrieving-models)
    - [集合](#collections)
    - [分塊處理結果](#chunking-results)
    - [使用 Lazy Collections 分塊](#chunking-using-lazy-collections)
    - [游標 (Cursors)](#cursors)
    - [進階子查詢](#advanced-subqueries)
- [取得單一 Model / 聚合值](#retrieving-single-models)
    - [取得或建立 Model](#retrieving-or-creating-models)
    - [取得聚合值](#retrieving-aggregates)
- [新增與更新 Model](#inserting-and-updating-models)
    - [新增](#inserts)
    - [更新](#updates)
    - [批量賦值 (Mass Assignment)](#mass-assignment)
    - [更新或新增 (Upserts)](#upserts)
- [刪除 Model](#deleting-models)
    - [軟刪除](#soft-deleting)
    - [查詢軟刪除的 Model](#querying-soft-deleted-models)
- [修剪 Model (Pruning)](#pruning-models)
- [複製 Model](#replicating-models)
- [查詢作用域 (Query Scopes)](#query-scopes)
    - [全域作用域](#global-scopes)
    - [局部作用域](#local-scopes)
    - [待處理屬性](#pending-attributes)
- [比較 Model](#comparing-models)
- [事件 (Events)](#events)
    - [使用閉包](#events-using-closures)
    - [觀察者 (Observers)](#observers)
    - [靜音事件 (Muting Events)](#muting-events)

<a name="introduction"></a>
## 介紹

Laravel 包含 Eloquent，這是一個物件關聯對映 (ORM)，讓您能愉快地與資料庫互動。使用 Eloquent 時，每個資料庫資料表都有一個對應的「Model」，用來與該資料表進行互動。除了從資料庫資料表中取得記錄之外，Eloquent Model 還允許您在資料表中新增、更新和刪除記錄。

> [!NOTE]
> 在開始之前，請務必在應用程式的 `config/database.php` 設定檔中設定資料庫連線。有關設定資料庫的更多資訊，請參閱[資料庫設定文件](/docs/{{version}}/database#configuration)。


<a name="generating-model-classes"></a>
## 生成 Model 類別

首先，讓我們建立一個 Eloquent Model。Model 通常位於 `app\Models` 目錄中，並繼承 `Illuminate\Database\Eloquent\Model` 類別。您可以使用 `make:model` [Artisan 指令](/docs/{{version}}/artisan)來生成新的 Model：

```shell
php artisan make:model Flight
```

如果您希望在生成 Model 時同時生成[資料庫遷移](/docs/{{version}}/migrations)，可以使用 `--migration` 或 `-m` 選項：

```shell
php artisan make:model Flight --migration
```

生成 Model 時，您也可以生成其他各種類型的類別，例如 Factory、Seeder、Policy、Controller 和表單請求(Form request)。此外，這些選項可以組合使用，以一次建立多個類別：

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
#### 檢視 Model

有時僅透過瀏覽程式碼很難確定 Model 的所有可用屬性和關聯。這時可以嘗試使用 `model:show` Artisan 指令，它提供了該 Model 所有屬性與關聯的便捷概覽：

```shell
php artisan model:show Flight
```

<a name="eloquent-model-conventions"></a>
## Eloquent Model 慣例

由 `make:model` 命令產生的 Model 會放置在 `app/Models` 目錄中。讓我們來看一個基本的 Model 類別，並討論一些 Eloquent 的核心慣例：

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

看過上面的範例後，您可能已經注意到我們並沒有告訴 Eloquent 哪個資料庫資料表對應於我們的 `Flight` Model。按照慣例，除非明確指定其他名稱，否則將使用類別名稱的「蛇形命名法 (snake case)」複數形式作為資料表名稱。因此在這種情況下，Eloquent 會假設 `Flight` Model 將記錄儲存在 `flights` 資料表中，而 `AirTrafficController` Model 則會將記錄儲存在 `air_traffic_controllers` 資料表中。

如果您的 Model 對應的資料庫資料表不符合此慣例，您可以使用 `Table` 屬性 (attribute) 手動指定 Model 的資料表名稱：

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

Eloquent 也會假設每個 Model 對應的資料庫資料表都有一個名為 `id` 的主鍵欄位。如有必要，您可以使用 `Table` 屬性上的 `key` 引數來指定不同的欄位作為 Model 的主鍵：

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

此外，Eloquent 預設主鍵是一個遞增的整數值，這表示 Eloquent 會自動將主鍵型別轉換為整數。如果您希望使用非遞增或非數值的主鍵，您應該在 `Table` 屬性上指定 `keyType` 和 `incrementing` 引數：

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

Eloquent 要求每個 Model 至少要有一個具唯一識別性的「ID」作為其主鍵。Eloquent Model 不支援「複合 (Composite)」主鍵。但是，除了資料表的唯一識別主鍵之外，您可以自由地在資料庫資料表中新增多欄位的唯一索引。

<a name="uuid-and-ulid-keys"></a>
### UUID 與 ULID 鍵

您可以選擇使用 UUID 來代替自動遞增整數作為 Eloquent Model 的主鍵。UUID 是由 36 個字元組成的通用唯一字母數字識別碼。

如果您希望 Model 使用 UUID 鍵而不是自動遞增整數鍵，可以在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUuids` trait。當然，您應該確保 Model 具有[對等的 UUID 主鍵欄位](/docs/{{version}}/migrations#column-method-uuid)：

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

預設情況下，`HasUuids` trait 會為您的 Model 產生 [UUIDv7](/docs/{{version}}/strings#method-str-uuid7) 識別碼。這些 UUID 對於建立索引的資料庫儲存更有效率，因為它們可以按字典順序進行排序。

您可以透過在 Model 上定義 `newUniqueId` 方法來覆寫特定 Model 的 UUID 產生過程。此外，您可以透過在 Model 上定義 `uniqueIds` 方法來指定哪些欄位應接收 UUID：

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

如果您願意，也可以選擇使用「ULID」代替 UUID。ULID 與 UUID 類似；但是它們只有 26 個字元的長度。與有序 UUID 一樣，ULID 也可以按字典順序排序，以實現高效率的資料庫索引。若要使用 ULID，您應該在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUlids` trait。您還應該確保 Model 具有[對等的 ULID 主鍵欄位](/docs/{{version}}/migrations#column-method-ulid)：

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

預設情況下，Eloquent 期望您的 Model 對應的資料庫資料表上存在 `created_at` 和 `updated_at` 欄位。當建立或更新 Model 時，Eloquent 會自動設定這些欄位的值。如果您不希望這些欄位由 Eloquent 自動管理，您可以在 Model 的 `Table` 屬性上將 `timestamps` 設定為 `false`：

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

如果您需要自訂 Model 時間戳記的格式，可以在 `Table` 屬性上使用 `dateFormat` 引數。這決定了日期屬性在資料庫中的儲存方式，以及 Model 序列化為陣列或 JSON 時的格式：

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

如果您需要自訂用來儲存時間戳記的欄位名稱，可以在 Model 上定義 `CREATED_AT` 和 `UPDATED_AT` 常數：

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

如果您想在執行 Model 操作時不修改 Model 的 `updated_at` 時間戳記，可以在傳入 `withoutTimestamps` 方法的閉包中對 Model 進行操作：

```php
Model::withoutTimestamps(fn () => $post->increment('reads'));
```

<a name="database-connections"></a>
### 資料庫連線

預設情況下，所有 Eloquent Model 都會使用為您應用程式設定的預設資料庫連線。如果您想指定與特定 Model 互動時應使用的不同連線，可以使用 `Connection` 屬性 (Attribute)：

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
### 預設屬性值

預設情況下，新實例化的 Model 實例不會包含任何屬性值。如果您想為 Model 的某些屬性定義預設值，可以在 Model 上定義 `$attributes` 屬性 (Property)。放在 `$attributes` 陣列中的屬性值應該採用原始的、「可儲存」的格式，就像剛從資料庫中讀取出來一樣：

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
### 設定 Eloquent 嚴格程度

Laravel 提供了幾種方法，讓您可以在各種情況下設定 Eloquent 的行為和「嚴格程度」。

首先，`preventLazyLoading` 方法接受一個可選的布林引數，用於表示是否應該阻止延遲載入 (Lazy Loading)。例如，您可能希望只在非正式環境中停用延遲載入，這樣即使正式環境程式碼中意外出現延遲載入的關聯，您的正式環境也能繼續正常運作。通常，此方法應該在應用程式的 `AppServiceProvider` 的 `boot` 方法中調用：

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

此外，您可以透過調用 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填充不可填充的屬性時拋出例外。這有助於在本地開發期間嘗試設定尚未加入 Model 的 `fillable` 陣列中的屬性時，防止發生未預期的錯誤：

```php
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

<a name="retrieving-models"></a>
## 取得 Model

建立好 Model 及其[關聯的資料庫資料表](/docs/{{version}}/migrations#generating-migrations)後，就可以開始從資料庫中取得資料。你可以將每個 Eloquent Model 視為強大的[查詢建構器](/docs/{{version}}/queries)，讓你能夠流暢地查詢與該 Model 相關聯的資料庫資料表。Model 的 `all` 方法將取得與該 Model 關聯之資料庫資料表中的所有記錄：

```php
use App\Models\Flight;

foreach (Flight::all() as $flight) {
    echo $flight->name;
}
```

<a name="building-queries"></a>
#### 建立查詢

Eloquent 的 `all` 方法會回傳 Model 資料表中的所有結果。然而，由於每個 Eloquent Model 本身就是一個[查詢建構器](/docs/{{version}}/queries)，因此你可以在查詢中加入其他條件約束，然後呼叫 `get` 方法來取得結果：

```php
$flights = Flight::where('active', 1)
    ->orderBy('name')
    ->limit(10)
    ->get();
```

> [!NOTE]
> 由於 Eloquent Model 就是查詢建構器，建議你檢閱 Laravel [查詢建構器](/docs/{{version}}/queries)所提供的所有方法。在撰寫 Eloquent 查詢時，你可以使用其中的任何方法。

<a name="refreshing-models"></a>
#### 重新整理 Model

若你手邊已經有一個從資料庫中取得的 Eloquent Model 實例，可以使用 `fresh` 與 `refresh` 方法來「重新整理」該 Model。`fresh` 方法會從資料庫中重新取得該 Model，現有的 Model 實例不會受到影響：

```php
$flight = Flight::where('number', 'FR 900')->first();

$freshFlight = $flight->fresh();
```

`refresh` 方法會使用來自資料庫的全新資料重新填充現有的 Model。此外，其所有已載入的關聯也會一併重新整理：

```php
$flight = Flight::where('number', 'FR 900')->first();

$flight->number = 'FR 456';

$flight->refresh();

$flight->number; // "FR 900"
```

<a name="collections"></a>
### 集合

如我們所見，像 `all` 和 `get` 這類 Eloquent 方法會從資料庫中取得多筆記錄。不過，這些方法回傳的並非一般的 PHP 陣列，而是一個 `Illuminate\Database\Eloquent\Collection` 的實例。

Eloquent 的 `Collection` 類別繼承了 Laravel 的基礎 `Illuminate\Support\Collection` 類別，該類別提供了[多種實用的方法](/docs/{{version}}/collections#available-methods)來操作資料集合。例如，`reject` 方法可用於根據傳入閉包的執行結果從集合中移除 Model：

```php
$flights = Flight::where('destination', 'Paris')->get();

$flights = $flights->reject(function (Flight $flight) {
    return $flight->cancelled;
});
```

除了 Laravel 基礎集合類別提供的方法之外，Eloquent 集合類別還提供了一些專門用於操作 Eloquent Model 集合的[額外方法](/docs/{{version}}/eloquent-collections#available-methods)。

由於所有 Laravel 的集合都實作了 PHP 的可迭代介面，因此你可以像操作陣列一樣對集合進行迴圈走訪：

```php
foreach ($flights as $flight) {
    echo $flight->name;
}
```

<a name="chunking-results"></a>
### 分塊處理結果

若嘗試透過 `all` 或 `get` 方法載入數萬筆 Eloquent 記錄，你的應用程式可能會耗盡記憶體。與其使用這些方法，不如使用 `chunk` 方法來更有效率地處理大量 Model。

`chunk` 方法會取得 Eloquent Model 的子集，並將它們傳遞給閉包進行處理。由於每次只會取得當前分塊的 Eloquent Model，因此在處理大量 Model 時，`chunk` 方法可以大幅降低記憶體用量：

```php
use App\Models\Flight;
use Illuminate\Database\Eloquent\Collection;

Flight::chunk(200, function (Collection $flights) {
    foreach ($flights as $flight) {
        // ...
    }
});
```

傳遞給 `chunk` 方法的第一個引數是你希望每個「分塊」接收的記錄數量。傳入作為第二個引數的閉包將會針對從資料庫取得的每個分塊執行一次。每次將分塊記錄傳遞給閉包時，都會執行一次資料庫查詢。

若你在迭代結果的同時，還會更新用於篩選 `chunk` 方法結果的欄位，則應該使用 `chunkById` 方法。在此類情境下使用 `chunk` 方法可能會導致非預期且不一致的結果。在內部實作上，`chunkById` 方法始終會取得其 `id` 欄位大於前一個分塊中最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->chunkById(200, function (Collection $flights) {
        $flights->each->update(['departed' => false]);
    }, column: 'id');
```

由於 `chunkById` 與 `lazyById` 方法會將自訂的「where」條件加入至正在執行的查詢中，因此通常建議在閉包中為你自己的條件進行[邏輯分組](/docs/{{version}}/queries#logical-grouping)：

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

`lazy` 方法的運作方式類似於 [`chunk` 方法](#chunking-results)，在底層也是透過分塊執行查詢。然而，`lazy` 方法不會直接將每個分塊傳入回呼函式中，而是回傳一個扁平化的 Eloquent Model [LazyCollection](/docs/{{version}}/collections#lazy-collections)，讓你可以像單一資料流一樣操作查詢結果：

```php
use App\Models\Flight;

foreach (Flight::lazy() as $flight) {
    // ...
}
```

若你在迭代結果的同時，還會更新用於篩選 `lazy` 方法結果的欄位，則應該使用 `lazyById` 方法。在內部實作上，`lazyById` 方法始終會取得其 `id` 欄位大於前一個分塊中最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->lazyById(200, column: 'id')
    ->each->update(['departed' => false]);
```

你可以使用 `lazyByIdDesc` 方法根據 `id` 的遞減順序來篩選結果。

<a name="cursors"></a>
### 游標 (Cursors)

類似於 `lazy` 方法，當疊代處理數萬筆 Eloquent Model 記錄時，可以使用 `cursor` 方法大幅減少應用程式的記憶體消耗。

`cursor` 方法只會執行單一資料庫查詢；然而，各個 Eloquent Model 直到被實際疊代時才會被實例化 (Hydrated)。因此，在疊代游標的過程中，任何時間點都只會有一個 Eloquent Model 保留在記憶體中。

> [!WARNING]
> 由於 `cursor` 方法在任何時間點只會在記憶體中保留單一 Eloquent Model，因此它無法預先載入關聯。如果您需要預先載入關聯，請考慮改用 [`lazy` 方法](#chunking-using-lazy-collections)。

在內部運作上，`cursor` 方法使用 PHP 的 [生成器 (Generators)](https://www.php.net/manual/en/language.generators.overview.php) 來實作此功能：

```php
use App\Models\Flight;

foreach (Flight::where('destination', 'Zurich')->cursor() as $flight) {
    // ...
}
```

`cursor` 會回傳一個 `Illuminate\Support\LazyCollection` 實例。[Lazy collections](/docs/{{version}}/collections#lazy-collections) 允許您使用一般 Laravel 集合上的許多集合方法，同時每次只將單一 Model 載入到記憶體中：

```php
use App\Models\User;

$users = User::cursor()->filter(function (User $user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```

雖然 `cursor` 方法比一般查詢使用的記憶體少得多（透過一次僅在記憶體中保留單一 Eloquent Model），但它最終仍可能會耗盡記憶體。這是[由於 PHP 的 PDO 驅動程式會在內部緩衝區中快取所有原始查詢結果](https://www.php.net/manual/en/mysqlinfo.concepts.buffering.php)。如果您正在處理極大量的 Eloquent 記錄，請考慮改用 [`lazy` 方法](#chunking-using-lazy-collections)。


<a name="advanced-subqueries"></a>
### 進階子查詢


<a name="subquery-selects"></a>
#### 子查詢 Selects

Eloquent 還提供了進階子查詢支援，讓您可以透過單一查詢從相關資料表中提取資訊。例如，假設我們有一個航班目的地 `destinations` 資料表，以及一個飛往目的地的航班 `flights` 資料表。`flights` 資料表包含一個 `arrived_at` 欄位，表示航班何時抵達目的地。

使用查詢建構器的 `select` 和 `addSelect` 方法提供的子查詢功能，我們可以使用單一查詢選取所有 `destinations` 以及最近抵達該目的地的航班名稱：

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

此外，查詢建構器的 `orderBy` 函式也支援子查詢。繼續使用我們的航班範例，我們可以使用此功能根據最後一班航班抵達該目的地的時間來對所有目的地進行排序。同樣地，這可以在執行單一資料庫查詢時完成：

```php
return Destination::orderByDesc(
    Flight::select('arrived_at')
        ->whereColumn('destination_id', 'destinations.id')
        ->orderByDesc('arrived_at')
        ->limit(1)
)->get();
```

<a name="retrieving-single-models"></a>
## 取得單一 Model / 聚合值

除了取得符合指定查詢的所有記錄之外，您也可以使用 `find`、`first` 或 `firstWhere` 方法來取得單一記錄。這些方法不會返回 Model 集合，而是返回單一 Model 實例：

```php
use App\Models\Flight;

// Retrieve a model by its primary key...
$flight = Flight::find(1);

// Retrieve the first model matching the query constraints...
$flight = Flight::where('active', 1)->first();

// Alternative to retrieving the first model matching the query constraints...
$flight = Flight::firstWhere('active', 1);
```

有時若找不到任何結果，您可能希望執行其他操作。`findOr` 與 `firstOr` 方法會返回單一 Model 實例，或者在未找到結果時執行給定的閉包。閉包所返回的值將被視為該方法的結果：

```php
$flight = Flight::findOr(1, function () {
    // ...
});

$flight = Flight::where('legs', '>', 3)->firstOr(function () {
    // ...
});
```


<a name="not-found-exceptions"></a>
#### 找不到時拋出例外 (Not Found Exceptions)

有時若找不到 Model，您可能希望拋出例外。這在路由或控制器中特別實用。`findOrFail` 與 `firstOrFail` 方法將取得查詢的第一個結果；然而，如果未找到任何結果，則會拋出 `Illuminate\Database\Eloquent\ModelNotFoundException`：

```php
$flight = Flight::findOrFail(1);

$flight = Flight::where('legs', '>', 3)->firstOrFail();
```

如果未捕獲 `ModelNotFoundException`，系統將自動向客戶端發送 404 HTTP 回應：

```php
use App\Models\Flight;

Route::get('/api/flights/{id}', function (string $id) {
    return Flight::findOrFail($id);
});
```


<a name="retrieving-or-creating-models"></a>
### 取得或建立 Model

`firstOrCreate` 方法會嘗試使用給定的欄位 / 數值配對來尋找資料庫記錄。如果在資料庫中找不到該 Model，則會將第一個陣列引數與可選的第二個陣列引數合併後的屬性插入一筆記錄。

`firstOrNew` 方法與 `firstOrCreate` 類似，會嘗試在資料庫中尋找符合給定屬性的記錄。但是，如果未找到 Model，則會返回一個新的 Model 實例。請注意，`firstOrNew` 返回的 Model 尚未保存至資料庫中。您需要手動呼叫 `save` 方法來將其保存：

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
### 取得聚合值

與 Eloquent Model 互動時，您也可以使用 Laravel [查詢建構器](/docs/{{version}}/queries) 所提供的 `count`、`sum`、`max` 以及其他[聚合方法](/docs/{{version}}/queries#aggregates)。正如您所預期的，這些方法會返回純量值，而非 Eloquent Model 實例：

```php
$count = Flight::where('active', 1)->count();

$max = Flight::where('active', 1)->max('price');
```

<a name="inserting-and-updating-models"></a>
## 新增與更新 Model

<a name="inserts"></a>
### 新增

當然，使用 Eloquent 時，我們不僅僅需要從資料庫中取得 Model，還需要新增資料。值得慶幸的是，Eloquent 讓這件事變得非常簡單。若要將新紀錄新增至資料庫，您應該實例化一個新的 Model 實例並設定該 Model 的屬性。接著，呼叫該 Model 實例上的 `save` 方法：

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

在這個範例中，我們將傳入 HTTP 請求中的 `name` 欄位指派給 `App\Models\Flight` Model 實例的 `name` 屬性。當我們呼叫 `save` 方法時，就會將一筆紀錄新增至資料庫中。呼叫 `save` 方法時，Model 的 `created_at` 與 `updated_at` 時間戳記將會自動設定，因此無需手動賦值。

如果您希望在資料庫交易 (Transaction) 內儲存 Model，可以使用 `saveOrFail` 方法。如果在儲存過程中拋出例外狀況，該交易將會自動復原 (Rollback)：

```php
$flight->saveOrFail();
```

或者，您也可以使用 `create` 方法，透過單一 PHP 語句來「儲存」新的 Model。`create` 方法將會回傳已新增的 Model 實例：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，您必須先在 Model 類別上指定 `Fillable` 或 `Guarded` 屬性。這些屬性是必要的，因為所有 Eloquent Model 預設都受到保護，以防止批量賦值 (Mass Assignment) 弱點。若要深入瞭解批量賦值，請參閱[批量賦值文件](#mass-assignment)。

<a name="updates"></a>
### 更新

`save` 方法也可以用來更新資料庫中已存在的 Model。若要更新 Model，您應該先將其取出，設定您想要更新的任何屬性，接著呼叫 Model 的 `save` 方法。同樣地，`updated_at` 時間戳記將會自動更新，因此無需手動設定其值：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->name = 'Paris to London';

$flight->save();
```

如果您希望在資料庫交易內更新 Model，可以使用 `updateOrFail` 方法。如果在更新過程中拋出例外狀況，該交易將會自動復原：

```php
$flight->updateOrFail(['name' => 'Paris to London']);
```

有時，您可能需要更新現有的 Model，或者在沒有相符 Model 時建立新的 Model。就像 `firstOrCreate` 方法一樣，`updateOrCreate` 方法會自動持久化 Model，因此無需手動呼叫 `save` 方法。

在下方的範例中，如果存在一筆 `departure` 地點為 `Oakland` 且 `destination` 地點為 `San Diego` 的航班，其 `price` 與 `discounted` 欄位將會被更新。如果不存在此類航班，則會建立一個新的航班，其屬性為第一個引數陣列與第二個引數陣列合併後的結果：

```php
$flight = Flight::updateOrCreate(
    ['departure' => 'Oakland', 'destination' => 'San Diego'],
    ['price' => 99, 'discounted' => 1]
);
```

當使用諸如 `firstOrCreate` 或 `updateOrCreate` 等方法時，您可能無法得知是建立了新的 Model 還是更新了現有的 Model。`wasRecentlyCreated` 屬性可用來指出該 Model 是否是在目前的生命週期中建立的：

```php
$flight = Flight::updateOrCreate(
    // ...
);

if ($flight->wasRecentlyCreated) {
    // New flight record was inserted...
}
```

<a name="mass-updates"></a>
#### 批次更新

也可以針對符合特定查詢條件的 Model 執行更新。在這個範例中，所有 `active` 為真且 `destination` 為 `San Diego` 的航班都會被標記為延誤：

```php
Flight::where('active', 1)
    ->where('destination', 'San Diego')
    ->update(['delayed' => 1]);
```

`update` 方法預期傳入一個包含欄位與值鍵值組的陣列，代表應該更新的欄位。`update` 方法會回傳受影響的資料列數。

> [!WARNING]
> 透過 Eloquent 發出批次更新時，更新後的 Model 將不會觸發 `saving`、`saved`、`updating` 與 `updated` 等 Model 事件。這是因為在發出批次更新時，實際上根本沒有從資料庫取出這些 Model。

<a name="examining-attribute-changes"></a>
#### 檢查屬性變更

Eloquent 提供了 `isDirty`、`isClean` 與 `wasChanged` 方法來檢查 Model 的內部狀態，並判斷其屬性自最初從資料庫取出以來發生了哪些變化。

`isDirty` 方法用來判斷自從 Model 取出以來是否有任何屬性被變更。您可以將特定的屬性名稱或屬性陣列傳遞給 `isDirty` 方法，以判斷是否有任何指定屬性變為「髒 (dirty)」。`isClean` 方法則用來判斷自 Model 取出以來屬性是否保持未變更。此方法同樣接受選填的屬性引數：

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

`wasChanged` 方法用來判斷在當前請求週期內最後一次儲存 Model 時，是否有任何屬性發生了變更。如有需要，您可以傳入屬性名稱以查看特定屬性是否已變更：

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

`getOriginal` 方法會回傳一個包含 Model 原始屬性的陣列，不論 Model 取出後發生了何種變更。如有需要，您可以傳入特定屬性名稱以取得該屬性的原始值：

```php
$user = User::find(1);

$user->name; // John
$user->email; // john@example.com

$user->name = 'Jack';
$user->name; // Jack

$user->getOriginal('name'); // John
$user->getOriginal(); // Array of original attributes...
```

`getChanges` 方法會回傳一個陣列，其中包含 Model 上次儲存時所變更的屬性；而 `getPrevious` 方法則會回傳一個陣列，其中包含 Model 上次儲存之前的原始屬性值：

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

你可以使用 `create` 方法，透過單一 PHP 陳述式來「儲存」一個新的 model。此方法會回傳已新增的 model 實例：

```php
use App\Models\Flight;

$flight = Flight::create([
    'name' => 'London to Paris',
]);
```

然而，在使用 `create` 方法之前，你需要在 model 類別上指定 `Fillable` 或 `Guarded` 屬性。這些屬性是必要的，因為預設情況下所有 Eloquent model 都會受到保護，以防止批量賦值弱點。

當使用者傳送了未預期的 HTTP 請求欄位，且該欄位更改了資料庫中你未預期的欄位時，就會發生批量賦值弱點。例如，惡意使用者可能會透過 HTTP 請求發送 `is_admin` 參數，接著該參數被傳遞至 model 的 `create` 方法，進而讓該使用者將自己提升為管理員。

因此，在開始之前，你應該定義哪些 model 屬性是可以被批量賦值的。你可以在 model 上使用 `Fillable` 屬性來達成此目的。例如，讓我們將 `Flight` model 的 `name` 屬性設為可批量賦值：

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

指定好哪些屬性可批量賦值後，你就可以使用 `create` 方法在資料庫中新增一筆記錄。`create` 方法會回傳新建立的 model 實例：

```php
$flight = Flight::create(['name' => 'London to Paris']);
```

如果你已經有一個 model 實例，你可以使用 `fill` 方法以屬性陣列來填入資料：

```php
$flight->fill(['name' => 'Amsterdam to Frankfurt']);
```


<a name="mass-assignment-json-columns"></a>
#### 批量賦值與 JSON 欄位

當賦值給 JSON 欄位時，每個欄位的可批量賦值鍵值必須在 model 的 `Fillable` 屬性中指定。出於安全考量，當使用 `Guarded` 屬性時，Laravel 不支援更新巢狀 JSON 屬性：

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

如果你希望讓所有屬性都可以被批量賦值，可以在 model 上使用 `Unguarded` 屬性。若你選擇取消 model 的保護，應特別小心，務必手動構建傳遞給 Eloquent 的 `fill`、`create` 與 `update` 方法的陣列：

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

預設情況下，執行批量賦值操作時，未包含在 `Fillable` 屬性中的屬性會被靜默捨棄。在正式環境中這是預期的行為；然而，在本地端開發期間，這可能會讓人感到困惑，不知道為何 model 的變更沒有生效。

如果你希望的話，可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填入不可賦值的屬性時拋出例外。通常，這個方法應該在應用程式的 `AppServiceProvider` 類別中的 `boot` 方法內呼叫：

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

Eloquent 的 `upsert` 方法可用於在單一不可部分完成 (atomic) 的操作中更新或建立記錄。該方法的第一個引數包含要新增或更新的值，而第二個引數列出了用於唯一識別關聯資料表中記錄的欄位。該方法的第三個也是最後一個引數是一個欄位陣列，表示若資料庫中已存在相符記錄時應更新的欄位。若 model 上已啟用時間戳記，`upsert` 方法將會自動設定 `created_at` 與 `updated_at` 時間戳記：

```php
Flight::upsert([
    ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
    ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
], uniqueBy: ['departure', 'destination'], update: ['price']);
```

> [!WARNING]
> 除了 SQL Server 之外的所有資料庫，都要求 `upsert` 方法第二個引數中的欄位必須具備「主鍵 (Primary)」或「唯一 (Unique)」索引。此外，MariaDB 與 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個引數，並一律使用資料表的主鍵和唯一索引來偵測現有記錄。

<a name="deleting-models"></a>
## 刪除 Model

要刪除 Model，您可以在 Model 實例上呼叫 `delete` 方法：

```php
use App\Models\Flight;

$flight = Flight::find(1);

$flight->delete();
```

如果您希望在資料庫交易中刪除 Model，可以使用 `deleteOrFail` 方法。如果在刪除過程中拋出例外，交易將會自動 rollback（回滾）：

```php
$flight->deleteOrFail();
```

<a name="deleting-an-existing-model-by-its-primary-key"></a>
#### 透過主鍵刪除現有 Model

在上面的範例中，我們在呼叫 `delete` 方法之前先從資料庫中取得 Model。然而，如果您知道 Model 的主鍵，可以透過呼叫 `destroy` 方法直接刪除 Model，而無需明確地取得它。除了接受單一主鍵外，`destroy` 方法還可以接受多個主鍵、主鍵陣列或主鍵[集合 (Collection)](/docs/{{version}}/collections)：

```php
Flight::destroy(1);

Flight::destroy(1, 2, 3);

Flight::destroy([1, 2, 3]);

Flight::destroy(collect([1, 2, 3]));
```

如果您正在使用[軟刪除 Model](#soft-deleting)，可以透過 `forceDestroy` 方法永久刪除 Model：

```php
Flight::forceDestroy(1);
```

> [!WARNING]
> `destroy` 方法會單獨載入每個 Model 並呼叫 `delete` 方法，以便為每個 Model 正確分派 `deleting` 和 `deleted` 事件。

<a name="deleting-models-using-queries"></a>
#### 使用查詢刪除 Model

當然，您可以建立 Eloquent 查詢來刪除符合查詢條件的所有 Model。在這個範例中，我們將刪除所有標記為未啟用的航班。如同批量更新一樣，批量刪除不會為被刪除的 Model 分派 Model 事件：

```php
$deleted = Flight::where('active', 0)->delete();
```

要刪除資料表中的所有 Model，您應該執行不帶任何條件的查詢：

```php
$deleted = Flight::query()->delete();
```

> [!WARNING]
> 透過 Eloquent 執行批量刪除語句時，不會為已刪除的 Model 分派 `deleting` 和 `deleted` Model 事件。這是因為在執行刪除語句時實際上從未取得這些 Model。

<a name="soft-deleting"></a>
### 軟刪除

除了實際從資料庫中移除記錄外，Eloquent 還可以「軟刪除」Model。當 Model 被軟刪除時，它們實際上並沒有從資料庫中移除。相反地，會在 Model 上設定一個 `deleted_at` 屬性，表示該 Model 被「刪除」的日期與時間。要為 Model 啟用軟刪除，請將 `Illuminate\Database\Eloquent\SoftDeletes` Trait 新增至 Model 中：

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
> `SoftDeletes` Trait 會自動將 `deleted_at` 屬性為您轉換為 `DateTime` / `Carbon` 實例。

您也應該將 `deleted_at` 欄位新增至資料庫表中。Laravel [Schema 建立器 (Schema Builder)](/docs/{{version}}/migrations) 包含一個輔助方法來建立此欄位：

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

現在，當您在 Model 上呼叫 `delete` 方法時，`deleted_at` 欄位將會設定為當前的日期與時間。然而，該 Model 的資料庫記錄將會保留在資料表中。查詢使用軟刪除的 Model 時，被軟刪除的 Model 將自動從所有查詢結果中排除。

要判斷給定的 Model 實例是否已被軟刪除，可以使用 `trashed` 方法：

```php
if ($flight->trashed()) {
    // ...
}
```

<a name="restoring-soft-deleted-models"></a>
#### 還原軟刪除的 Model

有時候您可能希望「取消刪除」軟刪除的 Model。要還原軟刪除的 Model，可以在 Model 實例上呼叫 `restore` 方法。`restore` 方法會將 Model 的 `deleted_at` 欄位設定為 `null`：

```php
$flight->restore();
```

您也可以在查詢中使用 `restore` 方法來還原多個 Model。同樣地，如同其他「批量」操作一樣，這不會為被還原的 Model 分派任何 Model 事件：

```php
Flight::withTrashed()
    ->where('airline_id', 1)
    ->restore();
```

`restore` 方法也可以在建立[關聯](/docs/{{version}}/eloquent-relationships)查詢時使用：

```php
$flight->history()->restore();
```

<a name="permanently-deleting-models"></a>
#### 永久刪除 Model

有時候您可能需要真正從資料庫中移除 Model。您可以使用 `forceDelete` 方法將軟刪除的 Model 從資料庫表中永久移除：

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

如上所述，軟刪除的 Model 將自動從查詢結果中排除。然而，您可以透過在查詢上呼叫 `withTrashed` 方法，強制將軟刪除的 Model 包含在查詢結果中：

```php
use App\Models\Flight;

$flights = Flight::withTrashed()
    ->where('account_id', 1)
    ->get();
```

`withTrashed` 方法也可以在建立[關聯](/docs/{{version}}/eloquent-relationships)查詢時呼叫：

```php
$flight->history()->withTrashed()->get();
```

<a name="retrieving-only-soft-deleted-models"></a>
#### 僅取得軟刪除的 Model

`onlyTrashed` 方法將**只**取得軟刪除的 Model：

```php
$flights = Flight::onlyTrashed()
    ->where('airline_id', 1)
    ->get();
```

<a name="pruning-models"></a>
## 修剪 Model (Pruning)

有時你可能會想定期刪除不再需要的 Model。為了達成這個目的，你可以將 `Illuminate\Database\Eloquent\Prunable` 或 `Illuminate\Database\Eloquent\MassPrunable` trait 加入到你想要定期修剪的 Model 中。將其中一個 trait 加入 Model 後，實作一個 `prunable` 方法，該方法會回傳一個 Eloquent 查詢建構器，用來解析出不再需要的 Model：

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

當將 Model 標記為 `Prunable` 時，你也可以在 Model 上定義一個 `pruning` 方法。該方法會在 Model 被刪除之前呼叫。在 Model 從資料庫中永久刪除之前，此方法對於刪除與該 Model 關聯的任何其他資源（例如儲存的檔案）非常有用：

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

在背後，`model:prune` 指令會自動偵測應用程式 `app/Models` 目錄內的「Prunable」Model。如果你的 Model 位於不同位置，可以使用 `--model` 選項來指定 Model 類別名稱：

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

你可以透過使用 `--pretend` 選項執行 `model:prune` 指令來測試你的 `prunable` 查詢。在模擬執行時，`model:prune` 指令只會回報如果實際執行該指令時會有多少筆記錄被修剪：

```shell
php artisan model:prune --pretend
```

> [!WARNING]
> 採用軟刪除的 Model 如果符合修剪查詢條件，將會被永久刪除 (`forceDelete`)。


<a name="mass-pruning"></a>
#### 批量修剪

當 Model 標記了 `Illuminate\Database\Eloquent\MassPrunable` trait 時，Model 會使用批量刪除查詢從資料庫中刪除。因此，`pruning` 方法不會被調用，也不會發送 `deleting` 和 `deleted` 的 Model 事件。這是因為 Model 在刪除前實際上從未被檢索出來，進而使修剪過程更加高效：

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

你可以使用 `replicate` 方法來建立一個尚未儲存的現有 Model 實例副本。當你擁有多個共享許多相同屬性的 Model 實例時，此方法特別實用：

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

若要排除一個或多個屬性不被複製到新的 Model 中，你可以傳遞一個陣列給 `replicate` 方法：

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
## 查詢作用域 (Query Scopes)


<a name="global-scopes"></a>
### 全域作用域

全域作用域允許你為給定 Model 的所有查詢添加限制條件。Laravel 自帶的[軟刪除](#soft-deleting)功能就是利用全域作用域，從資料庫中僅取得「未刪除」的 Model。編寫你自己的全域作用域，可以提供一個方便、簡單的方式，確保特定 Model 的每個查詢都能套用特定的限制條件。


<a name="generating-scopes"></a>
#### 生成作用域

若要生成新的全域作用域，你可以執行 `make:scope` Artisan 指令，該指令會將產生的作用域放置在應用程式的 `app/Models/Scopes` 目錄中：

```shell
php artisan make:scope AncientScope
```


<a name="writing-global-scopes"></a>
#### 編寫全域作用域

編寫全域作用域非常簡單。首先，使用 `make:scope` 指令生成一個實作 `Illuminate\Database\Eloquent\Scope` 介面的類別。`Scope` 介面要求你實作一個方法：`apply`。`apply` 方法可以依需求向查詢添加 `where` 條件或其他類型的子句：

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
> 如果你的全域作用域正在向查詢的 select 子句添加欄位，你應該使用 `addSelect` 方法而不是 `select`。這可以防止意外覆蓋查詢現有的 select 子句。


<a name="applying-global-scopes"></a>
#### 套用全域作用域

若要將全域作用域指派給 Model，你只需在 Model 上放置 `ScopedBy` 屬性即可：

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

或者，你也可以透過覆寫 Model 的 `booted` 方法並調用 Model 的 `addGlobalScope` 方法來手動註冊全域作用域。`addGlobalScope` 方法接受你的作用域實例作為其唯一的引數：

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

在上述範例將作用域添加到 `App\Models\User` Model 後，調用 `User::all()` 方法將會執行以下 SQL 查詢：

```sql
select * from `users` where `created_at` < 0021-02-18 00:00:00
```


<a name="anonymous-global-scopes"></a>
#### 匿名全域作用域

Eloquent 還允許你使用閉包來定義全域作用域，這對於不需要獨立類別的簡單作用域特別有用。使用閉包定義全域作用域時，你應該提供一個自訂的作用域名稱作為 `addGlobalScope` 方法的第一個引數：

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
#### 移除全域作用域

如果你想要為特定查詢移除全域作用域，可以使用 `withoutGlobalScope` 方法。該方法接受全域作用域的類別名稱作為其唯一引數：

```php
User::withoutGlobalScope(AncientScope::class)->get();
```

或者，如果你是使用閉包定義全域作用域，則應傳入指派給該全域作用域的字串名稱：

```php
User::withoutGlobalScope('ancient')->get();
```

如果你想要移除查詢中的數個甚至所有的全域作用域，可以使用 `withoutGlobalScopes` 與 `withoutGlobalScopesExcept` 方法：

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
### 局部作用域

局部作用域允許你定義一組通用的查詢條件，以便在整個應用程式中輕鬆重複使用。例如，你可能需要經常取得所有被歸類為「熱門」的使用者。若要定義作用域，請在 Eloquent 方法上加上 `Scope` 屬性。

作用域應始終返回相同的查詢建構器實例或 `void`：

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
#### 使用局部作用域

定義作用域後，你可以在查詢 Model 時調用作用域方法。你甚至可以鏈結調用各種不同的作用域：

```php
use App\Models\User;

$users = User::popular()->active()->orderBy('created_at')->get();
```

透過 `or` 查詢運算子組合多個 Eloquent Model 作用域時，可能需要使用閉包以實現正確的[邏輯分組](/docs/{{version}}/queries#logical-grouping)：

```php
$users = User::popular()->orWhere(function (Builder $query) {
    $query->active();
})->get();
```

然而，由於這樣寫可能較為繁瑣，Laravel 提供了一個「高階 (Higher Order)」的 `orWhere` 方法，允許你在不使用閉包的情況下流暢地鏈結作用域：

```php
$users = User::popular()->orWhere->active()->get();
```


<a name="dynamic-scopes"></a>
#### 動態作用域

有時你可能會希望定義一個能接受參數的作用域。首先，只需將額外的參數添加到作用域方法的簽名中。作用域參數應定義在 `$query` 參數之後：

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

將預期的引數添加到作用域方法的簽名後，你便可以在調用作用域時傳入這些引數：

```php
$users = User::ofType('admin')->get();
```

使用屬性標註的作用域方法應為 `protected`。當在 Model 類別內部調用屬性標註的作用域時，請透過查詢建構器實例進行調用（例如 `static::query()->ofType('admin')`），以確保調用能正確經由 Eloquent 的作用域機制進行處理。

<a name="pending-attributes"></a>
### 待處理屬性

如果您想使用作用域來建立擁有與該作用域約束條件相同屬性的 Model，您可以在建立作用域查詢時使用 `withAttributes` 方法：

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

`withAttributes` 方法會使用給定的屬性向查詢加入 `where` 條件，並且還會將這些給定的屬性加入到透過該作用域建立的任何 Model 中：

```php
$draft = Post::draft()->create(['title' => 'In Progress']);

$draft->hidden; // true
```

若要指示 `withAttributes` 方法不要向查詢加入 `where` 條件，您可以將 `asConditions` 引數設定為 `false`：

```php
$query->withAttributes([
    'hidden' => true,
], asConditions: false);
```

<a name="comparing-models"></a>
## 比較 Model

有時候您可能需要判斷兩個 Model 是否「相同」。可以使用 `is` 和 `isNot` 方法快速驗證兩個 Model 是否具有相同的主鍵、資料表以及資料庫連線：

```php
if ($post->is($anotherPost)) {
    // ...
}

if ($post->isNot($anotherPost)) {
    // ...
}
```

在使用 `belongsTo`、`hasOne`、`morphTo` 和 `morphOne` [關聯](/docs/{{version}}/eloquent-relationships)時，也可以使用 `is` 和 `isNot` 方法。當您想要比較關聯的 Model 而不想發送查詢來取得該 Model 時，這個方法特別實用：

```php
if ($post->author()->is($user)) {
    // ...
}
```

<a name="events"></a>
## 事件 (Events)

> [!NOTE]
> 想要將 Eloquent 事件直接廣播到客戶端應用程式嗎？請參考 Laravel 的[模型事件廣播 (Model Event Broadcasting)](/docs/{{version}}/broadcasting#model-broadcasting)。

Eloquent 模型會發送多種事件，讓你可以掛鉤至模型生命週期中的下列時刻：`retrieved`、`creating`、`created`、`updating`、`updated`、`saving`、`saved`、`deleting`、`deleted`、`trashed`、`forceDeleting`、`forceDeleted`、`restoring`、`restored` 以及 `replicating`。

當從資料庫取得現有的模型時，會發送 `retrieved` 事件。當新模型第一次被儲存時，會發送 `creating` 與 `created` 事件。當現有模型被修改並呼叫 `save` 方法時，會發送 `updating` / `updated` 事件。當模型被建立或更新時，無論模型的屬性是否有變更，都會發送 `saving` / `saved` 事件。以 `-ing` 結尾的事件名稱會在對模型的任何變更被持久化之前發送，而以 `-ed` 結尾的事件則會在對模型的變更被持久化之後發送。

若要開始監聽模型事件，請在你的 Eloquent 模型上定義一個 `$dispatchesEvents` 屬性。此屬性會將 Eloquent 模型生命週期的各個時間點映射到你自訂的[事件類別](/docs/{{version}}/events)。每個模型事件類別都應該在其建構函式中接收受影響模型的實例：

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

在定義並映射好 Eloquent 事件後，你可以使用[事件監聽器](/docs/{{version}}/events#defining-listeners)來處理這些事件。

> [!WARNING]
> 當透過 Eloquent 發出批量更新或刪除查詢時，受影響的模型將不會發送 `saved`、`updated`、`deleting` 和 `deleted` 模型事件。這是因為在執行批量更新或刪除時，實際上從未擷取過這些模型。

<a name="events-using-closures"></a>
### 使用閉包

除了使用自訂事件類別外，你也可以註冊在發送各種模型事件時執行的閉包。通常，你應該在模型的 `booted` 方法中註冊這些閉包：

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

如果需要，你可以在註冊模型事件時使用[可排入佇列的匿名事件監聽器](/docs/{{version}}/events#queueable-anonymous-event-listeners)。這將指示 Laravel 使用應用程式的[佇列](/docs/{{version}}/queues)在背景執行模型事件監聽器：

```php
use function Illuminate\Events\queueable;

static::created(queueable(function (User $user) {
    // ...
}));
```

<a name="observers"></a>
### 觀察者 (Observers)

<a name="defining-observers"></a>
#### 定義觀察者

如果你正在監聽特定模型上的許多事件，可以使用觀察者將所有監聽器分組到單一類別中。觀察者類別的方法名稱對應了你想要監聽的 Eloquent 事件。這些方法都將受影響的模型作為其唯一的引數。`make:observer` Artisan 指令是建立新觀察者類別最簡單的方法：

```shell
php artisan make:observer UserObserver --model=User
```

此指令會將新的觀察者放置在你的 `app/Observers` 目錄中。如果該目錄不存在，Artisan 會為你建立。全新的觀察者看起來會像這樣：

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

若要註冊觀察者，你可以在對應的模型上放置 `ObservedBy` 屬性：

```php
use App\Observers\UserObserver;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy([UserObserver::class])]
class User extends Authenticatable
{
    //
}
```

或者，你也可以透過在想要觀察的模型上呼叫 `observe` 方法來手動註冊觀察者。你可以在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中註冊觀察者：

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
> 觀察者還可以監聽其他事件，例如 `saving` 和 `retrieved`。這些事件在[事件](#events)說明文件中皆有描述。

<a name="observers-and-database-transactions"></a>
#### 觀察者與資料庫交易

當在資料庫交易中建立模型時，你可能會希望指示觀察者僅在資料庫交易認可 (Commit) 後才執行其事件處理函式。你可以透過在觀察者上實作 `ShouldHandleEventsAfterCommit` 介面來達成此目的。如果當前沒有進行中的資料庫交易，事件處理函式將會立即執行：

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
### 靜音事件 (Muting Events)

你偶爾可能需要暫時「靜音」模型觸發的所有事件。你可以使用 `withoutEvents` 方法來做到這一點。`withoutEvents` 方法接受一個閉包作為其唯一的引數。在此閉包內執行的任何程式碼都不會發送模型事件，且該閉包返回的任何值都將由 `withoutEvents` 方法返回：

```php
use App\Models\User;

$user = User::withoutEvents(function () {
    User::findOrFail(1)->delete();

    return User::find(2);
});
```

<a name="saving-a-single-model-without-events"></a>
#### 在不觸發事件的情況下儲存單一 Model

有時你可能會希望在不發送任何事件的情況下「儲存」特定的模型。你可以使用 `saveQuietly` 方法來完成此操作：

```php
$user = User::findOrFail(1);

$user->name = 'Victoria Faith';

$user->saveQuietly();
```

你也可以在不發送任何事件的情況下對特定模型進行「更新」、「刪除」、「軟刪除」、「還原」以及「複製」操作：

```php
$user->deleteQuietly();
$user->forceDeleteQuietly();
$user->restoreQuietly();
```