# 資料庫：資料遷移 (Migrations)

- [簡介](#introduction)
- [生成資料遷移](#generating-migrations)
    - [壓縮資料遷移](#squashing-migrations)
- [資料遷移結構](#migration-structure)
- [執行資料遷移](#running-migrations)
    - [回溯資料遷移](#rolling-back-migrations)
- [資料表](#tables)
    - [建立資料表](#creating-tables)
    - [更新資料表](#updating-tables)
    - [重新命名/刪除資料表](#renaming-and-dropping-tables)
- [欄位](#columns)
    - [建立欄位](#creating-columns)
    - [可用欄位類型](#available-column-types)
    - [欄位修飾語](#column-modifiers)
    - [修改欄位](#modifying-columns)
    - [重新命名欄位](#renaming-columns)
    - [刪除欄位](#dropping-columns)
- [索引](#indexes)
    - [建立索引](#creating-indexes)
    - [重新命名索引](#renaming-indexes)
    - [刪除索引](#dropping-indexes)
    - [外鍵約束](#foreign-key-constraints)
- [事件](#events)

<a name="introduction"></a>
## 簡介

資料遷移 (Migrations) 就像資料庫的版本控制，讓您的團隊能夠定義並分享應用程式的資料庫 Schema 定義。如果您曾經在從原始碼控制拉取變更後，不得不手動告知隊友在他們的本機資料庫 Schema 中新增一個欄位，那麼您就已經面對了資料庫遷移所解決的問題。

Laravel 的 `Schema` [Facade](/docs/{{version}}/facades) 為所有 Laravel 支援的資料庫系統提供資料庫無關性支援，以建立和操作資料表。通常，資料遷移會使用此 Facade 來建立和修改資料庫資料表和欄位。

<a name="generating-migrations"></a>
## 生成資料遷移

您可以使用 `make:migration` [Artisan 指令](/docs/{{version}}/artisan) 來生成資料庫遷移。新的資料遷移將會被放置在您的 `database/migrations` 目錄中。每個資料遷移檔案名稱都包含一個時間戳記，讓 Laravel 能夠決定資料遷移的順序：

```shell
php artisan make:migration create_flights_table
```

Laravel 會根據資料遷移的名稱來猜測資料表的名稱，以及該資料遷移是否會建立一個新的資料表。如果 Laravel 能夠從資料遷移名稱中判斷出資料表名稱，Laravel 將會預先填寫生成的資料遷移檔案，其中包含指定的資料表。否則，您可以直接在資料遷移檔案中手動指定資料表。

如果您想為生成的資料遷移指定一個自訂路徑，可以在執行 `make:migration` 指令時使用 `--path` 選項。所給的路徑應相對於您應用程式的根路徑。

> [!NOTE]  
> 資料遷移存根 (stubs) 可以使用 [stub publishing](/docs/{{version}}/artisan#stub-customization) 進行客製化。

<a name="squashing-migrations"></a>
### 壓縮資料遷移

隨著您建構應用程式，您可能會隨著時間累積越來越多的資料遷移。這可能會導致您的 `database/migrations` 目錄膨脹，其中可能包含數百個資料遷移。如果您願意，可以將您的資料遷移「壓縮 (squash)」成單一的 SQL 檔案。要開始，請執行 `schema:dump` 指令：

```shell
php artisan schema:dump

# Dump the current database schema and prune all existing migrations...
php artisan schema:dump --prune
```

當您執行此指令時，Laravel 會將一個「schema」檔案寫入您應用程式的 `database/schema` 目錄。該 schema 檔案的名稱將對應到資料庫連線。現在，當您嘗試遷移資料庫且沒有其他資料遷移被執行時，Laravel 將會首先執行您正在使用的資料庫連線的 schema 檔案中的 SQL 語句。執行完 schema 檔案的 SQL 語句後，Laravel 將會執行任何不屬於 schema 傾印中的剩餘資料遷移。

如果您的應用程式測試使用與您在本地開發期間通常使用的資料庫連線不同的連線，您應該確保已經使用該資料庫連線傾印了一個 schema 檔案，以便您的測試能夠建構您的資料庫。您可能希望在傾印您通常在本地開發期間使用的資料庫連線之後執行此操作：

```shell
php artisan schema:dump
php artisan schema:dump --database=testing --prune
```

您應該將您的資料庫 schema 檔案提交到原始碼控制，以便團隊中其他新開發人員能夠快速建立您應用程式的初始資料庫結構。

> [!WARNING]  
> 資料遷移壓縮僅適用於 MariaDB, MySQL, PostgreSQL, 和 SQLite 資料庫，並且會利用資料庫的命令列客戶端。

<a name="migration-structure"></a>
## 資料遷移結構

資料遷移類別包含兩個方法：`up` 和 `down`。`up` 方法用於為資料庫新增資料表、欄位或索引，而 `down` 方法應反轉 `up` 方法執行的操作。

在這兩個方法中，您可以使用 Laravel 的 schema builder 來表達性地建立和修改資料表。要了解 `Schema` builder 上所有可用的方法，請 [查閱其文件](#creating-tables)。例如，以下資料遷移會建立一個 `flights` 資料表：

    <?php

    use Illuminate\Database\Migrations\Migration;
    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    return new class extends Migration
    {
        /**
         * Run the migrations.
         */
        public function up(): void
        {
            Schema::create('flights', function (Blueprint $table) {
                $table->id();
                $table->string('name');
                $table->string('airline');
                $table->timestamps();
            });
        }

        /**
         * Reverse the migrations.
         */
        public function down(): void
        {
            Schema::drop('flights');
        }
    };

<a name="setting-the-migration-connection"></a>
#### 設定資料遷移連線

如果您的資料遷移將與應用程式預設資料庫連線以外的資料庫連線互動，您應該設定資料遷移的 `$connection` 屬性：

    /**
     * The database connection that should be used by the migration.
     *
     * @var string
     */
    protected $connection = 'pgsql';

    /**
     * Run the migrations.
     */
    public function up(): void
    {
        // ...
    }

<a name="running-migrations"></a>
## 執行資料遷移

要執行所有待執行的資料遷移，請執行 `migrate` Artisan 指令：

```shell
php artisan migrate
```

如果您想查看目前已執行的資料遷移，可以使用 `migrate:status` Artisan 指令：

```shell
php artisan migrate:status
```

如果您想查看資料遷移將執行的 SQL 語句，但不實際執行它們，可以向 `migrate` 指令提供 `--pretend` 旗標：

```shell
php artisan migrate --pretend
```

#### 隔離資料遷移的執行

如果您正在多個伺服器上部署應用程式，並將資料遷移作為部署流程的一部分，您可能不希望兩台伺服器同時嘗試遷移資料庫。為避免這種情況，您可以在調用 `migrate` 指令時使用 `isolated` 選項。

當提供了 `isolated` 選項時，Laravel 將使用您的應用程式的快取驅動程式獲取原子鎖定，然後才嘗試執行您的資料遷移。在鎖定被持有期間，所有其他嘗試運行 `migrate` 指令的操作將不會執行；然而，該指令仍將以成功的退出狀態碼結束：

```shell
php artisan migrate --isolated
```

> [!WARNING]
> 要利用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器都必須與相同的中央快取伺服器通信。

<a name="forcing-migrations-to-run-in-production"></a>
#### 強制在生產環境中執行資料遷移

有些資料遷移操作是破壞性的，這意味著它們可能會導致您丟失資料。為了保護您免受在生產資料庫上執行這些指令，在指令執行前，您將會收到確認提示。若要強制指令在沒有提示的情況下運行，請使用 `--force` 旗標：

```shell
php artisan migrate --force
```

<a name="rolling-back-migrations"></a>
### 回溯資料遷移

要回溯最近一次的資料遷移操作，您可以使用 `rollback` Artisan 指令。此指令將回溯最近的「批次」資料遷移，這可能包含多個資料遷移檔案：

```shell
php artisan migrate:rollback
```

您可以透過向 `rollback` 指令提供 `step` 選項來回溯有限數量的資料遷移。例如，以下指令將回溯最近的五個資料遷移：

```shell
php artisan migrate:rollback --step=5
```

您可以透過向 `rollback` 指令提供 `batch` 選項來回溯特定「批次」的資料遷移，其中 `batch` 選項對應於應用程式 `migrations` 資料庫表中的批次值。例如，以下指令將回溯第三批次中的所有資料遷移：

 ```shell
php artisan migrate:rollback --batch=3
 ```

如果您想查看資料遷移將執行的 SQL 語句，但不實際執行它們，可以向 `migrate:rollback` 指令提供 `--pretend` 旗標：

```shell
php artisan migrate:rollback --pretend
```

`migrate:reset` 指令將回溯應用程式的所有資料遷移：

```shell
php artisan migrate:reset
```

<a name="roll-back-migrate-using-a-single-command"></a>
#### 使用單一指令回溯並執行資料遷移

`migrate:refresh` 指令將回溯所有資料遷移，然後執行 `migrate` 指令。此指令有效地重新建立您的整個資料庫：

```shell
php artisan migrate:refresh

# Refresh the database and run all database seeds...
php artisan migrate:refresh --seed
```

您可以透過向 `refresh` 指令提供 `step` 選項來回溯並重新執行有限數量的資料遷移。例如，以下指令將回溯並重新執行最近的五個資料遷移：

```shell
php artisan migrate:refresh --step=5
```

<a name="drop-all-tables-migrate"></a>
#### 刪除所有資料表並執行資料遷移

`migrate:fresh` 指令將從資料庫中刪除所有資料表，然後執行 `migrate` 指令：

```shell
php artisan migrate:fresh

php artisan migrate:fresh --seed
```

預設情況下，`migrate:fresh` 指令只會從預設資料庫連線中刪除資料表。然而，您可以使用 `--database` 選項來指定應進行資料遷移的資料庫連線。資料庫連線名稱應與應用程式的 `database` [設定檔](/docs/{{version}}/configuration)中定義的連線相對應：

```shell
php artisan migrate:fresh --database=admin
```

> [!WARNING]
> `migrate:fresh` 指令將刪除所有資料庫資料表，無論其前綴為何。在與其他應用程式共用的資料庫上進行開發時，應謹慎使用此指令。

<a name="tables"></a>
## 資料表


<a name="creating-tables"></a>
### 建立資料表

要建立新的資料庫資料表，請使用 `Schema` Facade 上的 `create` 方法。`create` 方法接受兩個引數：第一個是資料表的名稱，第二個是一個閉包 (closure)，它會接收一個 `Blueprint` 物件，可用來定義新資料表：

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::create('users', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->string('email');
        $table->timestamps();
    });

建立資料表時，您可以使用任何 Schema 構建器的 [欄位方法](#creating-columns) 來定義資料表的欄位。


<a name="determining-table-column-existence"></a>
#### 判斷資料表/欄位是否存在

您可以使用 `hasTable`、`hasColumn` 和 `hasIndex` 方法來判斷資料表、欄位或索引是否存在：

    if (Schema::hasTable('users')) {
        // "users" 資料表存在...
    }

    if (Schema::hasColumn('users', 'email')) {
        // "users" 資料表存在且擁有 "email" 欄位...
    }

    if (Schema::hasIndex('users', ['email'], 'unique')) {
        // "users" 資料表存在且 "email" 欄位擁有唯一索引...
    }


<a name="database-connection-table-options"></a>
#### 資料庫連線與資料表選項

如果您想在非應用程式預設資料庫連線上執行 Schema 操作，請使用 `connection` 方法：

    Schema::connection('sqlite')->create('users', function (Blueprint $table) {
        $table->id();
    });

此外，還可以使用其他一些屬性與方法來定義資料表建立的其他層面。當使用 MariaDB 或 MySQL 時，`engine` 屬性可用來指定資料表的儲存引擎：

    Schema::create('users', function (Blueprint $table) {
        $table->engine('InnoDB');

        // ...
    });

`charset` 和 `collation` 屬性可用來指定使用 MariaDB 或 MySQL 時，所建立資料表的字元集和排序規則：

    Schema::create('users', function (Blueprint $table) {
        $table->charset('utf8mb4');
        $table->collation('utf8mb4_unicode_ci');

        // ...
    });

`temporary` 方法可用來指示資料表應為「暫時的 (temporary)」。暫時資料表僅在目前連線的資料庫會話中可見，並會在連線關閉時自動刪除：

    Schema::create('calculations', function (Blueprint $table) {
        $table->temporary();

        // ...
    });

如果您想為資料庫資料表新增「註解 (comment)」，您可以在資料表實例上呼叫 `comment` 方法。目前，資料表註解僅支援 MariaDB、MySQL 和 PostgreSQL：

    Schema::create('calculations', function (Blueprint $table) {
        $table->comment('Business calculations');

        // ...
    });


<a name="updating-tables"></a>
### 更新資料表

`Schema` Facade 上的 `table` 方法可用來更新現有資料表。與 `create` 方法類似，`table` 方法接受兩個引數：資料表的名稱和一個閉包，該閉包會接收一個 `Blueprint` 實例，您可以用它來新增欄位或索引到資料表：

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->integer('votes');
    });


<a name="renaming-and-dropping-tables"></a>
### 重新命名/刪除資料表

要重新命名現有的資料庫資料表，請使用 `rename` 方法：

    use Illuminate\Support\Facades\Schema;

    Schema::rename($from, $to);

要刪除現有的資料表，您可以使用 `drop` 或 `dropIfExists` 方法：

    Schema::drop('users');

    Schema::dropIfExists('users');


<a name="renaming-tables-with-foreign-keys"></a>
#### 重新命名帶有外鍵的資料表

在重新命名資料表之前，您應該確認資料表上的任何外鍵約束在您的資料遷移檔案中都有明確的名稱，而不是讓 Laravel 指定一個基於慣例的名稱。否則，外鍵約束名稱將會指向舊的資料表名稱。

<a name="columns"></a>
## 欄位


<a name="creating-columns"></a>
### 建立欄位

`Schema` facade 上的 `table` 方法可用於更新現有資料表。與 `create` 方法類似，`table` 方法接受兩個引數：資料表的名稱，以及一個接收 `Illuminate\Database\Schema\Blueprint` 實例的閉包，您可以使用此閉包將欄位新增至資料表：

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->integer('votes');
    });

<a name="available-column-types"></a>
### 可用欄位類型

Schema 產生器 (builder) 的藍圖 (blueprint) 提供了多種方法，對應到你可以加入資料庫資料表的不同欄位類型。每個可用方法都列在下面的表格中：

<style>
    .collection-method-list > p {
        columns: 10.8em 3; -moz-columns: 10.8em 3; -webkit-columns: 10.8em 3;
    }

    .collection-method-list a {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .collection-method code {
        font-size: 14px;
    }

    .collection-method:not(.first-collection-method) {
        margin-top: 50px;
    }
</style>


<a name="booleans-method-list"></a>
#### 布林類型

<div class="collection-method-list" markdown="1">

[boolean](#column-method-boolean)

</div>


<a name="strings-and-texts-method-list"></a>
#### 字串與文字類型

<div class="collection-method-list" markdown="1">

[char](#column-method-char)
[longText](#column-method-longText)
[mediumText](#column-method-mediumText)
[string](#column-method-string)
[text](#column-method-text)
[tinyText](#column-method-tinyText)

</div>


<a name="numbers--method-list"></a>
#### 數字類型

<div class="collection-method-list" markdown="1">

[bigIncrements](#column-method-bigIncrements)
[bigInteger](#column-method-bigInteger)
[decimal](#column-method-decimal)
[double](#column-method-double)
[float](#column-method-float)
[id](#column-method-id)
[increments](#column-method-increments)
[integer](#column-method-integer)
[mediumIncrements](#column-method-mediumIncrements)
[mediumInteger](#column-method-mediumInteger)
[smallIncrements](#column-method-smallIncrements)
[smallInteger](#column-method-smallInteger)
[tinyIncrements](#column-method-tinyIncrements)
[tinyInteger](#column-method-tinyInteger)
[unsignedBigInteger](#column-method-unsignedBigInteger)
[unsignedInteger](#column-method-unsignedInteger)
[unsignedMediumInteger](#column-method-unsignedMediumInteger)
[unsignedSmallInteger](#column-method-unsignedSmallInteger)
[unsignedTinyInteger](#column-method-unsignedTinyInteger)

</div>


<a name="dates-and-times-method-list"></a>
#### 日期與時間類型

<div class="collection-method-list" markdown="1">

[dateTime](#column-method-dateTime)
[dateTimeTz](#column-method-dateTimeTz)
[date](#column-method-date)
[time](#column-method-time)
[timeTz](#column-method-timeTz)
[timestamp](#column-method-timestamp)
[timestamps](#column-method-timestamps)
[timestampsTz](#column-method-timestampsTz)
[softDeletes](#column-method-softDeletes)
[softDeletesTz](#column-method-softDeletesTz)
[year](#column-method-year)

</div>


<a name="binaries-method-list"></a>
#### 二進位類型

<div class="collection-method-list" markdown="1">

[binary](#column-method-binary)

</div>


<a name="object-and-jsons-method-list"></a>
#### 物件與 Json 類型

<div class="collection-method-list" markdown="1">

[json](#column-method-json)
[jsonb](#column-method-jsonb)

</div>


<a name="uuids-and-ulids-method-list"></a>
#### UUID 與 ULID 類型

<div class="collection-method-list" markdown="1">

[ulid](#column-method-ulid)
[ulidMorphs](#column-method-ulidMorphs)
[uuid](#column-method-uuid)
[uuidMorphs](#column-method-uuidMorphs)
[nullableUlidMorphs](#column-method-nullableUlidMorphs)
[nullableUuidMorphs](#column-method-nullableUuidMorphs)

</div>


<a name="spatials-method-list"></a>
#### 空間類型

<div class="collection-method-list" markdown="1">

[geography](#column-method-geography)
[geometry](#column-method-geometry)

</div>


#### 關聯類型

<div class="collection-method-list" markdown="1">

[foreignId](#column-method-foreignId)
[foreignIdFor](#column-method-foreignIdFor)
[foreignUlid](#column-method-foreignUlid)
[foreignUuid](#column-method-foreignUuid)
[morphs](#column-method-morphs)
[nullableMorphs](#column-method-nullableMorphs)

</div>


<a name="spacifics-method-list"></a>
#### 特殊類型

<div class="collection-method-list" markdown="1">

[enum](#column-method-enum)
[set](#column-method-set)
[macAddress](#column-method-macAddress)
[ipAddress](#column-method-ipAddress)
[rememberToken](#column-method-rememberToken)
[vector](#column-method-vector)

</div>


<a name="column-method-bigIncrements"></a>
#### `bigIncrements()` {.collection-method .first-collection-method}

`bigIncrements` 方法會建立一個自動遞增的 `UNSIGNED BIGINT` (primary key) 等效欄位：

    $table->bigIncrements('id');


<a name="column-method-bigInteger"></a>
#### `bigInteger()` {.collection-method}

`bigInteger` 方法會建立一個 `BIGINT` 等效欄位：

    $table->bigInteger('votes');


<a name="column-method-binary"></a>
#### `binary()` {.collection-method}

`binary` 方法會建立一個 `BLOB` 等效欄位：

    $table->binary('photo');

當使用 MySQL、MariaDB 或 SQL Server 時，您可以傳遞 `length` 和 `fixed` 參數來建立 `VARBINARY` 或 `BINARY` 等效欄位：

    $table->binary('data', length: 16); // VARBINARY(16)

    $table->binary('data', length: 16, fixed: true); // BINARY(16)


<a name="column-method-boolean"></a>
#### `boolean()` {.collection-method}

`boolean` 方法會建立一個 `BOOLEAN` 等效欄位：

    $table->boolean('confirmed');


<a name="column-method-char"></a>
#### `char()` {.collection-method}

`char` 方法會建立一個指定長度的 `CHAR` 等效欄位：

    $table->char('name', length: 100);


<a name="column-method-dateTimeTz"></a>
#### `dateTimeTz()` {.collection-method}

`dateTimeTz` 方法會建立一個 `DATETIME` (帶有時區) 等效欄位，並可選擇性地指定小數秒精度：

    $table->dateTimeTz('created_at', precision: 0);


<a name="column-method-dateTime"></a>
#### `dateTime()` {.collection-method}

`dateTime` 方法會建立一個 `DATETIME` 等效欄位，並可選擇性地指定小數秒精度：

    $table->dateTime('created_at', precision: 0);


<a name="column-method-date"></a>
#### `date()` {.collection-method}

`date` 方法會建立一個 `DATE` 等效欄位：

    $table->date('created_at');


<a name="column-method-decimal"></a>
#### `decimal()` {.collection-method}

`decimal` 方法會建立一個 `DECIMAL` 等效欄位，並指定精度 (總位數) 和小數位數 (小數點後的位數)：

    $table->decimal('amount', total: 8, places: 2);


<a name="column-method-double"></a>
#### `double()` {.collection-method}

`double` 方法會建立一個 `DOUBLE` 等效欄位：

    $table->double('amount');


<a name="column-method-enum"></a>
#### `enum()` {.collection-method}

`enum` 方法會建立一個 `ENUM` 等效欄位，並帶有給定的有效值：

    $table->enum('difficulty', ['easy', 'hard']);


<a name="column-method-float"></a>
#### `float()` {.collection-method}

`float` 方法會建立一個 `FLOAT` 等效欄位，並帶有給定的精度：

    $table->float('amount', precision: 53);


<a name="column-method-foreignId"></a>
#### `foreignId()` {.collection-method}

`foreignId` 方法會建立一個 `UNSIGNED BIGINT` 等效欄位：

    $table->foreignId('user_id');


<a name="column-method-foreignIdFor"></a>
#### `foreignIdFor()` {.collection-method}

`foreignIdFor` 方法會為給定的模型類別增加一個 `{column}_id` 等效欄位。該欄位類型將依據模型鍵類型為 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`：

    $table->foreignIdFor(User::class);


<a name="column-method-foreignUlid"></a>
#### `foreignUlid()` {.collection-method}

`foreignUlid` 方法會建立一個 `ULID` 等效欄位：

    $table->foreignUlid('user_id');


<a name="column-method-foreignUuid"></a>
#### `foreignUuid()` {.collection-method}

`foreignUuid` 方法會建立一個 `UUID` 等效欄位：

    $table->foreignUuid('user_id');


<a name="column-method-geography"></a>
#### `geography()` {.collection-method}

`geography` 方法會建立一個 `GEOGRAPHY` 等效欄位，並帶有給定的空間類型和 SRID (空間參考系統識別碼)：

    $table->geography('coordinates', subtype: 'point', srid: 4326);

> [!NOTE]  
> 空間類型支援取決於您的資料庫驅動程式。請參考您的資料庫文件。如果您的應用程式使用 PostgreSQL 資料庫，您必須先安裝 [PostGIS](https://postgis.net) 擴充功能才能使用 `geography` 方法。


<a name="column-method-geometry"></a>
#### `geometry()` {.collection-method}

`geometry` 方法會建立一個 `GEOMETRY` 等效欄位，並帶有給定的空間類型和 SRID (空間參考系統識別碼)：

    $table->geometry('positions', subtype: 'point', srid: 0);

> [!NOTE]  
> 空間類型支援取決於您的資料庫驅動程式。請參考您的資料庫文件。如果您的應用程式使用 PostgreSQL 資料庫，您必須先安裝 [PostGIS](https://postgis.net) 擴充功能才能使用 `geometry` 方法。


<a name="column-method-id"></a>
#### `id()` {.collection-method}

`id` 方法是 `bigIncrements` 方法的別名。預設情況下，此方法會建立一個 `id` 欄位；不過，如果您想為該欄位指定不同的名稱，可以傳遞一個欄位名稱：

    $table->id();


<a name="column-method-increments"></a>
#### `increments()` {.collection-method}

`increments` 方法會建立一個自動遞增的 `UNSIGNED INTEGER` 等效欄位作為 primary key：

    $table->increments('id');


<a name="column-method-integer"></a>
#### `integer()` {.collection-method}

`integer` 方法會建立一個 `INTEGER` 等效欄位：

    $table->integer('votes');


<a name="column-method-ipAddress"></a>
#### `ipAddress()` {.collection-method}

`ipAddress` 方法會建立一個 `VARCHAR` 等效欄位：

    $table->ipAddress('visitor');

當使用 PostgreSQL 時，將會建立一個 `INET` 欄位。


<a name="column-method-json"></a>
#### `json()` {.collection-method}

`json` 方法會建立一個 `JSON` 等效欄位：

    $table->json('options');

當使用 SQLite 時，將會建立一個 `TEXT` 欄位。


<a name="column-method-jsonb"></a>
#### `jsonb()` {.collection-method}

`jsonb` 方法會建立一個 `JSONB` 等效欄位：

    $table->jsonb('options');

當使用 SQLite 時，將會建立一個 `TEXT` 欄位。


<a name="column-method-longText"></a>
#### `longText()` {.collection-method}

`longText` 方法會建立一個 `LONGTEXT` 等效欄位：

    $table->longText('description');

當使用 MySQL 或 MariaDB 時，您可以對該欄位應用 `binary` 字元集，以建立 `LONGBLOB` 等效欄位：

    $table->longText('data')->charset('binary'); // LONGBLOB


<a name="column-method-macAddress"></a>
#### `macAddress()` {.collection-method}

`macAddress` 方法會建立一個用於儲存 MAC 位址的欄位。某些資料庫系統，例如 PostgreSQL，為此類資料提供了專用的欄位類型。其他資料庫系統將使用字串等效欄位：

    $table->macAddress('device');


<a name="column-method-mediumIncrements"></a>
#### `mediumIncrements()` {.collection-method}

`mediumIncrements` 方法會建立一個自動遞增的 `UNSIGNED MEDIUMINT` 等效欄位作為 primary key：

    $table->mediumIncrements('id');


<a name="column-method-mediumInteger"></a>
#### `mediumInteger()` {.collection-method}

`mediumInteger` 方法會建立一個 `MEDIUMINT` 等效欄位：

    $table->mediumInteger('votes');


<a name="column-method-mediumText"></a>
#### `mediumText()` {.collection-method}

`mediumText` 方法會建立一個 `MEDIUMTEXT` 等效欄位：

    $table->mediumText('description');

當使用 MySQL 或 MariaDB 時，您可以對該欄位應用 `binary` 字元集，以建立 `MEDIUMBLOB` 等效欄位：

    $table->mediumText('data')->charset('binary'); // MEDIUMBLOB


<a name="column-method-morphs"></a>
#### `morphs()` {.collection-method}

`morphs` 方法是一個便捷方法，它會新增一個 `{column}_id` 等效欄位和一個 `{column}_type` `VARCHAR` 等效欄位。`{column}_id` 的欄位類型將依據模型鍵類型為 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`。

此方法旨在用於定義多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships) 所需的欄位。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 欄位：

    $table->morphs('taggable');


<a name="column-method-nullableMorphs"></a>
#### `nullableMorphs()` {.collection-method}

此方法類似於 [morphs](#column-method-morphs) 方法；不過，所建立的欄位將是「可為 null」的：

    $table->nullableMorphs('taggable');


<a name="column-method-nullableUlidMorphs"></a>
#### `nullableUlidMorphs()` {.collection-method}

此方法類似於 [ulidMorphs](#column-method-ulidMorphs) 方法；不過，所建立的欄位將是「可為 null」的：

    $table->nullableUlidMorphs('taggable');


<a name="column-method-nullableUuidMorphs"></a>
#### `nullableUuidMorphs()` {.collection-method}

此方法類似於 [uuidMorphs](#column-method-uuidMorphs) 方法；不過，所建立的欄位將是「可為 null」的：

    $table->nullableUuidMorphs('taggable');


<a name="column-method-rememberToken"></a>
#### `rememberToken()` {.collection-method}

`rememberToken` 方法會建立一個可為 null 的 `VARCHAR(100)` 等效欄位，旨在儲存當前的「記住我」[authentication token](/docs/{{version}}/authentication#remembering-users)：

    $table->rememberToken();


<a name="column-method-set"></a>
#### `set()` {.collection-method}

`set` 方法會建立一個 `SET` 等效欄位，並帶有給定的有效值列表：

    $table->set('flavors', ['strawberry', 'vanilla']);


<a name="column-method-smallIncrements"></a>
#### `smallIncrements()` {.collection-method}

`smallIncrements` 方法會建立一個自動遞增的 `UNSIGNED SMALLINT` 等效欄位作為 primary key：

    $table->smallIncrements('id');


<a name="column-method-smallInteger"></a>
#### `smallInteger()` {.collection-method}

`smallInteger` 方法會建立一個 `SMALLINT` 等效欄位：

    $table->smallInteger('votes');


<a name="column-method-softDeletesTz"></a>
#### `softDeletesTz()` {.collection-method}

`softDeletesTz` 方法會新增一個可為 null 的 `deleted_at` `TIMESTAMP` (帶有時區) 等效欄位，並可選擇性地指定小數秒精度。此欄位旨在儲存 Eloquent 「軟刪除」功能所需的 `deleted_at` 時間戳記：

    $table->softDeletesTz('deleted_at', precision: 0);


<a name="column-method-softDeletes"></a>
#### `softDeletes()` {.collection-method}

`softDeletes` 方法會新增一個可為 null 的 `deleted_at` `TIMESTAMP` 等效欄位，並可選擇性地指定小數秒精度。此欄位旨在儲存 Eloquent 「軟刪除」功能所需的 `deleted_at` 時間戳記：

    $table->softDeletes('deleted_at', precision: 0);


<a name="column-method-string"></a>
#### `string()` {.collection-method}

`string` 方法會建立一個指定長度的 `VARCHAR` 等效欄位：

    $table->string('name', length: 100);


<a name="column-method-text"></a>
#### `text()` {.collection-method}

`text` 方法會建立一個 `TEXT` 等效欄位：

    $table->text('description');

當使用 MySQL 或 MariaDB 時，您可以對該欄位應用 `binary` 字元集，以建立 `BLOB` 等效欄位：

    $table->text('data')->charset('binary'); // BLOB


<a name="column-method-timeTz"></a>
#### `timeTz()` {.collection-method}

`timeTz` 方法會建立一個 `TIME` (帶有時區) 等效欄位，並可選擇性地指定小數秒精度：

    $table->timeTz('sunrise', precision: 0);


<a name="column-method-time"></a>
#### `time()` {.collection-method}

`time` 方法會建立一個 `TIME` 等效欄位，並可選擇性地指定小數秒精度：

    $table->time('sunrise', precision: 0);


<a name="column-method-timestampTz"></a>
#### `timestampTz()` {.collection-method}

`timestampTz` 方法會建立一個 `TIMESTAMP` (帶有時區) 等效欄位，並可選擇性地指定小數秒精度：

    $table->timestampTz('added_at', precision: 0);


<a name="column-method-timestamp"></a>
#### `timestamp()` {.collection-method}

`timestamp` 方法會建立一個 `TIMESTAMP` 等效欄位，並可選擇性地指定小數秒精度：

    $table->timestamp('added_at', precision: 0);


<a name="column-method-timestampsTz"></a>
#### `timestampsTz()` {.collection-method}

`timestampsTz` 方法會建立 `created_at` 和 `updated_at` `TIMESTAMP` (帶有時區) 等效欄位，並可選擇性地指定小數秒精度：

    $table->timestampsTz(precision: 0);


<a name="column-method-timestamps"></a>
#### `timestamps()` {.collection-method}

`timestamps` 方法會建立 `created_at` 和 `updated_at` `TIMESTAMP` 等效欄位，並可選擇性地指定小數秒精度：

    $table->timestamps(precision: 0);


<a name="column-method-tinyIncrements"></a>
#### `tinyIncrements()` {.collection-method}

`tinyIncrements` 方法會建立一個自動遞增的 `UNSIGNED TINYINT` 等效欄位作為 primary key：

    $table->tinyIncrements('id');


<a name="column-method-tinyInteger"></a>
#### `tinyInteger()` {.collection-method}

`tinyInteger` 方法會建立一個 `TINYINT` 等效欄位：

    $table->tinyInteger('votes');


<a name="column-method-tinyText"></a>
#### `tinyText()` {.collection-method}

`tinyText` 方法會建立一個 `TINYTEXT` 等效欄位：

    $table->tinyText('notes');

當使用 MySQL 或 MariaDB 時，您可以對該欄位應用 `binary` 字元集，以建立 `TINYBLOB` 等效欄位：

    $table->tinyText('data')->charset('binary'); // TINYBLOB


<a name="column-method-unsignedBigInteger"></a>
#### `unsignedBigInteger()` {.collection-method}

`unsignedBigInteger` 方法會建立一個 `UNSIGNED BIGINT` 等效欄位：

    $table->unsignedBigInteger('votes');


<a name="column-method-unsignedInteger"></a>
#### `unsignedInteger()` {.collection-method}

`unsignedInteger` 方法會建立一個 `UNSIGNED INTEGER` 等效欄位：

    $table->unsignedInteger('votes');


<a name="column-method-unsignedMediumInteger"></a>
#### `unsignedMediumInteger()` {.collection-method}

`unsignedMediumInteger` 方法會建立一個 `UNSIGNED MEDIUMINT` 等效欄位：

    $table->unsignedMediumInteger('votes');


<a name="column-method-unsignedSmallInteger"></a>
#### `unsignedSmallInteger()` {.collection-method}

`unsignedSmallInteger` 方法會建立一個 `UNSIGNED SMALLINT` 等效欄位：

    $table->unsignedSmallInteger('votes');


<a name="column-method-unsignedTinyInteger"></a>
#### `unsignedTinyInteger()` {.collection-method}

`unsignedTinyInteger` 方法會建立一個 `UNSIGNED TINYINT` 等效欄位：

    $table->unsignedTinyInteger('votes');


<a name="column-method-ulidMorphs"></a>
#### `ulidMorphs()` {.collection-method}

`ulidMorphs` 方法是一個便捷方法，它會新增一個 `{column}_id` `CHAR(26)` 等效欄位和一個 `{column}_type` `VARCHAR` 等效欄位。

此方法旨在用於定義多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships) 所需的欄位，這些關聯使用 ULID 識別碼。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 欄位：

    $table->ulidMorphs('taggable');


<a name="column-method-uuidMorphs"></a>
#### `uuidMorphs()` {.collection-method}

`uuidMorphs` 方法是一個便捷方法，它會新增一個 `{column}_id` `CHAR(36)` 等效欄位和一個 `{column}_type` `VARCHAR` 等效欄位。

此方法旨在用於定義多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships) 所需的欄位，這些關聯使用 UUID 識別碼。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 欄位：

    $table->uuidMorphs('taggable');


<a name="column-method-ulid"></a>
#### `ulid()` {.collection-method}

`ulid` 方法會建立一個 `ULID` 等效欄位：

    $table->ulid('id');


<a name="column-method-uuid"></a>
#### `uuid()` {.collection-method}

`uuid` 方法會建立一個 `UUID` 等效欄位：

    $table->uuid('id');


<a name="column-method-vector"></a>
#### `vector()` {.collection-method}

`vector` 方法會建立一個 `vector` 等效欄位：

    $table->vector('embedding', dimensions: 100);


<a name="column-method-year"></a>
#### `year()` {.collection-method}

`year` 方法會建立一個 `YEAR` 等效欄位：

    $table->year('birth_year');

<a name="column-modifiers"></a>
### 欄位修飾語

除了上述的欄位類型之外，還有一些欄位「修飾語」可用於向資料庫資料表新增欄位。舉例來說，若要將欄位設為「可為空值」，您可以使用 `nullable` 方法：

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->string('email')->nullable();
    });

下表列出了所有可用的欄位修飾語。此列表不包含 [索引修飾語](#creating-indexes)：

<div class="overflow-auto">

| Modifier                            | Description                                                                                    |
| ----------------------------------- | ---------------------------------------------------------------------------------------------- |
| `->after('column')`                 | 將欄位放置在另一個欄位「之後」(MariaDB / MySQL)。                                               |
| `->autoIncrement()`                 | 將 `INTEGER` 欄位設定為自動遞增 (主鍵)。                                                       |
| `->charset('utf8mb4')`              | 為欄位指定字元集 (MariaDB / MySQL)。                                                           |
| `->collation('utf8mb4_unicode_ci')` | 為欄位指定排序規則。                                                                           |
| `->comment('my comment')`           | 為欄位新增註解 (MariaDB / MySQL / PostgreSQL)。                                                |
| `->default($value)`                 | 為欄位指定「預設」值。                                                                         |
| `->first()`                         | 將欄位放置在資料表「最前面」(MariaDB / MySQL)。                                                |
| `->from($integer)`                  | 設定自動遞增欄位的起始值 (MariaDB / MySQL / PostgreSQL)。                                      |
| `->invisible()`                     | 將欄位設為對 `SELECT *` 查詢「不可見」(MariaDB / MySQL)。                                       |
| `->nullable($value = true)`         | 允許 `NULL` 值插入到欄位中。                                                                   |
| `->storedAs($expression)`           | 建立一個儲存的生成欄位 (MariaDB / MySQL / PostgreSQL / SQLite)。                               |
| `->unsigned()`                      | 將 `INTEGER` 欄位設定為 `UNSIGNED` (MariaDB / MySQL)。                                          |
| `->useCurrent()`                    | 將 `TIMESTAMP` 欄位設定為使用 `CURRENT_TIMESTAMP` 作為預設值。                                 |
| `->useCurrentOnUpdate()`            | 將 `TIMESTAMP` 欄位設定為在記錄更新時使用 `CURRENT_TIMESTAMP` (MariaDB / MySQL)。              |
| `->virtualAs($expression)`          | 建立一個虛擬的生成欄位 (MariaDB / MySQL / SQLite)。                                            |
| `->generatedAs($expression)`        | 建立一個具有指定序列表選項的識別欄位 (PostgreSQL)。                                            |
| `->always()`                        | 定義序列值在識別欄位輸入時的優先順序 (PostgreSQL)。                                            |

</div>


<a name="default-expressions"></a>
#### 預設表達式

`default` 修飾語接受一個值或一個 `Illuminate\Database\Query\Expression` 實例。使用 `Expression` 實例將會阻止 Laravel 將值以引號包裝，並允許您使用資料庫特定的函式。在您需要為 JSON 欄位指定預設值時，這種情況特別有用：

    <?php

    use Illuminate\Support\Facades\Schema;
    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Database\Query\Expression;
    use Illuminate\Database\Migrations\Migration;

    return new class extends Migration
    {
        /**
         * Run the migrations.
         */
        public function up(): void
        {
            Schema::create('flights', function (Blueprint $table) {
                $table->id();
                $table->json('movies')->default(new Expression('(JSON_ARRAY())'));
                $table->timestamps();
            });
        }
    };

> [!WARNING]  
> 預設表達式的支援程度取決於您的資料庫驅動、資料庫版本和欄位類型。請參閱您的資料庫文件。


<a name="column-order"></a>
#### 欄位順序

在使用 MariaDB 或 MySQL 資料庫時，`after` 方法可用於在 Schema 中現有欄位之後新增欄位：

    $table->after('password', function (Blueprint $table) {
        $table->string('address_line1');
        $table->string('address_line2');
        $table->string('city');
    });


<a name="modifying-columns"></a>
### 修改欄位

`change` 方法允許您修改現有欄位的類型和屬性。例如，您可能希望增加 `string` 欄位的大小。為了展示 `change` 方法的實際應用，我們將 `name` 欄位的大小從 25 增加到 50。為此，我們只需定義欄位的新狀態，然後呼叫 `change` 方法：

    Schema::table('users', function (Blueprint $table) {
        $table->string('name', 50)->change();
    });

修改欄位時，您必須明確地包含所有要保留在欄位定義上的修飾語，任何遺漏的屬性都將被刪除。例如，若要保留 `unsigned`、`default` 和 `comment` 屬性，您在變更欄位時必須明確地呼叫每個修飾語：

    Schema::table('users', function (Blueprint $table) {
        $table->integer('votes')->unsigned()->default(1)->comment('my comment')->change();
    });

`change` 方法不會改變欄位的索引。因此，您在修改欄位時可以使用索引修飾語來明確地新增或刪除索引：

```php
// Add an index...
$table->bigIncrements('id')->primary()->change();

// Drop an index...
$table->char('postal_code', 10)->unique(false)->change();
```


<a name="renaming-columns"></a>
### 重新命名欄位

若要重新命名欄位，您可以使用 Schema Builder 提供的 `renameColumn` 方法：

    Schema::table('users', function (Blueprint $table) {
        $table->renameColumn('from', 'to');
    });


<a name="dropping-columns"></a>
### 刪除欄位

若要刪除欄位，您可以使用 Schema Builder 上的 `dropColumn` 方法：

    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('votes');
    });

您可以透過向 `dropColumn` 方法傳入一個欄位名稱陣列，來從資料表中刪除多個欄位：

    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn(['votes', 'avatar', 'location']);
    });


<a name="available-command-aliases"></a>
#### 可用指令別名

Laravel 提供了一些方便的方法，用於刪除常見類型的欄位。下表描述了這些方法：

<div class="overflow-auto">

| Command                             | Description                                           |
| ----------------------------------- | ----------------------------------------------------- |
| `$table->dropMorphs('morphable');`  | 刪除 `morphable_id` 和 `morphable_type` 欄位。         |
| `$table->dropRememberToken();`      | 刪除 `remember_token` 欄位。                          |
| `$table->dropSoftDeletes();`        | 刪除 `deleted_at` 欄位。                               |
| `$table->dropSoftDeletesTz();`      | `dropSoftDeletes()` 方法的別名。                      |
| `$table->dropTimestamps();`         | 刪除 `created_at` 和 `updated_at` 欄位。             |
| `$table->dropTimestampsTz();`       | `dropTimestamps()` 方法的別名。                       |

</div>

<a name="indexes"></a>
## 索引

<a name="creating-indexes"></a>
### 建立索引

Laravel 的 schema builder 支援多種索引類型。以下範例建立了一個新的 `email` 欄位，並指定其值應為唯一。若要建立索引，我們可以將 `unique` 方法鏈接到欄位定義上：

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->string('email')->unique();
    });

或者，您也可以在定義欄位後建立索引。為此，您應在 schema builder blueprint 上呼叫 `unique` 方法。此方法接受應接收唯一索引的欄位名稱：

    $table->unique('email');

您甚至可以向索引方法傳遞一個欄位陣列，以建立複合 (或組合) 索引：

    $table->index(['account_id', 'created_at']);

建立索引時，Laravel 會根據資料表、欄位名稱和索引類型自動生成索引名稱，但您可以向方法傳遞第二個參數，以自行指定索引名稱：

    $table->unique('email', 'unique_email');

<a name="available-index-types"></a>
#### 可用索引類型

Laravel 的 schema builder blueprint 類別提供了用於建立 Laravel 支援的各種索引類型的方法。每個索引方法都接受一個可選的第二個參數來指定索引的名稱。如果省略，名稱將根據資料表和用於索引的欄位名稱以及索引類型來推導。下表描述了所有可用的索引方法：

<div class="overflow-auto">

| Command                                          | Description                                                    |
| ------------------------------------------------ | -------------------------------------------------------------- |
| `$table->primary('id');`                         | 新增主鍵。                                            |
| `$table->primary(['id', 'parent_id']);`          | 新增複合鍵。                                           |
| `$table->unique('email');`                       | 新增唯一索引。                                           |
| `$table->index('state');`                        | 新增索引。                                                 |
| `$table->fullText('body');`                      | 新增全文索引 (MariaDB / MySQL / PostgreSQL)。         |
| `$table->fullText('body')->language('english');` | 新增指定語言的全文索引 (PostgreSQL)。 |
| `$table->spatialIndex('location');`              | 新增空間索引 (SQLite 除外)。                          |

</div>

<a name="renaming-indexes"></a>
### 重新命名索引

若要重新命名索引，您可以使用 schema builder blueprint 提供的 `renameIndex` 方法。此方法接受目前的索引名稱作為第一個參數，並接受所需的名稱作為第二個參數：

    $table->renameIndex('from', 'to')

<a name="dropping-indexes"></a>
### 刪除索引

若要刪除索引，您必須指定索引的名稱。預設情況下，Laravel 會根據資料表名稱、索引欄位名稱和索引類型自動分配索引名稱。以下是一些範例：

<div class="overflow-auto">

| Command                                                  | Description                                                 |
| -------------------------------------------------------- | ----------------------------------------------------------- |
| `$table->dropPrimary('users_id_primary');`               | 從「users」資料表刪除主鍵。                  |
| `$table->dropUnique('users_email_unique');`              | 從「users」資料表刪除唯一索引。                 |
| `$table->dropIndex('geo_state_index');`                  | 從「geo」資料表刪除基本索引。                    |
| `$table->dropFullText('posts_body_fulltext');`           | 從「posts」資料表刪除全文索引。              |
| `$table->dropSpatialIndex('geo_location_spatialindex');` | 從「geo」資料表刪除空間索引 (SQLite 除外)。 |

</div>

如果您向刪除索引的方法傳遞一個欄位陣列，慣用的索引名稱將根據資料表名稱、欄位和索引類型生成：

    Schema::table('geo', function (Blueprint $table) {
        $table->dropIndex(['state']); // Drops index 'geo_state_index'
    });

<a name="foreign-key-constraints"></a>
### 外鍵約束

Laravel 也支援建立外鍵約束，這些約束用於在資料庫層級強制實參完整性 (referential integrity)。例如，讓我們在 `posts` 資料表上定義一個 `user_id` 欄位，該欄位參考 `users` 資料表上的 `id` 欄位：

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('posts', function (Blueprint $table) {
        $table->unsignedBigInteger('user_id');

        $table->foreign('user_id')->references('id')->on('users');
    });

由於此語法相當冗長，Laravel 提供了額外的、更精簡的方法，這些方法利用慣例來提供更好的開發者體驗。當使用 `foreignId` 方法建立欄位時，上述範例可以改寫成：

    Schema::table('posts', function (Blueprint $table) {
        $table->foreignId('user_id')->constrained();
    });

`foreignId` 方法會建立一個等同於 `UNSIGNED BIGINT` 的欄位，而 `constrained` 方法將使用慣例來判斷所參考的資料表和欄位。如果您的資料表名稱不符合 Laravel 的慣例，您可以手動將其提供給 `constrained` 方法。此外，也可以指定要賦予所生成索引的名稱：

    Schema::table('posts', function (Blueprint $table) {
        $table->foreignId('user_id')->constrained(
            table: 'users', indexName: 'posts_user_id'
        );
    });

您還可以指定約束中「on delete (刪除時)」和「on update (更新時)」屬性的所需動作：

    $table->foreignId('user_id')
        ->constrained()
        ->onUpdate('cascade')
        ->onDelete('cascade');

這些動作也提供了另一種具表達力的語法：

<div class="overflow-auto">

| Method                        | Description                 |
| ----------------------------- | --------------------------- |
| `$table->cascadeOnUpdate();`  | 更新應連帶執行。            |
| `$table->restrictOnUpdate();` | 更新應受限。                |
| `$table->nullOnUpdate();`     | 更新應將外鍵值設為 null。   |
| `$table->noActionOnUpdate();` | 對更新不執行任何動作。      |
| `$table->cascadeOnDelete();`  | 刪除應連帶執行。            |
| `$table->restrictOnDelete();` | 刪除應受限。                |
| `$table->nullOnDelete();`     | 刪除應將外鍵值設為 null。   |
| `$table->noActionOnDelete();` | 如果存在子記錄，則阻止刪除。|

</div>

任何額外的 [欄位修飾語](#column-modifiers) 都必須在 `constrained` 方法之前呼叫：

    $table->foreignId('user_id')
        ->nullable()
        ->constrained();

<a name="dropping-foreign-keys"></a>
#### 刪除外鍵

為了刪除外鍵，您可以使用 `dropForeign` 方法，將要刪除的外鍵約束名稱作為引數傳入。外鍵約束與索引使用相同的命名慣例。換句話說，外鍵約束名稱是基於資料表名稱和約束中的欄位名稱，然後加上一個 "_foreign" 後綴：

    $table->dropForeign('posts_user_id_foreign');

此外，您可以將包含外鍵欄位名稱的陣列傳遞給 `dropForeign` 方法。該陣列將使用 Laravel 的約束命名慣例轉換為外鍵約束名稱：

    $table->dropForeign(['user_id']);

<a name="toggling-foreign-key-constraints"></a>
#### 切換外鍵約束

您可以使用以下方法在資料遷移中啟用或停用外鍵約束：

    Schema::enableForeignKeyConstraints();

    Schema::disableForeignKeyConstraints();

    Schema::withoutForeignKeyConstraints(function () {
        // Constraints disabled within this closure...
    });

> [!WARNING]  
> SQLite 預設會停用外鍵約束。當使用 SQLite 時，請確保在資料遷移中嘗試建立外鍵之前，在您的資料庫 [設定檔](/docs/{{version}}/database#configuration) 中「啟用外鍵支援」。

<a name="events"></a>
## 事件

為了方便，每個資料遷移操作都將會觸發一個 [事件](/docs/{{version}}/events)。以下所有事件都繼承基礎的 `Illuminate\Database\Events\MigrationEvent` 類別：

<div class="overflow-auto">

| 類別                                             | 描述                                       |
| ------------------------------------------------ | ------------------------------------------ |
| `Illuminate\Database\Events\MigrationsStarted`   | 即將執行一批資料遷移。                     |
| `Illuminate\Database\Events\MigrationsEnded`     | 一批資料遷移已執行完畢。                   |
| `Illuminate\Database\Events\MigrationStarted`    | 即將執行單一資料遷移。                     |
| `Illuminate\Database\Events\MigrationEnded`      | 單一資料遷移已執行完畢。                   |
| `Illuminate\Database\Events\NoPendingMigrations` | 資料遷移命令未找到待處理的資料遷移。       |
| `Illuminate\Database\Events\SchemaDumped`        | 資料庫結構描述傾印已完成。                 |
| `Illuminate\Database\Events\SchemaLoaded`        | 現有資料庫結構描述傾印已載入。             |

</div>