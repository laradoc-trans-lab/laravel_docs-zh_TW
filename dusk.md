# Laravel Dusk

- [介紹](#introduction)
- [安裝](#installation)
    - [管理 ChromeDriver 安裝](#managing-chromedriver-installations)
    - [使用其他瀏覽器](#using-other-browsers)
- [開始使用](#getting-started)
    - [產生測試](#generating-tests)
    - [每次測試後重置資料庫](#resetting-the-database-after-each-test)
    - [執行測試](#running-tests)
    - [環境處理](#environment-handling)
- [瀏覽器基礎](#browser-basics)
    - [建立瀏覽器](#creating-browsers)
    - [導覽](#navigation)
    - [調整瀏覽器視窗大小](#resizing-browser-windows)
    - [瀏覽器巨集](#browser-macros)
    - [身份驗證](#authentication)
    - [Cookies](#cookies)
    - [執行 JavaScript](#executing-javascript)
    - [擷取螢幕截圖](#taking-a-screenshot)
    - [將主控台輸出儲存到磁碟](#storing-console-output-to-disk)
    - [將頁面原始碼儲存到磁碟](#storing-page-source-to-disk)
- [與元素互動](#interacting-with-elements)
    - [Dusk 選擇器](#dusk-selectors)
    - [文字、值與屬性](#text-values-and-attributes)
    - [與表單互動](#interacting-with-forms)
    - [附加檔案](#attaching-files)
    - [按下按鈕](#pressing-buttons)
    - [點擊連結](#clicking-links)
    - [使用鍵盤](#using-the-keyboard)
    - [使用滑鼠](#using-the-mouse)
    - [JavaScript 對話框](#javascript-dialogs)
    - [與內嵌框架互動](#interacting-with-iframes)
    - [範圍化選擇器](#scoping-selectors)
    - [等待元素](#waiting-for-elements)
    - [將元素捲動到檢視區](#scrolling-an-element-into-view)
- [可用的斷言](#available-assertions)
- [頁面](#pages)
    - [產生頁面](#generating-pages)
    - [配置頁面](#configuring-pages)
    - [導覽至頁面](#navigating-to-pages)
    - [縮寫選擇器](#shorthand-selectors)
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
## 介紹

[Laravel Dusk](https://github.com/laravel/dusk) 提供了一個表達力強、易於使用的瀏覽器自動化與測試 API。預設情況下，Dusk 不要求您在您的本機電腦上安裝 JDK 或 Selenium。取而代之的是，Dusk 使用了一個獨立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安裝。但是，您可以自由使用任何其他與 Selenium 相容的驅動程式。

<a name="installation"></a>
## 安裝

要開始使用，您應該安裝 [Google Chrome](https://www.google.com/chrome) 並將 `laravel/dusk` Composer 依賴套件新增到您的專案中：

```shell
composer require laravel/dusk --dev
```

> [!WARNING]  
> 如果您手動註冊 Dusk 的服務提供者，您**絕不**應該在正式環境中註冊它，因為這樣做可能導致任意使用者能夠對您的應用程式進行身份驗證。

安裝 Dusk 套件後，執行 `dusk:install` Artisan 指令。`dusk:install` 指令將會建立 `tests/Browser` 目錄、一個範例 Dusk 測試，並為您的作業系統安裝 Chrome Driver 二進位檔：

```shell
php artisan dusk:install
```

接下來，設定您的應用程式 `.env` 檔案中的 `APP_URL` 環境變數。這個值應該與您用於在瀏覽器中存取應用程式的 URL 相符。

> [!NOTE]  
> 如果您正在使用 [Laravel Sail](/docs/{{version}}/sail) 來管理您的本機開發環境，請也參閱 Sail 說明文件關於 [設定和執行 Dusk 測試](/docs/{{version}}/sail#laravel-dusk) 的部分。

<a name="managing-chromedriver-installations"></a>
### 管理 ChromeDriver 安裝

如果您想安裝不同版本的 ChromeDriver，而非 Laravel Dusk 透過 `dusk:install` 指令所安裝的，您可以使用 `dusk:chrome-driver` 指令：

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
> Dusk 需要 `chromedriver` 二進位檔是可執行的。如果您在執行 Dusk 時遇到問題，您應該使用以下指令確保二進位檔是可執行的：`chmod -R 0755 vendor/laravel/dusk/bin/`。

<a name="using-other-browsers"></a>
### 使用其他瀏覽器

預設情況下，Dusk 使用 Google Chrome 以及獨立的 [ChromeDriver](https://sites.google.com/chromium.org/driver) 安裝來執行您的瀏覽器測試。但是，您可以啟動您自己的 Selenium 伺服器，並針對您希望的任何瀏覽器執行測試。

要開始使用，請開啟您的 `tests/DuskTestCase.php` 檔案，該檔案是您應用程式的基礎 Dusk 測試案例。在這個檔案中，您可以移除對 `startChromeDriver` 方法的呼叫。這將會阻止 Dusk 自動啟動 ChromeDriver：

    /**
     * Prepare for Dusk test execution.
     *
     * @beforeClass
     */
    public static function prepare(): void
    {
        // static::startChromeDriver();
    }

接下來，您可以修改 `driver` 方法以連接到您選擇的 URL 與連接埠。此外，您可以修改應傳遞給 WebDriver 的「desired capabilities (期望功能)」設定：

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

<a name="getting-started"></a>
## 開始使用


<a name="generating-tests"></a>
### 產生測試

要產生 Dusk 測試，請使用 `dusk:make` Artisan 命令。產生的測試將會放置在 `tests/Browser` 目錄中：

```shell
php artisan dusk:make LoginTest
```


<a name="resetting-the-database-after-each-test"></a>
### 每次測試後重置資料庫

您所編寫的大部分測試都會與從應用程式資料庫中檢索資料的頁面互動；然而，您的 Dusk 測試絕不應使用 `RefreshDatabase` trait。`RefreshDatabase` trait 利用了資料庫事務，這些事務將不適用或無法跨 HTTP 請求使用。相反地，您有兩種選擇：`DatabaseMigrations` trait 和 `DatabaseTruncation` trait。


<a name="reset-migrations"></a>
#### 使用資料庫遷移

`DatabaseMigrations` trait 將會在每次測試前執行您的資料庫遷移。然而，在每次測試前刪除並重新建立資料庫資料表通常比截斷資料表慢：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;

uses(DatabaseMigrations::class);

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
> 執行 Dusk 測試時，不能使用 SQLite 記憶體中資料庫。由於瀏覽器在自己的行程中執行，因此它將無法存取其他行程的記憶體中資料庫。


<a name="reset-truncation"></a>
#### 使用資料庫截斷

`DatabaseTruncation` trait 將會在第一次測試時遷移您的資料庫，以確保您的資料庫資料表已正確建立。然而，在後續測試中，資料庫的資料表將會被簡單地截斷——這比重新執行所有資料庫遷移提供了速度提升：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\DatabaseTruncation;
use Laravel\Dusk\Browser;

uses(DatabaseTruncation::class);

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

預設情況下，此 trait 將會截斷除了 `migrations` 資料表之外的所有資料表。如果您想自訂應被截斷的資料表，您可以在測試類別上定義一個 `$tablesToTruncate` 屬性：

> [!NOTE]  
> 如果您正在使用 Pest，您應該在基礎 `DuskTestCase` 類別或您的測試檔案所繼承的任何類別上定義屬性或方法。

    /**
     * Indicates which tables should be truncated.
     *
     * @var array
     */
    protected $tablesToTruncate = ['users'];

或者，您可以在測試類別上定義一個 `$exceptTables` 屬性，以指定應從截斷中排除的資料表：

    /**
     * Indicates which tables should be excluded from truncation.
     *
     * @var array
     */
    protected $exceptTables = ['users'];

要指定應截斷其資料表的資料庫連線，您可以在測試類別上定義一個 `$connectionsToTruncate` 屬性：

    /**
     * Indicates which connections should have their tables truncated.
     *
     * @var array
     */
    protected $connectionsToTruncate = ['mysql'];

如果您想在執行資料庫截斷之前或之後執行程式碼，您可以在測試類別上定義 `beforeTruncatingDatabase` 或 `afterTruncatingDatabase` 方法：

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


<a name="running-tests"></a>
### 執行測試

要執行您的瀏覽器測試，請執行 `dusk` Artisan 命令：

```shell
php artisan dusk
```

如果您上次執行 `dusk` 命令時有測試失敗，您可以透過使用 `dusk:fails` 命令首先重新執行失敗的測試來節省時間：

```shell
php artisan dusk:fails
```

`dusk` 命令接受 Pest / PHPUnit 測試執行器通常接受的任何引數，例如允許您僅執行特定 [群組](https://docs.phpunit.de/en/10.5/annotations.html#group) 的測試：

```shell
php artisan dusk --group=foo
```

> [!NOTE]  
> 如果您正在使用 [Laravel Sail](/docs/{{version}}/sail) 管理您的本地開發環境，請查閱 Sail 文件中關於 [配置與執行 Dusk 測試](/docs/{{version}}/sail#laravel-dusk) 的說明。


<a name="manually-starting-chromedriver"></a>
#### 手動啟動 ChromeDriver

預設情況下，Dusk 將會自動嘗試啟動 ChromeDriver。如果這對您的特定系統不起作用，您可以在執行 `dusk` 命令之前手動啟動 ChromeDriver。如果您選擇手動啟動 ChromeDriver，您應該將您的 `tests/DuskTestCase.php` 檔案中的以下行註解掉：

    /**
     * Prepare for Dusk test execution.
     *
     * @beforeClass
     */
    public static function prepare(): void
    {
        // static::startChromeDriver();
    }

此外，如果您在 9515 以外的埠上啟動 ChromeDriver，您應該修改同一類別中的 `driver` 方法以反映正確的埠：

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


<a name="environment-handling"></a>
### 環境處理

為了強制 Dusk 在執行測試時使用其自己的環境檔案，請在您的專案根目錄中建立一個 `.env.dusk.{environment}` 檔案。例如，如果您將從 `local` 環境啟動 `dusk` 命令，您應該建立一個 `.env.dusk.local` 檔案。

執行測試時，Dusk 將會備份您的 `.env` 檔案並將您的 Dusk 環境重新命名為 `.env`。測試完成後，您的 `.env` 檔案將會被還原。

<a name="browser-basics"></a>
## 瀏覽器基礎

<a name="creating-browsers"></a>
### 建立瀏覽器

首先，讓我們編寫一個測試來驗證我們能否登入應用程式。產生測試後，我們可以修改它以導覽至登入頁面，輸入一些憑證，然後點擊「Login」按鈕。要建立瀏覽器實例，您可以在 Dusk 測試中呼叫 `browse` 方法：

```php tab=Pest
<?php

use App\Models\User;
use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;

uses(DatabaseMigrations::class);

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

如您在上面的範例中看到的，`browse` 方法接受一個閉包。Dusk 會自動將瀏覽器實例傳遞給此閉包，這是用於與應用程式互動並進行斷言的主要物件。

<a name="creating-multiple-browsers"></a>
#### 建立多個瀏覽器

有時，您可能需要多個瀏覽器才能正確執行測試。例如，可能需要多個瀏覽器來測試與 WebSocket 互動的聊天畫面。要建立多個瀏覽器，只需向 `browse` 方法的閉包簽章中添加更多瀏覽器引數即可：

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

<a name="navigation"></a>
### 導覽

`visit` 方法可用於導覽至應用程式內的給定 URI：

    $browser->visit('/login');

您可以使用 `visitRoute` 方法導覽至[具名路由](/docs/{{version}}/routing#named-routes)：

    $browser->visitRoute($routeName, $parameters);

您可以使用 `back` 和 `forward` 方法導覽「回上頁」和「下一頁」：

    $browser->back();

    $browser->forward();

您可以使用 `refresh` 方法重新整理頁面：

    $browser->refresh();

<a name="resizing-browser-windows"></a>
### 調整瀏覽器視窗大小

您可以使用 `resize` 方法調整瀏覽器視窗的大小：

    $browser->resize(1920, 1080);

`maximize` 方法可用於將瀏覽器視窗最大化：

    $browser->maximize();

`fitContent` 方法將調整瀏覽器視窗大小以符合其內容：

    $browser->fitContent();

當測試失敗時，Dusk 會在擷取螢幕截圖之前自動調整瀏覽器大小以符合內容。您可以透過在測試中呼叫 `disableFitOnFailure` 方法來禁用此功能：

    $browser->disableFitOnFailure();

您可以使用 `move` 方法將瀏覽器視窗移動到螢幕上的不同位置：

    $browser->move($x = 100, $y = 100);

<a name="browser-macros"></a>
### 瀏覽器巨集

如果您想定義一個可以在各種測試中重複使用的自訂瀏覽器方法，您可以使用 `Browser` 類別上的 `macro` 方法。通常，您應該從[服務提供者](/docs/{{version}}/providers)的 `boot` 方法呼叫此方法：

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

`macro` 函數接受一個名稱作為其第一個引數，一個閉包作為其第二個引數。在 `Browser` 實例上呼叫該巨集方法時，巨集的閉包將被執行：

    $this->browse(function (Browser $browser) use ($user) {
        $browser->visit('/pay')
            ->scrollToElement('#credit-card-details')
            ->assertSee('Enter Credit Card Details');
    });

<a name="authentication"></a>
### 身份驗證

通常，您會測試需要身份驗證的頁面。您可以使用 Dusk 的 `loginAs` 方法，以避免在每個測試中都與應用程式的登入畫面互動。`loginAs` 方法接受與您的可驗證模型 (authenticatable model) 相關聯的主鍵或可驗證模型實例：

    use App\Models\User;
    use Laravel\Dusk\Browser;

    $this->browse(function (Browser $browser) {
        $browser->loginAs(User::find(1))
            ->visit('/home');
    });

[!WARNING]
使用 `loginAs` 方法後，使用者會話將在檔案中的所有測試中保留。

<a name="cookies"></a>
### Cookies

您可以使用 `cookie` 方法來取得或設定加密的 cookie 值。預設情況下，由 Laravel 建立的所有 cookie 都已加密：

    $browser->cookie('name');

    $browser->cookie('name', 'Taylor');

您可以使用 `plainCookie` 方法來取得或設定未加密的 cookie 值：

    $browser->plainCookie('name');

    $browser->plainCookie('name', 'Taylor');

您可以使用 `deleteCookie` 方法刪除給定的 cookie：

    $browser->deleteCookie('name');

<a name="executing-javascript"></a>
### 執行 JavaScript

您可以使用 `script` 方法在瀏覽器中執行任意 JavaScript 語句：

    $browser->script('document.documentElement.scrollTop = 0');

    $browser->script([
        'document.body.scrollTop = 0',
        'document.documentElement.scrollTop = 0',
    ]);

    $output = $browser->script('return window.location.pathname');

<a name="taking-a-screenshot"></a>
### 擷取螢幕截圖

您可以使用 `screenshot` 方法擷取螢幕截圖並以給定的檔名儲存。所有螢幕截圖都將儲存於 `tests/Browser/screenshots` 目錄中：

    $browser->screenshot('filename');

`responsiveScreenshots` 方法可用於在各種斷點處擷取一系列螢幕截圖：

    $browser->responsiveScreenshots('filename');

`screenshotElement` 方法可用於擷取頁面上特定元素的螢幕截圖：

    $browser->screenshotElement('#selector', 'filename');

<a name="storing-console-output-to-disk"></a>
### 將主控台輸出儲存到磁碟

您可以使用 `storeConsoleLog` 方法將當前瀏覽器的主控台輸出以給定的檔名寫入磁碟。主控台輸出將儲存於 `tests/Browser/console` 目錄中：

    $browser->storeConsoleLog('filename');

<a name="storing-page-source-to-disk"></a>
### 將頁面原始碼儲存到磁碟

您可以使用 `storeSource` 方法將當前頁面的原始碼以給定的檔名寫入磁碟。頁面原始碼將儲存於 `tests/Browser/source` 目錄中：

    $browser->storeSource('filename');

<a name="interacting-with-elements"></a>
## 與元素互動


<a name="dusk-selectors"></a>
### Dusk 選擇器

選擇好的 CSS 選擇器來與元素互動是編寫 Dusk 測試中最困難的部分之一。隨著時間的推移，前端的變更可能會導致以下 CSS 選擇器使您的測試失敗：

    // HTML...

    <button>Login</button>

    // Test...

    $browser->click('.login-page .container div > button');

Dusk 選擇器讓您可以專注於編寫有效的測試，而不是記住 CSS 選擇器。要定義一個選擇器，請在您的 HTML 元素中加入 `dusk` 屬性。然後，當與 Dusk 瀏覽器互動時，在選擇器前加上 `@` 即可在測試中操作該附加的元素：

    // HTML...

    <button dusk="login-button">Login</button>

    // Test...

    $browser->click('@login-button');

如果需要，您可以透過 `selectorHtmlAttribute` 方法自訂 Dusk 選擇器所使用的 HTML 屬性。通常，此方法應在應用程式的 `AppServiceProvider` 的 `boot` 方法中呼叫：

    use Laravel\Dusk\Dusk;

    Dusk::selectorHtmlAttribute('data-dusk');


<a name="text-values-and-attributes"></a>
### 文字、值與屬性


<a name="retrieving-setting-values"></a>
#### 擷取與設定值

Dusk 提供多種方法來與頁面上元素的當前值、顯示文字和屬性互動。例如，要獲取符合給定 CSS 或 Dusk 選擇器的元素的「值」，請使用 `value` 方法：

    // Retrieve the value...
    $value = $browser->value('selector');

    // Set the value...
    $browser->value('selector', 'value');

您可以使用 `inputValue` 方法來獲取具有給定欄位名稱的輸入元素的「值」：

    $value = $browser->inputValue('field');


<a name="retrieving-text"></a>
#### 擷取文字

`text` 方法可用於擷取符合給定選擇器元素的顯示文字：

    $text = $browser->text('selector');


<a name="retrieving-attributes"></a>
#### 擷取屬性

最後，`attribute` 方法可用於擷取符合給定選擇器元素的屬性值：

    $attribute = $browser->attribute('selector', 'value');


<a name="interacting-with-forms"></a>
### 與表單互動


<a name="typing-values"></a>
#### 輸入值

Dusk 提供多種方法用於與表單和輸入元素互動。首先，讓我們看看一個將文字輸入到輸入欄位的範例：

    $browser->type('email', 'taylor@laravel.com');

請注意，雖然該方法在必要時接受一個參數，但我們不需要將 CSS 選擇器傳遞給 `type` 方法。如果未提供 CSS 選擇器，Dusk 將會搜尋具有給定 `name` 屬性的 `input` 或 `textarea` 欄位。

要將文字附加到欄位而不清除其內容，您可以使用 `append` 方法：

    $browser->type('tags', 'foo')
        ->append('tags', ', bar, baz');

您可以使用 `clear` 方法清除輸入的值：

    $browser->clear('email');

您可以指示 Dusk 使用 `typeSlowly` 方法緩慢輸入。預設情況下，Dusk 會在按鍵之間暫停 100 毫秒。要自訂按鍵之間的暫停時間，您可以將適當的毫秒數作為第三個引數傳遞給該方法：

    $browser->typeSlowly('mobile', '+1 (202) 555-5555');

    $browser->typeSlowly('mobile', '+1 (202) 555-5555', 300);

您可以使用 `appendSlowly` 方法緩慢地附加文字：

    $browser->type('tags', 'foo')
        ->appendSlowly('tags', ', bar, baz');


<a name="dropdowns"></a>
#### 下拉選單

要選取 `select` 元素上可用的值，您可以使用 `select` 方法。與 `type` 方法一樣，`select` 方法不需要完整的 CSS 選擇器。當將值傳遞給 `select` 方法時，您應該傳遞底層選項值而不是顯示文字：

    $browser->select('size', 'Large');

您可以透過省略第二個引數來選取一個隨機選項：

    $browser->select('size');

透過提供陣列作為 `select` 方法的第二個引數，您可以指示該方法選取多個選項：

    $browser->select('categories', ['Art', 'Music']);


<a name="checkboxes"></a>
#### 核取方塊

要「勾選」核取方塊輸入，您可以使用 `check` 方法。與許多其他輸入相關方法一樣，不需要完整的 CSS 選擇器。如果找不到 CSS 選擇器匹配項，Dusk 將搜尋具有匹配 `name` 屬性的核取方塊：

    $browser->check('terms');

`uncheck` 方法可用於「取消勾選」核取方塊輸入：

    $browser->uncheck('terms');


<a name="radio-buttons"></a>
#### 選項按鈕

要「選取」`radio` 輸入選項，您可以使用 `radio` 方法。與許多其他輸入相關方法一樣，不需要完整的 CSS 選擇器。如果找不到 CSS 選擇器匹配項，Dusk 將搜尋具有匹配 `name` 和 `value` 屬性的 `radio` 輸入：

    $browser->radio('size', 'large');


<a name="attaching-files"></a>
### 附加檔案

`attach` 方法可用於將檔案附加到 `file` 輸入元素。與許多其他輸入相關方法一樣，不需要完整的 CSS 選擇器。如果找不到 CSS 選擇器匹配項，Dusk 將搜尋具有匹配 `name` 屬性的 `file` 輸入：

    $browser->attach('photo', __DIR__.'/photos/mountains.png');

> [!WARNING]  
> attach 函數要求您的伺服器已安裝並啟用 `Zip` PHP 擴充功能。


<a name="pressing-buttons"></a>
### 按下按鈕

`press` 方法可用於點擊頁面上的按鈕元素。傳遞給 `press` 方法的引數可以是按鈕的顯示文字或 CSS / Dusk 選擇器：

    $browser->press('Login');

提交表單時，許多應用程式會在按下表單的提交按鈕後將其停用，然後在表單提交的 HTTP 請求完成後重新啟用該按鈕。要按下按鈕並等待按鈕重新啟用，您可以使用 `pressAndWaitFor` 方法：

    // Press the button and wait a maximum of 5 seconds for it to be enabled...
    $browser->pressAndWaitFor('Save');

    // Press the button and wait a maximum of 1 second for it to be enabled...
    $browser->pressAndWaitFor('Save', 1);


<a name="clicking-links"></a>
### 點擊連結

要點擊連結，您可以在瀏覽器實例上使用 `clickLink` 方法。`clickLink` 方法將點擊具有給定顯示文字的連結：

    $browser->clickLink($linkText);

您可以使用 `seeLink` 方法來判斷頁面上是否顯示具有給定顯示文字的連結：

    if ($browser->seeLink($linkText)) {
        // ...
    }

> [!WARNING]  
> 這些方法與 jQuery 互動。如果頁面上沒有 jQuery，Dusk 將會自動將其注入頁面，以便在測試期間可用。

<a name="using-the-keyboard"></a>
### 使用鍵盤

`keys` 方法允許您向給定元素提供比 `type` 方法通常允許的更複雜的輸入序列。例如，您可以指示 Dusk 在輸入值時按住修飾鍵。在這個範例中，當 `taylor` 輸入到符合給定選擇器的元素時，`shift` 鍵將被按住。在輸入 `taylor` 後，將在不帶任何修飾鍵的情況下輸入 `swift`：

    $browser->keys('selector', ['{shift}', 'taylor'], 'swift');

`keys` 方法的另一個有價值的用途是將「鍵盤快捷鍵」組合傳送給應用程式的主要 CSS 選擇器：

    $browser->keys('.app', ['{command}', 'j']);

> [!NOTE]  
> 所有修飾鍵，例如 `{command}`，都用 `{}` 字元包圍，並與 `Facebook\WebDriver\WebDriverKeys` 類別中定義的常數相符，該類別可[在 GitHub 上找到](https://github.com/php-webdriver/php-webdriver/blob/master/lib/WebDriverKeys.php)。

<a name="fluent-keyboard-interactions"></a>
#### 流暢的鍵盤互動

Dusk 也提供 `withKeyboard` 方法，讓您能透過 `Laravel\Dusk\Keyboard` 類別流暢地執行複雜的鍵盤互動。`Keyboard` 類別提供了 `press`、`release`、`type` 和 `pause` 方法：

    use Laravel\Dusk\Keyboard;

    $browser->withKeyboard(function (Keyboard $keyboard) {
        $keyboard->press('c')
            ->pause(1000)
            ->release('c')
            ->type(['c', 'e', 'o']);
    });

<a name="keyboard-macros"></a>
#### 鍵盤巨集

如果您想定義可在測試套件中重複使用的自訂鍵盤互動，您可以使用 `Keyboard` 類別提供的 `macro` 方法。通常，您應該從服務提供者 (service provider) 的 `boot` 方法中呼叫此方法：

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

`macro` 函數接受一個名稱作為第一個引數，一個閉包作為第二個引數。當作為 `Keyboard` 實例上的方法呼叫巨集時，巨集的閉包將被執行：

    $browser->click('@textarea')
        ->withKeyboard(fn (Keyboard $keyboard) => $keyboard->copy())
        ->click('@another-textarea')
        ->withKeyboard(fn (Keyboard $keyboard) => $keyboard->paste());

<a name="using-the-mouse"></a>
### 使用滑鼠

<a name="clicking-on-elements"></a>
#### 點擊元素

`click` 方法可用於點擊符合給定 CSS 或 Dusk 選擇器的元素：

    $browser->click('.selector');

`clickAtXPath` 方法可用於點擊符合給定 XPath 運算式的元素：

    $browser->clickAtXPath('//div[@class = "selector"]');

`clickAtPoint` 方法可用於點擊瀏覽器可視區域內給定座標對處的最頂層元素：

    $browser->clickAtPoint($x = 0, $y = 0);

`doubleClick` 方法可用於模擬滑鼠的雙擊：

    $browser->doubleClick();

    $browser->doubleClick('.selector');

`rightClick` 方法可用於模擬滑鼠的右鍵點擊：

    $browser->rightClick();

    $browser->rightClick('.selector');

`clickAndHold` 方法可用於模擬滑鼠按鈕被點擊並按住。隨後呼叫 `releaseMouse` 方法將撤銷此行為並釋放滑鼠按鈕：

    $browser->clickAndHold('.selector');

    $browser->clickAndHold()
        ->pause(1000)
        ->releaseMouse();

`controlClick` 方法可用於模擬瀏覽器中的 `ctrl+click` 事件：

    $browser->controlClick();

    $browser->controlClick('.selector');

<a name="mouseover"></a>
#### 滑鼠懸停

當您需要將滑鼠移到符合給定 CSS 或 Dusk 選擇器的元素上方時，可以使用 `mouseover` 方法：

    $browser->mouseover('.selector');

<a name="drag-drop"></a>
#### 拖曳

`drag` 方法可用於將符合給定選擇器的元素拖曳到另一個元素：

    $browser->drag('.from-selector', '.to-selector');

或者，您可以將元素沿單一方向拖曳：

    $browser->dragLeft('.selector', $pixels = 10);
    $browser->dragRight('.selector', $pixels = 10);
    $browser->dragUp('.selector', $pixels = 10);
    $browser->dragDown('.selector', $pixels = 10);

最後，您可以按給定偏移量拖曳元素：

    $browser->dragOffset('.selector', $x = 10, $y = 10);

<a name="javascript-dialogs"></a>
### JavaScript 對話框

Dusk 提供了多種與 JavaScript 對話框互動的方法。例如，您可以使用 `waitForDialog` 方法等待 JavaScript 對話框出現。此方法接受一個選用引數，指示等待對話框出現的秒數：

    $browser->waitForDialog($seconds = null);

`assertDialogOpened` 方法可用於斷言對話框已顯示並包含給定的訊息：

    $browser->assertDialogOpened('Dialog message');

如果 JavaScript 對話框包含提示，您可以使用 `typeInDialog` 方法在提示中輸入值：

    $browser->typeInDialog('Hello World');

要透過點擊「確定」按鈕來關閉開啟的 JavaScript 對話框，您可以呼叫 `acceptDialog` 方法：

    $browser->acceptDialog();

要透過點擊「取消」按鈕來關閉開啟的 JavaScript 對話框，您可以呼叫 `dismissDialog` 方法：

    $browser->dismissDialog();

<a name="interacting-with-iframes"></a>
### 與內嵌框架互動

如果您需要與 iframe 中的元素互動，可以使用 `withinFrame` 方法。所有在提供給 `withinFrame` 方法的閉包中發生的元素互動，都將被限定在指定 iframe 的上下文中：

    $browser->withinFrame('#credit-card-details', function ($browser) {
        $browser->type('input[name="cardnumber"]', '4242424242424242')
            ->type('input[name="exp-date"]', '1224')
            ->type('input[name="cvc"]', '123')
            ->press('Pay');
    });

<a name="scoping-selectors"></a>
### 範圍化選擇器

有時您可能希望在給定選擇器內執行多個操作。例如，您可能希望斷言某個文字只存在於表格中，然後點擊該表格中的按鈕。您可以使用 `with` 方法來實現此目的。在提供給 `with` 方法的閉包中執行的所有操作都將被限定在原始選擇器範圍內：

    $browser->with('.table', function (Browser $table) {
        $table->assertSee('Hello World')
            ->clickLink('Delete');
    });

您可能偶爾需要在當前範圍之外執行斷言。您可以使用 `elsewhere` 和 `elsewhereWhenAvailable` 方法來實現此目的：

     $browser->with('.table', function (Browser $table) {
        // 當前範圍是 `body .table`...

        $browser->elsewhere('.page-title', function (Browser $title) {
            // 當前範圍是 `body .page-title`...
            $title->assertSee('Hello World');
        });

        $browser->elsewhereWhenAvailable('.page-title', function (Browser $title) {
            // 當前範圍是 `body .page-title`...
            $title->assertSee('Hello World');
        });
     });

<a name="waiting-for-elements"></a>
### 等待元素

測試大量使用 JavaScript 的應用程式時，在繼續測試之前，通常需要「等待」特定的元素或資料可用。Dusk 讓這變得輕而易舉。透過多種方法，您可以等待元素在頁面上可見，甚至等到給定的 JavaScript 表達式評估為 `true`。

<a name="waiting"></a>
#### 等待

如果您只需要讓測試暫停特定的毫秒數，請使用 `pause` 方法：

    $browser->pause(1000);

如果您只需要在給定條件為 `true` 時暫停測試，請使用 `pauseIf` 方法：

    $browser->pauseIf(App::environment('production'), 1000);

同樣地，如果您需要在給定條件不為 `true` 時暫停測試，您可以使用 `pauseUnless` 方法：

    $browser->pauseUnless(App::environment('testing'), 1000);

<a name="waiting-for-selectors"></a>
#### 等待選擇器

`waitFor` 方法可用來暫停測試執行，直到頁面上顯示出符合指定 CSS 或 Dusk 選擇器的元素。預設情況下，這將使測試暫停最多五秒，然後才會拋出例外。如有必要，您可以將自訂逾時閾值作為方法的第二個引數傳遞：

    // 最多等待五秒以獲取選擇器...
    $browser->waitFor('.selector');

    // 最多等待一秒以獲取選擇器...
    $browser->waitFor('.selector', 1);

您也可以等待直到符合指定選擇器的元素包含指定的文字：

    // 最多等待五秒以使選擇器包含指定文字...
    $browser->waitForTextIn('.selector', 'Hello World');

    // 最多等待一秒以使選擇器包含指定文字...
    $browser->waitForTextIn('.selector', 'Hello World', 1);

您也可以等到符合指定選擇器的元素從頁面中消失：

    // 最多等待五秒直到選擇器消失...
    $browser->waitUntilMissing('.selector');

    // 最多等待一秒直到選擇器消失...
    $browser->waitUntilMissing('.selector', 1);

或者，您也可以等待直到符合指定選擇器的元素被啟用或停用：

    // 最多等待五秒直到選擇器被啟用...
    $browser->waitUntilEnabled('.selector');

    // 最多等待一秒直到選擇器被啟用...
    $browser->waitUntilEnabled('.selector', 1);

    // 最多等待五秒直到選擇器被停用...
    $browser->waitUntilDisabled('.selector');

    // 最多等待一秒直到選擇器被停用...
    $browser->waitUntilDisabled('.selector', 1);

<a name="scoping-selectors-when-available"></a>
#### 選擇器可用時的範圍化

有時，您可能希望等待符合指定選擇器的元素出現，然後與該元素互動。例如，您可能希望等到模態視窗可用，然後在該模態視窗中按下「OK」按鈕。`whenAvailable` 方法可用來達成此目的。在給定閉包中執行的所有元素操作都將範圍化到原始選擇器：

    $browser->whenAvailable('.modal', function (Browser $modal) {
        $modal->assertSee('Hello World')
            ->press('OK');
    });

<a name="waiting-for-text"></a>
#### 等待文字

`waitForText` 方法可用來等待直到頁面上顯示出指定的文字：

    // 最多等待五秒以獲取文字...
    $browser->waitForText('Hello World');

    // 最多等待一秒以獲取文字...
    $browser->waitForText('Hello World', 1);

您可以使用 `waitUntilMissingText` 方法來等待直到顯示的文字從頁面中移除：

    // 最多等待五秒以使文字被移除...
    $browser->waitUntilMissingText('Hello World');

    // 最多等待一秒以使文字被移除...
    $browser->waitUntilMissingText('Hello World', 1);

<a name="waiting-for-links"></a>
#### 等待連結

`waitForLink` 方法可用來等待直到頁面上顯示出指定的連結文字：

    // 最多等待五秒以獲取連結...
    $browser->waitForLink('Create');

    // 最多等待一秒以獲取連結...
    $browser->waitForLink('Create', 1);

<a name="waiting-for-inputs"></a>
#### 等待輸入

`waitForInput` 方法可用來等待直到頁面上顯示出指定的輸入欄位：

    // 最多等待五秒以獲取輸入...
    $browser->waitForInput($field);

    // 最多等待一秒以獲取輸入...
    $browser->waitForInput($field, 1);

<a name="waiting-on-the-page-location"></a>
#### 等待頁面位置

當進行路徑斷言，例如 `$browser->assertPathIs('/home')` 時，如果 `window.location.pathname` 正在非同步更新，該斷言可能會失敗。您可以使用 `waitForLocation` 方法來等待位置成為給定值：

    $browser->waitForLocation('/secret');

`waitForLocation` 方法也可用來等待目前視窗位置成為完整的 URL：

    $browser->waitForLocation('https://example.com/path');

您也可以等待 [命名路由](/docs/{{version}}/routing#named-routes) 的位置：

    $browser->waitForRoute($routeName, $parameters);

<a name="waiting-for-page-reloads"></a>
#### 等待頁面重新載入

如果您需要在執行某個動作後等待頁面重新載入，請使用 `waitForReload` 方法：

    use Laravel\Dusk\Browser;

    $browser->waitForReload(function (Browser $browser) {
        $browser->press('Submit');
    })
    ->assertSee('Success!');

由於等待頁面重新載入的需求通常發生在點擊按鈕之後，為了方便起見，您可以使用 `clickAndWaitForReload` 方法：

    $browser->clickAndWaitForReload('.selector')
        ->assertSee('something');

<a name="waiting-on-javascript-expressions"></a>
#### 等待 JavaScript 表達式

有時，您可能希望暫停測試執行，直到給定的 JavaScript 表達式評估為 `true`。您可以使用 `waitUntil` 方法輕鬆達成此目的。當向此方法傳遞表達式時，您不需要包含 `return` 關鍵字或結尾分號：

    // 最多等待五秒以使表達式為 true...
    $browser->waitUntil('App.data.servers.length > 0');

    // 最多等待一秒以使表達式為 true...
    $browser->waitUntil('App.data.servers.length > 0', 1);

<a name="waiting-on-vue-expressions"></a>
#### 等待 Vue 表達式

`waitUntilVue` 和 `waitUntilVueIsNot` 方法可用來等待直到 [Vue component](https://vuejs.org) 屬性具有給定值：

    // 等待直到元件屬性包含給定值...
    $browser->waitUntilVue('user.name', 'Taylor', '@user');

    // 等待直到元件屬性不包含給定值...
    $browser->waitUntilVueIsNot('user.name', null, '@user');

<a name="waiting-for-javascript-events"></a>
#### 等待 JavaScript 事件

`waitForEvent` 方法可用來暫停測試執行，直到 JavaScript 事件發生：

    $browser->waitForEvent('load');

事件監聽器預設會附加到目前作用域，即 `body` 元素。當使用範圍化選擇器時，事件監聽器將會附加到匹配的元素：

    $browser->with('iframe', function (Browser $iframe) {
        // 等待 iframe 的載入事件...
        $iframe->waitForEvent('load');
    });

您也可以提供選擇器作為 `waitForEvent` 方法的第二個引數，以將事件監聽器附加到特定元素：

    $browser->waitForEvent('load', '.selector');

您也可以等待 `document` 和 `window` 物件上的事件：

    // 等待直到 document 被捲動...
    $browser->waitForEvent('scroll', 'document');

    // 最多等待五秒直到 window 被調整大小...
    $browser->waitForEvent('resize', 'window', 5);

<a name="waiting-with-a-callback"></a>
#### 使用回呼等待

Dusk 中的許多「等待」方法都依賴於底層的 `waitUsing` 方法。您可以直接使用此方法來等待給定的閉包傳回 `true`。`waitUsing` 方法接受最大等待秒數、評估閉包的間隔、閉包本身，以及可選的失敗訊息：

    $browser->waitUsing(10, 1, function () use ($something) {
        return $something->isReady();
    }, "Something wasn't ready in time.");

<a name="scrolling-an-element-into-view"></a>
### 將元素捲動到檢視區

有時候您可能無法點擊某個元素，因為它在瀏覽器的可視區域之外。`scrollIntoView` 方法將會捲動瀏覽器視窗，直到給定選擇器所指定的元素進入檢視區為止：

    $browser->scrollIntoView('.selector')
        ->click('.selector');

<a name="available-assertions"></a>
## 可用的斷言

Dusk 提供了多種可對應用程式進行的斷言。所有可用的斷言都記錄在下方列表中：

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

斷言頁面標題與給定的文字相符：

    $browser->assertTitle($title);


<a name="assert-title-contains"></a>
#### assertTitleContains

斷言頁面標題包含給定的文字：

    $browser->assertTitleContains($title);


<a name="assert-url-is"></a>
#### assertUrlIs

斷言目前 URL (不含查詢字串) 與給定的字串相符：

    $browser->assertUrlIs($url);


<a name="assert-scheme-is"></a>
#### assertSchemeIs

斷言目前 URL 方案 (scheme) 與給定的方案相符：

    $browser->assertSchemeIs($scheme);


<a name="assert-scheme-is-not"></a>
#### assertSchemeIsNot

斷言目前 URL 方案 (scheme) 與給定的方案不相符：

    $browser->assertSchemeIsNot($scheme);


<a name="assert-host-is"></a>
#### assertHostIs

斷言目前 URL 主機與給定的主機相符：

    $browser->assertHostIs($host);


<a name="assert-host-is-not"></a>
#### assertHostIsNot

斷言目前 URL 主機與給定的主機不相符：

    $browser->assertHostIsNot($host);


<a name="assert-port-is"></a>
#### assertPortIs

斷言目前 URL 埠號與給定的埠號相符：

    $browser->assertPortIs($port);


<a name="assert-port-is-not"></a>
#### assertPortIsNot

斷言目前 URL 埠號與給定的埠號不相符：

    $browser->assertPortIsNot($port);


<a name="assert-path-begins-with"></a>
#### assertPathBeginsWith

斷言目前 URL 路徑以給定的路徑開頭：

    $browser->assertPathBeginsWith('/home');


<a name="assert-path-ends-with"></a>
#### assertPathEndsWith

斷言目前 URL 路徑以給定的路徑結尾：

    $browser->assertPathEndsWith('/home');


<a name="assert-path-contains"></a>
#### assertPathContains

斷言目前 URL 路徑包含給定的路徑：

    $browser->assertPathContains('/home');


<a name="assert-path-is"></a>
#### assertPathIs

斷言目前路徑與給定的路徑相符：

    $browser->assertPathIs('/home');


<a name="assert-path-is-not"></a>
#### assertPathIsNot

斷言目前路徑與給定的路徑不相符：

    $browser->assertPathIsNot('/home');


<a name="assert-route-is"></a>
#### assertRouteIs

斷言目前 URL 與給定的 [具名路由](/docs/{{version}}/routing#named-routes) 的 URL 相符：

    $browser->assertRouteIs($name, $parameters);


<a name="assert-query-string-has"></a>
#### assertQueryStringHas

斷言給定的查詢字串參數存在：

    $browser->assertQueryStringHas($name);

斷言給定的查詢字串參數存在並具有給定的值：

    $browser->assertQueryStringHas($name, $value);


<a name="assert-query-string-missing"></a>
#### assertQueryStringMissing

斷言給定的查詢字串參數不存在：

    $browser->assertQueryStringMissing($name);


<a name="assert-fragment-is"></a>
#### assertFragmentIs

斷言 URL 的目前雜湊片段與給定的片段相符：

    $browser->assertFragmentIs('anchor');


<a name="assert-fragment-begins-with"></a>
#### assertFragmentBeginsWith

斷言 URL 的目前雜湊片段以給定的片段開頭：

    $browser->assertFragmentBeginsWith('anchor');


<a name="assert-fragment-is-not"></a>
#### assertFragmentIsNot

斷言 URL 的目前雜湊片段與給定的片段不相符：

    $browser->assertFragmentIsNot('anchor');


<a name="assert-has-cookie"></a>
#### assertHasCookie

斷言給定的加密 cookie 存在：

    $browser->assertHasCookie($name);


<a name="assert-has-plain-cookie"></a>
#### assertHasPlainCookie

斷言給定的未加密 cookie 存在：

    $browser->assertHasPlainCookie($name);


<a name="assert-cookie-missing"></a>
#### assertCookieMissing

斷言給定的加密 cookie 不存在：

    $browser->assertCookieMissing($name);


<a name="assert-plain-cookie-missing"></a>
#### assertPlainCookieMissing

斷言給定的未加密 cookie 不存在：

    $browser->assertPlainCookieMissing($name);


<a name="assert-cookie-value"></a>
#### assertCookieValue

斷言加密 cookie 具有給定的值：

    $browser->assertCookieValue($name, $value);

<a name="assert-plain-cookie-value"></a>
#### assertPlainCookieValue

斷言一個未加密的 Cookie 具有給定的值：

    $browser->assertPlainCookieValue($name, $value);


<a name="assert-see"></a>
#### assertSee

斷言頁面上存在給定的文字：

    $browser->assertSee($text);


<a name="assert-dont-see"></a>
#### assertDontSee

斷言頁面上不存在給定的文字：

    $browser->assertDontSee($text);


<a name="assert-see-in"></a>
#### assertSeeIn

斷言選擇器中存在給定的文字：

    $browser->assertSeeIn($selector, $text);


<a name="assert-dont-see-in"></a>
#### assertDontSeeIn

斷言選擇器中不存在給定的文字：

    $browser->assertDontSeeIn($selector, $text);


<a name="assert-see-anything-in"></a>
#### assertSeeAnythingIn

斷言選擇器中存在任何文字：

    $browser->assertSeeAnythingIn($selector);


<a name="assert-see-nothing-in"></a>
#### assertSeeNothingIn

斷言選擇器中不存在任何文字：

    $browser->assertSeeNothingIn($selector);


<a name="assert-script"></a>
#### assertScript

斷言給定的 JavaScript 表達式評估為給定的值：

    $browser->assertScript('window.isLoaded')
            ->assertScript('document.readyState', 'complete');


<a name="assert-source-has"></a>
#### assertSourceHas

斷言頁面中存在給定的原始碼：

    $browser->assertSourceHas($code);


<a name="assert-source-missing"></a>
#### assertSourceMissing

斷言頁面中不存在給定的原始碼：

    $browser->assertSourceMissing($code);


<a name="assert-see-link"></a>
#### assertSeeLink

斷言頁面上存在給定的連結：

    $browser->assertSeeLink($linkText);


<a name="assert-dont-see-link"></a>
#### assertDontSeeLink

斷言頁面上不存在給定的連結：

    $browser->assertDontSeeLink($linkText);


<a name="assert-input-value"></a>
#### assertInputValue

斷言給定的輸入欄位具有給定的值：

    $browser->assertInputValue($field, $value);


<a name="assert-input-value-is-not"></a>
#### assertInputValueIsNot

斷言給定的輸入欄位不具有給定的值：

    $browser->assertInputValueIsNot($field, $value);


<a name="assert-checked"></a>
#### assertChecked

斷言給定的核取方塊已勾選：

    $browser->assertChecked($field);


<a name="assert-not-checked"></a>
#### assertNotChecked

斷言給定的核取方塊未勾選：

    $browser->assertNotChecked($field);


<a name="assert-indeterminate"></a>
#### assertIndeterminate

斷言給定的核取方塊處於不確定狀態：

    $browser->assertIndeterminate($field);


<a name="assert-radio-selected"></a>
#### assertRadioSelected

斷言給定的單選欄位已選取：

    $browser->assertRadioSelected($field, $value);


<a name="assert-radio-not-selected"></a>
#### assertRadioNotSelected

斷言給定的單選欄位未選取：

    $browser->assertRadioNotSelected($field, $value);


<a name="assert-selected"></a>
#### assertSelected

斷言給定的下拉選單已選取給定的值：

    $browser->assertSelected($field, $value);


<a name="assert-not-selected"></a>
#### assertNotSelected

斷言給定的下拉選單未選取給定的值：

    $browser->assertNotSelected($field, $value);


<a name="assert-select-has-options"></a>
#### assertSelectHasOptions

斷言給定的值陣列可供選取：

    $browser->assertSelectHasOptions($field, $values);


<a name="assert-select-missing-options"></a>
#### assertSelectMissingOptions

斷言給定的值陣列不可供選取：

    $browser->assertSelectMissingOptions($field, $values);


<a name="assert-select-has-option"></a>
#### assertSelectHasOption

斷言給定的值可供給定欄位選取：

    $browser->assertSelectHasOption($field, $value);


<a name="assert-select-missing-option"></a>
#### assertSelectMissingOption

斷言給定的值不可供選取：

    $browser->assertSelectMissingOption($field, $value);


<a name="assert-value"></a>
#### assertValue

斷言符合給定選擇器的元素具有給定的值：

    $browser->assertValue($selector, $value);


<a name="assert-value-is-not"></a>
#### assertValueIsNot

斷言符合給定選擇器的元素不具有給定的值：

    $browser->assertValueIsNot($selector, $value);


<a name="assert-attribute"></a>
#### assertAttribute

斷言符合給定選擇器的元素在提供的屬性中具有給定的值：

    $browser->assertAttribute($selector, $attribute, $value);


<a name="assert-attribute-missing"></a>
#### assertAttributeMissing

斷言符合給定選擇器的元素遺失提供的屬性：

    $browser->assertAttributeMissing($selector, $attribute);



<a name="assert-attribute-contains"></a>
#### assertAttributeContains

斷言符合給定選擇器的元素在提供的屬性中包含給定的值：

    $browser->assertAttributeContains($selector, $attribute, $value);


<a name="assert-attribute-doesnt-contain"></a>
#### assertAttributeDoesntContain

斷言符合給定選擇器的元素在提供的屬性中不包含給定的值：

    $browser->assertAttributeDoesntContain($selector, $attribute, $value);


<a name="assert-aria-attribute"></a>
#### assertAriaAttribute

斷言符合給定選擇器的元素在提供的 aria 屬性中具有給定的值：

    $browser->assertAriaAttribute($selector, $attribute, $value);

例如，給定標記 `<button aria-label="Add"></button>`，您可以像這樣斷言 `aria-label` 屬性：

    $browser->assertAriaAttribute('button', 'label', 'Add')


<a name="assert-data-attribute"></a>
#### assertDataAttribute

斷言符合給定選擇器的元素在提供的 data 屬性中具有給定的值：

    $browser->assertDataAttribute($selector, $attribute, $value);

例如，給定標記 `<tr id="row-1" data-content="attendees"></tr>`，您可以像這樣斷言 `data-label` 屬性：

    $browser->assertDataAttribute('#row-1', 'content', 'attendees')


<a name="assert-visible"></a>
#### assertVisible

斷言符合給定選擇器的元素可見：

    $browser->assertVisible($selector);


<a name="assert-present"></a>
#### assertPresent

斷言符合給定選擇器的元素存在於原始碼中：

    $browser->assertPresent($selector);


<a name="assert-not-present"></a>
#### assertNotPresent

斷言符合給定選擇器的元素不存在於原始碼中：

    $browser->assertNotPresent($selector);


<a name="assert-missing"></a>
#### assertMissing

斷言符合給定選擇器的元素不可見：

    $browser->assertMissing($selector);


<a name="assert-input-present"></a>
#### assertInputPresent

斷言存在具有給定名稱的輸入：

    $browser->assertInputPresent($name);


<a name="assert-input-missing"></a>
#### assertInputMissing

斷言原始碼中不存在具有給定名稱的輸入：

    $browser->assertInputMissing($name);


<a name="assert-dialog-opened"></a>
#### assertDialogOpened

斷言已開啟具有給定訊息的 JavaScript 對話框：

    $browser->assertDialogOpened($message);


<a name="assert-enabled"></a>
#### assertEnabled

斷言給定的欄位已啟用：

    $browser->assertEnabled($field);


<a name="assert-disabled"></a>
#### assertDisabled

斷言給定的欄位已停用：

    $browser->assertDisabled($field);


<a name="assert-button-enabled"></a>
#### assertButtonEnabled

斷言給定的按鈕已啟用：

    $browser->assertButtonEnabled($button);


<a name="assert-button-disabled"></a>
#### assertButtonDisabled

斷言給定的按鈕已停用：

    $browser->assertButtonDisabled($button);

<a name="assert-focused"></a>
#### assertFocused

確認指定的欄位已獲取焦點：

    $browser->assertFocused($field);


<a name="assert-not-focused"></a>
#### assertNotFocused

確認指定的欄位未獲取焦點：

    $browser->assertNotFocused($field);


<a name="assert-authenticated"></a>
#### assertAuthenticated

確認使用者已通過身份驗證：

    $browser->assertAuthenticated();


<a name="assert-guest"></a>
#### assertGuest

確認使用者未通過身份驗證：

    $browser->assertGuest();


<a name="assert-authenticated-as"></a>
#### assertAuthenticatedAs

確認使用者已以指定的使用者身份通過身份驗證：

    $browser->assertAuthenticatedAs($user);


<a name="assert-vue"></a>
#### assertVue

Dusk 甚至允許您對 [Vue component](https://vuejs.org) 資料的狀態進行斷言。舉例來說，假設您的應用程式包含以下 Vue component：

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

您可以對 Vue component 的狀態進行斷言，如下所示：

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

確認指定的 Vue component 資料屬性與指定的值不符：

    $browser->assertVueIsNot($property, $value, $componentSelector = null);


<a name="assert-vue-contains"></a>
#### assertVueContains

確認指定的 Vue component 資料屬性是一個陣列且包含指定的值：

    $browser->assertVueContains($property, $value, $componentSelector = null);


<a name="assert-vue-doesnt-contain"></a>
#### assertVueDoesntContain

確認指定的 Vue component 資料屬性是一個陣列且不包含指定的值：

    $browser->assertVueDoesntContain($property, $value, $componentSelector = null);

<a name="pages"></a>
## 頁面

有時候，測試需要依序執行多個複雜動作。這會讓你的測試變得難以閱讀和理解。Dusk Pages (頁面) 允許你定義表達性動作，然後透過單一方法在特定頁面上執行。頁面也允許你為應用程式或單一頁面定義常用選擇器的快捷方式。


<a name="generating-pages"></a>
### 產生頁面

要產生一個頁面物件，請執行 `dusk:page` Artisan 命令。所有頁面物件都將放置在你的應用程式的 `tests/Browser/Pages` 目錄中：

    php artisan dusk:page Login


<a name="configuring-pages"></a>
### 配置頁面

預設情況下，頁面有三個方法：`url`、`assert` 和 `elements`。我們現在將討論 `url` 和 `assert` 方法。`elements` 方法將在[下面更詳細地討論](#shorthand-selectors)。


<a name="the-url-method"></a>
#### `url` 方法

`url` 方法應該回傳代表該頁面的 URL 路徑。Dusk 會在瀏覽器中導覽至該頁面時使用此 URL：

    /**
     * Get the URL for the page.
     */
    public function url(): string
    {
        return '/login';
    }


<a name="the-assert-method"></a>
#### `assert` 方法

`assert` 方法可以進行任何必要的斷言，以驗證瀏覽器確實位於指定頁面。實際上不需要在此方法中放置任何內容；但是，如果你願意，可以自由地進行這些斷言。這些斷言會在導覽至頁面時自動執行：

    /**
     * Assert that the browser is on the page.
     */
    public function assert(Browser $browser): void
    {
        $browser->assertPathIs($this->url());
    }


<a name="navigating-to-pages"></a>
### 導覽至頁面

一旦定義了頁面，你就可以使用 `visit` 方法導覽到它：

    use Tests\Browser\Pages\Login;

    $browser->visit(new Login);

有時候你可能已經在某個頁面上，需要將該頁面的選擇器和方法「載入」到目前的測試上下文中。這在按下按鈕並被重新導向到指定頁面而沒有明確導覽到該頁面時很常見。在這種情況下，你可以使用 `on` 方法來載入頁面：

    use Tests\Browser\Pages\CreatePlaylist;

    $browser->visit('/dashboard')
            ->clickLink('Create Playlist')
            ->on(new CreatePlaylist)
            ->assertSee('@create');


<a name="shorthand-selectors"></a>
### 縮寫選擇器

頁面類別中的 `elements` 方法允許你為頁面上的任何 CSS 選擇器定義快速、易於記憶的快捷方式。例如，讓我們為應用程式登入頁面中的「email」輸入欄位定義一個快捷方式：

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

一旦定義了快捷方式，你就可以在任何通常會使用完整 CSS 選擇器的地方使用該縮寫選擇器：

    $browser->type('@email', 'taylor@laravel.com');


<a name="global-shorthand-selectors"></a>
#### 全域縮寫選擇器

安裝 Dusk 後，一個基礎的 `Page` 類別會被放置在你的 `tests/Browser/Pages` 目錄中。這個類別包含一個 `siteElements` 方法，可用於定義應該在應用程式中每個頁面上都可用的全域縮寫選擇器：

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


<a name="page-methods"></a>
### 頁面方法

除了頁面上定義的預設方法之外，你還可以定義可以在整個測試中使用的額外方法。例如，假設我們正在建立一個音樂管理應用程式。應用程式的某個頁面常見的動作可能是建立一個播放清單。與其在每個測試中重寫建立播放清單的邏輯，不如在頁面類別上定義一個 `createPlaylist` 方法：

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

一旦定義了該方法，你就可以在任何使用該頁面的測試中使用它。瀏覽器實例會自動作為第一個參數傳遞給自訂頁面方法：

    use Tests\Browser\Pages\Dashboard;

    $browser->visit(new Dashboard)
            ->createPlaylist('My Playlist')
            ->assertSee('My Playlist');

<a name="components"></a>
## 元件

元件類似於 Dusk 的「頁面物件」，但旨在用於應用程式中重複使用的 UI 和功能區塊，例如導覽列或通知視窗。因此，元件不綁定特定 URL。

<a name="generating-components"></a>
### 產生元件

若要產生元件，請執行 `dusk:component` Artisan 命令。新元件會放置在 `tests/Browser/Components` 目錄中：

    php artisan dusk:component DatePicker

如上所示，「日期選擇器」就是一種可能存在於應用程式中多個頁面的元件範例。在整個測試套件的數十個測試中，手動編寫瀏覽器自動化邏輯來選擇日期可能會變得繁瑣。相反地，我們可以定義一個 Dusk 元件來表示日期選擇器，讓我們能夠將該邏輯封裝在元件中：

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


<a name="using-components"></a>
### 使用元件

一旦元件被定義，我們就可以在任何測試中輕鬆地從日期選擇器中選擇日期。而且，如果選擇日期所需的邏輯發生變化，我們只需更新該元件即可：

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\DatabaseMigrations;
use Laravel\Dusk\Browser;
use Tests\Browser\Components\DatePicker;

uses(DatabaseMigrations::class);

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

<a name="continuous-integration"></a>
## 持續整合

> [!WARNING]  
> 大多數 Dusk 持續整合配置預期您的 Laravel 應用程式是使用內建的 PHP 開發伺服器在 8000 埠提供服務。因此，在繼續之前，您應該確保您的持續整合環境中 `APP_URL` 環境變數的值為 `http://127.0.0.1:8000`。


<a name="running-tests-on-heroku-ci"></a>
### Heroku CI

要在 [Heroku CI](https://www.heroku.com/continuous-integration) 上執行 Dusk 測試，請將以下 Google Chrome buildpack 和腳本添加到您的 Heroku `app.json` 檔案中：

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


<a name="running-tests-on-travis-ci"></a>
### Travis CI

要在 [Travis CI](https://travis-ci.org) 上執行您的 Dusk 測試，請使用以下 `.travis.yml` 配置。由於 Travis CI 不是圖形環境，我們需要採取一些額外的步驟來啟動 Chrome 瀏覽器。此外，我們將使用 `php artisan serve` 來啟動 PHP 的內建網頁伺服器：

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

如果您正在使用 [GitHub Actions](https://github.com/features/actions) 來執行您的 Dusk 測試，您可以使用以下配置檔案作為起點。與 TravisCI 類似，我們將使用 `php artisan serve` 命令來啟動 PHP 的內建網頁伺服器：

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
      - uses: actions/checkout@v4
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

要了解更多關於在 Chipper CI 上執行 Dusk 測試的資訊，包括如何使用資料庫，請查閱 [官方 Chipper CI 文件](https://chipperci.com/docs/testing/laravel-dusk-new/)。