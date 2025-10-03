# HTTP 工作階段

- [簡介](#introduction)
    - [設定](#configuration)
    - [驅動器先決條件](#driver-prerequisites)
- [與工作階段互動](#interacting-with-the-session)
    - [擷取資料](#retrieving-data)
    - [儲存資料](#storing-data)
    - [快閃資料](#flash-data)
    - [刪除資料](#deleting-data)
    - [重新生成工作階段 ID](#regenerating-the-session-id)
- [工作階段快取](#session-cache)
- [工作階段阻擋](#session-blocking)
- [新增客製化工作階段驅動器](#adding-custom-session-drivers)
    - [實作驅動器](#implementing-the-driver)
    - [註冊驅動器](#registering-the-driver)

<a name="introduction"></a>
## 簡介

由於 HTTP 驅動的應用程式是無狀態的，工作階段提供了一種跨多個請求儲存使用者資訊的方式。這些使用者資訊通常被放置在一個持久性的儲存 / 後端中，以便後續請求可以存取。

Laravel 內建了多種工作階段後端，可透過表達性且統一的 API 存取。支援流行的後端，例如 [Memcached](https://memcached.org)、[Redis](https://redis.io) 和資料庫。

<a name="configuration"></a>
### 設定

您的應用程式工作階段設定檔儲存於 `config/session.php`。請務必檢閱此檔案中可用的選項。預設情況下，Laravel 配置為使用 `database` 工作階段驅動器。

工作階段 `driver` 設定選項定義了每個請求的工作階段資料將儲存於何處。Laravel 包含多種驅動器：

<div class="content-list" markdown="1">

- `file` - 工作階段儲存於 `storage/framework/sessions`。
- `cookie` - 工作階段儲存於安全加密的 cookies 中。
- `database` - 工作階段儲存於關聯式資料庫中。
- `memcached` / `redis` - 工作階段儲存於這些基於快取的快速儲存之一。
- `dynamodb` - 工作階段儲存於 AWS DynamoDB。
- `array` - 工作階段儲存於 PHP 陣列中，並且不會持久化。

</div>

> [!NOTE]
> array 驅動器主要用於 [測試](/docs/{{version}}/testing) 期間，並防止儲存於工作階段中的資料被持久化。

<a name="driver-prerequisites"></a>
### 驅動器先決條件

<a name="database"></a>
#### 資料庫

當使用 `database` 工作階段驅動器時，您需要確保擁有一個資料庫資料表來包含工作階段資料。通常，這已包含在 Laravel 預設的 `0001_01_01_000000_create_users_table.php` [資料庫遷移](/docs/{{version}}/migrations) 中；但是，如果由於任何原因您沒有 `sessions` 資料表，您可以使用 `make:session-table` Artisan 指令來產生此遷移：

```shell
php artisan make:session-table

php artisan migrate
```

<a name="redis"></a>
#### Redis

在搭配 Laravel 使用 Redis 工作階段之前，您需要透過 PECL 安裝 PhpRedis PHP 擴充功能，或透過 Composer 安裝 `predis/predis` 套件 (~1.0)。有關配置 Redis 的更多資訊，請查閱 Laravel 的 [Redis 文件](/docs/{{version}}/redis#configuration)。

> [!NOTE]
> `SESSION_CONNECTION` 環境變數，或 `session.php` 設定檔中的 `connection` 選項，可用於指定用於工作階段儲存的 Redis 連線。

<a name="interacting-with-the-session"></a>
## 與工作階段互動

<a name="retrieving-data"></a>
### 擷取資料

Laravel 中處理工作階段資料主要有兩種方式：全域 `session` 輔助函式以及透過 `Request` 實例。首先，讓我們看看如何透過 `Request` 實例存取工作階段，它可以在路由閉包或控制器方法中進行型別提示。請記住，控制器方法依賴項會透過 Laravel [service container](/docs/{{version}}/container) 自動注入：

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

當您從工作階段擷取項目時，您也可以傳遞一個預設值作為 `get` 方法的第二個參數。如果指定的鍵在工作階段中不存在，則會傳回此預設值。如果您傳遞一個閉包作為 `get` 方法的預設值，且請求的鍵不存在，則該閉包將會執行並傳回其結果：

```php
$value = $request->session()->get('key', 'default');

$value = $request->session()->get('key', function () {
    return 'default';
});
```

<a name="the-global-session-helper"></a>
#### 全域 Session 輔助函式

您也可以使用全域 `session` PHP 函式來從工作階段中擷取和儲存資料。當 `session` 輔助函式以單一字串參數呼叫時，它會傳回該工作階段鍵的值。當輔助函式以鍵/值對的陣列呼叫時，這些值將會儲存在工作階段中：

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
> 透過 HTTP 請求實例使用工作階段與使用全域 `session` 輔助函式之間幾乎沒有實際差異。這兩種方法都可以透過所有測試案例中可用的 `assertSessionHas` 方法進行 [測試](/docs/{{version}}/testing)。

<a name="retrieving-all-session-data"></a>
#### 擷取所有工作階段資料

如果您想擷取工作階段中的所有資料，可以使用 `all` 方法：

```php
$data = $request->session()->all();
```

<a name="retrieving-a-portion-of-the-session-data"></a>
#### 擷取部分工作階段資料

`only` 和 `except` 方法可用來擷取工作階段資料的子集：

```php
$data = $request->session()->only(['username', 'email']);

$data = $request->session()->except(['username', 'email']);
```

<a name="determining-if-an-item-exists-in-the-session"></a>
#### 判斷項目是否存在於工作階段中

為了判斷項目是否存在於工作階段中，您可以使用 `has` 方法。如果項目存在且不是 `null`，則 `has` 方法會傳回 `true`：

```php
if ($request->session()->has('users')) {
    // ...
}
```

即使項目值為 `null`，要判斷項目是否存在於工作階段中，您可以使用 `exists` 方法：

```php
if ($request->session()->exists('users')) {
    // ...
}
```

為了判斷項目是否不存在於工作階段中，您可以使用 `missing` 方法。如果項目不存在，則 `missing` 方法會傳回 `true`：

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
#### 推送至陣列工作階段值

`push` 方法可用來將新值推送到工作階段中作為陣列的值。例如，如果 `user.teams` 鍵包含一個團隊名稱陣列，您可以像這樣將新值推送到該陣列中：

```php
$request->session()->push('user.teams', 'developers');
```

<a name="retrieving-deleting-an-item"></a>
#### 擷取並刪除項目

`pull` 方法會以單一陳述式從工作階段中擷取並刪除項目：

```php
$value = $request->session()->pull('key', 'default');
```

<a name="incrementing-and-decrementing-session-values"></a>
#### 遞增與遞減工作階段值

如果您的工作階段資料包含您想遞增或遞減的整數，您可以使用 `increment` 和 `decrement` 方法：

```php
$request->session()->increment('count');

$request->session()->increment('count', $incrementBy = 2);

$request->session()->decrement('count');

$request->session()->decrement('count', $decrementBy = 2);
```

<a name="flash-data"></a>
### 快閃資料

有時您可能希望將項目儲存在工作階段中以供下一個請求使用。您可以使用 `flash` 方法來執行此操作。使用此方法儲存到工作階段中的資料將會立即可用，並在後續的 HTTP 請求期間有效。在後續的 HTTP 請求之後，快閃資料將會被刪除。快閃資料主要用於短期的狀態訊息：

```php
$request->session()->flash('status', 'Task was successful!');
```

如果您需要將快閃資料持續儲存多個請求，您可以使用 `reflash` 方法，它會將所有快閃資料保留額外一個請求。如果您只需要保留特定的快閃資料，可以使用 `keep` 方法：

```php
$request->session()->reflash();

$request->session()->keep(['username', 'email']);
```

若要僅針對當前請求保留快閃資料，您可以使用 `now` 方法：

```php
$request->session()->now('status', 'Task was successful!');
```

<a name="deleting-data"></a>
### 刪除資料

`forget` 方法會從工作階段中移除一筆資料。如果您想從工作階段中移除所有資料，可以使用 `flush` 方法：

```php
// Forget a single key...
$request->session()->forget('name');

// Forget multiple keys...
$request->session()->forget(['name', 'status']);

$request->session()->flush();
```

<a name="regenerating-the-session-id"></a>
### 重新生成工作階段 ID

重新生成工作階段 ID 通常是為了防止惡意使用者對您的應用程式發動 [session fixation](https://owasp.org/www-community/attacks/Session_fixation) 攻擊。

如果您使用 Laravel [application starter kits](/docs/{{version}}/starter-kits) 或 [Laravel Fortify](/docs/{{version}}/fortify) 中的任何一個，Laravel 會在認證期間自動重新生成工作階段 ID；但是，如果您需要手動重新生成工作階段 ID，可以使用 `regenerate` 方法：

```php
$request->session()->regenerate();
```

如果您需要以單一陳述式重新生成工作階段 ID 並從工作階段中移除所有資料，您可以使用 `invalidate` 方法：

```php
$request->session()->invalidate();
```

<a name="session-cache"></a>
## 工作階段快取

Laravel 的工作階段快取提供了一種便捷的方式，可以快取針對個別使用者工作階段範圍的資料。與全域應用程式快取不同，工作階段快取資料會依據每個工作階段自動隔離，並在工作階段過期或銷毀時被清除。工作階段快取支援所有常見的 [Laravel 快取方法](/docs/{{version}}/cache)，例如 `get`、`put`、`remember`、`forget` 等，但範圍限定於當前的工作階段。

工作階段快取非常適合儲存您希望在同一個工作階段內跨多個請求持久保存，但又不需要永久儲存的暫時性、使用者專屬的資料。這包括了諸如表單資料、暫時計算結果、API 回應，或任何其他應綁定到特定使用者工作階段的短暫資料。

您可以透過工作階段上的 `cache` 方法來存取工作階段快取：

```php
$discount = $request->session()->cache()->get('discount');

$request->session()->cache()->put(
    'discount', 10, now()->addMinutes(5)
);
```

有關 Laravel 快取方法的更多資訊，請參閱[快取文件](/docs/{{version}}/cache)。

<a name="session-blocking"></a>
## 工作階段阻擋

> [!WARNING]
> 若要利用工作階段阻擋功能，您的應用程式必須使用一個支援[原子鎖定](/docs/{{version}}/cache#atomic-locks)的快取驅動器。目前，這些快取驅動器包括了 `memcached`、`dynamodb`、`redis`、`mongodb` (包含在官方 `mongodb/laravel-mongodb` 套件中)、`database`、`file` 和 `array` 驅動器。此外，您不能使用 `cookie` 工作階段驅動器。

預設情況下，Laravel 允許使用相同工作階段的請求同時執行。因此，例如，如果您使用 JavaScript HTTP 函式庫向您的應用程式發出兩個 HTTP 請求，它們將會同時執行。對於許多應用程式來說，這不是問題；然而，在對兩個不同應用程式端點發出並行請求且這兩個端點都寫入資料到工作階段的一小部分應用程式中，可能會發生工作階段資料遺失。

為了緩解這個問題，Laravel 提供了功能，讓您能夠限制給定工作階段的並行請求。要開始使用，您可以簡單地將 `block` 方法串接到您的路由定義上。在此範例中，傳入 `/profile` 端點的請求將會取得一個工作階段鎖定。在此鎖定被持有的期間，任何傳入 `/profile` 或 `/order` 端點且共享相同工作階段 ID 的請求，將會等待第一個請求執行完成後才繼續執行：

```php
Route::post('/profile', function () {
    // ...
})->block($lockSeconds = 10, $waitSeconds = 10);

Route::post('/order', function () {
    // ...
})->block($lockSeconds = 10, $waitSeconds = 10);
```

`block` 方法接受兩個選用引數。`block` 方法接受的第一個引數是工作階段鎖定被釋放前應持有的最長秒數。當然，如果請求在此時間之前執行完成，鎖定將會更早被釋放。

`block` 方法接受的第二個引數是請求在嘗試取得工作階段鎖定時應等待的秒數。如果請求在給定秒數內無法取得工作階段鎖定，將會拋出 `Illuminate\Contracts\Cache\LockTimeoutException`。

如果這兩個引數都沒有傳遞，鎖定將會被取得最長 10 秒，且請求在嘗試取得鎖定時將會等待最長 10 秒：

```php
Route::post('/profile', function () {
    // ...
})->block();
```

<a name="adding-custom-session-drivers"></a>
## 新增客製化工作階段驅動器

<a name="implementing-the-driver"></a>
### 實作驅動器

如果現有的工作階段驅動器都不符合您的應用程式需求，Laravel 讓您可以編寫自己的工作階段處理器。您的客製化工作階段驅動器應實作 PHP 內建的 `SessionHandlerInterface`。這個介面包含了幾個簡單的方法。一個骨架形式的 MongoDB 實作如下所示：

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

由於 Laravel 不包含預設目錄來存放您的擴充功能。您可以將它們放置在任何您喜歡的地方。在此範例中，我們建立了一個 `Extensions` 目錄來存放 `MongoSessionHandler`。

由於這些方法的目的不易理解，以下是每個方法的目的概述：

<div class="content-list" markdown="1">

- `open` 方法通常用於基於檔案的工作階段儲存系統。由於 Laravel 提供了 `file` 工作階段驅動器，您很少需要在此方法中放入任何內容。您可以簡單地將此方法留空。
- `close` 方法，與 `open` 方法類似，通常也可以忽略。對於大多數驅動器來說，它不是必需的。
- `read` 方法應傳回與給定 `$sessionId` 相關聯的工作階段資料的字串版本。在您的驅動器中擷取或儲存工作階段資料時，不需要進行任何序列化或其他編碼，因為 Laravel 會為您執行序列化。
- `write` 方法應將與 `$sessionId` 相關聯的給定 `$data` 字串寫入某個持久儲存系統，例如 MongoDB 或您選擇的其他儲存系統。同樣，您不應執行任何序列化——Laravel 將會為您處理。
- `destroy` 方法應從持久儲存中移除與 `$sessionId` 相關聯的資料。
- `gc` 方法應銷毀所有比給定 `$lifetime` (一個 UNIX 時間戳記) 更舊的工作階段資料。對於像 Memcached 和 Redis 這樣會自動過期的系統，此方法可以留空。

</div>

<a name="registering-the-driver"></a>
### 註冊驅動器

一旦您的驅動器已實作完成，您就可以將其註冊到 Laravel。若要將額外的驅動器新增到 Laravel 的工作階段後端，您可以使用 `Session` [Facade](/docs/{{version}}/facades) 提供的 `extend` 方法。您應該從[服務提供者](/docs/{{version}}/providers)的 `boot` 方法呼叫 `extend` 方法。您可以從現有的 `App\Providers\AppServiceProvider` 進行此操作，或者建立一個全新的提供者：

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

一旦工作階段驅動器已註冊，您可以使用 `SESSION_DRIVER` 環境變數或在應用程式的 `config/session.php` 設定檔中，將 `mongo` 驅動器指定為您應用程式的工作階段驅動器。