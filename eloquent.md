# Eloquent：入門

- [簡介](#introduction)
- [產生模型類別](#generating-model-classes)
- [Eloquent 模型慣例](#eloquent-model-conventions)
    - [資料表名稱](#table-names)
    - [主鍵](#primary-keys)
    - [UUID 與 ULID 鍵](#uuid-and-ulid-keys)
    - [時間戳記](#timestamps)
    - [資料庫連線](#database-connections)
    - [預設屬性值](#default-attribute-values)
    - [設定 Eloquent 嚴格模式](#configuring-eloquent-strictness)
- [擷取模型](#retrieving-models)
    - [集合](#collections)
    - [分批處理結果](#chunking-results)
    - [使用 Lazy Collections 分批處理](#chunking-using-lazy-collections)
    - [游標](#cursors)
    - [進階子查詢](#advanced-subqueries)
- [擷取單一模型 / 聚合](#retrieving-single-models)
    - [擷取或建立模型](#retrieving-or-creating-models)
    - [擷取聚合](#retrieving-aggregates)
- [插入與更新模型](#inserting-and-updating-models)
    - [插入](#inserts)
    - [更新](#updates)
    - [大量賦值](#mass-assignment)
    - [更新或插入 (Upserts)](#upserts)
- [刪除模型](#deleting-models)
    - [軟刪除](#soft-deleting)
    - [查詢軟刪除的模型](#querying-soft-deleted-models)
- [修剪模型](#pruning-models)
- [複製模型](#replicating-models)
- [查詢範圍](#query-scopes)
    - [全域範圍](#global-scopes)
    - [局部範圍](#local-scopes)
    - [待處理屬性](#pending-attributes)
- [比較模型](#comparing-models)
- [事件](#events)
    - [使用閉包](#events-using-closures)
    - [觀察者](#observers)
    - [靜默事件](#muting-events)

<a name="introduction"></a>
## 簡介

Laravel 內建了 Eloquent，這是一個物件關聯對映 (ORM)，讓您能愉快地與資料庫互動。使用 Eloquent 時，每個資料庫資料表都有一個對應的「Model」，用於與該資料表互動。除了從資料庫資料表擷取記錄外，Eloquent 模型還允許您插入、更新和刪除資料表中的記錄。

> [!NOTE]  
> 在開始之前，請務必在應用程式的 `config/database.php` 設定檔中設定資料庫連線。有關設定資料庫的更多資訊，請參閱[資料庫設定說明文件](/docs/{{version}}/database#configuration)。

#### Laravel Bootcamp

如果您是 Laravel 的新手，歡迎跳到 [Laravel Bootcamp](https://bootcamp.laravel.com)。Laravel Bootcamp 將引導您使用 Eloquent 建立您的第一個 Laravel 應用程式。這是了解 Laravel 和 Eloquent 所提供的一切的好方法。

<a name="generating-model-classes"></a>
## 產生模型類別

首先，讓我們建立一個 Eloquent 模型。模型通常位於 `app\Models` 目錄中，並繼承 `Illuminate\Database\Eloquent\Model` 類別。您可以使用 `make:model` [Artisan 命令](/docs/{{version}}/artisan)來產生一個新模型：

```shell
php artisan make:model Flight
```

如果您想在產生模型時產生[資料庫遷移](/docs/{{version}}/migrations)，可以使用 `--migration` 或 `-m` 選項：

```shell
php artisan make:model Flight --migration
```

在產生模型時，您可以產生各種其他類型的類別，例如 factories、seeders、policies、controllers 和 form requests。此外，這些選項可以組合起來一次建立多個類別：

```shell
# 產生一個模型和一個 FlightFactory 類別...
php artisan make:model Flight --factory
php artisan make:model Flight -f

# 產生一個模型和一個 FlightSeeder 類別...
php artisan make:model Flight --seed
php artisan make:model Flight -s

# 產生一個模型和一個 FlightController 類別...
php artisan make:model Flight --controller
php artisan make:model Flight -c

# 產生一個模型、FlightController 資源類別和表單請求類別...
php artisan make:model Flight --controller --resource --requests
php artisan make:model Flight -crR

# 產生一個模型和一個 FlightPolicy 類別...
php artisan make:model Flight --policy

# 產生一個模型、遷移、factory、seeder 和 controller...
php artisan make:model Flight -mfsc

# 產生一個模型、遷移、factory、seeder、policy、controller 和表單請求的捷徑...
php artisan make:model Flight --all
php artisan make:model Flight -a

# 產生一個 pivot 模型...
php artisan make:model Member --pivot
php artisan make:model Member -p
```

<a name="inspecting-models"></a>
#### 檢查模型

有時，僅透過瀏覽模型的程式碼很難確定模型的所有可用屬性和關聯。您可以嘗試 `model:show` Artisan 命令，它提供了模型所有屬性和關聯的便捷概覽：

```shell
php artisan model:show Flight
```

<a name="eloquent-model-conventions"></a>
## Eloquent 模型慣例

由 `make:model` 命令產生的模型將會放置在 `app/Models` 目錄中。讓我們檢查一個基本的模型類別，並討論一些 Eloquent 的關鍵慣例：

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;

    class Flight extends Model
    {
        // ...
    }

<a name="table-names"></a>
### 資料表名稱

看過上面的範例後，您可能已經注意到我們沒有告訴 Eloquent 哪個資料庫資料表對應到我們的 `Flight` 模型。按照慣例，類別的「snake case」複數名稱將用作資料表名稱，除非明確指定另一個名稱。因此，在這種情況下，Eloquent 將假定 `Flight` 模型將記錄儲存在 `flights` 資料表中，而 `AirTrafficController` 模型將記錄儲存在 `air_traffic_controllers` 資料表中。

如果您的模型對應的資料庫資料表不符合此慣例，您可以透過在模型上定義 `table` 屬性來手動指定模型的資料表名稱：

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

Eloquent 還會假定每個模型對應的資料庫資料表都有一個名為 `id` 的主鍵欄位。如有必要，您可以在模型上定義一個受保護的 `$primaryKey` 屬性，以指定作為模型主鍵的不同欄位：

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

此外，Eloquent 假定主鍵是一個遞增的整數值，這表示 Eloquent 會自動將主鍵轉換為整數。如果您希望使用非遞增或非數值的主鍵，您必須在模型上定義一個公開的 `$incrementing` 屬性，並將其設定為 `false`：

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

如果您的模型主鍵不是整數，您應該在模型上定義一個受保護的 `$keyType` 屬性。此屬性應具有 `string` 值：

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

Eloquent 要求每個模型至少有一個唯一的識別「ID」作為其主鍵。Eloquent 模型不支援「複合」主鍵。但是，除了資料表的唯一識別主鍵之外，您可以自由地為資料庫資料表新增額外的多欄位唯一索引。

<a name="uuid-and-ulid-keys"></a>
### UUID 與 ULID 鍵

您可以選擇使用 UUID 而不是自動遞增的整數作為 Eloquent 模型的主鍵。UUID 是通用唯一的英數字元識別碼，長度為 36 個字元。

如果您希望模型使用 UUID 鍵而不是自動遞增的整數鍵，您可以在模型上使用 `Illuminate\Database\Eloquent\Concerns\HasUuids` trait。當然，您應該確保模型具有 [UUID 等效的主鍵欄位](/docs/{{version}}/migrations#column-method-uuid)：

    use Illuminate\Database\Eloquent\Concerns\HasUuids;
    use Illuminate\Database\Eloquent\Model;

    class Article extends Model
    {
        use HasUuids;

        // ...
    }

    $article = Article::create(['title' => 'Traveling to Europe']);

    $article->id; // "8f8e8478-9035-4d23-b9a7-62f4d2612ce5"

預設情況下，`HasUuids` trait 將為您的模型產生「有序」UUIDs。這些 UUIDs 對於索引資料庫儲存更有效率，因為它們可以按字典順序排序。

您可以透過在模型上定義 `newUniqueId` 方法來覆寫給定模型的 UUID 產生過程。此外，您可以透過在模型上定義 `uniqueIds` 方法來指定哪些欄位應接收 UUIDs：

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

如果您願意，您可以選擇使用「ULID」而不是 UUID。ULID 類似於 UUID；但是，它們的長度只有 26 個字元。與有序 UUID 一樣，ULID 可以按字典順序排序，以實現高效的資料庫索引。要使用 ULID，您應該在模型上使用 `Illuminate\Database\Eloquent\Concerns\HasUlids` trait。您還應該確保模型具有 [ULID 等效的主鍵欄位](/docs/{{version}}/migrations#column-method-ulid)：

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

預設情況下，Eloquent 預期您的模型對應的資料庫資料表上存在 `created_at` 和 `updated_at` 欄位。當
