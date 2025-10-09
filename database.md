# 資料庫：入門指南

- [介紹](#introduction)
    - [設定](#configuration)
    - [讀寫連線](#read-and-write-connections)
- [執行 SQL 查詢](#running-queries)
    - [使用多個資料庫連線](#using-multiple-database-connections)
    - [監聽查詢事件](#listening-for-query-events)
    - [監控累積查詢時間](#monitoring-cumulative-query-time)
- [資料庫交易](#database-transactions)
- [連接到資料庫 CLI](#connecting-to-the-database-cli)
- [檢查資料庫](#inspecting-your-databases)
- [監控資料庫](#monitoring-your-databases)

<a name="introduction"></a>
## 介紹

幾乎所有現代網路應用程式都會與資料庫互動。Laravel 讓透過裸 SQL (raw SQL)、[流暢查詢產生器](/docs/{{version}}/queries)以及 [Eloquent ORM](/docs/{{version}}/eloquent) 來與多種支援的資料庫互動變得極為簡單。目前，Laravel 針對五種資料庫提供第一方支援：

<div class="content-list" markdown="1">

- MariaDB 10.3+ ([版本政策](https://mariadb.org/about/#maintenance-policy))
- MySQL 5.7+ ([版本政策](https://en.wikipedia.org/wiki/MySQL#Release_history))
- PostgreSQL 10.0+ ([版本政策](https://www.postgresql.org/support/versioning/))
- SQLite 3.26.0+
- SQL Server 2017+ ([版本政策](https://docs.microsoft.com/en-us/lifecycle/products/?products=sql-server))

</div>

此外，MongoDB 透過 `mongodb/laravel-mongodb` 套件提供支援，此套件由 MongoDB 官方維護。查看 [Laravel MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/) 文件以獲取更多資訊。

<a name="configuration"></a>
### 設定

Laravel 資料庫服務的設定位於應用程式的 `config/database.php` 設定檔中。在此檔案中，您可以定義所有資料庫連線，並指定預設應使用的連線。此檔案中的大多數設定選項都由應用程式的環境變數值驅動。此檔案中提供了 Laravel 大部分支援資料庫系統的範例。

預設情況下，Laravel 的範例[環境設定](/docs/{{version}}/configuration#environment-configuration)已可與 [Laravel Sail](/docs/{{version}}/sail) 搭配使用，Laravel Sail 是一種用於在本機開發 Laravel 應用程式的 Docker 設定。但是，您可以根據本機資料庫的需求修改資料庫設定。

<a name="sqlite-configuration"></a>
#### SQLite 設定

SQLite 資料庫包含在檔案系統中的單一檔案內。您可以使用終端機中的 `touch` 命令來建立新的 SQLite 資料庫：`touch database/database.sqlite`。資料庫建立後，您可以將資料庫的絕對路徑放置在 `DB_DATABASE` 環境變數中，輕鬆設定環境變數以指向此資料庫：

```ini
DB_CONNECTION=sqlite
DB_DATABASE=/absolute/path/to/database.sqlite
```

預設情況下，SQLite 連線會啟用外部鍵約束。如果您想禁用它們，應將 `DB_FOREIGN_KEYS` 環境變數設定為 `false`：

```ini
DB_FOREIGN_KEYS=false
```

> [!NOTE]
> 如果您使用 [Laravel 安裝器](/docs/{{version}}/installation#creating-a-laravel-project) 建立 Laravel 應用程式並選擇 SQLite 作為資料庫，Laravel 將自動為您建立 `database/database.sqlite` 檔案並執行預設的[資料庫遷移](/docs/{{version}}/migrations)。

<a name="mssql-configuration"></a>
#### Microsoft SQL Server 設定

若要使用 Microsoft SQL Server 資料庫，您應確保已安裝 `sqlsrv` 和 `pdo_sqlsrv` PHP 擴充功能，以及它們可能需要的任何依賴項，例如 Microsoft SQL ODBC 驅動程式。

<a name="configuration-using-urls"></a>
#### 使用 URL 進行設定

通常，資料庫連線會使用多個設定值進行設定，例如 `host`、`database`、`username`、`password` 等。這些設定值中的每一個都有其對應的環境變數。這表示在生產伺服器上設定資料庫連線資訊時，您需要管理多個環境變數。

某些託管資料庫供應商，例如 AWS 和 Heroku，提供單一的資料庫「URL」，其中包含資料庫的所有連線資訊，以單一字串形式呈現。資料庫 URL 範例可能如下所示：

```html
mysql://root:password@127.0.0.1/forge?charset=UTF-8
```

這些 URL 通常遵循標準的結構描述慣例：

```html
driver://username:password@host:port/database?options
```

為方便起見，Laravel 支援這些 URL，作為使用多個設定選項設定資料庫的替代方案。如果 `url` (或對應的 `DB_URL` 環境變數) 設定選項存在，它將用於提取資料庫連線和憑證資訊。

<a name="read-and-write-connections"></a>
### 讀寫連線

有時您可能希望將一個資料庫連線用於 SELECT 語句，而另一個用於 INSERT、UPDATE 和 DELETE 語句。Laravel 讓這變得輕而易舉，無論您使用裸查詢、查詢產生器或 Eloquent ORM，都會始終使用正確的連線。

為了了解讀/寫連線應如何設定，讓我們看這個範例：

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

請注意，設定陣列中新增了三個鍵：`read`、`write` 和 `sticky`。`read` 和 `write` 鍵具有包含單一鍵 `host` 的陣列值。讀 (`read`) 和寫 (`write`) 連線的其餘資料庫選項將從主要的 `mysql` 設定陣列中合併。

只有當您希望覆寫主要 `mysql` 陣列中的值時，才需要將項目放置在 `read` 和 `write` 陣列中。因此，在此情況下，`192.168.1.1` 將用作「讀取」連線的 host，而 `192.168.1.3` 將用作「寫入」連線。主要 `mysql` 陣列中的資料庫憑證、前綴、字元集和所有其他選項將在兩個連線之間共用。當 `host` 設定陣列中存在多個值時，每個請求將隨機選擇一個資料庫主機。

<a name="the-sticky-option"></a>
#### `sticky` 選項

`sticky` 選項是一個*可選*值，可用於允許立即讀取在當前請求週期中寫入資料庫的記錄。如果 `sticky` 選項啟用，並且在當前請求週期中已對資料庫執行「寫入」操作，則任何後續的「讀取」操作都將使用「寫入」連線。這確保了在請求週期中寫入的任何資料都可以立即從資料庫中讀回。您需要自行決定這是否是您應用程式所需的行為。

<a name="running-queries"></a>
## 執行 SQL 查詢

設定好資料庫連線後，您可以使用 `DB` Facade 來執行查詢。`DB` Facade 提供了各種查詢類型的方法：`select`、`update`、`insert`、`delete` 和 `statement`。


<a name="running-a-select-query"></a>
#### 執行 SELECT 查詢

要執行基本的 SELECT 查詢，您可以使用 `DB` Facade 的 `select` 方法：

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

傳遞給 `select` 方法的第一個引數是 SQL 查詢，而第二個引數是需要繫結到查詢的任何參數繫結。通常，這些是 `where` 子句條件的值。參數繫結可提供對抗 SQL 注入的保護。

`select` 方法將始終傳回一個結果 `array`。陣列中的每個結果都將是一個 PHP `stdClass` 物件，代表資料庫中的一條記錄：

```php
use Illuminate\Support\Facades\DB;

$users = DB::select('select * from users');

foreach ($users as $user) {
    echo $user->name;
}
```


<a name="selecting-scalar-values"></a>
#### 查詢純量值

有時您的資料庫查詢可能只會產生單一的純量值。Laravel 允許您直接使用 `scalar` 方法取得此值，而無需從記錄物件中擷取查詢的純量結果：

```php
$burgers = DB::scalar(
    "select count(case when food = 'burger' then 1 end) as burgers from menu"
);
```


<a name="selecting-multiple-result-sets"></a>
#### 查詢多個結果集

如果您的應用程式呼叫的預存程序會傳回多個結果集，您可以使用 `selectResultSets` 方法來擷取預存程序傳回的所有結果集：

```php
[$options, $notifications] = DB::selectResultSets(
    "CALL get_user_options_and_notifications(?)", $request->user()->id
);
```


<a name="using-named-bindings"></a>
#### 使用具名繫結

除了使用 `?` 來表示參數繫結外，您還可以使用具名繫結來執行查詢：

```php
$results = DB::select('select * from users where id = :id', ['id' => 1]);
```


<a name="running-an-insert-statement"></a>
#### 執行 INSERT 陳述式

要執行 `insert` 陳述式，您可以使用 `DB` Facade 上的 `insert` 方法。與 `select` 類似，此方法將 SQL 查詢作為第一個引數，將繫結作為第二個引數：

```php
use Illuminate\Support\Facades\DB;

DB::insert('insert into users (id, name) values (?, ?)', [1, 'Marc']);
```


<a name="running-an-update-statement"></a>
#### 執行 UPDATE 陳述式

`update` 方法應用於更新資料庫中的現有記錄。此方法會傳回受陳述式影響的列數：

```php
use Illuminate\Support\Facades\DB;

$affected = DB::update(
    'update users set votes = 100 where name = ?',
    ['Anita']
);
```


<a name="running-a-delete-statement"></a>
#### 執行 DELETE 陳述式

`delete` 方法應用於從資料庫中刪除記錄。與 `update` 類似，此方法將傳回受影響的列數：

```php
use Illuminate\Support\Facades\DB;

$deleted = DB::delete('delete from users');
```


<a name="running-a-general-statement"></a>
#### 執行一般陳述式

有些資料庫陳述式不傳回任何值。對於這些類型的操作，您可以使用 `DB` Facade 上的 `statement` 方法：

```php
DB::statement('drop table users');
```


<a name="running-an-unprepared-statement"></a>
#### 執行未準備的陳述式

有時您可能希望執行 SQL 陳述式而不繫結任何值。您可以使用 `DB` Facade 的 `unprepared` 方法來實現此目的：

```php
DB::unprepared('update users set votes = 100 where name = "Dries"');
```

> [!WARNING]
> 由於未準備的陳述式不會繫結參數，因此它們可能容易受到 SQL 注入的影響。您絕不應在未準備的陳述式中允許使用者控制的值。


<a name="implicit-commits-in-transactions"></a>
#### 隱式提交

在交易中使用 `DB` Facade 的 `statement` 和 `unprepared` 方法時，您必須小心避免導致 [隱式提交](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html) 的陳述式。這些陳述式將導致資料庫引擎間接提交整個交易，使 Laravel 不知道資料庫的交易層級。此類陳述式的範例是建立資料庫表：

```php
DB::unprepared('create table a (col varchar(1) null)');
```

請參閱 MySQL 手冊以取得 [所有會觸發隱式提交的陳述式列表](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)。


<a name="using-multiple-database-connections"></a>
### 使用多個資料庫連線

如果您的應用程式在 `config/database.php` 設定檔中定義了多個連線，您可以透過 `DB` Facade 提供的 `connection` 方法來存取每個連線。傳遞給 `connection` 方法的連線名稱應與您的 `config/database.php` 設定檔中列出的其中一個連線相對應，或者使用 `config` 輔助函數在執行時設定的連線：

```php
use Illuminate\Support\Facades\DB;

$users = DB::connection('sqlite')->select(/* ... */);
```

您可以使用連線實例上的 `getPdo` 方法來存取連線的原始底層 PDO 實例：

```php
$pdo = DB::connection()->getPdo();
```


<a name="listening-for-query-events"></a>
### 監聽查詢事件

如果您想為應用程式執行的每個 SQL 查詢指定一個要叫用的閉包 (closure)，您可以使用 `DB` Facade 的 `listen` 方法。此方法對於記錄查詢或偵錯很有用。您可以在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中註冊您的查詢監聽閉包：

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

現代 Web 應用程式常見的效能瓶頸是查詢資料庫所花費的時間。幸運的是，如果 Laravel 在單次請求期間查詢資料庫的時間過長，它可以叫用您選擇的閉包或回呼。首先，向 `whenQueryingForLongerThan` 方法提供一個查詢時間閾值（以毫秒為單位）和閉包。您可以在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中叫用此方法：

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

您可以使用 `DB` Facade 提供的 `transaction` 方法，在資料庫交易中執行一組操作。如果交易閉包中拋出異常，交易將自動回溯 (rolled back)，並重新拋出該異常。如果閉包成功執行，交易將自動提交 (committed)。在使用 `transaction` 方法時，您不需要擔心手動回溯或提交：

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    DB::update('update users set votes = 1');

    DB::delete('delete from posts');
});
```


<a name="handling-deadlocks"></a>
#### 處理死結

`transaction` 方法接受一個選用的第二個參數，該參數定義了當發生死結時，交易應該重試的次數。一旦這些嘗試次數用盡，就會拋出異常：

```php
use Illuminate\Support\Facades\DB;

DB::transaction(function () {
    DB::update('update users set votes = 1');

    DB::delete('delete from posts');
}, attempts: 5);
```


<a name="manually-using-transactions"></a>
#### 手動使用交易

如果您想手動開始交易並完全控制回溯與提交，您可以使用 `DB` Facade 提供的 `beginTransaction` 方法：

```php
use Illuminate\Support\Facades\DB;

DB::beginTransaction();
```

您可以使用 `rollBack` 方法回溯交易：

```php
DB::rollBack();
```

最後，您可以使用 `commit` 方法提交交易：

```php
DB::commit();
```

> [!NOTE]
> `DB` Facade 的交易方法同時控制著 [查詢建構器](/docs/{{version}}/queries) 與 [Eloquent ORM](/docs/{{version}}/eloquent) 的交易。


<a name="connecting-to-the-database-cli"></a>
## 連接到資料庫 CLI

如果您想連接到資料庫的 CLI，您可以使用 `db` Artisan 命令：

```shell
php artisan db
```

如有需要，您可以指定資料庫連線名稱以連接到非預設連線的資料庫：

```shell
php artisan db mysql
```


<a name="inspecting-your-databases"></a>
## 檢查資料庫

使用 `db:show` 和 `db:table` Artisan 命令，您可以深入了解資料庫及其相關資料表。要查看資料庫的總覽，包括其大小、類型、開放連線數以及資料表的摘要，您可以使用 `db:show` 命令：

```shell
php artisan db:show
```

您可以透過 `--database` 選項向命令提供資料庫連線名稱，以指定要檢查哪個資料庫連線：

```shell
php artisan db:show --database=pgsql
```

如果您想在命令輸出中包含資料表行數和資料庫視圖詳細資訊，您可以分別提供 `--counts` 和 `--views` 選項。在大型資料庫上，檢索行數和視圖詳細資訊可能會很慢：

```shell
php artisan db:show --counts --views
```

此外，您還可以使用以下 `Schema` 方法來檢查資料庫：

```php
use Illuminate\Support\Facades\Schema;

$tables = Schema::getTables();
$views = Schema::getViews();
$columns = Schema::getColumns('users');
$indexes = Schema::getIndexes('users');
$foreignKeys = Schema::getForeignKeys('users');
```

如果您想檢查的資料庫連線不是您應用程式的預設連線，您可以使用 `connection` 方法：

```php
$columns = Schema::connection('sqlite')->getColumns('users');
```


<a name="table-overview"></a>
#### 資料表總覽

如果您想取得資料庫中個別資料表的總覽，您可以執行 `db:table` Artisan 命令。此命令提供了資料表的整體概覽，包括其欄位、類型、屬性、索引鍵和索引：

```shell
php artisan db:table users
```


<a name="monitoring-your-databases"></a>
## 監控資料庫

使用 `db:monitor` Artisan 命令，您可以指示 Laravel 在資料庫管理超過指定數量的開放連線時，分發一個 `Illuminate\Database\Events\DatabaseBusy` 事件。

首先，您應該將 `db:monitor` 命令排程為 [每分鐘執行一次](/docs/{{version}}/scheduling)。該命令接受您希望監控的資料庫連線設定名稱，以及在分發事件之前可以容忍的最大開放連線數：

```shell
php artisan db:monitor --databases=mysql,pgsql --max=100
```

僅僅排程此命令並不足以觸發通知以提醒您開放連線數。當命令遇到開放連線數超過閾值的資料庫時，將會分發一個 `DatabaseBusy` 事件。您應該在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊發送通知：

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