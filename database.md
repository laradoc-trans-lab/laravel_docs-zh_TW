# 資料庫：入門指南

- [簡介](#introduction)
    - [設定](#configuration)
    - [讀取與寫入連接](#read-and-write-connections)
- [執行 SQL 查詢](#running-queries)
    - [使用多個資料庫連接](#using-multiple-database-connections)
    - [監聽查詢事件](#listening-for-query-events)
    - [監控累計查詢時間](#monitoring-cumulative-query-time)
- [資料庫事務](#database-transactions)
- [連接到資料庫 CLI](#connecting-to-the-database-cli)
- [檢查您的資料庫](#inspecting-your-databases)
- [監控您的資料庫](#monitoring-your-databases)

<a name="introduction"></a>
## 簡介

幾乎每個現代的 Web 應用程式都會與資料庫進行互動。Laravel 使用原始 SQL、[流暢的查詢產生器 (Query Builder)](/docs/{{version}}/queries) 以及 [Eloquent ORM](/docs/{{version}}/eloquent)，讓在各種支援的資料庫之間進行互動變得非常簡單。目前，Laravel 為五種資料庫提供官方支援：

<div class="content-list" markdown="1">

- MariaDB 10.3+ ([版本政策](https://mariadb.org/about/#maintenance-policy))
- MySQL 5.7+ ([版本政策](https://en.wikipedia.org/wiki/MySQL#Release_history))
- PostgreSQL 10.0+ ([版本政策](https://www.postgresql.org/support/versioning/))
- SQLite 3.26.0+
- SQL Server 2017+ ([版本政策](https://docs.microsoft.com/en-us/lifecycle/products/?products=sql-server))

</div>

此外，MongoDB 透過 `mongodb/laravel-mongodb` 套件提供支援，該套件由 MongoDB 官方維護。請參閱 [Laravel MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/) 文件以取得更多資訊。


<a name="configuration"></a>
### 設定

Laravel 資料庫服務的設定位於應用程式的 `config/database.php` 設定檔中。在此檔案中，您可以定義所有的資料庫連接，並指定預設使用的連接。此檔案中的大多數設定選項都由應用程式環境變數的值驅動。此檔案中提供了大多數 Laravel 支援的資料庫系統範例。

預設情況下，Laravel 的範例[環境設定](/docs/{{version}}/configuration#environment-configuration)已準備好與 [Laravel Sail](/docs/{{version}}/sail) 搭配使用，這是一個用於在本地機器上開發 Laravel 應用程式的 Docker 設定。不過，您可以根據本地資料庫的需要自由修改資料庫設定。


<a name="sqlite-configuration"></a>
#### SQLite 設定

SQLite 資料庫包含在檔案系統中的單個檔案內。您可以使用終端機中的 `touch` 指令建立新的 SQLite 資料庫：`touch database/database.sqlite`。建立資料庫後，您可以透過在 `DB_DATABASE` 環境變數中放置資料庫的絕對路徑，輕鬆地將環境變數設定為指向此資料庫：

```ini
DB_CONNECTION=sqlite
DB_DATABASE=/absolute/path/to/database.sqlite
```

預設情況下，SQLite 連接會啟動外鍵約束。如果您想停用它們，應將 `DB_FOREIGN_KEYS` 環境變數設定為 `false`：

```ini
DB_FOREIGN_KEYS=false
```

> [!NOTE]
> 如果您使用 [Laravel 安裝程式](/docs/{{version}}/installation#creating-a-laravel-project)建立 Laravel 應用程式並選擇 SQLite 作為資料庫，Laravel 將自動為您建立 `database/database.sqlite` 檔案並執行預設的[資料庫遷移 (Migrations)](/docs/{{version}}/migrations)。


<a name="mssql-configuration"></a>
#### Microsoft SQL Server 設定

要使用 Microsoft SQL Server 資料庫，您應該確保已安裝 `sqlsrv` 和 `pdo_sqlsrv` PHP 擴充功能，以及它們可能需要的任何依賴項，例如 Microsoft SQL ODBC 驅動程式。


<a name="configuration-using-urls"></a>
#### 使用 URL 進行設定

通常，資料庫連接是使用多個設定值（如 `host`、`database`、`username`、`password` 等）來設定的。這些設定值中的每一個都有其對應的環境變數。這意味著在正式環境伺服器上設定資料庫連接資訊時，您需要管理多個環境變數。

一些受管理的資料庫提供商（如 AWS 和 Heroku）提供單個資料庫 "URL"，其中包含單個字串中資料庫的所有連接資訊。範例資料庫 URL 可能如下所示：

```html
mysql://root:password@127.0.0.1/forge?charset=UTF-8
```

這些 URL 通常遵循標準的 Schema 慣例：

```html
driver://username:password@host:port/database?options
```

為了方便起見，Laravel 支援這些 URL 作為使用多個設定選項來設定資料庫的另一種選擇。如果存在 `url`（或對應的 `DB_URL` 環境變數）設定選項，它將被用於提取資料庫連接和憑證資訊。


<a name="read-and-write-connections"></a>
### 讀取與寫入連接

有時您可能希望將一個資料庫連接用於 SELECT 語句，而將另一個連接用於 INSERT、UPDATE 和 DELETE 語句。Laravel 讓這件事變得輕而易舉，無論您是使用原始查詢、查詢產生器還是 Eloquent ORM，都會始終使用正確的連接。

要了解應該如何設定讀取 / 寫入連接，讓我們看看這個範例：

```php
'mysql' => [
    'driver' => 'mysql',
    
    'read' => [
        'host' => [
            '192.168.1.1',
            '196.168.1.2',
        ],
    ],
    'write' => [
        'host' => [
            '192.168.1.3',
        ],
    ],
    'sticky' => true,
    
    'port' => env('DB_PORT', '3306'),
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
        (PHP_VERSION_ID >= 80500 ? \Pdo\Mysql::ATTR_SSL_CA : \PDO::MYSQL_ATTR_SSL_CA) => env('MYSQL_ATTR_SSL_CA'),
    ]) : [],
],
```

請注意，設定陣列中添加了三個鍵：`read`、`write` 和 `sticky`。`read` 和 `write` 鍵的陣列值包含單個鍵：`host`。`read` 和 `write` 連接的其他資料庫選項將從主要的 `mysql` 設定陣列中合併。

如果您希望覆蓋主要 `mysql` 陣列中的值，則只需將項目放入 `read` 和 `write` 陣列中即可。因此，在這種情況下，`192.168.1.1` 將用作「讀取」連接的主機，而 `192.168.1.3` 將用於「寫入」連接。資料庫憑證、前綴、字元集以及主要 `mysql` 陣列中的所有其他選項將在兩個連接之間共享。當 `host` 設定陣列中存在多個值時，將為每個請求隨機選擇一個資料庫主機。


<a name="the-sticky-option"></a>
#### `sticky` 選項

`sticky` 選項是一個*選填*值，可用於允許立即讀取在當前請求週期內已寫入資料庫的紀錄。如果啟用了 `sticky` 選項，並且在當前請求週期內對資料庫執行了「寫入」操作，則任何進一步的「讀取」操作都將使用「寫入」連接。這可確保在請求週期內寫入的任何資料都可以在同一個請求期間立即從資料庫中讀回。您可以自行決定這是否是您應用程式所需的行為。

<a name="running-queries"></a>
## 執行 SQL 查詢

一旦設定好資料庫連接，您就可以使用 `DB` Facade 來執行查詢。`DB` Facade 為每種類型的查詢提供了對應的方法：`select`、`update`、`insert`、`delete` 以及 `statement`。


<a name="running-a-select-query"></a>
#### 執行 Select 查詢

要執行一個基本的 SELECT 查詢，您可以使用 `DB` Facade 上的 `select` 方法：

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

傳遞給 `select` 方法的第一個參數是 SQL 查詢語句，而第二個參數則是任何需要綁定到查詢中的參數。通常，這些是 `where` 子句的約束值。參數綁定提供了防止 SQL 插入攻擊的保護。

`select` 方法總是會回傳一個結果 `array`。陣列中的每個結果都會是一個 PHP `stdClass` 物件，用來代表資料庫中的一筆紀錄：

```php
use Illuminate\Support\Facades\DB;

$users = DB::select('select * from users');

foreach ($users as $user) {
    echo $user->name;
}
```


<a name="selecting-scalar-values"></a>
#### 取得純量值

有時您的資料庫查詢可能只會得到單一的純量值。Laravel 允許您直接使用 `scalar` 方法取得此值，而不需要從紀錄物件中提取查詢的純量結果：

```php
$burgers = DB::scalar(
    "select count(case when food = 'burger' then 1 end) as burgers from menu"
);
```


<a name="selecting-multiple-result-sets"></a>
#### 取得多個結果集

如果您的應用程式呼叫了會回傳多個結果集的儲存程序，您可以使用 `selectResultSets` 方法來取得該儲存程序回傳的所有結果集：

```php
[$options, $notifications] = DB::selectResultSets(
    "CALL get_user_options_and_notifications(?)", $request->user()->id
);
```


<a name="using-named-bindings"></a>
#### 使用命名綁定

除了使用 `?` 來代表您的參數綁定外，您也可以使用具名綁定來執行查詢：

```php
$results = DB::select('select * from users where id = :id', ['id' => 1]);
```


<a name="running-an-insert-statement"></a>
#### 執行 Insert 陳述式

若要執行一個 `insert` 陳述式，您可以使用 `DB` Facade 上的 `insert` 方法。與 `select` 相同，此方法接受 SQL 查詢作為第一個參數，並將綁定作為第二個參數：

```php
use Illuminate\Support\Facades\DB;

DB::insert('insert into users (id, name) values (?, ?)', [1, 'Marc']);
```


<a name="running-an-update-statement"></a>
#### 執行 Update 陳述式

`update` 方法用於更新資料庫中現有的紀錄。該方法會回傳受該陳述式影響的行數：

```php
use Illuminate\Support\Facades\DB;

$affected = DB::update(
    'update users set votes = 100 where name = ?',
    ['Anita']
);
```


<a name="running-a-delete-statement"></a>
#### 執行 Delete 陳述式

`delete` 方法用於從資料庫中刪除紀錄。與 `update` 一樣，該方法會回傳受影響的行數：

```php
use Illuminate\Support\Facades\DB;

$deleted = DB::delete('delete from users');
```


<a name="running-a-general-statement"></a>
#### 執行一般陳述式

某些資料庫陳述式不會回傳任何值。對於這類操作，您可以使用 `DB` Facade 上的 `statement` 方法：

```php
DB::statement('drop table users');
```


<a name="running-an-unprepared-statement"></a>
#### 執行未經預處理的陳述式

有時您可能想在不綁定任何值的情況下執行 SQL 陳述式。您可以使用 `DB` Facade 的 `unprepared` 方法來達成此目的：

```php
DB::unprepared('update users set votes = 100 where name = "Dries"');
```

> [!WARNING]
> 由於未經預處理的陳述式不綁定參數，它們可能容易受到 SQL 插入攻擊。您絕不應該在未經預處理的陳述式中允許使用者控制的值。


<a name="implicit-commits-in-transactions"></a>
#### 隱含提交

在交易中使用 `DB` Facade 的 `statement` 與 `unprepared` 方法時，必須小心避免會導致 [隱含提交](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html) 的陳述式。這些陳述式會導致資料庫引擎間接提交整個交易，使 Laravel 無法察覺資料庫的交易層級。建立資料庫表就是這類陳述式的一個例子：

```php
DB::unprepared('create table a (col varchar(1) null)');
```

請參考 MySQL 手冊以取得會觸發隱含提交的 [所有陳述式清單](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)。


<a name="using-multiple-database-connections"></a>
### 使用多個資料庫連接

如果您的應用程式在 `config/database.php` 設定檔中定義了多個連接，您可以透過 `DB` Facade 提供的 `connection` 方法來存取每個連接。傳遞給 `connection` 方法的連接名稱應對應於 `config/database.php` 設定檔中列出的連接之一，或是在執行期間使用 `config` 輔助函式設定的連接：

```php
use Illuminate\Support\Facades\DB;

$users = DB::connection('sqlite')->select(/* ... */);
```

您可以使用連接實例上的 `getPdo` 方法來存取底層原始的 PDO 實例：

```php
$pdo = DB::connection()->getPdo();
```


<a name="listening-for-query-events"></a>
### 監聽查詢事件

如果您想指定一個在應用程式執行每個 SQL 查詢時都會被呼叫的閉包，您可以使用 `DB` Facade 的 `listen` 方法。這個方法對於記錄查詢或除錯非常有用。您可以在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中註冊您的查詢監聽器閉包：

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
### 監控累計查詢時間

現代網頁應用程式常見的效能瓶頸在於查詢資料庫所花費的時間。幸運的是，當 Laravel 在單次請求中花費過多時間查詢資料庫時，它可以呼叫您指定的閉包或回呼。要開始使用，請向 `whenQueryingForLongerThan` 方法提供查詢時間閾值（以毫秒為單位）以及閉包。您可以在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

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

您可以使用 `DB` Facade 提供的 `transaction` 方法，在資料庫事務中執行一組操作。如果在事務閉包 (Closure) 內拋出異常，事務將自動回滾 (Roll Back) 且異常會被重新拋出。如果閉包執行成功，事務將自動提交 (Commit)。使用 `transaction` 方法時，您不需要擔心手動回滾或提交：

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    DB::update('update users set votes = 1');

    DB::delete('delete from posts');
});
```


<a name="handling-deadlocks"></a>
#### 處理死結

`transaction` 方法接受一個可選的第二個參數，用於定義發生死結 (Deadlock) 時應重試事務的次數。一旦耗盡這些嘗試次數，將會拋出異常：

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    DB::update('update users set votes = 1');

    DB::delete('delete from posts');
}, attempts: 5);
```


<a name="manually-using-transactions"></a>
#### 手動使用事務

如果您想手動開始事務並完全控制回滾與提交，可以使用 `DB` Facade 提供的 `beginTransaction` 方法：

```php
use Illuminate\Support\Facades\DB;

DB::beginTransaction();
```

您可以透過 `rollBack` 方法回滾事務：

```php
DB::rollBack();
```

最後，您可以透過 `commit` 方法提交事務：

```php
DB::commit();
```

> [!NOTE]
> `DB` Facade 的事務方法同時控制了 [查詢產生器 (Query Builder)](/docs/{{version}}/queries) 與 [Eloquent ORM](/docs/{{version}}/eloquent) 的事務。


<a name="connecting-to-the-database-cli"></a>
## 連接到資料庫 CLI

如果您想連接到資料庫的 CLI，可以使用 `db` Artisan 指令：

```shell
php artisan db
```

如果需要，您可以指定資料庫連接名稱，以連接到非預設的資料庫連接：

```shell
php artisan db mysql
```


<a name="inspecting-your-databases"></a>
## 檢查您的資料庫

使用 `db:show` 與 `db:table` Artisan 指令，您可以深入了解資料庫及其相關資料表。若要查看資料庫的概覽，包括其大小、類型、開啟的連接數以及資料表摘要，可以使用 `db:show` 指令：

```shell
php artisan db:show
```

您可以透過 `--database` 選項提供資料庫連接名稱，來指定要檢查的資料庫連接：

```shell
php artisan db:show --database=pgsql
```

如果您希望在指令輸出中包含資料表資料筆數與資料庫視圖 (View) 的詳細資訊，可以分別提供 `--counts` 與 `--views` 選項。在大型資料庫上，檢索資料筆數與視圖詳細資訊可能會很慢：

```shell
php artisan db:show --counts --views
```

此外，您可以使用以下 `Schema` 方法來檢查您的資料庫：

```php
use Illuminate\Support\Facades\Schema;

$tables = Schema::getTables();
$views = Schema::getViews();
$columns = Schema::getColumns('users');
$indexes = Schema::getIndexes('users');
$foreignKeys = Schema::getForeignKeys('users');
```

如果您想檢查非應用程式預設連接的資料庫連接，可以使用 `connection` 方法：

```php
$columns = Schema::connection('sqlite')->getColumns('users');
```


<a name="table-overview"></a>
#### 資料表概覽

如果您想查看資料庫中單個資料表的概覽，可以執行 `db:table` Artisan 指令。此指令提供了資料表的通用概覽，包括其欄位、類型、屬性、鍵 (Key) 與索引 (Index)：

```shell
php artisan db:table users
```


<a name="monitoring-your-databases"></a>
## 監控您的資料庫

使用 `db:monitor` Artisan 指令，您可以指示 Laravel 在資料庫管理的開啟連接數超過指定數量時發送 `Illuminate\Database\Events\DatabaseBusy` 事件。

首先，您應該排程 `db:monitor` 指令[每分鐘執行一次](/docs/{{version}}/scheduling)。該指令接受您希望監控的資料庫連接設定名稱，以及在發送事件前可容忍的最大開啟連接數：

```shell
php artisan db:monitor --databases=mysql,pgsql --max=100
```

單獨排程此指令不足以觸發提醒您開啟連接數的通知。當指令遇到開啟連接數超過閾值的資料庫時，將發送 `DatabaseBusy` 事件。您應該在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊發送通知：

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