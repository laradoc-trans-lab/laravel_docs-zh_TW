# 資料庫：遷移 (Migrations)

- [簡介](#introduction)
- [生成遷移](#generating-migrations)
    - [壓縮遷移](#squashing-migrations)
- [遷移結構](#migration-structure)
- [執行遷移](#running-migrations)
    - [回溯遷移](#rolling-back-migrations)
- [資料表](#tables)
    - [建立資料表](#creating-tables)
    - [更新資料表](#updating-tables)
    - [重新命名 / 刪除資料表](#renaming-and-dropping-tables)
- [欄位](#columns)
    - [建立欄位](#creating-columns)
    - [可用的欄位類型](#available-column-types)
    - [欄位修飾符](#column-modifiers)
    - [修改欄位](#modifying-columns)
    - [重新命名欄位](#renaming-columns)
    - [刪除欄位](#dropping-columns)
- [索引](#indexes)
    - [建立索引](#creating-indexes)
    - [重新命名索引](#renaming-indexes)
    - [刪除索引](#dropping-indexes)
    - [外部鍵約束](#foreign-key-constraints)
- [事件](#events)

<a name="introduction"></a>
## 簡介

遷移 (Migrations) 就像是資料庫的版本控制，讓你的團隊能夠定義並共用應用程式的資料庫綱要 (schema) 定義。如果你曾經在從版本控制拉取變更後，必須告訴隊友手動為他們的本地資料庫綱要新增一個欄位，那麼你已經遇到過資料庫遷移所解決的問題。

Laravel 的 `Schema` [Facade](/docs/{{version}}/facades) 提供了資料庫無關的支援，用於在所有 Laravel 支援的資料庫系統中建立和操作資料表。通常，遷移會使用這個 Facade 來建立和修改資料庫資料表和欄位。

<a name="generating-migrations"></a>
## 生成遷移

你可以使用 `make:migration` [Artisan 命令](/docs/{{version}}/artisan)來生成資料庫遷移。新的遷移將會被放置在你的 `database/migrations` 目錄中。每個遷移的檔案名稱都包含一個時間戳記，讓 Laravel 能夠判斷遷移的執行順序：

```shell
php artisan make:migration create_flights_table
```

Laravel 會根據遷移的名稱來嘗試猜測資料表的名稱，以及該遷移是否會建立一個新的資料表。如果 Laravel 能夠從遷移名稱中判斷出資料表名稱，Laravel 會預先填入生成的遷移檔案中指定的資料表。否則，你可能需要手動在遷移檔案中指定資料表。

如果你想為生成的遷移指定一個自訂路徑，可以在執行 `make:migration` 命令時使用 `--path` 選項。給定的路徑應該是相對於你應用程式的基礎路徑。

> [!NOTE]
> 遷移的樣板 (stub) 可以透過 [樣板發佈](/docs/{{version}}/artisan#stub-customization) 來進行自訂。

<a name="squashing-migrations"></a>
### 壓縮遷移

隨著你建構應用程式，你可能會隨著時間累積越來越多的遷移。這可能導致你的 `database/migrations` 目錄變得臃腫，可能包含數百個遷移。如果你願意，你可以將你的遷移「壓縮」成一個單一的 SQL 檔案。要開始，請執行 `schema:dump` 命令：

```shell
php artisan schema:dump

# 傾印目前的資料庫綱要並清除所有現有的遷移...
php artisan schema:dump --prune
```

當你執行這個命令時，Laravel 會在你的應用程式 `database/schema` 目錄中寫入一個「綱要」檔案。綱要檔案的名稱將與資料庫連線相對應。現在，當你嘗試遷移你的資料庫且沒有其他遷移被執行時，Laravel 會首先執行你正在使用的資料庫連線的綱要檔案中的 SQL 語句。在執行完綱要檔案的 SQL 語句後，Laravel 會執行任何未包含在綱要傾印中的剩餘遷移。

如果你的應用程式測試使用與你通常在本地開發期間使用的資料庫連線不同的連線，你應該確保你已經使用該資料庫連線傾印了一個綱要檔案，以便你的測試能夠建構你的資料庫。你可能希望在傾印完你通常在本地開發期間使用的資料庫連線後執行此操作：

```shell
php artisan schema:dump
php artisan schema:dump --database=testing --prune
```

你應該將你的資料庫綱要檔案提交到版本控制，以便團隊中其他新的開發人員可以快速建立你應用程式的初始資料庫結構。

> [!WARNING]
> 遷移壓縮僅適用於 MariaDB、MySQL、PostgreSQL 和 SQLite 資料庫，並利用資料庫的命令列客戶端。

<a name="migration-structure"></a>
## 遷移結構

一個遷移類別包含兩個方法：`up` 和 `down`。`up` 方法用於向資料庫新增資料表、欄位或索引，而 `down` 方法應該反轉 `up` 方法執行的操作。

在這兩個方法中，你都可以使用 Laravel 的綱要建構器 (schema builder) 來表達性地建立和修改資料表。要了解 `Schema` 建構器上所有可用的方法，請[查閱其說明文件](#creating-tables)。例如，以下遷移建立了一個 `flights` 資料表：

```php
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
```

<a name="setting-the-migration-connection"></a>
#### 設定遷移連線

如果你的遷移將與應用程式的預設資料庫連線以外的資料庫連線互動，你應該設定遷移的 `$connection` 屬性：

```php
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
```

<a name="skipping-migrations"></a>
#### 跳過遷移

有時，一個遷移可能旨在支援尚未啟用的功能，而你暫時不希望它執行。在這種情況下，你可以在遷移上定義一個 `shouldRun` 方法。如果 `shouldRun` 方法回傳 `false`，該遷移將會被跳過：

```php
use App\Models\Flights;
use Laravel\Pennant\Feature;

/**
 * Determine if this migration should run.
 */
public function shouldRun(): bool
{
    return Feature::active(Flights::class);
}
```

<a name="running-migrations"></a>
## 執行遷移

要執行所有未完成的遷移，請執行 `migrate` Artisan 命令：

```shell
php artisan migrate
```

如果你想查看目前已執行了哪些遷移，可以使用 `migrate:status` Artisan 命令：

```shell
php artisan migrate:status
```

如果你想查看遷移將執行的 SQL 語句，但不想實際執行它們，可以為 `migrate` 命令提供 `--pretend` 旗標：

```shell
php artisan migrate --pretend
```

#### 隔離遷移執行

如果你在多個伺服器上部署應用程式，並將遷移作為部署過程的一部分執行，你可能不希望兩個伺服器同時嘗試遷移資料庫。為避免這種情況，你可以在呼叫 `migrate` 命令時使用 `isolated` 選項。

當提供 `isolated` 選項時，Laravel 會在嘗試執行遷移之前，使用你應用程式的快取驅動程式取得一個原子鎖。在該鎖被持有的情況下，所有其他嘗試執行 `migrate` 命令的請求都不會執行；然而，該命令仍將以成功的結束狀態碼退出：

```shell
php artisan migrate --isolated
```

> [!WARNING]
> 要使用此功能，你的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為你應用程式的預設快取驅動程式。此外，所有伺服器必須與同一個中央快取伺服器通訊。

<a name="forcing-migrations-to-run-in-production"></a>
#### 強制在正式環境中執行遷移

某些遷移操作是破壞性的，這意味著它們可能會導致你丟失資料。為了保護你免於在正式環境資料庫上執行這些命令，在命令執行前會提示你確認。要強制命令在沒有提示的情況下執行，請使用 `--force` 旗標：

```shell
php artisan migrate --force
```

<a name="rolling-back-migrations"></a>
### 回溯遷移

要回溯最近一次的遷移操作，你可以使用 `rollback` Artisan 命令。此命令會回溯最近一個「批次」的遷移，其中可能包含多個遷移檔案：

```shell
php artisan migrate:rollback
```

你可以透過向 `rollback` 命令提供 `step` 選項來回溯有限數量的遷移。例如，以下命令將回溯最近的五個遷移：

```shell
php artisan migrate:rollback --step=5
```

你可以透過向 `rollback` 命令提供 `batch` 選項來回溯特定「批次」的遷移，其中 `batch` 選項對應於你應用程式 `migrations` 資料表中的批次值。例如，以下命令將回溯批次三中的所有遷移：

```shell
php artisan migrate:rollback --batch=3
```

如果你想查看遷移將執行的 SQL 語句，但不想實際執行它們，可以為 `migrate:rollback` 命令提供 `--pretend` 旗標：

```shell
php artisan migrate:rollback --pretend
```

`migrate:reset` 命令將回溯你應用程式的所有遷移：

```shell
php artisan migrate:reset
```

<a name="roll-back-migrate-using-a-single-command"></a>
#### 使用單一命令回溯並遷移

`migrate:refresh` 命令將回溯你所有的遷移，然後執行 `migrate` 命令。此命令有效地重新建立你的整個資料庫：

```shell
php artisan migrate:refresh

# 重新整理資料庫並執行所有資料庫填充...
php artisan migrate:refresh --seed
```

你可以透過向 `refresh` 命令提供 `step` 選項來回溯並重新遷移有限數量的遷移。例如，以下命令將回溯並重新遷移最近的五個遷移：

```shell
php artisan migrate:refresh --step=5
```

<a name="drop-all-tables-migrate"></a>
#### 刪除所有資料表並遷移

`migrate:fresh` 命令將從資料庫中刪除所有資料表，然後執行 `migrate` 命令：

```shell
php artisan migrate:fresh

php artisan migrate:fresh --seed
```

預設情況下，`migrate:fresh` 命令只會從預設的資料庫連線中刪除資料表。但是，你可以使用 `--database` 選項來指定應該遷移的資料庫連線。資料庫連線名稱應該與你應用程式的 `database` [設定檔](/docs/{{version}}/configuration) 中定義的連線相對應：

```shell
php artisan migrate:fresh --database=admin
```

> [!WARNING]
> `migrate:fresh` 命令將刪除所有資料庫資料表，無論其前綴為何。在與其他應用程式共用資料庫的開發環境中，應謹慎使用此命令。

<a name="tables"></a>
## 資料表

<a name="creating-tables"></a>
### 建立資料表

要建立新的資料庫資料表，請使用 `Schema` Facade 上的 `create` 方法。`create` 方法接受兩個參數：第一個是資料表的名稱，第二個是一個閉包 (closure)，它接收一個 `Blueprint` 物件，可用於定義新的資料表：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email');
    $table->timestamps();
});
```

建立資料表時，你可以使用綱要建構器的任何[欄位方法](#creating-columns)來定義資料表的欄位。

<a name="determining-table-column-existence"></a>
#### 判斷資料表 / 欄位是否存在

你可以使用 `hasTable`、`hasColumn` 和 `hasIndex` 方法來判斷資料表、欄位或索引是否存在：

```php
if (Schema::hasTable('users')) {
    // "users" 資料表存在...
}

if (Schema::hasColumn('users', 'email')) {
    // "users" 資料表存在且有一個 "email" 欄位...
}

if (Schema::hasIndex('users', ['email'], 'unique')) {
    // "users" 資料表存在且在 "email" 欄位上有一個唯一索引...
}
```

<a name="database-connection-table-options"></a>
#### 資料庫連線與資料表選項

如果你想在不是應用程式預設連線的資料庫連線上執行綱要操作，請使用 `connection` 方法：

```php
Schema::connection('sqlite')->create('users', function (Blueprint $table) {
    $table->id();
});
```

此外，還可以利用其他一些屬性和方法來定義資料表建立的其他方面。當使用 MariaDB 或 MySQL 時，`engine` 屬性可用於指定資料表的儲存引擎：

```php
Schema::create('users', function (Blueprint $table) {
    $table->engine('InnoDB');

    // ...
});
```

當使用 MariaDB 或 MySQL 時，`charset` 和 `collation` 屬性可用於指定所建立資料表的字元集和校對規則：

```php
Schema::create('users', function (Blueprint $table) {
    $table->charset('utf8mb4');
    $table->collation('utf8mb4_unicode_ci');

    // ...
});
```

`temporary` 方法可用於指示資料表應該是「暫時的」。暫時資料表僅對目前連線的資料庫會話可見，並在連線關閉時自動刪除：

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->temporary();

    // ...
});
```

如果你想為資料庫資料表新增「註解」，可以在資料表實例上呼叫 `comment` 方法。目前，資料表註解僅受 MariaDB、MySQL 和 PostgreSQL 支援：

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->comment('Business calculations');

    // ...
});
```

<a name="updating-tables"></a>
### 更新資料表

`Schema` Facade 上的 `table` 方法可用於更新現有的資料表。與 `create` 方法一樣，`table` 方法接受兩個參數：資料表的名稱和一個閉包，該閉包接收一個 `Blueprint` 實例，你可以使用它向資料表新增欄位或索引：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->integer('votes');
});
```

<a name="renaming-and-dropping-tables"></a>
### 重新命名 / 刪除資料表

要重新命名現有的資料庫資料表，請使用 `rename` 方法：

```php
use Illuminate\Support\Facades\Schema;

Schema::rename($from, $to);
```

要刪除現有的資料表，你可以使用 `drop` 或 `dropIfExists` 方法：

```php
Schema::drop('users');

Schema::dropIfExists('users');
```

<a name="renaming-tables-with-foreign-keys"></a>
#### 重新命名帶有外部鍵的資料表

在重新命名資料表之前，你應該驗證資料表上的任何外部鍵約束在你的遷移檔案中是否具有明確的名稱，而不是讓 Laravel 分配基於慣例的名稱。否則，外部鍵約束名稱將會指向舊的資料表名稱。

<a name="columns"></a>
## 欄位

<a name="creating-columns"></a>
### 建立欄位

`Schema` Facade 上的 `table` 方法可用於更新現有的資料表。與 `create` 方法一樣，`table` 方法接受兩個參數：資料表的名稱和一個閉包，該閉包接收一個 `Illuminate\Database\Schema\Blueprint` 實例，你可以使用它向資料表新增欄位：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->integer('votes');
});
```

<a name="available-column-types"></a>
### 可用的欄位類型

綱要建構器 Blueprint 提供了多種方法，這些方法對應於你可以新增到資料庫資料表中的不同欄位類型。下表列出了所有可用的方法：

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
#### 數值類型

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
#### 物件與 JSON 類型

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

`bigIncrements` 方法建立一個自動遞增的 `UNSIGNED BIGINT` (主鍵) 等效欄位：

```php
$table->bigIncrements('id');
```

<a name="column-method-bigInteger"></a>
#### `bigInteger()` {.collection-method}

`bigInteger` 方法建立一個 `BIGINT` 等效欄位：

```php
$table->bigInteger('votes');
```

<a name="column-method-binary"></a>
#### `binary()` {.collection-method}

`binary` 方法建立一個 `BLOB` 等效欄位：

```php
$table->binary('photo');
```

當使用 MySQL、MariaDB 或 SQL Server 時，你可以傳遞 `length` 和 `fixed` 參數來建立 `VARBINARY` 或 `BINARY` 等效欄位：

```php
$table->binary('data', length: 16); // VARBINARY(16)

$table->binary('data', length: 16, fixed: true); // BINARY(16)
```

<a name="column-method-boolean"></a>
#### `boolean()` {.collection-method}

`boolean` 方法建立一個 `BOOLEAN` 等效欄位：

```php
$table->boolean('confirmed');
```

<a name="column-method-char"></a>
#### `char()` {.collection-method}

`char` 方法建立一個指定長度的 `CHAR` 等效欄位：

```php
$table->char('name', length: 100);
```

<a name="column-method-dateTimeTz"></a>
#### `dateTimeTz()` {.collection-method}

`dateTimeTz` 方法建立一個 `DATETIME` (帶時區) 等效欄位，並可選地指定小數秒精度：

```php
$table->dateTimeTz('created_at', precision: 0);
```

<a name="column-method-dateTime"></a>
#### `dateTime()` {.collection-method}

`dateTime` 方法建立一個 `DATETIME` 等效欄位，並可選地指定小數秒精度：

```php
$table->dateTime('created_at', precision: 0);
```

<a name="column-method-date"></a>
#### `date()` {.collection-method}

`date` 方法建立一個 `DATE` 等效欄位：

```php
$table->date('created_at');
```

<a name="column-method-decimal"></a>
#### `decimal()` {.collection-method}

`decimal` 方法建立一個 `DECIMAL` 等效欄位，並指定給定的精度 (總位數) 和小數位數 (小數點後的位數)：

```php
$table->decimal('amount', total: 8, places: 2);
```

<a name="column-method-double"></a>
#### `double()` {.collection-method}

`double` 方法建立一個 `DOUBLE` 等效欄位：

```php
$table->double('amount');
```

<a name="column-method-enum"></a>
#### `enum()` {.collection-method}

`enum` 方法建立一個 `ENUM` 等效欄位，並指定給定的有效值：

```php
$table->enum('difficulty', ['easy', 'hard']);
```

當然，你可以使用 `Enum::cases()` 方法，而不是手動定義允許值的陣列：

```php
use App\Enums\Difficulty;

$table->enum('difficulty', Difficulty::cases());
```

<a name="column-method-float"></a>
#### `float()` {.collection-method}

`float` 方法建立一個 `FLOAT` 等效欄位，並指定給定的精度：

```php
$table->float('amount', precision: 53);
```

<a name="column-method-foreignId"></a>
#### `foreignId()` {.collection-method}

`foreignId` 方法建立一個 `UNSIGNED BIGINT` 等效欄位：

```php
$table->foreignId('user_id');
```

<a name="column-method-foreignIdFor"></a>
#### `foreignIdFor()` {.collection-method}

`foreignIdFor` 方法為給定的模型類別新增一個 `{column}_id` 等效欄位。該欄位類型將是 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`，具體取決於模型鍵類型：

```php
$table->foreignIdFor(User::class);
```

<a name="column-method-foreignUlid"></a>
#### `foreignUlid()` {.collection-method}

`foreignUlid` 方法建立一個 `ULID` 等效欄位：

```php
$table->foreignUlid('user_id');
```

<a name="column-method-foreignUuid"></a>
#### `foreignUuid()` {.collection-method}

`foreignUuid` 方法建立一個 `UUID` 等效欄位：

```php
$table->foreignUuid('user_id');
```

<a name="column-method-geography"></a>
#### `geography()` {.collection-method}

`geography` 方法建立一個 `GEOGRAPHY` 等效欄位，並指定給定的空間類型和 SRID (空間參考系統識別碼)：

```php
$table->geography('coordinates', subtype: 'point', srid: 4326);
```

> [!NOTE]
> 對空間類型的支援取決於你的資料庫驅動程式。請參閱你的資料庫說明文件。如果你的應用程式使用 PostgreSQL 資料庫，你必須在使用 `geography` 方法之前安裝 [PostGIS](https://postgis.net) 擴充功能。

<a name="column-method-geometry"></a>
#### `geometry()` {.collection-method}

`geometry` 方法建立一個 `GEOMETRY` 等效欄位，並指定給定的空間類型和 SRID (空間參考系統識別碼)：

```php
$table->geometry('positions', subtype: 'point', srid: 0);
```

> [!NOTE]
> 對空間類型的支援取決於你的資料庫驅動程式。請參閱你的資料庫說明文件。如果你的應用程式使用 PostgreSQL 資料庫，你必須在使用 `geometry` 方法之前安裝 [PostGIS](https://postgis.net) 擴充功能。

<a name="column-method-id"></a>
#### `id()` {.collection-method}

`id` 方法是 `bigIncrements` 方法的別名。預設情況下，該方法將建立一個 `id` 欄位；但是，如果你想為該欄位指定不同的名稱，可以傳遞一個欄位名稱：

```php
$table->id();
```

<a name="column-method-increments"></a>
#### `increments()` {.collection-method}

`increments` 方法建立一個自動遞增的 `UNSIGNED INTEGER` 等效欄位作為主鍵：

```php
$table->increments('id');
```

<a name="column-method-integer"></a>
#### `integer()` {.collection-method}

`integer` 方法建立一個 `INTEGER` 等效欄位：

```php
$table->integer('votes');
```

<a name="column-method-ipAddress"></a>
#### `ipAddress()` {.collection-method}

`ipAddress` 方法建立一個 `VARCHAR` 等效欄位：

```php
$table->ipAddress('visitor');
```

當使用 PostgreSQL 時，將會建立一個 `INET` 欄位。

<a name="column-method-json"></a>
#### `json()` {.collection-method}

`json` 方法建立一個 `JSON` 等效欄位：

```php
$table->json('options');
```

當使用 SQLite 時，將會建立一個 `TEXT` 欄位。

<a name="column-method-jsonb"></a>
#### `jsonb()` {.collection-method}

`jsonb` 方法建立一個 `JSONB` 等效欄位：

```php
$table->jsonb('options');
```

當使用 SQLite 時，將會建立一個 `TEXT` 欄位。

<a name="column-method-longText"></a>
#### `longText()` {.collection-method}

`longText` 方法建立一個 `LONGTEXT` 等效欄位：

```php
$table->longText('description');
```

當使用 MySQL 或 MariaDB 時，你可以為該欄位應用 `binary` 字元集，以建立一個 `LONGBLOB` 等效欄位：

```php
$table->longText('data')->charset('binary'); // LONGBLOB
```

<a name="column-method-macAddress"></a>
#### `macAddress()` {.collection-method}

`macAddress` 方法建立一個用於儲存 MAC 位址的欄位。某些資料庫系統，例如 PostgreSQL，具有專門用於此類資料的欄位類型。其他資料庫系統將使用字串等效欄位：

```php
$table->macAddress('device');
```

<a name="column-method-mediumIncrements"></a>
#### `mediumIncrements()` {.collection-method}

`mediumIncrements` 方法建立一個自動遞增的 `UNSIGNED MEDIUMINT` 等效欄位作為主鍵：

```php
$table->mediumIncrements('id');
```

<a name="column-method-mediumInteger"></a>
#### `mediumInteger()` {.collection-method}

`mediumInteger` 方法建立一個 `MEDIUMINT` 等效欄位：

```php
$table->mediumInteger('votes');
```

<a name="column-method-mediumText"></a>
#### `mediumText()` {.collection-method}

`mediumText` 方法建立一個 `MEDIUMTEXT` 等效欄位：

```php
$table->mediumText('description');
```

當使用 MySQL 或 MariaDB 時，你可以為該欄位應用 `binary` 字元集，以建立一個 `MEDIUMBLOB` 等效欄位：

```php
$table->mediumText('data')->charset('binary'); // MEDIUMBLOB
```

<a name="column-method-morphs"></a>
#### `morphs()` {.collection-method}

`morphs` 方法是一個方便的方法，它新增一個 `{column}_id` 等效欄位和一個 `{column}_type` `VARCHAR` 等效欄位。`{column}_id` 的欄位類型將是 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`，具體取決於模型鍵類型。

此方法旨在用於定義多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships) 所需的欄位。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 欄位：

```php
$table->morphs('taggable');
```

<a name="column-method-nullableMorphs"></a>
#### `nullableMorphs()` {.collection-method}

此方法類似於 [morphs](#column-method-morphs) 方法；但是，所建立的欄位將是「可為空值」的：

```php
$table->nullableMorphs('taggable');
```

<a name="column-method-nullableUlidMorphs"></a>
#### `nullableUlidMorphs()` {.collection-method}

此方法類似於 [ulidMorphs](#column-method-ulidMorphs) 方法；但是，所建立的欄位將是「可為空值」的：

```php
$table->nullableUlidMorphs('taggable');
```

<a name="column-method-nullableUuidMorphs"></a>
#### `nullableUuidMorphs()` {.collection-method}

此方法類似於 [uuidMorphs](#column-method-uuidMorphs) 方法；但是，所建立的欄位將是「可為空值」的：

```php
$table->nullableUuidMorphs('taggable');
```

<a name="column-method-rememberToken"></a>
#### `rememberToken()` {.collection-method}

`rememberToken` 方法建立一個可為空值的 `VARCHAR(100)` 等效欄位，用於儲存目前的「記住我」[認證權杖](/docs/{{version}}/authentication#remembering-users)：

```php
$table->rememberToken();
```

<a name="column-method-set"></a>
#### `set()` {.collection-method}

`set` 方法建立一個 `SET` 等效欄位，並指定給定的有效值列表：

```php
$table->set('flavors', ['strawberry', 'vanilla']);
```

<a name="column-method-smallIncrements"></a>
#### `smallIncrements()` {.collection-method}

`smallIncrements` 方法建立一個自動遞增的 `UNSIGNED SMALLINT` 等效欄位作為主鍵：

```php
$table->smallIncrements('id');
```

<a name="column-method-smallInteger"></a>
#### `smallInteger()` {.collection-method}

`smallInteger` 方法建立一個 `SMALLINT` 等效欄位：

```php
$table->smallInteger('votes');
```

<a name="column-method-softDeletesTz"></a>
#### `softDeletesTz()` {.collection-method}

`softDeletesTz` 方法新增一個可為空值的 `deleted_at` `TIMESTAMP` (帶時區) 等效欄位，並可選地指定小數秒精度。此欄位旨在儲存 Eloquent「軟刪除」功能所需的 `deleted_at` 時間戳記：

```php
$table->softDeletesTz('deleted_at', precision: 0);
```

<a name="column-method-softDeletes"></a>
#### `softDeletes()` {.collection-method}

`softDeletes` 方法新增一個可為空值的 `deleted_at` `TIMESTAMP` 等效欄位，並可選地指定小數秒精度。此欄位旨在儲存 Eloquent「軟刪除」功能所需的 `deleted_at` 時間戳記：

```php
$table->softDeletes('deleted_at', precision: 0);
```

<a name="column-method-string"></a>
#### `string()` {.collection-method}

`string` 方法建立一個指定長度的 `VARCHAR` 等效欄位：

```php
$table->string('name', length: 100);
```

<a name="column-method-text"></a>
#### `text()` {.collection-method}

`text` 方法建立一個 `TEXT` 等效欄位：

```php
$table->text('description');
```

當使用 MySQL 或 MariaDB 時，你可以為該欄位應用 `binary` 字元集，以建立一個 `BLOB` 等效欄位：

```php
$table->text('data')->charset('binary'); // BLOB
```

<a name="column-method-timeTz"></a>
#### `timeTz()` {.collection-method}

`timeTz` 方法建立一個 `TIME` (帶時區) 等效欄位，並可選地指定小數秒精度：

```php
$table->timeTz('sunrise', precision: 0);
```

<a name="column-method-time"></a>
#### `time()` {.collection-method}

`time` 方法建立一個 `TIME` 等效欄位，並可選地指定小數秒精度：

```php
$table->time('sunrise', precision: 0);
```

<a name="column-method-timestampTz"></a>
#### `timestampTz()` {.collection-method}

`timestampTz` 方法建立一個 `TIMESTAMP` (帶時區) 等效欄位，並可選地指定小數秒精度：

```php
$table->timestampTz('added_at', precision: 0);
```

<a name="column-method-timestamp"></a>
#### `timestamp()` {.collection-method}

`timestamp` 方法建立一個 `TIMESTAMP` 等效欄位，並可選地指定小數秒精度：

```php
$table->timestamp('added_at', precision: 0);
```

<a name="column-method-timestampsTz"></a>
#### `timestampsTz()` {.collection-method}

`timestampsTz` 方法建立 `created_at` 和 `updated_at` `TIMESTAMP` (帶時區) 等效欄位，並可選地指定小數秒精度：

```php
$table->timestampsTz(precision: 0);
```

<a name="column-method-timestamps"></a>
#### `timestamps()` {.collection-method}

`timestamps` 方法建立 `created_at` 和 `updated_at` `TIMESTAMP` 等效欄位，並可選地指定小數秒精度：

```php
$table->timestamps(precision: 0);
```

<a name="column-method-tinyIncrements"></a>
#### `tinyIncrements()` {.collection-method}

`tinyIncrements` 方法建立一個自動遞增的 `UNSIGNED TINYINT` 等效欄位作為主鍵：

```php
$table->tinyIncrements('id');
```

<a name="column-method-tinyInteger"></a>
#### `tinyInteger()` {.collection-method}

`tinyInteger` 方法建立一個 `TINYINT` 等效欄位：

```php
$table->tinyInteger('votes');
```

<a name="column-method-tinyText"></a>
#### `tinyText()` {.collection-method}

`tinyText` 方法建立一個 `TINYTEXT` 等效欄位：

```php
$table->tinyText('notes');
```

當使用 MySQL 或 MariaDB 時，你可以為該欄位應用 `binary` 字元集，以建立一個 `TINYBLOB` 等效欄位：

```php
$table->tinyText('data')->charset('binary'); // TINYBLOB
```

<a name="column-method-unsignedBigInteger"></a>
#### `unsignedBigInteger()` {.collection-method}

`unsignedBigInteger` 方法建立一個 `UNSIGNED BIGINT` 等效欄位：

```php
$table->unsignedBigInteger('votes');
```

<a name="column-method-unsignedInteger"></a>
#### `unsignedInteger()` {.collection-method}

`unsignedInteger` 方法建立一個 `UNSIGNED INTEGER` 等效欄位：

```php
$table->unsignedInteger('votes');
```

<a name="column-method-unsignedMediumInteger"></a>
#### `unsignedMediumInteger()` {.collection-method}

`unsignedMediumInteger` 方法建立一個 `UNSIGNED MEDIUMINT` 等效欄位：

```php
$table->unsignedMediumInteger('votes');
```

<a name="column-method-unsignedSmallInteger"></a>
#### `unsignedSmallInteger()` {.collection-method}

`unsignedSmallInteger` 方法建立一個 `UNSIGNED SMALLINT` 等效欄位：

```php
$table->unsignedSmallInteger('votes');
```

<a name="column-method-unsignedTinyInteger"></a>
#### `unsignedTinyInteger()` {.collection-method}

`unsignedTinyInteger` 方法建立一個 `UNSIGNED TINYINT` 等效欄位：

```php
$table->unsignedTinyInteger('votes');
```

<a name="column-method-ulidMorphs"></a>
#### `ulidMorphs()` {.collection-method}

`ulidMorphs` 方法是一個方便的方法，它新增一個 `{column}_id` `CHAR(26)` 等效欄位和一個 `{column}_type` `VARCHAR` 等效欄位。

此方法旨在用於定義使用 ULID 識別碼的多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships) 所需的欄位。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 欄位：

```php
$table->ulidMorphs('taggable');
```

<a name="column-method-uuidMorphs"></a>
#### `uuidMorphs()` {.collection-method}

`uuidMorphs` 方法是一個方便的方法，它新增一個 `{column}_id` `CHAR(36)` 等效欄位和一個 `{column}_type` `VARCHAR` 等效欄位。

此方法旨在用於定義使用 UUID 識別碼的多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships) 所需的欄位。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 欄位：

```php
$table->uuidMorphs('taggable');
```

<a name="column-method-ulid"></a>
#### `ulid()` {.collection-method}

`ulid` 方法建立一個 `ULID` 等效欄位：

```php
$table->ulid('id');
```

<a name="column-method-uuid"></a>
#### `uuid()` {.collection-method}

`uuid` 方法建立一個 `UUID` 等效欄位：

```php
$table->uuid('id');
```

<a name="column-method-vector"></a>
#### `vector()` {.collection-method}

`vector` 方法建立一個 `vector` 等效欄位：

```php
$table->vector('embedding', dimensions: 100);
```

<a name="column-method-year"></a>
#### `year()` {.collection-method}

`year` 方法建立一個 `YEAR` 等效欄位：

```php
$table->year('birth_year');
```

<a name="column-modifiers"></a>
### 欄位修飾符

除了上面列出的欄位類型之外，還有幾個欄位「修飾符」可以在向資料庫資料表新增欄位時使用。例如，要使欄位「可為空值」，你可以使用 `nullable` 方法：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->string('email')->nullable();
});
```

下表包含所有可用的欄位修飾符。此列表不
