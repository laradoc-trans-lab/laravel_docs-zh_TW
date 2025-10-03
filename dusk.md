# Laravel Dusk

- [簡介](#introduction)
- [安裝](#installation)
    - [管理 ChromeDriver 安裝](#managing-chromedriver-installations)
    - [使用其他瀏覽器](#using-other-browsers)
- [開始使用](#getting-started)
    - [產生測試](#generating-tests)
    - [在每次測試後重設資料庫](#resetting-the-database-after-each-test)
    - [執行測試](#running-tests)
    - [處理環境](#environment-handling)
- [瀏覽器基礎](#browser-basics)
    - [建立瀏覽器](#creating-browsers)
    - [導覽](#navigation)
    - [調整瀏覽器視窗大小](#resizing-browser-windows)
    - [瀏覽器巨集](#browser-macros)
    - [認證](#authentication)
    - [Cookie](#cookies)
    - [執行 JavaScript](#executing-javascript)
    - [擷取螢幕截圖](#taking-a-screenshot)
    - [將主控台輸出儲存至磁碟](#storing-console-output-to-disk)
    - [將頁面原始碼儲存至磁碟](#storing-page-source-to-disk)
- [與元素互動](#interacting-with-elements)
    - [Dusk 選擇器](#dusk-selectors)
    - [文字、值與屬性](#text-values-and-attributes)
    - [與表單互動](#interacting-with-forms)
    - [附加檔案](#attaching-files)
    - [按下按鈕](#pressing-buttons)
    - [點擊連結](#clicking-links)
    - [使用鍵盤](#using-the-keyboard)
    - [使用滑鼠](#using-the-mouse)
    - [JavaScript 對話方塊](#javascript-dialogs)
    - [與 Inline Frame 互動](#interacting-with-iframes)
    - [限定選擇器範圍](#scoping-selectors)
    - [等待元素](#waiting-for-elements)
    - [將元素捲動至可視範圍](#scrolling-an-element-into-view)
- [可用的斷言](#available-assertions)
- [頁面](#pages)
    - [產生頁面](#generating-pages)
    - [設定頁面](#configuring-pages)
    - [導覽至頁面](#navigating-to-pages)
    - [選擇器簡寫](#shorthand-selectors)
    - [頁面方法](#page-methods)
- [元件](#components)
    - [產生元件](#generating-components)
    - [使用元件](#using-components)
- [持續整合](#continuous-integration)
    - [Heroku CI](#running-tests-on-heroku-ci)
    - [Travis CI](#running-tests-on-travis-ci)
    - [GitHub Actions](#running-tests-on-github-actions)
    - [Chipper CI](#running-tests-on-chipper-ci)

<a name="introduction"></a>
## 簡介

> [!WARNING]
> [Pest 4](https://pestphp.com/) 現在包含了自動化瀏覽器測試，與 Laravel Dusk 相比，它提供了顯著的效能與可用性改進。對於新專案，我們建議使用 Pest 進行瀏覽器測試。

[Laravel Dusk](https://github.com/laravel/dusk) 提供了一個富有表達性、易於使用的瀏覽器自動化與測試 API。預設情況下，Dusk 不需要你在本機電腦上安裝 JDK 或 Selenium。取而代之的是，Dusk 使用獨立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安裝。不過，你也可以自由使用任何其他與 Selenium 相容的驅動程式。


<a name="installation"></a>
## 安裝

開始前，應先安裝 [Google Chrome](https://www.google.com/chrome) 並將 `laravel/dusk` 這個 Composer 依賴項新增至專案中：

```shell
composer require laravel/dusk --dev
```

> [!WARNING]
> 若要手動註冊 Dusk 的 Service Provider，則**絕對不應**在生產環境中註冊它，因為這麼做可能會導致任意使用者能通過你的應用程式的認證。

安裝完 Dusk 套件後，請執行 `dusk:install` 這個 Artisan 指令。`dusk:install` 指令會建立一個 `tests/Browser` 目錄、一個 Dusk 測試範例，並為你的作業系統安裝 Chrome Driver 執行檔：

```shell
php artisan dusk:install
```

接著，在應用程式的 `.env` 檔中設定 `APP_URL` 環境變數。這個值應符合你在瀏覽器中用來存取應用程式的 URL。

> [!NOTE]
> 若正在使用 [Laravel Sail](/docs/{{version}}/sail) 來管理本機開發環境，也請參考 Sail 文件中有關[設定並執行 Dusk 測試](/docs/{{version}}/sail#laravel-dusk)的部分。


<a name="managing-chromedriver-installations"></a>
### 管理 ChromeDriver 安裝

若想安裝與 Laravel Dusk 透過 `dusk:install` 指令所安裝的版本不同的 ChromeDriver，可以使用 `dusk:chrome-driver` 指令：

```shell

# 為你的作業系統安裝最新版的 ChromeDriver...
php artisan dusk:chrome-driver


# 為你的作業系統安裝指定版本的 ChromeDriver...
php artisan dusk:chrome-driver 86


# 為所有支援的作業系統安裝指定版本的 ChromeDriver...
php artisan dusk:chrome-driver --all

# 安裝符合你作業系統上偵測到的 Chrome / Chromium 版本的 ChromeDriver……
php artisan dusk:chrome-driver --detect
```

> [!WARNING]
> Dusk 需要 `chromedriver` 二進位檔是可執行的。若在執行 Dusk 時遇到問題，應使用下列指令來確保這些二進位檔是可執行的： `chmod -R 0755 vendor/laravel/dusk/bin/`。

<a name="using-other-browsers"></a>
### 使用其他瀏覽器

預設情況下，Dusk 會使用 Google Chrome 與獨立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安裝來執行瀏覽器測試。不過，你也可以啟動自己的 Selenium 伺服器，並針對任何你想要的瀏覽器執行測試。

首先，請開啟應用程式中 Dusk 測試的基礎測試案例檔 `tests/DuskTestCase.php`。在此檔案中，你可以移除對 `startChromeDriver` 方法的呼叫。這樣就會讓 Dusk 不再自動啟動 ChromeDriver：

```php
/**
 * Prepare for Dusk test execution.
 *
 * @beforeClass
 */
public static function prepare(): void
{
    // static::startChromeDriver();
}
```

接著，你可以修改 `driver` 方法來連線到你選擇的 URL 與 Port。此外，你也可以修改要傳遞給 WebDriver 的「所需功能 (desired capabilities)」：

```php
use Facebook\WebDriver\Remote\RemoteWebDriver;

/**
 * Create the RemoteWebDriver instance.
 */
protected function driver(): RemoteWebDriver
{
    return RemoteWebDriver::create(
        'http://localhost:4444/wd/hub', DesiredCapabilities::phantomjs()
    );
}
```

<a name="getting-started"></a>
## 開始使用

<a name="generating-tests"></a>
### 產生測試

若要產生 Dusk 測試，可使用 `dusk:make` 這個 Artisan 指令。產生的測試會被放在 `tests/Browser` 目錄內：

```shell
php artisan dusk:make LoginTest
```

<a name="resetting-the-database-after-each-test"></a>
### 在每次測試後重設資料庫

你所寫的大部分測試都會與從應用程式資料庫中擷取資料的頁面互動；不過，Dusk 測試絕對不應使用 `RefreshDatabase` Trait。`RefreshDatabase` Trait 會利用資料庫 Transaction，而 Transaction 在跨 HTTP 請求時並不適用或無法使用。相反地，你有兩個選擇：`DatabaseMigrations` Trait 與 `DatabaseTruncation` Trait。

<a name="reset-migrations"></a>
#### 使用資料庫遷移

`DatabaseMigrations` Trait 會在每次測試前執行資料庫遷移。不過，在每次測試時都卸除並重新建立資料庫資料表通常會比截斷 (Truncating) 資料表還慢：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;

pest()->use(DatabaseMigrations::class);

//
```

```php tab=PHPUnit
<?php

namespace Tests\Browser;

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    use DatabaseMigrations;

    //
}
```

> [!WARNING]
> 執行 Dusk 測試時，不能使用 SQLite 記憶體內 (in-memory) 資料庫。因為瀏覽器是在自己的處理序中執行，所以無法存取其他處理序的記憶體內資料庫。

<a name="reset-truncation"></a>
#### 使用資料庫截斷

`DatabaseTruncation` Trait 會在第一個測試時遷移資料庫，以確保資料庫資料表已正確建立。不過，在後續的測試中，資料庫的資料表只會被截斷 (Truncated) —— 這比重新執行所有資料庫遷移還要快：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\DatabaseTruncation;
use Laravel\Dusk\Browser;

pest()->use(DatabaseTruncation::class);

//
```

```php tab=PHPUnit
<?php

namespace Tests\Browser;

use App\Models\User;
use Illuminate\Foundation\Testing\DatabaseTruncation;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    use DatabaseTruncation;

    //
}
```

預設情況下，此 Trait 會截斷除了 `migrations` 資料表以外的所有資料表。若想自訂要截斷的資料表，可以在測試類別上定義一個 `$tablesToTruncate` 屬性：

> [!NOTE]
> 若正在使用 Pest，應在基礎的 `DuskTestCase` 類別或在測試檔案所繼承的任何類別上定義屬性或方法。

```php
/**
 * Indicates which tables should be truncated.
 *
 * @var array
 */
protected $tablesToTruncate = ['users'];
```

或者，你也可以在測試類別上定義一個 `$exceptTables` 屬性，以指定哪些資料表應從截斷中排除：

```php
/**
 * Indicates which tables should be excluded from truncation.
 *
 * @var array
 */
protected $exceptTables = ['users'];
```

若要指定其資料表應被截斷的資料庫連線，可在測試類別上定義一個 `$connectionsToTruncate` 屬性：

```php
/**
 * Indicates which connections should have their tables truncated.
 *
 * @var array
 */
protected $connectionsToTruncate = ['mysql'];
```

若想在執行資料庫截斷之前或之後執行程式碼，可在測試類別上定義 `beforeTruncatingDatabase` 或 `afterTruncatingDatabase` 方法：

```php
/**
 * Perform any work that should take place before the database has started truncating.
 */
protected function beforeTruncatingDatabase(): void
{
    //
}

/**
 * Perform any work that should take place after the database has finished truncating.
 */
protected function afterTruncatingDatabase(): void
{
    //
}
```

<a name="running-tests"></a>
### 執行測試

若要執行瀏覽器測試，請執行 `dusk` 這個 Artisan 指令：

```shell
php artisan dusk
```

若上次執行 `dusk` 指令時有測試失敗，可使用 `dusk:fails` 指令來優先重新執行失敗的測試，以節省時間：

```shell
php artisan dusk:fails
```

`dusk` 指令可接受 Pest / PHPUnit 測試執行器通常能接受的任何引數，例如，只執行給定 [群組 (group)](https://docs.phpunit.de/en/10.5/annotations.html#group) 的測試：

```shell
php artisan dusk --group=foo
```

> [!NOTE]
> 若你正使用 [Laravel Sail](/docs/{{version}}/sail) 來管理本機開發環境，請參閱 Sail 有關 [設定與執行 Dusk 測試](/docs/{{version}}/sail#laravel-dusk) 的說明文件。

<a name="manually-starting-chromedriver"></a>
#### 手動啟動 ChromeDriver

預設情況下，Dusk 會自動嘗試啟動 ChromeDriver。若在你的特定系統上無法正常運作，可以在執行 `dusk` 指令前手動啟動 ChromeDriver。若選擇手動啟動 ChromeDriver，則應將 `tests/DuskTestCase.php` 檔案中的下列這行註解掉：

```php
/**
 * Prepare for Dusk test execution.
 *
 * @beforeClass
 */
public static function prepare(): void
{
    // static::startChromeDriver();
}
```

此外，若在 9515 以外的 Port 上啟動 ChromeDriver，則應修改同一個類別中的 `driver` 方法，以反映正確的 Port：

```php
use Facebook\WebDriver\Remote\RemoteWebDriver;

/**
 * Create the RemoteWebDriver instance.
 */
protected function driver(): RemoteWebDriver
{
    return RemoteWebDriver::create(
        'http://localhost:9515', DesiredCapabilities::chrome()
    );
}
```

<a name="environment-handling"></a>
### 處理環境

若要強制 Dusk 在執行測試時使用自己的環境檔，請在專案根目錄下建立一個 `.env.dusk.{environment}` 檔案。舉例來說，若要在 `local` 環境下執行 `dusk` 指令，就應建立一個 `.env.dusk.local` 檔案。

執行測試時，Dusk 會備份你的 `.env` 檔，並將你的 Dusk 環境檔重新命名為 `.env`。測試完成後，原本的 `.env` 檔會被還原。

<a name="browser-basics"></a>
## 瀏覽器基礎


<a name="creating-browsers"></a>
### 建立瀏覽器

首先，讓我們來寫一個測試，用來驗證我們能登入應用程式。在產生測試後，我們可以修改該測試，讓它導覽至登入頁面、輸入一些憑證資料，並點擊「Login」按鈕。若要建立瀏覽器實體，可以在 Dusk 測試中呼叫 `browse` 方法：

```php tab=Pest
<?php

use App\Models\User;
use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;

pest()->use(DatabaseMigrations::class);

test('basic example', function () {
    $user = User::factory()->create([
        'email' => 'taylor@laravel.com',
    ]);

    $this->browse(function (Browser $browser) use ($user) {
        $browser->visit('/login')
            ->type('email', $user->email)
            ->type('password', 'password')
            ->press('Login')
            ->assertPathIs('/home');
    });
});
```

```php tab=PHPUnit
<?php

namespace Tests\Browser;

use App\Models\User;
use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    use DatabaseMigrations;

    /**
     * A basic browser test example.
     */
    public function test_basic_example(): void
    {
        $user = User::factory()->create([
            'email' => 'taylor@laravel.com',
        ]);

        $this->browse(function (Browser $browser) use ($user) {
            $browser->visit('/login')
                ->type('email', $user->email)
                ->type('password', 'password')
                ->press('Login')
                ->assertPathIs('/home');
        });
    }
}
```

如上方的範例所示，`browse` 方法接受一個閉包。Dusk 會自動將一個瀏覽器實體傳遞給該閉包，而這個實體就是用來與應用程式互動並對其進行斷言的主要物件。


<a name="creating-multiple-browsers"></a>
#### 建立多個瀏覽器

有時候，為了能正確地執行測試，我們可能會需要多個瀏覽器。舉例來說，在測試與 WebSocket 互動的聊天室畫面時，可能就會需要多個瀏覽器。若要建立多個瀏覽器，只要在傳給 `browse` 方法的閉包的簽章 (Signature) 中增加更多瀏覽器引數即可：

```php
$this->browse(function (Browser $first, Browser $second) {
    $first->loginAs(User::find(1))
        ->visit('/home')
        ->waitForText('Message');

    $second->loginAs(User::find(2))
        ->visit('/home')
        ->waitForText('Message')
        ->type('message', 'Hey Taylor')
        ->press('Send');

    $first->waitForText('Hey Taylor')
        ->assertSee('Jeffrey Way');
});
```


<a name="navigation"></a>
### 導覽

`visit` 方法可用來導覽至應用程式內的給定 URI：

```php
$browser->visit('/login');
```

我們可以使用 `visitRoute` 方法來導覽至[命名路由](/docs/{{version}}/routing#named-routes)：

```php
$browser->visitRoute($routeName, $parameters);
```

我們可以使用 `back` 與 `forward` 方法來導覽「上一頁」與「下一頁」：

```php
$browser->back();

$browser->forward();
```

我們可以使用 `refresh` 方法來重新整理頁面：

```php
$browser->refresh();
```


<a name="resizing-browser-windows"></a>
### 調整瀏覽器視窗大小

我們可以使用 `resize` 方法來調整瀏覽器視窗的大小：

```php
$browser->resize(1920, 1080);
```

`maximize` 方法可用來將瀏覽器視窗最大化：

```php
$browser->maximize();
```

`fitContent` 方法會調整瀏覽器視窗大小，以符合其內容的大小：

```php
$browser->fitContent();
```

當測試失敗時，Dusk 會在擷取螢幕截圖前自動調整瀏覽器大小以符合內容。我們可以在測試中呼叫 `disableFitOnFailure` 方法來停用此功能：

```php
$browser->disableFitOnFailure();
```

我們可以使用 `move` 方法來將瀏覽器視窗移動到螢幕上的不同位置：

```php
$browser->move($x = 100, $y = 100);
```


<a name="browser-macros"></a>
### 瀏覽器巨集

若想定義一個可重複用於多個測試中的自訂瀏覽器方法，我們可以使用 `Browser` 類別上的 `macro` 方法。一般來說，我們應在[服務提供者](/docs/{{version}}/providers)的 `boot` 方法中呼叫此方法：

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Laravel\Dusk\Browser;

class DuskServiceProvider extends ServiceProvider
{
    /**
     * Register Dusk's browser macros.
     */
    public function boot(): void
    {
        Browser::macro('scrollToElement', function (string $element = null) {
            $this->script("$('html, body').animate({ scrollTop: $('$element').offset().top }, 0);");

            return $this;
        });
    }
}
```

`macro` 函式接受一個名稱作為其第一個引數，並接受一個閉包作為其第二個引數。當在 `Browser` 實體上將巨集作為方法呼叫時，就會執行該巨集的閉包：

```php
$this->browse(function (Browser $browser) use ($user) {
    $browser->visit('/pay')
        ->scrollToElement('#credit-card-details')
        ->assertSee('Enter Credit Card Details');
});
```


<a name="authentication"></a>
### 認證

通常，我們會測試需要認證的頁面。我們可以使用 Dusk 的 `loginAs` 方法來避免在每次測試中都與應用程式的登入畫面互動。`loginAs` 方法接受一個與可認證 Model 相關的主鍵，或是一個可認證 Model 實體：

```php
use App\Models\User;
use Laravel\Dusk\Browser;

$this->browse(function (Browser $browser) {
    $browser->loginAs(User::find(1))
        ->visit('/home');
});
```

> [!WARNING]
> 在使用 `loginAs` 方法後，使用者的 Session 會在該檔案內的所有測試中被保留。


<a name="cookies"></a>
### Cookie

我們可以使用 `cookie` 方法來取得或設定一個已加密 Cookie 的值。預設情況下，所有由 Laravel 建立的 Cookie 都會被加密：

```php
$browser->cookie('name');

$browser->cookie('name', 'Taylor');
```

我們可以使用 `plainCookie` 方法來取得或設定一個未加密 Cookie 的值：

```php
$browser->plainCookie('name');

$browser->plainCookie('name', 'Taylor');
```

我們可以使用 `deleteCookie` 方法來刪除給定的 Cookie：

```php
$browser->deleteCookie('name');
```


<a name="executing-javascript"></a>
### 執行 JavaScript

我們可以使用 `script` 方法來在瀏覽器中執行任意的 JavaScript 陳述式：

```php
$browser->script('document.documentElement.scrollTop = 0');

$browser->script([
    'document.body.scrollTop = 0',
    'document.documentElement.scrollTop = 0',
]);

$output = $browser->script('return window.location.pathname');
```


<a name="taking-a-screenshot"></a>
### 擷取螢幕截圖

我們可以使用 `screenshot` 方法來擷取螢幕截圖，並以給定的檔名儲存。所有的螢幕截圖都會被存放在 `tests/Browser/screenshots` 目錄內：

```php
$browser->screenshot('filename');
```

`responsiveScreenshots` 方法可用來在一系列的斷點 (Breakpoint) 上擷取一系列的螢幕截圖：

```php
$browser->responsiveScreenshots('filename');
```

`screenshotElement` 方法可用來擷取頁面上某個特定元素的螢幕截圖：

```php
$browser->screenshotElement('#selector', 'filename');
```


<a name="storing-console-output-to-disk"></a>
### 將主控台輸出儲存至磁碟

我們可以使用 `storeConsoleLog` 方法來將目前瀏覽器的主控台輸出以給定的檔名寫入磁碟。主控台輸出會被儲存在 `tests/Browser/console` 目錄內：

```php
$browser->storeConsoleLog('filename');
```


<a name="storing-page-source-to-disk"></a>
### 將頁面原始碼儲存至磁碟

我們可以使用 `storeSource` 方法來將目前頁面的原始碼以給定的檔名寫入磁碟。頁面原始碼會被儲存在 `tests/Browser/source` 目錄內：

```php
$browser->storeSource('filename');
```

<a name="interacting-with-elements"></a>
## 與元素互動


<a name="dusk-selectors"></a>
### Dusk 選擇器

為要互動的元素選擇好的 CSS 選擇器是撰寫 Dusk 測試中最難的部分之一。隨著時間推移，前端的變更可能會導致像下面這樣的 CSS 選擇器讓你的測試失效：

```html
// HTML...

<button>Login</button>
```

```php
// Test...

$browser->click('.login-page .container div > button');
```

Dusk 選擇器讓你能專注於撰寫有效的測試，而不是去記 CSS 選擇器。若要定義一個選擇器，只要在 HTML 元素上加上一個 `dusk` 屬性即可。然後，在與 Dusk 瀏覽器互動時，只要在選擇器前加上 `@` 前置詞，就可以在測試中操作附加的元素：

```html
// HTML...

<button dusk="login-button">Login</button>
```

```php
// Test...

$browser->click('@login-button');
```

若有需要，也可以通過 `selectorHtmlAttribute` 方法來自訂 Dusk 選擇器所使用的 HTML 屬性。一般來說，應在應用程式 `AppServiceProvider` 的 `boot` 方法中呼叫此方法：

```php
use Laravel\Dusk\Dusk;

Dusk::selectorHtmlAttribute('data-dusk');
```


<a name="text-values-and-attributes"></a>
### 文字、值與屬性


<a name="retrieving-setting-values"></a>
#### 擷取與設定值

Dusk 提供了數個方法來與頁面上元素的目前值、顯示文字、以及屬性進行互動。舉例來說，若要取得符合特定 CSS 或 Dusk 選擇器的元素的「value」，請使用 `value` 方法：

```php
// Retrieve the value...
$value = $browser->value('selector');

// Set the value...
$browser->value('selector', 'value');
```

可以使用 `inputValue` 方法來取得具有特定欄位名稱的 Input 元素的「value」：

```php
$value = $browser->inputValue('field');
```


<a name="retrieving-text"></a>
#### 擷取文字

`text` 方法可用於擷取符合特定選擇器的元素的顯示文字：

```php
$text = $browser->text('selector');
```


<a name="retrieving-attributes"></a>
#### 擷取屬性

最後，`attribute` 方法可用於擷取符合特定選擇器的元素的屬性值：

```php
$attribute = $browser->attribute('selector', 'value');
```


<a name="interacting-with-forms"></a>
### 與表單互動


<a name="typing-values"></a>
#### 輸入值

Dusk 提供了多種方法來與表單和輸入元素互動。首先，讓我們先看一個在輸入欄位中輸入文字的範例：

```php
$browser->type('email', 'taylor@laravel.com');
```

請注意，雖然 `type` 方法在需要時可以接受 CSS 選擇器，但我們不一定要傳入它。如果沒有提供 CSS 選擇器，Dusk 將會搜尋具有給定 `name` 屬性的 `input` 或 `textarea` 欄位。

若要在不清除欄位內容的情況下附加文字，可以使用 `append` 方法：

```php
$browser->type('tags', 'foo')
    ->append('tags', ', bar, baz');
```

可以使用 `clear` 方法來清除輸入欄位的值：

```php
$browser->clear('email');
```

你可以使用 `typeSlowly` 方法來指示 Dusk 緩慢地輸入。預設情況下，Dusk 在每次按鍵之間會暫停 100 毫秒。若要自訂按鍵之間的暫停時間，可以將適當的毫秒數作為第三個參數傳給該方法：

```php
$browser->typeSlowly('mobile', '+1 (202) 555-5555');

$browser->typeSlowly('mobile', '+1 (202) 555-5555', 300);
```

可以使用 `appendSlowly` 方法來緩慢地附加文字：

```php
$browser->type('tags', 'foo')
    ->appendSlowly('tags', ', bar, baz');
```


<a name="dropdowns"></a>
#### 下拉式選單

若要選取 `select` 元素上的可用值，可以使用 `select` 方法。與 `type` 方法類似，`select` 方法不需要完整的 CSS 選擇器。當傳遞值給 `select` 方法時，你應該傳遞底層的選項值，而不是顯示文字：

```php
$browser->select('size', 'Large');
```

你可以省略第二個參數來選取一個隨機的選項：

```php
$browser->select('size');
```

透過提供一個陣列作為 `select` 方法的第二個參數，你可以指示該方法選取多個選項：

```php
$browser->select('categories', ['Art', 'Music']);
```


<a name="checkboxes"></a>
#### 核取方塊

若要「勾選」一個核取方塊輸入欄位，可以使用 `check` 方法。與許多其他與輸入相關的方法一樣，不需要完整的 CSS 選擇器。如果找不到匹配的 CSS 選擇器，Dusk 將會搜尋具有匹配 `name` 屬性的核取方塊：

```php
$browser->check('terms');
```

`uncheck` 方法可用於「取消勾選」一個核取方塊輸入欄位：

```php
$browser->uncheck('terms');
```


<a name="radio-buttons"></a>
#### 選項按鈕

若要「選取」一個 `radio` 輸入選項，可以使用 `radio` 方法。與許多其他與輸入相關的方法一樣，不需要完整的 CSS 選擇器。如果找不到匹配的 CSS 選擇器，Dusk 將會搜尋具有匹配 `name` 和 `value` 屬性的 `radio` 輸入：

```php
$browser->radio('size', 'large');
```


<a name="attaching-files"></a>
### 附加檔案

`attach` 方法可用於將檔案附加到 `file` 輸入元素。與許多其他與輸入相關的方法一樣，不需要完整的 CSS 選擇器。如果找不到匹配的 CSS 選擇器，Dusk 將會搜尋具有匹配 `name` 屬性的 `file` 輸入：

```php
$browser->attach('photo', __DIR__.'/photos/mountains.png');
```

> [!WARNING]
> attach 函式需要在你的伺服器上安裝並啟用 `Zip` PHP 擴充套件。


<a name="pressing-buttons"></a>
### 按下按鈕

`press` 方法可用於點擊頁面上的按鈕元素。傳給 `press` 方法的參數可以是按鈕的顯示文字或 CSS / Dusk 選擇器：

```php
$browser->press('Login');
```

在提交表單時，許多應用程式會在按下提交按鈕後禁用該按鈕，然後在表單提交的 HTTP 請求完成後重新啟用該按鈕。若要按下一個按鈕並等待該按鈕被重新啟用，可以使用 `pressAndWaitFor` 方法：

```php
// Press the button and wait a maximum of 5 seconds for it to be enabled...
$browser->pressAndWaitFor('Save');

// Press the button and wait a maximum of 1 second for it to be enabled...
$browser->pressAndWaitFor('Save', 1);
```


<a name="clicking-links"></a>
### 點擊連結

若要點擊一個連結，可以在瀏覽器實例上使用 `clickLink` 方法。`clickLink` 方法將點擊具有給定顯示文字的連結：

```php
$browser->clickLink($linkText);
```

可以使用 `seeLink` 方法來判斷具有給定顯示文字的連結是否在頁面上可見：

```php
if ($browser->seeLink($linkText)) {
    // ...
}
```

> [!WARNING]
> 這些方法會與 jQuery 互動。如果頁面上沒有 jQuery，Dusk 會自動將其注入頁面，以便在測試期間使用。


<a name="using-the-keyboard"></a>
### 使用鍵盤

`keys` 方法允許你向給定元素提供比 `type` 方法通常允許的更複雜的輸入序列。例如，你可以指示 Dusk 在輸入值時按住修飾鍵。在這個例子中，當在匹配給定選擇器的元素中輸入 `taylor` 時，`shift` 鍵將被按住。在輸入 `taylor` 之後，將不帶任何修飾鍵地輸入 `swift`：

```php
$browser->keys('selector', ['{shift}', 'taylor'], 'swift');
```

`keys` 方法的另一個有價值的用例是向應用程式的主要 CSS 選擇器發送「鍵盤快捷鍵」組合：

```php
$browser->keys('.app', ['{command}', 'j']);
```

> [!NOTE]
> 所有修飾鍵如 `{command}` 都被包裹在 `{}` 字元中，並且與 `Facebook\WebDriver\WebDriverKeys` 類別中定義的常數相匹配，你可以在 [GitHub 上找到這些常數](https://github.com/php-webdriver/php-webdriver/blob/master/lib/WebDriverKeys.php)。


<a name="fluent-keyboard-interactions"></a>
#### 流暢的鍵盤互動

Dusk 還提供了一個 `withKeyboard` 方法，讓你可以通過 `Laravel\Dusk\Keyboard` 類別流暢地執行複雜的鍵盤互動。`Keyboard` 類別提供了 `press`、`release`、`type` 和 `pause` 方法：

```php
use Laravel\Dusk\Keyboard;

$browser->withKeyboard(function (Keyboard $keyboard) {
    $keyboard->press('c')
        ->pause(1000)
        ->release('c')
        ->type(['c', 'e', 'o']);
});
```


<a name="keyboard-macros"></a>
#### 鍵盤巨集

如果你想定義可以在整個測試套件中輕鬆重複使用的自訂鍵盤互動，可以使用 `Keyboard` 類別提供的 `macro` 方法。通常，你應該從[服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

```php
<?php

namespace App\Providers;

use Facebook\WebDriver\WebDriverKeys;
use Illuminate\Support\ServiceProvider;
use Laravel\Dusk\Keyboard;
use Laravel\Dusk\OperatingSystem;

class DuskServiceProvider extends ServiceProvider
{
    /**
     * Register Dusk's browser macros.
     */
    public function boot(): void
    {
        Keyboard::macro('copy', function (string $element = null) {
            $this->type([
                OperatingSystem::onMac() ? WebDriverKeys::META : WebDriverKeys::CONTROL, 'c',
            ]);

            return $this;
        });

        Keyboard::macro('paste', function (string $element = null) {
            $this->type([
                OperatingSystem::onMac() ? WebDriverKeys::META : WebDriverKeys::CONTROL, 'v',
            ]);

            return $this;
        });
    }
}
```

`macro` 函式接受一個名稱作為其第一個參數，以及一個閉包作為其第二個參數。當在 `Keyboard` 實例上呼叫巨集作為方法時，將會執行該巨集的閉包：

```php
$browser->click('@textarea')
    ->withKeyboard(fn (Keyboard $keyboard) => $keyboard->copy())
    ->click('@another-textarea')
    ->withKeyboard(fn (Keyboard $keyboard) => $keyboard->paste());
```


<a name="using-the-mouse"></a>
### 使用滑鼠


<a name="clicking-on-elements"></a>
#### 點擊元素

`click` 方法可用於點擊匹配給定 CSS 或 Dusk 選擇器的元素：

```php
$browser->click('.selector');
```

`clickAtXPath` 方法可用於點擊匹配給定 XPath 表達式的元素：

```php
$browser->clickAtXPath('//div[@class = "selector"]');
```

`clickAtPoint` 方法可用於點擊瀏覽器可視區域內給定座標對的最上層元素：

```php
$browser->clickAtPoint($x = 0, $y = 0);
```

`doubleClick` 方法可用於模擬滑鼠的雙擊：

```php
$browser->doubleClick();

$browser->doubleClick('.selector');
```

`rightClick` 方法可用於模擬滑鼠的右擊：

```php
$browser->rightClick();

$browser->rightClick('.selector');
```

`clickAndHold` 方法可用於模擬滑鼠按鈕被點擊並按住。後續呼叫 `releaseMouse` 方法將撤銷此行為並釋放滑鼠按鈕：

```php
$browser->clickAndHold('.selector');

$browser->clickAndHold()
    ->pause(1000)
    ->releaseMouse();
```

`controlClick` 方法可用於在瀏覽器中模擬 `ctrl+click` 事件：

```php
$browser->controlClick();

$browser->controlClick('.selector');
```


<a name="mouseover"></a>
#### 滑鼠懸停

當你需要將滑鼠移到匹配給定 CSS 或 Dusk 選擇器的元素上時，可以使用 `mouseover` 方法：

```php
$browser->mouseover('.selector');
```


<a name="drag-drop"></a>
#### 拖放

`drag` 方法可用於將匹配給定選擇器的元素拖到另一個元素上：

```php
$browser->drag('.from-selector', '.to-selector');
```

或者，你可以將元素朝單一方向拖動：

```php
$browser->dragLeft('.selector', $pixels = 10);
$browser->dragRight('.selector', $pixels = 10);
$browser->dragUp('.selector', $pixels = 10);
$browser->dragDown('.selector', $pixels = 10);
```

最後，你可以按給定的偏移量拖動元素：

```php
$browser->dragOffset('.selector', $x = 10, $y = 10);
```


<a name="javascript-dialogs"></a>
### JavaScript 對話方塊

Dusk 提供了各種方法來與 JavaScript 對話方塊互動。例如，你可以使用 `waitForDialog` 方法來等待 JavaScript 對話方塊出現。此方法接受一個可選參數，指示等待對話方塊出現的秒數：

```php
$browser->waitForDialog($seconds = null);
```

`assertDialogOpened` 方法可用於斷言對話方塊已顯示並包含給定的訊息：

```php
$browser->assertDialogOpened('Dialog message');
```

如果 JavaScript 對話方塊包含提示，你可以使用 `typeInDialog` 方法在提示中輸入值：

```php
$browser->typeInDialog('Hello World');
```

若要透過點擊「確定」按鈕來關閉一個開啟的 JavaScript 對話方塊，你可以調用 `acceptDialog` 方法：

```php
$browser->acceptDialog();
```

若要透過點擊「取消」按鈕來關閉一個開啟的 JavaScript 對話方塊，你可以調用 `dismissDialog` 方法：

```php
$browser->dismissDialog();
```


<a name="interacting-with-iframes"></a>
### 與 Inline Frame 互動

如果你需要與 iframe 內的元素互動，可以使用 `withinFrame` 方法。所有在提供給 `withinFrame` 方法的閉包內發生的元素互動，都將被限定在指定 iframe 的上下文中：

```php
$browser->withinFrame('#credit-card-details', function ($browser) {
    $browser->type('input[name="cardnumber"]', '4242424242424242')
        ->type('input[name="exp-date"]', '1224')
        ->type('input[name="cvc"]', '123')
        ->press('Pay');
});
```


<a name="scoping-selectors"></a>
### 限定選擇器範圍

有時候你可能希望在給定的選擇器範圍內執行多個操作。例如，你可能希望斷言某些文字僅存在於一個表格中，然後點擊該表格內的一個按鈕。你可以使用 `with` 方法來完成這個任務。所有在給定 `with` 方法的閉包內執行的操作都將被限定在原始選擇器的範圍內：

```php
$browser->with('.table', function (Browser $table) {
    $table->assertSee('Hello World')
        ->clickLink('Delete');
});
```

你可能偶爾需要在當前範圍之外執行斷言。你可以使用 `elsewhere` 和 `elsewhereWhenAvailable` 方法來完成這個任務：

```php
$browser->with('.table', function (Browser $table) {
    // Current scope is `body .table`...

    $browser->elsewhere('.page-title', function (Browser $title) {
        // Current scope is `body .page-title`...
        $title->assertSee('Hello World');
    });

    $browser->elsewhereWhenAvailable('.page-title', function (Browser $title) {
        // Current scope is `body .page-title`...
        $title->assertSee('Hello World');
    });
});
```


<a name="waiting-for-elements"></a>
### 等待元素

在測試大量使用 JavaScript 的應用程式時，通常需要在進行測試之前「等待」某些元素或資料可用。Dusk 使這變得輕而易舉。使用各種方法，你可以等待元素在頁面上變得可見，甚至等到給定的 JavaScript 表達式評估為 `true`。


<a name="waiting"></a>
#### 等待

如果你只需要將測試暫停給定的毫秒數，請使用 `pause` 方法：

```php
$browser->pause(1000);
```

如果你只需要在給定條件為 `true` 時才暫停測試，請使用 `pauseIf` 方法：

```php
$browser->pauseIf(App::environment('production'), 1000);
```

同樣地，如果你需要暫停測試除非給定條件為 `true`，可以使用 `pauseUnless` 方法：

```php
$browser->pauseUnless(App::environment('testing'), 1000);
```


<a name="waiting-for-selectors"></a>
#### 等待選擇器

`waitFor` 方法可用於暫停測試的執行，直到匹配給定 CSS 或 Dusk 選擇器的元素顯示在頁面上。預設情況下，這將暫停測試最多五秒鐘，然後拋出例外。如果需要，你可以將自訂的超時閾值作為第二個參數傳遞給該方法：

```php
// Wait a maximum of five seconds for the selector...
$browser->waitFor('.selector');

// Wait a maximum of one second for the selector...
$browser->waitFor('.selector', 1);
```

你也可以等到匹配給定選擇器的元素包含給定的文字：

```php
// Wait a maximum of five seconds for the selector to contain the given text...
$browser->waitForTextIn('.selector', 'Hello World');

// Wait a maximum of one second for the selector to contain the given text...
$browser->waitForTextIn('.selector', 'Hello World', 1);
```

你也可以等到匹配給定選擇器的元素從頁面上消失：

```php
// Wait a maximum of five seconds until the selector is missing...
$browser->waitUntilMissing('.selector');

// Wait a maximum of one second until the selector is missing...
$browser->waitUntilMissing('.selector', 1);
```

或者，你可以等到匹配給定選擇器的元素被啟用或禁用：

```php
// Wait a maximum of five seconds until the selector is enabled...
$browser->waitUntilEnabled('.selector');

// Wait a maximum of one second until the selector is enabled...
$browser->waitUntilEnabled('.selector', 1);

// Wait a maximum of five seconds until the selector is disabled...
$browser->waitUntilDisabled('.selector');

// Wait a maximum of one second until the selector is disabled...
$browser->waitUntilDisabled('.selector', 1);
```


<a name="scoping-selectors-when-available"></a>
#### 當選擇器可用時限定範圍

有時候，你可能希望等待一個匹配給定選擇器的元素出現，然後與該元素互動。例如，你可能希望等到一個模態視窗可用，然後在該模態視窗內按下「確定」按鈕。`whenAvailable` 方法可用於完成這個任務。在給定的閉包內執行的所有元素操作都將被限定在原始選擇器的範圍內：

```php
$browser->whenAvailable('.modal', function (Browser $modal) {
    $modal->assertSee('Hello World')
        ->press('OK');
});
```


<a name="waiting-for-text"></a>
#### 等待文字

`waitForText` 方法可用於等待直到給定的文字顯示在頁面上：

```php
// Wait a maximum of five seconds for the text...
$browser->waitForText('Hello World');

// Wait a maximum of one second for the text...
$browser->waitForText('Hello World', 1);
```

你可以使用 `waitUntilMissingText` 方法來等待直到顯示的文字從頁面上被移除：

```php
// Wait a maximum of five seconds for the text to be removed...
$browser->waitUntilMissingText('Hello World');

// Wait a maximum of one second for the text to be removed...
$browser->waitUntilMissingText('Hello World', 1);
```


<a name="waiting-for-links"></a>
#### 等待連結

`waitForLink` 方法可用於等待直到給定的連結文字顯示在頁面上：

```php
// Wait a maximum of five seconds for the link...
$browser->waitForLink('Create');

// Wait a maximum of one second for the link...
$browser->waitForLink('Create', 1);
```


<a name="waiting-for-inputs"></a>
#### 等待輸入

`waitForInput` 方法可用於等待直到給定的輸入欄位在頁面上可見：

```php
// Wait a maximum of five seconds for the input...
$browser->waitForInput($field);

// Wait a maximum of one second for the input...
$browser->waitForInput($field, 1);
```


<a name="waiting-on-the-page-location"></a>
#### 等待頁面位置

當進行路徑斷言，例如 `$browser->assertPathIs('/home')` 時，如果 `window.location.pathname` 是非同步更新的，斷言可能會失敗。你可以使用 `waitForLocation` 方法來等待位置成為給定的值：

```php
$browser->waitForLocation('/secret');
```

`waitForLocation` 方法也可以用來等待當前的視窗位置是一個完整的 URL：

```php
$browser->waitForLocation('https://example.com/path');
```

你也可以等待[命名路由](/docs/{{version}}/routing#named-routes)的位置：

```php
$browser->waitForRoute($routeName, $parameters);
```


<a name="waiting-for-page-reloads"></a>
#### 等待頁面重新載入

如果你需要在執行一個動作後等待頁面重新載入，請使用 `waitForReload` 方法：

```php
use Laravel\Dusk\Browser;

$browser->waitForReload(function (Browser $browser) {
    $browser->press('Submit');
})
->assertSee('Success!');
```

由於等待頁面重新載入的需求通常發生在點擊按鈕之後，為了方便，你可以使用 `clickAndWaitForReload` 方法：

```php
$browser->clickAndWaitForReload('.selector')
    ->assertSee('something');
```


<a name="waiting-on-javascript-expressions"></a>
#### 等待 JavaScript 表達式

有時候你可能希望暫停測試的執行，直到一個給定的 JavaScript 表達式評估為 `true`。你可以使用 `waitUntil` 方法輕鬆完成這個任務。當向此方法傳遞一個表達式時，你不需要包含 `return` 關鍵字或結尾的分號：

```php
// Wait a maximum of five seconds for the expression to be true...
$browser->waitUntil('App.data.servers.length > 0');

// Wait a maximum of one second for the expression to be true...
$browser->waitUntil('App.data.servers.length > 0', 1);
```


<a name="waiting-on-vue-expressions"></a>
#### 等待 Vue 表達式

`waitUntilVue` 和 `waitUntilVueIsNot` 方法可用於等待直到一個 [Vue component](https://vuejs.org) 屬性具有給定的值：

```php
// Wait until the component attribute contains the given value...
$browser->waitUntilVue('user.name', 'Taylor', '@user');

// Wait until the component attribute doesn't contain the given value...
$browser->waitUntilVueIsNot('user.name', null, '@user');
```


<a name="waiting-for-javascript-events"></a>
#### 等待 JavaScript 事件

`waitForEvent` 方法可以用於暫停測試的執行，直到一個 JavaScript 事件發生：

```php
$browser->waitForEvent('load');
```

事件監聽器被附加到當前的範圍，預設是 `body` 元素。當使用限定範圍的選擇器時，事件監聽器將被附加到匹配的元素上：

```php
$browser->with('iframe', function (Browser $iframe) {
    // Wait for the iframe's load event...
    $iframe->waitForEvent('load');
});
```

你也可以提供一個選擇器作為 `waitForEvent` 方法的第二個參數，將事件監聽器附加到特定的元素上：

```php
$browser->waitForEvent('load', '.selector');
```

你也可以等待 `document` 和 `window` 物件上的事件：

```php
// Wait until the document is scrolled...
$browser->waitForEvent('scroll', 'document');

// Wait a maximum of five seconds until the window is resized...
$browser->waitForEvent('resize', 'window', 5);
```


<a name="waiting-with-a-callback"></a>
#### 使用回呼等待

Dusk 中的許多「等待」方法都依賴於底層的 `waitUsing` 方法。你可以直接使用此方法來等待給定的閉包返回 `true`。`waitUsing` 方法接受最大等待秒數、閉包的評估間隔、閉包本身，以及一個可選的失敗訊息：

```php
$browser->waitUsing(10, 1, function () use ($something) {
    return $something->isReady();
}, "Something wasn't ready in time.");
```


<a name="scrolling-an-element-into-view"></a>
### 將元素捲動至可視範圍

有時你可能無法點擊一個元素，因為它在瀏覽器的可視區域之外。`scrollIntoView` 方法將滾動瀏覽器視窗，直到給定選擇器的元素進入可視範圍：

```php
$browser->scrollIntoView('.selector')
    ->click('.selector');
```

<a name="available-assertions"></a>
## 可用的斷言

Dusk 提供了多種可用來對應用程式進行的斷言。所有可用的斷言都記錄在下方的清單中：

<style>
    .collection-method-list > p {
        columns: 10.8em 3; -moz-columns: 10.8em 3; -webkit-columns: 10.8em 3;
    }

    .collection-method-list a {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
</style>

<div class="collection-method-list" markdown="1">

[assertTitle](#assert-title)
[assertTitleContains](#assert-title-contains)
[assertUrlIs](#assert-url-is)
[assertSchemeIs](#assert-scheme-is)
[assertSchemeIsNot](#assert-scheme-is-not)
[assertHostIs](#assert-host-is)
[assertHostIsNot](#assert-host-is-not)
[assertPortIs](#assert-port-is)
[assertPortIsNot](#assert-port-is-not)
[assertPathBeginsWith](#assert-path-begins-with)
[assertPathEndsWith](#assert-path-ends-with)
[assertPathContains](#assert-path-contains)
[assertPathIs](#assert-path-is)
[assertPathIsNot](#assert-path-is-not)
[assertRouteIs](#assert-route-is)
[assertQueryStringHas](#assert-query-string-has)
[assertQueryStringMissing](#assert-query-string-missing)
[assertFragmentIs](#assert-fragment-is)
[assertFragmentBeginsWith](#assert-fragment-begins-with)
[assertFragmentIsNot](#assert-fragment-is-not)
[assertHasCookie](#assert-has-cookie)
[assertHasPlainCookie](#assert-has-plain-cookie)
[assertCookieMissing](#assert-cookie-missing)
[assertPlainCookieMissing](#assert-plain-cookie-missing)
[assertCookieValue](#assert-cookie-value)
[assertPlainCookieValue](#assert-plain-cookie-value)
[assertSee](#assert-see)
[assertDontSee](#assert-dont-see)
[assertSeeIn](#assert-see-in)
[assertDontSeeIn](#assert-dont-see-in)
[assertSeeAnythingIn](#assert-see-anything-in)
[assertSeeNothingIn](#assert-see-nothing-in)
[assertCount](#assert-count)
[assertScript](#assert-script)
[assertSourceHas](#assert-source-has)
[assertSourceMissing](#assert-source-missing)
[assertSeeLink](#assert-see-link)
[assertDontSeeLink](#assert-dont-see-link)
[assertInputValue](#assert-input-value)
[assertInputValueIsNot](#assert-input-value-is-not)
[assertChecked](#assert-checked)
[assertNotChecked](#assert-not-checked)
[assertIndeterminate](#assert-indeterminate)
[assertRadioSelected](#assert-radio-selected)
[assertRadioNotSelected](#assert-radio-not-selected)
[assertSelected](#assert-selected)
[assertNotSelected](#assert-not-selected)
[assertSelectHasOptions](#assert-select-has-options)
[assertSelectMissingOptions](#assert-select-missing-options)
[assertSelectHasOption](#assert-select-has-option)
[assertSelectMissingOption](#assert-select-missing-option)
[assertValue](#assert-value)
[assertValueIsNot](#assert-value-is-not)
[assertAttribute](#assert-attribute)
[assertAttributeMissing](#assert-attribute-missing)
[assertAttributeContains](#assert-attribute-contains)
[assertAttributeDoesntContain](#assert-attribute-doesnt-contain)
[assertAriaAttribute](#assert-aria-attribute)
[assertDataAttribute](#assert-data-attribute)
[assertVisible](#assert-visible)
[assertPresent](#assert-present)
[assertNotPresent](#assert-not-present)
[assertMissing](#assert-missing)
[assertInputPresent](#assert-input-present)
[assertInputMissing](#assert-input-missing)
[assertDialogOpened](#assert-dialog-opened)
[assertEnabled](#assert-enabled)
[assertDisabled](#assert-disabled)
[assertButtonEnabled](#assert-button-enabled)
[assertButtonDisabled](#assert-button-disabled)
[assertFocused](#assert-focused)
[assertNotFocused](#assert-not-focused)
[assertAuthenticated](#assert-authenticated)
[assertGuest](#assert-guest)
[assertAuthenticatedAs](#assert-authenticated-as)
[assertVue](#assert-vue)
[assertVueIsNot](#assert-vue-is-not)
[assertVueContains](#assert-vue-contains)
[assertVueDoesntContain](#assert-vue-doesnt-contain)

</div>


<a name="assert-title"></a>
#### assertTitle

斷言頁面標題符合給定的文字：

```php
$browser->assertTitle($title);
```


<a name="assert-title-contains"></a>
#### assertTitleContains

斷言頁面標題包含給定的文字：

```php
$browser->assertTitleContains($title);
```


<a name="assert-url-is"></a>
#### assertUrlIs

斷言目前的 URL (不含查詢字串) 符合給定的字串：

```php
$browser->assertUrlIs($url);
```


<a name="assert-scheme-is"></a>
#### assertSchemeIs

斷言目前的 URL Scheme 符合給定的 Scheme：

```php
$browser->assertSchemeIs($scheme);
```


<a name="assert-scheme-is-not"></a>
#### assertSchemeIsNot

斷言目前的 URL Scheme 不符合給定的 Scheme：

```php
$browser->assertSchemeIsNot($scheme);
```


<a name="assert-host-is"></a>
#### assertHostIs

斷言目前的 URL Host 符合給定的 Host：

```php
$browser->assertHostIs($host);
```


<a name="assert-host-is-not"></a>
#### assertHostIsNot

斷言目前的 URL Host 不符合給定的 Host：

```php
$browser->assertHostIsNot($host);
```


<a name="assert-port-is"></a>
#### assertPortIs

斷言目前的 URL Port 符合給定的 Port：

```php
$browser->assertPortIs($port);
```


<a name="assert-port-is-not"></a>
<h4>assertPortIsNot</h4>

斷言目前的 URL Port 不符合給定的 Port：

```php
$browser->assertPortIsNot($port);
```


<a name="assert-path-begins-with"></a>
#### assertPathBeginsWith

斷言目前的 URL 路徑以給定的路徑開頭：

```php
$browser->assertPathBeginsWith('/home');
```


<a name="assert-path-ends-with"></a>
#### assertPathEndsWith

斷言目前的 URL 路徑以給定的路徑結尾：

```php
$browser->assertPathEndsWith('/home');
```


<a name="assert-path-contains"></a>
#### assertPathContains

斷言目前的 URL 路徑包含給定的路徑：

```php
$browser->assertPathContains('/home');
```


<a name="assert-path-is"></a>
#### assertPathIs

斷言目前的路徑符合給定的路徑：

```php
$browser->assertPathIs('/home');
```


<a name="assert-path-is-not"></a>
#### assertPathIsNot

斷言目前的路徑不符合給定的路徑：

```php
$browser->assertPathIsNot('/home');
```


<a name="assert-route-is"></a>
#### assertRouteIs

斷言目前的 URL 符合給定的[命名路由](/docs/{{version}}/routing#named-routes) URL：

```php
$browser->assertRouteIs($name, $parameters);
```


<a name="assert-query-string-has"></a>
#### assertQueryStringHas

斷言給定的查詢字串參數存在：

```php
$browser->assertQueryStringHas($name);
```

斷言給定的查詢字串參數存在且具有給定的值：

```php
$browser->assertQueryStringHas($name, $value);
```


<a name="assert-query-string-missing"></a>
#### assertQueryStringMissing

斷言給定的查詢字串參數不存在：

```php
$browser->assertQueryStringMissing($name);
```


<a name="assert-fragment-is"></a>
#### assertFragmentIs

斷言 URL 目前的雜湊片段 (hash fragment) 符合給定的片段：

```php
$browser->assertFragmentIs('anchor');
```


<a name="assert-fragment-begins-with"></a>
#### assertFragmentBeginsWith

斷言 URL 目前的雜湊片段以給定的片段開頭：

```php
$browser->assertFragmentBeginsWith('anchor');
```


<a name="assert-fragment-is-not"></a>
#### assertFragmentIsNot

斷言 URL 目前的雜湊片段不符合給定的片段：

```php
$browser->assertFragmentIsNot('anchor');
```


<a name="assert-has-cookie"></a>
#### assertHasCookie

斷言給定的已加密 Cookie 存在：

```php
$browser->assertHasCookie($name);
```


<a name="assert-has-plain-cookie"></a>
#### assertHasPlainCookie

斷言給定的未加密 Cookie 存在：

```php
$browser->assertHasPlainCookie($name);
```


<a name="assert-cookie-missing"></a>
#### assertCookieMissing

斷言給定的已加密 Cookie 不存在：

```php
$browser->assertCookieMissing($name);
```


<a name="assert-plain-cookie-missing"></a>
#### assertPlainCookieMissing

斷言給定的未加密 Cookie 不存在：

```php
$browser->assertPlainCookieMissing($name);
```


<a name="assert-cookie-value"></a>
#### assertCookieValue

斷言一個已加密的 Cookie 具有給定的值：

```php
$browser->assertCookieValue($name, $value);
```


<a name="assert-plain-cookie-value"></a>
#### assertPlainCookieValue

斷言一個未加密的 Cookie 具有給定的值：

```php
$browser->assertPlainCookieValue($name, $value);
```


<a name="assert-see"></a>
#### assertSee

斷言給定的文字存在於頁面上：

```php
$browser->assertSee($text);
```


<a name="assert-dont-see"></a>
#### assertDontSee

斷言給定的文字不存在於頁面上：

```php
$browser->assertDontSee($text);
```


<a name="assert-see-in"></a>
#### assertSeeIn

斷言給定的文字存在於選擇器內：

```php
$browser->assertSeeIn($selector, $text);
```


<a name="assert-dont-see-in"></a>
#### assertDontSeeIn

斷言給定的文字不存在於選擇器內：

```php
$browser->assertDontSeeIn($selector, $text);
```


<a name="assert-see-anything-in"></a>
#### assertSeeAnythingIn

斷言選擇器內存在任何文字：

```php
$browser->assertSeeAnythingIn($selector);
```


<a name="assert-see-nothing-in"></a>
#### assertSeeNothingIn

斷言選擇器內沒有任何文字：

```php
$browser->assertSeeNothingIn($selector);
```


<a name="assert-count"></a>
#### assertCount

斷言符合給定選擇器的元素出現了指定的次數：

```php
$browser->assertCount($selector, $count);
```


<a name="assert-script"></a>
#### assertScript

斷言給定的 JavaScript 運算式求值結果為給定的值：

```php
$browser->assertScript('window.isLoaded')
    ->assertScript('document.readyState', 'complete');
```


<a name="assert-source-has"></a>
#### assertSourceHas

斷言給定的原始碼存在於頁面上：

```php
$browser->assertSourceHas($code);
```


<a name="assert-source-missing"></a>
#### assertSourceMissing

斷言給定的原始碼不存在於頁面上：

```php
$browser->assertSourceMissing($code);
```


<a name="assert-see-link"></a>
#### assertSeeLink

斷言給定的連結存在於頁面上：

```php
$browser->assertSeeLink($linkText);
```


<a name="assert-dont-see-link"></a>
#### assertDontSeeLink

斷言給定的連結不存在於頁面上：

```php
$browser->assertDontSeeLink($linkText);
```


<a name="assert-input-value"></a>
#### assertInputValue

斷言給定的輸入欄位具有給定的值：

```php
$browser->assertInputValue($field, $value);
```


<a name="assert-input-value-is-not"></a>
#### assertInputValueIsNot

斷言給定的輸入欄位不具有給定的值：

```php
$browser->assertInputValueIsNot($field, $value);
```


<a name="assert-checked"></a>
#### assertChecked

斷言給定的核取方塊 (checkbox) 已被勾選：

```php
$browser->assertChecked($field);
```


<a name="assert-not-checked"></a>
#### assertNotChecked

斷言給定的核取方塊未被勾選：

```php
$browser->assertNotChecked($field);
```


<a name="assert-indeterminate"></a>
#### assertIndeterminate

斷言給定的核取方塊處於不確定狀態 (indeterminate state)：

```php
$browser->assertIndeterminate($field);
```


<a name="assert-radio-selected"></a>
#### assertRadioSelected

斷言給定的選項按鈕 (radio field) 已被選取：

```php
$browser->assertRadioSelected($field, $value);
```


<a name="assert-radio-not-selected"></a>
#### assertRadioNotSelected

斷言給定的選項按鈕未被選取：

```php
$browser->assertRadioNotSelected($field, $value);
```


<a name="assert-selected"></a>
#### assertSelected

斷言給定的下拉式選單已選取給定的值：

```php
$browser->assertSelected($field, $value);
```


<a name="assert-not-selected"></a>
#### assertNotSelected

斷言給定的下拉式選單未選取給定的值：

```php
$browser->assertNotSelected($field, $value);
```


<a name="assert-select-has-options"></a>
#### assertSelectHasOptions

斷言給定的值陣列可在下拉式選單中選取：

```php
$browser->assertSelectHasOptions($field, $values);
```


<a name="assert-select-missing-options"></a>
#### assertSelectMissingOptions

斷言給定的值陣列無法在下拉式選單中選取：

```php
$browser->assertSelectMissingOptions($field, $values);
```


<a name="assert-select-has-option"></a>
#### assertSelectHasOption

斷言給定的值可在給定的欄位中選取：

```php
$browser->assertSelectHasOption($field, $value);
```


<a name="assert-select-missing-option"></a>
#### assertSelectMissingOption

斷言給定的值無法選取：

```php
$browser->assertSelectMissingOption($field, $value);
```


<a name="assert-value"></a>
#### assertValue

斷言符合給定選擇器的元素具有給定的值：

```php
$browser->assertValue($selector, $value);
```


<a name="assert-value-is-not"></a>
#### assertValueIsNot

斷言符合給定選擇器的元素不具有給定的值：

```php
$browser->assertValueIsNot($selector, $value);
```


<a name="assert-attribute"></a>
#### assertAttribute

斷言符合給定選擇器的元素在所提供的屬性中具有給定的值：

```php
$browser->assertAttribute($selector, $attribute, $value);
```


<a name="assert-attribute-missing"></a>
#### assertAttributeMissing

斷言符合給定選擇器的元素缺少所提供的屬性：

```php
$browser->assertAttributeMissing($selector, $attribute);
```


<a name="assert-attribute-contains"></a>
#### assertAttributeContains

斷言符合給定選擇器的元素在所提供的屬性中包含給定的值：

```php
$browser->assertAttributeContains($selector, $attribute, $value);
```


<a name="assert-attribute-doesnt-contain"></a>
#### assertAttributeDoesntContain

斷言符合給定選擇器的元素在所提供的屬性中不包含給定的值：

```php
$browser->assertAttributeDoesntContain($selector, $attribute, $value);
```


<a name="assert-aria-attribute"></a>
#### assertAriaAttribute

斷言符合給定選擇器的元素在所提供的 aria 屬性中具有給定的值：

```php
$browser->assertAriaAttribute($selector, $attribute, $value);
```

例如，給定標記 `<button aria-label="Add"></button>`，你可以像這樣對 `aria-label` 屬性進行斷言：

```php
$browser->assertAriaAttribute('button', 'label', 'Add')
```


<a name="assert-data-attribute"></a>
#### assertDataAttribute

斷言符合給定選擇器的元素在所提供的 data 屬性中具有給定的值：

```php
$browser->assertDataAttribute($selector, $attribute, $value);
```

例如，給定標記 `<tr id="row-1" data-content="attendees"></tr>`，你可以像這樣對 `data-label` 屬性進行斷言：

```php
$browser->assertDataAttribute('#row-1', 'content', 'attendees')
```


<a name="assert-visible"></a>
#### assertVisible

斷言符合給定選擇器的元素是可見的：

```php
$browser->assertVisible($selector);
```


<a name="assert-present"></a>
#### assertPresent

斷言符合給定選擇器的元素存在於原始碼中：

```php
$browser->assertPresent($selector);
```


<a name="assert-not-present"></a>
#### assertNotPresent

斷言符合給定選擇器的元素不存在於原始碼中：

```php
$browser->assertNotPresent($selector);
```


<a name="assert-missing"></a>
#### assertMissing

斷言符合給定選擇器的元素是不可見的：

```php
$browser->assertMissing($selector);
```


<a name="assert-input-present"></a>
#### assertInputPresent

斷言具有給定名稱的輸入欄位存在：

```php
$browser->assertInputPresent($name);
```


<a name="assert-input-missing"></a>
#### assertInputMissing

斷言具有給定名稱的輸入欄位不存在於原始碼中：

```php
$browser->assertInputMissing($name);
```


<a name="assert-dialog-opened"></a>
#### assertDialogOpened

斷言已開啟一個帶有給定訊息的 JavaScript 對話方塊：

```php
$browser->assertDialogOpened($message);
```


<a name="assert-enabled"></a>
#### assertEnabled

斷言給定的欄位是啟用的：

```php
$browser->assertEnabled($field);
```


<a name="assert-disabled"></a>
#### assertDisabled

斷言給定的欄位是禁用的：

```php
$browser->assertDisabled($field);
```


<a name="assert-button-enabled"></a>
#### assertButtonEnabled

斷言給定的按鈕是啟用的：

```php
$browser->assertButtonEnabled($button);
```


<a name="assert-button-disabled"></a>
#### assertButtonDisabled

斷言給定的按鈕是禁用的：

```php
$browser->assertButtonDisabled($button);
```


<a name="assert-focused"></a>
#### assertFocused

斷言給定的欄位是聚焦的：

```php
$browser->assertFocused($field);
```


<a name="assert-not-focused"></a>
#### assertNotFocused

斷言給定的欄位不是聚焦的：

```php
$browser->assertNotFocused($field);
```


<a name="assert-authenticated"></a>
#### assertAuthenticated

斷言使用者已通過認證：

```php
$browser->assertAuthenticated();
```


<a name="assert-guest"></a>
#### assertGuest

斷言使用者未通過認證：

```php
$browser->assertGuest();
```


<a name="assert-authenticated-as"></a>
#### assertAuthenticatedAs

斷言使用者已通過認證為給定的使用者：

```php
$browser->assertAuthenticatedAs($user);
```


<a name="assert-vue"></a>
#### assertVue

Dusk 甚至可以讓你對 [Vue component](https://vuejs.org) 的資料狀態進行斷言。例如，假設你的應用程式包含以下 Vue 元件：

    // HTML...

    <profile dusk="profile-component"></profile>

    // Component Definition...

    Vue.component('profile', {
        template: '<div>{{ user.name }}</div>',

        data: function () {
            return {
                user: {
                    name: 'Taylor'
                }
            };
        }
    });

你可以像這樣對 Vue 元件的狀態進行斷言：

```php tab=Pest
test('vue', function () {
    $this->browse(function (Browser $browser) {
        $browser->visit('/')
            ->assertVue('user.name', 'Taylor', '@profile-component');
    });
});
```

```php tab=PHPUnit
/**
 * A basic Vue test example.
 */
public function test_vue(): void
{
    $this->browse(function (Browser $browser) {
        $browser->visit('/')
            ->assertVue('user.name', 'Taylor', '@profile-component');
    });
}
```


<a name="assert-vue-is-not"></a>
#### assertVueIsNot

斷言給定的 Vue 元件資料屬性不符合給定的值：

```php
$browser->assertVueIsNot($property, $value, $componentSelector = null);
```


<a name="assert-vue-contains"></a>
#### assertVueContains

斷言給定的 Vue 元件資料屬性是一個陣列且包含給定的值：

```php
$browser->assertVueContains($property, $value, $componentSelector = null);
```


<a name="assert-vue-doesnt-contain"></a>
#### assertVueDoesntContain

斷言給定的 Vue 元件資料屬性是一個陣列且不包含給定的值：

```php
$browser->assertVueDoesntContain($property, $value, $componentSelector = null);
```

<a name="pages"></a>
## 頁面

有時，測試會需要依序執行數個複雜的動作。這會讓測試更難閱讀與理解。Dusk Pages 能讓我們定義一些富有表達力的動作，並能透過單一方法在給定頁面上執行。Pages 也讓我們能為應用程式或單一頁面定義常用選擇器的捷徑。

<a name="generating-pages"></a>
### 產生頁面

若要產生頁面物件，請執行 `dusk:page` 這個 Artisan 指令。所有頁面物件都會被放在應用程式的 `tests/Browser/Pages` 目錄中：

```shell
php artisan dusk:page Login
```

<a name="configuring-pages"></a>
### 設定頁面

預設情況下，Page 有三個方法：`url`、`assert`、`elements`。我們現在先來討論 `url` 與 `assert` 方法。`elements` 方法則會在[下面](#shorthand-selectors)詳細討論。

<a name="the-url-method"></a>
#### `url` 方法

`url` 方法應回傳代表該頁面的 URL 路徑。Dusk 在導覽至瀏覽器中的該頁面時會使用此 URL：

```php
/**
 * Get the URL for the page.
 */
public function url(): string
{
    return '/login';
}
```

<a name="the-assert-method"></a>
#### `assert` 方法

`assert` 方法可用來進行任何必要的斷言，以驗證瀏覽器是否真的在給定的頁面上。雖然不一定要在此方法中放置任何東西，但如果想的話可以自由地進行這些斷言。在導覽至該頁面時，這些斷言會自動執行：

```php
/**
 * Assert that the browser is on the page.
 */
public function assert(Browser $browser): void
{
    $browser->assertPathIs($this->url());
}
```

<a name="navigating-to-pages"></a>
### 導覽至頁面

定義好頁面後，就可以用 `visit` 方法來導覽至該頁面：

```php
use Tests\Browser\Pages\Login;

$browser->visit(new Login);
```

有時，我們可能已經在某個頁面上，而需要將該頁面的選擇器與方法「載入」到目前的測試情境中。這種情況在按下按鈕後被重新導向到某個頁面，而沒有明確導覽至該頁面時很常見。在這種情況下，我們可以使用 `on` 方法來載入頁面：

```php
use Tests\Browser\Pages\CreatePlaylist;

$browser->visit('/dashboard')
    ->clickLink('Create Playlist')
    ->on(new CreatePlaylist)
    ->assertSee('@create');
```

<a name="shorthand-selectors"></a>
### 選擇器簡寫

在 Page Class 中的 `elements` 方法能讓我們為頁面上的任何 CSS 選擇器定義快速、好記的捷徑。舉例來說，我們來為應用程式登入頁面上的「email」輸入欄位定義一個捷徑：

```php
/**
 * Get the element shortcuts for the page.
 *
 * @return array<string, string>
 */
public function elements(): array
{
    return [
        '@email' => 'input[name=email]',
    ];
}
```

定義好捷徑後，就可以在任何通常會使用完整 CSS 選擇器的地方使用這個簡寫選擇器：

```php
$browser->type('@email', 'taylor@laravel.com');
```

<a name="global-shorthand-selectors"></a>
#### 全域選擇器簡寫

安裝好 Dusk 後，`tests/Browser/Pages` 目錄中會有一個基礎的 `Page` Class。這個 Class 包含了一個 `siteElements` 方法，可用來定義全域選擇器簡寫，這些簡寫在整個應用程式中的每個頁面上都有效：

```php
/**
 * Get the global element shortcuts for the site.
 *
 * @return array<string, string>
 */
public static function siteElements(): array
{
    return [
        '@element' => '#selector',
    ];
}
```

<a name="page-methods"></a>
### 頁面方法

除了頁面上預設定義的方法外，我們還可以定義額外的方法，並在測試中使用。舉例來說，假設我們正在建立一個音樂管理應用程式。在應用程式的某個頁面上，一個常見的動作可能是建立播放清單。與其在每個測試中都重寫建立播放清單的邏輯，不如在 Page Class 上定義一個 `createPlaylist` 方法：

```php
<?php

namespace Tests\Browser\Pages;

use Laravel\Dusk\Browser;
use Laravel\Dusk\Page;

class Dashboard extends Page
{
    // Other page methods...

    /**
     * Create a new playlist.
     */
    public function createPlaylist(Browser $browser, string $name): void
    {
        $browser->type('name', $name)
            ->check('share')
            ->press('Create Playlist');
    }
}
```

定義好方法後，就可以在任何使用到該頁面的測試中使用。瀏覽器實體會自動被傳入為自訂頁面方法的第一個引數：

```php
use Tests\Browser\Pages\Dashboard;

$browser->visit(new Dashboard)
    ->createPlaylist('My Playlist')
    ->assertSee('My Playlist');
```

<a name="components"></a>
## 元件

元件與 Dusk 的「頁面物件」類似，但它們是用於在整個應用程式中重複使用的 UI 片段和功能，例如導覽列或通知視窗。因此，元件不與特定的 URL 綁定。

<a name="generating-components"></a>
### 產生元件

要產生元件，請執行 `dusk:component` Artisan 指令。新元件會被放置在 `tests/Browser/Components` 目錄中：

```shell
php artisan dusk:component DatePicker
```

如上所示，「日期選擇器」是在您的應用程式中可能存在於各種頁面上的元件範例。在您的測試套件中，為數十個測試手動撰寫選擇日期的瀏覽器自動化邏輯可能會變得很麻煩。相反地，我們可以定義一個 Dusk 元件來代表日期選擇器，從而將該邏輯封裝在元件中：

```php
<?php

namespace Tests\Browser\Components;

use Laravel\Dusk\Browser;
use Laravel\Dusk\Component as BaseComponent;

class DatePicker extends BaseComponent
{
    /**
     * Get the root selector for the component.
     */
    public function selector(): string
    {
        return '.date-picker';
    }

    /**
     * Assert that the browser page contains the component.
     */
    public function assert(Browser $browser): void
    {
        $browser->assertVisible($this->selector());
    }

    /**
     * Get the element shortcuts for the component.
     *
     * @return array<string, string>
     */
    public function elements(): array
    {
        return [
            '@date-field' => 'input.datepicker-input',
            '@year-list' => 'div > div.datepicker-years',
            '@month-list' => 'div > div.datepicker-months',
            '@day-list' => 'div > div.datepicker-days',
        ];
    }

    /**
     * Select the given date.
     */
    public function selectDate(Browser $browser, int $year, int $month, int $day): void
    {
        $browser->click('@date-field')
            ->within('@year-list', function (Browser $browser) use ($year) {
                $browser->click($year);
            })
            ->within('@month-list', function (Browser $browser) use ($month) {
                $browser->click($month);
            })
            ->within('@day-list', function (Browser $browser) use ($day) {
                $browser->click($day);
            });
    }
}
```

<a name="using-components"></a>
### 使用元件

一旦定義了元件，我們就可以在任何測試中輕鬆地在日期選擇器內選取日期。而且，如果選取日期所需的邏輯發生變化，我們只需要更新該元件即可：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\Browser\Components\DatePicker;

pest()->use(DatabaseMigrations::class);

test('basic example', function () {
    $this->browse(function (Browser $browser) {
        $browser->visit('/')
            ->within(new DatePicker, function (Browser $browser) {
                $browser->selectDate(2019, 1, 30);
            })
            ->assertSee('January');
    });
});
```

```php tab=PHPUnit
<?php

namespace Tests\Browser;

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\Browser\Components\DatePicker;
use Tests\DuskTestCase;

class ExampleTest extends DuskTestCase
{
    /**
     * A basic component test example.
     */
    public function test_basic_example(): void
    {
        $this->browse(function (Browser $browser) {
            $browser->visit('/')
                ->within(new DatePicker, function (Browser $browser) {
                    $browser->selectDate(2019, 1, 30);
                })
                ->assertSee('January');
        });
    }
}
```

`component` 方法可用於擷取一個範圍限定在給定元件的瀏覽器實例：

```php
$datePicker = $browser->component(new DatePickerComponent);

$datePicker->selectDate(2019, 1, 30);

$datePicker->assertSee('January');
```

<a name="continuous-integration"></a>
## 持續整合

> [!WARNING]
> 大多數 Dusk 持續整合設定都預期您的 Laravel 應用程式會透過內建的 PHP 開發伺服器在 8000 連接埠上提供服務。因此，在繼續之前，您應確保您的持續整合環境中 `APP_URL` 環境變數的值為 `http://127.0.0.1:8000`。

<a name="running-tests-on-heroku-ci"></a>
### Heroku CI

要在 [Heroku CI](https://www.heroku.com/continuous-integration) 上執行 Dusk 測試，請將下列 Google Chrome buildpack 和腳本新增至您的 Heroku `app.json` 檔案中：

```json
{
  "environments": {
    "test": {
      "buildpacks": [
        { "url": "heroku/php" },
        { "url": "https://github.com/heroku/heroku-buildpack-chrome-for-testing" }
      ],
      "scripts": {
        "test-setup": "cp .env.testing .env",
        "test": "nohup bash -c './vendor/laravel/dusk/bin/chromedriver-linux --port=9515 > /dev/null 2>&1 &' && nohup bash -c 'php artisan serve --no-reload > /dev/null 2>&1 &' && php artisan dusk"
      }
    }
  }
}
```

<a name="running-tests-on-travis-ci"></a>
### Travis CI

要在 [Travis CI](https://travis-ci.org) 上執行您的 Dusk 測試，請使用下列 `.travis.yml` 設定。由於 Travis CI 不是圖形化環境，我們需要採取一些額外的步驟來啟動 Chrome 瀏覽器。此外，我們將使用 `php artisan serve` 來啟動 PHP 的內建網頁伺服器：

```yaml
language: php

php:
  - 8.2

addons:
  chrome: stable

install:
  - cp .env.testing .env
  - travis_retry composer install --no-interaction --prefer-dist
  - php artisan key:generate
  - php artisan dusk:chrome-driver

before_script:
  - google-chrome-stable --headless --disable-gpu --remote-debugging-port=9222 http://localhost &
  - php artisan serve --no-reload &

script:
  - php artisan dusk
```

<a name="running-tests-on-github-actions"></a>
### GitHub Actions

如果您正在使用 [GitHub Actions](https://github.com/features/actions) 來執行您的 Dusk 測試，您可以使用下列設定檔作為起點。與 TravisCI 類似，我們將使用 `php artisan serve` 指令來啟動 PHP 的內建網頁伺服器：

```yaml
name: CI
on: [push]
jobs:

  dusk-php:
    runs-on: ubuntu-latest
    env:
      APP_URL: "http://127.0.0.1:8000"
      DB_USERNAME: root
      DB_PASSWORD: root
      MAIL_MAILER: log
    steps:
      - uses: actions/checkout@v5
      - name: Prepare The Environment
        run: cp .env.example .env
      - name: Create Database
        run: |
          sudo systemctl start mysql
          mysql --user="root" --password="root" -e "CREATE DATABASE \`my-database\` character set UTF8mb4 collate utf8mb4_bin;"
      - name: Install Composer Dependencies
        run: composer install --no-progress --prefer-dist --optimize-autoloader
      - name: Generate Application Key
        run: php artisan key:generate
      - name: Upgrade Chrome Driver
        run: php artisan dusk:chrome-driver --detect
      - name: Start Chrome Driver
        run: ./vendor/laravel/dusk/bin/chromedriver-linux --port=9515 &
      - name: Run Laravel Server
        run: php artisan serve --no-reload &
      - name: Run Dusk Tests
        run: php artisan dusk
      - name: Upload Screenshots
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: screenshots
          path: tests/Browser/screenshots
      - name: Upload Console Logs
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: console
          path: tests/Browser/console
```

<a name="running-tests-on-chipper-ci"></a>
### Chipper CI

如果您正在使用 [Chipper CI](https://chipperci.com) 來執行 Dusk 測試，您可以使用下列設定檔作為起點。我們將使用 PHP 的內建伺服器來執行 Laravel，以便我們可以監聽請求：

```yaml
# file .chipperci.yml
version: 1

environment:
  php: 8.2
  node: 16


# Include Chrome in the build environment
services:
  - dusk


# Build all commits
on:
   push:
      branches: .*

pipeline:
  - name: Setup
    cmd: |
      cp -v .env.example .env
      composer install --no-interaction --prefer-dist --optimize-autoloader
      php artisan key:generate

      # Create a dusk env file, ensuring APP_URL uses BUILD_HOST
      cp -v .env .env.dusk.ci
      sed -i "s@APP_URL=.*@APP_URL=http://$BUILD_HOST:8000@g" .env.dusk.ci

  - name: Compile Assets
    cmd: |
      npm ci --no-audit
      npm run build

  - name: Browser Tests
    cmd: |
      php -S [::0]:8000 -t public 2>server.log &
      sleep 2
      php artisan dusk:chrome-driver $CHROME_DRIVER
      php artisan dusk --env=ci
```

若要進一步了解如何在 Chipper CI 上執行 Dusk 測試，包含如何使用資料庫，請參閱 [Chipper CI 官方文件](https://chipperci.com/docs/testing/laravel-dusk-new/)。