# 資料庫：資料遷移

- [簡介](#introduction)
- [產生資料遷移](#generating-migrations)
    - [壓縮資料遷移](#squashing-migrations)
- [資料遷移結構](#migration-structure)
- [執行資料遷移](#running-migrations)
    - [復原資料遷移](#rolling-back-migrations)
- [資料表](#tables)
    - [建立資料表](#creating-tables)
    - [更新資料表](#updating-tables)
    - [重新命名 / 刪除資料表](#renaming-and-dropping-tables)
- [欄位](#columns)
    - [建立欄位](#creating-columns)
    - [可用的欄位型別](#available-column-types)
    - [欄位修改器](#column-modifiers)
    - [修改欄位](#modifying-columns)
    - [重新命名欄位](#renaming-columns)
    - [刪除欄位](#dropping-columns)
- [索引](#indexes)
    - [建立索引](#creating-indexes)
    - [重新命名索引](#renaming-indexes)
    - [刪除索引](#dropping-indexes)
    - [外部鍵限制](#foreign-key-constraints)
- [事件](#events)

<a name="introduction"></a>
## 簡介

資料遷移 (Migrations) 就像是資料庫的版本控制，讓您的團隊能夠定義並共享應用程式的資料庫綱要定義。如果您曾遇到這樣一個問題：在從原始碼控制拉取變更後，您必須告訴隊友手動在他們的本地資料庫綱要中新增一個欄位，那麼您就遇到了資料庫資料遷移所要解決的問題。

Laravel 的 `Schema` [facade](/docs/{{version}}/facades) 提供了與資料庫無關的支援，用於在 Laravel 支援的所有資料庫系統上建立和操作資料表。通常，資料遷移會使用此 facade 來建立和修改資料庫資料表及欄位。

<a name="generating-migrations"></a>
## 產生資料遷移

您可以使用 `make:migration` [Artisan command](/docs/{{version}}/artisan) 來產生一個資料庫資料遷移。新的資料遷移將會被放置在您的 `database/migrations` 目錄中。每個資料遷移檔案名稱都包含一個時間戳記，讓 Laravel 能夠判斷資料遷移的順序：

```shell
php artisan make:migration create_flights_table
```

Laravel 會根據資料遷移的名稱來嘗試猜測資料表的名稱，以及該資料遷移是否會建立新的資料表。如果 Laravel 能夠從資料遷移名稱判斷出資料表名稱，Laravel 將會預先填入產生的資料遷移檔案，其中包含指定的資料表。否則，您可以簡單地在資料遷移檔案中手動指定資料表。

如果您想為產生的資料遷移指定自訂路徑，可以在執行 `make:migration` 指令時使用 `--path` 選項。給定的路徑應相對於您應用程式的根目錄。

> [!NOTE]
> 資料遷移模板可以使用 [stub publishing](/docs/{{version}}/artisan#stub-customization) 進行自訂。

<a name="squashing-migrations"></a>
### 壓縮資料遷移

隨著您建構應用程式，您可能會隨著時間累積越來越多的資料遷移。這可能導致您的 `database/migrations` 目錄變得臃腫，可能包含數百個資料遷移。如果您願意，您可以將資料遷移「壓縮」成一個單一的 SQL 檔案。要開始，請執行 `schema:dump` 指令：

```shell
php artisan schema:dump

# Dump the current database schema and prune all existing migrations...
php artisan schema:dump --prune
```

當您執行此指令時，Laravel 會將一個「綱要」檔案寫入您應用程式的 `database/schema` 目錄。綱要檔案的名稱將對應到資料庫連線。現在，當您嘗試遷移您的資料庫且沒有其他資料遷移被執行時，Laravel 將首先執行您正在使用的資料庫連線的綱要檔案中的 SQL 語句。在執行完綱要檔案的 SQL 語句後，Laravel 將執行任何不屬於綱要傾印的剩餘資料遷移。

如果您的應用程式測試使用與您在本地開發期間通常使用的資料庫連線不同，您應該確保您已經使用該資料庫連線傾印了一個綱要檔案，以便您的測試能夠建立您的資料庫。您可能希望在傾印您在本地開發期間通常使用的資料庫連線之後執行此操作：

```shell
php artisan schema:dump
php artisan schema:dump --database=testing --prune
```

您應該將資料庫綱要檔案提交到原始碼控制，以便您團隊中的其他新開發人員可以快速建立您應用程式的初始資料庫結構。

> [!WARNING]
> 資料遷移壓縮僅適用於 MariaDB、MySQL、PostgreSQL 和 SQLite 資料庫，並利用了資料庫的命令列用戶端。

<a name="migration-structure"></a>
## 資料遷移結構

一個資料遷移類別包含兩個方法：`up` 和 `down`。`up` 方法用於向資料庫中新增資料表、欄位或索引，而 `down` 方法則應反轉 `up` 方法所執行的操作。

在這兩個方法中，您都可以使用 Laravel 綱要產生器以聲明式方式建立和修改資料表。要了解 `Schema` 產生器上所有可用的方法，請[查閱其文件](#creating-tables)。例如，以下資料遷移會建立一個 `flights` 資料表：

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
#### 設定資料遷移連線

如果您的資料遷移將與應用程式預設資料庫連線以外的資料庫連線互動，您應該設定資料遷移的 `$connection` 屬性：

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
#### 跳過資料遷移

有時，資料遷移可能旨在支援尚未啟用的功能，並且您不希望它現在執行。在這種情況下，您可以在資料遷移上定義一個 `shouldRun` 方法。如果 `shouldRun` 方法回傳 `false`，該資料遷移將會被跳過：

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
## 執行資料遷移

若要執行所有尚未執行的資料遷移，請執行 `migrate` Artisan 指令：

```shell
php artisan migrate
```

如果您想查看哪些資料遷移已執行，哪些仍在等待執行，您可以使用 `migrate:status` Artisan 指令：

```shell
php artisan migrate:status
```

如果您想查看資料遷移將要執行的 SQL 語句，而無需實際執行它們，您可以為 `migrate` 指令提供 `--pretend` 旗標：

```shell
php artisan migrate --pretend
```

#### 隔離資料遷移執行

如果您正在將應用程式部署到多個伺服器上，並將資料遷移作為部署流程的一部分執行，您可能不希望兩個伺服器同時嘗試執行資料庫遷移。為避免這種情況，您可以在呼叫 `migrate` 指令時使用 `isolated` 選項。

當提供了 `isolated` 選項時，Laravel 將在嘗試執行資料遷移之前，使用應用程式的快取驅動程式取得一個原子鎖。在該鎖定被持有期間，所有其他執行 `migrate` 指令的嘗試將不會執行；但是，該指令仍將以成功的結束狀態碼退出：

```shell
php artisan migrate --isolated
```

> [!WARNING]
> 要使用此功能，您的應用程式必須使用 `memcached`、`redis`、`dynamodb`、`database`、`file` 或 `array` 快取驅動程式作為應用程式的預設快取驅動程式。此外，所有伺服器都必須與相同的中央快取伺服器進行通訊。

<a name="forcing-migrations-to-run-in-production"></a>
#### 強制資料遷移在生產環境中執行

某些資料遷移操作是破壞性的，這意味著它們可能會導致您遺失資料。為了保護您免受在生產資料庫上執行這些指令的影響，在指令執行前將會要求確認。若要強制指令在沒有提示的情況下執行，請使用 `--force` 旗標：

```shell
php artisan migrate --force
```

<a name="rolling-back-migrations"></a>
### 復原資料遷移

若要復原最新一次的資料遷移操作，您可以使用 `rollback` Artisan 指令。此指令會復原最後一批資料遷移，其中可能包含多個資料遷移檔案：

```shell
php artisan migrate:rollback
```

您可以透過為 `rollback` 指令提供 `step` 選項來復原有限數量的資料遷移。例如，以下指令將復原最後五個資料遷移：

```shell
php artisan migrate:rollback --step=5
```

您可以透過為 `rollback` 指令提供 `batch` 選項來復原特定批次的資料遷移，其中 `batch` 選項對應於應用程式 `migrations` 資料庫表中的批次值。例如，以下指令將復原第三批中的所有資料遷移：

```shell
php artisan migrate:rollback --batch=3
```

如果您想查看資料遷移將要執行的 SQL 語句，而無需實際執行它們，您可以為 `migrate:rollback` 指令提供 `--pretend` 旗標：

```shell
php artisan migrate:rollback --pretend
```

`migrate:reset` 指令將復原應用程式的所有資料遷移：

```shell
php artisan migrate:reset
```

<a name="roll-back-migrate-using-a-single-command"></a>
#### 使用單一指令復原並執行資料遷移

`migrate:refresh` 指令將復原所有資料遷移，然後執行 `migrate` 指令。此指令有效地重新建立整個資料庫：

```shell
php artisan migrate:refresh

# Refresh the database and run all database seeds...
php artisan migrate:refresh --seed
```

您可以透過為 `refresh` 指令提供 `step` 選項來復原並重新執行有限數量的資料遷移。例如，以下指令將復原並重新執行最後五個資料遷移：

```shell
php artisan migrate:refresh --step=5
```

<a name="drop-all-tables-migrate"></a>
#### 刪除所有資料表並執行資料遷移

`migrate:fresh` 指令將刪除資料庫中的所有資料表，然後執行 `migrate` 指令：

```shell
php artisan migrate:fresh

php artisan migrate:fresh --seed
```

預設情況下，`migrate:fresh` 指令只會刪除預設資料庫連線中的資料表。但是，您可以使用 `--database` 選項來指定應執行資料遷移的資料庫連線。資料庫連線名稱應與應用程式 `database` [設定檔](/docs/{{version}}/configuration) 中定義的連線相對應：

```shell
php artisan migrate:fresh --database=admin
```

> [!WARNING]
> `migrate:fresh` 指令將刪除所有資料庫資料表，無論其前綴為何。在與其他應用程式共用的資料庫上開發時，應謹慎使用此指令。

<a name="tables"></a>
## 資料表

<a name="creating-tables"></a>
### 建立資料表

若要建立新的資料庫資料表，請使用 `Schema` Facade 上的 `create` 方法。`create` 方法接受兩個引數：第一個是資料表的名稱，第二個是一個接收 `Blueprint` 物件的閉包，可用於定義新資料表：

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

建立資料表時，您可以使用任何 Schema 產生器的[欄位方法](#creating-columns)來定義資料表的欄位。

<a name="determining-table-column-existence"></a>
#### 判斷資料表 / 欄位是否存在

您可以使用 `hasTable`、`hasColumn` 和 `hasIndex` 方法來判斷資料表、欄位或索引是否存在：

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

如果您想在非應用程式預設資料庫連線的資料庫連線上執行 Schema 操作，請使用 `connection` 方法：

```php
Schema::connection('sqlite')->create('users', function (Blueprint $table) {
    $table->id();
});
```

此外，還有其他一些屬性與方法可用於定義資料表建立的其他層面。使用 MariaDB 或 MySQL 時，`engine` 屬性可用於指定資料表的儲存引擎：

```php
Schema::create('users', function (Blueprint $table) {
    $table->engine('InnoDB');

    // ...
});
```

使用 MariaDB 或 MySQL 時，`charset` 和 `collation` 屬性可用於指定所建立資料表的字元集和排序規則：

```php
Schema::create('users', function (Blueprint $table) {
    $table->charset('utf8mb4');
    $table->collation('utf8mb4_unicode_ci');

    // ...
});
```

`temporary` 方法可用於指示資料表應為「暫時的」。暫時資料表僅對當前連線的資料庫會話可見，並在連線關閉時自動刪除：

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->temporary();

    // ...
});
```

如果您想為資料庫資料表新增「註解」，可以在資料表實例上呼叫 `comment` 方法。目前，資料表註解僅支援 MariaDB、MySQL 和 PostgreSQL：

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->comment('Business calculations');

    // ...
});
```

<a name="updating-tables"></a>
### 更新資料表

`Schema` Facade 上的 `table` 方法可用於更新現有的資料表。與 `create` 方法類似，`table` 方法接受兩個引數：資料表的名稱，以及一個接收 `Blueprint` 實例的閉包，您可以使用該實例向資料表新增欄位或索引：

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

若要刪除現有的資料表，您可以使用 `drop` 或 `dropIfExists` 方法：

```php
Schema::drop('users');

Schema::dropIfExists('users');
```

<a name="renaming-tables-with-foreign-keys"></a>
#### 重新命名帶有外部鍵的資料表

在重新命名資料表之前，您應驗證資料表上的任何外部鍵限制在您的資料遷移檔案中都具有明確的名稱，而不是讓 Laravel 指定一個基於慣例的名稱。否則，外部鍵限制名稱將指向舊的資料表名稱。

<a name="columns"></a>
## 欄位

<a name="creating-columns"></a>
### 建立欄位

`Schema` Facade 上的 `table` 方法可用於更新現有的資料表。類似於 `create` 方法，`table` 方法接受兩個引數：資料表的名稱，以及一個閉包 (closure)，它會接收一個 `Illuminate\Database\Schema\Blueprint` 實例 (instance)，您可以用來向資料表添加欄位：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->integer('votes');
});
```

<a name="available-column-types"></a>
### 可用的欄位型別

Schema Builder Blueprint 提供了多種方法，對應可新增至資料庫資料表的各種欄位型別。每種可用方法都列在下方的表格中：

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
#### 布林型別

<div class="collection-method-list" markdown="1">

[boolean](#column-method-boolean)

</div>


<a name="strings-and-texts-method-list"></a>
#### 字串與文字型別

<div class="collection-method-list" markdown="1">

[char](#column-method-char)
[longText](#column-method-longText)
[mediumText](#column-method-mediumText)
[string](#column-method-string)
[text](#column-method-text)
[tinyText](#column-method-tinyText)

</div>


<a name="numbers--method-list"></a>
#### 數值型別

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
#### 日期與時間型別

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
#### 二進位型別

<div class="collection-method-list" markdown="1">

[binary](#column-method-binary)

</div>


<a name="object-and-jsons-method-list"></a>
#### 物件與 JSON 型別

<div class="collection-method-list" markdown="1">

[json](#column-method-json)
[jsonb](#column-method-jsonb)

</div>


<a name="uuids-and-ulids-method-list"></a>
#### UUID 與 ULID 型別

<div class="collection-method-list" markdown="1">

[ulid](#column-method-ulid)
[ulidMorphs](#column-method-ulidMorphs)
[uuid](#column-method-uuid)
[uuidMorphs](#column-method-uuidMorphs)
[nullableUlidMorphs](#column-method-nullableUlidMorphs)
[nullableUuidMorphs](#column-method-nullableUuidMorphs)

</div>


<a name="spatials-method-list"></a>
#### 空間型別

<div class="collection-method-list" markdown="1">

[geography](#column-method-geography)
[geometry](#column-method-geometry)

</div>


#### 關聯型別

<div class="collection-method-list" markdown="1">

[foreignId](#column-method-foreignId)
[foreignIdFor](#column-method-foreignIdFor)
[foreignUlid](#column-method-foreignUlid)
[foreignUuid](#column-method-foreignUuid)
[morphs](#column-method-morphs)
[nullableMorphs](#column-method-nullableMorphs)

</div>


<a name="spacifics-method-list"></a>
#### 特殊型別

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

`bigIncrements` 方法會建立一個自動遞增的 `UNSIGNED BIGINT` (主鍵) 等效欄位：

```php
$table->bigIncrements('id');
```


<a name="column-method-bigInteger"></a>
#### `bigInteger()` {.collection-method}

`bigInteger` 方法會建立一個 `BIGINT` 等效欄位：

```php
$table->bigInteger('votes');
```


<a name="column-method-binary"></a>
#### `binary()` {.collection-method}

`binary` 方法會建立一個 `BLOB` 等效欄位：

```php
$table->binary('photo');
```

當使用 MySQL、MariaDB 或 SQL Server 時，您可以傳遞 `length` 和 `fixed` 引數來建立 `VARBINARY` 或 `BINARY` 等效欄位：

```php
$table->binary('data', length: 16); // VARBINARY(16)

$table->binary('data', length: 16, fixed: true); // BINARY(16)
```


<a name="column-method-boolean"></a>
#### `boolean()` {.collection-method}

`boolean` 方法會建立一個 `BOOLEAN` 等效欄位：

```php
$table->boolean('confirmed');
```


<a name="column-method-char"></a>
#### `char()` {.collection-method}

`char` 方法會建立一個帶有指定長度的 `CHAR` 等效欄位：

```php
$table->char('name', length: 100);
```


<a name="column-method-dateTimeTz"></a>
#### `dateTimeTz()` {.collection-method}

`dateTimeTz` 方法會建立一個帶有可選小數秒精度的 `DATETIME` (含時區) 等效欄位：

```php
$table->dateTimeTz('created_at', precision: 0);
```


<a name="column-method-dateTime"></a>
#### `dateTime()` {.collection-method}

`dateTime` 方法會建立一個帶有可選小數秒精度的 `DATETIME` 等效欄位：

```php
$table->dateTime('created_at', precision: 0);
```


<a name="column-method-date"></a>
#### `date()` {.collection-method}

`date` 方法會建立一個 `DATE` 等效欄位：

```php
$table->date('created_at');
```


<a name="column-method-decimal"></a>
#### `decimal()` {.collection-method}

`decimal` 方法會建立一個帶有指定精度 (總位數) 和小數位數 (小數點後位數) 的 `DECIMAL` 等效欄位：

```php
$table->decimal('amount', total: 8, places: 2);
```


<a name="column-method-double"></a>
#### `double()` {.collection-method}

`double` 方法會建立一個 `DOUBLE` 等效欄位：

```php
$table->double('amount');
```


<a name="column-method-enum"></a>
#### `enum()` {.collection-method}

`enum` 方法會建立一個帶有指定有效值的 `ENUM` 等效欄位：

```php
$table->enum('difficulty', ['easy', 'hard']);
```

當然，您可以使用 `Enum::cases()` 方法，而不是手動定義允許值的陣列：

```php
use App\Enums\Difficulty;

$table->enum('difficulty', Difficulty::cases());
```


<a name="column-method-float"></a>
#### `float()` {.collection-method}

`float` 方法會建立一個帶有指定精度的 `FLOAT` 等效欄位：

```php
$table->float('amount', precision: 53);
```


<a name="column-method-foreignId"></a>
#### `foreignId()` {.collection-method}

`foreignId` 方法會建立一個 `UNSIGNED BIGINT` 等效欄位：

```php
$table->foreignId('user_id');
```


<a name="column-method-foreignIdFor"></a>
#### `foreignIdFor()` {.collection-method}

`foreignIdFor` 方法會為給定的 Model Class 新增一個 `{column}_id` 等效欄位。欄位型別會根據 Model Key 型別而為 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`：

```php
$table->foreignIdFor(User::class);
```


<a name="column-method-foreignUlid"></a>
#### `foreignUlid()` {.collection-method}

`foreignUlid` 方法會建立一個 `ULID` 等效欄位：

```php
$table->foreignUlid('user_id');
```


<a name="column-method-foreignUuid"></a>
#### `foreignUuid()` {.collection-method}

`foreignUuid` 方法會建立一個 `UUID` 等效欄位：

```php
$table->foreignUuid('user_id');
```


<a name="column-method-geography"></a>
#### `geography()` {.collection-method}

`geography` 方法會建立一個帶有指定空間型別與 SRID (空間參考系統識別碼) 的 `GEOGRAPHY` 等效欄位：

```php
$table->geography('coordinates', subtype: 'point', srid: 4326);
```

> [!NOTE]
> 對於空間型別的支援取決於您的資料庫驅動。請參考您資料庫的文件。如果您的應用程式正在使用 PostgreSQL 資料庫，在使用 `geography` 方法之前，您必須安裝 [PostGIS](https://postgis.net) 擴充功能。


<a name="column-method-geometry"></a>
#### `geometry()` {.collection-method}

`geometry` 方法會建立一個帶有指定空間型別與 SRID (空間參考系統識別碼) 的 `GEOMETRY` 等效欄位：

```php
$table->geometry('positions', subtype: 'point', srid: 0);
```

> [!NOTE]
> 對於空間型別的支援取決於您的資料庫驅動。請參考您資料庫的文件。如果您的應用程式正在使用 PostgreSQL 資料庫，在使用 `geometry` 方法之前，您必須安裝 [PostGIS](https://postgis.net) 擴充功能。


<a name="column-method-id"></a>
#### `id()` {.collection-method}

`id` 方法是 `bigIncrements` 方法的別名。預設情況下，此方法會建立一個 `id` 欄位；但是，如果您想為該欄位指定不同的名稱，您可以傳遞一個欄位名稱：

```php
$table->id();
```


<a name="column-method-increments"></a>
#### `increments()` {.collection-method}

`increments` 方法會建立一個作為主鍵的自動遞增 `UNSIGNED INTEGER` 等效欄位：

```php
$table->increments('id');
```


<a name="column-method-integer"></a>
#### `integer()` {.collection-method}

`integer` 方法會建立一個 `INTEGER` 等效欄位：

```php
$table->integer('votes');
```


<a name="column-method-ipAddress"></a>
#### `ipAddress()` {.collection-method}

`ipAddress` 方法會建立一個 `VARCHAR` 等效欄位：

```php
$table->ipAddress('visitor');
```

當使用 PostgreSQL 時，會建立一個 `INET` 欄位。


<a name="column-method-json"></a>
#### `json()` {.collection-method}

`json` 方法會建立一個 `JSON` 等效欄位：

```php
$table->json('options');
```

當使用 SQLite 時，會建立一個 `TEXT` 欄位。


<a name="column-method-jsonb"></a>
#### `jsonb()` {.collection-method}

`jsonb` 方法會建立一個 `JSONB` 等效欄位：

```php
$table->jsonb('options');
```

當使用 SQLite 時，會建立一個 `TEXT` 欄位。


<a name="column-method-longText"></a>
#### `longText()` {.collection-method}

`longText` 方法會建立一個 `LONGTEXT` 等效欄位：

```php
$table->longText('description');
```

當使用 MySQL 或 MariaDB 時，您可以對欄位套用 `binary` 字元集，以建立 `LONGBLOB` 等效欄位：

```php
$table->longText('data')->charset('binary'); // LONGBLOB
```


<a name="column-method-macAddress"></a>
#### `macAddress()` {.collection-method}

`macAddress` 方法會建立一個旨在儲存 MAC 位址的欄位。某些資料庫系統 (例如 PostgreSQL) 為此型別資料提供了專用欄位型別。其他資料庫系統則會使用字串等效欄位：

```php
$table->macAddress('device');
```


<a name="column-method-mediumIncrements"></a>
#### `mediumIncrements()` {.collection-method}

`mediumIncrements` 方法會建立一個作為主鍵的自動遞增 `UNSIGNED MEDIUMINT` 等效欄位：

```php
$table->mediumIncrements('id');
```


<a name="column-method-mediumInteger"></a>
#### `mediumInteger()` {.collection-method}

`mediumInteger` 方法會建立一個 `MEDIUMINT` 等效欄位：

```php
$table->mediumInteger('votes');
```


<a name="column-method-mediumText"></a>
#### `mediumText()` {.collection-method}

`mediumText` 方法會建立一個 `MEDIUMTEXT` 等效欄位：

```php
$table->mediumText('description');
```

當使用 MySQL 或 MariaDB 時，您可以對欄位套用 `binary` 字元集，以建立 `MEDIUMBLOB` 等效欄位：

```php
$table->mediumText('data')->charset('binary'); // MEDIUMBLOB
```


<a name="column-method-morphs"></a>
#### `morphs()` {.collection-method}

`morphs` 方法是一個便捷方法，用於新增 `{column}_id` 等效欄位和 `{column}_type` `VARCHAR` 等效欄位。`{column}_id` 的欄位型別會根據 Model Key 型別而為 `UNSIGNED BIGINT`、`CHAR(36)` 或 `CHAR(26)`。

此方法旨在用於定義多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)所需的欄位。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 欄位：

```php
$table->morphs('taggable');
```


<a name="column-method-nullableMorphs"></a>
#### `nullableMorphs()` {.collection-method}

此方法類似於 [morphs](#column-method-morphs) 方法；但是，所建立的欄位將是「可為 Null」的：

```php
$table->nullableMorphs('taggable');
```


<a name="column-method-nullableUlidMorphs"></a>
#### `nullableUlidMorphs()` {.collection-method}

此方法類似於 [ulidMorphs](#column-method-ulidMorphs) 方法；但是，所建立的欄位將是「可為 Null」的：

```php
$table->nullableUlidMorphs('taggable');
```


<a name="column-method-nullableUuidMorphs"></a>
#### `nullableUuidMorphs()` {.collection-method}

此方法類似於 [uuidMorphs](#column-method-uuidMorphs) 方法；但是，所建立的欄位將是「可為 Null」的：

```php
$table->nullableUuidMorphs('taggable');
```


<a name="column-method-rememberToken"></a>
#### `rememberToken()` {.collection-method}

`rememberToken` 方法會建立一個可為 null 的 `VARCHAR(100)` 等效欄位，用於儲存目前的「記住我」[認證權杖](/docs/{{version}}/authentication#remembering-users)：

```php
$table->rememberToken();
```


<a name="column-method-set"></a>
#### `set()` {.collection-method}

`set` 方法會建立一個帶有指定有效值列表的 `SET` 等效欄位：

```php
$table->set('flavors', ['strawberry', 'vanilla']);
```


<a name="column-method-smallIncrements"></a>
#### `smallIncrements()` {.collection-method}

`smallIncrements` 方法會建立一個作為主鍵的自動遞增 `UNSIGNED SMALLINT` 等效欄位：

```php
$table->smallIncrements('id');
```


<a name="column-method-smallInteger"></a>
#### `smallInteger()` {.collection-method}

`smallInteger` 方法會建立一個 `SMALLINT` 等效欄位：

```php
$table->smallInteger('votes');
```


<a name="column-method-softDeletesTz"></a>
#### `softDeletesTz()` {.collection-method}

`softDeletesTz` 方法會新增一個可為 null 的 `deleted_at` `TIMESTAMP` (含時區) 等效欄位，帶有可選的小數秒精度。此欄位旨在儲存 Eloquent「軟刪除」功能所需的 `deleted_at` 時間戳記：

```php
$table->softDeletesTz('deleted_at', precision: 0);
```


<a name="column-method-softDeletes"></a>
#### `softDeletes()` {.collection-method}

`softDeletes` 方法會新增一個可為 null 的 `deleted_at` `TIMESTAMP` 等效欄位，帶有可選的小數秒精度。此欄位旨在儲存 Eloquent「軟刪除」功能所需的 `deleted_at` 時間戳記：

```php
$table->softDeletes('deleted_at', precision: 0);
```


<a name="column-method-string"></a>
#### `string()` {.collection-method}

`string` 方法會建立一個帶有指定長度的 `VARCHAR` 等效欄位：

```php
$table->string('name', length: 100);
```


<a name="column-method-text"></a>
#### `text()` {.collection-method}

`text` 方法會建立一個 `TEXT` 等效欄位：

```php
$table->text('description');
```

當使用 MySQL 或 MariaDB 時，您可以對欄位套用 `binary` 字元集，以建立 `BLOB` 等效欄位：

```php
$table->text('data')->charset('binary'); // BLOB
```


<a name="column-method-timeTz"></a>
#### `timeTz()` {.collection-method}

`timeTz` 方法會建立一個帶有可選小數秒精度的 `TIME` (含時區) 等效欄位：

```php
$table->timeTz('sunrise', precision: 0);
```


<a name="column-method-time"></a>
#### `time()` {.collection-method}

`time` 方法會建立一個帶有可選小數秒精度的 `TIME` 等效欄位：

```php
$table->time('sunrise', precision: 0);
```


<a name="column-method-timestampTz"></a>
#### `timestampTz()` {.collection-method}

`timestampTz` 方法會建立一個帶有可選小數秒精度的 `TIMESTAMP` (含時區) 等效欄位：

```php
$table->timestampTz('added_at', precision: 0);
```


<a name="column-method-timestamp"></a>
#### `timestamp()` {.collection-method}

`timestamp` 方法會建立一個帶有可選小數秒精度的 `TIMESTAMP` 等效欄位：

```php
$table->timestamp('added_at', precision: 0);
```


<a name="column-method-timestampsTz"></a>
#### `timestampsTz()` {.collection-method}

`timestampsTz` 方法會建立 `created_at` 和 `updated_at` `TIMESTAMP` (含時區) 等效欄位，帶有可選的小數秒精度：

```php
$table->timestampsTz(precision: 0);
```


<a name="column-method-timestamps"></a>
#### `timestamps()` {.collection-method}

`timestamps` 方法會建立 `created_at` 和 `updated_at` `TIMESTAMP` 等效欄位，帶有可選的小數秒精度：

```php
$table->timestamps(precision: 0);
```


<a name="column-method-tinyIncrements"></a>
#### `tinyIncrements()` {.collection-method}

`tinyIncrements` 方法會建立一個作為主鍵的自動遞增 `UNSIGNED TINYINT` 等效欄位：

```php
$table->tinyIncrements('id');
```


<a name="column-method-tinyInteger"></a>
#### `tinyInteger()` {.collection-method}

`tinyInteger` 方法會建立一個 `TINYINT` 等效欄位：

```php
$table->tinyInteger('votes');
```


<a name="column-method-tinyText"></a>
#### `tinyText()` {.collection-method}

`tinyText` 方法會建立一個 `TINYTEXT` 等效欄位：

```php
$table->tinyText('notes');
```

當使用 MySQL 或 MariaDB 時，您可以對欄位套用 `binary` 字元集，以建立 `TINYBLOB` 等效欄位：

```php
$table->tinyText('data')->charset('binary'); // TINYBLOB
```


<a name="column-method-unsignedBigInteger"></a>
#### `unsignedBigInteger()` {.collection-method}

`unsignedBigInteger` 方法會建立一個 `UNSIGNED BIGINT` 等效欄位：

```php
$table->unsignedBigInteger('votes');
```


<a name="column-method-unsignedInteger"></a>
#### `unsignedInteger()` {.collection-method}

`unsignedInteger` 方法會建立一個 `UNSIGNED INTEGER` 等效欄位：

```php
$table->unsignedInteger('votes');
```


<a name="column-method-unsignedMediumInteger"></a>
#### `unsignedMediumInteger()` {.collection-method}

`unsignedMediumInteger` 方法會建立一個 `UNSIGNED MEDIUMINT` 等效欄位：

```php
$table->unsignedMediumInteger('votes');
```


<a name="column-method-unsignedSmallInteger"></a>
#### `unsignedSmallInteger()` {.collection-method}

`unsignedSmallInteger` 方法會建立一個 `UNSIGNED SMALLINT` 等效欄位：

```php
$table->unsignedSmallInteger('votes');
```


<a name="column-method-unsignedTinyInteger"></a>
#### `unsignedTinyInteger()` {.collection-method}

`unsignedTinyInteger` 方法會建立一個 `UNSIGNED TINYINT` 等效欄位：

```php
$table->unsignedTinyInteger('votes');
```


<a name="column-method-ulidMorphs"></a>
#### `ulidMorphs()` {.collection-method}

`ulidMorphs` 方法是一個便捷方法，用於新增 `{column}_id` `CHAR(26)` 等效欄位和 `{column}_type` `VARCHAR` 等效欄位。

此方法旨在用於定義使用 ULID 識別碼的多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)所需的欄位。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 欄位：

```php
$table->ulidMorphs('taggable');
```


<a name="column-method-uuidMorphs"></a>
#### `uuidMorphs()` {.collection-method}

`uuidMorphs` 方法是一個便捷方法，用於新增 `{column}_id` `CHAR(36)` 等效欄位和 `{column}_type` `VARCHAR` 等效欄位。

此方法旨在用於定義使用 UUID 識別碼的多型 [Eloquent 關聯](/docs/{{version}}/eloquent-relationships)所需的欄位。在以下範例中，將會建立 `taggable_id` 和 `taggable_type` 欄位：

```php
$table->uuidMorphs('taggable');
```


<a name="column-method-ulid"></a>
#### `ulid()` {.collection-method}

`ulid` 方法會建立一個 `ULID` 等效欄位：

```php
$table->ulid('id');
```


<a name="column-method-uuid"></a>
#### `uuid()` {.collection-method}

`uuid` 方法會建立一個 `UUID` 等效欄位：

```php
$table->uuid('id');
```


<a name="column-method-vector"></a>
#### `vector()` {.collection-method}

`vector` 方法會建立一個 `vector` 等效欄位：

```php
$table->vector('embedding', dimensions: 100);
```


<a name="column-method-year"></a>
#### `year()` {.collection-method}

`year` 方法會建立一個 `YEAR` 等效欄位：

```php
$table->year('birth_year');
```

<a name="column-modifiers"></a>
### 欄位修改器

除了上述列出的欄位型別之外，在新增欄位至資料表時，還有一些欄位「修改器」可用。舉例來說，若要將欄位設定為「可為空 (nullable)」，可以使用 `nullable` 方法：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->string('email')->nullable();
});
```

下表包含所有可用的欄位修改器。此列表不包含[索引修改器](#creating-indexes)：

<div class="overflow-auto">

| Modifier                            | Description                                                                                    |
| ----------------------------------- | ---------------------------------------------------------------------------------------------- |
| `->after('column')`                 | 將欄位放置在另一個欄位「之後」(MariaDB / MySQL)。                                                |
| `->autoIncrement()`                 | 將 `INTEGER` 欄位設定為自動遞增(主鍵)。                                                         |
| `->charset('utf8mb4')`              | 為欄位指定字元集(MariaDB / MySQL)。                                                             |
| `->collation('utf8mb4_unicode_ci')` | 為欄位指定排序規則。                                                                             |
| `->comment('my comment')`           | 為欄位新增註解(MariaDB / MySQL / PostgreSQL)。                                                  |
| `->default($value)`                 | 為欄位指定「預設」值。                                                                           |
| `->first()`                         | 將欄位放置在資料表「最前面」(MariaDB / MySQL)。                                                  |
| `->from($integer)`                  | 設定自動遞增欄位的起始值(MariaDB / MySQL / PostgreSQL)。                                         |
| `->invisible()`                     | 使欄位對於 `SELECT *` 查詢「不可見」(MariaDB / MySQL)。                                         |
| `->nullable($value = true)`         | 允許 `NULL` 值插入到欄位中。                                                                   |
| `->storedAs($expression)`           | 建立一個儲存的生成欄位(MariaDB / MySQL / PostgreSQL / SQLite)。                                  |
| `->unsigned()`                      | 將 `INTEGER` 欄位設定為 `UNSIGNED`(MariaDB / MySQL)。                                           |
| `->useCurrent()`                    | 將 `TIMESTAMP` 欄位設定為使用 `CURRENT_TIMESTAMP` 作為預設值。                                  |
| `->useCurrentOnUpdate()`            | 將 `TIMESTAMP` 欄位設定為在記錄更新時使用 `CURRENT_TIMESTAMP`(MariaDB / MySQL)。               |
| `->virtualAs($expression)`          | 建立一個虛擬的生成欄位(MariaDB / MySQL / SQLite)。                                              |
| `->generatedAs($expression)`        | 建立具有指定序列選項的身份欄位(PostgreSQL)。                                                   |
| `->always()`                        | 定義序列值對身份欄位輸入的優先權(PostgreSQL)。                                                 |

</div>


<a name="default-expressions"></a>
#### 預設表達式

`default` 修改器接受一個值或一個 `Illuminate\Database\Query\Expression` 實例。使用 `Expression` 實例會阻止 Laravel 將值包裝在引號中，並允許您使用資料庫特定的函數。在需要為 JSON 欄位分配預設值時，這尤其有用：

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
> 對於預設表達式的支援取決於您的資料庫驅動程式、資料庫版本和欄位型別。請參閱您的資料庫文件。


<a name="column-order"></a>
#### 欄位順序

當使用 MariaDB 或 MySQL 資料庫時，可以使用 `after` 方法在資料表中的現有欄位之後新增欄位：

```php
$table->after('password', function (Blueprint $table) {
    $table->string('address_line1');
    $table->string('address_line2');
    $table->string('city');
});
```


<a name="modifying-columns"></a>
### 修改欄位

`change` 方法允許您修改現有欄位的型別和屬性。例如，您可能希望增加 `string` 欄位的長度。要了解 `change` 方法的實際應用，讓我們將 `name` 欄位的長度從 25 增加到 50。要實現這一點，我們只需定義欄位的新狀態，然後呼叫 `change` 方法：

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('name', 50)->change();
});
```

修改欄位時，您必須明確地包含所有您希望保留在欄位定義上的修改器——任何缺失的屬性都將被刪除。例如，若要保留 `unsigned`、`default` 和 `comment` 屬性，在更改欄位時必須明確呼叫每個修改器：

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')->unsigned()->default(1)->comment('my comment')->change();
});
```

`change` 方法不會更改欄位的索引。因此，您可以在修改欄位時使用索引修改器來明確新增或刪除索引：

```php
// Add an index...
$table->bigIncrements('id')->primary()->change();

// Drop an index...
$table->char('postal_code', 10)->unique(false)->change();
```


<a name="renaming-columns"></a>
### 重新命名欄位

若要重新命名欄位，您可以使用 Schema Builder 提供的 `renameColumn` 方法：

```php
Schema::table('users', function (Blueprint $table) {
    $table->renameColumn('from', 'to');
});
```


<a name="dropping-columns"></a>
### 刪除欄位

若要刪除欄位，您可以使用 Schema Builder 上的 `dropColumn` 方法：

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

Laravel 提供了幾種方便的方法來刪除常見型別的欄位。下表描述了這些方法：

<div class="overflow-auto">

| Command                             | Description                                           |
| ----------------------------------- | ----------------------------------------------------- |
| `$table->dropMorphs('morphable');`  | 刪除 `morphable_id` 和 `morphable_type` 欄位。         |
| `$table->dropRememberToken();`      | 刪除 `remember_token` 欄位。                          |
| `$table->dropSoftDeletes();`        | 刪除 `deleted_at` 欄位。                              |
| `$table->dropSoftDeletesTz();`      | `dropSoftDeletes()` 方法的別名。                       |
| `$table->dropTimestamps();`         | 刪除 `created_at` 和 `updated_at` 欄位。              |
| `$table->dropTimestampsTz();`       | `dropTimestamps()` 方法的別名。                        |

</div>

<a name="indexes"></a>
## 索引


<a name="creating-indexes"></a>
### 建立索引

Laravel 的 Schema 建構器支援多種類型的索引。以下範例建立了一個新的 `email` 欄位並指定其值必須是唯一的。要建立這個索引，我們可以在欄位定義上鏈接 `unique` 方法：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('users', function (Blueprint $table) {
    $table->string('email')->unique();
});
```

或者，您可以在定義欄位後建立索引。為此，您應該在 Schema 建構器藍圖上呼叫 `unique` 方法。此方法接受應建立唯一索引的欄位名稱：

```php
$table->unique('email');
```

您甚至可以將欄位陣列傳遞給索引方法，以建立複合索引：

```php
$table->index(['account_id', 'created_at']);
```

建立索引時，Laravel 會根據資料表、欄位名稱和索引類型自動產生索引名稱，但您可以傳遞第二個引數給方法以自行指定索引名稱：

```php
$table->unique('email', 'unique_email');
```


<a name="available-index-types"></a>
#### 可用的索引型別

Laravel 的 Schema 建構器藍圖類別提供了建立 Laravel 支援的每種索引類型的方法。每個索引方法都接受一個可選的第二個引數來指定索引的名稱。如果省略，名稱將根據用於索引的資料表和欄位名稱以及索引類型派生。每個可用的索引方法都將在下表中說明：

<div class="overflow-auto">

| 指令                                             | 說明                                                       |
| ------------------------------------------------ | ---------------------------------------------------------- |
| `$table->primary('id');`                         | 新增主鍵。                                                 |
| `$table->primary(['id', 'parent_id']);`          | 新增複合鍵。                                               |
| `$table->unique('email');`                       | 新增唯一索引。                                             |
| `$table->index('state');`                        | 新增索引。                                                 |
| `$table->fullText('body');`                      | 新增全文索引 (MariaDB / MySQL / PostgreSQL)。              |
| `$table->fullText('body')->language('english');` | 新增指定語言的全文索引 (PostgreSQL)。                      |
| `$table->spatialIndex('location');`              | 新增空間索引 (SQLite 除外)。                               |

</div>


<a name="renaming-indexes"></a>
### 重新命名索引

要重新命名索引，您可以使用 Schema 建構器藍圖提供的 `renameIndex` 方法。此方法接受當前索引名稱作為第一個引數，以及所需的名稱作為第二個引數：

```php
$table->renameIndex('from', 'to')
```


<a name="dropping-indexes"></a>
### 刪除索引

要刪除索引，您必須指定索引的名稱。預設情況下，Laravel 會根據資料表名稱、索引欄位名稱和索引類型自動分配索引名稱。以下是一些範例：

<div class="overflow-auto">

| 指令                                                       | 說明                                                    |
| ---------------------------------------------------------- | ------------------------------------------------------- |
| `$table->dropPrimary('users_id_primary');`               | 從「users」資料表刪除主鍵。                             |
| `$table->dropUnique('users_email_unique');`              | 從「users」資料表刪除唯一索引。                         |
| `$table->dropIndex('geo_state_index');`                  | 從「geo」資料表刪除基本索引。                           |
| `$table->dropFullText('posts_body_fulltext');`           | 從「posts」資料表刪除全文索引。                         |
| `$table->dropSpatialIndex('geo_location_spatialindex');` | 從「geo」資料表刪除空間索引 (SQLite 除外)。             |

</div>

如果您將欄位陣列傳遞給刪除索引的方法，則會根據資料表名稱、欄位和索引類型產生約定的索引名稱：

```php
Schema::table('geo', function (Blueprint $table) {
    $table->dropIndex(['state']); // Drops index 'geo_state_index'
});
```

<a name="foreign-key-constraints"></a>
### 外部鍵限制

Laravel 也提供建立外部鍵限制的支援，這些限制用於在資料庫層級強制實施參照完整性。例如，我們可以在 `posts` 資料表上定義一個 `user_id` 欄位，它參照 `users` 資料表上的 `id` 欄位：

```php
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

Schema::table('posts', function (Blueprint $table) {
    $table->unsignedBigInteger('user_id');

    $table->foreign('user_id')->references('id')->on('users');
});
```

由於這種語法較為冗長，Laravel 提供了額外更簡潔的方法，這些方法利用慣例來提供更好的開發者體驗。當使用 `foreignId` 方法建立欄位時，上述範例可以改寫為以下形式：

```php
Schema::table('posts', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained();
});
```

`foreignId` 方法會建立一個與 `UNSIGNED BIGINT` 等效的欄位，而 `constrained` 方法將使用慣例來判斷所參照的資料表和欄位。如果您的資料表名稱與 Laravel 的慣例不符，您可以手動將其提供給 `constrained` 方法。此外，也可以指定應分配給生成索引的名稱：

```php
Schema::table('posts', function (Blueprint $table) {
    $table->foreignId('user_id')->constrained(
        table: 'users', indexName: 'posts_user_id'
    );
});
```

您還可以為限制的「on delete」和「on update」屬性指定所需的動作：

```php
$table->foreignId('user_id')
    ->constrained()
    ->onUpdate('cascade')
    ->onDelete('cascade');
```

也為這些動作提供了另一種表達性語法：

<div class="overflow-auto">

| 方法                            | 說明                                       |
| ----------------------------- | ------------------------------------------------- |
| `$table->cascadeOnUpdate();`  | 更新應級聯。                           |
| `$table->restrictOnUpdate();` | 更新應受限制。                     |
| `$table->nullOnUpdate();`     | 更新應將外部鍵值設為 null。 |
| `$table->noActionOnUpdate();` | 更新時不執行任何動作。                             |
| `$table->cascadeOnDelete();`  | 刪除應級聯。                           |
| `$table->restrictOnDelete();` | 刪除應受限制。                     |
| `$table->nullOnDelete();`     | 刪除應將外部鍵值設為 null。 |
| `$table->noActionOnDelete();` | 若存在子紀錄則阻止刪除。          |

</div>

任何額外的[欄位修改器](#column-modifiers)都必須在 `constrained` 方法之前呼叫：

```php
$table->foreignId('user_id')
    ->nullable()
    ->constrained();
```

<a name="dropping-foreign-keys"></a>
#### 刪除外部鍵

若要刪除外部鍵，您可以使用 `dropForeign` 方法，將要刪除的外部鍵限制名稱作為引數傳入。外部鍵限制使用與索引相同的命名慣例。換句話說，外部鍵限制名稱是基於資料表名稱和限制中的欄位，然後加上「\_foreign」字尾：

```php
$table->dropForeign('posts_user_id_foreign');
```

或者，您可以傳入一個包含外部鍵欄位名稱的陣列給 `dropForeign` 方法。此陣列將使用 Laravel 的限制命名慣例轉換為外部鍵限制名稱：

```php
$table->dropForeign(['user_id']);
```

<a name="toggling-foreign-key-constraints"></a>
#### 切換外部鍵限制

您可以使用以下方法在資料遷移中啟用或禁用外部鍵限制：

```php
Schema::enableForeignKeyConstraints();

Schema::disableForeignKeyConstraints();

Schema::withoutForeignKeyConstraints(function () {
    // Constraints disabled within this closure...
});
```

> [!WARNING]
> SQLite 預設會禁用外部鍵限制。當使用 SQLite 時，請確保在嘗試於資料遷移中建立外部鍵限制之前，於您的資料庫設定中[啟用外部鍵支援](/docs/{{version}}/database#configuration)。

<a name="events"></a>
## 事件

為方便起見，每個資料遷移操作都會觸發一個 [事件](/docs/{{version}}/events)。以下所有事件都繼承了基礎的 `Illuminate\Database\Events\MigrationEvent` 類別：

<div class="overflow-auto">

| 類別                                            | 說明                                       |
| ------------------------------------------------ | ------------------------------------------------ |
| `Illuminate\Database\Events\MigrationsStarted`   | 一批資料遷移即將執行。   |
| `Illuminate\Database\Events\MigrationsEnded`     | 一批資料遷移已執行完畢。    |
| `Illuminate\Database\Events\MigrationStarted`    | 單一資料遷移即將執行。      |
| `Illuminate\Database\Events\MigrationEnded`      | 單一資料遷移已執行完畢。       |
| `Illuminate\Database\Events\NoPendingMigrations` | 資料遷移指令未發現任何待執行遷移。 |
| `Illuminate\Database\Events\SchemaDumped`        | 資料庫結構傾印已完成。            |
| `Illuminate\Database\Events\SchemaLoaded`        | 現有資料庫結構傾印已載入。     |

</div>