# HTTP Session

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動器先決條件](#driver-prerequisites)
- [與 Session 互動](#interacting-with-the-session)
    - [擷取資料](#retrieving-data)
    - [儲存資料](#storing-data)
    - [快閃資料](#flash-data)
    - [刪除資料](#deleting-data)
    - [重新產生 Session ID](#regenerating-the-session-id)
- [Session 阻擋](#session-blocking)
- [新增自訂 Session 驅動器](#adding-custom-session-drivers)
    - [實作驅動器](#implementing-the-driver)
    - [註冊驅動器](#registering-the-driver)

<a name="introduction"></a>
## 簡介

由於 HTTP 驅動的應用程式是無狀態的，因此 session 提供了一種跨越多個請求來儲存使用者資訊的方式。該使用者資訊通常放置在持久性儲存 / 後端中，可從後續請求中存取。

Laravel 內建多種 session 後端，可透過具表達性、統一的 API 進行存取。支援流行的後端，例如 [Memcached](https://memcached.org)、[Redis](https://redis.io) 和資料庫。

<a name="configuration"></a>
### 設定

您的應用程式的 session 設定檔儲存在 `config/session.php`。請務必檢閱此檔案中可用的選項。依預設，Laravel 配置為使用 `database` session 驅動器。

session 的 `driver` 設定選項定義了每個請求的 session 資料儲存位置。Laravel 包含了多種驅動器：

<div class="content-list" markdown="1">

- `file` - session 儲存於 `storage/framework/sessions`。
- `cookie` - session 儲存於安全、加密的 cookie 中。
- `database` - session 儲存於關聯式資料庫中。
- `memcached` / `redis` - session 儲存於這些基於快取的快速儲存之一。
- `dynamodb` - session 儲存於 AWS DynamoDB 中。
- `array` - session 儲存於 PHP 陣列中且不會被持久化。

</div>

> [!NOTE]  
> array 驅動器主要用於 [測試](/docs/{{version}}/testing)，並防止儲存在 session 中的資料被持久化。

<a name="driver-prerequisites"></a>
### 驅動器先決條件

<a name="database"></a>
#### 資料庫

使用 `database` session 驅動器時，您需要確保有一個資料庫表格來包含 session 資料。通常，這包含在 Laravel 的預設 `0001_01_01_000000_create_users_table.php` [資料庫遷移](/docs/{{version}}/migrations) 中；但是，如果由於任何原因您沒有 `sessions` 表格，您可以使用 `make:session-table` Artisan 命令來產生此遷移檔：

```shell
php artisan make:session-table

php artisan migrate
```

<a name="redis"></a>
#### Redis

在使用 Laravel 的 Redis session 之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或者透過 Composer 安裝 `predis/predis` 套件 (~1.0)。有關配置 Redis 的更多資訊，請查閱 Laravel 的 [Redis 文件](/docs/{{version}}/redis#configuration)。

> [!NOTE]  
> `SESSION_CONNECTION` 環境變數，或 `session.php` 設定檔中的 `connection` 選項，可用來指定哪個 Redis 連線用於 session 儲存。

<a name="interacting-with-the-session"></a>
## 與 Session 互動

<a name="retrieving-data"></a>
### 擷取資料

在 Laravel 中，有兩種主要方式可以處理 session 資料：全域 `session` 輔助函數以及透過 `Request` 實例。首先，讓我們看看如何透過 `Request` 實例存取 session，此實例可以在路由閉包或控制器方法上進行型別提示。請記住，控制器方法的依賴會透過 Laravel [服務容器](/docs/{{version}}/container)自動注入：

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

當您從 session 中擷取一個項目時，您也可以將預設值作為第二個引數傳入 `get` 方法。如果 session 中不存在指定的鍵，則會傳回此預設值。如果您將一個閉包作為預設值傳給 `get` 方法，且請求的鍵不存在，則該閉包將會被執行並傳回其結果：

    $value = $request->session()->get('key', 'default');

    $value = $request->session()->get('key', function () {
        return 'default';
    });

<a name="the-global-session-helper"></a>
#### 全域 Session 輔助函數

您也可以使用全域 `session` PHP 函數來擷取與儲存 session 中的資料。當 `session` 輔助函數以單一字串引數呼叫時，它將傳回該 session 鍵的值。當該輔助函數以鍵/值配對陣列呼叫時，這些值將會儲存至 session 中：

    Route::get('/home', function () {
        // Retrieve a piece of data from the session...
        $value = session('key');

        // Specifying a default value...
        $value = session('key', 'default');

        // Store a piece of data in the session...
        session(['key' => 'value']);
    });

> [!NOTE]
> 透過 HTTP 請求實例使用 session 與使用全域 `session` 輔助函數之間幾乎沒有實質差異。這兩種方法都可以透過 `assertSessionHas` 方法進行[測試](/docs/{{version}}/testing)，此方法可在您所有的測試案例中使用。

<a name="retrieving-all-session-data"></a>
#### 擷取所有 Session 資料

如果您想擷取 session 中所有的資料，可以使用 `all` 方法：

    $data = $request->session()->all();

<a name="retrieving-a-portion-of-the-session-data"></a>
#### 擷取部分 Session 資料

`only` 與 `except` 方法可用於擷取部分 session 資料：

    $data = $request->session()->only(['username', 'email']);

    $data = $request->session()->except(['username', 'email']);

<a name="determining-if-an-item-exists-in-the-session"></a>
#### 判斷項目是否存在於 Session 中

要判斷項目是否存在於 session 中，可以使用 `has` 方法。如果該項目存在且不為 `null`，則 `has` 方法會傳回 `true`：

    if ($request->session()->has('users')) {
        // ...
    }

要判斷項目是否存在於 session 中，即使其值為 `null`，您可以使用 `exists` 方法：

    if ($request->session()->exists('users')) {
        // ...
    }

要判斷項目是否不存在於 session 中，您可以使用 `missing` 方法。如果該項目不存在，則 `missing` 方法會傳回 `true`：

    if ($request->session()->missing('users')) {
        // ...
    }

<a name="storing-data"></a>
### 儲存資料

要將資料儲存到 session 中，您通常會使用請求實例的 `put` 方法或全域 `session` 輔助函數：

    // Via a request instance...
    $request->session()->put('key', 'value');

    // Via the global "session" helper...
    session(['key' => 'value']);

<a name="pushing-to-array-session-values"></a>
#### 將值推入 Session 陣列中

`push` 方法可用於將新值推入作為陣列的 session 值中。例如，如果 `user.teams` 鍵包含一個團隊名稱陣列，您可以如此將新值推入該陣列：

    $request->session()->push('user.teams', 'developers');

<a name="retrieving-deleting-an-item"></a>
#### 擷取並刪除項目

`pull` 方法將在單一語句中擷取並刪除 session 中的一個項目：

    $value = $request->session()->pull('key', 'default');

<a name="incrementing-and-decrementing-session-values"></a>
#### 遞增與遞減 Session 值

如果您的 session 資料中包含您希望遞增或遞減的整數，您可以使用 `increment` 與 `decrement` 方法：

    $request->session()->increment('count');

    $request->session()->increment('count', $incrementBy = 2);

    $request->session()->decrement('count');

    $request->session()->decrement('count', $decrementBy = 2);

<a name="flash-data"></a>
### 快閃資料

有時您可能希望將項目儲存到 session 中供下一個請求使用。您可以使用 `flash` 方法來實現。使用此方法儲存到 session 中的資料將立即可用，並在後續的 HTTP 請求期間可用。在後續的 HTTP 請求之後，快閃資料將會被刪除。快閃資料主要用於短期狀態訊息：

    $request->session()->flash('status', 'Task was successful!');

如果您需要將快閃資料保留多個請求，您可以使用 `reflash` 方法，它會將所有快閃資料保留給額外的請求。如果您只需要保留特定的快閃資料，您可以使用 `keep` 方法：

    $request->session()->reflash();

    $request->session()->keep(['username', 'email']);

如果您僅需要在目前請求中保留快閃資料，您可以使用 `now` 方法：

    $request->session()->now('status', 'Task was successful!');

<a name="deleting-data"></a>
### 刪除資料

`forget` 方法將從 session 中刪除一筆資料。如果您想刪除 session 中的所有資料，可以使用 `flush` 方法：

    // Forget a single key...
    $request->session()->forget('name');

    // Forget multiple keys...
    $request->session()->forget(['name', 'status']);

    $request->session()->flush();

<a name="regenerating-the-session-id"></a>
### 重新產生 Session ID

重新產生 session ID 通常是為了防止惡意使用者利用 [session 綁定](https://owasp.org/www-community/attacks/Session_fixation) 攻擊您的應用程式。

如果您使用 Laravel 的任何[應用程式入門套件](/docs/{{version}}/starter-kits)或 [Laravel Fortify](/docs/{{version}}/fortify)，Laravel 會在身份驗證期間自動重新產生 session ID；但是，如果您需要手動重新產生 session ID，您可以使用 `regenerate` 方法：

    $request->session()->regenerate();

如果您需要在單一語句中重新產生 session ID 並刪除 session 中的所有資料，您可以使用 `invalidate` 方法：

    $request->session()->invalidate();

<a name="session-blocking"></a>
## Session 阻擋

> [!WARNING]  
> 要使用 Session 阻擋功能，你的應用程式必須使用支援 [原子鎖](/docs/{{version}}/cache#atomic-locks) 的快取驅動器。目前，這些快取驅動器包括 `memcached`、`dynamodb`、`redis`、`mongodb` (包含在官方的 `mongodb/laravel-mongodb` 套件中)、`database`、`file` 及 `array` 驅動器。此外，你不能使用 `cookie` Session 驅動器。

預設情況下，Laravel 允許使用相同 Session 的請求同時執行。例如，如果你使用 JavaScript HTTP 函式庫向你的應用程式發出兩個 HTTP 請求，它們將會同時執行。對於許多應用程式來說，這不是問題；然而，在少數對兩個不同應用程式端點發出並行請求且這兩個端點都會寫入資料到 Session 的應用程式中，可能會發生 Session 資料遺失。

為了緩解這個問題，Laravel 提供了功能讓你限制指定 Session 的並行請求。首先，你可以在路由定義上串接 `block` 方法。在這個範例中，對 `/profile` 端點傳入的請求將會取得一個 Session 鎖定。當這個鎖定被持有時，任何共享相同 Session ID 且傳入 `/profile` 或 `/order` 端點的請求，都將會等待第一個請求完成執行後，才會繼續執行：

    Route::post('/profile', function () {
        // ...
    })->block($lockSeconds = 10, $waitSeconds = 10);

    Route::post('/order', function () {
        // ...
    })->block($lockSeconds = 10, $waitSeconds = 10);

`block` 方法接受兩個選用引數。`block` 方法接受的第一個引數是 Session 鎖定在被釋放前應被持有的最長時間（秒）。當然，如果請求在此時間之前完成執行，鎖定將會提早被釋放。

`block` 方法接受的第二個引數是請求在嘗試取得 Session 鎖定時應等待的秒數。如果請求在給定的秒數內無法取得 Session 鎖定，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`。

如果這兩個引數都沒有傳入，鎖定將會被持有最多 10 秒，而請求在嘗試取得鎖定時也將等待最多 10 秒：

    Route::post('/profile', function () {
        // ...
    })->block();


<a name="adding-custom-session-drivers"></a>
## 新增自訂 Session 驅動器


<a name="implementing-the-driver"></a>
### 實作驅動器

如果現有的 Session 驅動器都不符合你應用程式的需求，Laravel 讓你可以編寫自己的 Session 處理器。你的自訂 Session 驅動器應實作 PHP 內建的 `SessionHandlerInterface`。此介面只包含幾個簡單的方法。一個 Stub 化的 MongoDB 實作範例如下：

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

由於 Laravel 沒有提供預設的目錄來存放你的擴充功能。你可以將它們放置在你喜歡的任何地方。在這個範例中，我們建立了一個 `Extensions` 目錄來存放 `MongoSessionHandler`。

由於這些方法的用途不容易理解，以下是每個方法的用途概述：

<div class="content-list" markdown="1">

- `open` 方法通常用於基於檔案的 Session 儲存系統。由於 Laravel 內建了 `file` Session 驅動器，你很少需要在此方法中放入任何內容。你可以簡單地讓此方法保持空白。
- `close` 方法與 `open` 方法一樣，通常也可以被忽略。對於大多數驅動器來說，它不是必需的。
- `read` 方法應傳回與給定 `$sessionId` 相關聯的 Session 資料的字串版本。在你的驅動器中擷取或儲存 Session 資料時，無需進行任何序列化或其他編碼，因為 Laravel 會為你執行序列化。
- `write` 方法應將與 `$sessionId` 相關聯的給定 `$data` 字串寫入某個持久性儲存系統，例如 MongoDB 或你選擇的其他儲存系統。同樣地，你不應執行任何序列化 – Laravel 會已經為你處理好了。
- `destroy` 方法應從持久性儲存中移除與 `$sessionId` 相關聯的資料。
- `gc` 方法應銷毀所有比給定 `$lifetime`（一個 UNIX 時間戳記）更舊的 Session 資料。對於像 Memcached 和 Redis 這樣會自動過期的系統，此方法可以留空。

</div>


<a name="registering-the-driver"></a>
### 註冊驅動器

一旦你的驅動器已實作，你就可以將它註冊到 Laravel。要將額外的驅動器加入到 Laravel 的 Session 後端，你可以使用 `Session` [Facade](/docs/{{version}}/facades) 提供的 `extend` 方法。你應該從 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫 `extend` 方法。你可以從現有的 `App\Providers\AppServiceProvider` 中執行此操作，或者建立一個全新的提供者：

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

一旦 Session 驅動器註冊完成，你就可以使用 `SESSION_DRIVER` 環境變數或在應用程式的 `config/session.php` 設定檔中，將 `mongo` 驅動器指定為你應用程式的 Session 驅動器。