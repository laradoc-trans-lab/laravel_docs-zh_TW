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
    - [Collection 集合](#collections)
    - [分批處理結果](#chunking-results)
    - [使用 Lazy Collection 集合分批處理](#chunking-using-lazy-collections)
    - [游標](#cursors)
    - [進階子查詢](#advanced-subqueries)
- [擷取單一 Model / 彙總](#retrieving-single-models)
    - [擷取或建立 Model](#retrieving-or-creating-models)
    - [擷取彙總](#retrieving-aggregates)
- [插入與更新 Model](#inserting-and-updating-models)
    - [插入](#inserts)
    - [更新](#updates)
    - [大量指派](#mass-assignment)
    - [更新或插入](#upserts)
- [刪除 Model](#deleting-models)
    - [軟刪除](#soft-deleting)
    - [查詢軟刪除的 Model](#querying-soft-deleted-models)
- [修剪 Model](#pruning-models)
- [複製 Model](#replicating-models)
- [查詢 Scope](#query-scopes)
    - [全域 Scope](#global-scopes)
    - [本地 Scope](#local-scopes)
    - [待定屬性](#pending-attributes)
- [比較 Model](#comparing-models)
- [事件](#events)
    - [使用 Closure](#events-using-closures)
    - [Observer 觀察者](#observers)
    - [靜音事件](#muting-events)

<a name="introduction"></a>
## 簡介

Laravel 包含了 Eloquent，這是一個物件關聯映射 (ORM)，讓您能愉快地與資料庫互動。使用 Eloquent 時，每個資料庫資料表都有一個對應的「Model」，用於與該資料表互動。除了從資料庫資料表擷取記錄之外，Eloquent model 還允許您插入、更新和刪除資料表中的記錄。

> [!NOTE]  
> 在開始之前，請務必在應用程式的 `config/database.php` 設定檔中配置資料庫連線。有關配置資料庫的更多資訊，請查閱[資料庫設定文件](/docs/{{version}}/database#configuration)。

#### Laravel Bootcamp

如果您是 Laravel 的新手，請隨時參與 [Laravel Bootcamp](https://bootcamp.laravel.com)。Laravel Bootcamp 將引導您使用 Eloquent 建立您的第一個 Laravel 應用程式。這是了解 Laravel 和 Eloquent 提供的一切功能的絕佳方式。

<a name="generating-model-classes"></a>
## 產生 Model 類別

首先，讓我們先建立一個 Eloquent model。Model 通常位於 `app\Models` 目錄中，並擴展 `Illuminate\Database\Eloquent\Model` 類別。您可以使用 `make:model` [Artisan 命令](/docs/{{version}}/artisan)來產生一個新的 model：

```shell
php artisan make:model Flight
```

如果您想在產生 model 時同時產生[資料庫遷移](/docs/{{version}}/migrations)，您可以使用 `--migration` 或 `-m` 選項：

```shell
php artisan make:model Flight --migration
```

在產生 model 時，您還可以產生各種其他類型的類別，例如工廠 (factories)、填充器 (seeders)、策略 (policies)、控制器 (controllers) 和表單請求 (form requests)。此外，這些選項可以組合使用以一次建立多個類別：

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

有時候僅透過瀏覽 model 的程式碼，很難確定 model 的所有可用屬性和關聯。您可以嘗試使用 `model:show` Artisan 命令，它提供了 model 所有屬性和關聯的便捷概覽：

```shell
php artisan model:show Flight
```

<a name="eloquent-model-conventions"></a>
## Eloquent Model 慣例

由 `make:model` 命令產生的 Model 將會放置於 `app/Models` 目錄。讓我們檢查一個基本的 Model 類別，並討論一些 Eloquent 的關鍵慣例：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        // ...
    }


<a name="table-names"></a>
### 資料表名稱

瀏覽完上述範例後，您可能已經注意到我們並未告知 Eloquent 哪一個資料庫資料表對應到我們的 `Flight` Model。依照慣例，類別的「蛇形命名 (snake case)」複數形式將作為資料表名稱，除非明確指定其他名稱。因此，在此情況下，Eloquent 會假設 `Flight` Model 將記錄儲存於 `flights` 資料表，而 `AirTrafficController` Model 則會將記錄儲存於 `air_traffic_controllers` 資料表。

如果您的 Model 對應的資料庫資料表不符合此慣例，您可以透過在 Model 上定義一個 `table` 屬性來手動指定 Model 的資料表名稱：

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


<a name="primary-keys"></a>
### 主鍵

Eloquent 也會假設每個 Model 對應的資料庫資料表都具有一個名為 `id` 的主鍵欄位。如有必要，您可以在 Model 上定義一個受保護的 `$primaryKey` 屬性，以指定作為 Model 主鍵的不同欄位：

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

此外，Eloquent 假設主鍵是一個遞增的整數值，這表示 Eloquent 會自動將主鍵轉換為整數。如果您希望使用非遞增或非數字的主鍵，您必須在 Model 上定義一個設定為 `false` 的公開 `$incrementing` 屬性：

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

如果您 Model 的主鍵不是整數，您應該在 Model 上定義一個受保護的 `$keyType` 屬性。此屬性的值應為 `string`：

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


<a name="composite-primary-keys"></a>
#### 「複合」主鍵

Eloquent 要求每個 Model 都必須至少有一個唯一的「ID」作為其主鍵。Eloquent Model 不支援「複合」主鍵。然而，除了資料表的唯一識別主鍵之外，您可以自由地為資料庫資料表新增額外的多欄唯一索引。


<a name="uuid-and-ulid-keys"></a>
### UUID 與 ULID 鍵

您可以選擇使用 UUID 而非自動遞增的整數作為 Eloquent Model 的主鍵。UUID 是通用唯一的英數字識別符，長度為 36 個字元。

如果您希望 Model 使用 UUID 鍵而非自動遞增的整數鍵，您可以在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUuids` trait。當然，您應該確保 Model 具有 [UUID 等效主鍵欄位](/docs/{{version}}/migrations#column-method-uuid)：

    use Illuminate\Database\Eloquent\Concerns\HasUuids;
    use Illuminate\Database\Eloquent\Model;

    class Article extends Model
    {
        use HasUuids;

        // ...
    }

    $article = Article::create(['title' => 'Traveling to Europe']);

    $article->id; // "8f8e8478-9035-4d23-b9a7-62f4d2612ce5"

預設情況下，`HasUuids` trait 將為您的 Model 產生 [「有序」UUID](/docs/{{version}}/strings#method-str-ordered-uuid)。這些 UUID 對於索引資料庫儲存更有效率，因為它們可以按字典順序排序。

您可以透過在 Model 上定義 `newUniqueId` 方法來覆寫特定 Model 的 UUID 產生過程。此外，您可以透過在 Model 上定義 `uniqueIds` 方法來指定哪些欄位應接收 UUID：

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

如果您願意，您可以選擇使用 ULID 而非 UUID。ULID 類似於 UUID；然而，它們的長度只有 26 個字元。與有序 UUID 類似，ULID 也可以按字典順序排序，以實現高效的資料庫索引。要使用 ULID，您應該在 Model 上使用 `Illuminate\Database\Eloquent\Concerns\HasUlids` trait。您還應該確保 Model 具有 [ULID 等效主鍵欄位](/docs/{{version}}/migrations#column-method-ulid)：

    use Illuminate\Database\Eloquent\Concerns\HasUlids;
    use Illuminate\Database\Eloquent\Model;

    class Article extends Model
    {
        use HasUlids;

        // ...
    }

    $article = Article::create(['title' => 'Traveling to Asia']);

    $article->id; // "01gd4d3tgrrfqeda94gdbtdk5c"


<a name="timestamps"></a>
### 時間戳記

預設情況下，Eloquent 期望您的 Model 對應的資料庫資料表上存在 `created_at` 和 `updated_at` 欄位。當 Model 被建立或更新時，Eloquent 會自動設定這些欄位的值。如果您不希望這些欄位由 Eloquent 自動管理，您應該在 Model 上定義一個設定為 `false` 的 `$timestamps` 屬性：

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

如果您需要自訂 Model 時間戳記的格式，請在 Model 上設定 `$dateFormat` 屬性。此屬性決定了日期屬性在資料庫中的儲存方式，以及當 Model 被序列化為陣列或 JSON 時的格式：

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

如果您需要自訂用於儲存時間戳記的欄位名稱，您可以在 Model 上定義 `CREATED_AT` 和 `UPDATED_AT` 常數：

    <?php

    class Flight extends Model
    {
        const CREATED_AT = 'creation_date';
        const UPDATED_AT = 'updated_date';
    }

如果您想在執行 Model 操作時，不修改 Model 的 `updated_at` 時間戳記，您可以在提供給 `withoutTimestamps` 方法的閉包中操作該 Model：

    Model::withoutTimestamps(fn () => $post->increment('reads'));


<a name="database-connections"></a>
### 資料庫連線

預設情況下，所有 Eloquent Model 都將使用應用程式配置的預設資料庫連線。如果您想指定與特定 Model 互動時應使用的不同連線，您應該在 Model 上定義一個 `$connection` 屬性：

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

<a name="default-attribute-values"></a>
### 預設屬性值

預設情況下，新實例化的 Model 實例將不包含任何屬性值。如果您想為 Model 的某些屬性定義預設值，可以在 Model 上定義一個 `$attributes` 屬性。放置在 `$attributes` 陣列中的屬性值應該是其原始的、「可儲存」格式，就像它們剛從資料庫中讀取出來一樣：

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


<a name="configuring-eloquent-strictness"></a>
### 設定 Eloquent 的嚴謹度

Laravel 提供數種方法，讓您可以設定 Eloquent 在各種情況下的行為與「嚴謹度」。

首先，`preventLazyLoading` 方法接受一個選用的布林引數，該引數指示是否應該阻止延遲載入 (lazy loading)。舉例來說，您可能希望僅在非正式環境中禁用延遲載入，以便即使在正式環境程式碼中意外出現了延遲載入的關聯，您的正式環境仍能正常運作。通常，此方法應在應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫：

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

此外，您可以指示 Laravel 在嘗試填充不可填充 (unfillable) 屬性時拋出例外，方法是呼叫 `preventSilentlyDiscardingAttributes` 方法。這有助於在本地開發期間，防止在嘗試設定未添加到 Model 的 `fillable` 陣列中的屬性時，發生意外錯誤：

```php
Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
```

<a name="retrieving-models"></a>
## 擷取 Model

建立 Model 及其[關聯的資料表](/docs/{{version}}/migrations#generating-migrations)後，您就可以開始從資料庫擷取資料了。您可以將每個 Eloquent Model 視為一個強大的[查詢產生器](/docs/{{version}}/queries)，讓您可以流暢地查詢與 Model 關聯的資料表。Model 的 `all` 方法將會從 Model 關聯的資料表中擷取所有記錄：

    use App\Models\Flight;

    foreach (Flight::all() as $flight) {
        echo $flight->name;
    }


<a name="building-queries"></a>
#### 建立查詢

Eloquent 的 `all` 方法會傳回 Model 資料表中的所有結果。然而，由於每個 Eloquent Model 都作為一個[查詢產生器](/docs/{{version}}/queries)，您可以為查詢新增額外條件，然後呼叫 `get` 方法來擷取結果：

    $flights = Flight::where('active', 1)
        ->orderBy('name')
        ->take(10)
        ->get();

> [!NOTE]
> 由於 Eloquent Model 是查詢產生器，您應該檢閱 Laravel [查詢產生器](/docs/{{version}}/queries)提供的所有方法。在編寫 Eloquent 查詢時，您可以使用這些方法中的任何一個。


<a name="refreshing-models"></a>
#### 重新整理 Model

如果您已經有一個從資料庫擷取的 Eloquent Model 實例，您可以使用 `fresh` 與 `refresh` 方法來「重新整理 (refresh)」該 Model。`fresh` 方法將會從資料庫中重新擷取 Model。現有的 Model 實例將不受影響：

    $flight = Flight::where('number', 'FR 900')->first();

    $freshFlight = $flight->fresh();

`refresh` 方法將會使用資料庫中的新資料來重新載入現有 Model。此外，其所有已載入的關聯也將會重新整理：

    $flight = Flight::where('number', 'FR 900')->first();

    $flight->number = 'FR 456';

    $flight->refresh();

    $flight->number; // "FR 900"


<a name="collections"></a>
### Collection 集合

如我們所見，Eloquent 的 `all` 和 `get` 等方法會從資料庫中擷取多個記錄。然而，這些方法不會傳回純 PHP 陣列。相反地，它會傳回一個 `Illuminate\Database\Eloquent\Collection` 實例。

Eloquent `Collection` 類別擴充了 Laravel 的基礎 `Illuminate\Support\Collection` 類別，它提供了[多種實用方法](/docs/{{version}}/collections#available-methods)來處理資料集合。例如，`reject` 方法可以用來根據呼叫閉包的結果從集合中移除 Model：

```php
$flights = Flight::where('destination', 'Paris')->get();

$flights = $flights->reject(function (Flight $flight) {
    return $flight->cancelled;
});
```

除了 Laravel 基礎集合類別提供的方法外，Eloquent 集合類別還提供了[一些額外方法](/docs/{{version}}/eloquent-collections#available-methods)，專門用於處理 Eloquent Model 集合。

由於 Laravel 的所有集合都實作了 PHP 的可疊代介面，您可以像遍歷陣列一樣遍歷集合：

```php
foreach ($flights as $flight) {
    echo $flight->name;
}
```


<a name="chunking-results"></a>
### 分批處理結果

如果您的應用程式嘗試透過 `all` 或 `get` 方法載入數萬條 Eloquent 記錄，可能會導致記憶體不足。您可以使用 `chunk` 方法來更有效率地處理大量 Model，而不是使用這些方法。

`chunk` 方法會擷取 Eloquent Model 的子集，並將它們傳遞給閉包進行處理。由於每次只擷取當前批次的 Eloquent Model，因此在使用大量 Model 時，`chunk` 方法將會大幅減少記憶體使用量：

```php
use App\Models\Flight;
use Illuminate\Database\Eloquent\Collection;

Flight::chunk(200, function (Collection $flights) {
    foreach ($flights as $flight) {
        // ...
    }
});
```

傳遞給 `chunk` 方法的第一個參數是您希望每個「批次」接收的記錄數量。作為第二個參數傳遞的閉包將會針對從資料庫中擷取的每個批次被呼叫。每次擷取傳遞給閉包的記錄批次時，都將會執行一次資料庫查詢。

如果您根據某個欄位過濾 `chunk` 方法的結果，並且在遍歷結果時同時更新該欄位，則應該使用 `chunkById` 方法。在這些情況下使用 `chunk` 方法可能會導致意外且不一致的結果。在內部，`chunkById` 方法總是會擷取 `id` 欄位大於前一批次最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->chunkById(200, function (Collection $flights) {
        $flights->each->update(['departed' => false]);
    }, column: 'id');
```

由於 `chunkById` 和 `lazyById` 方法會為正在執行的查詢新增自己的「where」條件，您通常應該在閉包中將自己的條件進行[邏輯分組](/docs/{{version}}/queries#logical-grouping)：

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
### 使用 Lazy Collection 集合分批處理

`lazy` 方法的運作方式與 [the `chunk` method](#chunking-results) 相似，它在幕後以批次方式執行查詢。然而，`lazy` 方法不會將每個批次直接傳遞給回呼，而是傳回扁平化的 Eloquent Model [`LazyCollection`](/docs/{{version}}/collections#lazy-collections)，這讓您可以像處理單一串流一樣處理結果：

```php
use App\Models\Flight;

foreach (Flight::lazy() as $flight) {
    // ...
}
```

如果您根據某個欄位過濾 `lazy` 方法的結果，並且在遍歷結果時同時更新該欄位，則應該使用 `lazyById` 方法。在內部，`lazyById` 方法總是會擷取 `id` 欄位大於前一批次最後一個 Model 的 Model：

```php
Flight::where('departed', true)
    ->lazyById(200, column: 'id')
    ->each->update(['departed' => false]);
```

您可以使用 `lazyByIdDesc` 方法根據 `id` 的降序過濾結果。

<a name="cursors"></a>
### 游標

與`lazy`方法類似，`cursor`方法可用於在迭代數萬個 Eloquent model 紀錄時，顯著降低應用程式的記憶體消耗。

`cursor`方法只會執行單一資料庫查詢；然而，個別的 Eloquent model 在實際迭代前並不會被實體化 (hydrated)。因此，在迭代游標時，任一時刻記憶體中只會保留一個 Eloquent model。

> [!WARNING]  
> 由於`cursor`方法一次只在記憶體中保留一個 Eloquent model，因此它無法預先載入 (eager load) 關聯。如果您需要預先載入關聯，請考慮改用[ `lazy` 方法](#chunking-using-lazy-collections)。

在內部，`cursor`方法使用 PHP [generators](https://www.php.net/manual/en/language.generators.overview.php) 來實作此功能：

```php
use App\Models\Flight;

foreach (Flight::where('destination', 'Zurich')->cursor() as $flight) {
    // ...
}
```

`cursor`方法會回傳一個`Illuminate\Support\LazyCollection`實例。[Lazy collections](/docs/{{version}}/collections#lazy-collections) 讓您可以使用典型 Laravel collection 中的許多 collection 方法，同時一次只載入一個 model 到記憶體中：

```php
use App\Models\User;

$users = User::cursor()->filter(function (User $user) {
    return $user->id > 500;
});

foreach ($users as $user) {
    echo $user->id;
}
```

儘管`cursor`方法比一般查詢使用更少的記憶體 (一次只在記憶體中保留一個 Eloquent model)，它最終仍然會耗盡記憶體。這是由於 [PHP 的 PDO 驅動程式會在內部緩存所有原始查詢結果](https://www.php.net/manual/en/mysqlinfo.concepts.buffering.php)。如果您正在處理大量 Eloquent 紀錄，請考慮改用[ `lazy` 方法](#chunking-using-lazy-collections)。

<a name="advanced-subqueries"></a>
### 進階子查詢

<a name="subquery-selects"></a>
#### 子查詢選取

Eloquent 也提供了進階的子查詢支援，讓您可以透過單一查詢從相關資料表中提取資訊。舉例來說，假設我們有一個航班的`destinations`資料表和一個`flights`到目的地的資料表。`flights`資料表包含一個`arrived_at`欄位，表示航班抵達目的地的時間。

利用查詢建構器 (query builder) 的`select`和`addSelect`方法所提供的子查詢功能，我們可以使用單一查詢來選取所有`destinations`以及最近抵達該目的地的航班名稱：

    use App\Models\Destination;
    use App\Models\Flight;

    return Destination::addSelect(['last_flight' => Flight::select('name')
        ->whereColumn('destination_id', 'destinations.id')
        ->orderByDesc('arrived_at')
        ->limit(1)
    ])->get();

<a name="subquery-ordering"></a>
#### 子查詢排序

此外，查詢建構器 (query builder) 的`orderBy`功能也支援子查詢。沿用我們的航班範例，我們可以利用此功能根據上次航班抵達目的地的時間來排序所有目的地。同樣地，這可以透過執行單一資料庫查詢來完成：

    return Destination::orderByDesc(
        Flight::select('arrived_at')
            ->whereColumn('destination_id', 'destinations.id')
            ->orderByDesc('arrived_at')
            ->limit(1)
    )->get();

<a name="retrieving-single-models"></a>
## 擷取單一 Model / 彙總

除了擷取符合給定查詢的所有記錄外，您也可以使用 `find`、`first` 或 `firstWhere` 方法來擷取單一記錄。這些方法不會回傳 Model 集合 (Collection)，而是回傳單一 Model 實例：

    use App\Models\Flight;

    // 依據主鍵擷取 Model...
    $flight = Flight::find(1);

    // 擷取符合查詢限制的第一個 Model...
    $flight = Flight::where('active', 1)->first();

    // 擷取符合查詢限制的第一個 Model 的替代方法...
    $flight = Flight::firstWhere('active', 1);

有時，如果找不到任何結果，您可能希望執行其他操作。`findOr` 和 `firstOr` 方法將回傳單一 Model 實例，或者，如果找不到結果，則執行給定的閉包。閉包回傳的值將被視為方法的結果：

    $flight = Flight::findOr(1, function () {
        // ...
    });

    $flight = Flight::where('legs', '>', 3)->firstOr(function () {
        // ...
    });


<a name="not-found-exceptions"></a>
#### 找不到異常

有時，如果找不到 Model，您可能希望拋出異常。這在路由或控制器中特別有用。`findOrFail` 和 `firstOrFail` 方法將擷取查詢的第一個結果；然而，如果找不到結果，將拋出 `Illuminate\Database\Eloquent\ModelNotFoundException`：

    $flight = Flight::findOrFail(1);

    $flight = Flight::where('legs', '>', 3)->firstOrFail();

如果未捕捉到 `ModelNotFoundException`，則會自動向客戶端發送 404 HTTP 回應：

    use App\Models\Flight;

    Route::get('/api/flights/{id}', function (string $id) {
        return Flight::findOrFail($id);
    });


<a name="retrieving-or-creating-models"></a>
### 擷取或建立 Model

`firstOrCreate` 方法將嘗試使用給定的欄位/值對來定位資料庫記錄。如果 Model 在資料庫中找不到，則將插入一條記錄，其屬性是將第一個陣列引數與可選的第二個陣列引數合併後的結果：

`firstOrNew` 方法，與 `firstOrCreate` 類似，將嘗試在資料庫中定位與給定屬性相符的記錄。然而，如果找不到 Model，則將回傳一個新的 Model 實例。請注意，`firstOrNew` 回傳的 Model 尚未持久化到資料庫。您需要手動呼叫 `save` 方法來持久化它：

    use App\Models\Flight;

    // 依據名稱擷取 flight，如果不存在則建立...
    $flight = Flight::firstOrCreate([
        'name' => 'London to Paris'
    ]);

    // 依據名稱擷取 flight，如果不存在則使用名稱、delayed 和 arrival_time 屬性建立它...
    $flight = Flight::firstOrCreate(
        ['name' => 'London to Paris'],
        ['delayed' => 1, 'arrival_time' => '11:30']
    );

    // 依據名稱擷取 flight，如果不存在則實例化一個新的 Flight 實例...
    $flight = Flight::firstOrNew([
        'name' => 'London to Paris'
    ]);

    // 依據名稱擷取 flight，如果不存在則使用名稱、delayed 和 arrival_time 屬性實例化...
    $flight = Flight::firstOrNew(
        ['name' => 'Tokyo to Sydney'],
        ['delayed' => 1, 'arrival_time' => '11:30']
    );


<a name="retrieving-aggregates"></a>
### 擷取彙總

在使用 Eloquent Model 時，您也可以使用 Laravel [查詢產生器](/docs/{{version}}/queries)提供的 `count`、`sum`、`max` 和其他[彙總方法](/docs/{{version}}/queries#aggregates)。正如您可能預期的那樣，這些方法回傳一個純量值，而不是 Eloquent Model 實例：

    $count = Flight::where('active', 1)->count();

    $max = Flight::where('active', 1)->max('price');

<a name="inserting-and-updating-models"></a>
## 插入與更新 Model

<a name="inserts"></a>
### 插入

當然，使用 Eloquent 時，我們不僅需要從資料庫中擷取 Model。我們還需要插入新記錄。值得慶幸的是，Eloquent 讓這變得簡單。要將新記錄插入資料庫，您應該實例化一個新的 Model 實例並設定 Model 上的屬性。然後，呼叫 Model 實例上的 `save` 方法：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

在此範例中，我們將傳入 HTTP 請求中的 `name` 欄位指派給 `App\Models\Flight` Model 實例的 `name` 屬性。當我們呼叫 `save` 方法時，記錄將會被插入資料庫。Model 的 `created_at` 和 `updated_at` 時間戳記在 `save` 方法被呼叫時會自動設定，因此無需手動設定。

或者，您可以使用 `create` 方法，透過單一 PHP 語句來「儲存」新的 Model。插入的 Model 實例將由 `create` 方法返回給您：

    use App\Models\Flight;

    $flight = Flight::create([
        'name' => 'London to Paris',
    ]);

然而，在使用 `create` 方法之前，您需要在庫的 Model 類別上指定 `fillable` 或 `guarded` 屬性。這些屬性是必需的，因為所有 Eloquent Model 預設都受到大量指派漏洞的保護。要了解更多關於大量指派的資訊，請參閱[大量指派文件](#mass-assignment)。

<a name="updates"></a>
### 更新

`save` 方法也可用於更新資料庫中已存在的 Model。要更新 Model，您應該先擷取它並設定您希望更新的任何屬性。然後，您應該呼叫 Model 的 `save` 方法。同樣地，`updated_at` 時間戳記將會自動更新，因此無需手動設定其值：

    use App\Models\Flight;

    $flight = Flight::find(1);

    $flight->name = 'Paris to London';

    $flight->save();

偶爾，您可能需要更新一個現有的 Model，或者在沒有匹配 Model 存在的情況下建立一個新的 Model。與 `firstOrCreate` 方法一樣，`updateOrCreate` 方法會持久化 Model，因此無需手動呼叫 `save` 方法。

在下面的範例中，如果存在一個 `departure` 為 `Oakland` 且 `destination` 為 `San Diego` 的航班，則其 `price` 和 `discounted` 欄位將會更新。如果沒有這樣的航班，則會建立一個新航班，其屬性是將第一個引數陣列與第二個引數陣列合併的結果：

    $flight = Flight::updateOrCreate(
        ['departure' => 'Oakland', 'destination' => 'San Diego'],
        ['price' => 99, 'discounted' => 1]
    );

<a name="mass-updates"></a>
#### 大量更新

也可以對符合給定查詢的 Model 執行更新。在此範例中，所有 `active` 且 `destination` 為 `San Diego` 的航班都將被標記為延遲：

    Flight::where('active', 1)
        ->where('destination', 'San Diego')
        ->update(['delayed' => 1]);

`update` 方法預期一個由欄位和值組成的陣列，代表應更新的欄位。`update` 方法會返回受影響的行數。

> [!WARNING]  
> 透過 Eloquent 執行大量更新時，`saving`、`saved`、`updating` 和 `updated` 等 Model 事件將不會針對已更新的 Model 觸發。這是因為在執行大量更新時，Model 並未實際被擷取。

<a name="examining-attribute-changes"></a>
#### 檢查屬性變更

Eloquent 提供了 `isDirty`、`isClean` 和 `wasChanged` 方法來檢查 Model 的內部狀態，並判斷其屬性自 Model 最初被擷取以來有何變化。

`isDirty` 方法判斷 Model 的任何屬性自 Model 被擷取以來是否已更改。您可以傳遞特定的屬性名稱或屬性陣列給 `isDirty` 方法，以判斷其中任何屬性是否「已變更 (dirty)」。`isClean` 方法將判斷屬性自 Model 被擷取以來是否保持不變。此方法也接受一個可選的屬性引數：

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

`wasChanged` 方法判斷 Model 在當前請求週期中上次儲存時是否有任何屬性被更改。如果需要，您可以傳遞一個屬性名稱來查看特定屬性是否被更改：

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

`getOriginal` 方法返回一個陣列，其中包含 Model 的原始屬性，而無論 Model 自被擷取以來是否有任何更改。如果需要，您可以傳遞特定的屬性名稱來獲取特定屬性的原始值：

    $user = User::find(1);

    $user->name; // John
    $user->email; // john@example.com

    $user->name = "Jack";
    $user->name; // Jack

    $user->getOriginal('name'); // John
    $user->getOriginal(); // Array of original attributes...

<a name="mass-assignment"></a>
### 大量指派

您可以使用 `create` 方法，以單一 PHP 語句「儲存」新的 Model。此方法會回傳新插入的 Model 實例：

    use App\Models\Flight;

    $flight = Flight::create([
        'name' => 'London to Paris',
    ]);

然而，在使用 `create` 方法之前，您需要為 Model 類別指定 `fillable` 或 `guarded` 屬性。這些屬性是必要的，因為所有 Eloquent Model 預設都受到保護，以防範大量指派漏洞。

當使用者傳入一個意外的 HTTP 請求欄位，而該欄位卻更改了資料庫中您不希望被更改的欄位時，就會發生大量指派漏洞。例如，惡意使用者可能會透過 HTTP 請求傳送一個 `is_admin` 參數，然後此參數被傳遞到 Model 的 `create` 方法，從而允許該使用者將自己提升為管理員。

因此，首先您應該定義哪些 Model 屬性是可大量指派的。您可以使用 Model 上的 `$fillable` 屬性來完成此操作。例如，我們將 `Flight` Model 的 `name` 屬性設定為可大量指派：

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

一旦您指定了哪些屬性是可大量指派的，您就可以使用 `create` 方法在資料庫中插入一條新記錄。`create` 方法會回傳新建立的 Model 實例：

    $flight = Flight::create(['name' => 'London to Paris']);

如果您已經有 Model 實例，您可以使用 `fill` 方法以屬性陣列填充它：

    $flight->fill(['name' => 'Amsterdam to Frankfurt']);

<a name="mass-assignment-json-columns"></a>
#### 大量指派與 JSON 欄位

在指派 JSON 欄位時，每個欄位的可大量指派的鍵必須在 Model 的 `$fillable` 陣列中指定。為了安全性，Laravel 在使用 `guarded` 屬性時不支援更新巢狀 JSON 屬性：

    /**
     * The attributes that are mass assignable.
     *
     * @var array<int, string>
     */
    protected $fillable = [
        'options->enabled',
    ];

<a name="allowing-mass-assignment"></a>
#### 允許大量指派

如果您希望將所有屬性都設定為可大量指派，您可以將 Model 的 `$guarded` 屬性定義為一個空陣列。如果您選擇解除 Model 的保護，則應該特別注意始終手動建立傳遞給 Eloquent `fill`、`create` 和 `update` 方法的陣列：

    /**
     * The attributes that aren't mass assignable.
     *
     * @var array<string>|bool
     */
    protected $guarded = [];

<a name="mass-assignment-exceptions"></a>
#### 大量指派例外

預設情況下，不在 `$fillable` 陣列中的屬性在執行大量指派操作時會被默默地丟棄。在正式環境中，這是預期的行為；然而，在本地開發期間，這可能會導致 Model 變更沒有生效的困惑。

如果您願意，可以透過呼叫 `preventSilentlyDiscardingAttributes` 方法，指示 Laravel 在嘗試填充不可填充屬性時拋出例外。通常，此方法應在應用程式 `AppServiceProvider` 類別的 `boot` 方法中呼叫：

    use Illuminate\Database\Eloquent\Model;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        Model::preventSilentlyDiscardingAttributes($this->app->isLocal());
    }

<a name="upserts"></a>
### 更新或插入

Eloquent 的 `upsert` 方法可用於在單一原子性操作中更新或建立記錄。該方法的第一個引數包含要插入或更新的值，而第二個引數則列出在關聯資料表中唯一識別記錄的欄位。該方法的第三個也是最後一個引數是一個欄位陣列，如果資料庫中已存在匹配的記錄，則應更新這些欄位。如果 Model 上啟用時間戳記，`upsert` 方法將自動設定 `created_at` 和 `updated_at` 時間戳記：

    Flight::upsert([
        ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' => 99],
        ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
    ], uniqueBy: ['departure', 'destination'], update: ['price']);

> [!WARNING]  
> 除 SQL Server 外的所有資料庫都要求 `upsert` 方法第二個引數中的欄位具有「主鍵」或「唯一索引」。此外，MariaDB 和 MySQL 資料庫驅動程式會忽略 `upsert` 方法的第二個引數，並始終使用資料表的主鍵和唯一索引來檢測現有記錄。

<a name="deleting-models"></a>
## 刪除 Model

要刪除 Model，您可以呼叫 Model 實例上的 `delete` 方法：

    use App\Models\Flight;

    $flight = Flight::find(1);

    $flight->delete();


<a name="deleting-an-existing-model-by-its-primary-key"></a>
#### 透過主鍵刪除現有的 Model

在上述範例中，我們在呼叫 `delete` 方法之前會先從資料庫中擷取 Model。然而，如果您知道 Model 的主鍵，則可以直接呼叫 `destroy` 方法來刪除 Model，而無需明確地擷取它。除了接受單一主鍵外，`destroy` 方法還可接受多個主鍵、一個主鍵陣列或一個 [collection](/docs/{{version}}/collections) 集合：

    Flight::destroy(1);

    Flight::destroy(1, 2, 3);

    Flight::destroy([1, 2, 3]);

    Flight::destroy(collect([1, 2, 3]));

如果您正在使用 [軟刪除 Model](#soft-deleting)，則可透過 `forceDestroy` 方法來永久刪除 Model：

    Flight::forceDestroy(1);

> [!WARNING]  
> `destroy` 方法會單獨載入每個 Model 並呼叫 `delete` 方法，以便正確地觸發每個 Model 的 `deleting` 與 `deleted` 事件。


<a name="deleting-models-using-queries"></a>
#### 透過查詢刪除 Model

當然，您可以建立 Eloquent 查詢來刪除所有符合查詢條件的 Model。在此範例中，我們將刪除所有標記為非啟用的 flights。如同大量更新一樣，大量刪除也不會為被刪除的 Model 觸發 Model 事件：

    $deleted = Flight::where('active', 0)->delete();

要刪除資料表中的所有 Model，您應該執行一個不帶任何條件的查詢：

    $deleted = Flight::query()->delete();

> [!WARNING]  
> 透過 Eloquent 執行大量刪除語句時，受影響的 Model 將不會觸發 `deleting` 與 `deleted` Model 事件。這是因為在執行刪除語句時，Model 並未實際被擷取。


<a name="soft-deleting"></a>
### 軟刪除

除了實際從資料庫中刪除紀錄外，Eloquent 還可以「軟刪除」Model。當 Model 被軟刪除時，它們並不會實際從資料庫中移除。相反地，Model 上會設定一個 `deleted_at` 屬性，表示 Model 被「刪除」的日期與時間。若要為 Model 啟用軟刪除，請將 `Illuminate\Database\Eloquent\SoftDeletes` Trait 加入 Model：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\SoftDeletes;

    class Flight extends Model
    {
        use SoftDeletes;
    }

> [!NOTE]  
> `SoftDeletes` Trait 會自動將 `deleted_at` 屬性轉換為 `DateTime` / `Carbon` 實例。

您還應該將 `deleted_at` 欄位新增到資料庫資料表中。Laravel 的 [schema builder](/docs/{{version}}/migrations) 包含一個輔助方法來建立此欄位：

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('flights', function (Blueprint $table) {
        $table->softDeletes();
    });

    Schema::table('flights', function (Blueprint $table) {
        $table->dropSoftDeletes();
    });

現在，當您在 Model 上呼叫 `delete` 方法時，`deleted_at` 欄位將會設定為目前的日期與時間。然而，Model 的資料庫紀錄將會保留在資料表中。當查詢使用軟刪除的 Model 時，軟刪除的 Model 將會自動從所有查詢結果中排除。

要判斷給定的 Model 實例是否已被軟刪除，您可以使用 `trashed` 方法：

    if ($flight->trashed()) {
        // ...
    }


<a name="restoring-soft-deleted-models"></a>
#### 復原軟刪除的 Model

有時您可能希望「取消刪除」一個軟刪除的 Model。要復原軟刪除的 Model，您可以在 Model 實例上呼叫 `restore` 方法。`restore` 方法會將 Model 的 `deleted_at` 欄位設定為 `null`：

    $flight->restore();

您也可以在查詢中使用 `restore` 方法來復原多個 Model。同樣地，如同其他「大量」操作，這不會為被復原的 Model 觸發任何 Model 事件：

    Flight::withTrashed()
            ->where('airline_id', 1)
            ->restore();

在建立 [關聯](/docs/{{version}}/eloquent-relationships) 查詢時，也可以使用 `restore` 方法：

    $flight->history()->restore();


<a name="permanently-deleting-models"></a>
#### 永久刪除 Model

有時您可能需要真正地從資料庫中移除 Model。您可以使用 `forceDelete` 方法來永久地從資料庫資料表中移除軟刪除的 Model：

    $flight->forceDelete();

在建立 Eloquent 關聯查詢時，也可以使用 `forceDelete` 方法：

    $flight->history()->forceDelete();


<a name="querying-soft-deleted-models"></a>
### 查詢軟刪除的 Model


<a name="including-soft-deleted-models"></a>
#### 包含軟刪除的 Model

如上所述，軟刪除的 Model 將會自動從查詢結果中排除。然而，您可以在查詢中呼叫 `withTrashed` 方法，強制讓軟刪除的 Model 包含在查詢結果中：

    use App\Models\Flight;

    $flights = Flight::withTrashed()
        ->where('account_id', 1)
        ->get();

在建立 [關聯](/docs/{{version}}/eloquent-relationships) 查詢時，也可以呼叫 `withTrashed` 方法：

    $flight->history()->withTrashed()->get();


<a name="retrieving-only-soft-deleted-models"></a>
#### 僅擷取軟刪除的 Model

`onlyTrashed` 方法將會**僅**擷取軟刪除的 Model：

    $flights = Flight::onlyTrashed()
        ->where('airline_id', 1)
        ->get();

<a name="pruning-models"></a>
## 修剪 Model

有時你可能希望定期刪除不再需要的 Model。為了達成此目的，你可以將 `Illuminate\Database\Eloquent\Prunable` 或 `Illuminate\Database\Eloquent\MassPrunable` trait 加入到你想要定期修剪的 Model 中。將其中一個 trait 加入到 Model 後，實作一個 `prunable` 方法，此方法會回傳一個 Eloquent 查詢建構器，用以找出不再需要的 Model：

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

當將 Model 標記為 `Prunable` 時，你也可以在 Model 上定義一個 `pruning` 方法。此方法會在 Model 刪除前被呼叫。在 Model 從資料庫中被永久移除之前，此方法可用於刪除與 Model 相關的任何額外資源，例如儲存的檔案：

    /**
     * Prepare the model for pruning.
     */
    protected function pruning(): void
    {
        // ...
    }

設定好可修剪的 Model 後，你應該在應用程式的 `routes/console.php` 檔案中排程 `model:prune` Artisan 指令。你可以自由選擇此指令應運行的適當時間間隔：

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('model:prune')->daily();

在幕後，`model:prune` 指令會自動偵測應用程式 `app/Models` 目錄中「Prunable」的 Model。如果你的 Model 位於不同的位置，你可以使用 `--model` 選項來指定 Model 類別名稱：

    Schedule::command('model:prune', [
        '--model' => [Address::class, Flight::class],
    ])->daily();

如果你希望在修剪所有其他偵測到的 Model 時，將某些 Model 排除在修剪之外，你可以使用 `--except` 選項：

    Schedule::command('model:prune', [
        '--except' => [Address::class, Flight::class],
    ])->daily();

你可以執行帶有 `--pretend` 選項的 `model:prune` 指令來測試你的 `prunable` 查詢。在模擬時，`model:prune` 指令只會報告如果實際運行該指令會修剪多少筆記錄：

```shell
php artisan model:prune --pretend
```

> [!WARNING]  
> 軟刪除的 Model 如果符合修剪查詢，將會被永久刪除 (`forceDelete`)。

<a name="mass-pruning"></a>
#### 大量修剪

當 Model 被標記 `Illuminate\Database\Eloquent\MassPrunable` trait 時，它們會使用大量刪除查詢從資料庫中刪除。因此，`pruning` 方法將不會被呼叫，`deleting` 和 `deleted` Model 事件也不會被分派。這是因為在刪除之前，Model 從未實際被擷取，從而使修剪過程更有效率：

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

<a name="replicating-models"></a>
## 複製 Model

你可以使用 `replicate` 方法建立現有 Model 實例的未儲存副本。當你的 Model 實例共享許多相同的屬性時，此方法特別有用：

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

要將一個或多個屬性排除在新 Model 的複製之外，你可以將一個陣列傳遞給 `replicate` 方法：

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

<a name="query-scopes"></a>
## 查詢 Scope


<a name="global-scopes"></a>
### 全域 Scope

全域 Scope 讓您能夠為指定 Model 的所有查詢新增限制。Laravel 自帶的 [軟刪除](#soft-deleting) 功能便利用全域 Scope，只從資料庫中擷取「未刪除」的 Model。編寫您自己的全域 Scope，可以方便、簡單地確保指定 Model 的每個查詢都接收到某些限制。


<a name="generating-scopes"></a>
#### 產生 Scope

要產生新的全域 Scope，您可以呼叫 `make:scope` Artisan 命令，這會將產生的 Scope 放置在應用程式的 `app/Models/Scopes` 目錄中：

```shell
php artisan make:scope AncientScope
```


<a name="writing-global-scopes"></a>
#### 編寫全域 Scope

編寫全域 Scope 很簡單。首先，使用 `make:scope` 命令來產生一個實作 `Illuminate\Database\Eloquent\Scope` 介面的類別。`Scope` 介面要求您實作一個方法：`apply`。`apply` 方法可以根據需要，為查詢新增 `where` 限制或其他類型的子句：

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

> [!NOTE]  
> 如果您的全域 Scope 要為查詢的 select 子句新增欄位，您應該使用 `addSelect` 方法而不是 `select`。這將防止意外替換查詢中現有的 select 子句。


<a name="applying-global-scopes"></a>
#### 應用全域 Scope

要為 Model 指派全域 Scope，您可以簡單地將 `ScopedBy` 屬性放置在 Model 上：

    <?php

    namespace App\Models;

    use App\Models\Scopes\AncientScope;
    use Illuminate\Database\Eloquent\Attributes\ScopedBy;

    #[ScopedBy([AncientScope::class])]
    class User extends Model
    {
        //
    }

或者，您也可以透過覆寫 Model 的 `booted` 方法並呼叫 Model 的 `addGlobalScope` 方法來手動註冊全域 Scope。`addGlobalScope` 方法接受您的 Scope 實例作為其唯一引數：

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

在上面的範例中，將 Scope 新增到 `App\Models\User` Model 後，呼叫 `User::all()` 方法將執行以下 SQL 查詢：

```sql
select * from `users` where `created_at` < 0021-02-18 00:00:00
```


<a name="anonymous-global-scopes"></a>
#### 匿名全域 Scope

Eloquent 也允許您使用 Closure 定義全域 Scope，這對於不需要獨立類別的簡單 Scope 特別有用。當使用 Closure 定義全域 Scope 時，您應該提供一個自選的 Scope 名稱作為 `addGlobalScope` 方法的第一個引數：

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


<a name="removing-global-scopes"></a>
#### 移除全域 Scope

如果您想為指定查詢移除全域 Scope，可以使用 `withoutGlobalScope` 方法。此方法接受全域 Scope 的類別名稱作為其唯一引數：

    User::withoutGlobalScope(AncientScope::class)->get();

或者，如果您是使用 Closure 定義全域 Scope，則應該傳遞您為該全域 Scope 指派的字串名稱：

    User::withoutGlobalScope('ancient')->get();

如果您想移除查詢中的多個甚至所有全域 Scope，可以使用 `withoutGlobalScopes` 方法：

    // Remove all of the global scopes...
    User::withoutGlobalScopes()->get();

    // Remove some of the global scopes...
    User::withoutGlobalScopes([
        FirstScope::class, SecondScope::class
    ])->get();


<a name="local-scopes"></a>
### 本地 Scope

本地 Scope 讓您能夠定義常用的查詢限制集合，這些限制可以在您的應用程式中輕鬆重複使用。例如，您可能需要頻繁地擷取所有被認為「受歡迎」的使用者。要定義一個 Scope，請在 Eloquent Model 方法前加上 `scope` 前綴。

Scope 應始終回傳相同的查詢產生器實例或 `void`：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Builder;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * Scope a query to only include popular users.
         */
        public function scopePopular(Builder $query): void
        {
            $query->where('votes', '>', 100);
        }

        /**
         * Scope a query to only include active users.
         */
        public function scopeActive(Builder $query): void
        {
            $query->where('active', 1);
        }
    }


<a name="utilizing-a-local-scope"></a>
#### 利用本地 Scope

定義 Scope 後，您可以在查詢 Model 時呼叫 Scope 方法。然而，在呼叫方法時不應包含 `scope` 前綴。您甚至可以串聯呼叫多個 Scope：

    use App\Models\User;

    $users = User::popular()->active()->orderBy('created_at')->get();

透過 `or` 查詢運算子組合多個 Eloquent Model Scope 可能需要使用 Closure 才能達到正確的 [邏輯分組](/docs/{{version}}/queries#logical-grouping)：

    $users = User::popular()->orWhere(function (Builder $query) {
        $query->active();
    })->get();

然而，由於這可能很繁瑣，Laravel 提供了一個「高階」的 `orWhere` 方法，讓您可以流暢地串聯 Scope，而無需使用 Closure：

    $users = User::popular()->orWhere->active()->get();


<a name="dynamic-scopes"></a>
#### 動態 Scope

有時您可能希望定義一個接受參數的 Scope。首先，只需將您的額外參數新增到 Scope 方法的簽章中。Scope 參數應在 `$query` 參數之後定義：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Builder;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * Scope a query to only include users of a given type.
         */
        public function scopeOfType(Builder $query, string $type): void
        {
            $query->where('type', $type);
        }
    }

將預期引數新增到 Scope 方法的簽章後，您可以在呼叫 Scope 時傳遞引數：

    $users = User::ofType('admin')->get();

<a name="pending-attributes"></a>
### 待定屬性

如果您想使用 Scope 來建立與用來限制該 Scope 的 Model 具有相同屬性的 Model，您可以在建立 Scope 查詢時使用 `withAttributes` 方法：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Builder;
    use Illuminate\Database\Eloquent\Model;

    class Post extends Model
    {
        /**
         * Scope the query to only include drafts.
         */
        public function scopeDraft(Builder $query): void
        {
            $query->withAttributes([
                'hidden' => true,
            ]);
        }
    }

`withAttributes` 方法將使用給定的屬性為查詢新增 `where` 子句限制，它還會將這些屬性新增至透過該 Scope 建立的任何 Model 中：

    $draft = Post::draft()->create(['title' => 'In Progress']);

    $draft->hidden; // true

<a name="comparing-models"></a>
## 比較 Model

有時您可能需要判斷兩個 Model 是否「相同」。`is` 和 `isNot` 方法可用來快速驗證兩個 Model 是否擁有相同的主鍵、資料表及資料庫連線：

    if ($post->is($anotherPost)) {
        // ...
    }

    if ($post->isNot($anotherPost)) {
        // ...
    }

`is` 和 `isNot` 方法也適用於 `belongsTo`、`hasOne`、`morphTo` 及 `morphOne` 的 [關聯](/docs/{{version}}/eloquent-relationships)。當您想比較相關聯的 Model 而不需要執行查詢來擷取該 Model 時，此方法特別有用：

    if ($post->author()->is($user)) {
        // ...
    }

<a name="events"></a>
## 事件

> [!NOTE]  
> 想要將您的 Eloquent 事件直接廣播至您的用戶端應用程式嗎？請參考 Laravel 的 [Model 事件廣播文件](/docs/{{version}}/broadcasting#model-broadcasting)。

Eloquent Model 會分派數個事件，讓您可以掛載 (hook into) 到 Model 生命週期中的下列時機點：`retrieved`、`creating`、`created`、`updating`、`updated`、`saving`、`saved`、`deleting`、`deleted`、`trashed`、`forceDeleting`、`forceDeleted`、`restoring`、`restored` 和 `replicating`。

當現有的 Model 從資料庫中被擷取時，會分派 `retrieved` 事件。當新的 Model 首次儲存時，會分派 `creating` 和 `created` 事件。當現有的 Model 被修改且呼叫 `save` 方法時，會分派 `updating` / `updated` 事件。當 Model 被建立或更新時，即使 Model 的屬性未曾變更，也會分派 `saving` / `saved` 事件。以 `-ing` 結尾的事件會在 Model 的任何變更持久化之前分派，而以 `-ed` 結尾的事件則會在 Model 的變更持久化之後分派。

要開始監聽 Model 事件，請在您的 Eloquent Model 上定義 `$dispatchesEvents` 屬性。這個屬性會將 Eloquent Model 生命週期中的各個時機點對應到您自己的 [事件類別](/docs/{{version}}/events)。每個 Model 事件類別都應該預期透過其建構函式接收受影響 Model 的實例：

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

在定義並對應了 Eloquent 事件之後，您可以使用 [事件監聽器](/docs/{{version}}/events#defining-listeners) 來處理這些事件。

> [!WARNING]  
> 當透過 Eloquent 發出大量更新或刪除查詢時，受影響的 Model 將不會分派 `saved`、`updated`、`deleting` 和 `deleted` 等 Model 事件。這是因為在執行大量更新或刪除時，Model 實際上並未被擷取。

<a name="events-using-closures"></a>
### 使用 Closure

您可以使用 Closure 來註冊，使其在分派各種 Model 事件時執行，而不必使用自訂事件類別。通常，您應該在 Model 的 `booted` 方法中註冊這些 Closure：

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

如果需要，您可以在註冊 Model 事件時使用 [可佇列的匿名事件監聽器](/docs/{{version}}/events#queuable-anonymous-event-listeners)。這將指示 Laravel 使用您應用程式的 [佇列](/docs/{{version}}/queues) 在背景執行 Model 事件監聽器：

    use function Illuminate\Events\queueable;

    static::created(queueable(function (User $user) {
        // ...
    }));

<a name="observers"></a>
### Observer 觀察者

<a name="defining-observers"></a>
#### 定義 Observer 觀察者

如果您正在監聽給定 Model 上的許多事件，您可以使用 Observer 觀察者將所有監聽器分組到一個類別中。Observer 觀察者類別具有反映您希望監聽的 Eloquent 事件的方法名稱。這些方法中的每一個都會接收受影響的 Model 作為其唯一引數。`make:observer` Artisan 命令是建立新 Observer 觀察者類別最簡單的方法：

```shell
php artisan make:observer UserObserver --model=User
```

此命令會將新的 Observer 觀察者放置在您的 `app/Observers` 目錄中。如果此目錄不存在，Artisan 會為您建立它。您新的 Observer 觀察者會如下所示：

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

要註冊 Observer 觀察者，您可以將 `ObservedBy` 屬性放置在對應的 Model 上：

    use App\Observers\UserObserver;
    use Illuminate\Database\Eloquent\Attributes\ObservedBy;

    #[ObservedBy([UserObserver::class])]
    class User extends Authenticatable
    {
        //
    }

或者，您可以透過在您希望觀察的 Model 上呼叫 `observe` 方法來手動註冊 Observer 觀察者。您可以在應用程式的 `AppServiceProvider` 類別的 `boot` 方法中註冊 Observer 觀察者：

    use App\Models\User;
    use App\Observers\UserObserver;

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        User::observe(UserObserver::class);
    }

> [!NOTE]  
> Observer 觀察者還可以監聽其他事件，例如 `saving` 和 `retrieved`。這些事件在 [事件](#events) 文件中有詳細描述。

<a name="observers-and-database-transactions"></a>
#### Observer 觀察者與資料庫交易

當 Model 在資料庫交易中建立時，您可能希望指示 Observer 觀察者僅在資料庫交易提交後執行其事件處理器。您可以透過在 Observer 觀察者上實作 `ShouldHandleEventsAfterCommit` 介面來實現此目的。如果資料庫交易未在進行中，事件處理器將會立即執行：

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

<a name="muting-events"></a>
### 靜音事件

您可能偶爾需要暫時「靜音」由 Model 觸發的所有事件。您可以使用 `withoutEvents` 方法來實現此目的。`withoutEvents` 方法只接受一個 Closure 作為其唯一引數。在此 Closure 中執行的任何程式碼都不會分派 Model 事件，並且 Closure 返回的任何值都將由 `withoutEvents` 方法返回：

    use App\Models\User;

    $user = User::withoutEvents(function () {
        User::findOrFail(1)->delete();

        return User::find(2);
    });

<a name="saving-a-single-model-without-events"></a>
#### 不分派事件儲存單一 Model

有時您可能希望在不分派任何事件的情況下「儲存」給定的 Model。您可以使用 `saveQuietly` 方法來實現此目的：

    $user = User::findOrFail(1);

    $user->name = 'Victoria Faith';

    $user->saveQuietly();

您也可以在不分派任何事件的情況下「更新」、「刪除」、「軟刪除」、「還原」和「複製」給定的 Model：

    $user->deleteQuietly();
    $user->forceDeleteQuietly();
    $user->restoreQuietly();