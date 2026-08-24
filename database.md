# 資料庫：入門指南

- [介紹](#introduction)
    - [設定](#configuration)
    - [讀取與寫入連線](#read-and-write-connections)
    - [池化 PostgreSQL 連線](#pooled-postgresql-connections)
- [執行 SQL 查詢](#running-queries)
    - [使用多個資料庫連線](#using-multiple-database-connections)
    - [監聽查詢事件](#listening-for-query-events)
    - [監控累計查詢時間](#monitoring-cumulative-query-time)
- [資料庫交易](#database-transactions)
- [連線至資料庫 CLI](#connecting-to-the-database-cli)
- [檢視你的資料庫](#inspecting-your-databases)
- [監控你的資料庫](#monitoring-your-databases)

<a name="introduction"></a>
## 介紹

幾乎所有現代 Web 應用程式都會與資料庫進行互動。Laravel 透過原生 SQL、[流暢的查詢建構器](/docs/{{version}}/queries) 以及 [Eloquent ORM](/docs/{{version}}/eloquent)，讓與各種支援的資料庫進行互動變得極其簡單。目前，Laravel 為五種資料庫提供第一方支援：

<div class="content-list" markdown="1">

- MariaDB 10.3+ ([版本政策](https://mariadb.org/about/#maintenance-policy))
- MySQL 5.7+ ([版本政策](https://en.wikipedia.org/wiki/MySQL#Release_history))
- PostgreSQL 10.0+ ([版本政策](https://www.postgresql.org/support/versioning/))
- SQLite 3.26.0+
- SQL Server 2017+ ([版本政策](https://docs.microsoft.com/en-us/lifecycle/products/?products=sql-server))

</div>

此外，還能透過由 MongoDB 官方維護的 `mongodb/laravel-mongodb` 套件支援 MongoDB。請參閱 [Laravel MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/) 文件以獲取更多資訊。


<a name="configuration"></a>
### 設定

Laravel 資料庫服務的設定檔位於應用程式的 `config/database.php`。在此檔案中，你可以定義所有的資料庫連線，並指定預設應該使用哪一個連線。該檔案中的大部分設定選項都是由應用程式的環境變數值所驅動。此檔案中提供了 Laravel 支援的大多數資料庫系統的範例。

預設情況下，Laravel 的範例[環境設定](/docs/{{version}}/configuration#environment-configuration)已預先配置好，可直接搭配 [Laravel Sail](/docs/{{version}}/sail) 使用，Sail 是一個用於在本地端機器上開發 Laravel 應用程式的 Docker 設定。不過，你可以根據本地端資料庫的需求隨意修改資料庫設定。


<a name="sqlite-configuration"></a>
#### SQLite 設定

SQLite 資料庫包含在檔案系統中的單一檔案內。你可以在終端機中使用 `touch` 指令建立一個新的 SQLite 資料庫：`touch database/database.sqlite`。建立資料庫後，你可以透過在 `DB_DATABASE` 環境變數中放置資料庫的絕對路徑，輕鬆設定環境變數以指向此資料庫：

```ini
DB_CONNECTION=sqlite
DB_DATABASE=/absolute/path/to/database.sqlite
```

預設情況下，SQLite 連線會啟用外鍵約束 (Foreign Key Constraints)。如果你想要停用它們，應將 `DB_FOREIGN_KEYS` 環境變數設定為 `false`：

```ini
DB_FOREIGN_KEYS=false
```

> [!NOTE]
> 如果你使用 [Laravel 安裝程式](/docs/{{version}}/installation#creating-a-laravel-project)建立 Laravel 應用程式並選擇 SQLite 作為資料庫，Laravel 將會自動建立 `database/database.sqlite` 檔案並為你執行預設的[資料庫遷移](/docs/{{version}}/migrations)。


<a name="mssql-configuration"></a>
#### Microsoft SQL Server 設定

若要使用 Microsoft SQL Server 資料庫，你應確保已安裝 `sqlsrv` 和 `pdo_sqlsrv` PHP 擴充套件，以及它們可能需要的任何依賴套件（例如 Microsoft SQL ODBC 驅動程式）。


<a name="configuration-using-urls"></a>
#### 使用 URL 進行設定

通常，資料庫連線會使用多個設定值進行設定，例如 `host`、`database`、`username`、`password` 等。這些設定值中的每一個都有其對應的環境變數。這意味著在正式環境伺服器上設定資料庫連線資訊時，你需要管理數個環境變數。

某些受管資料庫供應商（例如 AWS 和 Heroku）提供單一資料庫「URL」，它在單一字串中包含了資料庫的所有連線資訊。範例資料庫 URL 可能類似於以下內容：

```html
mysql://root:password@127.0.0.1/forge?charset=UTF-8
```

這些 URL 通常遵循標準的架構慣例：

```html
driver://username:password@host:port/database?options
```

為了方便起見，Laravel 支援這些 URL 作為使用多個設定選項來設定資料庫的替代方案。如果存在 `url`（或對應的 `DB_URL` 環境變數）設定選項，它將用於擷取資料庫連線和憑證資訊。


<a name="read-and-write-connections"></a>
### 讀取與寫入連線

有時你可能希望將一個資料庫連線用於 SELECT 語句，並將另一個資料庫連線用於 INSERT、UPDATE 和 DELETE 語句。Laravel 讓這變得輕而易舉，無論你使用的是原始查詢、查詢建構器還是 Eloquent ORM，都將始終使用適當的連線。

若要了解應該如何設定讀取／寫入連線，讓我們看看這個範例：

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

請注意，設定陣列中已新增了三個鍵：`read`、`write` 和 `sticky`。`read` 和 `write` 鍵的陣列值包含單一個鍵：`host`。`read` 和 `write` 連線的其餘資料庫選項將從主要 `mysql` 設定陣列中合併。

你只需要在 `read` 和 `write` 陣列中放置項目，以覆蓋主要 `mysql` 陣列中的值。因此，在此情況下，`192.168.1.1` 將用作「讀取」連線的主機，而 `192.168.1.3` 將用作「寫入」連線的主機。主要 `mysql` 陣列中的資料庫憑證、前綴、字元集和所有其他選項將在兩個連線之間共用。當 `host` 設定陣列中存在多個值時，將為每個請求隨機選擇一個資料庫主機。


<a name="the-sticky-option"></a>
#### `sticky` 選項

`sticky` 選項是一個*可選的*值，可用於允許立即讀取在當前請求週期內已寫入資料庫的記錄。如果啟用了 `sticky` 選項，且在當前請求週期內對資料庫執行了「寫入」操作，則任何後續的「讀取」操作都將使用「寫入」連線。這可確保在該請求週期內寫入的任何資料都可以在該同一個請求期間立即從資料庫中讀回。你可以自行決定這是否是應用程式所需的行為。

<a name="pooled-postgresql-connections"></a>
### 池化 PostgreSQL 連線

許多託管式 PostgreSQL 提供商透過 PgBouncer 或連線代理等服務提供交易模式的連線池。這些連線池非常適合應用程式查詢，但某些結構定義（schema）操作、遷移以及維護指令則需要直接與資料庫連線。

若要在 PostgreSQL 中使用交易連線池，請照常設定池化連線，並透過 `direct` 設定選項提供直接連線的詳細資訊：

```php
'pgsql' => [
    'driver' => 'pgsql',
    // ...
    'pooled' => env('DB_POOLED', false),
    'direct' => array_filter([
        'host' => env('DB_DIRECT_HOST'),
        'port' => env('DB_DIRECT_PORT'),
        'username' => env('DB_DIRECT_USERNAME'),
        'password' => env('DB_DIRECT_PASSWORD'),
        'sslmode' => env('DB_DIRECT_SSLMODE'),
    ]),
],
```

當 PostgreSQL 連線設定為池化時，Laravel 會自動為池化連線啟用模擬預編譯語句（emulated prepares）。直接連線則會繼承未在 `direct` 設定中明確定義的任何選項，並預設使用原生預編譯語句（native prepares）。

Laravel 會自動將直接連線用於遷移、結構定義傾印與還原、`db:wipe`、`db:show` 以及 `db:table`。當啟用池化模式且已設定直接連線時，`db` 指令預設也會使用直接連線；你可以傳入 `--pooled` 選項以改為連線至池化連線：

```shell
php artisan db --pooled
```

如果你需要在應用程式中明確使用直接連線，請在連線名稱後方加上 `::direct` 後綴：

```php
DB::connection('pgsql::direct')->statement('create extension if not exists "uuid-ossp"');
```

<a name="running-queries"></a>
## 執行 SQL 查詢

當你設定好資料庫連線後，就可以使用 `DB` Facade 執行查詢。`DB` Facade 為每種類型的查詢都提供了對應的方法：`select`、`update`、`insert`、`delete` 以及 `statement`。

<a name="running-a-select-query"></a>
#### 執行 Select 查詢

若要執行基本的 SELECT 查詢，可以使用 `DB` Facade 上的 `select` 方法：

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

傳遞給 `select` 方法的第一個引數是 SQL 查詢語句，而第二個引數則是需要綁定到查詢中的任何參數綁定值。通常這些是 `where` 子句條件的值。參數綁定提供了防範 SQL 注入攻擊（SQL Injection）的保護機制。

`select` 方法總是會回傳一個結果 `array`。陣列中的每個結果都會是一個代表資料庫紀錄的 PHP `stdClass` 物件：

```php
use Illuminate\Support\Facades\DB;

$users = DB::select('select * from users');

foreach ($users as $user) {
    echo $user->name;
}
```

<a name="selecting-scalar-values"></a>
#### 取得純量值

有時你的資料庫查詢可能會回傳單一純量值（Scalar Value）。Laravel 允許你直接使用 `scalar` 方法取得此值，而不需要從紀錄物件中取得查詢的純量結果：

```php
$burgers = DB::scalar(
    "select count(case when food = 'burger' then 1 end) as burgers from menu"
);
```

<a name="selecting-multiple-result-sets"></a>
#### 取得多個結果集

如果你的應用程式呼叫了回傳多個結果集的預存程序（Stored Procedures），可以使用 `selectResultSets` 方法來取得該預存程序回傳的所有結果集：

```php
[$options, $notifications] = DB::selectResultSets(
    "CALL get_user_options_and_notifications(?)", $request->user()->id
);
```

<a name="using-named-bindings"></a>
#### 使用具名綁定

除了使用 `?` 來表示參數綁定之外，你也可以使用具名綁定來執行查詢：

```php
$results = DB::select('select * from users where id = :id', ['id' => 1]);
```

<a name="running-an-insert-statement"></a>
#### 執行 Insert 語句

若要執行 `insert` 語句，可以使用 `DB` Facade 上的 `insert` 方法。如同 `select` 一樣，此方法的第一個引數接受 SQL 查詢，第二個引數接受綁定值：

```php
use Illuminate\Support\Facades\DB;

DB::insert('insert into users (id, name) values (?, ?)', [1, 'Marc']);
```

<a name="running-an-update-statement"></a>
#### 執行 Update 語句

`update` 方法應該用於更新資料庫中現有的紀錄。該語句影響的行數將由該方法回傳：

```php
use Illuminate\Support\Facades\DB;

$affected = DB::update(
    'update users set votes = 100 where name = ?',
    ['Anita']
);
```

<a name="running-a-delete-statement"></a>
#### 執行 Delete 語句

`delete` 方法應該用於刪除資料庫中的紀錄。與 `update` 一樣，該方法將回傳受影響的行數：

```php
use Illuminate\Support\Facades\DB;

$deleted = DB::delete('delete from users');
```

<a name="running-a-general-statement"></a>
#### 執行一般語句

某些資料庫語句不會回傳任何值。對於這種類型的操作，可以使用 `DB` Facade 上的 `statement` 方法：

```php
DB::statement('drop table users');
```

<a name="running-an-unprepared-statement"></a>
#### 執行未預備語句

有時你可能會想在不綁定任何值的情況下執行 SQL 語句。你可以使用 `DB` Facade 的 `unprepared` 方法來完成此操作：

```php
DB::unprepared('update users set votes = 100 where name = "Dries"');
```

> [!WARNING]
> 由於未預備語句（Unprepared Statements）不綁定參數，因此它們可能容易遭受 SQL 注入攻擊。你絕對不應該在未預備語句中允許使用者控制的值。

<a name="implicit-commits-in-transactions"></a>
#### 隱式提交

在交易中使用 `DB` Facade 的 `statement` 和 `unprepared` 方法時，你必須小心避免使用會導致[隱式提交 (Implicit Commits)](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html) 的語句。這些語句會導致資料庫引擎間接提交整個交易，使 Laravel 無法掌握資料庫的交易層級。這類語句的一個範例就是建立資料庫資料表：

```php
DB::unprepared('create table a (col varchar(1) null)');
```

請參考 MySQL 手冊以獲取觸發隱式提交的[所有語句清單](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)。

<a name="using-multiple-database-connections"></a>
### 使用多個資料庫連線

若你的應用程式在 `config/database.php` 設定檔中定義了多個連線，你可以透過 `DB` Facade 提供的 `connection` 方法存取每個連線。傳遞給 `connection` 方法的連線名稱應對應至 `config/database.php` 設定檔中列出的其中一個連線，或者在執行時期使用 `config` 輔助函式設定的連線：

```php
use Illuminate\Support\Facades\DB;

$users = DB::connection('sqlite')->select(/* ... */);
```

你可以使用連線實例上的 `getPdo` 方法來存取連線底層原始的 PDO 實例：

```php
$pdo = DB::connection()->getPdo();
```

<a name="listening-for-query-events"></a>
### 監聽查詢事件

如果你想要指定一個閉包，在應用程式執行每個 SQL 查詢時被調用，可以使用 `DB` Facade 的 `listen` 方法。這個方法對於記錄查詢日誌或除錯非常有用。你可以在[服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中註冊查詢監聽器閉包：

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

現代 Web 應用程式常見的效能瓶頸，就是耗費在查詢資料庫上的時間。值得慶幸的是，當 Laravel 在單次請求中花費過多時間查詢資料庫時，它可以調用你指定的閉包或回呼函式。要開始使用此功能，請向 `whenQueryingForLongerThan` 方法提供查詢時間門檻值（以毫秒為單位）以及閉包。你可以在[服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中調用此方法：

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
## 資料庫交易

你可以使用 `DB` Facade 提供的 `transaction` 方法，在資料庫交易中執行一組操作。如果在交易閉包內拋出例外，交易將會自動復原 (Rollback) 並重新拋出該例外。如果閉包成功執行，交易將會自動提交 (Commit)。在使用 `transaction` 方法時，你不需要擔心手動復原或提交：

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    DB::update('update users set votes = 1');

    DB::delete('delete from posts');
});
```


<a name="handling-deadlocks"></a>
#### 處理死結

`transaction` 方法接受可選的第二個引數，用於定義發生死結 (Deadlock) 時交易應該重試的次數。一旦用盡這些重試次數，將會拋出例外：

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    DB::update('update users set votes = 1');

    DB::delete('delete from posts');
}, attempts: 5);
```


<a name="manually-using-transactions"></a>
#### 手動使用交易

如果你想要手動開始交易，並完全掌控復原與提交，可以使用 `DB` Facade 提供的 `beginTransaction` 方法：

```php
use Illuminate\Support\Facades\DB;

DB::beginTransaction();
```

你可以透過 `rollBack` 方法來復原交易：

```php
DB::rollBack();
```

最後，你可以透過 `commit` 方法來提交交易：

```php
DB::commit();
```

> [!NOTE]
> `DB` Facade 的交易方法同時控制了[查詢建構器](/docs/{{version}}/queries)與 [Eloquent ORM](/docs/{{version}}/eloquent) 的交易。


<a name="connecting-to-the-database-cli"></a>
## 連線至資料庫 CLI

如果你想要連線到資料庫的 CLI，可以使用 `db` Artisan 指令：

```shell
php artisan db
```

如有需要，你可以指定資料庫連線名稱，以連線到非預設連線的資料庫：

```shell
php artisan db mysql
```


<a name="inspecting-your-databases"></a>
## 檢視你的資料庫

使用 `db:show` 和 `db:table` Artisan 指令，你可以深入了解資料庫及其相關的資料表。若要查看資料庫的總覽，包括其大小、類型、開啟連線數以及資料表的摘要，你可以使用 `db:show` 指令：

```shell
php artisan db:show
```

你可以透過 `--database` 選項向指令提供資料庫連線名稱，藉此指定要檢查的資料庫連線：

```shell
php artisan db:show --database=pgsql
```

如果你希望在指令輸出中包含資料表列數 (Row counts) 和資料庫檢視表 (View) 的詳細資訊，可以分別提供 `--counts` 和 `--views` 選項。在大型資料庫上，取得列數和檢視表詳細資訊可能會比較慢：

```shell
php artisan db:show --counts --views
```

此外，你也可以使用下列 `Schema` 方法來檢視你的資料庫：

```php
use Illuminate\Support\Facades\Schema;

$tables = Schema::getTables();
$views = Schema::getViews();
$columns = Schema::getColumns('users');
$indexes = Schema::getIndexes('users');
$foreignKeys = Schema::getForeignKeys('users');
```

如果你想檢查非應用程式預設連線的資料庫，可以使用 `connection` 方法：

```php
$columns = Schema::connection('sqlite')->getColumns('users');
```


<a name="table-overview"></a>
#### 資料表總覽

如果你想取得資料庫中個別資料表的總覽，可以執行 `db:table` Artisan 指令。此指令提供資料庫資料表的一般總覽，包括其欄位、型別、屬性、索引鍵 (Key) 和索引 (Index)：

```shell
php artisan db:table users
```


<a name="monitoring-your-databases"></a>
## 監控你的資料庫

使用 `db:monitor` Artisan 指令，你可以指示 Laravel 在資料庫管理的開啟連線數超過指定數量時，發送 `Illuminate\Database\Events\DatabaseBusy` 事件。

若要開始使用，你應該排程 `db:monitor` 指令[每分鐘執行一次](/docs/{{version}}/scheduling)。該指令接受你想監控的資料庫連線設定名稱，以及在發送事件之前所能容忍的最大開啟連線數：

```shell
php artisan db:monitor --databases=mysql,pgsql --max=100
```

單純排程此指令不足以觸發警示開啟連線數的通知。當指令遇到開啟連線數超過閥值的資料庫時，將會發送 `DatabaseBusy` 事件。你應該在應用程式的 `AppServiceProvider` 中監聽此事件，以便向你或你的開發團隊發送通知：

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