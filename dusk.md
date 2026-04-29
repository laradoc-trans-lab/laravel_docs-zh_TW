# Laravel Dusk

- [簡介](#introduction)
- [安裝](#installation)
    - [管理 ChromeDriver 安裝](#managing-chromedriver-installations)
    - [使用其他瀏覽器](#using-other-browsers)
- [入門指南](#getting-started)
    - [產生測試](#generating-tests)
    - [每次測試後重設資料庫](#resetting-the-database-after-each-test)
    - [執行測試](#running-tests)
    - [環境處理](#environment-handling)
- [瀏覽器基礎](#browser-basics)
    - [建立瀏覽器](#creating-browsers)
    - [導覽](#navigation)
    - [調整瀏覽器視窗大小](#resizing-browser-windows)
    - [瀏覽器巨集 (Macros)](#browser-macros)
    - [認證](#authentication)
    - [Cookie](#cookies)
    - [執行 JavaScript](#executing-javascript)
    - [擷取螢幕截圖](#taking-a-screenshot)
    - [將主控台輸出儲存至磁碟](#storing-console-output-to-disk)
    - [將網頁原始碼儲存至磁碟](#storing-page-source-to-disk)
- [與元素互動](#interacting-with-elements)
    - [Dusk 選擇器](#dusk-selectors)
    - [文字、數值與屬性](#text-values-and-attributes)
    - [與表單互動](#interacting-with-forms)
    - [附加檔案](#attaching-files)
    - [按下按鈕](#pressing-buttons)
    - [點擊連結](#clicking-links)
    - [使用鍵盤](#using-the-keyboard)
    - [使用滑鼠](#using-the-mouse)
    - [JavaScript 對話框](#javascript-dialogs)
    - [與內嵌框架 (iframes) 互動](#interacting-with-iframes)
    - [限定選擇器範圍](#scoping-selectors)
    - [等待元素](#waiting-for-elements)
    - [將元素捲動至可見範圍](#scrolling-an-element-into-view)
- [可用的斷言 (Assertions)](#available-assertions)
- [頁面 (Pages)](#pages)
    - [產生頁面](#generating-pages)
    - [設定頁面](#configuring-pages)
    - [導覽至頁面](#navigating-to-pages)
    - [速記選擇器](#shorthand-selectors)
    - [頁面方法](#page-methods)
- [元件 (Components)](#components)
    - [產生元件](#generating-components)
    - [使用元件](#using-components)
- [持續整合 (CI)](#continuous-integration)
    - [Heroku CI](#running-tests-on-heroku-ci)
    - [Travis CI](#running-tests-on-travis-ci)
    - [GitHub Actions](#running-tests-on-github-actions)
    - [Chipper CI](#running-tests-on-chipper-ci)

<a name="introduction"></a>
## 簡介

> [!WARNING]
> [Pest 4](https://pestphp.com/) 現在包含了自動化瀏覽器測試，與 Laravel Dusk 相比，它在效能和易用性方面有顯著的提升。對於新專案，我們建議使用 Pest 進行瀏覽器測試。

[Laravel Dusk](https://github.com/laravel/dusk) 提供了一套富有表現力且易於使用的瀏覽器自動化與測試 API。預設情況下，Dusk 不需要你在本地電腦安裝 JDK 或 Selenium。相反地，Dusk 使用獨立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安裝。然而，你也可以自由地使用任何其他相容於 Selenium 的驅動程式。


<a name="installation"></a>
## 安裝

要開始使用，你應該安裝 [Google Chrome](https://www.google.com/chrome) 並將 `laravel/dusk` Composer 依賴新增到你的專案中：

```shell
composer require laravel/dusk --dev
```

> [!WARNING]
> 如果你是手動註冊 Dusk 的服務提供者(Service Providers)，則**絕對不要**在正式環境中註冊它，因為這樣做可能會導致任意使用者能夠通過你的應用程式認證。

安裝好 Dusk 套件後，執行 `dusk:install` Artisan 指令。`dusk:install` 指令會建立一個 `tests/Browser` 目錄、一個範例 Dusk 測試，並為你的作業系統安裝 Chrome Driver 二進位檔：

```shell
php artisan dusk:install
```

接著，在應用程式的 `.env` 檔案中設定 `APP_URL` 環境變數。此數值應與你在瀏覽器中存取應用程式時使用的 URL 一致。

> [!NOTE]
> 如果你正在使用 [Laravel Sail](/docs/{{version}}/sail) 管理你的本地開發環境，請同時參閱 Sail 說明文件中有關[設定與執行 Dusk 測試](/docs/{{version}}/sail#laravel-dusk)的部分。


<a name="managing-chromedriver-installations"></a>
### 管理 ChromeDriver 安裝

如果你想安裝與 Laravel Dusk 透過 `dusk:install` 指令安裝的不同的 ChromeDriver 版本，可以使用 `dusk:chrome-driver` 指令：

```shell
# Install the latest version of ChromeDriver for your OS...
php artisan dusk:chrome-driver

# Install a given version of ChromeDriver for your OS...
php artisan dusk:chrome-driver 86

# Install a given version of ChromeDriver for all supported OSs...
php artisan dusk:chrome-driver --all

# Install the version of ChromeDriver that matches the detected version of Chrome / Chromium for your OS...
php artisan dusk:chrome-driver --detect
```

> [!WARNING]
> Dusk 要求 `chromedriver` 二進位檔必須是可執行的。如果你在執行 Dusk 時遇到問題，應確保使用以下指令將二進位檔設為可執行：`chmod -R 0755 vendor/laravel/dusk/bin/`。


<a name="using-other-browsers"></a>
### 使用其他瀏覽器

預設情況下，Dusk 使用 Google Chrome 和獨立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安裝來執行瀏覽器測試。然而，你也可以啟動自己的 Selenium 伺服器，並針對任何你想要的瀏覽器執行測試。

要開始使用，請開啟你的 `tests/DuskTestCase.php` 檔案，這是你應用程式的基礎 Dusk 測試案例類別。在此檔案中，你可以移除對 `startChromeDriver` 方法的呼叫。這將停止 Dusk 自動啟動 ChromeDriver：

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

接著，你可以修改 `driver` 方法以連接到你選擇的 URL 和連接埠。此外，你還可以修改應傳遞給 WebDriver 的「偏好的功能設定 (desired capabilities)」：

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
## 入門指南


<a name="generating-tests"></a>
### 產生測試

要產生一個 Dusk 測試，請使用 `dusk:make` Artisan 指令。產生的測試將會放置在 `tests/Browser` 目錄中：

```shell
php artisan dusk:make LoginTest
```


<a name="resetting-the-database-after-each-test"></a>
### 每次測試後重設資料庫

你撰寫的大多數測試都會與從應用程式資料庫中檢索資料的頁面進行互動；然而，你的 Dusk 測試絕不應該使用 `RefreshDatabase` trait。`RefreshDatabase` trait 利用了資料庫交易，這在跨 HTTP 請求時並不適用或無法使用。相反地，你有兩個選擇：`DatabaseMigrations` trait 和 `DatabaseTruncation` trait。


<a name="reset-migrations"></a>
#### 使用資料庫遷移

`DatabaseMigrations` trait 會在每次測試前執行你的資料庫遷移。然而，在每次測試時刪除並重新建立資料庫資料表通常比截斷 (Truncating) 資料表慢：

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
> 執行 Dusk 測試時無法使用 SQLite 記憶體資料庫。由於瀏覽器是在其自身的行程(Processes)中執行的，它將無法存取其他行程的記憶體資料庫。


<a name="reset-truncation"></a>
#### 使用資料庫截斷 (Database Truncation)

`DatabaseTruncation` trait 會在第一次測試時遷移你的資料庫，以確保你的資料庫資料表已正確建立。然而，在後續的測試中，資料庫的資料表將僅被截斷 (Truncated) —— 這比重新執行所有資料庫遷移提供了更快的速度：

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

預設情況下，此 trait 會截斷除 `migrations` 表以外的所有資料表。如果你想自定義要截斷的資料表，可以在測試類別上定義 `$tablesToTruncate` 屬性：

> [!NOTE]
> 如果你正在使用 Pest，你應該在基礎的 `DuskTestCase` 類別或你的測試檔案所繼承的任何類別上定義屬性或方法。

```php
/**
 * Indicates which tables should be truncated.
 *
 * @var array
 */
protected $tablesToTruncate = ['users'];
```

或者，你可以在測試類別上定義 `$exceptTables` 屬性，以指定哪些資料表應排除在截斷之外：

```php
/**
 * Indicates which tables should be excluded from truncation.
 *
 * @var array
 */
protected $exceptTables = ['users'];
```

要指定應截斷其資料表的資料庫連線，你可以在測試類別上定義 `$connectionsToTruncate` 屬性：

```php
/**
 * Indicates which connections should have their tables truncated.
 *
 * @var array
 */
protected $connectionsToTruncate = ['mysql'];
```

如果你想在資料庫截斷執行之前或之後執行程式碼，可以在測試類別上定義 `beforeTruncatingDatabase` 或 `afterTruncatingDatabase` 方法：

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

要執行你的瀏覽器測試，請執行 `dusk` Artisan 指令：

```shell
php artisan dusk
```

如果你上次執行 `dusk` 指令時有測試失敗，可以使用 `dusk:fails` 指令先重新執行失敗的測試以節省時間：

```shell
php artisan dusk:fails
```

`dusk` 指令接受 Pest / PHPUnit 測試執行器通常接受的任何引數，例如允許你僅執行給定 [群組 (Group)](https://docs.phpunit.de/en/10.5/annotations.html#group) 的測試：

```shell
php artisan dusk --group=foo
```

> [!NOTE]
> 如果你使用 [Laravel Sail](/docs/{{version}}/sail) 來管理你的本地開發環境，請參閱 Sail 檔案中有關[設定與執行 Dusk 測試](/docs/{{version}}/sail#laravel-dusk)的部分。


<a name="manually-starting-chromedriver"></a>
#### 手動啟動 ChromeDriver

根據預設，Dusk 會自動嘗試啟動 ChromeDriver。如果這在你的特定系統上無法運作，你可以在執行 `dusk` 指令之前手動啟動 ChromeDriver。如果你選擇手動啟動 ChromeDriver，你應該註解掉 `tests/DuskTestCase.php` 檔案中的以下這行：

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

此外，如果你在 9515 以外的連接埠啟動 ChromeDriver，則應修改同一類別的 `driver` 方法以反映正確的連接埠：

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
### 環境處理

要強制 Dusk 在執行測試時使用其專屬的環境檔案，請在專案根目錄建立一個 `.env.dusk.{environment}` 檔案。例如，如果你將從 `local` 環境啟動 `dusk` 指令，則應建立一個 `.env.dusk.local` 檔案。

執行測試時，Dusk 會備份你的 `.env` 檔案並將你的 Dusk 環境重新命名為 `.env`。測試完成後，你的 `.env` 檔案將會被還原。

<a name="browser-basics"></a>
## 瀏覽器基礎

<a name="creating-browsers"></a>
### 建立瀏覽器

首先，讓我們編寫一個驗證能否登入應用程式的測試。產生測試後，我們可以修改它以導覽至登入頁面、輸入憑證，並點擊「Login」按鈕。若要建立瀏覽器執行個體，你可以在 Dusk 測試中呼叫 `browse` 方法：

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

如上例所示，`browse` 方法接受一個閉包。Dusk 會自動將瀏覽器執行個體傳遞給此閉包，它是與應用程式互動並進行斷言的主要物件。

<a name="creating-multiple-browsers"></a>
#### 建立多個瀏覽器

有時你可能需要多個瀏覽器才能正確執行測試。例如，測試與 WebSocket 互動的聊天畫面時，可能需要多個瀏覽器。若要建立多個瀏覽器，只需在傳遞給 `browse` 方法的閉包簽章中增加更多瀏覽器引數即可：

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

`visit` 方法可用於導覽至應用程式內的特定 URI：

```php
$browser->visit('/login');
```

你可以使用 `visitRoute` 方法導覽至[具名路由](/docs/{{version}}/routing#named-routes)：

```php
$browser->visitRoute($routeName, $parameters);
```

你可以使用 `back` 與 `forward` 方法來進行「後退」與「前進」導覽：

```php
$browser->back();

$browser->forward();
```

你可以使用 `refresh` 方法來重新整理頁面：

```php
$browser->refresh();
```

<a name="resizing-browser-windows"></a>
### 調整瀏覽器視窗大小

你可以使用 `resize` 方法來調整瀏覽器視窗的大小：

```php
$browser->resize(1920, 1080);
```

`maximize` 方法可用於將瀏覽器視窗最大化：

```php
$browser->maximize();
```

`fitContent` 方法會調整瀏覽器視窗的大小以符合其內容的大小：

```php
$browser->fitContent();
```

當測試失敗時，Dusk 會在擷取螢幕截圖前自動調整瀏覽器大小以符合內容。你可以在測試中呼叫 `disableFitOnFailure` 方法來停用此功能：

```php
$browser->disableFitOnFailure();
```

你可以使用 `move` 方法將瀏覽器視窗移動到螢幕上的不同位置：

```php
$browser->move($x = 100, $y = 100);
```

<a name="browser-macros"></a>
### 瀏覽器巨集 (Macros)

如果你想定義一個可以在多種測試中重複使用的自定義瀏覽器方法，可以使用 `Browser` 類別上的 `macro` 方法。通常，你應該在[服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

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

`macro` 函式接受一個名稱作為第一個引數，以及一個閉包作為第二個引數。當在 `Browser` 執行個體上將巨集作為方法呼叫時，將執行該巨集的閉包：

```php
$this->browse(function (Browser $browser) use ($user) {
    $browser->visit('/pay')
        ->scrollToElement('#credit-card-details')
        ->assertSee('Enter Credit Card Details');
});
```

<a name="authentication"></a>
### 認證

你經常會測試需要認證的頁面。你可以使用 Dusk 的 `loginAs` 方法，以避免在每次測試期間都與應用程式的登入畫面互動。`loginAs` 方法接受與可認證模型相關聯的主鍵或一個可認證模型的執行個體：

```php
use App\Models\User;
use Laravel\Dusk\Browser;

$this->browse(function (Browser $browser) {
    $browser->loginAs(User::find(1))
        ->visit('/home');
});
```

> [!WARNING]
> 使用 `loginAs` 方法後，該使用者的 Session 將會在該檔案內的所有測試中維持。

<a name="cookies"></a>
### Cookie

你可以使用 `cookie` 方法來獲取或設定加密 Cookie 的值。預設情況下，Laravel 建立的所有 Cookie 都會經過加密：

```php
$browser->cookie('name');

$browser->cookie('name', 'Taylor');
```

你可以使用 `plainCookie` 方法來獲取或設定未加密 Cookie 的值：

```php
$browser->plainCookie('name');

$browser->plainCookie('name', 'Taylor');
```

你可以使用 `deleteCookie` 方法來刪除指定的 Cookie：

```php
$browser->deleteCookie('name');
```

<a name="executing-javascript"></a>
### 執行 JavaScript

你可以使用 `script` 方法在瀏覽器中執行任意的 JavaScript 語句：

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

你可以使用 `screenshot` 方法擷取螢幕截圖並以給定的檔名儲存。所有螢幕截圖都將儲存在 `tests/Browser/screenshots` 目錄中：

```php
$browser->screenshot('filename');
```

`responsiveScreenshots` 方法可用於在各種斷點下擷取一系列螢幕截圖：

```php
$browser->responsiveScreenshots('filename');
```

`screenshotElement` 方法可用於擷取頁面上特定元素的螢幕截圖：

```php
$browser->screenshotElement('#selector', 'filename');
```

<a name="storing-console-output-to-disk"></a>
### 將主控台輸出儲存至磁碟

你可以使用 `storeConsoleLog` 方法將當前瀏覽器的主控台輸出以給定的檔名寫入磁碟。主控台輸出將儲存在 `tests/Browser/console` 目錄中：

```php
$browser->storeConsoleLog('filename');
```

<a name="storing-page-source-to-disk"></a>
### 將網頁原始碼儲存至磁碟

你可以使用 `storeSource` 方法將當前頁面的原始碼以給定的檔名寫入磁碟。頁面原始碼將儲存在 `tests/Browser/source` 目錄中：

```php
$browser->storeSource('filename');
```

<a name="interacting-with-elements"></a>
## 與元素互動


<a name="dusk-selectors"></a>
### Dusk 選擇器

選擇良好的 CSS 選擇器來與元素互動是撰寫 Dusk 測試中最困難的部分之一。隨著時間推移，前端的變更可能會導致像下面這樣的 CSS 選擇器使你的測試失敗：

```html
// HTML...

<button>Login</button>
```

```php
// Test...

$browser->click('.login-page .container div > button');
```

Dusk 選擇器讓你專注於撰寫有效的測試，而不是去記憶 CSS 選擇器。要定義選擇器，請在你的 HTML 元素中加入 `dusk` 屬性。接著，在與 Dusk 瀏覽器互動時，為選擇器加上 `@` 前綴，即可在測試中操作該元素：

```html
// HTML...

<button dusk="login-button">Login</button>
```

```php
// Test...

$browser->click('@login-button');
```

如果需要，你可以透過 `selectorHtmlAttribute` 方法自訂 Dusk 選擇器使用的 HTML 屬性。通常，此方法應該在應用程式的 `AppServiceProvider` 中的 `boot` 方法內呼叫：

```php
use Laravel\Dusk\Dusk;

Dusk::selectorHtmlAttribute('data-dusk');
```


<a name="text-values-and-attributes"></a>
### 文字、數值與屬性


<a name="retrieving-setting-values"></a>
#### 取得與設定數值

Dusk 提供了多種方法來與頁面上元素的當前數值、顯示文字和屬性進行互動。例如，若要取得與給定 CSS 或 Dusk 選擇器相符的元素「數值 (value)」，請使用 `value` 方法：

```php
// Retrieve the value...
$value = $browser->value('selector');

// Set the value...
$browser->value('selector', 'value');
```

你可以使用 `inputValue` 方法來取得具有給定欄位名稱之輸入元素的「數值 (value)」：

```php
$value = $browser->inputValue('field');
```


<a name="retrieving-text"></a>
#### 取得文字

`text` 方法可用於取得與給定選擇器相符之元素的顯示文字：

```php
$text = $browser->text('selector');
```


<a name="retrieving-attributes"></a>
#### 取得屬性

最後，`attribute` 方法可以用來取得與給定選擇器相符之元素的屬性值：

```php
$attribute = $browser->attribute('selector', 'value');
```


<a name="interacting-with-forms"></a>
### 與表單互動


<a name="typing-values"></a>
#### 輸入數值

Dusk 提供了多種方法來與表單和輸入元素互動。首先，讓我們看一個在輸入欄位中輸入文字的範例：

```php
$browser->type('email', 'taylor@laravel.com');
```

請注意，雖然該方法在必要時可以接受一個 CSS 選擇器，但我們不一定要傳入 `type` 方法。如果沒有提供 CSS 選擇器，Dusk 將會搜尋具有給定 `name` 屬性的 `input` 或 `textarea` 欄位。

若要在不清除欄位內容的情況下附加文字，你可以使用 `append` 方法：

```php
$browser->type('tags', 'foo')
    ->append('tags', ', bar, baz');
```

你可以使用 `clear` 方法清除輸入框的數值：

```php
$browser->clear('email');
```

你可以使用 `typeSlowly` 方法指示 Dusk 慢速輸入。預設情況下，Dusk 在按鍵之間會暫停 100 毫秒。若要自訂按鍵之間的時間，你可以將相應的毫秒數作為該方法的第三個引數傳入：

```php
$browser->typeSlowly('mobile', '+1 (202) 555-5555');

$browser->typeSlowly('mobile', '+1 (202) 555-5555', 300);
```

你可以使用 `appendSlowly` 方法來慢速附加文字：

```php
$browser->type('tags', 'foo')
    ->appendSlowly('tags', ', bar, baz');
```


<a name="dropdowns"></a>
#### 下拉選單

若要選擇 `select` 元素上的可用數值，你可以使用 `select` 方法。與 `type` 方法一樣，`select` 方法不需要完整的 CSS 選擇器。將數值傳遞給 `select` 方法時，你應該傳遞底層的 option 數值，而不是顯示文字：

```php
$browser->select('size', 'Large');
```

你可以省略第二個引數來選擇隨機選項：

```php
$browser->select('size');
```

藉由提供一個陣列作為 `select` 方法的第二個引數，你可以指示該方法選擇多個選項：

```php
$browser->select('categories', ['Art', 'Music']);
```


<a name="checkboxes"></a>
#### 核取方塊

若要「勾選」一個核取方塊輸入，你可以使用 `check` 方法。與許多其他與輸入相關的方法一樣，不需要完整的 CSS 選擇器。如果找不到相符的 CSS 選擇器，Dusk 會搜尋具有相符 `name` 屬性的核取方塊：

```php
$browser->check('terms');
```

`uncheck` 方法可以用來「取消勾選」一個核取方塊輸入：

```php
$browser->uncheck('terms');
```


<a name="radio-buttons"></a>
#### 選項按鈕

若要「選擇」一個 `radio` 輸入選項，你可以使用 `radio` 方法。與許多其他與輸入相關的方法一樣，不需要完整的 CSS 選擇器。如果找不到相符的 CSS 選擇器，Dusk 會搜尋具有相符 `name` 和 `value` 屬性的 `radio` 輸入：

```php
$browser->radio('size', 'large');
```


<a name="attaching-files"></a>
### 附加檔案

`attach` 方法可用於將檔案附加到 `file` 輸入元素。與許多其他與輸入相關的方法一樣，不需要完整的 CSS 選擇器。如果找不到相符的 CSS 選擇器，Dusk 會搜尋具有相符 `name` 屬性的 `file` 輸入：

```php
$browser->attach('photo', __DIR__.'/photos/mountains.png');
```

> [!WARNING]
> attach 函式要求在你的伺服器上安裝並啟用 `Zip` PHP 延伸模組。


<a name="pressing-buttons"></a>
### 按下按鈕

`press` 方法可用於點擊頁面上的按鈕元素。提供給 `press` 方法的引數可以是按鈕的顯示文字，或是 CSS / Dusk 選擇器：

```php
$browser->press('Login');
```

提交表單時，許多應用程式會在按下按鈕後將其停用，並在表單提交的 HTTP 請求完成後重新啟用按鈕。若要按下按鈕並等待按鈕重新啟用，你可以使用 `pressAndWaitFor` 方法：

```php
// Press the button and wait a maximum of 5 seconds for it to be enabled...
$browser->pressAndWaitFor('Save');

// Press the button and wait a maximum of 1 second for it to be enabled...
$browser->pressAndWaitFor('Save', 1);
```


<a name="clicking-links"></a>
### 點擊連結

若要點擊連結，你可以在瀏覽器實例上使用 `clickLink` 方法。`clickLink` 方法會點擊具有給定顯示文字的連結：

```php
$browser->clickLink($linkText);
```

你可以使用 `seeLink` 方法來判斷頁面上是否可以看到具有給定顯示文字的連結：

```php
if ($browser->seeLink($linkText)) {
    // ...
}
```

> [!WARNING]
> 這些方法會與 jQuery 互動。如果頁面上沒有 jQuery，Dusk 會自動將其注入頁面，以便在測試期間可以使用。

<a name="using-the-keyboard"></a>
### 使用鍵盤

`keys` 方法允許你向指定元素提供比 `type` 方法更複雜的輸入序列。例如，你可以指示 Dusk 在輸入數值時按住修飾鍵。在此範例中，當 `taylor` 被輸入到符合指定選擇器的元素時，將會按住 `shift` 鍵。在 `taylor` 輸入完成後，將在不使用任何修飾鍵的情況下輸入 `swift`：

```php
$browser->keys('selector', ['{shift}', 'taylor'], 'swift');
```

`keys` 方法的另一個實用情境是向應用程式的主 CSS 選擇器發送「鍵盤快速鍵」組合：

```php
$browser->keys('.app', ['{command}', 'j']);
```

> [!NOTE]
> 所有修飾鍵（例如 `{command}`）都必須包圍在 `{}` 字元中，並對應到 `Facebook\WebDriver\WebDriverKeys` 類別中定義的常數，該類別可以在 [GitHub 上找到](https://github.com/php-webdriver/php-webdriver/blob/master/lib/WebDriverKeys.php)。


<a name="fluent-keyboard-interactions"></a>
#### 流暢的鍵盤互動

Dusk 還提供了 `withKeyboard` 方法，讓你能夠透過 `Laravel\Dusk\Keyboard` 類別流暢地執行複雜的鍵盤互動。`Keyboard` 類別提供了 `press`、`release`、`type` 與 `pause` 方法：

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
#### 鍵盤巨集 (Macros)

如果你想要定義可以在整個測試套件中輕鬆重複使用的自定義鍵盤互動，可以使用 `Keyboard` 類別提供的 `macro` 方法。通常，你應該在[服務提供者(Service Providers)](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

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

`macro` 函式接受名稱作為第一個引數，閉包作為第二個引數。當在 `Keyboard` 實例上將巨集作為方法呼叫時，將執行該巨集的閉包：

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

`click` 方法可用於點擊符合指定 CSS 或 Dusk 選擇器的元素：

```php
$browser->click('.selector');
```

`clickAtXPath` 方法可用於點擊符合指定 XPath 表達式的元素：

```php
$browser->clickAtXPath('//div[@class = "selector"]');
```

`clickAtPoint` 方法可用於點擊相對於瀏覽器可見區域的給定座標配對處最頂層的元素：

```php
$browser->clickAtPoint($x = 0, $y = 0);
```

`doubleClick` 方法可用於模擬滑鼠連點兩次：

```php
$browser->doubleClick();

$browser->doubleClick('.selector');
```

`rightClick` 方法可用於模擬滑鼠右鍵點擊：

```php
$browser->rightClick();

$browser->rightClick('.selector');
```

`clickAndHold` 方法可用於模擬滑鼠按鈕被點擊並按住。後續呼叫 `releaseMouse` 方法將撤銷此行為並放開滑鼠按鈕：

```php
$browser->clickAndHold('.selector');

$browser->clickAndHold()
    ->pause(1000)
    ->releaseMouse();
```

`controlClick` 方法可用於在瀏覽器內模擬 `ctrl+click` 事件：

```php
$browser->controlClick();

$browser->controlClick('.selector');
```


<a name="mouseover"></a>
#### 滑鼠移入 (Mouseover)

當你需要將滑鼠移動到符合指定 CSS 或 Dusk 選擇器的元素上方時，可以使用 `mouseover` 方法：

```php
$browser->mouseover('.selector');
```


<a name="drag-drop"></a>
#### 拖放 (Drag and Drop)

`drag` 方法可用於將符合指定選擇器的元素拖曳到另一個元素：

```php
$browser->drag('.from-selector', '.to-selector');
```

或者，你也可以沿單一方向拖曳元素：

```php
$browser->dragLeft('.selector', $pixels = 10);
$browser->dragRight('.selector', $pixels = 10);
$browser->dragUp('.selector', $pixels = 10);
$browser->dragDown('.selector', $pixels = 10);
```

最後，你也可以根據給定的位移量拖曳元素：

```php
$browser->dragOffset('.selector', $x = 10, $y = 10);
```


<a name="javascript-dialogs"></a>
### JavaScript 對話框

Dusk 提供了多種方法與 JavaScript 對話框互動。例如，你可以使用 `waitForDialog` 方法等待 JavaScript 對話框出現。此方法接受一個選用引數，用於指定等待對話框出現的秒數：

```php
$browser->waitForDialog($seconds = null);
```

`assertDialogOpened` 方法可用於斷言對話框已顯示且包含指定的訊息：

```php
$browser->assertDialogOpened('Dialog message');
```

如果 JavaScript 對話框包含輸入框 (prompt)，你可以使用 `typeInDialog` 方法在輸入框中輸入內容：

```php
$browser->typeInDialog('Hello World');
```

若要點擊「確定 (OK)」按鈕來關閉開啟的 JavaScript 對話框，可以呼叫 `acceptDialog` 方法：

```php
$browser->acceptDialog();
```

若要點擊「取消 (Cancel)」按鈕來關閉開啟的 JavaScript 對話框，可以呼叫 `dismissDialog` 方法：

```php
$browser->dismissDialog();
```


<a name="interacting-with-iframes"></a>
### 與內嵌框架 (iframes) 互動

如果你需要與 iframe 內的元素互動，可以使用 `withinFrame` 方法。在提供給 `withinFrame` 方法的閉包內發生的所有元素互動，其範圍都將被限定在指定的 iframe 上下文中：

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

有時你可能希望在將所有操作限定在給定選擇器內的同時執行多個操作。例如，你可能希望斷言某些文字僅存在於表格中，然後點擊該表格內的按鈕。你可以使用 `with` 方法來達成此目的。在提供給 `with` 方法的閉包內執行的所有操作，其範圍都將被限定在原始選擇器中：

```php
$browser->with('.table', function (Browser $table) {
    $table->assertSee('Hello World')
        ->clickLink('Delete');
});
```

你偶爾可能需要在目前範圍之外執行斷言。你可以使用 `elsewhere` 與 `elsewhereWhenAvailable` 方法來達成此目的：

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

在測試大量使用 JavaScript 的應用程式時，通常需要在繼續測試之前「等待」特定的元素或資料就緒。Dusk 讓這件事變得輕而易舉。透過多種方法，您可以等待元素在頁面上變得可見，甚至可以等待直到指定的 JavaScript 運算式評估結果為 `true` 為止。


<a name="waiting"></a>
#### 等待

如果您只是需要讓測試暫停指定的毫秒數，請使用 `pause` 方法：

```php
$browser->pause(1000);
```

如果您只需要在給定條件為 `true` 時才暫停測試，請使用 `pauseIf` 方法：

```php
$browser->pauseIf(App::environment('production'), 1000);
```

同樣地，如果您需要暫停測試直到給定條件為 `true` 為止，可以使用 `pauseUnless` 方法：

```php
$browser->pauseUnless(App::environment('testing'), 1000);
```


<a name="waiting-for-selectors"></a>
#### 等待選擇器

`waitFor` 方法可用於暫停測試的執行，直到與指定的 CSS 或 Dusk 選擇器相符的元素顯示在頁面上。預設情況下，這會在擲出例外之前暫停測試最多五秒。如有需要，您可以將自定義的逾時閾值作為第二個引數傳遞給該方法：

```php
// Wait a maximum of five seconds for the selector...
$browser->waitFor('.selector');

// Wait a maximum of one second for the selector...
$browser->waitFor('.selector', 1);
```

您也可以等待直到與指定選擇器相符的元素包含指定的文字：

```php
// Wait a maximum of five seconds for the selector to contain the given text...
$browser->waitForTextIn('.selector', 'Hello World');

// Wait a maximum of one second for the selector to contain the given text...
$browser->waitForTextIn('.selector', 'Hello World', 1);
```

您也可以等待直到與指定選擇器相符的元素從頁面上消失：

```php
// Wait a maximum of five seconds until the selector is missing...
$browser->waitUntilMissing('.selector');

// Wait a maximum of one second until the selector is missing...
$browser->waitUntilMissing('.selector', 1);
```

或者，您可以等待直到與指定選擇器相符的元素變為啟用 (Enabled) 或停用 (Disabled) 狀態：

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

有時候，您可能希望等待符合指定選擇器的元素出現，然後再與該元素進行互動。例如，您可能希望等待直到互動視窗 (Modal) 出現，然後按下該視窗內的「OK」按鈕。`whenAvailable` 方法可用於達成此目的。在給定閉包內執行的所有元素操作都將被限定在原始選擇器的範圍內：

```php
$browser->whenAvailable('.modal', function (Browser $modal) {
    $modal->assertSee('Hello World')
        ->press('OK');
});
```


<a name="waiting-for-text"></a>
#### 等待文字

`waitForText` 方法可用於等待直到指定的文字顯示在頁面上：

```php
// Wait a maximum of five seconds for the text...
$browser->waitForText('Hello World');

// Wait a maximum of one second for the text...
$browser->waitForText('Hello World', 1);
```

您可以使用 `waitUntilMissingText` 方法來等待直到顯示的文字從頁面上移除：

```php
// Wait a maximum of five seconds for the text to be removed...
$browser->waitUntilMissingText('Hello World');

// Wait a maximum of one second for the text to be removed...
$browser->waitUntilMissingText('Hello World', 1);
```


<a name="waiting-for-links"></a>
#### 等待連結

`waitForLink` 方法可用於等待直到指定的連結文字顯示在頁面上：

```php
// Wait a maximum of five seconds for the link...
$browser->waitForLink('Create');

// Wait a maximum of one second for the link...
$browser->waitForLink('Create', 1);
```


<a name="waiting-for-inputs"></a>
#### 等待輸入欄位

`waitForInput` 方法可用於等待直到指定的輸入欄位在頁面上可見：

```php
// Wait a maximum of five seconds for the input...
$browser->waitForInput($field);

// Wait a maximum of one second for the input...
$browser->waitForInput($field, 1);
```


<a name="waiting-on-the-page-location"></a>
#### 等待頁面位置

當進行路徑斷言（例如 `$browser->assertPathIs('/home')`）時，如果 `window.location.pathname` 是非同步更新的，斷言可能會失敗。您可以使用 `waitForLocation` 方法來等待位置變為指定的值：

```php
$browser->waitForLocation('/secret');
```

`waitForLocation` 方法也可用於等待目前的視窗位置變為一個完整的 URL：

```php
$browser->waitForLocation('https://example.com/path');
```

您也可以等待[具名路由](/docs/{{version}}/routing#named-routes)的位置：

```php
$browser->waitForRoute($routeName, $parameters);
```


<a name="waiting-for-page-reloads"></a>
#### 等待頁面重新載入

如果您需要在執行某個動作後等待頁面重新載入，請使用 `waitForReload` 方法：

```php
use Laravel\Dusk\Browser;

$browser->waitForReload(function (Browser $browser) {
    $browser->press('Submit');
})
->assertSee('Success!');
```

由於等待頁面重新載入的需求通常發生在點擊按鈕之後，為了方便起見，您可以使用 `clickAndWaitForReload` 方法：

```php
$browser->clickAndWaitForReload('.selector')
    ->assertSee('something');
```


<a name="waiting-on-javascript-expressions"></a>
#### 等待 JavaScript 運算式

有時候您可能希望暫停測試的執行，直到指定的 JavaScript 運算式評估結果為 `true`。您可以使用 `waitUntil` 方法輕鬆達成此目的。將運算式傳遞給此方法時，不需要包含 `return` 關鍵字或結尾的分號：

```php
// Wait a maximum of five seconds for the expression to be true...
$browser->waitUntil('App.data.servers.length > 0');

// Wait a maximum of one second for the expression to be true...
$browser->waitUntil('App.data.servers.length > 0', 1);
```


<a name="waiting-on-vue-expressions"></a>
#### 等待 Vue 運算式

`waitUntilVue` 和 `waitUntilVueIsNot` 方法可用於等待直到 [Vue 元件](https://vuejs.org) 屬性具有給定值：

```php
// Wait until the component attribute contains the given value...
$browser->waitUntilVue('user.name', 'Taylor', '@user');

// Wait until the component attribute doesn't contain the given value...
$browser->waitUntilVueIsNot('user.name', null, '@user');
```


<a name="waiting-for-javascript-events"></a>
#### 等待 JavaScript 事件

`waitForEvent` 方法可用於暫停測試的執行，直到某個 JavaScript 事件發生為止：

```php
$browser->waitForEvent('load');
```

事件監聽器會被附加到目前的範圍，預設情況下是 `body` 元素。當使用限定範圍的選擇器時，事件監聽器將會附加到相符的元素上：

```php
$browser->with('iframe', function (Browser $iframe) {
    // Wait for the iframe's load event...
    $iframe->waitForEvent('load');
});
```

您也可以提供一個選擇器作為 `waitForEvent` 方法的第二個引數，以便將事件監聽器附加到特定的元素上：

```php
$browser->waitForEvent('load', '.selector');
```

您也可以等待 `document` 和 `window` 物件上的事件：

```php
// Wait until the document is scrolled...
$browser->waitForEvent('scroll', 'document');

// Wait a maximum of five seconds until the window is resized...
$browser->waitForEvent('resize', 'window', 5);
```


<a name="waiting-with-a-callback"></a>
#### 使用回呼等待

Dusk 中的許多「等待」方法都依賴於底層的 `waitUsing` 方法。您可以直接使用此方法來等待指定的閉包回傳 `true`。`waitUsing` 方法接受等待的最大秒數、評估閉包的間隔時間、閉包本身，以及選用的失敗訊息：

```php
$browser->waitUsing(10, 1, function () use ($something) {
    return $something->isReady();
}, "Something wasn't ready in time.");
```

<a name="scrolling-an-element-into-view"></a>
### 將元素捲動至可見範圍

有時候你可能無法點擊某個元素，因為它位於瀏覽器的可視區域之外。`scrollIntoView` 方法會捲動瀏覽器視窗，直到指定選擇器的元素出現在視圖中：

```php
$browser->scrollIntoView('.selector')
    ->click('.selector');
```

<a name="available-assertions"></a>
## 可用的斷言 (Assertions)

Dusk 提供各種斷言 (Assertions)，供您對應用程式進行驗證。所有可用的斷言都列在下表中：

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

斷言頁面標題是否符合給定的文字：

```php
$browser->assertTitle($title);
```


<a name="assert-title-contains"></a>
#### assertTitleContains

斷言頁面標題是否包含給定的文字：

```php
$browser->assertTitleContains($title);
```


<a name="assert-url-is"></a>
#### assertUrlIs

斷言目前的 URL（不含查詢字串）是否符合給定的字串：

```php
$browser->assertUrlIs($url);
```


<a name="assert-scheme-is"></a>
#### assertSchemeIs

斷言目前的 URL 協定 (Scheme) 是否符合給定的協定：

```php
$browser->assertSchemeIs($scheme);
```


<a name="assert-scheme-is-not"></a>
#### assertSchemeIsNot

斷言目前的 URL 協定是否不符合給定的協定：

```php
$browser->assertSchemeIsNot($scheme);
```


<a name="assert-host-is"></a>
#### assertHostIs

斷言目前的 URL 主機 (Host) 是否符合給定的主機：

```php
$browser->assertHostIs($host);
```


<a name="assert-host-is-not"></a>
#### assertHostIsNot

斷言目前的 URL 主機是否不符合給定的主機：

```php
$browser->assertHostIsNot($host);
```


<a name="assert-port-is"></a>
#### assertPortIs

斷言目前的 URL 連接埠 (Port) 是否符合給定的連接埠：

```php
$browser->assertPortIs($port);
```


<a name="assert-port-is-not"></a>
#### assertPortIsNot

斷言目前的 URL 連接埠是否不符合給定的連接埠：

```php
$browser->assertPortIsNot($port);
```


<a name="assert-path-begins-with"></a>
#### assertPathBeginsWith

斷言目前的 URL 路徑是否以給定的路徑開始：

```php
$browser->assertPathBeginsWith('/home');
```


<a name="assert-path-ends-with"></a>
#### assertPathEndsWith

斷言目前的 URL 路徑是否以給定的路徑結束：

```php
$browser->assertPathEndsWith('/home');
```


<a name="assert-path-contains"></a>
#### assertPathContains

斷言目前的 URL 路徑是否包含給定的路徑：

```php
$browser->assertPathContains('/home');
```


<a name="assert-path-is"></a>
#### assertPathIs

斷言目前的路徑是否符合給定的路徑：

```php
$browser->assertPathIs('/home');
```


<a name="assert-path-is-not"></a>
#### assertPathIsNot

斷言目前的路徑是否不符合給定的路徑：

```php
$browser->assertPathIsNot('/home');
```


<a name="assert-route-is"></a>
#### assertRouteIs

斷言目前的 URL 是否符合給定的[具名路由](/docs/{{version}}/routing#named-routes) URL：

```php
$browser->assertRouteIs($name, $parameters);
```


<a name="assert-query-string-has"></a>
#### assertQueryStringHas

斷言給定的查詢字串參數是否存在：

```php
$browser->assertQueryStringHas($name);
```

斷言給定的查詢字串參數是否存在且符合給定的數值：

```php
$browser->assertQueryStringHas($name, $value);
```


<a name="assert-query-string-missing"></a>
#### assertQueryStringMissing

斷言給定的查詢字串參數是否不存在：

```php
$browser->assertQueryStringMissing($name);
```


<a name="assert-fragment-is"></a>
#### assertFragmentIs

斷言 URL 目前的雜湊片段 (Hash Fragment) 是否符合給定的片段：

```php
$browser->assertFragmentIs('anchor');
```


<a name="assert-fragment-begins-with"></a>
#### assertFragmentBeginsWith

斷言 URL 目前的雜湊片段是否以給定的片段開始：

```php
$browser->assertFragmentBeginsWith('anchor');
```


<a name="assert-fragment-is-not"></a>
#### assertFragmentIsNot

斷言 URL 目前的雜湊片段是否不符合給定的片段：

```php
$browser->assertFragmentIsNot('anchor');
```


<a name="assert-has-cookie"></a>
#### assertHasCookie

斷言給定的加密 Cookie 是否存在：

```php
$browser->assertHasCookie($name);
```


<a name="assert-has-plain-cookie"></a>
#### assertHasPlainCookie

斷言給定的未加密 Cookie 是否存在：

```php
$browser->assertHasPlainCookie($name);
```


<a name="assert-cookie-missing"></a>
#### assertCookieMissing

斷言給定的加密 Cookie 是否不存在：

```php
$browser->assertCookieMissing($name);
```


<a name="assert-plain-cookie-missing"></a>
#### assertPlainCookieMissing

斷言給定的未加密 Cookie 是否不存在：

```php
$browser->assertPlainCookieMissing($name);
```

<a name="assert-cookie-value"></a>
#### assertCookieValue

斷言加密的 Cookie 具有指定的數值：

```php
$browser->assertCookieValue($name, $value);
```


<a name="assert-plain-cookie-value"></a>
#### assertPlainCookieValue

斷言未加密的 Cookie 具有指定的數值：

```php
$browser->assertPlainCookieValue($name, $value);
```


<a name="assert-see"></a>
#### assertSee

斷言指定的文字存在於頁面上：

```php
$browser->assertSee($text);
```


<a name="assert-dont-see"></a>
#### assertDontSee

斷言指定的文字不存在於頁面上：

```php
$browser->assertDontSee($text);
```


<a name="assert-see-in"></a>
#### assertSeeIn

斷言指定的文字存在於選擇器 (Selector) 內：

```php
$browser->assertSeeIn($selector, $text);
```


<a name="assert-dont-see-in"></a>
#### assertDontSeeIn

斷言指定的文字不存在於選擇器內：

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

斷言符合指定選擇器的元素出現的次數：

```php
$browser->assertCount($selector, $count);
```


<a name="assert-script"></a>
#### assertScript

斷言指定的 JavaScript 運算式執行結果為指定的數值：

```php
$browser->assertScript('window.isLoaded')
    ->assertScript('document.readyState', 'complete');
```


<a name="assert-source-has"></a>
#### assertSourceHas

斷言指定的原始碼存在於頁面上：

```php
$browser->assertSourceHas($code);
```


<a name="assert-source-missing"></a>
#### assertSourceMissing

斷言指定的原始碼不存在於頁面上：

```php
$browser->assertSourceMissing($code);
```


<a name="assert-see-link"></a>
#### assertSeeLink

斷言指定的連結存在於頁面上：

```php
$browser->assertSeeLink($linkText);
```


<a name="assert-dont-see-link"></a>
#### assertDontSeeLink

斷言指定的連結不存在於頁面上：

```php
$browser->assertDontSeeLink($linkText);
```


<a name="assert-input-value"></a>
#### assertInputValue

斷言指定的輸入欄位具有指定的數值：

```php
$browser->assertInputValue($field, $value);
```


<a name="assert-input-value-is-not"></a>
#### assertInputValueIsNot

斷言指定的輸入欄位不具備指定的數值：

```php
$browser->assertInputValueIsNot($field, $value);
```


<a name="assert-checked"></a>
#### assertChecked

斷言指定的核取方塊 (Checkbox) 已勾選：

```php
$browser->assertChecked($field);
```


<a name="assert-not-checked"></a>
#### assertNotChecked

斷言指定的核取方塊未勾選：

```php
$browser->assertNotChecked($field);
```


<a name="assert-indeterminate"></a>
#### assertIndeterminate

斷言指定的核取方塊處於不定 (Indeterminate) 狀態：

```php
$browser->assertIndeterminate($field);
```


<a name="assert-radio-selected"></a>
#### assertRadioSelected

斷言指定的單選按鈕 (Radio) 欄位已選取：

```php
$browser->assertRadioSelected($field, $value);
```


<a name="assert-radio-not-selected"></a>
#### assertRadioNotSelected

斷言指定的單選按鈕欄位未選取：

```php
$browser->assertRadioNotSelected($field, $value);
```


<a name="assert-selected"></a>
#### assertSelected

斷言指定的下拉選單已選取指定的數值：

```php
$browser->assertSelected($field, $value);
```


<a name="assert-not-selected"></a>
#### assertNotSelected

斷言指定的下拉選單未選取指定的數值：

```php
$browser->assertNotSelected($field, $value);
```


<a name="assert-select-has-options"></a>
#### assertSelectHasOptions

斷言指定的數值陣列可供選取：

```php
$browser->assertSelectHasOptions($field, $values);
```


<a name="assert-select-missing-options"></a>
#### assertSelectMissingOptions

斷言指定的數值陣列不可供選取：

```php
$browser->assertSelectMissingOptions($field, $values);
```


<a name="assert-select-has-option"></a>
#### assertSelectHasOption

斷言指定的數值在指定欄位中可供選取：

```php
$browser->assertSelectHasOption($field, $value);
```


<a name="assert-select-missing-option"></a>
#### assertSelectMissingOption

斷言指定的數值不可供選取：

```php
$browser->assertSelectMissingOption($field, $value);
```


<a name="assert-value"></a>
#### assertValue

斷言符合指定選擇器的元素具有指定的數值：

```php
$browser->assertValue($selector, $value);
```


<a name="assert-value-is-not"></a>
#### assertValueIsNot

斷言符合指定選擇器的元素不具備指定的數值：

```php
$browser->assertValueIsNot($selector, $value);
```


<a name="assert-attribute"></a>
#### assertAttribute

斷言符合指定選擇器的元素其指定的屬性具有指定的數值：

```php
$browser->assertAttribute($selector, $attribute, $value);
```


<a name="assert-attribute-missing"></a>
#### assertAttributeMissing

斷言符合指定選擇器的元素缺少指定的屬性：

```php
$browser->assertAttributeMissing($selector, $attribute);
```


<a name="assert-attribute-contains"></a>
#### assertAttributeContains

斷言符合指定選擇器的元素其指定的屬性包含指定的數值：

```php
$browser->assertAttributeContains($selector, $attribute, $value);
```


<a name="assert-attribute-doesnt-contain"></a>
#### assertAttributeDoesntContain

斷言符合指定選擇器的元素其指定的屬性不包含指定的數值：

```php
$browser->assertAttributeDoesntContain($selector, $attribute, $value);
```


<a name="assert-aria-attribute"></a>
#### assertAriaAttribute

斷言符合指定選擇器的元素其指定的 aria 屬性具有指定的數值：

```php
$browser->assertAriaAttribute($selector, $attribute, $value);
```

例如，給定標記 `<button aria-label="Add"></button>`，您可以像這樣針對 `aria-label` 屬性進行斷言：

```php
$browser->assertAriaAttribute('button', 'label', 'Add')
```


<a name="assert-data-attribute"></a>
#### assertDataAttribute

斷言符合指定選擇器的元素其指定的 data 屬性具有指定的數值：

```php
$browser->assertDataAttribute($selector, $attribute, $value);
```

例如，給定標記 `<tr id="row-1" data-content="attendees"></tr>`，您可以像這樣針對 `data-content` 屬性進行斷言：

```php
$browser->assertDataAttribute('#row-1', 'content', 'attendees')
```


<a name="assert-visible"></a>
#### assertVisible

斷言符合指定選擇器的元素是可見的：

```php
$browser->assertVisible($selector);
```


<a name="assert-present"></a>
#### assertPresent

斷言符合指定選擇器的元素存在於原始碼中：

```php
$browser->assertPresent($selector);
```


<a name="assert-not-present"></a>
#### assertNotPresent

斷言符合指定選擇器的元素不存在於原始碼中：

```php
$browser->assertNotPresent($selector);
```


<a name="assert-missing"></a>
#### assertMissing

斷言符合指定選擇器的元素是不可見的：

```php
$browser->assertMissing($selector);
```


<a name="assert-input-present"></a>
#### assertInputPresent

斷言具有指定名稱的輸入欄位存在：

```php
$browser->assertInputPresent($name);
```


<a name="assert-input-missing"></a>
#### assertInputMissing

斷言具有指定名稱的輸入欄位不存在於原始碼中：

```php
$browser->assertInputMissing($name);
```

<a name="assert-dialog-opened"></a>
#### assertDialogOpened

斷言包含指定訊息的 JavaScript 對話框已被開啟：

```php
$browser->assertDialogOpened($message);
```


<a name="assert-enabled"></a>
#### assertEnabled

斷言給定的欄位已啟用：

```php
$browser->assertEnabled($field);
```


<a name="assert-disabled"></a>
#### assertDisabled

斷言給定的欄位已停用：

```php
$browser->assertDisabled($field);
```


<a name="assert-button-enabled"></a>
#### assertButtonEnabled

斷言給定的按鈕已啟用：

```php
$browser->assertButtonEnabled($button);
```


<a name="assert-button-disabled"></a>
#### assertButtonDisabled

斷言給定的按鈕已停用：

```php
$browser->assertButtonDisabled($button);
```


<a name="assert-focused"></a>
#### assertFocused

斷言給定的欄位已獲得焦點：

```php
$browser->assertFocused($field);
```


<a name="assert-not-focused"></a>
#### assertNotFocused

斷言給定的欄位未獲得焦點：

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

斷言使用者為訪客 (未通過認證)：

```php
$browser->assertGuest();
```


<a name="assert-authenticated-as"></a>
#### assertAuthenticatedAs

斷言使用者已作為指定的使用者通過認證：

```php
$browser->assertAuthenticatedAs($user);
```


<a name="assert-vue"></a>
#### assertVue

Dusk 甚至允許你對 [Vue component](https://vuejs.org) 資料的狀態進行斷言。例如，假設你的應用程式包含以下 Vue 元件：

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

斷言給定的 Vue 元件資料屬性不符合給定的數值：

```php
$browser->assertVueIsNot($property, $value, $componentSelector = null);
```


<a name="assert-vue-contains"></a>
#### assertVueContains

斷言給定的 Vue 元件資料屬性是一個陣列且包含給定的數值：

```php
$browser->assertVueContains($property, $value, $componentSelector = null);
```


<a name="assert-vue-doesnt-contain"></a>
#### assertVueDoesntContain

斷言給定的 Vue 元件資料屬性是一個陣列且不包含給定的數值：

```php
$browser->assertVueDoesntContain($property, $value, $componentSelector = null);
```

<a name="pages"></a>
## 頁面 (Pages)

有時，測試需要按順序執行多個複雜的操作。這會使你的測試難以閱讀與理解。Dusk Pages 允許你定義具表達性的動作，接著可以透過單一方法在指定頁面上執行。Pages 還允許你為應用程式或單個頁面的常用選擇器定義捷徑。


<a name="generating-pages"></a>
### 產生頁面

若要產生一個頁面物件，請執行 `dusk:page` Artisan 指令。所有的頁面物件都會被放置在應用程式的 `tests/Browser/Pages` 目錄中：

```shell
php artisan dusk:page Login
```


<a name="configuring-pages"></a>
### 設定頁面

預設情況下，頁面有三個方法：`url`、`assert` 與 `elements`。我們現在將討論 `url` 與 `assert` 方法。`elements` 方法將在[下方詳細討論](#shorthand-selectors)。


<a name="the-url-method"></a>
#### `url` 方法

`url` 方法應該回傳代表該頁面的 URL 路徑。Dusk 在瀏覽器中導覽至該頁面時會使用此 url：

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

`assert` 方法可以進行任何必要的斷言 (Assertions)，以驗證瀏覽器是否確實位於該頁面上。實際上並不一定要在此方法內放置任何內容；但如果你願意，可以自由地進行這些斷言。這些斷言在導覽至該頁面時會自動執行：

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

一旦定義了頁面，你就可以使用 `visit` 方法導覽至該頁面：

```php
use Tests\Browser\Pages\Login;

$browser->visit(new Login);
```

有時你可能已經在某個頁面上，需要將該頁面的選擇器與方法「載入」到目前的測試上下文 (Context) 中。這在按下按鈕並被重新導向到指定頁面，而沒有明確導覽至該頁面時很常見。在這種情況下，你可以使用 `on` 方法來載入頁面：

```php
use Tests\Browser\Pages\CreatePlaylist;

$browser->visit('/dashboard')
    ->clickLink('Create Playlist')
    ->on(new CreatePlaylist)
    ->assertSee('@create');
```


<a name="shorthand-selectors"></a>
### 速記選擇器

頁面類別中的 `elements` 方法允許你為頁面上的任何 CSS 選擇器定義快速且易於記憶的捷徑。例如，讓我們為應用程式登入頁面的 "email" 輸入欄位定義一個捷徑：

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

一旦定義了捷徑，你就可以在通常使用完整 CSS 選擇器的任何地方使用該速記選擇器：

```php
$browser->type('@email', 'taylor@laravel.com');
```


<a name="global-shorthand-selectors"></a>
#### 全域速記選擇器

安裝 Dusk 後，一個基礎的 `Page` 類別會被放置在你的 `tests/Browser/Pages` 目錄中。該類別包含一個 `siteElements` 方法，可用於定義整個應用程式中每個頁面都可用的全域速記選擇器：

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

除了頁面上定義的預設方法外，你還可以定義可在整個測試中使用的額外方法。例如，想像我們正在建立一個音樂管理應用程式。應用程式中某個頁面的常見動作可能是建立播放清單。你可以不用在每個測試中重寫建立播放清單的邏輯，而是在頁面類別上定義一個 `createPlaylist` 方法：

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

一旦定義了該方法，你就可以在任何使用該頁面的測試中使用它。瀏覽器實例將會自動作為第一個引數傳遞給自定義的頁面方法：

```php
use Tests\Browser\Pages\Dashboard;

$browser->visit(new Dashboard)
    ->createPlaylist('My Playlist')
    ->assertSee('My Playlist');
```

<a name="components"></a>
## 元件 (Components)

元件與 Dusk 的「頁面物件 (page objects)」類似，但旨在用於整個應用程式中重複使用的 UI 片段與功能，例如導覽列或通知視窗。因此，元件不會綁定到特定的 URL。

<a name="generating-components"></a>
### 產生元件

若要產生一個元件，請執行 `dusk:component` Artisan 指令。新元件會放置在 `tests/Browser/Components` 目錄中：

```shell
php artisan dusk:component DatePicker
```

如上所示，「日期選擇器 (date picker)」是一個可能存在於應用程式中多個不同頁面的元件範例。在整個測試套件的數十個測試中，手動撰寫選擇日期的瀏覽器自動化邏輯會變得非常繁瑣。相反地，我們可以定義一個 Dusk 元件來代表日期選擇器，讓我們將該邏輯封裝在元件內：

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

元件定義完成後，我們就可以在任何測試中輕鬆地在日期選擇器中選擇日期。而且，如果選擇日期的邏輯需要變更，我們只需要更新該元件即可：

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

`component` 方法可用於取得範圍限定在給定元件內的瀏覽器實例：

```php
$datePicker = $browser->component(new DatePickerComponent);

$datePicker->selectDate(2019, 1, 30);

$datePicker->assertSee('January');
```

<a name="continuous-integration"></a>
## 持續整合 (CI)

> [!WARNING]
> 大多數 Dusk 的持續整合設定都預期你的 Laravel 應用程式是使用內建的 PHP 開發伺服器在連接埠 8000 上執行。因此，在繼續之前，你應確保你的持續整合環境中，環境變數 `APP_URL` 的值為 `http://127.0.0.1:8000`。


<a name="running-tests-on-heroku-ci"></a>
### Heroku CI

若要在 [Heroku CI](https://www.heroku.com/continuous-integration) 上執行 Dusk 測試，請將下列 Google Chrome Buildpack 與指令碼新增到你的 Heroku `app.json` 檔案中：

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

若要在 [Travis CI](https://travis-ci.org) 上執行 Dusk 測試，請使用下列 `.travis.yml` 設定。由於 Travis CI 並非圖形化環境，我們需要採取一些額外步驟來啟動 Chrome 瀏覽器。此外，我們將使用 `php artisan serve` 來啟動 PHP 內建的網頁伺服器：

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

如果你使用 [GitHub Actions](https://github.com/features/actions) 來執行你的 Dusk 測試，可以使用下列設定檔作為起點。如同 TravisCI，我們將使用 `php artisan serve` 命令來啟動 PHP 的內建網頁伺服器：

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

如果你使用 [Chipper CI](https://chipperci.com) 來執行你的 Dusk 測試，可以使用下列設定檔作為起點。我們將使用 PHP 的內建伺服器來執行 Laravel，以便接收請求：

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

若要進一步了解如何在 Chipper CI 上執行 Dusk 測試（包含如何使用資料庫），請參閱 [Chipper CI 官方文件](https://chipperci.com/docs/testing/laravel-dusk-new/)。