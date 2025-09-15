# HTTP Session

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動程式先決條件](#driver-prerequisites)
- [與 Session 互動](#interacting-with-the-session)
    - [擷取資料](#retrieving-data)
    - [儲存資料](#storing-data)
    - [快閃資料](#flash-data)
    - [刪除資料](#deleting-data)
    - [重新產生 Session ID](#regenerating-the-session-id)
- [Session 阻擋](#session-blocking)
- [新增自訂 Session 驅動程式](#adding-custom-session-drivers)
    - [實作驅動程式](#implementing-the-driver)
    - [註冊驅動程式](#registering-the-driver)

<a name="introduction"></a>
## 簡介

由於 HTTP 驅動的應用程式是無狀態的，Session 提供了一種在多個請求之間儲存使用者資訊的方法。這些使用者資訊通常會被放置在一個持久性儲存/後端中，以便後續的請求可以存取。

Laravel 內建了多種 Session 後端，這些後端透過表達性且統一的 API 進行存取。它支援流行的後端，例如 [Memcached](https://memcached.org)、[Redis](https://redis.io) 和資料庫。

<a name="configuration"></a>
### 設定

應用程式的 Session 設定檔儲存在 `config/session.php`。請務必檢閱此檔案中可用的選項。預設情況下，Laravel 配置為使用 `database` Session 驅動程式。

Session 的 `driver` 設定選項定義了每個請求的 Session 資料將儲存在何處。Laravel 包含了多種驅動程式：

<div class="content-list" markdown="1">

- `file` - Session 儲存在 `storage/framework/sessions` 中。
- `cookie` - Session 儲存在安全、加密的 Cookie 中。
- `database` - Session 儲存在關聯式資料庫中。
- `memcached` / `redis` - Session 儲存在這些快速、基於快取的儲存中。
- `dynamodb` - Session 儲存在 AWS DynamoDB 中。
- `array` - Session 儲存在 PHP 陣列中，不會被持久化。

</div>

> [!NOTE]  
> `array` 驅動程式主要用於 [測試](/docs/{{version}}/testing) 期間，並防止儲存在 Session 中的資料被持久化。

<a name="driver-prerequisites"></a>
### 驅動程式先決條件

<a name="database"></a>
#### Database

當使用 `database` Session 驅動程式時，您需要確保有一個資料庫表格來包含 Session 資料。通常，這會包含在 Laravel 預設的 `0001_01_01_000000_create_users_table.php` [資料庫遷移](/docs/{{version}}/migrations) 中；但是，如果由於任何原因您沒有 `sessions` 表格，您可以使用 `make:session-table` Artisan 命令來產生此遷移：

```shell
php artisan make:session-table

php artisan migrate
```

<a name="redis"></a>
#### Redis

在使用 Redis Session 之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或透過 Composer 安裝 `predis/predis` 套件 (~1.0)。有關配置 Redis 的更多資訊，請參閱 Laravel 的 [Redis 說明文件](/docs/{{version}}/redis#configuration)。

> [!NOTE]  
> `SESSION_CONNECTION` 環境變數，或 `session.php` 設定檔中的 `connection` 選項，可用於指定哪個 Redis 連線用於 Session 儲存。

<a name="interacting-with-the-session"></a>
## 與 Session 互動

<a name="retrieving-data"></a>
### 擷取資料

在 Laravel 中處理 Session 資料主要有兩種方式：全域的 `session` 輔助函式和透過 `Request` 實例。首先，讓我們看看如何透過 `Request` 實例存取 Session，這可以在路由閉包或控制器方法上進行型別提示。請記住，控制器方法的依賴項會透過 Laravel [Service Container](/docs/{{version}}/container) 自動注入：

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\Request;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * Show the profile for the given user.
         */
        public function show(Request $request, string $id): View
        {
            $value = $request->session()->get('key');

            // ...

            $user = $this->users->find($id);

            return view('user.profile', ['user' => $user]);
        }
    }

當您從 Session 擷取項目時，您也可以將預設值作為 `get` 方法的第二個參數傳遞。如果 Session 中不存在指定的鍵，則會回傳此預設值。如果您將閉包作為預設值傳遞給 `get` 方法，並且請求的鍵不存在，則該閉包將被執行並回傳其結果：

    $value = $request->session()->get('key', 'default');

    $value = $request->session()->get('key', function () {
        return 'default';
    });

<a name="the-global-session-helper"></a>
#### 全域 Session 輔助函式

您也可以使用全域的 `session` PHP 函式來擷取和儲存 Session 中的資料。當 `session` 輔助函式以單一字串參數呼叫時，它將回傳該 Session 鍵的值。當輔助函式以鍵/值對陣列呼叫時，這些值將儲存在 Session 中：

    Route::get('/home', function () {
        // Retrieve a piece of data from the session...
        $value = session('key');

        // Specifying a default value...
        $value = session('key', 'default');

        // Store a piece of data in the session...
        session(['key' => 'value']);
    });

> [!NOTE]  
> 透過 HTTP 請求實例使用 Session 與使用全域 `session` 輔助函式之間幾乎沒有實際差異。這兩種方法都可以透過所有測試案例中可用的 `assertSessionHas` 方法進行 [測試](/docs/{{version}}/testing)。

<a name="retrieving-all-session-data"></a>
#### 擷取所有 Session 資料

如果您想擷取 Session 中的所有資料，可以使用 `all` 方法：

    $data = $request->session()->all();

<a name="retrieving-a-portion-of-the-session-data"></a>
#### 擷取部分 Session 資料

`only` 和 `except` 方法可用於擷取 Session 資料的子集：

    $data = $request->session()->only(['username', 'email']);

    $data = $request->session()->except(['username', 'email']);

<a name="determining-if-an-item-exists-in-the-session"></a>
#### 判斷項目是否存在於 Session 中

要判斷項目是否存在於 Session 中，您可以使用 `has` 方法。如果項目存在且不為 `null`，`has` 方法會回傳 `true`：

    if ($request->session()->has('users')) {
        // ...
    }

要判斷項目是否存在於 Session 中，即使其值為 `null`，您可以使用 `exists` 方法：

    if ($request->session()->exists('users')) {
        // ...
    }

要判斷項目是否不存在於 Session 中，您可以使用 `missing` 方法。如果項目不存在，`missing` 方法會回傳 `true`：

    if ($request->session()->missing('users')) {
        // ...
    }

<a name="storing-data"></a>
### 儲存資料

要將資料儲存在 Session 中，您通常會使用請求實例的 `put` 方法或全域的 `session` 輔助函式：

    // Via a request instance...
    $request->session()->put('key', 'value');

    // Via the global "session" helper...
    session(['key' => 'value']);

<a name="pushing-to-array-session-values"></a>
#### 推入陣列 Session 值

`push` 方法可用於將新值推入作為陣列的 Session 值。例如，如果 `user.teams` 鍵包含一個團隊名稱陣列，您可以像這樣將新值推入陣列：

    $request->session()->push('user.teams', 'developers');

<a name="retrieving-deleting-an-item"></a>
#### 擷取並刪除項目

`pull` 方法將在單一陳述式中從 Session 擷取並刪除項目：

    $value = $request->session()->pull('key', 'default');

<a name="incrementing-and-decrementing-session-values"></a>
#### 遞增與遞減 Session 值

如果您的 Session 資料包含您希望遞增或遞減的整數，您可以使用 `increment` 和 `decrement` 方法：

    $request->session()->increment('count');

    $request->session()->increment('count', $incrementBy = 2);

    $request->session()->decrement('count');

    $request->session()->decrement('count', $decrementBy = 2);

<a name="flash-data"></a>
### 快閃資料

有時您可能希望將項目儲存在 Session 中以供下一個請求使用。您可以使用 `flash` 方法來實現。使用此方法儲存在 Session 中的資料將立即可用，並在隨後的 HTTP 請求期間可用。在隨後的 HTTP 請求之後，快閃資料將被刪除。快閃資料主要用於短暫的狀態訊息：

    $request->session()->flash('status', 'Task was successful!');

如果您需要將快閃資料持久化多個請求，您可以使用 `reflash` 方法，它將保留所有快閃資料以供額外的請求使用。如果您只需要保留特定的快閃資料，您可以使用 `keep` 方法：

    $request->session()->reflash();

    $request->session()->keep(['username', 'email']);

要僅將快閃資料持久化到當前請求，您可以使用 `now` 方法：

    $request->session()->now('status', 'Task was successful!');

<a name="deleting-data"></a>
### 刪除資料

`forget` 方法將從 Session 中移除一筆資料。如果您想從 Session 中移除所有資料，您可以使用 `flush` 方法：

    // Forget a single key...
    $request->session()->forget('name');

    // Forget multiple keys...
    $request->session()->forget(['name', 'status']);

    $request->session()->flush();

<a name="regenerating-the-session-id"></a>
### 重新產生 Session ID

重新產生 Session ID 通常是為了防止惡意使用者利用 [Session 固定攻擊](https://owasp.org/www-community/attacks/Session_fixation) 來攻擊您的應用程式。

如果您正在使用 Laravel [應用程式入門套件](/docs/{{version}}/starter-kits) 或 [Laravel Fortify](/docs/{{version}}/fortify)，Laravel 會在認證期間自動重新產生 Session ID；但是，如果您需要手動重新產生 Session ID，您可以使用 `regenerate` 方法：

    $request->session()->regenerate();

如果您需要在單一陳述式中重新產生 Session ID 並從 Session 中移除所有資料，您可以使用 `invalidate` 方法：

    $request->session()->invalidate();

<a name="session-blocking"></a>
## Session 阻擋

> [!WARNING]  
> 要使用 Session 阻擋，您的應用程式必須使用支援 [原子鎖](/docs/{{version}}/cache#atomic-locks) 的快取驅動程式。目前，這些快取驅動程式包括 `memcached`、`dynamodb`、`redis`、`mongodb` (包含在官方 `mongodb/laravel-mongodb` 套件中)、`database`、`file` 和 `array` 驅動程式。此外，您不能使用 `cookie` Session 驅動程式。

預設情況下，Laravel 允許使用相同 Session 的請求並行執行。因此，例如，如果您使用 JavaScript HTTP 函式庫向您的應用程式發出兩個 HTTP 請求，它們將同時執行。對於許多應用程式來說，這不是問題；但是，在少數應用程式中，如果同時向兩個不同的應用程式端點發出請求，並且這兩個端點都向 Session 寫入資料，則可能會發生 Session 資料遺失。

為了緩解這個問題，Laravel 提供了功能，允許您限制給定 Session 的並行請求。要開始使用，您只需將 `block` 方法鏈接到您的路由定義上。在此範例中，對 `/profile` 端點的傳入請求將取得 Session 鎖。當此鎖被持有時，任何對 `/profile` 或 `/order` 端點的傳入請求，如果共用相同的 Session ID，將會等待第一個請求執行完成後再繼續執行：

    Route::post('/profile', function () {
        // ...
    })->block($lockSeconds = 10, $waitSeconds = 10);

    Route::post('/order', function () {
        // ...
    })->block($lockSeconds = 10, $waitSeconds = 10);

`block` 方法接受兩個可選參數。`block` 方法接受的第一個參數是 Session 鎖在釋放之前應持有的最大秒數。當然，如果請求在此時間之前執行完成，鎖將會更早釋放。

`block` 方法接受的第二個參數是請求在嘗試取得 Session 鎖時應等待的秒數。如果請求無法在給定秒數內取得 Session 鎖，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`。

如果這兩個參數都沒有傳遞，則鎖將最多保持 10 秒，並且請求在嘗試取得鎖時最多等待 10 秒：

    Route::post('/profile', function () {
        // ...
    })->block();

<a name="adding-custom-session-drivers"></a>
## 新增自訂 Session 驅動程式

<a name="implementing-the-driver"></a>
### 實作驅動程式

如果現有的 Session 驅動程式都不符合您的應用程式需求，Laravel 允許您編寫自己的 Session 處理器。您的自訂 Session 驅動程式應實作 PHP 內建的 `SessionHandlerInterface`。此介面只包含幾個簡單的方法。一個簡化的 MongoDB 實作範例如下：

    <?php

    namespace App\Extensions;

    class MongoSessionHandler implements \SessionHandlerInterface
    {
        public function open($savePath, $sessionName) {}
        public function close() {}
        public function read($sessionId) {}
        public function write($sessionId, $data) {}
        public function destroy($sessionId) {}
        public function gc($lifetime) {}
    }

由於 Laravel 不包含用於存放擴充功能的預設目錄。您可以將它們放置在任何您喜歡的位置。在此範例中，我們建立了一個 `Extensions` 目錄來存放 `MongoSessionHandler`。

由於這些方法的目的並不容易理解，以下是每個方法目的的概述：

<div class="content-list" markdown="1">

- `open` 方法通常用於基於檔案的 Session 儲存系統。由於 Laravel 內建了 `file` Session 驅動程式，您很少需要在這個方法中放入任何內容。您可以簡單地將此方法留空。
- `close` 方法，就像 `open` 方法一樣，通常也可以忽略。對於大多數驅動程式來說，它是不需要的。
- `read` 方法應回傳與給定 `$sessionId` 相關聯的 Session 資料的字串版本。在您的驅動程式中擷取或儲存 Session 資料時，無需進行任何序列化或其他編碼，因為 Laravel 會為您執行序列化。
- `write` 方法應將與 `$sessionId` 相關聯的給定 `$data` 字串寫入某個持久性儲存系統，例如 MongoDB 或您選擇的其他儲存系統。同樣，您不應執行任何序列化 - Laravel 會為您處理。
- `destroy` 方法應從持久性儲存中移除與 `$sessionId` 相關聯的資料。
- `gc` 方法應銷毀所有比給定 `$lifetime` (一個 UNIX 時間戳記) 更舊的 Session 資料。對於像 Memcached 和 Redis 這樣會自動過期的系統，此方法可以留空。

</div>

<a name="registering-the-driver"></a>
### 註冊驅動程式

一旦您的驅動程式實作完成，您就可以將其註冊到 Laravel。要向 Laravel 的 Session 後端新增額外的驅動程式，您可以使用 `Session` [Facade](/docs/{{version}}/facades) 提供的 `extend` 方法。您應該從 [Service Provider](/docs/{{version}}/providers) 的 `boot` 方法中呼叫 `extend` 方法。您可以從現有的 `App\Providers\AppServiceProvider` 中執行此操作，或建立一個全新的 Provider：

    <?php

    namespace App\Providers;

    use App\Extensions\MongoSessionHandler;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\Session;
    use Illuminate\Support\ServiceProvider;

    class SessionServiceProvider extends ServiceProvider
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
            Session::extend('mongo', function (Application $app) {
                // Return an implementation of SessionHandlerInterface...
                return new MongoSessionHandler;
            });
        }
    }

一旦 Session 驅動程式註冊完成，您就可以使用 `SESSION_DRIVER` 環境變數或在應用程式的 `config/session.php` 設定檔中，將 `mongo` 驅動程式指定為您應用程式的 Session 驅動程式。
