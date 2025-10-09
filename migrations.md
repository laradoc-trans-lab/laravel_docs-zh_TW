# 資料庫：遷移

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
- [資料欄](#columns)
    - [建立資料欄](#creating-columns)
    - [可用的資料欄類型](#available-column-types)
    - [資料欄修飾符](#column-modifiers)
    - [修改資料欄](#modifying-columns)
    - [重新命名資料欄](#renaming-columns)
    - [刪除資料欄](#dropping-columns)
- [索引](#indexes)
    - [建立索引](#creating-indexes)
    - [重新命名索引](#renaming-indexes)
    - [刪除索引](#dropping-indexes)
    - [外部鍵限制](#foreign-key-constraints)
- [事件](#events)

<a name="introduction"></a>
## 簡介

遷移就像您資料庫的版本控制，允許您的團隊定義並共享應用程式的資料庫結構定義。如果您曾經在從版本控制拉取變更後，不得不告訴隊友手動在他們本地資料庫結構定義中新增一個資料欄，那麼您就已經面臨過資料庫遷移所解決的問題。

Laravel `Schema` [外觀](/docs/{{version}}/facades)為所有 Laravel 支援的資料庫系統提供了與資料庫無關的支援，用於建立和操作資料表。通常，遷移會使用此外觀來建立和修改資料庫資料表和資料欄。

<a name="generating-migrations"></a>
## 生成遷移

您可以使用 `make:migration` [Artisan 指令](/docs/{{version}}/artisan)來生成資料庫遷移。新的遷移將會放置在您的 `database/migrations` 目錄中。每個遷移檔案名稱都包含一個時間戳記，允許 Laravel 判斷遷移的順序：

```shell
php artisan make:migration create_flights_table
```

Laravel 會使用遷移的名稱來嘗試猜測資料表的名稱，以及該遷移是否會建立一個新的資料表。如果 Laravel 能夠從遷移名稱中確定資料表名稱，Laravel 會使用指定的資料表預填生成的遷移檔案。否則，您可以手動在遷移檔案中指定資料表。

如果您想為生成的遷移指定一個自訂路徑，可以在執行 `make:migration` 指令時使用 `--path` 選項。給定的路徑應相對於您的應用程式基礎路徑。

> [!NOTE]
> 遷移 stub 可透過 [stub publishing](/docs/{{version}}/artisan#stub-customization) 進行客製化。

<a name="squashing-migrations"></a>
### 壓縮遷移

隨著您建立應用程式，您可能會隨著時間累積越來越多的遷移。這可能導致您的 `database/migrations` 目錄變得臃腫，包含數百個遷移。如果您願意，您可以將您的遷移「壓縮」成單一 SQL 檔案。要開始，請執行 `schema:dump` 指令：

```shell
php artisan schema:dump

# Dump the current database schema and prune all existing migrations...
php artisan schema:dump --prune
```

當您執行此指令時，Laravel 會將一個「結構定義」檔案寫入應用程式的 `database/schema` 目錄。結構定義檔案的名稱會對應到資料庫連線。現在，當您嘗試遷移您的資料庫且尚未執行任何其他遷移時，Laravel 會首先執行您正在使用的資料庫連線的結構定義檔案中的 SQL 語句。在執行完結構定義檔案的 SQL 語句後，Laravel 會執行任何不屬於結構定義傾印的剩餘遷移。

如果您的應用程式測試使用與您在本地開發期間通常使用的資料庫連線不同的連線，您應該確保您已經使用該資料庫連線傾印了一個結構定義檔案，以便您的測試能夠建構您的資料庫。您可能希望在傾印您在本地開發期間通常使用的資料庫連線之後執行此操作：

```shell
php artisan schema:dump
php artisan schema:dump --database=testing --prune
```

您應該將您的資料庫結構定義檔案提交到版本控制，以便團隊中的其他新開發人員可以快速建立應用程式的初始資料庫結構。

> [!WARNING]
> 遷移壓縮僅適用於 MariaDB、MySQL、PostgreSQL 和 SQLite 資料庫，並利用資料庫的命令列用戶端。

<a name="migration-structure"></a>
## 遷移結構

一個遷移類別包含兩個方法：`up` 和 `down`。`up` 方法用於向您的資料庫新增資料表、資料欄或索引，而 `down` 方法應該反轉 `up` 方法執行的操作。

在這兩個方法中，您都可以使用 Laravel 結構定義建構器來表達性地建立和修改資料表。要了解 `Schema` 建構器上所有可用的方法，[請查閱其文件](#creating-tables)。例如，以下遷移建立了一個 `flights` 資料表：

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

有時，某個遷移可能旨在支援尚未啟用的功能，而您不希望它立即執行。在這種情況下，您可以在遷移上定義一個 `shouldRun` 方法。如果 `shouldRun` 方法回傳 `false`，則該遷移將被跳過：

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

若要執行所有尚未執行的遷移，請執行 `migrate` Artisan 命令：

```shell
php artisan migrate
```

若您想查看目前已執行的遷移，您可以使用 `migrate:status` Artisan 命令：

```shell
php artisan migrate:status
```

若您想查看遷移將會執行的 SQL 語句，而不實際執行它們，您可以為 `migrate` 命令提供 `--pretend` 旗標：

```shell
php artisan migrate --pretend
```

#### 隔離遷移執行

若您在多個伺服器上部署應用程式，並將遷移作為部署流程的一部分執行，您可能不希望兩個伺服器同時嘗試遷移資料庫。為避免這種情況，您可以在呼叫 `migrate` 命令時使用 `isolated` 選項。

提供 `isolated` 選項後，Laravel 會在使用應用程式的快取驅動獲取一個原子鎖，然後才會嘗試執行您的遷移。鎖定被佔用期間，所有其他嘗試執行 `migrate` 命令的行為都不會執行；然而，該命令仍將以成功的退出狀態碼退出：

```shell
php artisan migrate --isolated
```

> [!WARNING]
> 若要利用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動作為您的應用程式的預設快取驅動。此外，所有伺服器都必須與相同的中央快取伺服器通訊。

<a name="forcing-migrations-to-run-in-production"></a>
#### 強制遷移在正式環境中執行

某些遷移操作是破壞性的，這表示它們可能會導致您遺失資料。為了保護您免於在正式資料庫上執行這些命令，在命令執行前會提示您進行確認。若要強制執行命令而無需提示，請使用 `--force` 旗標：

```shell
php artisan migrate --force
```

<a name="rolling-back-migrations"></a>
### 回溯遷移

若要回溯最新的遷移操作，您可以使用 `rollback` Artisan 命令。此命令會回溯最後一個「批次」的遷移，其中可能包含多個遷移檔案：

```shell
php artisan migrate:rollback
```

您可以透過為 `rollback` 命令提供 `step` 選項，來回溯有限數量的遷移。例如，以下命令將回溯最後五個遷移：

```shell
php artisan migrate:rollback --step=5
```

您可以透過為 `rollback` 命令提供 `batch` 選項，來回溯特定「批次」的遷移，其中 `batch` 選項對應於您應用程式的 `migrations` 資料庫資料表中的批次值。例如，以下命令將回溯第三個批次中的所有遷移：

```shell
php artisan migrate:rollback --batch=3
```

若您想查看遷移將會執行的 SQL 語句，而不實際執行它們，您可以為 `migrate:rollback` 命令提供 `--pretend` 旗標：

```shell
php artisan migrate:rollback --pretend
```

`migrate:reset` 命令將回溯您應用程式的所有遷移：

```shell
php artisan migrate:reset
```

<a name="roll-back-migrate-using-a-single-command"></a>
#### 使用單一命令回溯並遷移

`migrate:refresh` 命令將回溯您所有的遷移，然後執行 `migrate` 命令。此命令會有效地重建您的整個資料庫：

```shell
php artisan migrate:refresh

# Refresh the database and run all database seeds...
php artisan migrate:refresh --seed
```

您可以透過為 `refresh` 命令提供 `step` 選項，來回溯並重新遷移有限數量的遷移。例如，以下命令將回溯並重新遷移最後五個遷移：

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

預設情況下，`migrate:fresh` 命令只會從預設資料庫連線中刪除資料表。然而，您可以使用 `--database` 選項來指定應遷移的資料庫連線。資料庫連線名稱應對應於您應用程式 `database` [設定檔](/docs/{{version}}/configuration)中定義的連線：

```shell
php artisan migrate:fresh --database=admin
```

> [!WARNING]
> `migrate:fresh` 命令將無論其前綴如何，都會刪除所有資料庫資料表。在與其他應用程式共享的資料庫上進行開發時，應謹慎使用此命令。

<a name="tables"></a>
## 資料表

<a name="creating-tables"></a>
### 建立資料表

要建立新的資料庫資料表，請使用 `Schema` Facade 上的 `create` 方法。`create` 方法接受兩個引數：第一個是資料表的名稱，第二個是一個閉包 (closure)，此閉包會收到一個 `Blueprint` 物件，可用於定義新的資料表：

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

建立資料表時，您可以使用 Schema Builder 的任何[資料欄方法](#creating-columns)來定義資料表的資料欄。

<a name="determining-table-column-existence"></a>
#### 判斷資料表 / 資料欄是否存在

您可以使用 `hasTable`、`hasColumn` 和 `hasIndex` 方法來判斷資料表、資料欄或索引是否存在：

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

如果您想對非應用程式預設連線的資料庫連線執行 Schema 操作，請使用 `connection` 方法：

```php
Schema::connection('sqlite')->create('users', function (Blueprint $table) {
    $table->id();
});
```

此外，還可以使用其他一些屬性和方法來定義資料表建立的其他方面。當使用 MariaDB 或 MySQL 時，`engine` 屬性可用於指定資料表的儲存引擎：

```php
Schema::create('users', function (Blueprint $table) {
    $table->engine('InnoDB');

    // ...
});
```

當使用 MariaDB 或 MySQL 時，`charset` 和 `collation` 屬性可用於指定所建立資料表的字元集和定序：

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

如果您想為資料庫資料表新增「註解」，可以呼叫資料表實例上的 `comment` 方法。資料表註解目前僅支援 MariaDB、MySQL 和 PostgreSQL：

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->comment('Business calculations');

    // ...
});
```

<a name="updating-tables"></a>
### 更新資料表

`Schema` Facade 上的 `table` 方法可用於更新現有的資料表。與 `create` 方法一樣，`table` 方法接受兩個引數：資料表的名稱和一個閉包，此閉包會收到一個 `Blueprint` 實例，您可以使用它來新增資料欄或索引到資料表：

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

要刪除現有的資料表，您可以使用 `drop` 或 `dropIfExists` 方法：

```php
Schema::drop('users');

Schema::dropIfExists('users');
```

<a name="renaming-tables-with-foreign-keys"></a>
#### 重新命名包含外部鍵的資料表

在重新命名資料表之前，您應該驗證資料表上的任何外部鍵限制在您的遷移檔案中是否有明確的名稱，而不是讓 Laravel 指定基於約定的名稱。否則，外部鍵限制名稱將會參考舊的資料表名稱。

<a name="columns"></a>
## 資料欄


<a name="creating-columns"></a>
### 建立資料欄

`Schema` Facade 上的 `table` 方法可用於更新現有資料表。就像 `create` 方法一樣，`table` 方法接受兩個引數：資料表的名稱，以及一個接收 `Illuminate\Database\Schema\Blueprint` 實例的閉包，您可以使用該實例向資料表添加資料欄：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->integer('votes');
});
```

<a name="available-column-types"></a>
### 可用的資料欄類型

綱要建構器藍圖 (schema builder blueprint) 提供了多種方法，對應可添加到資料庫資料表中的各種資料欄類型。下表列出了所有可用的方法：

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

`bigIncrements` 方法會建立一個自動遞增的 `UNSIGNED BIGINT` (主鍵) 等效資料欄：

```php
$table->bigIncrements('id');
```


<a name="column-method-bigInteger"></a>
#### `bigInteger()` {.collection-method}

`bigInteger` 方法會建立一個 `BIGINT` 等效資料欄：

```php
$table->bigInteger('votes');
```


<a name="column-method-binary"></a>
#### `binary()` {.collection-method}

`binary` 方法會建立一個 `BLOB` 等效資料欄：

```php
$table->binary('photo');
```

當使用 MySQL、MariaDB 或 SQL Server 時，您可以傳遞 `length` 和 `fixed` 引數以建立 `VARBINARY` 或 `BINARY` 等效資料欄：

```php
$table->binary('data', length: 16); // VARBINARY(16)

$table->binary('data', length: 16, fixed: true); // BINARY(16)
```


<a name="column-method-boolean"></a>
#### `boolean()` {.collection-method}

`boolean` 方法會建立一個 `BOOLEAN` 等效資料欄：

```php
$table->boolean('confirmed');
```


<a name="column-method-char"></a>
#### `char()` {.collection-method}

`char` 方法會建立一個具有指定長度的 `CHAR` 等效資料欄：

```php
$table->char('name', length: 100);
```


<a name="column-method-dateTimeTz"></a>
#### `dateTimeTz()` {.collection-method}

`dateTimeTz` 方法會建立一個 `DATETIME` (帶時區) 等效資料欄，並可選地帶有秒數小數精確度：

```php
$table->dateTimeTz('created_at', precision: 0);
```


<a name="column-method-dateTime"></a>
#### `dateTime()` {.collection-method}

`dateTime` 方法會建立一個 `DATETIME` 等效資料欄，並可選地帶有秒數小數精確度：

```php
$table->dateTime('created_at', precision: 0);
```


<a name="column-method-date"></a>
#### `date()` {.collection-method}

`date` 方法會建立一個 `DATE` 等效資料欄：

```php
$table->date('created_at');
```


<a name="column-method-decimal"></a>
#### `decimal()` {.collection-method}

`decimal` 方法會建立一個 `DECIMAL` 等效資料欄，具有指定的總位數 (precision) 和小數位數 (scale)：

```php
$table->decimal('amount', total: 8, places: 2);
```


<a name="column-method-double"></a>
#### `double()` {.collection-method}

`double` 方法會建立一個 `DOUBLE` 等效資料欄：

```php
$table->double('amount');
```


<a name="column-method-enum"></a>
#### `enum()` {.collection-method}

`enum` 方法會建立一個 `ENUM` 等效資料欄，具有指定的有效值：

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

`float` 方法會建立一個 `FLOAT` 等效資料欄，具有指定的精確度：

```php
$table->float('amount', precision: 53);
```


<a name="column-method-foreignId"></a>
#### `foreignId()` {.collection-method}

`foreignId` 方法會建立一個 `UNSIGNED BIGINT` 等效資料欄：

```php
$table->foreignId('user_id');
```


<a name="column-method-foreignIdFor"></a>
#### `foreignIdFor()` {.collection-method}

`foreignIdFor` 方法會為給定的模型類別增加一個 `{column}_id` 等效資料欄。此資料欄的類型將根據模型鍵的類型而定，可以是 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`：

```php
$table->foreignIdFor(User::class);
```


<a name="column-method-foreignUlid"></a>
#### `foreignUlid()` {.collection-method}

`foreignUlid` 方法會建立一個 `ULID` 等效資料欄：

```php
$table->foreignUlid('user_id');
```


<a name="column-method-foreignUuid"></a>
#### `foreignUuid()` {.collection-method}

`foreignUuid` 方法會建立一個 `UUID` 等效資料欄：

```php
$table->foreignUuid('user_id');
```


<a name="column-method-geography"></a>
#### `geography()` {.collection-method}

`geography` 方法會建立一個 `GEOGRAPHY` 等效資料欄，具有指定的空間類型和 SRID (空間參考系統識別碼)：

```php
$table->geography('coordinates', subtype: 'point', srid: 4326);
```

> [!NOTE]
> 空間類型支援取決於您的資料庫驅動程式。請參閱您的資料庫文件。如果您的應用程式使用 PostgreSQL 資料庫，您必須在可以使用 `geography` 方法之前安裝 [PostGIS](https://postgis.net) 擴充功能。


<a name="column-method-geometry"></a>
#### `geometry()` {.collection-method}

`geometry` 方法會建立一個 `GEOMETRY` 等效資料欄，具有指定的空間類型和 SRID (空間參考系統識別碼)：

```php
$table->geometry('positions', subtype: 'point', srid: 0);
```

> [!NOTE]
> 空間類型支援取決於您的資料庫驅動程式。請參閱您的資料庫文件。如果您的應用程式使用 PostgreSQL 資料庫，您必須在可以使用 `geometry` 方法之前安裝 [PostGIS](https://postgis.net) 擴充功能。


<a name="column-method-id"></a>
#### `id()` {.collection-method}

`id` 方法是 `bigIncrements` 方法的別名。預設情況下，該方法將建立一個 `id` 資料欄；但是，如果你想為該資料欄指定不同的名稱，可以傳遞一個資料欄名稱：

```php
$table->id();
```


<a name="column-method-increments"></a>
#### `increments()` {.collection-method}

`increments` 方法會建立一個自動遞增的 `UNSIGNED INTEGER` 等效資料欄作為主鍵：

```php
$table->increments('id');
```


<a name="column-method-integer"></a>
#### `integer()` {.collection-method}

`integer` 方法會建立一個 `INTEGER` 等效資料欄：

```php
$table->integer('votes');
```


<a name="column-method-ipAddress"></a>
#### `ipAddress()` {.collection-method}

`ipAddress` 方法會建立一個 `VARCHAR` 等效資料欄：

```php
$table->ipAddress('visitor');
```

當使用 PostgreSQL 時，將會建立一個 `INET` 資料欄。


<a name="column-method-json"></a>
#### `json()` {.collection-method}

`json` 方法會建立一個 `JSON` 等效資料欄：

```php
$table->json('options');
```

當使用 SQLite 時，將會建立一個 `TEXT` 資料欄。


<a name="column-method-jsonb"></a>
#### `jsonb()` {.collection-method}

`jsonb` 方法會建立一個 `JSONB` 等效資料欄：

```php
$table->jsonb('options');
```

當使用 SQLite 時，將會建立一個 `TEXT` 資料欄。


<a name="column-method-longText"></a>
#### `longText()` {.collection-method}

`longText` 方法會建立一個 `LONGTEXT` 等效資料欄：

```php
$table->longText('description');
```

當使用 MySQL 或 MariaDB 時，你可以對該資料欄應用 `binary` 字元集，以建立一個 `LONGBLOB` 等效資料欄：

```php
$table->longText('data')->charset('binary'); // LONGBLOB
```


<a name="column-method-macAddress"></a>
#### `macAddress()` {.collection-method}

`macAddress` 方法會建立一個旨在儲存 MAC 位址的資料欄。某些資料庫系統 (例如 PostgreSQL) 具有專用於此類資料的資料欄類型。其他資料庫系統將使用字串等效資料欄：

```php
$table->macAddress('device');
```


<a name="column-method-mediumIncrements"></a>
#### `mediumIncrements()` {.collection-method}

`mediumIncrements` 方法會建立一個自動遞增的 `UNSIGNED MEDIUMINT` 等效資料欄作為主鍵：

```php
$table->mediumIncrements('id');
```


<a name="column-method-mediumInteger"></a>
#### `mediumInteger()` {.collection-method}

`mediumInteger` 方法會建立一個 `MEDIUMINT` 等效資料欄：

```php
$table->mediumInteger('votes');
```


<a name="column-method-mediumText"></a>
#### `mediumText()` {.collection-method}

`mediumText` 方法會建立一個 `MEDIUMTEXT` 等效資料欄：

```php
$table->mediumText('description');
```

當使用 MySQL 或 MariaDB 時，你可以對該資料欄應用 `binary` 字元集，以建立一個 `MEDIUMBLOB` 等效資料欄：

```php
$table->mediumText('data')->charset('binary'); // MEDIUMBLOB
```


<a name="column-method-morphs"></a>
#### `morphs()` {.collection-method}

`morphs` 方法是一個方便的方法，它會增加一個 `{column}_id` 等效資料欄和一個 `{column}_type` `VARCHAR` 等效資料欄。`{column}_id` 的資料欄類型將根據模型鍵的類型而定，可以是 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`。

此方法旨在用於定義多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships) 所需的資料欄。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 資料欄：

```php
$table->morphs('taggable');
```


<a name="column-method-nullableMorphs"></a>
#### `nullableMorphs()` {.collection-method}

此方法類似於 [morphs](#column-method-morphs) 方法；然而，所建立的資料欄將是「可為空 (nullable)」的：

```php
$table->nullableMorphs('taggable');
```


<a name="column-method-nullableUlidMorphs"></a>
#### `nullableUlidMorphs()` {.collection-method}

此方法類似於 [ulidMorphs](#column-method-ulidMorphs) 方法；然而，所建立的資料欄將是「可為空 (nullable)」的：

```php
$table->nullableUlidMorphs('taggable');
```


<a name="column-method-nullableUuidMorphs"></a>
#### `nullableUuidMorphs()` {.collection-method}

此方法類似於 [uuidMorphs](#column-method-uuidMorphs) 方法；然而，所建立的資料欄將是「可為空 (nullable)」的：

```php
$table->nullableUuidMorphs('taggable');
```


<a name="column-method-rememberToken"></a>
#### `rememberToken()` {.collection-method}

`rememberToken` 方法會建立一個可為空、`VARCHAR(100)` 等效資料欄，旨在儲存當前的「記住我」[認證權杖](/docs/{{version}}/authentication#remembering-users)：

```php
$table->rememberToken();
```


<a name="column-method-set"></a>
#### `set()` {.collection-method}

`set` 方法會建立一個 `SET` 等效資料欄，具有指定的有效值列表：

```php
$table->set('flavors', ['strawberry', 'vanilla']);
```


<a name="column-method-smallIncrements"></a>
#### `smallIncrements()` {.collection-method}

`smallIncrements` 方法會建立一個自動遞增的 `UNSIGNED SMALLINT` 等效資料欄作為主鍵：

```php
$table->smallIncrements('id');
```


<a name="column-method-smallInteger"></a>
#### `smallInteger()` {.collection-method}

`smallInteger` 方法會建立一個 `SMALLINT` 等效資料欄：

```php
$table->smallInteger('votes');
```


<a name="column-method-softDeletesTz"></a>
#### `softDeletesTz()` {.collection-method}

`softDeletesTz` 方法會增加一個可為空、`deleted_at` `TIMESTAMP` (帶時區) 等效資料欄，並可選地帶有秒數小數精確度。此資料欄旨在儲存 Eloquent 「軟刪除」功能所需的 `deleted_at` 時間戳記：

```php
$table->softDeletesTz('deleted_at', precision: 0);
```


<a name="column-method-softDeletes"></a>
#### `softDeletes()` {.collection-method}

`softDeletes` 方法會增加一個可為空、`deleted_at` `TIMESTAMP` 等效資料欄，並可選地帶有秒數小數精確度。此資料欄旨在儲存 Eloquent 「軟刪除」功能所需的 `deleted_at` 時間戳記：

```php
$table->softDeletes('deleted_at', precision: 0);
```


<a name="column-method-string"></a>
#### `string()` {.collection-method}

`string` 方法會建立一個指定長度的 `VARCHAR` 等效資料欄：

```php
$table->string('name', length: 100);
```


<a name="column-method-text"></a>
#### `text()` {.collection-method}

`text` 方法會建立一個 `TEXT` 等效資料欄：

```php
$table->text('description');
```

當使用 MySQL 或 MariaDB 時，你可以對該資料欄應用 `binary` 字元集，以建立一個 `BLOB` 等效資料欄：

```php
$table->text('data')->charset('binary'); // BLOB
```


<a name="column-method-timeTz"></a>
#### `timeTz()` {.collection-method}

`timeTz` 方法會建立一個 `TIME` (帶時區) 等效資料欄，並可選地帶有秒數小數精確度：

```php
$table->timeTz('sunrise', precision: 0);
```


<a name="column-method-time"></a>
#### `time()` {.collection-method}

`time` 方法會建立一個 `TIME` 等效資料欄，並可選地帶有秒數小數精確度：

```php
$table->time('sunrise', precision: 0);
```


<a name="column-method-timestampTz"></a>
#### `timestampTz()` {.collection-method}

`timestampTz` 方法會建立一個 `TIMESTAMP` (帶時區) 等效資料欄，並可選地帶有秒數小數精確度：

```php
$table->timestampTz('added_at', precision: 0);
```


<a name="column-method-timestamp"></a>
#### `timestamp()` {.collection-method}

`timestamp` 方法會建立一個 `TIMESTAMP` 等效資料欄，並可選地帶有秒數小數精確度：

```php
$table->timestamp('added_at', precision: 0);
```


<a name="column-method-timestampsTz"></a>
#### `timestampsTz()` {.collection-method}

`timestampsTz` 方法會建立 `created_at` 和 `updated_at` `TIMESTAMP` (帶時區) 等效資料欄，並可選地帶有秒數小數精確度：

```php
$table->timestampsTz(precision: 0);
```


<a name="column-method-timestamps"></a>
#### `timestamps()` {.collection-method}

`timestamps` 方法會建立 `created_at` 和 `updated_at` `TIMESTAMP` 等效資料欄，並可選地帶有秒數小數精確度：

```php
$table->timestamps(precision: 0);
```


<a name="column-method-tinyIncrements"></a>
#### `tinyIncrements()` {.collection-method}

`tinyIncrements` 方法會建立一個自動遞增的 `UNSIGNED TINYINT` 等效資料欄作為主鍵：

```php
$table->tinyIncrements('id');
```


<a name="column-method-tinyInteger"></a>
#### `tinyInteger()` {.collection-method}

`tinyInteger` 方法會建立一個 `TINYINT` 等效資料欄：

```php
$table->tinyInteger('votes');
```


<a name="column-method-tinyText"></a>
#### `tinyText()` {.collection-method}

`tinyText` 方法會建立一個 `TINYTEXT` 等效資料欄：

```php
$table->tinyText('notes');
```

當使用 MySQL 或 MariaDB 時，你可以對該資料欄應用 `binary` 字元集，以建立一個 `TINYBLOB` 等效資料欄：

```php
$table->tinyText('data')->charset('binary'); // TINYBLOB
```


<a name="column-method-unsignedBigInteger"></a>
#### `unsignedBigInteger()` {.collection-method}

`unsignedBigInteger` 方法會建立一個 `UNSIGNED BIGINT` 等效資料欄：

```php
$table->unsignedBigInteger('votes');
```


<a name="column-method-unsignedInteger"></a>
#### `unsignedInteger()` {.collection-method}

`unsignedInteger` 方法會建立一個 `UNSIGNED INTEGER` 等效資料欄：

```php
$table->unsignedInteger('votes');
```


<a name="column-method-unsignedMediumInteger"></a>
#### `unsignedMediumInteger()` {.collection-method}

`unsignedMediumInteger` 方法會建立一個 `UNSIGNED MEDIUMINT` 等效資料欄：

```php
$table->unsignedMediumInteger('votes');
```


<a name="column-method-unsignedSmallInteger"></a>
#### `unsignedSmallInteger()` {.collection-method}

`unsignedSmallInteger` 方法會建立一個 `UNSIGNED SMALLINT` 等效資料欄：

```php
$table->unsignedSmallInteger('votes');
```


<a name="column-method-unsignedTinyInteger"></a>
#### `unsignedTinyInteger()` {.collection-method}

`unsignedTinyInteger` 方法會建立一個 `UNSIGNED TINYINT` 等效資料欄：

```php
$table->unsignedTinyInteger('votes');
```


<a name="column-method-ulidMorphs"></a>
#### `ulidMorphs()` {.collection-method}

`ulidMorphs` 方法是一個方便的方法，它會增加一個 `{column}_id` `CHAR(26)` 等效資料欄和一個 `{column}_type` `VARCHAR` 等效資料欄。

此方法旨在用於定義多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships) 所需的資料欄，這些關聯使用 ULID 識別符。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 資料欄：

```php
$table->ulidMorphs('taggable');
```


<a name="column-method-uuidMorphs"></a>
#### `uuidMorphs()` {.collection-method}

`uuidMorphs` 方法是一個方便的方法，它會增加一個 `{column}_id` `CHAR(36)` 等效資料欄和一個 `{column}_type` `VARCHAR` 等效資料欄。

此方法旨在用於定義多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships) 所需的資料欄，這些關聯使用 UUID 識別符。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 資料欄：

```php
$table->uuidMorphs('taggable');
```


<a name="column-method-ulid"></a>
#### `ulid()` {.collection-method}

`ulid` 方法會建立一個 `ULID` 等效資料欄：

```php
$table->ulid('id');
```


<a name="column-method-uuid"></a>
#### `uuid()` {.collection-method}

`uuid` 方法會建立一個 `UUID` 等效資料欄：

```php
$table->uuid('id');
```


<a name="column-method-vector"></a>
#### `vector()` {.collection-method}

`vector` 方法會建立一個 `vector` 等效資料欄：

```php
$table->vector('embedding', dimensions: 100);
```


<a name="column-method-year"></a>
#### `year()` {.collection-method}

`year` 方法會建立一個 `YEAR` 等效資料欄：

```php
$table->year('birth_year');
```

<a name="column-modifiers"></a>
### 資料欄修飾符

除了上方列出的資料欄類型之外，還有數種資料欄「修飾符」可用於向資料庫資料表新增資料欄。例如，若要讓資料欄「可為 Null」，您可以使用 `nullable` 方法：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->string('email')->nullable();
});
```

下表包含所有可用的資料欄修飾符。此列表不包含[索引修飾符](#creating-indexes)：

<div class="overflow-auto">

| Modifier                            | Description                                                                                    |
| ----------------------------------- | ---------------------------------------------------------------------------------------------- |
| `->after('column')`                 | 將資料欄置於另一個資料欄「之後」(MariaDB / MySQL)。                                     |
| `->autoIncrement()`                 | 將 `INTEGER` 資料欄設定為自動遞增 (主鍵)。                                      |
| `->charset('utf8mb4')`              | 指定資料欄的字元集 (MariaDB / MySQL)。                                          |
| `->collation('utf8mb4_unicode_ci')` | 指定資料欄的排序規則。                                                            |
| `->comment('my comment')`           | 為資料欄新增註解 (MariaDB / MySQL / PostgreSQL)。                                      |
| `->default($value)`                 | 為資料欄指定「預設」值。                                                      |
| `->first()`                         | 將資料欄置於資料表「最前面」(MariaDB / MySQL)。                                       |
| `->from($integer)`                  | 設定自動遞增欄位的起始值 (MariaDB / MySQL / PostgreSQL)。                           |
| `->invisible()`                     | 使資料欄對 `SELECT *` 查詢「不可見」(MariaDB / MySQL)。                           |
| `->nullable($value = true)`         | 允許資料欄插入 `NULL` 值。                                            |
| `->storedAs($expression)`           | 建立儲存的生成資料欄 (MariaDB / MySQL / PostgreSQL / SQLite)。                      |
| `->unsigned()`                      | 將 `INTEGER` 資料欄設定為 `UNSIGNED` (MariaDB / MySQL)。                                         |
| `->useCurrent()`                    | 將 `TIMESTAMP` 資料欄設定為使用 `CURRENT_TIMESTAMP` 作為預設值。                           |
| `->useCurrentOnUpdate()`            | 在記錄更新時，將 `TIMESTAMP` 資料欄設定為使用 `CURRENT_TIMESTAMP` (MariaDB / MySQL)。 |
| `->virtualAs($expression)`          | 建立虛擬的生成資料欄 (MariaDB / MySQL / SQLite)。                                  |
| `->generatedAs($expression)`        | 建立具有指定序列選項的識別資料欄 (PostgreSQL)。                        |
| `->always()`                        | 定義識別資料欄的序列值相對於輸入的優先順序 (PostgreSQL)。      |

</div>


<a name="default-expressions"></a>
#### 預設表達式

`default` 修飾符接受一個值或一個 `Illuminate\Database\Query\Expression` 實例。使用 `Expression` 實例將阻止 Laravel 將值用引號包裝，並允許您使用資料庫特定函數。這在需要為 JSON 資料欄指定預設值時特別有用：

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
> 預設表達式的支援程度取決於您的資料庫驅動程式、資料庫版本和欄位類型。請參閱您的資料庫文件。


<a name="column-order"></a>
#### 資料欄順序

使用 MariaDB 或 MySQL 資料庫時，可以使用 `after` 方法在 Schema 中現有資料欄之後新增資料欄：

```php
$table->after('password', function (Blueprint $table) {
    $table->string('address_line1');
    $table->string('address_line2');
    $table->string('city');
});
```


<a name="modifying-columns"></a>
### 修改資料欄

`change` 方法允許您修改現有資料欄的類型和屬性。例如，您可能希望增加 `string` 資料欄的大小。要查看 `change` 方法的實際應用，我們將 `name` 資料欄的大小從 25 增加到 50。為此，我們只需定義資料欄的新狀態，然後呼叫 `change` 方法：

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('name', 50)->change();
});
```

修改資料欄時，您必須明確包含要保留在資料欄定義上的所有修飾符——任何遺漏的屬性都將被捨棄。例如，若要保留 `unsigned`、`default` 和 `comment` 屬性，您在修改資料欄時必須明確呼叫每個修飾符：

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')->unsigned()->default(1)->comment('my comment')->change();
});
```

`change` 方法不會更改資料欄的索引。因此，您在修改資料欄時可以使用索引修飾符明確新增或刪除索引：

```php
// Add an index...
$table->bigIncrements('id')->primary()->change();

// Drop an index...
$table->char('postal_code', 10)->unique(false)->change();
```


<a name="renaming-columns"></a>
### 重新命名資料欄

若要重新命名資料欄，您可以使用 Schema Builder 提供的 `renameColumn` 方法：

```php
Schema::table('users', function (Blueprint $table) {
    $table->renameColumn('from', 'to');
});
```


<a name="dropping-columns"></a>
### 刪除資料欄

若要刪除資料欄，您可以使用 Schema Builder 上的 `dropColumn` 方法：

```php
Schema::table('users', function (Blueprint $table) {
    $table->dropColumn('votes');
});
```

您可以透過向 `dropColumn` 方法傳遞資料欄名稱陣列，從資料表刪除多個資料欄：

```php
Schema::table('users', function (Blueprint $table) {
    $table->dropColumn(['votes', 'avatar', 'location']);
});
```


<a name="available-command-aliases"></a>
#### 可用命令別名

Laravel 提供了數種與刪除常見資料欄類型相關的便捷方法。下表描述了這些方法：

<div class="overflow-auto">

| Command                             | Description                                           |
| ----------------------------------- | ----------------------------------------------------- |
| `$table->dropMorphs('morphable');`  | 刪除 `morphable_id` 和 `morphable_type` 資料欄。 |
| `$table->dropRememberToken();`      | 刪除 `remember_token` 資料欄。                     |
| `$table->dropSoftDeletes();`        | 刪除 `deleted_at` 資料欄。                         |
| `$table->dropSoftDeletesTz();`      | `dropSoftDeletes()` 方法的別名。                  |
| `$table->dropTimestamps();`         | 刪除 `created_at` 和 `updated_at` 資料欄。       |
| `$table->dropTimestampsTz();`       | `dropTimestamps()` 方法的別名。                   |

</div>

<a name="indexes"></a>
## 索引

<a name="creating-indexes"></a>
### 建立索引

Laravel 的 schema builder 支援多種索引類型。以下範例會建立一個新的 `email` 資料欄，並指定其值應該是 unique 的。要建立索引，我們可以將 `unique` 方法鏈接到資料欄定義上：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->string('email')->unique();
});
```

另外，你也可以在定義資料欄之後再建立索引。為此，你應該在 schema builder blueprint 上呼叫 `unique` 方法。此方法接受應該接收 unique 索引的資料欄名稱：

```php
$table->unique('email');
```

你甚至可以將一個資料欄陣列傳遞給索引方法，以建立複合式 (compound 或 composite) 索引：

```php
$table->index(['account_id', 'created_at']);
```

建立索引時，Laravel 會根據資料表、資料欄名稱和索引類型自動產生索引名稱，但你也可以傳遞第二個參數給該方法，來自訂索引名稱：

```php
$table->unique('email', 'unique_email');
```

<a name="available-index-types"></a>
#### 可用的索引類型

Laravel 的 schema builder Blueprint class 提供了用於建立 Laravel 支援的每種索引類型的方法。每個索引方法都接受一個可選的第二個參數，以指定索引的名稱。如果省略，名稱將根據資料表、用於索引的資料欄名稱以及索引類型推導而來。每個可用的索引方法都將在下表中描述：

<div class="overflow-auto">

| Command                                          | Description                                       |
| ------------------------------------------------ | ------------------------------------------------- |
| `$table->primary('id');`                         | 新增 primary key。                                |
| `$table->primary(['id', 'parent_id']);`          | 新增複合鍵。                                      |
| `$table->unique('email');`                       | 新增 unique 索引。                                |
| `$table->index('state');`                        | 新增索引。                                        |
| `$table->fullText('body');`                      | 新增全文索引 (MariaDB / MySQL / PostgreSQL)。     |
| `$table->fullText('body')->language('english');` | 新增指定語言的全文索引 (PostgreSQL)。             |
| `$table->spatialIndex('location');`              | 新增空間索引 (SQLite 除外)。                      |

</div>

<a name="renaming-indexes"></a>
### 重新命名索引

要重新命名索引，你可以使用 schema builder blueprint 提供的 `renameIndex` 方法。此方法接受目前的索引名稱作為第一個參數，以及期望的名稱作為第二個參數：

```php
$table->renameIndex('from', 'to')
```

<a name="dropping-indexes"></a>
### 刪除索引

要刪除索引，你必須指定索引的名稱。預設情況下，Laravel 會根據資料表名稱、索引資料欄名稱和索引類型自動分配索引名稱。以下是一些範例：

<div class="overflow-auto">

| Command                                                  | Description                                  |
| -------------------------------------------------------- | -------------------------------------------- |
| `$table->dropPrimary('users_id_primary');`               | 從 "users" 資料表刪除 primary key。          |
| `$table->dropUnique('users_email_unique');`              | 從 "users" 資料表刪除 unique 索引。          |
| `$table->dropIndex('geo_state_index');`                  | 從 "geo" 資料表刪除基本索引。                |
| `$table->dropFullText('posts_body_fulltext');`           | 從 "posts" 資料表刪除全文索引。              |
| `$table->dropSpatialIndex('geo_location_spatialindex');` | 從 "geo" 資料表刪除空間索引 (SQLite 除外)。 |

</div>

如果你將一個資料欄陣列傳遞給刪除索引的方法，則常規的索引名稱將根據資料表名稱、資料欄和索引類型生成：

```php
Schema::table('geo', function (Blueprint $table) {
    $table->dropIndex(['state']); // Drops index 'geo_state_index'
});
```

<a name="foreign-key-constraints"></a>
### 外部鍵限制

Laravel 也支援建立外部鍵限制，用於在資料庫層級強制實行參照完整性。舉例來說，讓我們在 `posts` 資料表上定義一個 `user_id` 資料欄，其參照 `users` 資料表上的 `id` 資料欄：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('posts', function (Blueprint $table) {
    $table->unsignedBigInteger('user_id');

    $table->foreign('user_id')->references('id')->on('users');
});
```

由於這種語法較為冗長，Laravel 提供了額外更簡潔的方法，利用慣例來提供更好的開發者體驗。當使用 `foreignId` 方法建立資料欄時，上述範例可以改寫成這樣：

```php
Schema::table('posts', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained();
});
```

`foreignId` 方法會建立一個等同於 `UNSIGNED BIGINT` 的資料欄，而 `constrained` 方法將使用慣例來決定所參照的資料表和資料欄。如果您的資料表名稱不符合 Laravel 的慣例，您可以手動將其提供給 `constrained` 方法。此外，也可以指定要賦予給所生成索引的名稱：

```php
Schema::table('posts', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained(
        table: 'users', indexName: 'posts_user_id'
    );
});
```

您也可以為此限制條件的「on delete」和「on update」屬性指定所需的動作：

```php
$table->foreignId('user_id')
    ->constrained()
    ->onUpdate('cascade')
    ->onDelete('cascade');
```

針對這些動作，也提供了另一種表達性語法：

<div class="overflow-auto">

| 方法                            | 描述                                       |
| ------------------------------- | ------------------------------------------ |
| `$table->cascadeOnUpdate();`    | 更新時應連帶更新。                         |
| `$table->restrictOnUpdate();`   | 更新時應受限制。                           |
| `$table->nullOnUpdate();`       | 更新時應將外部鍵值設為 null。              |
| `$table->noActionOnUpdate();`   | 更新時不執行任何動作。                     |
| `$table->cascadeOnDelete();`    | 刪除時應連帶刪除。                         |
| `$table->restrictOnDelete();`   | 刪除時應受限制。                           |
| `$table->nullOnDelete();`       | 刪除時應將外部鍵值設為 null。              |
| `$table->noActionOnDelete();`   | 若存在子記錄，則阻止刪除。                 |

</div>

任何額外的 [資料欄修飾符](#column-modifiers) 都必須在 `constrained` 方法之前呼叫：

```php
$table->foreignId('user_id')
    ->nullable()
    ->constrained();
```

<a name="dropping-foreign-keys"></a>
#### 刪除外部鍵

要刪除外部鍵，您可以使用 `dropForeign` 方法，將要刪除的外部鍵限制名稱作為引數傳入。外部鍵限制使用與索引相同的命名慣例。換句話說，外部鍵限制名稱是基於資料表名稱、限制中的資料欄名稱，後綴為「\_foreign」：

```php
$table->dropForeign('posts_user_id_foreign');
```

或者，您可以將包含持有外部鍵的資料欄名稱的陣列傳遞給 `dropForeign` 方法。該陣列將根據 Laravel 的限制命名慣例轉換為外部鍵限制名稱：

```php
$table->dropForeign(['user_id']);
```

<a name="toggling-foreign-key-constraints"></a>
#### 切換外部鍵限制

您可以使用以下方法，在遷移中啟用或停用外部鍵限制：

```php
Schema::enableForeignKeyConstraints();

Schema::disableForeignKeyConstraints();

Schema::withoutForeignKeyConstraints(function () {
    // Constraints disabled within this closure...
});
```

> [!WARNING]
> SQLite 預設會停用外部鍵限制。使用 SQLite 時，請確保在嘗試於遷移中建立外部鍵之前，在您的資料庫設定中 [啟用外部鍵支援](/docs/{{version}}/database#configuration)。

<a name="events"></a>
## 事件

為了方便，每個遷移操作都會分派一個 [事件](/docs/{{version}}/events)。以下所有事件都繼承了基礎的 `Illuminate\Database\Events\MigrationEvent` 類別：

<div class="overflow-auto">

| 類別                                            | 描述                                      |
| ------------------------------------------------ | ------------------------------------------------ |
| `Illuminate\Database\Events\MigrationsStarted`   | 一批遷移即將被執行。   |
| `Illuminate\Database\Events\MigrationsEnded`     | 一批遷移已完成執行。    |
| `Illuminate\Database\Events\MigrationStarted`    | 單個遷移即將被執行。      |
| `Illuminate\Database\Events\MigrationEnded`      | 單個遷移已完成執行。       |
| `Illuminate\Database\Events\NoPendingMigrations` | 遷移指令未發現待處理的遷移。 |
| `Illuminate\Database\Events\SchemaDumped`        | 資料庫 Schema 傾印已完成。            |
| `Illuminate\Database\Events\SchemaLoaded`        | 現有的資料庫 Schema 傾印已載入。     |

</div>