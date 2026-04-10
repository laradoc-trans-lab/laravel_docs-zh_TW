# 資料庫：遷移

- [簡介](#introduction)
- [產生遷移](#generating-migrations)
    - [壓縮遷移](#squashing-migrations)
- [遷移結構](#migration-structure)
- [執行遷移](#running-migrations)
    - [回滾遷移](#rolling-back-migrations)
- [資料表](#tables)
    - [建立資料表](#creating-tables)
    - [更新資料表](#updating-tables)
    - [重新命名 / 刪除資料表](#renaming-and-dropping-tables)
- [欄位](#columns)
    - [建立欄位](#creating-columns)
    - [可用的欄位類型](#available-column-types)
    - [欄位修改器](#column-modifiers)
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

遷移就像是資料庫的版本控制，讓您的團隊能夠定義並共享應用程式的資料庫結構定義。如果您曾經在從原始碼控制拉取變更後，必須告知隊友手動在本地資料庫結構中新增欄位，那麼您就遇過資料庫遷移所解決的問題。

Laravel `Schema` [facade](/docs/{{version}}/facades) 提供了與資料庫無關的支援，可用於在所有 Laravel 支援的資料庫系統中建立與操作資料表。通常，遷移會使用這個 facade 來建立與修改資料庫資料表及欄位。


<a name="generating-migrations"></a>
## 產生遷移

您可以使用 `make:migration` [Artisan 命令](/docs/{{version}}/artisan) 來產生資料庫遷移。新的遷移將被放置在您的 `database/migrations` 目錄中。每個遷移檔案名稱都包含一個時間戳記，讓 Laravel 能決定遷移的執行順序：

```shell
php artisan make:migration create_flights_table
```

Laravel 會利用遷移的名稱來嘗試推測資料表的名稱，以及該遷移是否將建立新資料表。如果 Laravel 能從遷移名稱中確定資料表名稱，Laravel 會在產生的遷移檔案中預先填入指定的資料表。否則，您只需在遷移檔案中手動指定資料表即可。

如果您想為產生的遷移指定自定義路徑，可以在執行 `make:migration` 命令時使用 `--path` 選項。給定的路徑應相對於您應用程式的基礎路徑。

> [!NOTE]
> 遷移存根 (stubs) 可以透過 [存根發布](/docs/{{version}}/artisan#stub-customization) 進行自定義。


<a name="squashing-migrations"></a>
### 壓縮遷移

隨著您開發應用程式，時間久了可能會累積越來越多的遷移。這會導致您的 `database/migrations` 目錄變得臃腫，可能包含數百個遷移檔案。如果您願意，您可以將遷移「壓縮」成單一的 SQL 檔案。若要開始，請執行 `schema:dump` 命令：

```shell
php artisan schema:dump

# Dump the current database schema and prune all existing migrations...
php artisan schema:dump --prune
```

當您執行此命令時，Laravel 會將一個「結構 (schema)」檔案寫入您應用程式的 `database/schema` 目錄中。結構檔案的名稱將與資料庫連線相對應。現在，當您嘗試遷移資料庫且沒有其他遷移被執行時，Laravel 會首先執行您所使用的資料庫連線結構檔案中的 SQL 語句。在執行結構檔案的 SQL 語句後，Laravel 會執行任何不屬於結構 dump 的剩餘遷移。

如果您的應用程式測試使用了與本地開發時不同的資料庫連線，您應該確保已使用該資料庫連線 dump 一份結構檔案，以便您的測試能夠建立資料庫。您可能希望在 dump 了本地開發常用的資料庫連線之後執行此操作：

```shell
php artisan schema:dump
php artisan schema:dump --database=testing --prune
```

您應該將資料庫結構檔案提交至原始碼控制，以便團隊中的其他新開發人員能快速建立應用程式的初始資料庫結構。

> [!WARNING]
> 遷移壓縮僅適用於 MariaDB、MySQL、PostgreSQL 和 SQLite 資料庫，且需要使用資料庫的命令列客戶端。


<a name="migration-structure"></a>
## 遷移結構

一個遷移類別包含兩個方法：`up` 與 `down`。`up` 方法用於將新資料表、欄位或索引新增到您的資料庫，而 `down` 方法則應反轉 `up` 方法所執行的操作。

在這兩個方法中，您可以使用 Laravel schema builder 直觀地建立與修改資料表。若要了解 `Schema` builder 上所有可用的方法，請[查看其文件](#creating-tables)。例如，以下遷移會建立一個 `flights` 資料表：

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

如果您的遷移將與應用程式預設資料庫連線以外的資料庫連線進行互動，您應該設定遷移的 `$connection` 屬性：

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

有時候，某個遷移可能是為了支援一個尚未啟用的功能，而您還不想讓它執行。在這種情況下，您可以在遷移中定義一個 `shouldRun` 方法。如果 `shouldRun` 方法返回 `false`，該遷移將被跳過：

```php
use App\Models\Flight;
use Laravel\Pennant\Feature;

/**
 * Determine if this migration should run.
 */
public function shouldRun(): bool
{
    return Feature::active(Flight::class);
}
```

<a name="running-migrations"></a>
## 執行遷移

若要執行所有尚未執行的遷移，請執行 `migrate` Artisan 指令：

```shell
php artisan migrate
```

如果您想查看哪些遷移已經執行，以及哪些還在等待中，可以使用 `migrate:status` Artisan 指令：

```shell
php artisan migrate:status
```

如果您在 `migrate` 指令中提供 `--step` 選項，該指令將把每次遷移視為獨立的批次，讓您稍後可以使用 `migrate:rollback` 指令回滾單個遷移：

```shell
php artisan migrate --step
```

如果您想在不實際執行的情況下查看遷移將執行的 SQL 語句，可以為 `migrate` 指令提供 `--pretend` 旗標：

```shell
php artisan migrate --pretend
```


<a name="isolating-migration-execution"></a>
#### 隔離遷移執行

如果您將應用程式部署在多台伺服器上，並將執行遷移作為部署過程的一部分，您可能不希望兩台伺服器同時嘗試遷移資料庫。為了避免這種情況，您可以在調用 `migrate` 指令時使用 `isolated` 選項。

當提供 `isolated` 選項時，Laravel 會在嘗試執行遷移之前，使用應用程式的快取驅動程式獲取一個原子鎖。當該鎖被持有時，所有其他執行 `migrate` 指令的嘗試都不會執行；然而，該指令仍會以成功的退出狀態碼結束：

```shell
php artisan migrate --isolated
```

> [!WARNING]
> 若要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 作為預設的快取驅動程式。此外，所有伺服器必須與同一個中央快取伺服器通訊。


<a name="forcing-migrations-to-run-in-production"></a>
#### 強制在正式環境執行遷移

某些遷移操作是破壞性的，這意味著它們可能會導致您丟失資料。為了防止您在正式環境資料庫上執行這些指令，系統會在指令執行前要求您確認。若要強制在沒有提示的情況下執行指令，請使用 `--force` 旗標：

```shell
php artisan migrate --force
```


<a name="rolling-back-migrations"></a>
### 回滾遷移

若要回滾最近一次的遷移操作，您可以使用 `rollback` Artisan 指令。此指令會回滾最後一個遷移「批次」，其中可能包含多個遷移文件：

```shell
php artisan migrate:rollback
```

您可以透過向 `rollback` 指令提供 `step` 選項來回滾有限數量的遷移。例如，以下指令將回滾最後五次遷移：

```shell
php artisan migrate:rollback --step=5
```

您可以透過向 `rollback` 指令提供 `batch` 選項來回滾特定的遷移「批次」，其中 `batch` 選項對應於應用程式 `migrations` 資料表中的批次值。例如，以下指令將回滾批次三中的所有遷移：

```shell
php artisan migrate:rollback --batch=3
```

如果您想在不實際執行的情況下查看遷移將執行的 SQL 語句，可以為 `migrate:rollback` 指令提供 `--pretend` 旗標：

```shell
php artisan migrate:rollback --pretend
```

`migrate:reset` 指令將回滾應用程式的所有遷移：

```shell
php artisan migrate:reset
```


<a name="roll-back-migrate-using-a-single-command"></a>
#### 使用單一指令進行回滾與遷移

`migrate:refresh` 指令將回滾所有遷移，然後執行 `migrate` 指令。此指令實際上會重新建立您的整個資料庫：

```shell
php artisan migrate:refresh

# Refresh the database and run all database seeds...
php artisan migrate:refresh --seed
```

您可以透過向 `refresh` 指令提供 `step` 選項來回滾並重新遷移有限數量的遷移。例如，以下指令將回滾並重新遷移最後五次遷移：

```shell
php artisan migrate:refresh --step=5
```


<a name="drop-all-tables-migrate"></a>
#### 刪除所有資料表並遷移

`migrate:fresh` 指令將從資料庫中刪除所有資料表，然後執行 `migrate` 指令：

```shell
php artisan migrate:fresh

php artisan migrate:fresh --seed
```

預設情況下，`migrate:fresh` 指令僅刪除預設資料庫連線中的資料表。然而，您可以使用 `--database` 選項來指定應遷移的資料庫連線。資料庫連線名稱應對應於應用程式 `database` [設定檔](/docs/{{version}}/configuration) 中定義的連線：

```shell
php artisan migrate:fresh --database=admin
```

> [!WARNING]
> `migrate:fresh` 指令將刪除所有資料庫資料表，無論其前綴為何。在與其他應用程式共用資料庫進行開發時，請謹慎使用此指令。

<a name="tables"></a>
## 資料表


<a name="creating-tables"></a>
### 建立資料表

若要建立新的資料庫資料表，請使用 `Schema` facade 的 `create` 方法。`create` 方法接受兩個引數：第一個是資料表的名稱，而第二個是一個接收 `Blueprint` 物件的閉包，可用於定義新資料表：

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

在建立資料表時，您可以使用 schema builder 的任何 [欄位方法](#creating-columns) 來定義資料表的欄位。


<a name="determining-table-column-existence"></a>
#### 判斷資料表 / 欄位是否存在

您可以使用 `hasTable`、`hasColumn` 與 `hasIndex` 方法來判斷資料表、欄位或索引是否存在：

```php
if (Schema::hasTable('users')) {
    // The "users" table exists...
}

if (Schema::hasColumn('users', 'email')) {
    // The "users" table exists and has an "email" column...
}

if (Schema::hasIndex('users', ['email'], 'unique')) {
    // The "users" table exists and has a unique index on the "email" column...
}
```


<a name="database-connection-table-options"></a>
#### 資料庫連線與資料表選項

如果您想在非應用程式預設的資料庫連線上執行 schema 操作，請使用 `connection` 方法：

```php
Schema::connection('sqlite')->create('users', function (Blueprint $table) {
    $table->id();
});
```

此外，還可以使用一些其他屬性和方法來定義資料表建立的其他面向。當使用 MariaDB 或 MySQL 時，可以使用 `engine` 屬性來指定資料表的儲存引擎：

```php
Schema::create('users', function (Blueprint $table) {
    $table->engine('InnoDB');

    // ...
});
```

當使用 MariaDB 或 MySQL 時，可以使用 `charset` 與 `collation` 屬性來指定建立資料表的字元集與排序規則：

```php
Schema::create('users', function (Blueprint $table) {
    $table->charset('utf8mb4');
    $table->collation('utf8mb4_unicode_ci');

    // ...
});
```

可以使用 `temporary` 方法來表示該資料表應為「暫存表」。暫存表僅對目前連線的資料庫會話可見，且在連線關閉時會自動被刪除：

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->temporary();

    // ...
});
```

如果您想為資料庫資料表添加「註解」，可以在資料表實例上呼叫 `comment` 方法。目前僅 MariaDB、MySQL 與 PostgreSQL 支援資料表註解：

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->comment('Business calculations');

    // ...
});
```


<a name="updating-tables"></a>
### 更新資料表

`Schema` facade 的 `table` 方法可用於更新現有資料表。與 `create` 方法類似，`table` 方法接受兩個引數：資料表的名稱以及一個接收 `Blueprint` 實例的閉包，您可以使用該實例向資料表添加欄位或索引：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->integer('votes');
});
```


<a name="renaming-and-dropping-tables"></a>
### 重新命名 / 刪除資料表

若要重新命名現有的資料庫資料表，請使用 `rename` 方法：

```php
use Illuminate\Support\Facades\Schema;

Schema::rename($from, $to);
```

若要刪除現有資料表，您可以使用 `drop` 或 `dropIfExists` 方法：

```php
Schema::drop('users');

Schema::dropIfExists('users');
```


<a name="renaming-tables-with-foreign-keys"></a>
#### 重新命名具有外鍵的資料表

在重新命名資料表之前，您應該確認該資料表上的所有外鍵約束在遷移文件中都有明確的名稱，而不是讓 Laravel 根據慣例分配名稱。否則，外鍵約束的名稱將會指向舊的資料表名稱。

<a name="columns"></a>
## 欄位


<a name="creating-columns"></a>
### 建立欄位

`Schema` Facade 上的 `table` 方法可用於更新現有的資料表。如同 `create` 方法，`table` 方法接受兩個引數：資料表名稱以及一個接收 `Illuminate\Database\Schema\Blueprint` 實例的閉包，您可以使用該實例來為資料表新增欄位：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->integer('votes');
});
```

<a name="available-column-types"></a>
### 可用的欄位類型

Schema 構建器的 Blueprint 提供了多種方法，對應於您可以添加到資料庫資料表中的不同欄位類型。所有可用方法列於下表：

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


<a name="relationship-method-list"></a>
#### 關聯類型

<div class="collection-method-list" markdown="1">

[foreignId](#column-method-foreignId)
[foreignIdFor](#column-method-foreignIdFor)
[foreignUlid](#column-method-foreignUlid)
[foreignUuid](#column-method-foreignUuid)
[morphs](#column-method-morphs)
[nullableMorphs](#column-method-nullableMorphs)

</div>


<a name="specifics-method-list"></a>
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

`bigIncrements` 方法會建立一個自動遞增的 `UNSIGNED BIGINT` (主鍵) 等同欄位：

```php
$table->bigIncrements('id');
```


<a name="column-method-bigInteger"></a>
#### `bigInteger()` {.collection-method}

`bigInteger` 方法會建立一個 `BIGINT` 等同欄位：

```php
$table->bigInteger('votes');
```


<a name="column-method-binary"></a>
#### `binary()` {.collection-method}

`binary` 方法會建立一個 `BLOB` 等同欄位：

```php
$table->binary('photo');
```

當使用 MySQL、MariaDB 或 SQL Server 時，您可以傳遞 `length` 與 `fixed` 引數來建立 `VARBINARY` 或 `BINARY` 等同欄位：

```php
$table->binary('data', length: 16); // VARBINARY(16)

$table->binary('data', length: 16, fixed: true); // BINARY(16)
```


<a name="column-method-boolean"></a>
#### `boolean()` {.collection-method}

`boolean` 方法會建立一個 `BOOLEAN` 等同欄位：

```php
$table->boolean('confirmed');
```


<a name="column-method-char"></a>
#### `char()` {.collection-method}

`char` 方法會建立一個給定長度的 `CHAR` 等同欄位：

```php
$table->char('name', length: 100);
```


<a name="column-method-dateTimeTz"></a>
#### `dateTimeTz()` {.collection-method}

`dateTimeTz` 方法會建立一個 `DATETIME` (包含時區) 等同欄位，並可選擇小數秒精度：

```php
$table->dateTimeTz('created_at', precision: 0);
```


<a name="column-method-dateTime"></a>
#### `dateTime()` {.collection-method}

`dateTime` 方法會建立一個 `DATETIME` 等同欄位，並可選擇小數秒精度：

```php
$table->dateTime('created_at', precision: 0);
```


<a name="column-method-date"></a>
#### `date()` {.collection-method}

`date` 方法會建立一個 `DATE` 等同欄位：

```php
$table->date('created_at');
```


<a name="column-method-decimal"></a>
#### `decimal()` {.collection-method}

`decimal` 方法會建立一個 `DECIMAL` 等同欄位，並指定精度 (總位數) 與刻度 (小數位數)：

```php
$table->decimal('amount', total: 8, places: 2);
```


<a name="column-method-double"></a>
#### `double()` {.collection-method}

`double` 方法會建立一個 `DOUBLE` 等同欄位：

```php
$table->double('amount');
```


<a name="column-method-enum"></a>
#### `enum()` {.collection-method}

`enum` 方法會建立一個 `ENUM` 等同欄位，並指定有效的值：

```php
$table->enum('difficulty', ['easy', 'hard']);
```

當然，您可以使用 `Enum::cases()` 方法，而不是手動定義一個允許值的陣列：

```php
use App\Enums\Difficulty;

$table->enum('difficulty', Difficulty::cases());
```


<a name="column-method-float"></a>
#### `float()` {.collection-method}

`float` 方法會建立一個 `FLOAT` 等同欄位，並指定精度：

```php
$table->float('amount', precision: 53);
```


<a name="column-method-foreignId"></a>
#### `foreignId()` {.collection-method}

`foreignId` 方法會建立一個 `UNSIGNED BIGINT` 等同欄位：

```php
$table->foreignId('user_id');
```


<a name="column-method-foreignIdFor"></a>
#### `foreignIdFor()` {.collection-method}

`foreignIdFor` 方法會為給定的模型類別增加一個 `{column}_id` 等同欄位。根據模型金鑰類型，欄位類型將會是 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`：

```php
$table->foreignIdFor(User::class);
```


<a name="column-method-foreignUlid"></a>
#### `foreignUlid()` {.collection-method}

`foreignUlid` 方法會建立一個 `ULID` 等同欄位：

```php
$table->foreignUlid('user_id');
```


<a name="column-method-foreignUuid"></a>
#### `foreignUuid()` {.collection-method}

`foreignUuid` 方法會建立一個 `UUID` 等同欄位：

```php
$table->foreignUuid('user_id');
```


<a name="column-method-geography"></a>
#### `geography()` {.collection-method}

`geography` 方法會建立一個 `GEOGRAPHY` 等同欄位，並指定空間類型與 SRID (空間參考系統識別碼)：

```php
$table->geography('coordinates', subtype: 'point', srid: 4326);
```

> [!NOTE]
> 空間類型的支援取決於您的資料庫驅動程式。請參閱您的資料庫文件。如果您的應用程式使用的是 PostgreSQL 資料庫，在可以使用 `geography` 方法之前，您必須安裝 [PostGIS](https://postgis.net) 擴充功能。


<a name="column-method-geometry"></a>
#### `geometry()` {.collection-method}

`geometry` 方法會建立一個 `GEOMETRY` 等同欄位，並指定空間類型與 SRID (空間參考系統識別碼)：

```php
$table->geometry('positions', subtype: 'point', srid: 0);
```

> [!NOTE]
> 空間類型的支援取決於您的資料庫驅動程式。請參閱您的資料庫文件。如果您的應用程式使用的是 PostgreSQL 資料庫，在可以使用 `geometry` 方法之前，您必須安裝 [PostGIS](https://postgis.net) 擴充功能。


<a name="column-method-id"></a>
#### `id()` {.collection-method}

`id` 方法是 `bigIncrements` 方法的別名。預設情況下，此方法會建立一個 `id` 欄位；不過，如果您想為該欄位指定不同的名稱，可以傳遞欄位名稱：

```php
$table->id();
```


<a name="column-method-increments"></a>
#### `increments()` {.collection-method}

`increments` 方法會建立一個自動遞增的 `UNSIGNED INTEGER` 等同欄位作為主鍵：

```php
$table->increments('id');
```


<a name="column-method-integer"></a>
#### `integer()` {.collection-method}

`integer` 方法會建立一個 `INTEGER` 等同欄位：

```php
$table->integer('votes');
```


<a name="column-method-ipAddress"></a>
#### `ipAddress()` {.collection-method}

`ipAddress` 方法會建立一個 `VARCHAR` 等同欄位：

```php
$table->ipAddress('visitor');
```

使用 PostgreSQL 時，將建立一個 `INET` 欄位。


<a name="column-method-json"></a>
#### `json()` {.collection-method}

`json` 方法會建立一個 `JSON` 等同欄位：

```php
$table->json('options');
```

使用 SQLite 時，將建立一個 `TEXT` 欄位。


<a name="column-method-jsonb"></a>
#### `jsonb()` {.collection-method}

`jsonb` 方法會建立一個 `JSONB` 等同欄位：

```php
$table->jsonb('options');
```

使用 SQLite 時，將建立一個 `TEXT` 欄位。


<a name="column-method-longText"></a>
#### `longText()` {.collection-method}

`longText` 方法會建立一個 `LONGTEXT` 等同欄位：

```php
$table->longText('description');
```

當使用 MySQL 或 MariaDB 時，您可以對欄位套用 `binary` 字元集以建立 `LONGBLOB` 等同欄位：

```php
$table->longText('data')->charset('binary'); // LONGBLOB
```


<a name="column-method-macAddress"></a>
#### `macAddress()` {.collection-method}

`macAddress` 方法會建立一個旨在存放 MAC 地址的欄位。某些資料庫系統（如 PostgreSQL）為此類數據提供了專用的欄位類型，其他資料庫系統則會使用字串等同欄位：

```php
$table->macAddress('device');
```


<a name="column-method-mediumIncrements"></a>
#### `mediumIncrements()` {.collection-method}

`mediumIncrements` 方法會建立一個自動遞增的 `UNSIGNED MEDIUMINT` 等同欄位作為主鍵：

```php
$table->mediumIncrements('id');
```


<a name="column-method-mediumInteger"></a>
#### `mediumInteger()` {.collection-method}

`mediumInteger` 方法會建立一個 `MEDIUMINT` 等同欄位：

```php
$table->mediumInteger('votes');
```


<a name="column-method-mediumText"></a>
#### `mediumText()` {.collection-method}

`mediumText` 方法會建立一個 `MEDIUMTEXT` 等同欄位：

```php
$table->mediumText('description');
```

當使用 MySQL 或 MariaDB 時，您可以對欄位套用 `binary` 字元集以建立 `MEDIUMBLOB` 等同欄位：

```php
$table->mediumText('data')->charset('binary'); // MEDIUMBLOB
```


<a name="column-method-morphs"></a>
#### `morphs()` {.collection-method}

`morphs` 方法是一個便捷方法，會增加一個 `{column}_id` 等同欄位以及一個 `{column}_type` `VARCHAR` 等同欄位。根據模型金鑰類型，`{column}_id` 的欄位類型將會是 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`。

此方法旨在用於定義多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships) 所需的欄位。在以下範例中，將建立 `taggable_id` 與 `taggable_type` 欄位：

```php
$table->morphs('taggable');
```


<a name="column-method-nullableMorphs"></a>
#### `nullableMorphs()` {.collection-method}

此方法與 [morphs](#column-method-morphs) 方法類似；但建立的欄位將是「可為空 (nullable)」的：

```php
$table->nullableMorphs('taggable');
```


<a name="column-method-nullableUlidMorphs"></a>
#### `nullableUlidMorphs()` {.collection-method}

此方法與 [ulidMorphs](#column-method-ulidMorphs) 方法類似；但建立的欄位將是「可為空 (nullable)」的：

```php
$table->nullableUlidMorphs('taggable');
```


<a name="column-method-nullableUuidMorphs"></a>
#### `nullableUuidMorphs()` {.collection-method}

此方法與 [uuidMorphs](#column-method-uuidMorphs) 方法類似；但建立的欄位將是「可為空 (nullable)」的：

```php
$table->nullableUuidMorphs('taggable');
```


<a name="column-method-rememberToken"></a>
#### `rememberToken()` {.collection-method}

`rememberToken` 方法會建立一個可為空 (nullable) 的 `VARCHAR(100)` 等同欄位，旨在儲存目前的「記住我」[認證令牌](/docs/{{version}}/authentication#remembering-users)：

```php
$table->rememberToken();
```


<a name="column-method-set"></a>
#### `set()` {.collection-method}

`set` 方法會建立一個 `SET` 等同欄位，並指定有效值的列表：

```php
$table->set('flavors', ['strawberry', 'vanilla']);
```


<a name="column-method-smallIncrements"></a>
#### `smallIncrements()` {.collection-method}

`smallIncrements` 方法會建立一個自動遞增的 `UNSIGNED SMALLINT` 等同欄位作為主鍵：

```php
$table->smallIncrements('id');
```


<a name="column-method-smallInteger"></a>
#### `smallInteger()` {.collection-method}

`smallInteger` 方法會建立一個 `SMALLINT` 等同欄位：

```php
$table->smallInteger('votes');
```


<a name="column-method-softDeletesTz"></a>
#### `softDeletesTz()` {.collection-method}

`softDeletesTz` 方法會增加一個可為空 (nullable) 的 `deleted_at` `TIMESTAMP` (包含時區) 等同欄位，並可選擇小數秒精度。此欄位旨在儲存 Eloquent 「軟刪除」功能所需的 `deleted_at` 時間戳記：

```php
$table->softDeletesTz('deleted_at', precision: 0);
```


<a name="column-method-softDeletes"></a>
#### `softDeletes()` {.collection-method}

`softDeletes` 方法會增加一個可為空 (nullable) 的 `deleted_at` `TIMESTAMP` 等同欄位，並可選擇小數秒精度。此欄位旨在儲存 Eloquent 「軟刪除」功能所需的 `deleted_at` 時間戳記：

```php
$table->softDeletes('deleted_at', precision: 0);
```


<a name="column-method-string"></a>
#### `string()` {.collection-method}

`string` 方法會建立一個給定長度的 `VARCHAR` 等同欄位：

```php
$table->string('name', length: 100);
```


<a name="column-method-text"></a>
#### `text()` {.collection-method}

`text` 方法會建立一個 `TEXT` 等同欄位：

```php
$table->text('description');
```

當使用 MySQL 或 MariaDB 時，您可以對欄位套用 `binary` 字元集以建立 `BLOB` 等同欄位：

```php
$table->text('data')->charset('binary'); // BLOB
```


<a name="column-method-timeTz"></a>
#### `timeTz()` {.collection-method}

`timeTz` 方法會建立一個 `TIME` (包含時區) 等同欄位，並可選擇小數秒精度：

```php
$table->timeTz('sunrise', precision: 0);
```


<a name="column-method-time"></a>
#### `time()` {.collection-method}

`time` 方法會建立一個 `TIME` 等同欄位，並可選擇小數秒精度：

```php
$table->time('sunrise', precision: 0);
```


<a name="column-method-timestampTz"></a>
#### `timestampTz()` {.collection-method}

`timestampTz` 方法會建立一個 `TIMESTAMP` (包含時區) 等同欄位，並可選擇小數秒精度：

```php
$table->timestampTz('added_at', precision: 0);
```


<a name="column-method-timestamp"></a>
#### `timestamp()` {.collection-method}

`timestamp` 方法會建立一個 `TIMESTAMP` 等同欄位，並可選擇小數秒精度：

```php
$table->timestamp('added_at', precision: 0);
```


<a name="column-method-timestampsTz"></a>
#### `timestampsTz()` {.collection-method}

`timestampsTz` 方法會建立 `created_at` 與 `updated_at` `TIMESTAMP` (包含時區) 等同欄位，並可選擇小數秒精度：

```php
$table->timestampsTz(precision: 0);
```


<a name="column-method-timestamps"></a>
#### `timestamps()` {.collection-method}

`timestamps` 方法會建立 `created_at` 與 `updated_at` `TIMESTAMP` 等同欄位，並可選擇小數秒精度：

```php
$table->timestamps(precision: 0);
```


<a name="column-method-tinyIncrements"></a>
#### `tinyIncrements()` {.collection-method}

`tinyIncrements` 方法會建立一個自動遞增的 `UNSIGNED TINYINT` 等同欄位作為主鍵：

```php
$table->tinyIncrements('id');
```


<a name="column-method-tinyInteger"></a>
#### `tinyInteger()` {.collection-method}

`tinyInteger` 方法會建立一個 `TINYINT` 等同欄位：

```php
$table->tinyInteger('votes');
```


<a name="column-method-tinyText"></a>
#### `tinyText()` {.collection-method}

`tinyText` 方法會建立一個 `TINYTEXT` 等同欄位：

```php
$table->tinyText('notes');
```

當使用 MySQL 或 MariaDB 時，您可以對欄位套用 `binary` 字元集以建立 `TINYBLOB` 等同欄位：

```php
$table->tinyText('data')->charset('binary'); // TINYBLOB
```


<a name="column-method-unsignedBigInteger"></a>
#### `unsignedBigInteger()` {.collection-method}

`unsignedBigInteger` 方法會建立一個 `UNSIGNED BIGINT` 等同欄位：

```php
$table->unsignedBigInteger('votes');
```


<a name="column-method-unsignedInteger"></a>
#### `unsignedInteger()` {.collection-method}

`unsignedInteger` 方法會建立一個 `UNSIGNED INTEGER` 等同欄位：

```php
$table->unsignedInteger('votes');
```


<a name="column-method-unsignedMediumInteger"></a>
#### `unsignedMediumInteger()` {.collection-method}

`unsignedMediumInteger` 方法會建立一個 `UNSIGNED MEDIUMINT` 等同欄位：

```php
$table->unsignedMediumInteger('votes');
```


<a name="column-method-unsignedSmallInteger"></a>
#### `unsignedSmallInteger()` {.collection-method}

`unsignedSmallInteger` 方法會建立一個 `UNSIGNED SMALLINT` 等同欄位：

```php
$table->unsignedSmallInteger('votes');
```


<a name="column-method-unsignedTinyInteger"></a>
#### `unsignedTinyInteger()` {.collection-method}

`unsignedTinyInteger` 方法會建立一個 `UNSIGNED TINYINT` 等同欄位：

```php
$table->unsignedTinyInteger('votes');
```


<a name="column-method-ulidMorphs"></a>
#### `ulidMorphs()` {.collection-method}

`ulidMorphs` 方法是一個便捷方法，會增加一個 `{column}_id` `CHAR(26)` 等同欄位以及一個 `{column}_type` `VARCHAR` 等同欄位。

此方法旨在用於定義使用 ULID 識別碼的多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships) 所需的欄位。在以下範例中，將建立 `taggable_id` 與 `taggable_type` 欄位：

```php
$table->ulidMorphs('taggable');
```


<a name="column-method-uuidMorphs"></a>
#### `uuidMorphs()` {.collection-method}

`uuidMorphs` 方法是一個便捷方法，會增加一個 `{column}_id` `CHAR(36)` 等同欄位以及一個 `{column}_type` `VARCHAR` 等同欄位。

此方法旨在用於定義使用 UUID 識別碼的 [多型 Eloquent 關聯](/docs/{{version}}/eloquent-relationships#polymorphic-relationships) 所需的欄位。在以下範例中，將建立 `taggable_id` 與 `taggable_type` 欄位：

```php
$table->uuidMorphs('taggable');
```


<a name="column-method-ulid"></a>
#### `ulid()` {.collection-method}

`ulid` 方法會建立一個 `ULID` 等同欄位：

```php
$table->ulid('id');
```


<a name="column-method-uuid"></a>
#### `uuid()` {.collection-method}

`uuid` 方法會建立一個 `UUID` 等同欄位：

```php
$table->uuid('id');
```


<a name="column-method-vector"></a>
#### `vector()` {.collection-method}

`vector` 方法會建立一個 `vector` 等同欄位：

```php
$table->vector('embedding', dimensions: 100);
```

當使用 PostgreSQL 時，在建立 `vector` 欄位之前必須載入 `pgvector` 擴充功能：

```php
Schema::ensureVectorExtensionExists();
```


<a name="column-method-year"></a>
#### `year()` {.collection-method}

`year` 方法會建立一個 `YEAR` 等同欄位：

```php
$table->year('birth_year');
```

<a name="column-modifiers"></a>
### 欄位修改器

除了上述列出的欄位類型外，您在將欄位新增到資料表時，還可以使用幾個欄位「修改器」。例如，若要使欄位可為空值 (nullable)，可以使用 `nullable` 方法：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->string('email')->nullable();
});
```

下表包含了所有可用的欄位修改器。此列表不包含 [索引修改器](#creating-indexes)：

<div class="overflow-auto">

| 修改器                            | 描述                                                                                    |
| ----------------------------------- | ---------------------------------------------------------------------------------------------- |
| `->after('column')`                 | 將欄位放置在另一個欄位「之後」(MariaDB / MySQL)。                                     |
| `->autoIncrement()`                 | 將 `INTEGER` 欄位設定為自動遞增 (主鍵)。                                      |
| `->charset('utf8mb4')`              | 指定欄位的字元集 (MariaDB / MySQL)。                                      |
| `->collation('utf8mb4_unicode_ci')` | 指定欄位的排序規則 (collation)。                                                            |
| `->comment('my comment')`           | 為欄位新增註解 (MariaDB / MySQL / PostgreSQL)。                                      |
| `->default($value)`                 | 指定欄位的「預設」值。                                                      |
| `->first()`                         | 將欄位放置在資料表的「最前面」(MariaDB / MySQL)。                                       |
| `->from($integer)`                  | 設定自動遞增欄位的起始值 (MariaDB / MySQL / PostgreSQL)。           |
| `->instant()`                       | 使用即時操作新增或修改欄位 (MySQL)。                                   |
| `->invisible()`                     | 使欄位對 `SELECT *` 查詢「不可見」(MariaDB / MySQL)。                           |
| `->lock($mode)`                     | 指定欄位操作的鎖定模式 (MySQL)。                                          |
| `->nullable($value = true)`         | 允許在欄位中插入 `NULL` 值。                                            |
| `->storedAs($expression)`           | 建立儲存的產出欄位 (stored generated column) (MariaDB / MySQL / PostgreSQL / SQLite)。                      |
| `->unsigned()`                      | 將 `INTEGER` 欄位設定為 `UNSIGNED` (MariaDB / MySQL)。                                         |
| `->useCurrent()`                    | 將 `TIMESTAMP` 欄位設定為使用 `CURRENT_TIMESTAMP` 作為預設值。                           |
| `->useCurrentOnUpdate()`            | 將 `TIMESTAMP` 欄位設定為在紀錄更新時使用 `CURRENT_TIMESTAMP` (MariaDB / MySQL)。 |
| `->virtualAs($expression)`          | 建立虛擬的產出欄位 (virtual generated column) (MariaDB / MySQL / SQLite)。                                  |
| `->generatedAs($expression)`        | 建立具有指定序列選項的識別欄位 (identity column) (PostgreSQL)。                        |
| `->always()`                        | 定義識別欄位中序列值優先於輸入值的優先級 (PostgreSQL)。      |

</div>


<a name="default-expressions"></a>
#### 預設運算式

`default` 修改器接受一個值或一個 `Illuminate\Database\Query\Expression` 實例。使用 `Expression` 實例將防止 Laravel 將值包裹在引號中，並允許您使用資料庫特定的函數。在需要為 JSON 欄位分配預設值時，這特別有用：

```php
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
```

> [!WARNING]
> 預設運算式的支援取決於您的資料庫驅動程式、資料庫版本以及欄位類型。請參考您的資料庫文件。


<a name="column-order"></a>
#### 欄位順序

當使用 MariaDB 或 MySQL 資料庫時，可以使用 `after` 方法在架構中現有欄位之後新增欄位：

```php
$table->after('password', function (Blueprint $table) {
    $table->string('address_line1');
    $table->string('address_line2');
    $table->string('city');
});
```


<a name="instant-column-operations"></a>
#### 即時欄位操作

當使用 MySQL 時，您可以將 `instant` 修改器鏈結到欄位定義上，以表示該欄位應使用 MySQL 的「即時 (instant)」演算法來新增或修改。此演算法允許在不重新建立整個資料表的情況下執行某些架構變更，使其無論資料表大小如何，幾乎都能即時完成：

```php
$table->string('name')->nullable()->instant();
```

即時欄位新增只能將欄位附加到資料表的末尾，因此 `instant` 修改器不能與 `after` 或 `first` 修改器結合使用。此外，該演算法並不支援所有欄位類型或操作。如果請求的操作不相容，MySQL 將會拋出錯誤。

請參考 [MySQL 文件](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html) 以確定哪些操作與即時欄位修改相容。


<a name="ddl-locking"></a>
#### DDL 鎖定

當使用 MySQL 時，您可以將 `lock` 修改器鏈結到欄位、索引或外鍵定義上，以控制架構操作期間的資料表鎖定。MySQL 支援多種鎖定模式：`none` 允許併發讀取和寫入，`shared` 允許併發讀取但阻止寫入，`exclusive` 阻止所有併發存取，而 `default` 則讓 MySQL 選擇最合適的模式：

```php
$table->string('name')->lock('none');

$table->index('email')->lock('shared');
```

如果請求的鎖定模式與操作不相容，MySQL 將會拋出錯誤。`lock` 修改器可以與 `instant` 修改器結合使用，以進一步優化架構變更：

```php
$table->string('name')->instant()->lock('none');
```


<a name="modifying-columns"></a>
### 修改欄位

`change` 方法允許您修改現有欄位的類型和屬性。例如，您可能希望增加 `string` 欄位的大小。為了展示 `change` 方法的實際運作，讓我們將 `name` 欄位的大小從 25 增加到 50。為了實現這一點，我們只需定義欄位的新狀態，然後呼叫 `change` 方法：

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('name', 50)->change();
});
```

在修改欄位時，您必須在欄位定義中明確地包含所有您想要保留的修改器——任何遺漏的屬性都將被移除。例如，若要保留 `unsigned`、`default` 和 `comment` 屬性，您在更改欄位時必須明確呼叫每個修改器：

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')->unsigned()->default(1)->comment('my comment')->change();
});
```

`change` 方法不會更改欄位的索引。因此，您可以在修改欄位時使用索引修改器來明確地新增或刪除索引：

```php
// Add an index...
$table->bigIncrements('id')->primary()->change();

// Drop an index...
$table->char('postal_code', 10)->unique(false)->change();
```

<a name="renaming-columns"></a>
### 重新命名欄位

若要重新命名欄位，您可以使用 schema builder 提供的 `renameColumn` 方法：

```php
Schema::table('users', function (Blueprint $table) {
    $table->renameColumn('from', 'to');
});
```


<a name="dropping-columns"></a>
### 刪除欄位

若要刪除欄位，您可以使用 schema builder 上的 `dropColumn` 方法：

```php
Schema::table('users', function (Blueprint $table) {
    $table->dropColumn('votes');
});
```

您可以透過將欄位名稱陣列傳遞給 `dropColumn` 方法，從資料表中刪除多個欄位：

```php
Schema::table('users', function (Blueprint $table) {
    $table->dropColumn(['votes', 'avatar', 'location']);
});
```


<a name="available-command-aliases"></a>
#### 可用的指令別名

Laravel 提供了一些與刪除常見類型欄位相關的便利方法。下表說明了這些方法：

<div class="overflow-auto">

| 指令                             | 描述                                                 |
| ----------------------------------- | ----------------------------------------------------- |
| `$table->dropMorphs('morphable');`  | 刪除 `morphable_id` 與 `morphable_type` 欄位。 |
| `$table->dropRememberToken();`      | 刪除 `remember_token` 欄位。                         |
| `$table->dropSoftDeletes();`        | 刪除 `deleted_at` 欄位。                             |
| `$table->dropSoftDeletesTz();`      | `dropSoftDeletes()` 方法的別名。                    |
| `$table->dropTimestamps();`         | 刪除 `created_at` 與 `updated_at` 欄位。           |
| `$table->dropTimestampsTz();`       | `dropTimestamps()` 方法的別名。                     |

</div>

<a name="indexes"></a>
## 索引


<a name="creating-indexes"></a>
### 建立索引

Laravel 的 schema builder 支援多種索引類型。以下範例建立了一個新的 `email` 欄位，並指定其值必須是唯一的。若要建立索引，我們可以將 `unique` 方法鏈接至欄位定義：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->string('email')->unique();
});
```

或者，您也可以在定義欄位後建立索引。若要這樣做，您應該在 schema builder 的 blueprint 上呼叫 `unique` 方法。此方法接受應接收唯一索引的欄位名稱：

```php
$table->unique('email');
```

您甚至可以將欄位陣列傳遞給索引方法，以建立複合索引 (compound or composite index)：

```php
$table->index(['account_id', 'created_at']);
```

在建立索引時，Laravel 會根據資料表、欄位名稱和索引類型自動產生索引名稱，但您也可以向方法傳遞第二個引數來自行指定索引名稱：

```php
$table->unique('email', 'unique_email');
```


<a name="available-index-types"></a>
#### 可用的索引類型

Laravel 的 schema builder blueprint 類別提供了建立 Laravel 支援的每種索引類型的方法。每個索引方法都接受一個可選的第二個引數來指定索引名稱。如果省略，名稱將根據用於索引的資料表和欄位名稱以及索引類型而衍生。下表描述了所有可用的索引方法：

<div class="overflow-auto">

| 指令                                           | 描述                                                       |
| ------------------------------------------------ | -------------------------------------------------------------- |
| `$table->primary('id');`                         | 新增主鍵。                                                   |
| `$table->primary(['id', 'parent_id']);`          | 新增複合鍵。                                                 |
| `$table->unique('email');`                       | 新增唯一索引。                                               |
| `$table->index('state');`                        | 新增索引。                                                   |
| `$table->fullText('body');`                      | 新增全文字索引 (MariaDB / MySQL / PostgreSQL)。              |
| `$table->fullText('body')->language('english');` | 新增指定語言的全文字索引 (PostgreSQL)。                      |
| `$table->spatialIndex('location');`              | 新增空間索引 (SQLite 除外)。                                 |

</div>


<a name="online-index-creation"></a>
#### 線上索引建立

預設情況下，在大表上建立索引可能會鎖定資料表，並在建立索引期間阻擋讀取或寫入。當使用 PostgreSQL 或 SQL Server 時，您可以將 `online` 方法鏈接到索引定義中，以在不鎖定資料表的情況下建立索引，讓您的應用程式在索引建立期間能繼續讀取和寫入資料：

```php
$table->string('email')->unique()->online();
```

當使用 PostgreSQL 時，這會將 `CONCURRENTLY` 選項添加到索引建立陳述式中。當使用 SQL Server 時，這會添加 `WITH (online = on)` 選項。


<a name="renaming-indexes"></a>
### 重新命名索引

若要重新命名索引，您可以使用 schema builder blueprint 提供的 `renameIndex` 方法。此方法接受目前的索引名稱作為第一個引數，以及所需的名稱作為第二個引數：

```php
$table->renameIndex('from', 'to')
```


<a name="dropping-indexes"></a>
### 刪除索引

若要刪除索引，您必須指定索引的名稱。預設情況下，Laravel 會根據資料表名稱、索引欄位名稱和索引類型自動分配索引名稱。以下是一些範例：

<div class="overflow-auto">

| 指令                                                  | 描述                                                     |
| -------------------------------------------------------- | ----------------------------------------------------------- |
| `$table->dropPrimary('users_id_primary');`               | 從 "users" 資料表刪除主鍵。                              |
| `$table->dropUnique('users_email_unique');`              | 從 "users" 資料表刪除唯一索引。                           |
| `$table->dropIndex('geo_state_index');`                  | 從 "geo" 資料表刪除基本索引。                             |
| `$table->dropFullText('posts_body_fulltext');`           | 從 "posts" 資料表刪除全文字索引。                         |
| `$table->dropSpatialIndex('geo_location_spatialindex');` | 從 "geo" 資料表刪除空間索引 (SQLite 除外)。             |

</div>

如果您將欄位陣列傳遞給刪除索引的方法，則會根據資料表名稱、欄位和索引類型產生慣例的索引名稱：

```php
Schema::table('geo', function (Blueprint $table) {
    $table->dropIndex(['state']); // Drops index 'geo_state_index'
});
```

<a name="foreign-key-constraints"></a>
### 外鍵約束

Laravel 也提供建立外鍵約束的支援，用於在資料庫層級強制執行參照完整性。例如，讓我們在 `posts` 資料表上定義一個 `user_id` 欄位，用來參照 `users` 資料表的 `id` 欄位：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('posts', function (Blueprint $table) {
    $table->unsignedBigInteger('user_id');

    $table->foreign('user_id')->references('id')->on('users');
});
```

由於這種語法較為冗長，Laravel 提供了其他更簡潔的方法，利用慣例來提供更好的開發體驗。當使用 `foreignId` 方法來建立欄位時，上述範例可以改寫如下：

```php
Schema::table('posts', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained();
});
```

`foreignId` 方法會建立一個等同於 `UNSIGNED BIGINT` 的欄位，而 `constrained` 方法則會根據慣例來判定被參照的資料表與欄位。如果您的資料表名稱不符合 Laravel 的慣例，您可以手動將其提供給 `constrained` 方法。此外，也可以指定要分配給產生的索引之名稱：

```php
Schema::table('posts', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained(
        table: 'users', indexName: 'posts_user_id'
    );
});
```

您也可以指定約束的 "on delete" 與 "on update" 屬性的期望動作：

```php
$table->foreignId('user_id')
    ->constrained()
    ->onUpdate('cascade')
    ->onDelete('cascade');
```

這些動作也提供了一種替代的、更具表達力的語法：

<div class="overflow-auto">

| Method                        | Description                                       |
| ----------------------------- | ------------------------------------------------- |
| `$table->cascadeOnUpdate();`  | 更新應級聯。                                       |
| `$table->restrictOnUpdate();` | 更新應受限。                                       |
| `$table->nullOnUpdate();`     | 更新應將外鍵值設為 null。                         |
| `$table->noActionOnUpdate();` | 更新時不採取任何動作。                             |
| `$table->cascadeOnDelete();`  | 刪除應級聯。                                       |
| `$table->restrictOnDelete();` | 刪除應受限。                                       |
| `$table->nullOnDelete();`     | 刪除應將外鍵值設為 null。                         |
| `$table->noActionOnDelete();` | 如果子紀錄存在則防止刪除。                         |

</div>

任何額外的 [欄位修改器](#column-modifiers) 必須在 `constrained` 方法之前呼叫：

```php
$table->foreignId('user_id')
    ->nullable()
    ->constrained();
```


<a name="dropping-foreign-keys"></a>
#### 刪除外鍵

要刪除外鍵，您可以使用 `dropForeign` 方法，並將要刪除的外鍵約束名稱作為引數傳入。外鍵約束使用與索引相同的命名慣例。換句話說，外鍵約束名稱是基於資料表名稱與約束中的欄位，後接 "_foreign" 字尾：

```php
$table->dropForeign('posts_user_id_foreign');
```

或者，您也可以將包含持有外鍵之欄位名稱的陣列傳遞給 `dropForeign` 方法。該陣列將使用 Laravel 的約束命名慣例轉換為外鍵約束名稱：

```php
$table->dropForeign(['user_id']);
```


<a name="toggling-foreign-key-constraints"></a>
#### 切換外鍵約束

您可以使用以下方法在遷移中啟用或停用外鍵約束：

```php
Schema::enableForeignKeyConstraints();

Schema::disableForeignKeyConstraints();

Schema::withoutForeignKeyConstraints(function () {
    // Constraints disabled within this closure...
});
```

> [!WARNING]
> SQLite 預設會停用外鍵約束。使用 SQLite 時，請在嘗試於遷移中建立外鍵之前，確保已在資料庫設定中 [啟用外鍵支援](/docs/{{version}}/database#configuration)。

<a name="events"></a>
## 事件

為了方便起見，每次遷移操作都會發送一個 [事件](/docs/{{version}}/events)。以下所有事件都繼承自基礎的 `Illuminate\Database\Events\MigrationEvent` 類別：

<div class="overflow-auto">

| 類別                                             | 說明                                           |
| ------------------------------------------------ | ------------------------------------------------ |
| `Illuminate\Database\Events\MigrationsStarted`   | 一批遷移即將開始執行。                           |
| `Illuminate\Database\Events\MigrationsEnded`     | 一批遷移已執行完畢。                              |
| `Illuminate\Database\Events\MigrationStarted`    | 單個遷移即將開始執行。                           |
| `Illuminate\Database\Events\MigrationEnded`      | 單個遷移已執行完畢。                              |
| `Illuminate\Database\Events\NoPendingMigrations` | 遷移指令未發現任何待處理的遷移。                 |
| `Illuminate\Database\Events\SchemaDumped`        | 資料庫結構傾印 (schema dump) 已完成。            |
| `Illuminate\Database\Events\SchemaLoaded`        | 現有的資料庫結構傾印 (schema dump) 已載入。     |

</div>