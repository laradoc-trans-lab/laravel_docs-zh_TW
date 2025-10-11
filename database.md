# 資料庫：入門

- [導言](#introduction)
    - [配置](#configuration)
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
## 導言

幾乎所有現代的 Web 應用程式都與資料庫互動。Laravel 透過原生的 SQL、一個 [流暢的查詢產生器](/docs/{{version}}/queries) 和 [Eloquent ORM](/docs/{{version}}/eloquent)，讓與各種支援的資料庫互動變得極其簡單。目前，Laravel 提供對五種資料庫的第一方支援：

<div class="content-list" markdown="1">

- MariaDB 10.3+ ([版本策略](https://mariadb.org/about/#maintenance-policy))
- MySQL 5.7+ ([版本策略](https://en.wikipedia.org/wiki/MySQL#Release_history))
- PostgreSQL 10.0+ ([版本策略](https://www.postgresql.org/support/versioning/))
- SQLite 3.26.0+
- SQL Server 2017+ ([版本策略](https://docs.microsoft.com/en-us/lifecycle/products/?products=sql-server))

</div>

此外，MongoDB 透過 `mongodb/laravel-mongodb` 套件提供支援，此套件由 MongoDB 官方維護。請查閱 [Laravel MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/) 文件以獲取更多資訊。


<a name="configuration"></a>
### 配置

Laravel 資料庫服務的配置位於應用程式的 `config/database.php` 配置檔中。在此檔案中，您可以定義所有資料庫連線，並指定應預設使用哪個連線。此檔案中的大多數配置選項都受應用程式環境變數的值驅動。此檔案提供了 Laravel 大多數支援資料庫系統的範例。

預設情況下，Laravel 的範例 [環境配置](/docs/{{version}}/configuration#environment-configuration) 已準備好與 [Laravel Sail](/docs/{{version}}/sail) 一起使用，它是一個用於在本機開發 Laravel 應用程式的 Docker 配置。但是，您可以根據本機資料庫的需要自由修改資料庫配置。


<a name="sqlite-configuration"></a>
#### SQLite 配置

SQLite 資料庫包含在檔案系統中的單一檔案中。您可以在終端機中使用 `touch` 指令來建立新的 SQLite 資料庫：`touch database/database.sqlite`。資料庫建立後，您可以透過將資料庫的絕對路徑放置在 `DB_DATABASE` 環境變數中，輕鬆配置環境變數以指向此資料庫：

```ini
DB_CONNECTION=sqlite
DB_DATABASE=/absolute/path/to/database.sqlite
```

預設情況下，SQLite 連線會啟用外部鍵限制。如果您想禁用它們，則應將 `DB_FOREIGN_KEYS` 環境變數設定為 `false`：

```ini
DB_FOREIGN_KEYS=false
```

> [!NOTE]  
> 如果您使用 [Laravel 安裝器](/docs/{{version}}/installation#creating-a-laravel-project) 建立您的 Laravel 應用程式並選擇 SQLite 作為資料庫，Laravel 將會自動為您建立 `database/database.sqlite` 檔案並執行預設的 [資料庫遷移](/docs/{{version}}/migrations) 。


<a name="mssql-configuration"></a>
#### Microsoft SQL Server 配置

若要使用 Microsoft SQL Server 資料庫，您應確保已安裝 `sqlsrv` 和 `pdo_sqlsrv` PHP 擴充功能，以及它們可能需要的任何依賴項，例如 Microsoft SQL ODBC 驅動程式。


<a name="configuration-using-urls"></a>
#### 使用 URL 配置

通常，資料庫連線會使用多個配置值來設定，例如 `host`、`database`、`username`、`password` 等。這些配置值中的每一個都有其對應的環境變數。這意味著在生產伺服器上配置資料庫連線資訊時，您需要管理多個環境變數。

某些託管資料庫供應商，例如 AWS 和 Heroku，提供一個包含資料庫所有連線資訊的單一資料庫「URL」。資料庫 URL 的範例如下：

```html
mysql://root:password@127.0.0.1/forge?charset=UTF-8
```

這些 URL 通常遵循標準的架構慣例：

```html
driver://username:password@host:port/database?options
```

為方便起見，Laravel 支援這些 URL，作為使用多個配置選項來設定資料庫的替代方案。如果存在 `url`（或對應的 `DB_URL` 環境變數）配置選項，它將用於提取資料庫連線和憑證資訊。


<a name="read-and-write-connections"></a>
### 讀寫連線

有時您可能希望對 SELECT 語句使用一個資料庫連線，而對 INSERT、UPDATE 和 DELETE 語句使用另一個。Laravel 使這一切變得輕而易舉，無論您是使用原生查詢、查詢產生器還是 Eloquent ORM，都將始終使用正確的連線。

要了解如何配置讀寫連線，請看此範例：

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

請注意，配置陣列中已新增三個鍵：`read`、`write` 和 `sticky`。`read` 和 `write` 鍵具有包含單一鍵：`host` 的陣列值。`read` 和 `write` 連線的其他資料庫選項將從主要的 `mysql` 配置陣列中合併。

您只有在希望覆寫主要 `mysql` 陣列中的值時，才需要將項目放入 `read` 和 `write` 陣列中。因此，在此情況下，`192.168.1.1` 將用作「讀取」連線的主機，而 `192.168.1.3` 將用作「寫入」連線。主要 `mysql` 陣列中的資料庫憑證、前綴、字元集和所有其他選項將在兩個連線之間共享。當 `host` 配置陣列中存在多個值時，每次請求將隨機選擇一個資料庫主機。


<a name="the-sticky-option"></a>
#### `sticky` 選項

`sticky` 選項是一個 *可選* 值，可用於允許立即讀取在當前請求週期中寫入資料庫的記錄。如果 `sticky` 選項已啟用，並且在當前請求週期中已對資料庫執行了「寫入」操作，則任何後續的「讀取」操作都將使用「寫入」連線。這確保了在請求週期中寫入的任何資料可以在同一請求期間立即從資料庫讀回。您可以自行決定這是否是您應用程式所需的行為。

<a name="running-queries"></a>
## 執行 SQL 查詢

配置好資料庫連線後，您可以使用 `DB` facade 來執行查詢。`DB` facade 提供了針對每種類型查詢的方法：`select`、`update`、`insert`、`delete` 和 `statement`。


<a name="running-a-select-query"></a>
#### 執行 SELECT 查詢

要執行基本的 SELECT 查詢，您可以使用 `DB` facade 上的 `select` 方法：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
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

傳遞給 `select` 方法的第一個參數是 SQL 查詢，而第二個參數是任何需要綁定到查詢的參數綁定。通常，這些是 `where` 子句約束的值。參數綁定提供了對 SQL 注入攻擊的保護。

`select` 方法將始終返回一個 `array` 的結果。陣列中的每個結果都將是一個代表資料庫記錄的 PHP `stdClass` 物件：

    use Illuminate\Support\Facades\DB;

    $users = DB::select('select * from users');

    foreach ($users as $user) {
        echo $user->name;
    }


<a name="selecting-scalar-values"></a>
#### 選擇純量值

有時您的資料庫查詢可能只返回一個純量值。Laravel 允許您直接使用 `scalar` 方法檢索此值，而無需從記錄物件中檢索查詢的純量結果：

    $burgers = DB::scalar(
        "select count(case when food = 'burger' then 1 end) as burgers from menu"
    );


<a name="selecting-multiple-result-sets"></a>
#### 選擇多個結果集

如果您的應用程式呼叫返回多個結果集的儲存程序，您可以使用 `selectResultSets` 方法來檢索儲存程序返回的所有結果集：

    [$options, $notifications] = DB::selectResultSets(
        "CALL get_user_options_and_notifications(?)", $request->user()->id
    );


<a name="using-named-bindings"></a>
#### 使用命名綁定

除了使用 `?` 來表示您的參數綁定之外，您還可以使用命名綁定來執行查詢：

    $results = DB::select('select * from users where id = :id', ['id' => 1]);


<a name="running-an-insert-statement"></a>
#### 執行 INSERT 語句

要執行 `insert` 語句，您可以使用 `DB` facade 上的 `insert` 方法。與 `select` 類似，此方法將 SQL 查詢作為第一個參數，綁定作為第二個參數：

    use Illuminate\Support\Facades\DB;

    DB::insert('insert into users (id, name) values (?, ?)', [1, 'Marc']);


<a name="running-an-update-statement"></a>
#### 執行 UPDATE 語句

`update` 方法應用於更新資料庫中的現有記錄。該方法將返回受語句影響的行數：

    use Illuminate\Support\Facades\DB;

    $affected = DB::update(
        'update users set votes = 100 where name = ?',
        ['Anita']
    );


<a name="running-a-delete-statement"></a>
#### 執行 DELETE 語句

`delete` 方法應用於從資料庫中刪除記錄。與 `update` 類似，該方法將返回受影響的行數：

    use Illuminate\Support\Facades\DB;

    $deleted = DB::delete('delete from users');


<a name="running-a-general-statement"></a>
#### 執行一般語句

有些資料庫語句不返回任何值。對於這些類型的操作，您可以使用 `DB` facade 上的 `statement` 方法：

    DB::statement('drop table users');


<a name="running-an-unprepared-statement"></a>
#### 執行未準備的語句

有時您可能希望執行 SQL 語句而不綁定任何值。您可以使用 `DB` facade 的 `unprepared` 方法來實現此目的：

    DB::unprepared('update users set votes = 100 where name = "Dries"');

> [!WARNING]  
> 由於未準備的語句不綁定參數，它們可能容易受到 SQL 注入攻擊。您絕不應在未準備的語句中允許使用者控制的值。


<a name="implicit-commits-in-transactions"></a>
#### 交易中的隱式提交

當在交易中使用 `DB` facade 的 `statement` 和 `unprepared` 方法時，您必須小心避免導致 [隱式提交](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html) 的語句。這些語句將導致資料庫引擎間接提交整個交易，使 Laravel 不知道資料庫的交易層級。此類語句的一個範例是建立資料庫表格：

    DB::unprepared('create table a (col varchar(1) null)');

請參閱 MySQL 手冊以獲取 [所有觸發隱式提交的語句列表](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html)。


<a name="using-multiple-database-connections"></a>
### 使用多個資料庫連線

如果您的應用程式在 `config/database.php` 配置檔中定義了多個連線，您可以透過 `DB` facade 提供的 `connection` 方法來存取每個連線。傳遞給 `connection` 方法的連線名稱應與您的 `config/database.php` 配置檔中列出的連線之一相對應，或在執行時使用 `config` 輔助函數配置：

    use Illuminate\Support\Facades\DB;

    $users = DB::connection('sqlite')->select(/* ... */);

您可以使用連線實例上的 `getPdo` 方法來存取原始的底層 PDO 實例：

    $pdo = DB::connection()->getPdo();


<a name="listening-for-query-events"></a>
### 監聽查詢事件

如果您想指定一個閉包，該閉包在應用程式執行的每個 SQL 查詢時被調用，您可以使用 `DB` facade 的 `listen` 方法。此方法對於記錄查詢或除錯可能很有用。您可以在 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中註冊您的查詢監聽閉包：

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

<a name="monitoring-cumulative-query-time"></a>
### 監控累積查詢時間

現代網路應用程式常見的效能瓶頸之一，就是它們花費在查詢資料庫上的時間。幸好，當 Laravel 在單一請求期間，花費太多時間查詢資料庫時，它能夠調用您選擇的閉包或回呼。若要開始，請提供一個查詢時間閾值（以毫秒為單位）以及一個閉包給 `whenQueryingForLongerThan` 方法。您可以在[服務提供者](/docs/{{version}}/providers)的 `boot` 方法中調用這個方法：

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

<a name="database-transactions"></a>
## 資料庫交易

您可以使用 `DB` facade 提供的 `transaction` 方法，在資料庫交易中執行一組操作。如果在交易閉包 (closure) 中拋出例外，交易將自動回溯 (rolled back)，並重新拋出該例外。如果閉包執行成功，交易將自動提交 (committed)。當您使用 `transaction` 方法時，無需擔心手動回溯或提交：

    use Illuminate\Support\Facades\DB;

    DB::transaction(function () {
        DB::update('update users set votes = 1');

        DB::delete('delete from posts');
    });


<a name="handling-deadlocks"></a>
#### 處理死結 (Deadlocks)

`transaction` 方法接受一個可選的第二個參數，它定義了當發生死結 (deadlock) 時，交易應該重試的次數。一旦這些嘗試次數用盡，將會拋出一個例外：

    use Illuminate\Support\Facades\DB;

    DB::transaction(function () {
        DB::update('update users set votes = 1');

        DB::delete('delete from posts');
    }, 5);


<a name="manually-using-transactions"></a>
#### 手動使用交易

如果您想手動開始一項交易，並完全控制回溯和提交，可以使用 `DB` facade 提供的 `beginTransaction` 方法：

    use Illuminate\Support\Facades\DB;

    DB::beginTransaction();

您可以透過 `rollBack` 方法回溯交易：

    DB::rollBack();

最後，您可以透過 `commit` 方法提交交易：

    DB::commit();

> [!NOTE]  
> `DB` facade 的交易方法控制著 [query builder](/docs/{{version}}/queries) 和 [Eloquent ORM](/docs/{{version}}/eloquent) 的交易。


<a name="connecting-to-the-database-cli"></a>
## 連接到資料庫 CLI

如果您想連接到資料庫的命令列介面 (CLI)，可以使用 `db` Artisan 命令：

```shell
php artisan db
```

如有需要，您可以指定一個資料庫連線名稱，來連接到非預設的資料庫連線：

```shell
php artisan db mysql
```


<a name="inspecting-your-databases"></a>
## 檢查資料庫

透過使用 `db:show` 和 `db:table` Artisan 命令，您可以深入了解資料庫及其相關表格。若要查看資料庫的概覽，包括其大小、類型、開啟的連線數，以及其表格的摘要，可以使用 `db:show` 命令：

```shell
php artisan db:show
```

您可以透過 `--database` 選項向命令提供資料庫連線名稱，來指定應該檢查哪個資料庫連線：

```shell
php artisan db:show --database=pgsql
```

如果您想在命令的輸出中包含表格列計數和資料庫視圖詳細資訊，可以分別提供 `--counts` 和 `--views` 選項。在大型資料庫上，檢索列計數和視圖詳細資訊可能會很慢：

```shell
php artisan db:show --counts --views
```

此外，您可以使用以下 `Schema` 方法來檢查您的資料庫：

    use Illuminate\Support\Facades\Schema;

    $tables = Schema::getTables();
    $views = Schema::getViews();
    $columns = Schema::getColumns('users');
    $indexes = Schema::getIndexes('users');
    $foreignKeys = Schema::getForeignKeys('users');

如果您想檢查一個不是應用程式預設連線的資料庫連線，可以使用 `connection` 方法：

    $columns = Schema::connection('sqlite')->getColumns('users');


<a name="table-overview"></a>
#### 表格概覽

如果您想取得資料庫中個別表格的概覽，可以執行 `db:table` Artisan 命令。此命令提供資料庫表格的概覽，包括其欄位 (columns)、類型 (types)、屬性 (attributes)、鍵 (keys) 和索引 (indexes)：

```shell
php artisan db:table users
```


<a name="monitoring-your-databases"></a>
## 監控資料庫

使用 `db:monitor` Artisan 命令，您可以指示 Laravel 在資料庫管理的開啟連線數超過指定數量時，分派一個 `Illuminate\Database\Events\DatabaseBusy` 事件。

首先，您應該排程 `db:monitor` 命令，使其 [每分鐘執行一次](/docs/{{version}}/scheduling)。該命令接受您希望監控的資料庫連線配置名稱，以及在分派事件之前，可容忍的最大開啟連線數：

```shell
php artisan db:monitor --databases=mysql,pgsql --max=100
```

單獨排程此命令不足以觸發通知，提醒您開啟連線的數量。當命令遇到一個開啟連線數超過您設定閾值的資料庫時，將會分派一個 `DatabaseBusy` 事件。您應該在應用程式的 `AppServiceProvider` 中監聽此事件，以便向您或您的開發團隊發送通知：

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