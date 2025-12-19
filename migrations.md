# 資料庫：Migrations

- [簡介](#introduction)
- [生成 Migrations](#generating-migrations)
    - [壓縮 Migrations](#squashing-migrations)
- [Migration 結構](#migration-structure)
- [執行 Migrations](#running-migrations)
    - [回滾 Migrations](#rolling-back-migrations)
- [資料表](#tables)
    - [建立資料表](#creating-tables)
    - [更新資料表](#updating-tables)
    - [重新命名 / 移除資料表](#renaming-and-dropping-tables)
- [欄位](#columns)
    - [建立欄位](#creating-columns)
    - [可用的欄位類型](#available-column-types)
    - [欄位修飾詞](#column-modifiers)
    - [修改欄位](#modifying-columns)
    - [重新命名欄位](#renaming-columns)
    - [移除欄位](#dropping-columns)
- [索引](#indexes)
    - [建立索引](#creating-indexes)
    - [重新命名索引](#renaming-indexes)
    - [移除索引](#dropping-indexes)
    - [外鍵約束](#foreign-key-constraints)
- [事件](#events)

<a name="introduction"></a>
## 簡介

Migrations 就像是資料庫的版本控制，讓您的團隊可以定義並共享應用程式的資料庫結構定義。如果您曾經必須在從版本控制拉取變更後，告訴同事手動在其本地資料庫結構中新增一個欄位，那麼您就遇到了資料庫 Migrations 所要解決的問題。

Laravel `Schema` [Facade](/docs/{{version}}/facades) 提供了一種與資料庫無關的支援，用於在 Laravel 支援的所有資料庫系統中建立和操作資料表。通常，Migrations 會使用此 Facade 來建立和修改資料庫資料表與欄位。


<a name="generating-migrations"></a>
## 生成 Migrations

您可以使用 `make:migration` [Artisan 指令](/docs/{{version}}/artisan)來生成資料庫 Migration。新的 Migration 將會被放置在您的 `database/migrations` 目錄中。每個 Migration 檔名都包含一個時間戳記，這讓 Laravel 可以確定 Migrations 的執行順序：

```shell
php artisan make:migration create_flights_table
```

Laravel 會根據 Migration 的名稱嘗試猜測資料表的名稱，以及該 Migration 是否要建立一個新資料表。如果 Laravel 能夠從 Migration 名稱中確定資料表名稱，則會在生成的 Migration 檔案中預填指定的資料表。否則，您只需手動在 Migration 檔案中指定資料表即可。

如果您想為生成的 Migration 指定自定義路徑，可以在執行 `make:migration` 指令時使用 `--path` 選項。給定的路徑應該相對於應用程式的根路徑。

> [!NOTE]
> Migration Stub 可以使用 [Stub 發布](/docs/{{version}}/artisan#stub-customization)進行自定義。


<a name="squashing-migrations"></a>
### 壓縮 Migrations

隨著您開發應用程式，隨著時間推移，您可能會累積越來越多的 Migrations。這可能會導致您的 `database/migrations` 目錄變得臃腫，甚至可能有數百個 Migrations。如果您願意，可以將 Migrations 「壓縮 (Squash)」成單個 SQL 檔案。要開始執行，請執行 `schema:dump` 指令：

```shell
php artisan schema:dump

# Dump the current database schema and prune all existing migrations...
php artisan schema:dump --prune
```

當您執行此指令時，Laravel 會將一個「Schema」檔案寫入應用程式的 `database/schema` 目錄中。Schema 檔案的名稱將對應於資料庫連線。現在，當您嘗試遷移資料庫且沒有執行過其他 Migrations 時，Laravel 將首先執行您正在使用的資料庫連線之 Schema 檔案中的 SQL 語句。在執行 Schema 檔案的 SQL 語句後，Laravel 將執行不屬於 Schema Dump 的任何剩餘 Migrations。

如果您的應用程式測試使用的資料庫連線與您在本地開發時通常使用的不同，則應確保已使用該資料庫連線導出 Schema 檔案，以便您的測試能夠建立資料庫。您可能希望在導出平常在本地開發時使用的資料庫連線後執行此操作：

```shell
php artisan schema:dump
php artisan schema:dump --database=testing --prune
```

您應該將資料庫 Schema 檔案提交到版本控制中，以便團隊中的其他新開發人員可以快速建立應用程式的初始資料庫結構。

> [!WARNING]
> Migration 壓縮功能僅適用於 MariaDB、MySQL、PostgreSQL 和 SQLite 資料庫，並利用了資料庫的命令列用戶端。


<a name="migration-structure"></a>
## Migration 結構

一個 Migration 類別包含兩個方法：`up` 和 `down`。`up` 方法用於向資料庫新增新的資料表、欄位或索引，而 `down` 方法則應該復原由 `up` 方法執行的操作。

在這兩個方法中，您都可以使用 Laravel Schema 構建器來直觀地建立和修改資料表。要瞭解 `Schema` 構建器上所有可用的方法，請[查看其文件](#creating-tables)。例如，以下 Migration 建立了一個 `flights` 資料表：

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
#### 設定 Migration 連線

如果您的 Migration 將與應用程式預設資料庫連線以外的資料庫連線進行互動，您應該設定 Migration 的 `$connection` 屬性：

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
#### 跳過 Migrations

有時候，Migration 可能是為了支援尚未啟用的功能，而您現在還不希望它執行。在這種情況下，您可以在 Migration 中定義一個 `shouldRun` 方法。如果 `shouldRun` 方法回傳 `false`，則該 Migration 將被跳過：

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
## 執行 Migrations

若要執行所有待處理的 Migration，請執行 `migrate` Artisan 指令：

```shell
php artisan migrate
```

如果您想查看哪些 Migration 已經執行、哪些仍在等待執行，您可以使用 `migrate:status` Artisan 指令：

```shell
php artisan migrate:status
```

如果您想查看 Migration 將執行的 SQL 語句而不想實際執行它們，您可以在 `migrate` 指令中提供 `--pretend` 旗標：

```shell
php artisan migrate --pretend
```


<a name="isolating-migration-execution"></a>
#### 隔離 Migration 執行

如果您在多台伺服器上部署應用程式，並將執行 Migration 作為部署流程的一部分，您可能不希望兩台伺服器同時嘗試遷移資料庫。為了避免這種情況，您可以在呼叫 `migrate` 指令時使用 `isolated` 選項。

當提供 `isolated` 選項時，Laravel 會在嘗試執行 Migration 之前，使用應用程式的快取驅動程式取得一個原子鎖 (Atomic Lock)。在持有該鎖期間，所有其他執行 `migrate` 指令的嘗試都不會執行；然而，該指令仍會以成功的結束狀態碼退出：

```shell
php artisan migrate --isolated
```

> [!WARNING]
> 若要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 作為應用程式預設的快取驅動程式。此外，所有伺服器都必須與同一個中央快取伺服器進行通訊。


<a name="forcing-migrations-to-run-in-production"></a>
#### 在正式環境中強制執行 Migration

某些 Migration 操作是具有破壞性的，這意味著它們可能會導致資料遺失。為了防止您在正式環境資料庫上執行這些指令，在執行指令之前系統會提示您進行確認。若要強制指令在沒有提示的情況下執行，請使用 `--force` 旗標：

```shell
php artisan migrate --force
```


<a name="rolling-back-migrations"></a>
### 回滾 Migrations

若要回滾最後一次的 Migration 操作，您可以使用 `rollback` Artisan 指令。此指令會回滾最後一「批 (Batch)」的 Migration，其中可能包含多個 Migration 檔案：

```shell
php artisan migrate:rollback
```

您可以透過向 `rollback` 指令提供 `step` 選項來回滾特定數量的 Migration。例如，以下指令將回滾最後五個 Migration：

```shell
php artisan migrate:rollback --step=5
```

您可以透過向 `rollback` 指令提供 `batch` 選項來回滾特定「批次」的 Migration，其中 `batch` 選項對應於應用程式 `migrations` 資料庫表中的批次值。例如，以下指令將回滾批次三中的所有 Migration：

```shell
php artisan migrate:rollback --batch=3
```

如果您想查看 Migration 將執行的 SQL 語句而不想實際執行它們，您可以在 `migrate:rollback` 指令中使用 `--pretend` 旗標：

```shell
php artisan migrate:rollback --pretend
```

`migrate:reset` 指令將回滾應用程式所有的 Migration：

```shell
php artisan migrate:reset
```


<a name="roll-back-migrate-using-a-single-command"></a>
#### 使用單一指令回滾並遷移

`migrate:refresh` 指令將回滾所有 Migration，然後執行 `migrate` 指令。此指令可有效地重新建立您的整個資料庫：

```shell
php artisan migrate:refresh

# Refresh the database and run all database seeds...
php artisan migrate:refresh --seed
```

您可以透過向 `refresh` 指令提供 `step` 選項來回滾並重新遷移特定數量的 Migration。例如，以下指令將回滾並重新遷移最後五個 Migration：

```shell
php artisan migrate:refresh --step=5
```


<a name="drop-all-tables-migrate"></a>
#### 移除所有資料表並遷移

`migrate:fresh` 指令將從資料庫中移除所有資料表，然後執行 `migrate` 指令：

```shell
php artisan migrate:fresh

php artisan migrate:fresh --seed
```

預設情況下，`migrate:fresh` 指令僅從預設資料庫連接中移除資料表。但是，您可以使用 `--database` 選項指定應遷移的資料庫連接。資料庫連接名稱應對應於應用程式 `database` [設定檔](/docs/{{version}}/configuration)中定義的連接：

```shell
php artisan migrate:fresh --database=admin
```

> [!WARNING]
> `migrate:fresh` 指令將移除所有資料庫表，無論其前綴為何。在與其他應用程式共享的資料庫上進行開發時，應謹慎使用此指令。

<a name="tables"></a>
## 資料表


<a name="creating-tables"></a>
### 建立資料表

若要建立新的資料表，請使用 `Schema` Facade 的 `create` 方法。`create` 方法接受兩個參數：第一個是資料表的名稱，第二個則是接收 `Blueprint` 物件的閉包 (Closure)，可用於定義新資料表：

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

建立資料表時，您可以使用 Schema 建構器的任何 [欄位方法](#creating-columns) 來定義資料表的欄位。


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

如果您想在非應用程式預設連線的資料庫連線執行 Schema 操作，請使用 `connection` 方法：

```php
Schema::connection('sqlite')->create('users', function (Blueprint $table) {
    $table->id();
});
```

此外，還可以使用其他一些屬性和方法來定義資料表建立的其他面向。在使用 MariaDB 或 MySQL 時，可以使用 `engine` 屬性來指定資料表的儲存引擎：

```php
Schema::create('users', function (Blueprint $table) {
    $table->engine('InnoDB');

    // ...
});
```

在使用 MariaDB 或 MySQL 時，可以使用 `charset` 與 `collation` 屬性來指定所建立資料表的字元集與定序 (Collation)：

```php
Schema::create('users', function (Blueprint $table) {
    $table->charset('utf8mb4');
    $table->collation('utf8mb4_unicode_ci');

    // ...
});
```

`temporary` 方法可用於表示該資料表應該是「暫時性」的。暫時性資料表僅對當前連線的資料庫工作階段 (Session) 可見，並在連線關閉時自動刪除：

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->temporary();

    // ...
});
```

如果您想為資料表加入「註解 (Comment)」，可以在資料表執行個體上呼叫 `comment` 方法。資料表註解目前僅由 MariaDB、MySQL 與 PostgreSQL 支援：

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->comment('Business calculations');

    // ...
});
```


<a name="updating-tables"></a>
### 更新資料表

`Schema` Facade 的 `table` 方法可用於更新現有的資料表。與 `create` 方法類似，`table` 方法接受兩個參數：資料表名稱，以及一個接收 `Blueprint` 執行個體的閉包，您可以使用該執行個體向資料表新增欄位或索引：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->integer('votes');
});
```


<a name="renaming-and-dropping-tables"></a>
### 重新命名 / 移除資料表

若要重新命名現有的資料表，請使用 `rename` 方法：

```php
use Illuminate\Support\Facades\Schema;

Schema::rename($from, $to);
```

若要移除現有的資料表，您可以使用 `drop` 或 `dropIfExists` 方法：

```php
Schema::drop('users');

Schema::dropIfExists('users');
```


<a name="renaming-tables-with-foreign-keys"></a>
#### 重新命名帶有外鍵的資料表

在重新命名資料表之前，您應該確認資料表上的任何外鍵約束在 Migration 檔案中都有明確的名稱，而不是讓 Laravel 指定基於慣例的名稱。否則，外鍵約束名稱將會指向舊的資料表名稱。

<a name="columns"></a>
## 欄位


<a name="creating-columns"></a>
### 建立欄位

`Schema` Facade 上的 `table` 方法可用於更新現有的資料表。就像 `create` 方法一樣，`table` 方法接受兩個參數：資料表的名稱，以及一個接收 `Illuminate\Database\Schema\Blueprint` 實例的閉包 (Closure)，您可以使用它來向資料表新增欄位：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->integer('votes');
});
```

<a name="available-column-types"></a>
### 可用的欄位類型

Schema 構建器 Blueprint 提供了多種方法，對應到您可以新增到資料表中的不同欄位類型。下表列出了所有可用的方法：

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

`bigIncrements` 方法建立一個自動遞增且為 `UNSIGNED BIGINT` (主鍵) 的等效欄位：

```php
$table->bigIncrements('id');
```


<a name="column-method-bigInteger"></a>
#### `bigInteger()` {.collection-method}

`bigInteger` 方法建立一個 `BIGINT` 的等效欄位：

```php
$table->bigInteger('votes');
```


<a name="column-method-binary"></a>
#### `binary()` {.collection-method}

`binary` 方法建立一個 `BLOB` 的等效欄位：

```php
$table->binary('photo');
```

使用 MySQL、MariaDB 或 SQL Server 時，您可以傳遞 `length` 與 `fixed` 參數來建立 `VARBINARY` 或 `BINARY` 的等效欄位：

```php
$table->binary('data', length: 16); // VARBINARY(16)

$table->binary('data', length: 16, fixed: true); // BINARY(16)
```


<a name="column-method-boolean"></a>
#### `boolean()` {.collection-method}

`boolean` 方法建立一個 `BOOLEAN` 的等效欄位：

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

`dateTimeTz` 方法建立一個 `DATETIME` (含時區) 的等效欄位，並可選擇性指定小數秒數精度：

```php
$table->dateTimeTz('created_at', precision: 0);
```


<a name="column-method-dateTime"></a>
#### `dateTime()` {.collection-method}

`dateTime` 方法建立一個 `DATETIME` 的等效欄位，並可選擇性指定小數秒數精度：

```php
$table->dateTime('created_at', precision: 0);
```


<a name="column-method-date"></a>
#### `date()` {.collection-method}

`date` 方法建立一個 `DATE` 的等效欄位：

```php
$table->date('created_at');
```


<a name="column-method-decimal"></a>
#### `decimal()` {.collection-method}

`decimal` 方法建立一個具備指定精度 (總位數) 與標度 (小數位數) 的 `DECIMAL` 等效欄位：

```php
$table->decimal('amount', total: 8, places: 2);
```


<a name="column-method-double"></a>
#### `double()` {.collection-method}

`double` 方法建立一個 `DOUBLE` 的等效欄位：

```php
$table->double('amount');
```


<a name="column-method-enum"></a>
#### `enum()` {.collection-method}

`enum` 方法建立一個具備指定有效值列表的 `ENUM` 等效欄位：

```php
$table->enum('difficulty', ['easy', 'hard']);
```

當然，您可以使用 `Enum::cases()` 方法，而不是手動定義一個包含允許值的陣列：

```php
use App\Enums\Difficulty;

$table->enum('difficulty', Difficulty::cases());
```


<a name="column-method-float"></a>
#### `float()` {.collection-method}

`float` 方法建立一個指定精度的 `FLOAT` 等效欄位：

```php
$table->float('amount', precision: 53);
```


<a name="column-method-foreignId"></a>
#### `foreignId()` {.collection-method}

`foreignId` 方法建立一個 `UNSIGNED BIGINT` 的等效欄位：

```php
$table->foreignId('user_id');
```


<a name="column-method-foreignIdFor"></a>
#### `foreignIdFor()` {.collection-method}

`foreignIdFor` 方法為給定的模型類別新增一個 `{column}_id` 等效欄位。欄位類型將根據模型的主鍵類型而定，可能是 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`：

```php
$table->foreignIdFor(User::class);
```


<a name="column-method-foreignUlid"></a>
#### `foreignUlid()` {.collection-method}

`foreignUlid` 方法建立一個 `ULID` 的等效欄位：

```php
$table->foreignUlid('user_id');
```


<a name="column-method-foreignUuid"></a>
#### `foreignUuid()` {.collection-method}

`foreignUuid` 方法建立一個 `UUID` 的等效欄位：

```php
$table->foreignUuid('user_id');
```


<a name="column-method-geography"></a>
#### `geography()` {.collection-method}

`geography` 方法建立一個具備指定空間類型與 SRID (空間參考系統識別碼) 的 `GEOGRAPHY` 等效欄位：

```php
$table->geography('coordinates', subtype: 'point', srid: 4326);
```

> [!NOTE]
> 對空間類型的支援取決於您的資料庫驅動程式。請參閱您的資料庫文件。如果您的應用程式使用的是 PostgreSQL 資料庫，則必須在選用 `geography` 方法之前安裝 [PostGIS](https://postgis.net) 擴充功能。


<a name="column-method-geometry"></a>
#### `geometry()` {.collection-method}

`geometry` 方法建立一個具備指定空間類型與 SRID (空間參考系統識別碼) 的 `GEOMETRY` 等效欄位：

```php
$table->geometry('positions', subtype: 'point', srid: 0);
```

> [!NOTE]
> 對空間類型的支援取決於您的資料庫驅動程式。請參閱您的資料庫文件。如果您的應用程式使用的是 PostgreSQL 資料庫，則必須在選用 `geometry` 方法之前安裝 [PostGIS](https://postgis.net) 擴充功能。


<a name="column-method-id"></a>
#### `id()` {.collection-method}

`id` 方法是 `bigIncrements` 方法的別名。預設情況下，該方法將建立一個 `id` 欄位；但是，如果您想為該欄位指定不同的名稱，可以傳遞一個欄位名稱：

```php
$table->id();
```


<a name="column-method-increments"></a>
#### `increments()` {.collection-method}

`increments` 方法建立一個自動遞增且為 `UNSIGNED INTEGER` (主鍵) 的等效欄位：

```php
$table->increments('id');
```


<a name="column-method-integer"></a>
#### `integer()` {.collection-method}

`integer` 方法建立一個 `INTEGER` 的等效欄位：

```php
$table->integer('votes');
```


<a name="column-method-ipAddress"></a>
#### `ipAddress()` {.collection-method}

`ipAddress` 方法建立一個 `VARCHAR` 的等效欄位：

```php
$table->ipAddress('visitor');
```

當使用 PostgreSQL 時，將建立一個 `INET` 欄位。


<a name="column-method-json"></a>
#### `json()` {.collection-method}

`json` 方法建立一個 `JSON` 的等效欄位：

```php
$table->json('options');
```

當使用 SQLite 時，將建立一個 `TEXT` 欄位。


<a name="column-method-jsonb"></a>
#### `jsonb()` {.collection-method}

`jsonb` 方法建立一個 `JSONB` 的等效欄位：

```php
$table->jsonb('options');
```

當使用 SQLite 時，將建立一個 `TEXT` 欄位。


<a name="column-method-longText"></a>
#### `longText()` {.collection-method}

`longText` 方法建立一個 `LONGTEXT` 的等效欄位：

```php
$table->longText('description');
```

使用 MySQL 或 MariaDB 時，您可以對該欄位套用 `binary` 字元集，以建立一個 `LONGBLOB` 等效欄位：

```php
$table->longText('data')->charset('binary'); // LONGBLOB
```


<a name="column-method-macAddress"></a>
#### `macAddress()` {.collection-method}

`macAddress` 方法建立一個旨在保存 MAC 位址的欄位。某些資料庫系統（例如 PostgreSQL）有專門用於此類資料的欄位類型。其他資料庫系統則會使用字串等效欄位：

```php
$table->macAddress('device');
```


<a name="column-method-mediumIncrements"></a>
#### `mediumIncrements()` {.collection-method}

`mediumIncrements` 方法建立一個自動遞增且為 `UNSIGNED MEDIUMINT` (主鍵) 的等效欄位：

```php
$table->mediumIncrements('id');
```


<a name="column-method-mediumInteger"></a>
#### `mediumInteger()` {.collection-method}

`mediumInteger` 方法建立一個 `MEDIUMINT` 的等效欄位：

```php
$table->mediumInteger('votes');
```


<a name="column-method-mediumText"></a>
#### `mediumText()` {.collection-method}

`mediumText` 方法建立一個 `MEDIUMTEXT` 的等效欄位：

```php
$table->mediumText('description');
```

使用 MySQL 或 MariaDB 時，您可以對該欄位套用 `binary` 字元集，以建立一個 `MEDIUMBLOB` 等效欄位：

```php
$table->mediumText('data')->charset('binary'); // MEDIUMBLOB
```


<a name="column-method-morphs"></a>
#### `morphs()` {.collection-method}

`morphs` 方法是一個便捷方法，它會新增一個 `{column}_id` 等效欄位和一個 `{column}_type` 的 `VARCHAR` 等效欄位。`{column}_id` 的欄位類型將取決於模型的主鍵類型，可能是 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`。

此方法旨在定義多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)所需的欄位。在以下範例中，將建立 `taggable_id` 和 `taggable_type` 欄位：

```php
$table->morphs('taggable');
```


<a name="column-method-nullableMorphs"></a>
#### `nullableMorphs()` {.collection-method}

該方法與 [morphs](#column-method-morphs) 方法類似；但是，建立的欄位將會是「可為空 (nullable)」的：

```php
$table->nullableMorphs('taggable');
```


<a name="column-method-nullableUlidMorphs"></a>
#### `nullableUlidMorphs()` {.collection-method}

該方法與 [ulidMorphs](#column-method-ulidMorphs) 方法類似；但是，建立的欄位將會是「可為空 (nullable)」的：

```php
$table->nullableUlidMorphs('taggable');
```


<a name="column-method-nullableUuidMorphs"></a>
#### `nullableUuidMorphs()` {.collection-method}

該方法與 [uuidMorphs](#column-method-uuidMorphs) 方法類似；但是，建立的欄位將會是「可為空 (nullable)」的：

```php
$table->nullableUuidMorphs('taggable');
```


<a name="column-method-rememberToken"></a>
#### `rememberToken()` {.collection-method}

`rememberToken` 方法建立一個可為空且為 `VARCHAR(100)` 的等效欄位，該欄位旨在儲存目前的「記住我」[身分驗證權杖](/docs/{{version}}/authentication#remembering-users)：

```php
$table->rememberToken();
```


<a name="column-method-set"></a>
#### `set()` {.collection-method}

`set` 方法建立一個具備指定有效值列表的 `SET` 等效欄位：

```php
$table->set('flavors', ['strawberry', 'vanilla']);
```


<a name="column-method-smallIncrements"></a>
#### `smallIncrements()` {.collection-method}

`smallIncrements` 方法建立一個自動遞增且為 `UNSIGNED SMALLINT` (主鍵) 的等效欄位：

```php
$table->smallIncrements('id');
```


<a name="column-method-smallInteger"></a>
#### `smallInteger()` {.collection-method}

`smallInteger` 方法建立一個 `SMALLINT` 的等效欄位：

```php
$table->smallInteger('votes');
```


<a name="column-method-softDeletesTz"></a>
#### `softDeletesTz()` {.collection-method}

`softDeletesTz` 方法新增一個可為空且為 `TIMESTAMP` (含時區) 的 `deleted_at` 等效欄位，並可選擇性指定小數秒數精度。此欄位旨在儲存 Eloquent「軟刪除」功能所需的 `deleted_at` 時間戳記：

```php
$table->softDeletesTz('deleted_at', precision: 0);
```


<a name="column-method-softDeletes"></a>
#### `softDeletes()` {.collection-method}

`softDeletes` 方法新增一個可為空且為 `TIMESTAMP` 的 `deleted_at` 等效欄位，並可選擇性指定小數秒數精度。此欄位旨在儲存 Eloquent「軟刪除」功能所需的 `deleted_at` 時間戳記：

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

`text` 方法建立一個 `TEXT` 的等效欄位：

```php
$table->text('description');
```

使用 MySQL 或 MariaDB 時，您可以對該欄位套用 `binary` 字元集，以建立一個 `BLOB` 等效欄位：

```php
$table->text('data')->charset('binary'); // BLOB
```


<a name="column-method-timeTz"></a>
#### `timeTz()` {.collection-method}

`timeTz` 方法建立一個 `TIME` (含時區) 的等效欄位，並可選擇性指定小數秒數精度：

```php
$table->timeTz('sunrise', precision: 0);
```


<a name="column-method-time"></a>
#### `time()` {.collection-method}

`time` 方法建立一個 `TIME` 的等效欄位，並可選擇性指定小數秒數精度：

```php
$table->time('sunrise', precision: 0);
```


<a name="column-method-timestampTz"></a>
#### `timestampTz()` {.collection-method}

`timestampTz` 方法建立一個 `TIMESTAMP` (含時區) 的等效欄位，並可選擇性指定小數秒數精度：

```php
$table->timestampTz('added_at', precision: 0);
```


<a name="column-method-timestamp"></a>
#### `timestamp()` {.collection-method}

`timestamp` 方法建立一個 `TIMESTAMP` 的等效欄位，並可選擇性指定小數秒數精度：

```php
$table->timestamp('added_at', precision: 0);
```


<a name="column-method-timestampsTz"></a>
#### `timestampsTz()` {.collection-method}

`timestampsTz` 方法建立 `created_at` 與 `updated_at` 的 `TIMESTAMP` (含時區) 等效欄位，並可選擇性指定小數秒數精度：

```php
$table->timestampsTz(precision: 0);
```


<a name="column-method-timestamps"></a>
#### `timestamps()` {.collection-method}

`timestamps` 方法建立 `created_at` 與 `updated_at` 的 `TIMESTAMP` 等效欄位，並可選擇性指定小數秒數精度：

```php
$table->timestamps(precision: 0);
```


<a name="column-method-tinyIncrements"></a>
#### `tinyIncrements()` {.collection-method}

`tinyIncrements` 方法建立一個自動遞增且為 `UNSIGNED TINYINT` (主鍵) 的等效欄位：

```php
$table->tinyIncrements('id');
```


<a name="column-method-tinyInteger"></a>
#### `tinyInteger()` {.collection-method}

`tinyInteger` 方法建立一個 `TINYINT` 的等效欄位：

```php
$table->tinyInteger('votes');
```


<a name="column-method-tinyText"></a>
#### `tinyText()` {.collection-method}

`tinyText` 方法建立一個 `TINYTEXT` 的等效欄位：

```php
$table->tinyText('notes');
```

使用 MySQL 或 MariaDB 時，您可以對該欄位套用 `binary` 字元集，以建立一個 `TINYBLOB` 等效欄位：

```php
$table->tinyText('data')->charset('binary'); // TINYBLOB
```


<a name="column-method-unsignedBigInteger"></a>
#### `unsignedBigInteger()` {.collection-method}

`unsignedBigInteger` 方法建立一個 `UNSIGNED BIGINT` 的等效欄位：

```php
$table->unsignedBigInteger('votes');
```


<a name="column-method-unsignedInteger"></a>
#### `unsignedInteger()` {.collection-method}

`unsignedInteger` 方法建立一個 `UNSIGNED INTEGER` 的等效欄位：

```php
$table->unsignedInteger('votes');
```


<a name="column-method-unsignedMediumInteger"></a>
#### `unsignedMediumInteger()` {.collection-method}

`unsignedMediumInteger` 方法建立一個 `UNSIGNED MEDIUMINT` 的等效欄位：

```php
$table->unsignedMediumInteger('votes');
```


<a name="column-method-unsignedSmallInteger"></a>
#### `unsignedSmallInteger()` {.collection-method}

`unsignedSmallInteger` 方法建立一個 `UNSIGNED SMALLINT` 的等效欄位：

```php
$table->unsignedSmallInteger('votes');
```


<a name="column-method-unsignedTinyInteger"></a>
#### `unsignedTinyInteger()` {.collection-method}

`unsignedTinyInteger` 方法建立一個 `UNSIGNED TINYINT` 的等效欄位：

```php
$table->unsignedTinyInteger('votes');
```


<a name="column-method-ulidMorphs"></a>
#### `ulidMorphs()` {.collection-method}

`ulidMorphs` 方法是一個便捷方法，它會新增一個 `{column}_id` 的 `CHAR(26)` 等效欄位和一個 `{column}_type` 的 `VARCHAR` 等效欄位。

此方法旨在定義使用 ULID 識別碼的多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)所需的欄位。在以下範例中，將建立 `taggable_id` 和 `taggable_type` 欄位：

```php
$table->ulidMorphs('taggable');
```


<a name="column-method-uuidMorphs"></a>
#### `uuidMorphs()` {.collection-method}

`uuidMorphs` 方法是一個便捷方法，它會新增一個 `{column}_id` 的 `CHAR(36)` 等效欄位和一個 `{column}_type` 的 `VARCHAR` 等效欄位。

此方法旨在定義使用 UUID 識別碼的多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)所需的欄位。在以下範例中，將建立 `taggable_id` 和 `taggable_type` 欄位：

```php
$table->uuidMorphs('taggable');
```


<a name="column-method-ulid"></a>
#### `ulid()` {.collection-method}

`ulid` 方法建立一個 `ULID` 的等效欄位：

```php
$table->ulid('id');
```


<a name="column-method-uuid"></a>
#### `uuid()` {.collection-method}

`uuid` 方法建立一個 `UUID` 的等效欄位：

```php
$table->uuid('id');
```


<a name="column-method-vector"></a>
#### `vector()` {.collection-method}

`vector` 方法建立一個 `vector` 的等效欄位：

```php
$table->vector('embedding', dimensions: 100);
```


<a name="column-method-year"></a>
#### `year()` {.collection-method}

`year` 方法建立一個 `YEAR` 的等效欄位：

```php
$table->year('birth_year');
```

<a name="column-modifiers"></a>
### 欄位修飾詞

除了上方列出的欄位類型外，在將欄位加入資料庫資料表時，還有多種欄位「修飾詞」可供使用。例如，若要讓欄位「可為空 (nullable)」，您可以使用 `nullable` 方法：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->string('email')->nullable();
});
```

下表包含所有可用的欄位修飾詞。此列表不包含 [索引修飾詞](#creating-indexes)：

<div class="overflow-auto">

| 修飾詞 | 說明 |
| ----------------------------------- | ---------------------------------------------------------------------------------------------- |
| `->after('column')` | 將欄位放置於另一個欄位「之後」(MariaDB / MySQL)。 |
| `->autoIncrement()` | 將 `INTEGER` 欄位設定為自動遞增 (主鍵)。 |
| `->charset('utf8mb4')` | 為欄位指定字元集 (MariaDB / MySQL)。 |
| `->collation('utf8mb4_unicode_ci')` | 為欄位指定排序規則。 |
| `->comment('my comment')` | 為欄位加上註解 (MariaDB / MySQL / PostgreSQL)。 |
| `->default($value)` | 為欄位指定「預設值」。 |
| `->first()` | 將欄位放置在資料表的「首位」(MariaDB / MySQL)。 |
| `->from($integer)` | 設定自動遞增欄位的起始值 (MariaDB / MySQL / PostgreSQL)。 |
| `->invisible()` | 對 `SELECT *` 查詢而言，使欄位變為「隱形」(MariaDB / MySQL)。 |
| `->nullable($value = true)` | 允許將 `NULL` 值插入欄位。 |
| `->storedAs($expression)` | 建立一個儲存型產生的欄位 (MariaDB / MySQL / PostgreSQL / SQLite)。 |
| `->unsigned()` | 將 `INTEGER` 欄位設定為 `UNSIGNED` (MariaDB / MySQL)。 |
| `->useCurrent()` | 設定 `TIMESTAMP` 欄位預設使用 `CURRENT_TIMESTAMP`。 |
| `->useCurrentOnUpdate()` | 設定 `TIMESTAMP` 欄位在紀錄更新時使用 `CURRENT_TIMESTAMP` (MariaDB / MySQL)。 |
| `->virtualAs($expression)` | 建立一個虛擬型產生的欄位 (MariaDB / MySQL / SQLite)。 |
| `->generatedAs($expression)` | 使用指定的序列選項建立一個識別欄位 (Identity column) (PostgreSQL)。 |
| `->always()` | 為識別欄位定義序列值優於輸入值的權限 (PostgreSQL)。 |

</div>


<a name="default-expressions"></a>
#### 預設值運算式

`default` 修飾詞接受一個值或一個 `Illuminate\Database\Query\Expression` 實例。使用 `Expression` 實例可以防止 Laravel 將該值包在引號中，並允許您使用資料庫特定的函數。這在您需要為 JSON 欄位分配預設值時特別有用：

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
> 對預設值運算式的支援取決於您的資料庫驅動程式、資料庫版本以及欄位類型。請參考您所使用的資料庫文件。


<a name="column-order"></a>
#### 欄位順序

使用 MariaDB 或 MySQL 資料庫時，可以使用 `after` 方法將欄位加入 Schema 中現有欄位的後方：

```php
$table->after('password', function (Blueprint $table) {
    $table->string('address_line1');
    $table->string('address_line2');
    $table->string('city');
});
```


<a name="modifying-columns"></a>
### 修改欄位

`change` 方法允許您修改現有欄位的類型與屬性。例如，您可能希望增加 `string` 欄位的大小。為了展示 `change` 方法，讓我們將 `name` 欄位的大小從 25 增加到 50。要達成此目的，我們只需定義欄位的新狀態，然後呼叫 `change` 方法：

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('name', 50)->change();
});
```

修改欄位時，您必須明確地包含所有您想要保留在欄位定義中的修飾詞 —— 任何缺失的屬性都將被移除。例如，為了保留 `unsigned`、`default` 與 `comment` 屬性，您必須在修改欄位時明確呼叫每個修飾詞：

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')->unsigned()->default(1)->comment('my comment')->change();
});
```

`change` 方法不會更改欄位的索引。因此，在修改欄位時，您可以使用索引修飾詞來明確地新增或移除索引：

```php
// Add an index...
$table->bigIncrements('id')->primary()->change();

// Drop an index...
$table->char('postal_code', 10)->unique(false)->change();
```


<a name="renaming-columns"></a>
### 重新命名欄位

若要重新命名欄位，您可以使用 Schema 建構器提供的 `renameColumn` 方法：

```php
Schema::table('users', function (Blueprint $table) {
    $table->renameColumn('from', 'to');
});
```


<a name="dropping-columns"></a>
### 移除欄位

若要移除欄位，您可以使用 Schema 建構器上的 `dropColumn` 方法：

```php
Schema::table('users', function (Blueprint $table) {
    $table->dropColumn('votes');
});
```

您可以藉由傳遞一個欄位名稱陣列給 `dropColumn` 方法，從資料表中移除多個欄位：

```php
Schema::table('users', function (Blueprint $table) {
    $table->dropColumn(['votes', 'avatar', 'location']);
});
```


<a name="available-command-aliases"></a>
#### 可用的指令別名

Laravel 提供了幾個方便的方法，用於移除常見類型的欄位。下表說明了其中的每一個方法：

<div class="overflow-auto">

| 指令 | 說明 |
| ----------------------------------- | ----------------------------------------------------- |
| `$table->dropMorphs('morphable');` | 移除 `morphable_id` 與 `morphable_type` 欄位。 |
| `$table->dropRememberToken();` | 移除 `remember_token` 欄位。 |
| `$table->dropSoftDeletes();` | 移除 `deleted_at` 欄位。 |
| `$table->dropSoftDeletesTz();` | `dropSoftDeletes()` 方法的別名。 |
| `$table->dropTimestamps();` | 移除 `created_at` 與 `updated_at` 欄位。 |
| `$table->dropTimestampsTz();` | `dropTimestamps()` 方法的別名。 |

</div>

<a name="indexes"></a>
## 索引


<a name="creating-indexes"></a>
### 建立索引

Laravel 的結構產生器支援多種索引類型。以下範例建立了一個新的 `email` 欄位，並指定其值必須是唯一的。若要建立索引，我們可以將 `unique` 方法鏈結到欄位定義上：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->string('email')->unique();
});
```

或者，您可以在定義欄位後再建立索引。若要這麼做，您應該在結構產生器藍圖上呼叫 `unique` 方法。此方法接受應接收唯一索引的欄位名稱：

```php
$table->unique('email');
```

您甚至可以向索引方法傳遞一個欄位陣列，以建立複合（或組合）索引：

```php
$table->index(['account_id', 'created_at']);
```

建立索引時，Laravel 會根據資料表、欄位名稱和索引類型自動產生索引名稱，但您可以向該方法傳遞第二個參數，以自行指定索引名稱：

```php
$table->unique('email', 'unique_email');
```


<a name="available-index-types"></a>
#### 可用的索引類型

Laravel 的結構產生器藍圖類別提供了建立 Laravel 支援的每種索引類型的方法。每個索引方法都接受一個選用的第二個參數來指定索引名稱。如果省略，名稱將根據用於索引的資料表和欄位名稱以及索引類型衍生而來。下表說明了每個可用的索引方法：

<div class="overflow-auto">

| 指令 | 描述 |
| ------------------------------------------------ | -------------------------------------------------------------- |
| `$table->primary('id');`                         | 新增一個主鍵。 |
| `$table->primary(['id', 'parent_id']);`          | 新增複合鍵。 |
| `$table->unique('email');`                       | 新增一個唯一索引。 |
| `$table->index('state');`                        | 新增一個索引。 |
| `$table->fullText('body');`                      | 新增一個全文索引 (MariaDB / MySQL / PostgreSQL)。 |
| `$table->fullText('body')->language('english');` | 新增一個指定語言的全文索引 (PostgreSQL)。 |
| `$table->spatialIndex('location');`              | 新增一個空間索引 (除 SQLite 外)。 |

</div>


<a name="renaming-indexes"></a>
### 重新命名索引

若要重新命名索引，您可以使用結構產生器藍圖提供的 `renameIndex` 方法。此方法接受目前的索引名稱作為其第一個參數，並將所需的名稱作為其第二個參數：

```php
$table->renameIndex('from', 'to')
```


<a name="dropping-indexes"></a>
### 移除索引

若要移除索引，您必須指定索引的名稱。預設情況下，Laravel 會根據資料表名稱、索引欄位的名稱和索引類型自動分配索引名稱。以下是一些範例：

<div class="overflow-auto">

| 指令 | 描述 |
| -------------------------------------------------------- | ----------------------------------------------------------- |
| `$table->dropPrimary('users_id_primary');`               | 從 "users" 資料表中移除主鍵。 |
| `$table->dropUnique('users_email_unique');`              | 從 "users" 資料表中移除唯一索引。 |
| `$table->dropIndex('geo_state_index');`                  | 從 "geo" 資料表中移除基本索引。 |
| `$table->dropFullText('posts_body_fulltext');`           | 從 "posts" 資料表中移除全文索引。 |
| `$table->dropSpatialIndex('geo_location_spatialindex');` | 從 "geo" 資料表中移除空間索引 (除 SQLite 外)。 |

</div>

如果您將欄位陣列傳遞給移除索引的方法，則會根據資料表名稱、欄位和索引類型產生慣例的索引名稱：

```php
Schema::table('geo', function (Blueprint $table) {
    $table->dropIndex(['state']); // Drops index 'geo_state_index'
});
```

<a name="foreign-key-constraints"></a>
### 外鍵約束

Laravel 也提供建立外鍵約束的支持，用於在資料庫層級強制執行參照完整性。例如，讓我們在 `posts` 資料表上定義一個 `user_id` 欄位，它參照 `users` 資料表的 `id` 欄位：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('posts', function (Blueprint $table) {
    $table->unsignedBigInteger('user_id');

    $table->foreign('user_id')->references('id')->on('users');
});
```

由於此語法相當冗長，Laravel 提供了額外的、更簡潔的方法，利用慣例來提供更好的開發體驗。當使用 `foreignId` 方法建立欄位時，上述範例可以改寫如下：

```php
Schema::table('posts', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained();
});
```

`foreignId` 方法會建立一個等同於 `UNSIGNED BIGINT` 的欄位，而 `constrained` 方法將使用慣例來確定所參照的資料表和欄位。如果您的資料表名稱不符合 Laravel 的慣例，您可以手動將其提供給 `constrained` 方法。此外，也可以指定應分配給生成的索引的名稱：

```php
Schema::table('posts', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained(
        table: 'users', indexName: 'posts_user_id'
    );
});
```

您還可以為約束的 "on delete" 和 "on update" 屬性指定所需的動作：

```php
$table->foreignId('user_id')
    ->constrained()
    ->onUpdate('cascade')
    ->onDelete('cascade');
```

這些動作也提供了另一種具表達力的語法：

<div class="overflow-auto">

| 方法 | 描述 |
| ----------------------------- | ------------------------------------------------- |
| `$table->cascadeOnUpdate();` | 更新時應連動 (Cascade)。 |
| `$table->restrictOnUpdate();` | 更新時應受限制。 |
| `$table->nullOnUpdate();` | 更新時應將外鍵值設為 null。 |
| `$table->noActionOnUpdate();` | 更新時不採取動作。 |
| `$table->cascadeOnDelete();` | 刪除時應連動 (Cascade)。 |
| `$table->restrictOnDelete();` | 刪除時應受限制。 |
| `$table->nullOnDelete();` | 刪除時應將外鍵值設為 null。 |
| `$table->noActionOnDelete();` | 如果子紀錄存在，則防止刪除。 |

</div>

任何額外的[欄位修飾詞](#column-modifiers)都必須在 `constrained` 方法之前呼叫：

```php
$table->foreignId('user_id')
    ->nullable()
    ->constrained();
```


<a name="dropping-foreign-keys"></a>
#### 移除外鍵

要移除外鍵，您可以使用 `dropForeign` 方法，並將要刪除的外鍵約束名稱作為參數傳遞。外鍵約束使用與索引相同的命名慣例。換句話說，外鍵約束名稱是基於資料表名稱和約束中的欄位名稱，後接 "\_foreign" 字尾：

```php
$table->dropForeign('posts_user_id_foreign');
```

或者，您可以將包含持有外鍵的欄位名稱的陣列傳遞給 `dropForeign` 方法。該陣列將根據 Laravel 的約束命名慣例轉換為外鍵約束名稱：

```php
$table->dropForeign(['user_id']);
```


<a name="toggling-foreign-key-constraints"></a>
#### 切換外鍵約束

您可以在 migration 中使用以下方法啟用或停用外鍵約束：

```php
Schema::enableForeignKeyConstraints();

Schema::disableForeignKeyConstraints();

Schema::withoutForeignKeyConstraints(function () {
    // Constraints disabled within this closure...
});
```

> [!WARNING]
> SQLite 預設會停用外鍵約束。使用 SQLite 時，請確保在嘗試於 migration 中建立外鍵之前，已在資料庫設定中[啟用外鍵支援](/docs/{{version}}/database#configuration)。

<a name="events"></a>
## 事件

為了方便起見，每個 Migration 操作都會分派一個 [事件](/docs/{{version}}/events)。以下所有事件都繼承了基礎的 `Illuminate\Database\Events\MigrationEvent` 類別：

<div class="overflow-auto">

| Class                                            | Description                                      |
| ------------------------------------------------ | ------------------------------------------------ |
| `Illuminate\Database\Events\MigrationsStarted`   | 一整批 Migration 即將執行。                      |
| `Illuminate\Database\Events\MigrationsEnded`     | 一整批 Migration 已執行完畢。                    |
| `Illuminate\Database\Events\MigrationStarted`    | 單個 Migration 即將執行。                        |
| `Illuminate\Database\Events\MigrationEnded`      | 單個 Migration 已執行完畢。                      |
| `Illuminate\Database\Events\NoPendingMigrations` | Migration 命令找不到待處理的 Migration。         |
| `Illuminate\Database\Events\SchemaDumped`        | 資料庫結構傾印 (Dump) 已完成。                   |
| `Illuminate\Database\Events\SchemaLoaded`        | 已載入現有的資料庫結構傾印 (Dump)。              |

</div>