# Laravel Dusk

- [簡介](#introduction)
- [安裝](#installation)
    - [管理 ChromeDriver 安裝](#managing-chromedriver-installations)
    - [使用其他瀏覽器](#using-other-browsers)
- [開始使用](#getting-started)
    - [產生測試](#generating-tests)
    - [每次測試後重設資料庫](#resetting-the-database-after-each-test)
    - [執行測試](#running-tests)
    - [環境處理](#environment-handling)
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
    - [與內嵌框架互動](#interacting-with-iframes)
    - [限定選擇器範圍](#scoping-selectors)
    - [等待元素](#waiting-for-elements)
    - [將元素捲動至視野中](#scrolling-an-element-into-view)
- [可用的斷言](#available-assertions)
- [頁面](#pages)
    - [產生頁面](#generating-pages)
    - [配置頁面](#configuring-pages)
    - [導覽至頁面](#navigating-to-pages)
    - [簡寫選擇器](#shorthand-selectors)
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

[Laravel Dusk](https://github.com/laravel/dusk) 提供了一個表達性強、易於使用的瀏覽器自動化與測試 API。預設情況下，Dusk 不需要你在本機電腦上安裝 JDK 或 Selenium。相反地，Dusk 使用一個獨立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安裝。不過，你可以自由使用任何其他與 Selenium 相容的驅動程式。


<a name="installation"></a>
## 安裝

首先，你應該安裝 [Google Chrome](https://www.google.com/chrome) 並在你的專案中新增 `laravel/dusk` Composer 依賴套件：

```shell
composer require laravel/dusk --dev
```

> [!WARNING]
> 如果你正在手動註冊 Dusk 的服務提供者 (service provider)，你**絕不**應該在你的正式環境中註冊它，因為這樣做可能會導致任意使用者能夠向你的應用程式進行認證。

安裝 Dusk 套件後，執行 `dusk:install` Artisan 指令。`dusk:install` 指令會建立一個 `tests/Browser` 目錄、一個範例 Dusk 測試，並為你的作業系統安裝 Chrome Driver 二進位檔：

```shell
php artisan dusk:install
```

接著，在你的應用程式的 `.env` 檔案中設定 `APP_URL` 環境變數。此值應與你在瀏覽器中存取應用程式所使用的 URL 相符。

> [!NOTE]
> 如果你正在使用 [Laravel Sail](/docs/{{version}}/sail) 來管理你的本機開發環境，請同時查閱 Sail 文件中關於 [配置與執行 Dusk 測試](/docs/{{version}}/sail#laravel-dusk) 的說明。


<a name="managing-chromedriver-installations"></a>
### 管理 ChromeDriver 安裝

如果你想安裝與 Laravel Dusk 透過 `dusk:install` 指令所安裝的不同版本 ChromeDriver，你可以使用 `dusk:chrome-driver` 指令：

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
> Dusk 要求 `chromedriver` 二進位檔是可執行的。如果你在執行 Dusk 時遇到問題，應使用以下指令確保二進位檔是可執行的：`chmod -R 0755 vendor/laravel/dusk/bin/`。


<a name="using-other-browsers"></a>
### 使用其他瀏覽器

預設情況下，Dusk 使用 Google Chrome 和獨立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安裝來執行你的瀏覽器測試。但是，你可以啟動你自己的 Selenium 伺服器並針對任何你想要的瀏覽器執行測試。

首先，打開你的 `tests/DuskTestCase.php` 檔案，這是你的應用程式的 Dusk 基礎測試案例。在此檔案中，你可以移除對 `startChromeDriver` 方法的呼叫。這將阻止 Dusk 自動啟動 ChromeDriver：

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

接著，你可以修改 `driver` 方法以連接到你選擇的 URL 和埠。此外，你還可以修改應傳遞給 WebDriver 的「所需功能 (desired capabilities)」：

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

要產生一個 Dusk 測試，請使用 `dusk:make` Artisan 指令。產生的測試將會放在 `tests/Browser` 目錄中：

```shell
php artisan dusk:make LoginTest
```


<a name="resetting-the-database-after-each-test"></a>
### 每次測試後重設資料庫

您編寫的大部分測試都會與從應用程式資料庫中擷取資料的頁面互動；然而，您的 Dusk 測試不應使用 `RefreshDatabase` trait。`RefreshDatabase` trait 利用資料庫交易，這將不適用或無法跨 HTTP 請求使用。相反地，您有兩種選擇：`DatabaseMigrations` trait 和 `DatabaseTruncation` trait。


<a name="reset-migrations"></a>
#### 使用 Database Migrations

`DatabaseMigrations` trait 會在每次測試之前執行您的資料庫遷移。然而，對於每次測試都刪除並重新建立您的資料庫資料表通常比截斷資料表慢：

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
> 在執行 Dusk 測試時，不能使用 SQLite 記憶體內資料庫。由於瀏覽器在其自己的程序中執行，它將無法存取其他程序的記憶體內資料庫。


<a name="reset-truncation"></a>
#### 使用 Database Truncation

`DatabaseTruncation` trait 將會在第一次測試時遷移您的資料庫，以確保您的資料庫資料表已正確建立。然而，在後續測試中，資料庫的資料表將會被簡單地截斷，這比重新執行所有資料庫遷移提供了速度提升：

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

預設情況下，此 trait 將截斷所有資料表，除了 `migrations` 資料表。如果您想要自訂應該截斷的資料表，您可以在測試類別上定義一個 `$tablesToTruncate` 屬性：

> [!NOTE]
> 如果您正在使用 Pest，您應該在基礎 `DuskTestCase` 類別或您的測試檔案所擴展的任何類別上定義屬性或方法。

```php
/**
 * Indicates which tables should be truncated.
 *
 * @var array
 */
protected $tablesToTruncate = ['users'];
```

或者，您可以在測試類別上定義一個 `$exceptTables` 屬性，以指定應該從截斷中排除的資料表：

```php
/**
 * Indicates which tables should be excluded from truncation.
 *
 * @var array
 */
protected $exceptTables = ['users'];
```

要指定應該截斷其資料表的資料庫連線，您可以在測試類別上定義一個 `$connectionsToTruncate` 屬性：

```php
/**
 * Indicates which connections should have their tables truncated.
 *
 * @var array
 */
protected $connectionsToTruncate = ['mysql'];
```

如果您想在資料庫截斷執行之前或之後執行程式碼，您可以在測試類別上定義 `beforeTruncatingDatabase` 或 `afterTruncatingDatabase` 方法：

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

要執行您的瀏覽器測試，請執行 `dusk` Artisan 指令：

```shell
php artisan dusk
```

如果您上次執行 `dusk` 指令時發生測試失敗，您可以透過先使用 `dusk:fails` 指令重新執行失敗的測試來節省時間：

```shell
php artisan dusk:fails
```

`dusk` 指令接受任何通常由 Pest / PHPUnit 測試執行器接受的參數，例如允許您只為給定的 [group](https://docs.phpunit.de/en/10.5/annotations.html#group) 執行測試：

```shell
php artisan dusk --group=foo
```

> [!NOTE]
> 如果您正在使用 [Laravel Sail](/docs/{{version}}/sail) 管理您的本地開發環境，請參閱 Sail 文件中關於 [配置和執行 Dusk 測試](/docs/{{version}}/sail#laravel-dusk) 的說明。


<a name="manually-starting-chromedriver"></a>
#### 手動啟動 ChromeDriver

預設情況下，Dusk 將會自動嘗試啟動 ChromeDriver。如果這不適用於您的特定系統，您可以在執行 `dusk` 指令之前手動啟動 ChromeDriver。如果您選擇手動啟動 ChromeDriver，您應該註釋掉您 `tests/DuskTestCase.php` 檔案中的以下行：

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

此外，如果您在除了 9515 以外的連接埠上啟動 ChromeDriver，您應該修改相同類別的 `driver` 方法以反映正確的連接埠：

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

若要強制 Dusk 在執行測試時使用其自己的環境檔案，請在您的專案根目錄中建立一個 `.env.dusk.{environment}` 檔案。例如，如果您將從 `local` 環境中啟動 `dusk` 指令，您應該建立一個 `.env.dusk.local` 檔案。

執行測試時，Dusk 會備份您的 `.env` 檔案，並將您的 Dusk 環境重新命名為 `.env`。一旦測試完成，您的 `.env` 檔案將會被還原。

<a name="browser-basics"></a>
## 瀏覽器基礎

<a name="creating-browsers"></a>
### 建立瀏覽器

首先，我們來編寫一個測試，以驗證我們可以登入我們的應用程式。在產生測試之後，我們可以修改它以導覽到登入頁面，輸入一些憑證，然後點擊「Login」按鈕。若要建立瀏覽器實例，您可以從您的 Dusk 測試中呼叫 `browse` 方法：

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

如您在上面的範例中看到的，`browse` 方法接受一個閉包。Dusk 將自動把瀏覽器實例傳遞給此閉包，它是用於與您的應用程式互動並進行斷言的主要物件。

<a name="creating-multiple-browsers"></a>
#### 建立多個瀏覽器

有時候您可能需要多個瀏覽器才能正確執行測試。例如，測試一個與 websockets 互動的聊天畫面可能需要多個瀏覽器。若要建立多個瀏覽器，只需在 `browse` 方法的閉包簽章中新增更多瀏覽器引數：

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

`visit` 方法可用於導覽到您的應用程式中指定的 URI：

```php
$browser->visit('/login');
```

您可以使用 `visitRoute` 方法導覽到 [具名路由](/docs/{{version}}/routing#named-routes)：

```php
$browser->visitRoute($routeName, $parameters);
```

您可以使用 `back` 和 `forward` 方法「返回」和「前進」：

```php
$browser->back();

$browser->forward();
```

您可以使用 `refresh` 方法重新整理頁面：

```php
$browser->refresh();
```

<a name="resizing-browser-windows"></a>
### 調整瀏覽器視窗大小

您可以使用 `resize` 方法調整瀏覽器視窗的大小：

```php
$browser->resize(1920, 1080);
```

`maximize` 方法可用於最大化瀏覽器視窗：

```php
$browser->maximize();
```

`fitContent` 方法將調整瀏覽器視窗大小以符合其內容大小：

```php
$browser->fitContent();
```

當測試失敗時，Dusk 將在擷取螢幕截圖之前自動調整瀏覽器大小以符合內容。您可以透過在測試中呼叫 `disableFitOnFailure` 方法來停用此功能：

```php
$browser->disableFitOnFailure();
```

您可以使用 `move` 方法將瀏覽器視窗移動到螢幕上的不同位置：

```php
$browser->move($x = 100, $y = 100);
```

<a name="browser-macros"></a>
### 瀏覽器巨集

如果您想定義一個可以在各種測試中重複使用的自訂瀏覽器方法，您可以使用 `Browser` 類別上的 `macro` 方法。通常，您應該從 [服務提供者](/docs/{{version}}/providers) 的 `boot` 方法中呼叫此方法：

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

`macro` 函數接受一個名稱作為其第一個引數，以及一個閉包作為其第二個引數。當作為 `Browser` 實例上的方法呼叫巨集時，巨集的閉包將會被執行：

```php
$this->browse(function (Browser $browser) use ($user) {
    $browser->visit('/pay')
        ->scrollToElement('#credit-card-details')
        ->assertSee('Enter Credit Card Details');
});
```

<a name="authentication"></a>
### 認證

通常，您會測試需要認證的頁面。您可以使用 Dusk 的 `loginAs` 方法來避免在每次測試期間與應用程式的登入畫面互動。`loginAs` 方法接受與您的可認證模型相關聯的主鍵或可認證模型實例：

```php
use App\Models\User;
use Laravel\Dusk\Browser;

$this->browse(function (Browser $browser) {
    $browser->loginAs(User::find(1))
        ->visit('/home');
});
```

> [!WARNING]
> 使用 `loginAs` 方法後，使用者會話將在檔案中的所有測試中維持。

<a name="cookies"></a>
### Cookie

您可以使用 `cookie` 方法來取得或設定加密 Cookie 的值。預設情況下，Laravel 建立的所有 Cookie 都會被加密：

```php
$browser->cookie('name');

$browser->cookie('name', 'Taylor');
```

您可以使用 `plainCookie` 方法來取得或設定未加密 Cookie 的值：

```php
$browser->plainCookie('name');

$browser->plainCookie('name', 'Taylor');
```

您可以使用 `deleteCookie` 方法刪除指定的 Cookie：

```php
$browser->deleteCookie('name');
```

<a name="executing-javascript"></a>
### 執行 JavaScript

您可以使用 `script` 方法在瀏覽器中執行任意 JavaScript 陳述式：

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

您可以使用 `screenshot` 方法擷取螢幕截圖並以指定檔名儲存。所有螢幕截圖將儲存在 `tests/Browser/screenshots` 目錄中：

```php
$browser->screenshot('filename');
```

`responsiveScreenshots` 方法可用於在各種斷點處擷取一系列螢幕截圖：

```php
$browser->responsiveScreenshots('filename');
```

`screenshotElement` 方法可用於擷取頁面上特定元素的螢幕截圖：

```php
$browser->screenshotElement('#selector', 'filename');
```

<a name="storing-console-output-to-disk"></a>
### 將主控台輸出儲存至磁碟

您可以使用 `storeConsoleLog` 方法將當前瀏覽器的主控台輸出以指定檔名寫入磁碟。主控台輸出將儲存在 `tests/Browser/console` 目錄中：

```php
$browser->storeConsoleLog('filename');
```

<a name="storing-page-source-to-disk"></a>
### 將頁面原始碼儲存至磁碟

您可以使用 `storeSource` 方法將當前頁面的原始碼以指定檔名寫入磁碟。頁面原始碼將儲存在 `tests/Browser/source` 目錄中：

```php
$browser->storeSource('filename');
```

<a name="interacting-with-elements"></a>
## 與元素互動


<a name="dusk-selectors"></a>
### Dusk 選擇器

替元素選擇好的 CSS 選擇器是編寫 Dusk 測試中最困難的部分之一。隨著時間推移，前端的變更可能會導致以下 CSS 選擇器破壞您的測試：

```html
// HTML...

<button>Login</button>
```

```php
// Test...

$browser->click('.login-page .container div > button');
```

Dusk 選擇器讓您可以專注於編寫有效的測試，而不是記住 CSS 選擇器。要定義一個選擇器，請在您的 HTML 元素中新增 `dusk` 屬性。然後，當您與 Dusk 瀏覽器互動時，請在選擇器前面加上 `@`，以在測試中操作該附加元素：

```html
// HTML...

<button dusk="login-button">Login</button>
```

```php
// Test...

$browser->click('@login-button');
```

如果需要，您可以透過 `selectorHtmlAttribute` 方法自訂 Dusk 選擇器所使用的 HTML 屬性。通常，這個方法應該在您應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫：

```php
use Laravel\Dusk\Dusk;

Dusk::selectorHtmlAttribute('data-dusk');
```


<a name="text-values-and-attributes"></a>
### 文字、值與屬性


<a name="retrieving-setting-values"></a>
#### 擷取與設定值

Dusk 提供了多種方法來與頁面元素的當前值、顯示文字和屬性進行互動。例如，要取得符合給定 CSS 或 Dusk 選擇器的元素的「值」，請使用 `value` 方法：

```php
// Retrieve the value...
$value = $browser->value('selector');

// Set the value...
$browser->value('selector', 'value');
```

您可以使用 `inputValue` 方法來取得具有給定欄位名稱的輸入元素的「值」：

```php
$value = $browser->inputValue('field');
```


<a name="retrieving-text"></a>
#### 擷取文字

`text` 方法可用於擷取符合給定選擇器之元素的顯示文字：

```php
$text = $browser->text('selector');
```


<a name="retrieving-attributes"></a>
#### 擷取屬性

最後，`attribute` 方法可用於擷取符合給定選擇器之元素的屬性值：

```php
$attribute = $browser->attribute('selector', 'value');
```


<a name="interacting-with-forms"></a>
### 與表單互動


<a name="typing-values"></a>
#### 輸入值

Dusk 提供了多種方法來與表單和輸入元素互動。首先，讓我們看看將文字輸入到輸入欄位中的範例：

```php
$browser->type('email', 'taylor@laravel.com');
```

請注意，儘管該方法在必要時接受一個參數，但我們不需要將 CSS 選擇器傳遞給 `type` 方法。如果未提供 CSS 選擇器，Dusk 將會搜尋具有給定 `name` 屬性的 `input` 或 `textarea` 欄位。

要在不清除內容的情況下向欄位附加文字，您可以使用 `append` 方法：

```php
$browser->type('tags', 'foo')
    ->append('tags', ', bar, baz');
```

您可以使用 `clear` 方法清除輸入的值：

```php
$browser->clear('email');
```

您可以使用 `typeSlowly` 方法指示 Dusk 緩慢輸入。預設情況下，Dusk 會在每次按鍵之間暫停 100 毫秒。要自訂按鍵之間的時間量，您可以將適當的毫秒數作為第三個參數傳遞給該方法：

```php
$browser->typeSlowly('mobile', '+1 (202) 555-5555');

$browser->typeSlowly('mobile', '+1 (202) 555-5555', 300);
```

您可以使用 `appendSlowly` 方法緩慢附加文字：

```php
$browser->type('tags', 'foo')
    ->appendSlowly('tags', ', bar, baz');
```


<a name="dropdowns"></a>
#### 下拉式選單

要選取 `select` 元素上的可用值，您可以使用 `select` 方法。與 `type` 方法一樣，`select` 方法不需要完整的 CSS 選擇器。當將值傳遞給 `select` 方法時，您應該傳遞底層的選項值而不是顯示文字：

```php
$browser->select('size', 'Large');
```

您可以省略第二個參數來選取隨機選項：

```php
$browser->select('size');
```

透過將陣列作為 `select` 方法的第二個參數，您可以指示該方法選取多個選項：

```php
$browser->select('categories', ['Art', 'Music']);
```


<a name="checkboxes"></a>
#### 核取方塊

要「核取」一個核取方塊輸入，您可以使用 `check` 方法。與許多其他輸入相關方法一樣，不需要完整的 CSS 選擇器。如果找不到符合的 CSS 選擇器，Dusk 將會搜尋具有匹配 `name` 屬性的核取方塊。

```php
$browser->check('terms');
```

`uncheck` 方法可用於「取消核取」一個核取方塊輸入：

```php
$browser->uncheck('terms');
```


<a name="radio-buttons"></a>
#### 單選按鈕

要「選取」一個 `radio` 輸入選項，您可以使用 `radio` 方法。與許多其他輸入相關方法一樣，不需要完整的 CSS 選擇器。如果找不到符合的 CSS 選擇器，Dusk 將會搜尋具有匹配 `name` 和 `value` 屬性的 `radio` 輸入。

```php
$browser->radio('size', 'large');
```


<a name="attaching-files"></a>
### 附加檔案

`attach` 方法可用於將檔案附加到 `file` 輸入元素。與許多其他輸入相關方法一樣，不需要完整的 CSS 選擇器。如果找不到符合的 CSS 選擇器，Dusk 將會搜尋具有匹配 `name` 屬性的 `file` 輸入。

```php
$browser->attach('photo', __DIR__.'/photos/mountains.png');
```

> [!WARNING]
> 附加功能要求您的伺服器安裝並啟用 `Zip` PHP 擴充功能。


<a name="pressing-buttons"></a>
### 按下按鈕

`press` 方法可用於點擊頁面上的按鈕元素。傳遞給 `press` 方法的參數可以是按鈕的顯示文字，也可以是 CSS / Dusk 選擇器：

```php
$browser->press('Login');
```

當提交表單時，許多應用程式會在按下表單的提交按鈕後將其停用，然後在表單提交的 HTTP 請求完成時重新啟用該按鈕。要按下按鈕並等待按鈕重新啟用，您可以使用 `pressAndWaitFor` 方法：

```php
// Press the button and wait a maximum of 5 seconds for it to be enabled...
$browser->pressAndWaitFor('Save');

// Press the button and wait a maximum of 1 second for it to be enabled...
$browser->pressAndWaitFor('Save', 1);
```


<a name="clicking-links"></a>
### 點擊連結

要點擊連結，您可以在瀏覽器實例上使用 `clickLink` 方法。`clickLink` 方法將點擊具有給定顯示文字的連結：

```php
$browser->clickLink($linkText);
```

您可以使用 `seeLink` 方法來判斷具有給定顯示文字的連結是否在頁面上可見：

```php
if ($browser->seeLink($linkText)) {
    // ...
}
```

> [!WARNING]
> 這些方法與 jQuery 互動。如果頁面上沒有 jQuery，Dusk 將會自動將其注入頁面，以便在測試期間可用。

<a name="using-the-keyboard"></a>
### 使用鍵盤

`keys` 方法允許您向給定元素提供比 `type` 方法更複雜的輸入序列。例如，您可以指示 Dusk 在輸入值時按住修飾鍵。在這個範例中，當 `taylor` 輸入到與給定選擇器匹配的元素時，將按住 `shift` 鍵。在 `taylor` 輸入後，將不帶任何修飾鍵輸入 `swift`：

```php
$browser->keys('selector', ['{shift}', 'taylor'], 'swift');
```

`keys` 方法的另一個有用的用例是向應用程式的主要 CSS 選擇器發送「鍵盤快速鍵」組合：

```php
$browser->keys('.app', ['{command}', 'j']);
```

> [!NOTE]
> 所有修飾鍵，例如 `{command}`，都用 `{}` 字符包圍，並且與 `Facebook\WebDriver\WebDriverKeys` 類中定義的常數匹配，這些常數可以在 [GitHub](https://github.com/php-webdriver/php-webdriver/blob/master/lib/WebDriverKeys.php) 上找到。

<a name="fluent-keyboard-interactions"></a>
#### 流暢的鍵盤互動

Dusk 還提供了一個 `withKeyboard` 方法，允許您透過 `Laravel\Dusk\Keyboard` 類流暢地執行複雜的鍵盤互動。`Keyboard` 類提供了 `press`、`release`、`type` 和 `pause` 方法：

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

如果您想定義可在整個測試套件中輕鬆重複使用的自訂鍵盤互動，您可以使用 `Keyboard` 類提供的 `macro` 方法。通常，您應該從服務提供者 (service provider) 的 `boot` 方法呼叫此方法：

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

`macro` 函數接受一個名稱作為其第一個參數，以及一個閉包作為其第二個參數。當作為 `Keyboard` 實例上的方法呼叫巨集時，巨集的閉包將被執行：

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

`click` 方法可用於點擊與給定 CSS 或 Dusk 選擇器匹配的元素：

```php
$browser->click('.selector');
```

`clickAtXPath` 方法可用於點擊與給定 XPath 表達式匹配的元素：

```php
$browser->clickAtXPath('//div[@class = "selector"]');
```

`clickAtPoint` 方法可用於點擊瀏覽器可視區域內給定座標對處的最頂層元素：

```php
$browser->clickAtPoint($x = 0, $y = 0);
```

`doubleClick` 方法可用於模擬滑鼠的雙擊：

```php
$browser->doubleClick();

$browser->doubleClick('.selector');
```

`rightClick` 方法可用於模擬滑鼠的右鍵點擊：

```php
$browser->rightClick();

$browser->rightClick('.selector');
```

`clickAndHold` 方法可用於模擬滑鼠按鈕被點擊並按住。隨後呼叫 `releaseMouse` 方法將撤銷此行為並釋放滑鼠按鈕：

```php
$browser->clickAndHold('.selector');

$browser->clickAndHold()
    ->pause(1000)
    ->releaseMouse();
```

`controlClick` 方法可用於模擬瀏覽器中的 `ctrl+click` 事件：

```php
$browser->controlClick();

$browser->controlClick('.selector');
```

<a name="mouseover"></a>
#### 滑鼠懸停

當您需要將滑鼠懸停在與給定 CSS 或 Dusk 選擇器匹配的元素上時，可以使用 `mouseover` 方法：

```php
$browser->mouseover('.selector');
```

<a name="drag-drop"></a>
#### 拖放

`drag` 方法可用於將與給定選擇器匹配的元素拖曳到另一個元素：

```php
$browser->drag('.from-selector', '.to-selector');
```

或者，您可以單向拖曳元素：

```php
$browser->dragLeft('.selector', $pixels = 10);
$browser->dragRight('.selector', $pixels = 10);
$browser->dragUp('.selector', $pixels = 10);
$browser->dragDown('.selector', $pixels = 10);
```

最後，您可以透過給定偏移量拖曳元素：

```php
$browser->dragOffset('.selector', $x = 10, $y = 10);
```

<a name="javascript-dialogs"></a>
### JavaScript 對話方塊

Dusk 提供各種方法來與 JavaScript 對話方塊互動。例如，您可以使用 `waitForDialog` 方法來等待 JavaScript 對話方塊出現。此方法接受一個可選參數，指示等待對話方塊出現的秒數：

```php
$browser->waitForDialog($seconds = null);
```

`assertDialogOpened` 方法可用於斷言對話方塊已顯示並包含給定訊息：

```php
$browser->assertDialogOpened('Dialog message');
```

如果 JavaScript 對話方塊包含提示，您可以使用 `typeInDialog` 方法在提示中輸入值：

```php
$browser->typeInDialog('Hello World');
```

要透過點擊「OK」按鈕關閉開啟的 JavaScript 對話方塊，您可以調用 `acceptDialog` 方法：

```php
$browser->acceptDialog();
```

要透過點擊「Cancel」按鈕關閉開啟的 JavaScript 對話方塊，您可以調用 `dismissDialog` 方法：

```php
$browser->dismissDialog();
```

<a name="interacting-with-iframes"></a>
### 與內嵌框架互動

如果您需要與 iframe 中的元素互動，您可以使用 `withinFrame` 方法。所有在提供給 `withinFrame` 方法的閉包中發生的元素互動，都將被限定在指定 iframe 的上下文範圍內：

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

有時您可能希望在給定選擇器範圍內執行多個操作。例如，您可能希望斷言某些文字僅存在於表格中，然後點擊該表格中的按鈕。您可以使用 `with` 方法來實現此目的。在提供給 `with` 方法的閉包中執行的所有操作都將限定在原始選擇器範圍內：

```php
$browser->with('.table', function (Browser $table) {
    $table->assertSee('Hello World')
        ->clickLink('Delete');
});
```

您可能偶爾需要在當前範圍之外執行斷言。您可以使用 `elsewhere` 和 `elsewhereWhenAvailable` 方法來實現此目的：

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

當測試大量使用 JavaScript 的應用程式時，在繼續進行測試之前，通常有必要「等待」某些元素或資料可用。Dusk 讓這一切變得輕而易舉。透過多種方法，您可以等待元素在頁面上變得可見，甚至可以等待直到給定的 JavaScript 表達式評估為 `true`。


<a name="waiting"></a>
#### 等待

如果您只需要將測試暫停給定的毫秒數，請使用 `pause` 方法：

```php
$browser->pause(1000);
```

如果您只需要在給定條件為 `true` 時暫停測試，請使用 `pauseIf` 方法：

```php
$browser->pauseIf(App::environment('production'), 1000);
```

同樣地，除非給定條件為 `true`，否則您可能需要暫停測試，您可以這麼做，請使用 `pauseUnless` 方法：

```php
$browser->pauseUnless(App::environment('testing'), 1000);
```


<a name="waiting-for-selectors"></a>
#### 等待選擇器

`waitFor` 方法可用來暫停測試執行，直到頁面上顯示與給定 CSS 或 Dusk 選擇器相符的元素為止。預設情況下，這會將測試暫停最多五秒，然後拋出例外。如有必要，您可以將自訂逾時閾值作為方法的第二個引數傳遞：

```php
// Wait a maximum of five seconds for the selector...
$browser->waitFor('.selector');

// Wait a maximum of one second for the selector...
$browser->waitFor('.selector', 1);
```

您也可以等到與給定選擇器相符的元素包含給定文字為止：

```php
// Wait a maximum of five seconds for the selector to contain the given text...
$browser->waitForTextIn('.selector', 'Hello World');

// Wait a maximum of one second for the selector to contain the given text...
$browser->waitForTextIn('.selector', 'Hello World', 1);
```

您也可以等到與給定選擇器相符的元素從頁面中消失為止：

```php
// Wait a maximum of five seconds until the selector is missing...
$browser->waitUntilMissing('.selector');

// Wait a maximum of one second until the selector is missing...
$browser->waitUntilMissing('.selector', 1);
```

或者，您可以等到與給定選擇器相符的元素啟用或停用為止：

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
#### 條件式限定選擇器範圍

有時，您可能希望等待與給定選擇器相符的元素出現，然後與該元素互動。例如，您可能希望等待直到對話框視窗可用，然後在對話框中按下「OK」按鈕。`whenAvailable` 方法可用來達成此目的。在給定閉包中執行的所有元素操作都將限定於原始選擇器：

```php
$browser->whenAvailable('.modal', function (Browser $modal) {
    $modal->assertSee('Hello World')
        ->press('OK');
});
```


<a name="waiting-for-text"></a>
#### 等待文字

`waitForText` 方法可用來等待直到頁面上顯示給定文字：

```php
// Wait a maximum of five seconds for the text...
$browser->waitForText('Hello World');

// Wait a maximum of one second for the text...
$browser->waitForText('Hello World', 1);
```

您可以使用 `waitUntilMissingText` 方法來等待直到顯示的文字已從頁面中移除：

```php
// Wait a maximum of five seconds for the text to be removed...
$browser->waitUntilMissingText('Hello World');

// Wait a maximum of one second for the text to be removed...
$browser->waitUntilMissingText('Hello World', 1);
```


<a name="waiting-for-links"></a>
#### 等待連結

`waitForLink` 方法可用來等待直到頁面上顯示給定連結文字：

```php
// Wait a maximum of five seconds for the link...
$browser->waitForLink('Create');

// Wait a maximum of one second for the link...
$browser->waitForLink('Create', 1);
```


<a name="waiting-for-inputs"></a>
#### 等待輸入

`waitForInput` 方法可用來等待直到給定的輸入欄位在頁面上可見：

```php
// Wait a maximum of five seconds for the input...
$browser->waitForInput($field);

// Wait a maximum of one second for the input...
$browser->waitForInput($field, 1);
```


<a name="waiting-on-the-page-location"></a>
#### 等待頁面位置

當進行像 `$browser->assertPathIs('/home')` 這樣的路徑斷言時，如果 `window.location.pathname` 正在非同步更新，斷言可能會失敗。您可以使用 `waitForLocation` 方法來等待位置成為給定值：

```php
$browser->waitForLocation('/secret');
```

`waitForLocation` 方法也可用來等待目前視窗位置成為完整的 URL：

```php
$browser->waitForLocation('https://example.com/path');
```

您也可以等待 [具名路由](/docs/{{version}}/routing#named-routes) 的位置：

```php
$browser->waitForRoute($routeName, $parameters);
```


<a name="waiting-for-page-reloads"></a>
#### 等待頁面重新載入

如果您需要在執行動作後等待頁面重新載入，請使用 `waitForReload` 方法：

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
#### 等待 JavaScript 表達式

有時您可能希望暫停測試的執行，直到給定的 JavaScript 表達式評估為 `true`。您可以使用 `waitUntil` 方法輕鬆達成此目的。當向此方法傳遞表達式時，您不需要包含 `return` 關鍵字或結尾分號：

```php
// Wait a maximum of five seconds for the expression to be true...
$browser->waitUntil('App.data.servers.length > 0');

// Wait a maximum of one second for the expression to be true...
$browser->waitUntil('App.data.servers.length > 0', 1);
```


<a name="waiting-on-vue-expressions"></a>
#### 等待 Vue 表達式

`waitUntilVue` 和 `waitUntilVueIsNot` 方法可用來等待直到 [Vue component](https://vuejs.org) 屬性具有給定值：

```php
// Wait until the component attribute contains the given value...
$browser->waitUntilVue('user.name', 'Taylor', '@user');

// Wait until the component attribute doesn't contain the given value...
$browser->waitUntilVueIsNot('user.name', null, '@user');
```


<a name="waiting-for-javascript-events"></a>
#### 等待 JavaScript 事件

`waitForEvent` 方法可用來暫停測試執行，直到發生 JavaScript 事件為止：

```php
$browser->waitForEvent('load');
```

事件監聽器附加到目前範圍，預設為 `body` 元素。當使用限定範圍的選擇器時，事件監聽器將附加到相符的元素：

```php
$browser->with('iframe', function (Browser $iframe) {
    // Wait for the iframe's load event...
    $iframe->waitForEvent('load');
});
```

您也可以提供一個選擇器作為 `waitForEvent` 方法的第二個引數，將事件監聽器附加到特定的元素：

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

Dusk 中的許多「等待」方法都依賴於底層的 `waitUsing` 方法。您可以直接使用此方法來等待給定的閉包回傳 `true`。`waitUsing` 方法接受最大等待秒數、評估閉包的間隔、閉包，以及一個可選的失敗訊息：

```php
$browser->waitUsing(10, 1, function () use ($something) {
    return $something->isReady();
}, "Something wasn't ready in time.");
```

<a name="scrolling-an-element-into-view"></a>
### 將元素捲動至視野中

有時您可能無法點擊某個元素，因為它位於瀏覽器的可視區域之外。`scrollIntoView` 方法將會捲動瀏覽器視窗，直到指定選擇器的元素進入視野中：

```php
$browser->scrollIntoView('.selector')
    ->click('.selector');
```

<a name="available-assertions"></a>
## 可用的斷言

Dusk 提供了多種可對您的應用程式進行的斷言。所有可用的斷言都列於下方清單中：

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

斷言頁面標題與給定文字相符：

```php
$browser->assertTitle($title);
```


<a name="assert-title-contains"></a>
#### assertTitleContains

斷言頁面標題包含給定文字：

```php
$browser->assertTitleContains($title);
```


<a name="assert-url-is"></a>
#### assertUrlIs

斷言當前 URL（不包含查詢字串）與給定字串相符：

```php
$browser->assertUrlIs($url);
```


<a name="assert-scheme-is"></a>
#### assertSchemeIs

斷言當前 URL 協議與給定協議相符：

```php
$browser->assertSchemeIs($scheme);
```


<a name="assert-scheme-is-not"></a>
#### assertSchemeIsNot

斷言當前 URL 協議與給定協議不符：

```php
$browser->assertSchemeIsNot($scheme);
```


<a name="assert-host-is"></a>
#### assertHostIs

斷言當前 URL 主機與給定主機相符：

```php
$browser->assertHostIs($host);
```


<a name="assert-host-is-not"></a>
#### assertHostIsNot

斷言當前 URL 主機與給定主機不符：

```php
$browser->assertHostIsNot($host);
```


<a name="assert-port-is"></a>
#### assertPortIs

斷言當前 URL 埠號與給定埠號相符：

```php
$browser->assertPortIs($port);
```


<a name="assert-port-is-not"></a>
#### assertPortIsNot

斷言當前 URL 埠號與給定埠號不符：

```php
$browser->assertPortIsNot($port);
```


<a name="assert-path-begins-with"></a>
#### assertPathBeginsWith

斷言當前 URL 路徑以給定路徑開頭：

```php
$browser->assertPathBeginsWith('/home');
```


<a name="assert-path-ends-with"></a>
#### assertPathEndsWith

斷言當前 URL 路徑以給定路徑結尾：

```php
$browser->assertPathEndsWith('/home');
```


<a name="assert-path-contains"></a>
#### assertPathContains

斷言當前 URL 路徑包含給定路徑：

```php
$browser->assertPathContains('/home');
```


<a name="assert-path-is"></a>
#### assertPathIs

斷言當前路徑與給定路徑相符：

```php
$browser->assertPathIs('/home');
```


<a name="assert-path-is-not"></a>
#### assertPathIsNot

斷言當前路徑與給定路徑不符：

```php
$browser->assertPathIsNot('/home');
```


<a name="assert-route-is"></a>
#### assertRouteIs

斷言當前 URL 與給定 [具名路由](/docs/{{version}}/routing#named-routes) 的 URL 相符：

```php
$browser->assertRouteIs($name, $parameters);
```


<a name="assert-query-string-has"></a>
#### assertQueryStringHas

斷言給定的查詢字串參數存在：

```php
$browser->assertQueryStringHas($name);
```

斷言給定的查詢字串參數存在並具有給定值：

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

斷言 URL 的當前雜湊片段與給定片段相符：

```php
$browser->assertFragmentIs('anchor');
```


<a name="assert-fragment-begins-with"></a>
#### assertFragmentBeginsWith

斷言 URL 的當前雜湊片段以給定片段開頭：

```php
$browser->assertFragmentBeginsWith('anchor');
```


<a name="assert-fragment-is-not"></a>
#### assertFragmentIsNot

斷言 URL 的當前雜湊片段與給定片段不符：

```php
$browser->assertFragmentIsNot('anchor');
```


<a name="assert-has-cookie"></a>
#### assertHasCookie

斷言給定的加密 Cookie 存在：

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

斷言給定的加密 Cookie 不存在：

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

斷言加密的 Cookie 具有指定的值：

```php
$browser->assertCookieValue($name, $value);
```


<a name="assert-plain-cookie-value"></a>
#### assertPlainCookieValue

斷言未加密的 Cookie 具有指定的值：

```php
$browser->assertPlainCookieValue($name, $value);
```


<a name="assert-see"></a>
#### assertSee

斷言頁面上存在指定的文字：

```php
$browser->assertSee($text);
```


<a name="assert-dont-see"></a>
#### assertDontSee

斷言頁面上不存在指定的文字：

```php
$browser->assertDontSee($text);
```


<a name="assert-see-in"></a>
#### assertSeeIn

斷言在選擇器內存在指定的文字：

```php
$browser->assertSeeIn($selector, $text);
```


<a name="assert-dont-see-in"></a>
#### assertDontSeeIn

斷言在選擇器內不存在指定的文字：

```php
$browser->assertDontSeeIn($selector, $text);
```


<a name="assert-see-anything-in"></a>
#### assertSeeAnythingIn

斷言在選擇器內存在任何文字：

```php
$browser->assertSeeAnythingIn($selector);
```


<a name="assert-see-nothing-in"></a>
#### assertSeeNothingIn

斷言在選擇器內不存在任何文字：

```php
$browser->assertSeeNothingIn($selector);
```


<a name="assert-count"></a>
#### assertCount

斷言符合指定選擇器的元素出現指定次數：

```php
$browser->assertCount($selector, $count);
```


<a name="assert-script"></a>
#### assertScript

斷言指定的 JavaScript 表達式評估為指定的值：

```php
$browser->assertScript('window.isLoaded')
    ->assertScript('document.readyState', 'complete');
```


<a name="assert-source-has"></a>
#### assertSourceHas

斷言頁面上存在指定的原始碼：

```php
$browser->assertSourceHas($code);
```


<a name="assert-source-missing"></a>
#### assertSourceMissing

斷言頁面上不存在指定的原始碼：

```php
$browser->assertSourceMissing($code);
```


<a name="assert-see-link"></a>
#### assertSeeLink

斷言頁面上存在指定的連結：

```php
$browser->assertSeeLink($linkText);
```


<a name="assert-dont-see-link"></a>
#### assertDontSeeLink

斷言頁面上不存在指定的連結：

```php
$browser->assertDontSeeLink($linkText);
```


<a name="assert-input-value"></a>
#### assertInputValue

斷言指定的輸入欄位具有指定的值：

```php
$browser->assertInputValue($field, $value);
```


<a name="assert-input-value-is-not"></a>
#### assertInputValueIsNot

斷言指定的輸入欄位不具有指定的值：

```php
$browser->assertInputValueIsNot($field, $value);
```


<a name="assert-checked"></a>
#### assertChecked

斷言指定的核取方塊已選取：

```php
$browser->assertChecked($field);
```


<a name="assert-not-checked"></a>
#### assertNotChecked

斷言指定的核取方塊未選取：

```php
$browser->assertNotChecked($field);
```


<a name="assert-indeterminate"></a>
#### assertIndeterminate

斷言指定的核取方塊處於不確定狀態：

```php
$browser->assertIndeterminate($field);
```


<a name="assert-radio-selected"></a>
#### assertRadioSelected

斷言指定的 Radio 選項已選取：

```php
$browser->assertRadioSelected($field, $value);
```


<a name="assert-radio-not-selected"></a>
#### assertRadioNotSelected

斷言指定的 Radio 選項未選取：

```php
$browser->assertRadioNotSelected($field, $value);
```


<a name="assert-selected"></a>
#### assertSelected

斷言指定的下拉式選單已選取指定的值：

```php
$browser->assertSelected($field, $value);
```


<a name="assert-not-selected"></a>
#### assertNotSelected

斷言指定的下拉式選單未選取指定的值：

```php
$browser->assertNotSelected($field, $value);
```


<a name="assert-select-has-options"></a>
#### assertSelectHasOptions

斷言指定的值陣列可供選取：

```php
$browser->assertSelectHasOptions($field, $values);
```


<a name="assert-select-missing-options"></a>
#### assertSelectMissingOptions

斷言指定的值陣列不可供選取：

```php
$browser->assertSelectMissingOptions($field, $values);
```


<a name="assert-select-has-option"></a>
#### assertSelectHasOption

斷言指定的值可供指定欄位選取：

```php
$browser->assertSelectHasOption($field, $value);
```


<a name="assert-select-missing-option"></a>
#### assertSelectMissingOption

斷言指定的值不可供選取：

```php
$browser->assertSelectMissingOption($field, $value);
```


<a name="assert-value"></a>
#### assertValue

斷言符合指定選擇器的元素具有指定的值：

```php
$browser->assertValue($selector, $value);
```


<a name="assert-value-is-not"></a>
#### assertValueIsNot

斷言符合指定選擇器的元素不具有指定的值：

```php
$browser->assertValueIsNot($selector, $value);
```


<a name="assert-attribute"></a>
#### assertAttribute

斷言符合指定選擇器的元素在提供的屬性中具有指定的值：

```php
$browser->assertAttribute($selector, $attribute, $value);
```


<a name="assert-attribute-missing"></a>
#### assertAttributeMissing

斷言符合指定選擇器的元素缺少提供的屬性：

```php
$browser->assertAttributeMissing($selector, $attribute);
```


<a name="assert-attribute-contains"></a>
#### assertAttributeContains

斷言符合指定選擇器的元素在提供的屬性中包含指定的值：

```php
$browser->assertAttributeContains($selector, $attribute, $value);
```


<a name="assert-attribute-doesnt-contain"></a>
#### assertAttributeDoesntContain

斷言符合指定選擇器的元素在提供的屬性中不包含指定的值：

```php
$browser->assertAttributeDoesntContain($selector, $attribute, $value);
```


<a name="assert-aria-attribute"></a>
#### assertAriaAttribute

斷言符合指定選擇器的元素在提供的 aria 屬性中具有指定的值：

```php
$browser->assertAriaAttribute($selector, $attribute, $value);
```

例如，給定標記 `<button aria-label="Add"></button>`，您可以像這樣斷言 `aria-label` 屬性：

```php
$browser->assertAriaAttribute('button', 'label', 'Add')
```


<a name="assert-data-attribute"></a>
#### assertDataAttribute

斷言符合指定選擇器的元素在提供的 data 屬性中具有指定的值：

```php
$browser->assertDataAttribute($selector, $attribute, $value);
```

例如，給定標記 `<tr id="row-1" data-content="attendees"></tr>`，您可以像這樣斷言 `data-label` 屬性：

```php
$browser->assertDataAttribute('#row-1', 'content', 'attendees')
```


<a name="assert-visible"></a>
#### assertVisible

斷言符合指定選擇器的元素可見：

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

斷言符合指定選擇器的元素不可見：

```php
$browser->assertMissing($selector);
```


<a name="assert-input-present"></a>
#### assertInputPresent

斷言存在具有指定名稱的輸入欄位：

```php
$browser->assertInputPresent($name);
```


<a name="assert-input-missing"></a>
#### assertInputMissing

斷言在原始碼中不存在具有指定名稱的輸入欄位：

```php
$browser->assertInputMissing($name);
```

<a name="assert-dialog-opened"></a>
#### assertDialogOpened

斷言一個包含給定訊息的 JavaScript 對話方塊已開啟：

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

斷言給定的欄位已禁用：

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

斷言給定的按鈕已禁用：

```php
$browser->assertButtonDisabled($button);
```


<a name="assert-focused"></a>
#### assertFocused

斷言給定的欄位已聚焦：

```php
$browser->assertFocused($field);
```


<a name="assert-not-focused"></a>
#### assertNotFocused

斷言給定的欄位未聚焦：

```php
$browser->assertNotFocused($field);
```


<a name="assert-authenticated"></a>
#### assertAuthenticated

斷言使用者已認證：

```php
$browser->assertAuthenticated();
```


<a name="assert-guest"></a>
#### assertGuest

斷言使用者未認證：

```php
$browser->assertGuest();
```


<a name="assert-authenticated-as"></a>
#### assertAuthenticatedAs

斷言使用者已以給定使用者身分進行認證：

```php
$browser->assertAuthenticatedAs($user);
```


<a name="assert-vue"></a>
#### assertVue

Dusk 甚至允許您對 [Vue component](https://vuejs.org) 資料的狀態進行斷言。舉例來說，假設您的應用程式包含以下 Vue 元件：

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

您可以像這樣對 Vue 元件的狀態進行斷言：

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

斷言給定的 Vue 元件資料屬性不符合給定值：

```php
$browser->assertVueIsNot($property, $value, $componentSelector = null);
```


<a name="assert-vue-contains"></a>
#### assertVueContains

斷言給定的 Vue 元件資料屬性是陣列且包含給定值：

```php
$browser->assertVueContains($property, $value, $componentSelector = null);
```


<a name="assert-vue-doesnt-contain"></a>
#### assertVueDoesntContain

斷言給定的 Vue 元件資料屬性是陣列且不包含給定值：

```php
$browser->assertVueDoesntContain($property, $value, $componentSelector = null);
```

<a name="pages"></a>
## 頁面

有時，測試需要依序執行數個複雜的動作。這可能會使您的測試更難閱讀和理解。Dusk 頁面讓您可以定義富有表達力的動作，然後可以透過單一方法在給定頁面上執行這些動作。頁面也允許您為應用程式或單一頁面定義常用選擇器的快捷方式。

<a name="generating-pages"></a>
### 產生頁面

若要產生頁面物件，請執行 `dusk:page` Artisan 指令。所有頁面物件都將放置在應用程式的 `tests/Browser/Pages` 目錄中：

```shell
php artisan dusk:page Login
```

<a name="configuring-pages"></a>
### 配置頁面

預設情況下，頁面有三個方法：`url`、`assert` 和 `elements`。我們現在將討論 `url` 和 `assert` 方法。`elements` 方法將在 [下方詳述](#shorthand-selectors)。

<a name="the-url-method"></a>
#### `url` 方法

`url` 方法應傳回代表該頁面的 URL 路徑。Dusk 將在瀏覽器中導覽至該頁面時使用此 URL：

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

`assert` 方法可以執行任何必要的斷言，以驗證瀏覽器確實位於給定頁面上。實際上不必在此方法中放置任何內容；但是，如果您願意，可以自由地執行這些斷言。這些斷言將在導覽至頁面時自動執行：

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

定義頁面後，您可以使用 `visit` 方法導覽至該頁面：

```php
use Tests\Browser\Pages\Login;

$browser->visit(new Login);
```

有時您可能已經在給定頁面上，需要將該頁面的選擇器和方法「載入」到目前的測試情境中。當按下按鈕並被重新導向到某個頁面而沒有明確導覽到該頁面時，這種情況很常見。在這種情況下，您可以使用 `on` 方法載入該頁面：

```php
use Tests\Browser\Pages\CreatePlaylist;

$browser->visit('/dashboard')
    ->clickLink('Create Playlist')
    ->on(new CreatePlaylist)
    ->assertSee('@create');
```

<a name="shorthand-selectors"></a>
### 簡寫選擇器

頁面類別中的 `elements` 方法允許您為頁面上的任何 CSS 選擇器定義快速、易於記憶的快捷方式。例如，讓我們為應用程式登入頁面的「email」輸入欄位定義一個快捷方式：

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

定義快捷方式後，您可以在任何通常使用完整 CSS 選擇器的地方使用簡寫選擇器：

```php
$browser->type('@email', 'taylor@laravel.com');
```

<a name="global-shorthand-selectors"></a>
#### 全域簡寫選擇器

安裝 Dusk 後，一個基礎 `Page` 類別將會放置在您的 `tests/Browser/Pages` 目錄中。此類別包含一個 `siteElements` 方法，可用於定義應該在應用程式中所有頁面都可用的全域簡寫選擇器：

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

除了頁面上定義的預設方法外，您還可以定義可在測試中使用的其他方法。例如，假設我們正在建置一個音樂管理應用程式。應用程式某個頁面的一個常見動作可能是建立一個播放清單。與其在每個測試中重複編寫建立播放清單的邏輯，您可以在頁面類別上定義一個 `createPlaylist` 方法：

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

定義方法後，您可以在任何使用該頁面的測試中使用它。瀏覽器實例將自動作為第一個參數傳遞給自訂頁面方法：

```php
use Tests\Browser\Pages\Dashboard;

$browser->visit(new Dashboard)
    ->createPlaylist('My Playlist')
    ->assertSee('My Playlist');
```

<a name="components"></a>
## 元件

元件與 Dusk 的「頁面物件」類似，但它專用於應用程式中重複使用的 UI 片段和功能，例如導覽列或通知視窗。因此，元件不綁定到特定的 URL。

<a name="generating-components"></a>
### 產生元件

若要產生元件，請執行 `dusk:component` Artisan 命令。新的元件會放置在 `tests/Browser/Components` 目錄中：

```shell
php artisan dusk:component DatePicker
```

如上所示，「日期選擇器」就是一個可能存在於應用程式中多個頁面上的元件範例。若要在整個測試套件中的數十個測試中手動編寫瀏覽器自動化邏輯來選擇日期會很麻煩。因此，我們可以定義一個 Dusk 元件來代表日期選擇器，讓我們能夠將該邏輯封裝在元件中：

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

一旦元件被定義，我們就可以在任何測試中輕鬆地從日期選擇器中選擇日期。而且，如果選擇日期所需的邏輯發生變化，我們只需要更新元件即可：

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

`component` 方法可用於擷取限定於給定元件的瀏覽器實例：

```php
$datePicker = $browser->component(new DatePickerComponent);

$datePicker->selectDate(2019, 1, 30);

$datePicker->assertSee('January');
```

<a name="continuous-integration"></a>
## 持續整合

> [!WARNING]
> 大多數 Dusk 持續整合配置預期您的 Laravel 應用程式會透過內建的 PHP 開發伺服器在 8000 埠提供服務。因此，在繼續之前，您應該確保您的持續整合環境具有 `APP_URL` 環境變數值 `http://127.0.0.1:8000`。

<a name="running-tests-on-heroku-ci"></a>
### Heroku CI

要在 [Heroku CI](https://www.heroku.com/continuous-integration) 上執行 Dusk 測試，請將以下 Google Chrome buildpack 和指令碼新增至您的 Heroku `app.json` 檔案中：

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

要在 [Travis CI](https://travis-ci.org) 上執行您的 Dusk 測試，請使用以下 `.travis.yml` 配置。由於 Travis CI 不是一個圖形化環境，我們需要採取一些額外步驟來啟動 Chrome 瀏覽器。此外，我們將使用 `php artisan serve` 來啟動 PHP 的內建網頁伺服器：

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

如果您正在使用 [GitHub Actions](https://github.com/features/actions) 來執行您的 Dusk 測試，您可以使用以下配置檔案作為起點。與 TravisCI 類似，我們將使用 `php artisan serve` 指令來啟動 PHP 的內建網頁伺服器：

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

如果您正在使用 [Chipper CI](https://chipperci.com) 來執行您的 Dusk 測試，您可以使用以下配置檔案作為起點。我們將使用 PHP 的內建伺服器來執行 Laravel，以便我們可以監聽請求：

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

要了解更多關於在 Chipper CI 上執行 Dusk 測試的資訊，包括如何使用資料庫，請查閱 [Chipper CI 官方文件](https://chipperci.com/docs/testing/laravel-dusk-new/)。