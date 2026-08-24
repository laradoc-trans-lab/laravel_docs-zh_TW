# HTTP Session

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動需求](#driver-prerequisites)
- [操作 Session](#interacting-with-the-session)
    - [取得資料](#retrieving-data)
    - [儲存資料](#storing-data)
    - [快閃資料](#flash-data)
    - [刪除資料](#deleting-data)
    - [重新產生 Session ID](#regenerating-the-session-id)
- [Session 快取](#session-cache)
- [Session 阻塞](#session-blocking)
- [新增自訂 Session 驅動](#adding-custom-session-drivers)
    - [實作驅動](#implementing-the-driver)
    - [註冊驅動](#registering-the-driver)

<a name="introduction"></a>
## 簡介

由於 HTTP 驅動的應用程式是無狀態的（Stateless），因此 Session 提供了一種跨多個請求儲存使用者相關資訊的方法。這些使用者資訊通常會存放在持久化儲存空間／後端中，以便在後續的請求中存取。

Laravel 內建了各種 Session 後端支援，並可透過直覺且統一的 API 進行存取。內建支援包含熱門的後端，如 [Memcached](https://memcached.org)、[Redis](https://redis.io) 以及資料庫。


<a name="configuration"></a>
### 設定

你的應用程式 Session 設定檔儲存於 `config/session.php`。請務必查看此檔案中提供的可用選項。預設情況下，Laravel 被設定為使用 `database` Session 驅動。

Session 的 `driver` 設定選項定義了每個請求的 Session 資料將會儲存在何處。Laravel 包含了多種驅動：

<div class="content-list" markdown="1">

- `file` - Session 儲存於 `storage/framework/sessions`。
- `cookie` - Session 儲存於安全且經過加密的 Cookie 中。
- `database` - Session 儲存於關聯式資料庫中。
- `memcached` / `redis` - Session 儲存於這些快速且基於快取的儲存區之一。
- `dynamodb` - Session 儲存於 AWS DynamoDB。
- `array` - Session 儲存於 PHP 陣列中，不會被持久化保留。

</div>

> [!NOTE]
> array 驅動主要用於[測試](/docs/{{version}}/testing)期間，可防止儲存在 Session 中的資料被持久化保留。


<a name="driver-prerequisites"></a>
### 驅動需求


<a name="database"></a>
#### 資料庫

使用 `database` Session 驅動時，你需要確保有一個資料庫表來存放 Session 資料。通常，這已經包含在 Laravel 預設的 `0001_01_01_000000_create_users_table.php` [資料庫遷移](/docs/{{version}}/migrations)中；然而，若出於任何原因你沒有 `sessions` 表，你可以使用 `make:session-table` Artisan 命令來產生這個遷移：

```shell
php artisan make:session-table

php artisan migrate
```


<a name="redis"></a>
#### Redis

在 Laravel 中使用 Redis Session 之前，你需要透過 PECL 安裝 PhpRedis PHP 擴充套件，或是透過 Composer 安裝 `predis/predis` 套件。關於設定 Redis 的更多資訊，請參考 Laravel 的 [Redis 文件](/docs/{{version}}/redis#configuration)。

> [!NOTE]
> 可以使用 `SESSION_CONNECTION` 環境變數、或是 `session.php` 設定檔中的 `connection` 選項，來指定用於 Session 儲存的 Redis 連線。

<a name="interacting-with-the-session"></a>
## 操作 Session

<a name="retrieving-data"></a>
### 取得資料

在 Laravel 中，主要有兩種操作 Session 資料的方式：全域 `session` 輔助函式以及透過 `Request` 實例。首先，讓我們看看如何透過 `Request` 實例存取 Session，你可以在路由閉包或控制器方法上對其進行型別提示（Type-hint）。請記住，控制器的方法依賴會透過 Laravel 的[服務容器](/docs/{{version}}/container)自動注入：

```php
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
```

當你從 Session 取得項目時，也可以傳送預設值作為 `get` 方法的第二個引數。若指定的金鑰不存在於 Session 中，將會傳回該預設值。如果你傳送一個閉包作為 `get` 方法的預設值，且請求的金鑰不存在，則該閉包將會被執行並傳回其結果：

```php
$value = $request->session()->get('key', 'default');

$value = $request->session()->get('key', function () {
    return 'default';
});
```

<a name="the-global-session-helper"></a>
#### 全域 Session 輔助函式

你也可以使用全域的 `session` PHP 函式來取得與儲存 Session 中的資料。當呼叫 `session` 輔助函式並傳入單一字串引數時，它會傳回該 Session 金鑰的值。當傳入包含金鑰與值的陣列時，這些值將會被儲存在 Session 中：

```php
Route::get('/home', function () {
    // Retrieve a piece of data from the session...
    $value = session('key');

    // Specifying a default value...
    $value = session('key', 'default');

    // Store a piece of data in the session...
    session(['key' => 'value']);
});
```

> [!NOTE]
> 透過 HTTP 請求實例與透過全域 `session` 輔助函式來使用 Session，在實務上幾乎沒有差異。兩種方法都可以透過所有測試案例中皆可使用的 `assertSessionHas` 方法進行[測試](/docs/{{version}}/testing)。

<a name="retrieving-all-session-data"></a>
#### 取得所有 Session 資料

如果你想要取得 Session 中的所有資料，可以使用 `all` 方法：

```php
$data = $request->session()->all();
```

<a name="retrieving-a-portion-of-the-session-data"></a>
#### 取得部分 Session 資料

`only` 和 `except` 方法可用於取得 Session 資料的子集：

```php
$data = $request->session()->only(['username', 'email']);

$data = $request->session()->except(['username', 'email']);
```

<a name="determining-if-an-item-exists-in-the-session"></a>
#### 判斷 Session 中是否存在某個項目

若要判斷 Session 中是否存在某個項目，可以使用 `has` 方法。若該項目存在且不為 `null`，`has` 方法會傳回 `true`：

```php
if ($request->session()->has('users')) {
    // ...
}
```

若要判斷 Session 中是否存在某個項目（即使其值為 `null`），可以使用 `exists` 方法：

```php
if ($request->session()->exists('users')) {
    // ...
}
```

若要判斷 Session 中是否不存在某個項目，可以使用 `missing` 方法。若項目不存在，`missing` 方法會傳回 `true`：

```php
if ($request->session()->missing('users')) {
    // ...
}
```

<a name="storing-data"></a>
### 儲存資料

若要在 Session 中儲存資料，通常會使用請求實例的 `put` 方法或是全域 `session` 輔助函式：

```php
// Via a request instance...
$request->session()->put('key', 'value');

// Via the global "session" helper...
session(['key' => 'value']);
```

<a name="pushing-to-array-session-values"></a>
#### 推入 Session 陣列值

`push` 方法可用於將新值推入型別為陣列的 Session 值中。例如，若 `user.teams` 金鑰包含一個團隊名稱陣列，你可以像這樣將新值推入該陣列：

```php
$request->session()->push('user.teams', 'developers');
```

<a name="retrieving-deleting-an-item"></a>
#### 取得並刪除項目

`pull` 方法可以用單一語句從 Session 中取得並刪除項目：

```php
$value = $request->session()->pull('key', 'default');
```

<a name="incrementing-and-decrementing-session-values"></a>
#### 增加與減少 Session 值

如果你的 Session 資料包含想要增加或減少的整數，可以使用 `increment` 和 `decrement` 方法：

```php
$request->session()->increment('count');

$request->session()->increment('count', $incrementBy = 2);

$request->session()->decrement('count');

$request->session()->decrement('count', $decrementBy = 2);
```

<a name="flash-data"></a>
### 快閃資料

有時你可能希望在 Session 中儲存項目以供下一次請求使用。你可以使用 `flash` 方法來達到此目的。使用此方法儲存在 Session 中的資料將會立即生效，並保留至下一次 HTTP 請求期間。在下一次 HTTP 請求結束後，快閃資料將會被刪除。快閃資料主要適用於短暫發送的狀態訊息：

```php
$request->session()->flash('status', 'Task was successful!');
```

如果你需要將快閃資料保留多個請求，可以使用 `reflash` 方法，這會將所有快閃資料再保留一次請求。如果你只需要保留特定的快閃資料，可以使用 `keep` 方法：

```php
$request->session()->reflash();

$request->session()->keep(['username', 'email']);
```

若要僅將快閃資料保留在當前請求中，可以使用 `now` 方法：

```php
$request->session()->now('status', 'Task was successful!');
```

<a name="deleting-data"></a>
### 刪除資料

`forget` 方法會從 Session 中移除一筆資料。如果你想移除 Session 中的所有資料，可以使用 `flush` 方法：

```php
// Forget a single key...
$request->session()->forget('name');

// Forget multiple keys...
$request->session()->forget(['name', 'status']);

$request->session()->flush();
```

<a name="regenerating-the-session-id"></a>
### 重新產生 Session ID

重新產生 Session ID 通常是為了防止惡意使用者對你的應用程式進行 [Session 固定 (Session Fixation)](https://owasp.org/www-community/attacks/Session_fixation) 攻擊。

如果你使用的是 Laravel [應用程式入門套件](/docs/{{version}}/starter-kits)之一或是 [Laravel Fortify](/docs/{{version}}/fortify)，Laravel 會在認證期間自動重新產生 Session ID；不過，如果你需要手動重新產生 Session ID，可以使用 `regenerate` 方法：

```php
$request->session()->regenerate();
```

如果你需要重新產生 Session ID 並在單一語句中清除 Session 中的所有資料，可以使用 `invalidate` 方法：

```php
$request->session()->invalidate();
```

<a name="session-cache"></a>
## Session 快取

Laravel 的 Session 快取提供了一種便捷的方式來快取作用域限定於個別使用者 Session 的資料。與全域應用程式快取不同，Session 快取資料會自動依據每個 Session 進行隔離，並在 Session 過期或銷毀時自動清理。Session 快取支援所有熟悉的 [Laravel 快取方法](/docs/{{version}}/cache)，例如 `get`、`put`、`remember`、`forget` 等等，但作用域僅限於當前的 Session。

Session 快取非常適合用於儲存臨時且針對特定使用者的資料，這些資料您希望在同一 Session 內跨多個請求保留，但不需要永久儲存。這包括表單資料、臨時計算結果、API 回應，或任何其他應該與特定使用者 Session 綁定的短暫資料。

您可以透過 Session 上的 `cache` 方法存取 Session 快取：

```php
$discount = $request->session()->cache()->get('discount');

$request->session()->cache()->put(
    'discount', 10, now()->plus(minutes: 5)
);
```

關於 Laravel 快取方法的更多資訊，請參考[快取文件](/docs/{{version}}/cache)。


<a name="session-blocking"></a>
## Session 阻塞

> [!WARNING]
> 若要使用 Session 阻塞功能，您的應用程式必須使用支援[原子鎖](/docs/{{version}}/cache#atomic-locks)的快取驅動。目前這些快取驅動包括 `memcached`、`dynamodb`、`redis`、`mongodb`（包含於官方的 `mongodb/laravel-mongodb` 套件中）、`database`、`file` 與 `array` 驅動。此外，您不能使用 `cookie` Session 驅動。

預設情況下，Laravel 允許使用相同 Session 的請求同時執行。例如，如果您使用 JavaScript HTTP 函式庫向應用程式發送兩個 HTTP 請求，它們將會同時執行。對許多應用程式來說，這不是問題；然而，在少數情況下，若應用程式向兩個不同的端點同時發送請求且這兩個端點都會寫入資料到 Session 時，可能會發生 Session 資料遺失的問題。

為了解決這個問題，Laravel 提供了允許您限制指定 Session 之同時請求數量的功能。若要開始使用，您只需在路由定義上鏈結 `block` 方法即可。在此範例中，對 `/profile` 端點傳入的請求將會取得一個 Session 鎖。當持有此鎖時，任何共享相同 Session ID 且進入 `/profile` 或 `/order` 端點的傳入請求，都會等待第一個請求執行完畢後才繼續執行：

```php
Route::post('/profile', function () {
    // ...
})->block($lockSeconds = 10, $waitSeconds = 10);

Route::post('/order', function () {
    // ...
})->block($lockSeconds = 10, $waitSeconds = 10);
```

`block` 方法接受兩個選填引數。`block` 方法接受的第一個引數是 Session 鎖在釋放前應持有的最大秒數。當然，如果請求在此時間之前完成執行，鎖將會提早釋放。

`block` 方法接受的第二個引數是請求在嘗試取得 Session 鎖時應等待的秒數。如果請求無法在指定的秒數內取得 Session 鎖，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException` 異常。

如果兩個引數都沒有傳入，鎖將最多保持 10 秒，且請求在嘗試取得鎖時最多會等待 10 秒：

```php
Route::post('/profile', function () {
    // ...
})->block();
```


<a name="adding-custom-session-drivers"></a>
## 新增自訂 Session 驅動


<a name="implementing-the-driver"></a>
### 實作驅動

如果現有的 Session 驅動都無法滿足您的應用程式需求，Laravel 允許您撰寫自己的 Session 處理常式。您的自訂 Session 驅動應該實作 PHP 內建的 `SessionHandlerInterface`。這個介面僅包含幾個簡單的方法。一個存根 (Stubbed) 的 MongoDB 實作如下所示：

```php
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
```

由於 Laravel 沒有包含用於存放擴充功能的預設目錄，您可以隨意將它們放置在任何您喜歡的地方。在此範例中，我們建立了一個 `Extensions` 目錄來存放 `MongoSessionHandler`。

由於這些方法的用途可能無法一目瞭然，以下是每個方法用途的概覽：

<div class="content-list" markdown="1">

- `open` 方法通常用於基於檔案的 Session 儲存系統。由於 Laravel 隨附了 `file` Session 驅動，您很少需要在此方法中撰寫任何內容。您可以直接保持此方法為空白。
- `close` 方法與 `open` 方法類似，通常也可以被忽略。對於大多數驅動程式而言，並不需要此方法。
- `read` 方法應回傳與給定 `$sessionId` 相關聯的 Session 資料字串版本。在驅動中取得或儲存 Session 資料時，不需要進行任何序列化或其他編碼，因為 Laravel 會為您執行序列化。
- `write` 方法應將給定與 `$sessionId` 關聯的 `$data` 字串寫入某個持久化儲存系統，例如 MongoDB 或您選擇的其他儲存系統。同樣地，您不應執行任何序列化——Laravel 已經為您處理好了。
- `destroy` 方法應從持久化儲存中移除與 `$sessionId` 關聯的資料。
- `gc` 方法應銷毀所有比給定 `$lifetime`（即 UNIX 時間戳記）更舊的 Session 資料。對於像是 Memcached 與 Redis 這類會自動過期的系統，此方法可以保持空白。

</div>


<a name="registering-the-driver"></a>
### 註冊驅動

當您的驅動實作完成後，即可將其註冊至 Laravel。若要向 Laravel 的 Session 後端新增其他驅動，您可以使用 `Session` [Facade](/docs/{{version}}/facades) 所提供的 `extend` 方法。您應該在[服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫 `extend` 方法。您可以在現有的 `App\Providers\AppServiceProvider` 中進行此操作，或是建立一個全新的服務提供者：

```php
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
```

Session 驅動註冊完畢後，您便可以使用 `SESSION_DRIVER` 環境變數或在應用程式的 `config/session.php` 設定檔中指定 `mongo` 驅動作為您應用程式的 Session 驅動。