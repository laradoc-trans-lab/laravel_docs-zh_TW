# 資料庫：入門

- [介紹](#introduction)
    - [設定](#configuration)
    - [讀寫連線](#read-and-write-connections)
- [執行 SQL 查詢](#running-queries)
    - [使用多重資料庫連線](#using-multiple-database-connections)
    - [監聽查詢事件](#listening-for-query-events)
    - [監控累積查詢時間](#monitoring-cumulative-query-time)
- [資料庫事務](#database-transactions)
- [連線至資料庫 CLI](#connecting-to-the-database-cli)
- [檢查你的資料庫](#inspecting-your-databases)
- [監控你的資料庫](#monitoring-your-databases)

<a name="introduction"></a>
## 介紹

幾乎所有現代網路應用程式都與資料庫互動。Laravel 透過使用原生 SQL、[流暢查詢產生器](/docs/{{version}}/queries) 和 [Eloquent ORM](/docs/{{version}}/eloquent)，讓與多種支援資料庫的互動變得極其簡單。目前，Laravel 官方支援五種資料庫：

<div class="content-list" markdown="1">

- MariaDB 10.3+ ([版本政策](https://mariadb.org/about/#maintenance-policy))
- MySQL 5.7+ ([版本政策](https://en.wikipedia.org/wiki/MySQL#Release_history))
- PostgreSQL 10.0+ ([版本政策](https://www.postgresql.org/support/versioning/))
- SQLite 3.26.0+
- SQL Server 2017+ ([版本政策](https://docs.microsoft.com/en-us/lifecycle/products/?products=sql-server))

</div>

此外，透過 `mongodb/laravel-mongodb` 套件也支援 MongoDB，該套件由 MongoDB 官方維護。請查閱 [Laravel MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/) 文件以獲取更多資訊。

<a name="configuration"></a>
### 設定

Laravel 資料庫服務的設定位於應用程式的 `config/database.php` 設定檔中。在此檔案中，您可以定義所有資料庫連線，並指定預設要使用的連線。此檔案中的大多數設定選項都由應用程式環境變數的值驅動。此檔案中提供了大多數 Laravel 支援的資料庫系統範例。

預設情況下，Laravel 的範例[環境設定](/docs/{{version}}/configuration#environment-configuration)已可與 [Laravel Sail](/docs/{{version}}/sail) 一起使用，Laravel Sail 是一種用於在您的本機電腦上開發 Laravel 應用程式的 Docker 設定。但是，您可以根據本地資料庫的需求自由修改資料庫設定。

<a name="sqlite-configuration"></a>
#### SQLite 設定

SQLite 資料庫儲存在檔案系統中的單一檔案內。您可以使用終端機中的 `touch` 指令建立一個新的 SQLite 資料庫：`touch database/database.sqlite`。資料庫建立後，您可以透過將資料庫的絕對路徑放在 `DB_DATABASE` 環境變數中，輕鬆設定您的環境變數以指向此資料庫：

```ini
DB_CONNECTION=sqlite
DB_DATABASE=/absolute/path/to/database.sqlite
```

預設情況下，SQLite 連線已啟用外部鍵約束。如果您想禁用它們，您應該將 `DB_FOREIGN_KEYS` 環境變數設定為 `false`：

```ini
DB_FOREIGN_KEYS=false
```

> [!NOTE]
> 如果您使用 [Laravel 安裝程式](/docs/{{version}}/installation#creating-a-laravel-project) 建立您的 Laravel 應用程式並選擇 SQLite 作為您的資料庫，Laravel 將會自動建立一個 `database/database.sqlite` 檔案，並為您執行預設的[資料庫遷移](/docs/{{version}}/migrations)。

<a name="mssql-configuration"></a>
#### Microsoft SQL Server 設定

要使用 Microsoft SQL Server 資料庫，您應該確保已安裝 `sqlsrv` 和 `pdo_sqlsrv` PHP 擴充功能，以及它們可能需要的任何依賴項，例如 Microsoft SQL ODBC 驅動程式。

<a name="configuration-using-urls"></a>
#### 使用 URL 設定

通常，資料庫連線是使用多個設定值進行設定的，例如 `host`、`database`、`username`、`password` 等。每個這些設定值都有其對應的環境變數。這表示在生產伺服器上設定資料庫連線資訊時，您需要管理多個環境變數。

一些受管資料庫供應商，例如 AWS 和 Heroku，提供一個單一的資料庫「URL」，其中包含資料庫的所有連線資訊，以單一字串表示。一個範例資料庫 URL 可能如下所示：

```html
mysql://root:password@127.0.0.1/forge?charset=UTF-8
```

這些 URL 通常遵循標準的 Schema 慣例：

```html
driver://username:password@host:port/database?options
```

為了方便，Laravel 支援這些 URL，作為使用多個設定選項來設定資料庫的替代方案。如果 `url` (或對應的 `DB_URL` 環境變數) 設定選項存在，它將用於提取資料庫連線和憑證資訊。

<a name="read-and-write-connections"></a>
### 讀寫連線

有時您可能希望對 SELECT 語句使用一個資料庫連線，而對 INSERT、UPDATE 和 DELETE 語句使用另一個資料庫連線。Laravel 讓這變得輕而易舉，無論您使用的是原生查詢、查詢產生器還是 Eloquent ORM，都將始終使用正確的連線。

要了解如何設定讀寫連線，請看這個範例：

```php
'mysql' => [
    'read' => [
        'host' => [
            '192.168.1.1',
            '196.168.1.2',
        ],
    ],
    'write' => [
        'host' => [
            '196.168.1.3',
        ],
    ],
    'sticky' => true,

    'database' => env('DB_DATABASE', 'laravel'),
    'username' => env('DB_USERNAME', 'root'),
    'password' => env('DB_PASSWORD', ''),
    'unix_socket' => env('DB_SOCKET', ''),
    'charset' => env('DB_CHARSET', 'utf8mb4'),
    'collation' => env('DB_COLLATION', 'utf8mb4_unicode_ci'),
    'prefix' => '',
    'prefix_indexes' => true,
    'strict' => true,
    'engine' => null,
    'options' => extension_loaded('pdo_mysql') ? array_filter([
        PDO::MYSQL_ATTR_SSL_CA => env('MYSQL_ATTR_SSL_CA'),
    ]) : [],
],
```

請注意，設定陣列中新增了三個鍵：`read`、`write` 和 `sticky`。`read` 和 `write` 鍵具有包含單一鍵：`host` 的陣列值。`read` 和 `write` 連線的其他資料庫選項將從主要的 `mysql` 設定陣列中合併。

只有當您希望覆寫主要 `mysql` 陣列中的值時，才需要將項目放入 `read` 和 `write` 陣列中。因此，在本例中，`192.168.1.1` 將用作「讀取」連線的主機，而 `192.168.1.3` 將用作「寫入」連線的主機。資料庫憑證、前綴、字元集以及主要 `mysql` 陣列中的所有其他選項將在兩個連線之間共用。當 `host` 設定陣列中存在多個值時，每次請求都會隨機選擇一個資料庫主機。

<a name="the-sticky-option"></a>
#### `sticky` 選項

`sticky` 選項是一個 *可選* 值，可用於允許立即讀取在當前請求生命週期內已寫入資料庫的記錄。如果 `sticky` 選項已啟用，並且在當前請求生命週期內已對資料庫執行了「寫入」操作，則任何後續的「讀取」操作將使用「寫入」連線。這確保了在請求生命週期內寫入的任何資料可以在同一個請求中立即從資料庫中讀取回來。由您決定這是否是您應用程式所需的行為。

<a name="running-queries"></a>
## 執行 SQL 查詢

一旦你設定好資料庫連線，就可以使用 `DB` Facade 來執行查詢。`DB` Facade 為每種查詢類型提供了方法：`select`、`update`、`insert`、`delete` 和 `statement`。


<a name="running-a-select-query"></a>
#### 執行 Select 查詢

要執行一個基本的 SELECT 查詢，你可以使用 `DB` Facade 上的 `select` 方法：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Support\Facades\DB;
use Illuminate\View\View;

class UserController extends Controller
{
    /**
     * Show a list of all of the application's users.
     */
    public function index(): View
    {
        $users = DB::select('select * from users where active = ?', [1]);

        return view('user.index', ['users' => $users]);
    }
}
```

傳遞給 `select` 方法的第一個引數是 SQL 查詢，而第二個引數是任何需要繫結到查詢的參數繫結。通常，這些是 `where` 子句約束的值。參數繫結提供了防止 SQL 注入的保護。

`select` 方法總是會回傳一個 `array` 結果陣列。陣列中的每個結果都將是一個 PHP `stdClass` 物件，代表資料庫中的一條記錄：

```php
use Illuminate\Support\Facades\DB;

$users = DB::select('select * from users');

foreach ($users as $user) {
    echo $user->name;
}
```


<a name="selecting-scalar-values"></a>
#### 選取純量值

有時你的資料庫查詢可能會產生一個單一的純量值。Laravel 允許你直接使用 `scalar` 方法擷取此值，而不必從記錄物件中擷取查詢的純量結果：

```php
$burgers = DB::scalar(
    "select count(case when food = 'burger' then 1 end) as burgers from menu"
);
```


<a name="selecting-multiple-result-sets"></a>
#### 選取多個結果集

如果你的應用程式呼叫的儲存程序會回傳多個結果集，你可以使用 `selectResultSets` 方法來擷取儲存程序回傳的所有結果集：

```php
[$options, $notifications] = DB::selectResultSets(
    "CALL get_user_options_and_notifications(?)", $request->user()->id
);
```


<a name="using-named-bindings"></a>
#### 使用具名繫結

你可以使用具名繫結來執行查詢，而不是使用 `?` 來表示你的參數繫結：

```php
$results = DB::select('select * from users where id = :id', ['id' => 1]);
```


<a name="running-an-insert-statement"></a>
#### 執行 Insert 陳述式

要執行 `insert` 陳述式，你可以使用 `DB` Facade 上的 `insert` 方法。與 `select` 類似，這個方法將 SQL 查詢作為第一個引數，繫結作為第二個引數：

```php
use Illuminate\Support\Facades\DB;

DB::insert('insert into users (id, name) values (?, ?)', [1, 'Marc']);
```


<a name="running-an-update-statement"></a>
#### 執行 Update 陳述式

`update` 方法應用於更新資料庫中現有的記錄。該方法會回傳受該陳述式影響的列數：

```php
use Illuminate\Support\Facades\DB;

$affected = DB::update(
    'update users set votes = 100 where name = ?',
    ['Anita']
);
```


<a name="running-a-delete-statement"></a>
#### 執行 Delete 陳述式

`delete` 方法應用於從資料庫中刪除記錄。與 `update` 類似，該方法會回傳受影響的列數：

```php
use Illuminate\Support\Facades\DB;

$deleted = DB::delete('delete from users');
```


<a name="running-a-general-statement"></a>
#### 執行一般陳述式

有些資料庫陳述式不回傳任何值。對於這些類型的操作，你可以使用 `DB` Facade 上的 `statement` 方法：

```php
DB::statement('drop table users');
```


<a name="running-an-unprepared-statement"></a>
#### 執行未準備陳述式

有時你可能想要執行一個不繫結任何值的 SQL 陳述式。你可以使用 `DB` Facade 的 `unprepared` 方法來達成此目的：

```php
DB::unprepared('update users set votes = 100 where name = "Dries"');
```

> [!WARNING]
> 由於未準備的陳述式不會繫結參數，因此它們可能容易受到 SQL 注入的影響。你絕不應該在未準備的陳述式中允許使用者控制的值。


<a name="implicit-commits-in-transactions"></a>
#### 隱式提交

在事務中使用 `DB` Facade 的 `statement` 和 `unprepared` 方法時，你必須小心避免會導致[隱式提交](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)的陳述式。這些陳述式會導致資料庫引擎間接提交整個事務，讓 Laravel 不知道資料庫的事務級別。一個此類陳述式的範例是建立資料庫表：

```php
DB::unprepared('create table a (col varchar(1) null)');
```

請參閱 MySQL 手冊，以獲取會觸發[隱式提交](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)的所有陳述式列表。


<a name="using-multiple-database-connections"></a>
### 使用多重資料庫連線

如果你的應用程式在 `config/database.php` 設定檔中定義了多個連線，你可以透過 `DB` Facade 提供的 `connection` 方法來存取每個連線。傳遞給 `connection` 方法的連線名稱應該與 `config/database.php` 設定檔中列出的連線之一相對應，或者使用 `config` 輔助函式在執行時進行設定：

```php
use Illuminate\Support\Facades\DB;

$users = DB::connection('sqlite')->select(/* ... */);
```

你可以使用連線實例上的 `getPdo` 方法來存取連線的原始底層 PDO 實例：

```php
$pdo = DB::connection()->getPdo();
```


<a name="listening-for-query-events"></a>
### 監聽查詢事件

如果你想為應用程式執行的每個 SQL 查詢指定一個閉包，可以使用 `DB` Facade 的 `listen` 方法。這個方法對於記錄查詢或除錯很有用。你可以在[服務提供者](/docs/{{version}}/providers)的 `boot` 方法中註冊你的查詢監聽器閉包：

```php
<?php

namespace App\Providers;

use Illuminate\Database\Events\QueryExecuted;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        // ...
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        DB::listen(function (QueryExecuted $query) {
            // $query->sql;
            // $query->bindings;
            // $query->time;
            // $query->toRawSql();
        });
    }
}
```


<a name="monitoring-cumulative-query-time"></a>
### 監控累積查詢時間

現代網路應用程式常見的效能瓶頸是它們花費在查詢資料庫上的時間。幸運的是，當 Laravel 在單一請求中查詢資料庫花費太多時間時，它可以呼叫你選擇的閉包或回呼。要開始使用，請向 `whenQueryingForLongerThan` 方法提供一個查詢時間閾值（以毫秒為單位）和一個閉包。你可以在[服務提供者](/docs/{{version}}/providers)的 `boot` 方法中呼叫此方法：

```php
<?php

namespace App\Providers;

use Illuminate\Database\Connection;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\ServiceProvider;
use Illuminate\Database\Events\QueryExecuted;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     */
    public function register(): void
    {
        // ...
    }

    /**
     * Bootstrap any application services.
     */
    public function boot(): void
    {
        DB::whenQueryingForLongerThan(500, function (Connection $connection, QueryExecuted $event) {
            // Notify development team...
        });
    }
}
```

<a name="database-transactions"></a>
## 資料庫事務

你可以使用 `DB` Facade 所提供的 `transaction` 方法，在資料庫事務中執行一系列操作。如果在事務閉包中拋出異常，該事務將自動回滾，並重新拋出異常。如果閉包成功執行，該事務將自動提交。使用 `transaction` 方法時，你無需擔心手動回滾或提交。

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    DB::update('update users set votes = 1');

    DB::delete('delete from posts');
});
```

<a name="handling-deadlocks"></a>
#### 處理死鎖

`transaction` 方法接受一個可選的第二個參數，用於定義當發生死鎖時，事務應該重試的次數。一旦這些嘗試用盡，將會拋出異常：

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    DB::update('update users set votes = 1');

    DB::delete('delete from posts');
}, 5);
```

<a name="manually-using-transactions"></a>
#### 手動使用事務

如果你想手動開始一個事務，並完全控制回滾與提交，你可以使用 `DB` Facade 所提供的 `beginTransaction` 方法：

```php
use Illuminate\Support\Facades\DB;

DB::beginTransaction();
```

你可以透過 `rollBack` 方法回滾事務：

```php
DB::rollBack();
```

最後，你可以透過 `commit` 方法提交事務：

```php
DB::commit();
```

> [!NOTE]
> `DB` Facade 的事務方法控制著[查詢建構器](/docs/{{version}}/queries)與 [Eloquent ORM](/docs/{{version}}/eloquent) 的事務。

<a name="connecting-to-the-database-cli"></a>
## 連線至資料庫 CLI

如果你想連線到資料庫的 CLI，你可以使用 `db` Artisan 命令：

```shell
php artisan db
```

如有需要，你可以指定一個資料庫連線名稱，以連線到非預設的資料庫連線：

```shell
php artisan db mysql
```

<a name="inspecting-your-databases"></a>
## 檢查你的資料庫

使用 `db:show` 和 `db:table` Artisan 命令，你可以深入了解你的資料庫及其相關表格。若要查看資料庫的概述，包括其大小、類型、開啟連線數以及表格摘要，你可以使用 `db:show` 命令：

```shell
php artisan db:show
```

你可以透過 `--database` 選項向命令提供資料庫連線名稱，以指定應檢查哪個資料庫連線：

```shell
php artisan db:show --database=pgsql
```

如果你想在命令輸出中包含表格列計數和資料庫視圖詳細資訊，可以分別提供 `--counts` 和 `--views` 選項。在大型資料庫上，檢索列計數和視圖詳細資訊可能會很慢：

```shell
php artisan db:show --counts --views
```

此外，你可以使用以下 `Schema` 方法來檢查你的資料庫：

```php
use Illuminate\Support\Facades\Schema;

$tables = Schema::getTables();
$views = Schema::getViews();
$columns = Schema::getColumns('users');
$indexes = Schema::getIndexes('users');
$foreignKeys = Schema::getForeignKeys('users');
```

如果你想檢查非應用程式預設連線的資料庫連線，你可以使用 `connection` 方法：

```php
$columns = Schema::connection('sqlite')->getColumns('users');
```

<a name="table-overview"></a>
#### 表格概述

如果你想取得資料庫中單個表格的概述，你可以執行 `db:table` Artisan 命令。此命令提供資料庫表格的概覽，包括其欄位、類型、屬性、鍵和索引：

```shell
php artisan db:table users
```

<a name="monitoring-your-databases"></a>
## 監控你的資料庫

使用 `db:monitor` Artisan 命令，你可以指示 Laravel 在資料庫管理的開啟連線數超過指定數量時，發送一個 `Illuminate\Database\Events\DatabaseBusy` 事件。

首先，你應該將 `db:monitor` 命令[排程為每分鐘執行一次](/docs/{{version}}/scheduling)。該命令接受你希望監控的資料庫連線設定名稱，以及在發送事件之前應容忍的最多開啟連線數：

```shell
php artisan db:monitor --databases=mysql,pgsql --max=100
```

單獨排程此命令並不足以觸發通知來提醒你開啟連線數。當命令遇到開啟連線數超過閾值的資料庫時，將會發送 `DatabaseBusy` 事件。你應該在應用程式的 `AppServiceProvider` 中監聽此事件，以便向你或你的開發團隊發送通知：

```php
use App\Notifications\DatabaseApproachingMaxConnections;
use Illuminate\Database\Events\DatabaseBusy;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Notification;

/**
 * Bootstrap any application services.
 */
public function boot(): void
{
    Event::listen(function (DatabaseBusy $event) {
        Notification::route('mail', 'dev@example.com')
            ->notify(new DatabaseApproachingMaxConnections(
                $event->connectionName,
                $event->connections
            ));
    });
}
```