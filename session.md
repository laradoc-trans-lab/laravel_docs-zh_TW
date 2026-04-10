# HTTP 工作階段

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動程式先決條件](#driver-prerequisites)
- [與工作階段互動](#interacting-with-the-session)
    - [擷取資料](#retrieving-data)
    - [儲存資料](#storing-data)
    - [快閃資料](#flash-data)
    - [刪除資料](#deleting-data)
    - [重新產生工作階段 ID](#regenerating-the-session-id)
- [工作階段快取](#session-cache)
- [工作階段阻擋](#session-blocking)
- [新增自訂工作階段驅動程式](#adding-custom-session-drivers)
    - [實作驅動程式](#implementing-the-driver)
    - [註冊驅動程式](#registering-the-driver)

<a name="introduction"></a>
## 簡介

由於 HTTP 驅動的應用程式是無狀態的，因此工作階段提供了一種跨多個請求儲存使用者資訊的方式。這些使用者資訊通常會被放置在一個持久儲存 / 後端中，以便從後續請求中存取。

Laravel 隨附了多種工作階段後端，可透過表達性且統一的 API 進行存取。支援流行的後端，例如 [Memcached](https://memcached.org)、[Redis](https://redis.io) 和資料庫。


<a name="configuration"></a>
### 設定

您的應用程式工作階段設定檔儲存在 `config/session.php`。請務必檢閱此檔案中可用的選項。預設情況下，Laravel 配置為使用 `database` 工作階段驅動程式。

工作階段的 `driver` 設定選項定義了每個請求的工作階段資料將儲存在何處。Laravel 包含多種驅動程式：

<div class="content-list" markdown="1">

- `file` - 工作階段儲存在 `storage/framework/sessions`。
- `cookie` - 工作階段儲存在安全、加密的 cookie 中。
- `database` - 工作階段儲存在關聯式資料庫中。
- `memcached` / `redis` - 工作階段儲存在這些快速、基於快取的儲存之一。
- `dynamodb` - 工作階段儲存在 AWS DynamoDB 中。
- `array` - 工作階段儲存在 PHP 陣列中，且不會持久化。

</div>

> [!NOTE]
> array 驅動程式主要用於 [測試](/docs/{{version}}/testing) 期間，並防止儲存在工作階段中的資料持久化。


<a name="driver-prerequisites"></a>
### 驅動程式先決條件


<a name="database"></a>
#### 資料庫

使用 `database` 工作階段驅動程式時，您需要確保擁有一個資料庫資料表來包含工作階段資料。通常，這會包含在 Laravel 預設的 `0001_01_01_000000_create_users_table.php` [資料庫遷移](/docs/{{version}}/migrations) 中；但是，如果由於任何原因您沒有 `sessions` 資料表，您可以使用 `make:session-table` Artisan 指令來產生此遷移：

```shell
php artisan make:session-table

php artisan migrate
```


<a name="redis"></a>
#### Redis

在使用 Laravel 的 Redis 工作階段之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或透過 Composer 安裝 `predis/predis` 套件 (~1.0)。有關配置 Redis 的更多資訊，請查閱 Laravel 的 [Redis 文件](/docs/{{version}}/redis#configuration)。

> [!NOTE]
> `SESSION_CONNECTION` 環境變數，或 `session.php` 設定檔中的 `connection` 選項，可用於指定用於工作階段儲存的 Redis 連線。

<a name="interacting-with-the-session"></a>
## 與工作階段互動


<a name="retrieving-data"></a>
### 擷取資料

在 Laravel 中，有兩種主要方式可以處理工作階段資料：全域的 `session` 輔助函式以及透過 `Request` 實例。首先，讓我們看看如何透過 `Request` 實例來存取工作階段，此實例可以在路由閉包或控制器方法上進行型別提示。請記住，控制器方法的依賴項會透過 Laravel 的 [服務容器](/docs/{{version}}/container) 自動注入：

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

當您從工作階段中擷取項目時，您也可以將預設值作為第二個引數傳遞給 `get` 方法。如果指定的鍵在工作階段中不存在，則會傳回此預設值。如果您將閉包作為預設值傳遞給 `get` 方法，且所請求的鍵不存在，則該閉包將會執行並傳回其結果：

```php
$value = $request->session()->get('key', 'default');

$value = $request->session()->get('key', function () {
    return 'default';
});
```


<a name="the-global-session-helper"></a>
#### 全域 Session 輔助函式

您也可以使用全域的 `session` PHP 函式來從工作階段中擷取和儲存資料。當 `session` 輔助函式以單一字串引數呼叫時，它將傳回該工作階段鍵的值。當該輔助函式以鍵／值陣列呼叫時，這些值將會儲存在工作階段中：

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
> 使用 HTTP 請求實例來存取工作階段，與使用全域 `session` 輔助函式之間，實際上沒有太大差異。這兩種方法都可以透過 `assertSessionHas` 方法進行 [測試](/docs/{{version}}/testing)，該方法適用於您所有的測試案例。


<a name="retrieving-all-session-data"></a>
#### 擷取所有工作階段資料

如果您想擷取工作階段中的所有資料，可以使用 `all` 方法：

```php
$data = $request->session()->all();
```


<a name="retrieving-a-portion-of-the-session-data"></a>
#### 擷取部分工作階段資料

`only` 和 `except` 方法可用於擷取工作階段資料的子集：

```php
$data = $request->session()->only(['username', 'email']);

$data = $request->session()->except(['username', 'email']);
```


<a name="determining-if-an-item-exists-in-the-session"></a>
#### 判斷工作階段中是否存在項目

若要判斷某個項目是否存在於工作階段中，您可以使用 `has` 方法。如果該項目存在且不為 `null`，`has` 方法會傳回 `true`：

```php
if ($request->session()->has('users')) {
    // ...
}
```

若要判斷某個項目是否存在於工作階段中，即使其值為 `null`，您也可以使用 `exists` 方法：

```php
if ($request->session()->exists('users')) {
    // ...
}
```

若要判斷某個項目是否不存在於工作階段中，您可以使用 `missing` 方法。如果該項目不存在，`missing` 方法會傳回 `true`：

```php
if ($request->session()->missing('users')) {
    // ...
}
```


<a name="storing-data"></a>
### 儲存資料

若要將資料儲存到工作階段中，您通常會使用請求實例的 `put` 方法或全域 `session` 輔助函式：

```php
// Via a request instance...
$request->session()->put('key', 'value');

// Via the global "session" helper...
session(['key' => 'value']);
```


<a name="pushing-to-array-session-values"></a>
#### 推入陣列型工作階段值

`push` 方法可用於將新值推入作為陣列的工作階段值。例如，如果 `user.teams` 鍵包含一個團隊名稱陣列，您可以像這樣將新值推入該陣列：

```php
$request->session()->push('user.teams', 'developers');
```


<a name="retrieving-deleting-an-item"></a>
#### 擷取並刪除項目

`pull` 方法將在一個陳述式中從工作階段擷取並刪除一個項目：

```php
$value = $request->session()->pull('key', 'default');
```


<a name="incrementing-and-decrementing-session-values"></a>
#### 遞增和遞減工作階段值

如果您的工作階段資料包含一個您希望遞增或遞減的整數，您可以使用 `increment` 和 `decrement` 方法：

```php
$request->session()->increment('count');

$request->session()->increment('count', $incrementBy = 2);

$request->session()->decrement('count');

$request->session()->decrement('count', $decrementBy = 2);
```


<a name="flash-data"></a>
### 快閃資料

有時您可能希望在工作階段中儲存項目以供下一次請求使用。您可以使用 `flash` 方法來實現。使用此方法儲存在工作階段中的資料將立即可用，並在隨後的 HTTP 請求期間可用。在隨後的 HTTP 請求之後，快閃資料將被刪除。快閃資料主要用於短暫的狀態訊息：

```php
$request->session()->flash('status', 'Task was successful!');
```

如果您需要讓您的快閃資料在數個請求中保持持久，您可以使用 `reflash` 方法，這將使所有快閃資料再保留一個額外的請求。如果您只需要保留特定的快閃資料，可以使用 `keep` 方法：

```php
$request->session()->reflash();

$request->session()->keep(['username', 'email']);
```

若只想讓快閃資料在當前請求中保持持久，您可以使用 `now` 方法：

```php
$request->session()->now('status', 'Task was successful!');
```


<a name="deleting-data"></a>
### 刪除資料

`forget` 方法將從工作階段中移除一塊資料。如果您想從工作階段中移除所有資料，可以使用 `flush` 方法：

```php
// Forget a single key...
$request->session()->forget('name');

// Forget multiple keys...
$request->session()->forget(['name', 'status']);

$request->session()->flush();
```


<a name="regenerating-the-session-id"></a>
### 重新產生工作階段 ID

重新產生工作階段 ID 通常是為了防止惡意使用者利用 [工作階段固定攻擊](https://owasp.org/www-community/attacks/Session_fixation) 來攻擊您的應用程式。

如果您正在使用 Laravel 的 [應用程式入門套件](/docs/{{version}}/starter-kits) 或 [Laravel Fortify](/docs/{{version}}/fortify)，Laravel 會在認證期間自動重新產生工作階段 ID；但是，如果您需要手動重新產生工作階段 ID，可以使用 `regenerate` 方法：

```php
$request->session()->regenerate();
```

如果您需要在一個陳述式中重新產生工作階段 ID 並移除工作階段中的所有資料，您可以使用 `invalidate` 方法：

```php
$request->session()->invalidate();
```

<a name="session-cache"></a>
## 工作階段快取

Laravel 的工作階段快取提供了一種便捷的方式來快取限定於個別使用者工作階段的資料。與全域應用程式快取不同，工作階段快取資料會自動依每個工作階段進行隔離，並在工作階段過期或被銷毀時自動清除。工作階段快取支援所有熟悉的 [Laravel 快取方法](/docs/{{version}}/cache)，例如 `get`、`put`、`remember`、`forget` 等，但其作用範圍僅限於當前工作階段。

工作階段快取非常適合用於儲存臨時的、使用者專屬的資料，這些資料需要在同一個工作階段中的多個請求之間保持持久，但不需要永久儲存。這包括表單資料、臨時計算結果、API 回應，或任何其他應與特定使用者工作階段綁定的暫時性資料。

您可以透過工作階段上的 `cache` 方法來存取工作階段快取：

```php
$discount = $request->session()->cache()->get('discount');

$request->session()->cache()->put(
    'discount', 10, now()->plus(minutes: 5)
);
```

有關 Laravel 快取方法的更多資訊，請查閱[快取文件](/docs/{{version}}/cache)。

<a name="session-blocking"></a>
## 工作階段阻擋

> [!WARNING]
> 若要使用工作階段阻擋功能，您的應用程式必須使用支援[原子鎖定](/docs/{{version}}/cache#atomic-locks)的快取驅動程式。目前，這些快取驅動程式包括 `memcached`、`dynamodb`、`redis`、`mongodb` (包含於官方的 `mongodb/laravel-mongodb` 套件中)、`database`、`file` 和 `array` 驅動程式。此外，您不得使用 `cookie` 工作階段驅動程式。

預設情況下，Laravel 允許使用相同工作階段的請求並行執行。因此，舉例來說，如果您使用 JavaScript HTTP 函式庫向應用程式發出兩個 HTTP 請求，它們將會同時執行。對於許多應用程式來說，這不是問題；然而，在少數應用程式中，如果同時向兩個不同的應用程式端點發出請求，且兩者都寫入資料到工作階段，則可能會發生工作階段資料遺失。

為了緩解這個問題，Laravel 提供了功能，讓您可以限制給定工作階段的並行請求。要開始使用，您只需將 `block` 方法鏈接到您的路由定義上。在此範例中，對 `/profile` 端點的傳入請求將會取得工作階段鎖定。當此鎖定被持有時，任何對 `/profile` 或 `/order` 端點的傳入請求，如果共用相同的工作階段 ID，將會等待第一個請求執行完成後才繼續執行：

```php
Route::post('/profile', function () {
    // ...
})->block($lockSeconds = 10, $waitSeconds = 10);

Route::post('/order', function () {
    // ...
})->block($lockSeconds = 10, $waitSeconds = 10);
```

`block` 方法接受兩個可選參數。`block` 方法接受的第一個參數是工作階段鎖定在釋放前應保持的最大秒數。當然，如果請求在此時間之前完成執行，鎖定將會更早釋放。

`block` 方法接受的第二個參數是請求在嘗試獲取工作階段鎖定時應該等待的秒數。如果請求無法在給定秒數內獲取工作階段鎖定，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`。

如果這些參數都沒有傳遞，鎖定將最多保持 10 秒，並且請求在嘗試獲取鎖定時將最多等待 10 秒：

```php
Route::post('/profile', function () {
    // ...
})->block();
```

<a name="adding-custom-session-drivers"></a>
## 新增自訂工作階段驅動程式

<a name="implementing-the-driver"></a>
### 實作驅動程式

如果現有的工作階段驅動程式都不符合您的應用程式需求，Laravel 允許您自行編寫工作階段處理器。您的自訂工作階段驅動程式應該實作 PHP 內建的 `SessionHandlerInterface`。這個介面只包含幾個簡單的方法。一個存根化的 MongoDB 實作如下所示：

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

由於 Laravel 不包含用於存放擴充功能預設目錄。您可以將它們放置在任何您喜歡的地方。在這個範例中，我們建立了一個 `Extensions` 目錄來存放 `MongoSessionHandler`。

由於這些方法的用途並非一目瞭然，這裡將概述每個方法的用途：

<div class="content-list" markdown="1">

- `open` 方法通常用於基於檔案的工作階段儲存系統。由於 Laravel 附帶 `file` 工作階段驅動程式，您很少需要在這個方法中放入任何內容。您可以直接將這個方法留空。
- `close` 方法與 `open` 方法一樣，通常也可以忽略。對於大多數驅動程式來說，它並不需要。
- `read` 方法應該傳回與給定 `$sessionId` 相關的工作階段資料的字串版本。在您的驅動程式中擷取或儲存工作階段資料時，無需進行任何序列化或其他編碼，因為 Laravel 會為您執行序列化。
- `write` 方法應該將與 `$sessionId` 相關的給定 `$data` 字串寫入某個持久性儲存系統，例如 MongoDB 或您選擇的其他儲存系統。同樣地，您不應該執行任何序列化 – Laravel 已經為您處理了這個部分。
- `destroy` 方法應該從持久性儲存中移除與 `$sessionId` 相關的資料。
- `gc` 方法應該銷毀所有早於給定 `$lifetime`（一個 UNIX 時間戳）的工作階段資料。對於像 Memcached 和 Redis 這樣會自行過期的系統，這個方法可以留空。

</div>

<a name="registering-the-driver"></a>
### 註冊驅動程式

一旦您的驅動程式實作完成，您就可以將其註冊到 Laravel。要為 Laravel 的工作階段後端新增額外的驅動程式，您可以使用 `Session` [Facade](/docs/{{version}}/facades) 提供的 `extend` 方法。您應該從[服務提供者](/docs/{{version}}/providers)的 `boot` 方法中呼叫 `extend` 方法。您可以從現有的 `App\Providers\AppServiceProvider` 中執行此操作，或者建立一個全新的提供者：

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

一旦工作階段驅動程式被註冊，您就可以使用 `SESSION_DRIVER` 環境變數或在應用程式的 `config/session.php` 配置檔案中，將 `mongo` 驅動程式指定為您的應用程式工作階段驅動程式。